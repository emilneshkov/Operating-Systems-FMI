# Седмица 10 — Процеси в C

## Съдържание

1. [Дървото на процесите](#process-tree)
2. [Как работят процесите — адресно пространство](#address-space)
3. [fork() — създаване на нов процес](#fork)
4. [getpid() и getppid()](#getpid)
5. [exec() фамилията — замяна на програма](#exec)
6. [fork() + exec() pattern](#fork-exec)
7. [wait() и waitpid()](#wait)
8. [kill() — изпращане на сигнали](#kill)
9. [sigaction() — прихващане на сигнали](#sigaction)
10. [Context Switching](#context-switching)

---

<a id="process-tree"></a>

## Дървото на процесите

В Linux **всеки процес има точно един родител**. Единственото изключение е `init` (PID 1) — той е коренът на дървото и родител на всичко останало. Това дърво не е просто визуализация — то е фундаментална структура на ядрото.

```
PID 1: systemd (init)
├── PID 234: sshd
│   └── PID 891: sshd (сесия)
│       └── PID 892: bash
│           ├── PID 1203: vim notes.c
│           └── PID 1204: gcc main.c
├── PID 301: cron
└── PID 412: nginx
    ├── PID 413: nginx (worker)
    └── PID 414: nginx (worker)
```

Всеки процес знае своя **PID** и **PPID (Parent PID)**:

```c
#include <unistd.h>
#include <stdio.h>

pid_t getpid(void);   // върни PID на текущия процес
pid_t getppid(void);  // върни PID на родителя

int main(void) {
    dprintf(STDOUT_FILENO, "Аз съм PID  %d\n", getpid());
    dprintf(STDOUT_FILENO, "Родителят е %d\n", getppid());
    return 0;
}
```

```bash
pstree -p          # визуализира дървото с PID-ове
ps axf             # ls-подобен изглед с ASCII дърво
```

### Защо дървото е важно

**Наследяване на ресурси.** Когато процес се ражда, той наследява от родителя си: отворените file descriptors, environment variables, текущата директория, права (UID/GID), signal handlers. Това е механизмът, по който shell-ът предава FD 0/1/2 на всяка стартирана команда.

**Orphan процеси.** Ако родителят умре преди детето, детето се "осиновява" от `init` (PID 1) — ядрото автоматично задава `ppid = 1`. Затова процесите никога не остават без родител.

**Zombie процеси.** Когато дете завърши (`exit()`), то не изчезва веднага — остава като **zombie** в таблицата на процесите, докато родителят не прочете exit кода му чрез `wait()`. Ако родителят никога не извика `wait()`, zombie-то остава завинаги (или докато родителят умре).

```c
#include <sys/wait.h>

pid_t wait(int *status);      // чака КОЕ ДА Е дете да завърши
pid_t waitpid(pid_t pid, int *status, int options);  // чака конкретно дете
```

---

<a id="address-space"></a>

## Как работят процесите — адресно пространство

Всеки процес живее в собствена **виртуална адресна среда** — изолирано пространство от адреси, което е негово и само негово. Два процеса могат да имат данни на един и същ виртуален адрес, без да се притесняват един за друг — ядрото ги превежда до различни физически адреси.

### Структурата на адресното пространство

На 64-битова Linux система типичното адресно пространство на един процес изглежда така:

```
Висок адрес (0xFFFFFFFFFFFFFFFF)
┌────────────────────────────┐
│       Kernel space         │  ← недостъпен от user space
├────────────────────────────┤  0xFFFF800000000000
│         Stack              │  ← расте надолу (↓)
│    (локални променливи,    │
│     return адреси,         │
│     аргументи на функции)  │
├────────────────────────────┤
│           ↓                │
│      (свободно)            │
│           ↑                │
├────────────────────────────┤
│          Heap              │  ← расте нагоре (↑)
│    (malloc / динамична     │
│         памет)             │
├────────────────────────────┤
│   BSS segment              │  ← неинициализирани глобални
│   (нулирани при старт)     │
├────────────────────────────┤
│   Data segment             │  ← инициализирани глобални и static
├────────────────────────────┤
│   Text segment             │  ← машинен код (read-only)
├────────────────────────────┤
│   (Reserved / NULL guard)  │  ← адрес 0 е умишлено невалиден
└────────────────────────────┘
Нисък адрес (0x0000000000000000)
```

| Сегмент   | Съдържа                                        | Свойства              |
| --------- | ---------------------------------------------- | --------------------- |
| **Text**  | Машинният код на програмата                    | read-only, executable |
| **Data**  | Инициализирани глобални и `static` променливи  | read-write            |
| **BSS**   | Неинициализирани глобални (нулирани от ядрото) | read-write            |
| **Heap**  | Динамична памет (`malloc`)                     | расте нагоре          |
| **Stack** | Локални променливи, return адреси, аргументи   | расте надолу          |

```c
int global_init = 42;       // Data segment
int global_uninit;          // BSS segment (= 0 при старт)
static int s = 10;          // Data segment

int main(void) {
    int local = 5;          // Stack
    int *p = malloc(100);   // Heap (p е на stack, данните са на heap)
    return 0;
}
```

### Copy-on-Write (COW)

При `fork()` детето получава **копие** на адресното пространство на родителя. Буквалното копиране на всички данни би било бавно — затова ядрото използва **Copy-on-Write**:

- Веднага след `fork()`, родителят и детето **споделят физическите pages** в паметта (само четат)
- Когато **някой от двата процеса пише** в page, ядрото прави копие само на тази page
- Останалите (непроменени) pages продължават да се споделят

```
След fork():
  Родител        Дете
  ┌───────┐     ┌───────┐
  │ page A│────►│ page A│  ← споделена (COW)
  │ page B│────►│ page B│  ← споделена (COW)
  └───────┘     └───────┘

Родителят пише в page A:
  Родител        Дете
  ┌───────┐     ┌───────┐
  │page A'│     │ page A│  ← копие само на тази page
  │ page B│────►│ page B│  ← все още споделена
  └───────┘     └───────┘
```

---

<a id="fork"></a>

## `fork()` — създаване на нов процес

`fork()` е системното извикване, което създава нов процес. То е **единственият начин** да се роди нов процес в Linux (освен специалните случаи `clone()` и `vfork()`).

### Какво точно прави fork()

Когато извикаш `fork()`, ядрото прави **пълно копие на текущия процес**. Новият процес (детето) получава:

- Копие на цялото адресно пространство — код, данни, стек, heap (чрез COW)
- Копие на FD таблицата — всички отворени файлове се наследяват
- Копие на environment variables
- Копие на signal handlers и маски
- **Нов уникален PID** — единствената разлика от родителя
- PPID = PID на родителя

Изпълнението на двата процеса **продължава от същото място** — реда след `fork()`. Не се стартира нова програма, не се изпълнява `main()` отначало. И родителят, и детето продължават от следващата инструкция.

```
             pid_t pid = fork();   ← и двата процеса започват оттук нататък
                        │
             ┌──────────┴──────────┐
             │                     │
        РОДИТЕЛ                  ДЕТЕ
      pid = <число>            pid = 0
      (PID на детето)          (сигнал "аз съм детето")
```

Точно затова проверяваш върнатата стойност — тя е **единственият начин** да разбереш дали си в родителя или в детето:

```c
pid_t pid = fork();

if (pid == -1) {
    // fork() не успя — дете не е създадено
    // errno: EAGAIN (твърде много процеси) или ENOMEM (няма памет)
    err(1, "fork");
}

if (pid == 0) {
    // ── Тук сме в ДЕТЕТО ──
    // pid == 0 означава "fork ми върна 0, значим аз съм детето"
    // getpid() тук ще върне PID-а на детето (напр. 101)
    // getppid() ще върне PID-а на родителя (напр. 100)
    dprintf(STDOUT_FILENO, "Дете: PID=%d, PPID=%d\n", getpid(), getppid());
    _exit(0);
}

// ── Тук сме в РОДИТЕЛЯ ──
// pid > 0 и съдържа точно PID-а на детето
// getpid() тук ще върне оригиналния PID (напр. 100)
dprintf(STDOUT_FILENO, "Родител: PID=%d, детето е PID=%d\n", getpid(), pid);
```

### Какво се наследява и какво не

```
Наследява се (копира):             НЕ се наследява:
  ✓ адресно пространство (COW)       ✗ PID (детето получава нов)
  ✓ отворени FD                      ✗ pending сигнали
  ✓ environment variables            ✗ memory locks (mlock)
  ✓ current working directory        ✗ timers (alarm, setitimer)
  ✓ signal handlers                  ✗ UID/GID промени от setuid
  ✓ UID, GID, права
  ✓ nice стойност
```

### `pid_t` — типът за PID

`pid_t` е знаков целочислен тип (обикновено `int32_t`), дефиниран в `<unistd.h>`. Знаковият тип позволява `-1` за грешка и `0` за детето:

```c
pid_t pid    = fork();    // резултат от fork: 0, >0, или -1
pid_t my     = getpid();  // PID на текущия процес (винаги > 0)
pid_t parent = getppid(); // PID на родителя
```

### `_exit()` vs `exit()`

В дете след `fork()` се използва `_exit()` вместо `exit()`:

```c
// exit() преди завършване:
//   1. Изпълнява всички atexit() handler-и
//   2. Flush-ва stdio буфери (fflush)
//   3. Вика _exit()
// Проблемът: родителят и детето споделят stdio буфери — двоен flush!

// _exit() — директен syscall, без никакъв cleanup
//   → безопасно в дете след fork()
```

```c
if (pid == 0) {
    // ... работа на детето ...
    _exit(0);    // ← правилно в дете
}
```

### Пример — паралелна обработка

```c
#include <unistd.h>
#include <sys/wait.h>
#include <err.h>
#include <stdio.h>

#define NUM_WORKERS 4

int main(void) {
    for (int i = 0; i < NUM_WORKERS; i++) {
        pid_t pid = fork();
        if (pid == -1)
            err(1, "fork");

        if (pid == 0) {
            dprintf(STDOUT_FILENO, "Worker %d стартира (PID %d)\n", i, getpid());
            _exit(0);
        }
        // Родителят продължава цикъла и прави следващото дете
    }

    // Родителят чака всичките деца
    int status;
    pid_t finished;
    while ((finished = wait(&status)) > 0)
        dprintf(STDOUT_FILENO, "Worker PID %d завърши\n", finished);

    return 0;
}
```

---

<a id="getpid"></a>

## `getpid()` и `getppid()`

```c
#include <unistd.h>

pid_t getpid(void);   // PID на текущия процес
pid_t getppid(void);  // PID на родителя (Parent PID)
```

И двете са прости syscall-и без аргументи — никога не връщат грешка. `getpid()` е гарантирано > 0. `getppid()` връща 1 ако родителят е починал и процесът е осиновен от init.

```c
#include <unistd.h>
#include <stdio.h>

int main(void) {
    dprintf(STDOUT_FILENO, "Моят PID:      %d\n", getpid());
    dprintf(STDOUT_FILENO, "Родителят ми:  %d\n", getppid());

    pid_t pid = fork();
    if (pid == 0) {
        // В детето
        dprintf(STDOUT_FILENO, "Дете  — PID: %d, PPID: %d\n", getpid(), getppid());
        _exit(0);
    }
    // В родителя
    dprintf(STDOUT_FILENO, "Родител — PID: %d, дете е: %d\n", getpid(), pid);
    wait(NULL);
    return 0;
}
```

Типична употреба:

```c
// Логиране с PID — полезно при много паралелни процеси
dprintf(STDERR_FILENO, "[PID %d] Грешка: %s\n", getpid(), strerror(errno));

// Уникално имe на файл за всеки процес (избягва конфликти)
char tmpfile[64];
snprintf(tmpfile, sizeof(tmpfile), "/tmp/work.%d", getpid());

// Провери дали процес съществува (kill с сигнал 0 не изпраща нищо)
if (kill(some_pid, 0) == 0)
    dprintf(STDOUT_FILENO, "Процес %d съществува\n", some_pid);
```

---

<a id="exec"></a>

## `exec()` фамилията — замяна на програма

### Какво точно прави exec()

`exec()` **заменя текущия процес с нова програма**. Не създава нов процес — взима съществуващия процес и зарежда в него изцяло нов код и данни.

Когато извикаш `execvp("ls", args)`:

```
1. Ядрото намери изпълнимия файл "ls" в $PATH
2. Изчисти адресното пространство на текущия процес:
      - text segment (стария код) → презапиши с кода на ls
      - data/BSS segment          → презапиши с данните на ls
      - stack                     → нулирай, инициализирай отново
      - heap                      → изчисти
3. Запази (НЕ изчиства):
      - PID                       → остава същият!
      - PPID                      → остава същият!
      - Отворените FD             → наследяват се (освен O_CLOEXEC)
      - UID, GID
4. Задай новите argv и envp
5. Прехвърли изпълнение към entry point-а на ls
```

**`exec()` не се връща при успех.** Кодът след `execvp()` се изпълнява само при грешка:

```c
execvp("ls", args);
// Ако стигнем тук — exec() се е провалил
err(1, "execvp");   // ← изпълнява се САМО при грешка
```

```
Преди exec():                     След exec():
  bash (PID 500)                    ls (PID 500)
  [bash код]          exec("ls")    [ls код]
  [bash данни]      ──────────────► [ls данни]
  [bash стек]                       [ls стек]
  FD: 0,1,2                         FD: 0,1,2  ← наследени!
  PID = 500                         PID = 500  ← непроменен!
```

### Защо exec() не създава нов процес

Разделянето `fork()` + `exec()` е умишлено и мощно. Между двете извиквания детето може да:

- Пренасочи stdin/stdout/stderr преди новата програма да стартира
- Затвори ненужни FD
- Смени текущата директория (`chdir`)
- Промени UID/GID (`setuid`)

Всичко това се случва **преди** новата програма изобщо да стартира — без тя да знае нищо за това. Именно така shell-ът имплементира `>`, `<`, `|` и `sudo`.

### Фамилията exec — шест варианта

Всички са обвивки около `execve()` — единственото реално системно извикване. Имената следват конвенция от суфикси:

| Суфикс | Значение                                                            |
| ------ | ------------------------------------------------------------------- |
| `l`    | **l**ist — аргументите се подават като отделни параметри (variadic) |
| `v`    | **v**ector — аргументите се подават като масив (`char *argv[]`)     |
| `p`    | **p**ath — търси изпълнимия файл в `$PATH` (не е нужен пълен път)   |
| `e`    | **e**nvironment — подаваш собствен environment вместо да наследяваш |

```
execl   = list
execlp  = list + path
execle  = list + environment
execv   = vector
execvp  = vector + path
execve  = vector + environment  ← единственият реален syscall
```

### `execl` — list вариант

Аргументите се изреждат директно като параметри. Задължително завършва с `NULL`:

```c
#include <unistd.h>

int execl(const char *path, const char *arg0, ..., NULL);
// path → пълният път до изпълнимия файл
// arg0 → по конвенция: името на програмата (argv[0])
// ...  → останалите аргументи
// NULL → задължителен терминатор
```

```c
// Еквивалент на: /bin/ls -l /home
execl("/bin/ls", "ls", "-l", "/home", NULL);

// Еквивалент на: /bin/echo hello world
execl("/bin/echo", "echo", "hello", "world", NULL);

// При грешка — execl се върща (-1)
if (execl("/bin/ls", "ls", "-l", NULL) == -1)
    err(1, "execl");
```

### `execlp` — list + path

Идентично с `execl`, но **търси програмата в `$PATH`** — не е нужен пълен път. Удобен когато извикваш стандартни системни команди:

```c
int execlp(const char *file, const char *arg0, ..., NULL);
// file → само името на програмата (без пълен път)
// Търси в директориите от $PATH: /usr/bin, /bin, /usr/local/bin...
```

```c
execlp("ls", "ls", "-l", "/home", NULL);   // намира /bin/ls автоматично
execlp("gcc", "gcc", "-o", "out", "main.c", NULL);
execlp("python3", "python3", "script.py", NULL);

// Еквивалентни извиквания:
execl("/bin/ls",      "ls", "-l", NULL);   // пълен път — трябва да знаеш пътя
execlp("ls",          "ls", "-l", NULL);   // само името — PATH го открива
```

**Кога `execlp` пред `execl`:** почти винаги когато извикваш стандартни програми (`ls`, `grep`, `gcc`…). `execl` само когато имаш нужда от конкретна версия на програма на специфично място (`/usr/local/bin/python3` вместо системния `python3`).

### `execv` — vector вариант

Аргументите се подават като **масив от стрингове**, завършващ с `NULL`. По-удобен когато броят аргументи не е известен по време на компилация:

```c
int execv(const char *path, char *const argv[]);
// argv[0] → по конвенция: името на програмата
// argv[последен] → NULL
```

```c
char *args[] = { "ls", "-l", "-a", "/home", NULL };
execv("/bin/ls", args);

// Динамично изграждане на аргументи
char *cmd[] = { "grep", "-r", pattern, directory, NULL };
execv("/usr/bin/grep", cmd);
```

### `execvp` — vector + path

Комбинира `execv` с търсене в `$PATH`. Това е **най-използваният вариант** в практиката — аргументите са в масив (лесно се изгражда динамично) и не е нужен пълен път:

```c
int execvp(const char *file, char *const argv[]);
// file   → само името (търси се в PATH)
// argv   → масив от аргументи, argv[0] = името, последният = NULL
```

```c
// Фиксирани аргументи
char *args[] = { "gcc", "-Wall", "-o", "prog", "main.c", NULL };
execvp("gcc", args);

// Динамично изградени аргументи
char *argv[10];
int  argc = 0;
argv[argc++] = "grep";
argv[argc++] = "-r";
argv[argc++] = pattern;    // стойност от runtime
if (recursive)
    argv[argc++] = "-l";
argv[argc++] = directory;
argv[argc]   = NULL;       // задължителен терминатор
execvp("grep", argv);

// Предаване на argv директно от main()
// (main получава char *argv[] — точно подходящо за execvp)
int main(int argc, char *argv[]) {
    if (argc < 2)
        errx(1, "употреба: %s <команда> [аргументи...]", argv[0]);
    execvp(argv[1], &argv[1]);   // предай останалите аргументи
    err(1, "execvp: %s", argv[1]);
}
```

### `execve` — vector + environment (истинският syscall)

Единственото реално системно извикване. Подаваш собствен environment като масив от `"KEY=VALUE"` стрингове:

```c
int execve(const char *path, char *const argv[], char *const envp[]);
// envp → масив от "KEY=VALUE" стрингове, завършващ с NULL
```

```c
char *args[] = { "env", NULL };
char *env[]  = {
    "PATH=/usr/bin:/bin",
    "HOME=/tmp",
    "USER=nobody",
    NULL
};
execve("/usr/bin/env", args, env);
// Ще покаже само трите зададени environment variables
```

### `execle` — list + environment

```c
int execle(const char *path, const char *arg0, ..., NULL, char *const envp[]);
// envp е последният аргумент, след NULL терминатора
```

```c
char *env[] = { "PATH=/bin", "HOME=/tmp", NULL };
execle("/bin/sh", "sh", "-c", "echo $HOME", NULL, env);
```

### Обобщение на фамилията

```c
// Всичките шест — всяка е различна комбинация от суфикси:
execl  ("/bin/ls", "ls", "-l", NULL);
execlp ("ls",      "ls", "-l", NULL);
execle ("/bin/ls", "ls", "-l", NULL,    env);
execv  ("/bin/ls", argv);
execvp ("ls",      argv);
execve ("/bin/ls", argv, env);
```

> **Кое да изберем?** В повечето случаи: `execvp` (вектор + PATH) когато аргументите са динамични, `execlp` (лист + PATH) когато аргументите са известни предварително. `execve` само когато трябва пълен контрол над environment-а.

---

<a id="fork-exec"></a>

## `fork()` + `exec()` + `wait()` pattern

Това е **основният pattern за стартиране на нова програма** в Unix/Linux. Разбирането му е ключово — именно така работи всеки shell, всеки процес мениджър, и почти всяка програма, която стартира друга програма.

### Защо три стъпки, а не само exec()

Логичният въпрос е: защо не извикаш директно `exec()` вместо `fork()` + `exec()`? Защото `exec()` **унищожава текущия процес** — след успешен `exec()` оригиналната програма е изчезнала завинаги. Ако shell-ът извикаше `exec("ls")` директно, shell-ът щеше да се замени с `ls` и след края му нямаше да има shell.

```
ГРЕШЕН подход (без fork):
  bash  →  exec("ls")  →  ls работи  →  ls приключва  →  ?
                                                          (bash е изчезнал!)

ПРАВИЛЕН подход (fork + exec):
  bash  →  fork()  →  bash копие  →  exec("ls")  →  ls работи  →  ls приключва
        └─────────────────────────────────────────────────────────────────────►
                  bash чака с wait()                              bash продължава
```

### Стъпка по стъпка

```
Стъпка 1: fork()
═══════════════════════════════════════════════════════
  bash (PID 100)              КОПИЕ на bash (PID 101)
  pid = fork()          →     pid = 0
  pid == 101                  pid == 0  ← детето знае, че е дете
  → влиза в родителски       → влиза в детски клон
    клон на кода               на кода

Стъпка 2: exec() — само в детето
═══════════════════════════════════════════════════════
  bash (PID 100)              КОПИЕ на bash (PID 101)
  чака с wait()               execvp("ls", args)
                              ↓
                              ls (PID 101)
                              кодът на bash е заменен
                              с кода на ls
                              ls започва от main()

Стъпка 3: wait() — родителят чака
═══════════════════════════════════════════════════════
  bash (PID 100)              ls (PID 101)
  wait() блокира              ... работи ...
  |                           exit(0)
  └── wait() се събужда  ←────┘
  bash продължава нормално
```

### Минимален пример

```c
#include <unistd.h>
#include <sys/wait.h>
#include <err.h>

int main(void) {
    // Стъпка 1: fork() — създай копие
    pid_t pid = fork();
    if (pid == -1)
        err(1, "fork");

    if (pid == 0) {
        // ── ДЕТЕ ──
        // Стъпка 2: exec() — замени се с "ls -l"
        char *args[] = { "ls", "-l", NULL };
        execvp("ls", args);
        err(1, "execvp");   // стигаме тук само при грешка
    }

    // ── РОДИТЕЛ ──
    // Стъпка 3: wait() — изчакай детето
    int status;
    waitpid(pid, &status, 0);

    dprintf(STDOUT_FILENO, "ls завърши с код %d\n", WEXITSTATUS(status));
    return 0;
}
```

### Пренасочване на FD между fork() и exec()

Тъй като детето наследява FD таблицата от родителя, прозорецът **между `fork()` и `exec()`** е идеалното място да пренасочиш stdin/stdout/stderr — новата програма ще ги получи вече пренасочени, без да знае нищо за това.

Точно така shell-ът имплементира `ls > output.txt`:

```c
pid_t pid = fork();
if (pid == 0) {
    // ── ДЕТЕ: между fork() и exec() ──

    // Отвори файла за stdout
    int fd = open("output.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (fd == -1)
        err(1, "open");

    // dup2(src, dst) — копира fd върху dst, затваряйки dst ако е отворен
    // Резултат: FD 1 (stdout) вече сочи към output.txt
    if (dup2(fd, STDOUT_FILENO) == -1)
        err(1, "dup2");

    close(fd);   // оригиналният fd вече не е нужен (stdout е копиран)

    // exec() наследява новата FD таблица
    // ls ще пише в output.txt, мислейки, че пише в stdout
    execlp("ls", "ls", "-l", NULL);
    err(1, "execlp");
}

waitpid(pid, NULL, 0);
// output.txt вече съдържа изхода на ls
```

```
FD таблица преди fork():        FD таблица в детето след dup2():
  FD 0 → stdin (keyboard)         FD 0 → stdin (keyboard)
  FD 1 → stdout (terminal)        FD 1 → output.txt       ← сменено!
  FD 2 → stderr (terminal)        FD 2 → stderr (terminal)
  FD 3 → output.txt               FD 3 → output.txt
                                   (затваряме FD 3 след dup2)
```

### Пълен пример — функция `run()`

```c
#include <unistd.h>
#include <sys/wait.h>
#include <err.h>
#include <stdio.h>

// Стартира команда, чака я да завърши, връща exit кода
int run(char *argv[]) {
    pid_t pid = fork();
    if (pid == -1)
        err(1, "fork");

    if (pid == 0) {
        // Дете: замени се с командата
        execvp(argv[0], argv);
        // Ако exec() се върне — командата не е намерена (exit код 127 по конвенция)
        err(127, "%s", argv[0]);
    }

    // Родител: изчакай и върни exit кода
    int status;
    if (waitpid(pid, &status, 0) == -1)
        err(1, "waitpid");

    if (WIFEXITED(status))
        return WEXITSTATUS(status);

    if (WIFSIGNALED(status)) {
        dprintf(STDERR_FILENO, "%s: убит от сигнал %d\n",
            argv[0], WTERMSIG(status));
        return -1;
    }

    return -1;
}

int main(void) {
    char *ls[]  = { "ls", "-la", "/tmp", NULL };
    char *cat[] = { "cat", "/etc/hostname", NULL };
    char *gcc[] = { "gcc", "-Wall", "-o", "prog", "main.c", NULL };

    int code;

    code = run(ls);
    dprintf(STDOUT_FILENO, "ls завърши с код: %d\n\n", code);

    code = run(cat);
    dprintf(STDOUT_FILENO, "cat завърши с код: %d\n\n", code);

    code = run(gcc);
    if (code != 0)
        errx(code, "компилацията се провали с код %d", code);

    dprintf(STDOUT_FILENO, "Всичко успешно.\n");
    return 0;
}
```

### Как shell имплементира pipeline

`cmd1 | cmd2` е fork+exec+wait за **два процеса, свързани с pipe**:

```c
// ls | grep ".c"

int pipefd[2];
pipe(pipefd);   // pipefd[0] = четене, pipefd[1] = писане

// Първо дете: ls
pid_t pid1 = fork();
if (pid1 == 0) {
    dup2(pipefd[1], STDOUT_FILENO);  // stdout → pipe write end
    close(pipefd[0]);
    close(pipefd[1]);
    execlp("ls", "ls", NULL);
    err(1, "ls");
}

// Второ дете: grep
pid_t pid2 = fork();
if (pid2 == 0) {
    dup2(pipefd[0], STDIN_FILENO);   // stdin ← pipe read end
    close(pipefd[0]);
    close(pipefd[1]);
    execlp("grep", "grep", ".c", NULL);
    err(1, "grep");
}

// Родителят затваря pipe-а и чака и двете деца
close(pipefd[0]);
close(pipefd[1]);
waitpid(pid1, NULL, 0);
waitpid(pid2, NULL, 0);
```

### `WIFEXITED` и `WEXITSTATUS` — четене на exit кода

```c
int status;
waitpid(pid, &status, 0);

if (WIFEXITED(status)) {
    int code = WEXITSTATUS(status);
    dprintf(STDOUT_FILENO, "Exit код: %d\n", code);
}

if (WIFSIGNALED(status)) {
    dprintf(STDOUT_FILENO, "Убито от сигнал: %d\n", WTERMSIG(status));
}
```

---

<a id="wait"></a>

## `wait()` и `waitpid()`

Когато дете приключи с `exit()`, то не изчезва веднага — остава като **zombie** в таблицата на процесите, докато родителят не прочете exit кода му. `wait()` и `waitpid()` са системните извиквания, с които родителят прибира този exit код и позволява на ядрото да освободи ресурсите на детето.

### `wait()` — чака кое да е дете

```c
#include <sys/wait.h>

pid_t wait(int *status);
// Блокира докато НЯКОЕ дете не завърши
// status → указател към int, в който се записва информацията за изхода
//           (може да е NULL ако не те интересува)
// Връща: PID на завършилото дете, или -1 при грешка (ECHILD = няма деца)
```

```c
pid_t child = fork();
if (child == 0) {
    // дете
    _exit(42);
}

// Родителят чака
int status;
pid_t who = wait(&status);

dprintf(STDOUT_FILENO, "Завърши дете PID %d\n", who);

if (WIFEXITED(status))
    dprintf(STDOUT_FILENO, "Exit код: %d\n", WEXITSTATUS(status));  // 42
```

### `waitpid()` — чака конкретно дете

```c
pid_t waitpid(pid_t pid, int *status, int options);
// pid     → кое дете да чака (вижте таблицата по-долу)
// status  → като при wait()
// options → допълнителни флагове
// Връща: PID на завършилото дете, 0 (при WNOHANG), или -1 при грешка
```

**Стойности на `pid`:**

| Стойност | Значение                                  |
| -------- | ----------------------------------------- |
| `> 0`    | Чака конкретното дете с този PID          |
| `0`      | Чака кое да е дете в същата process group |
| `-1`     | Чака кое да е дете (като `wait()`)        |
| `< -1`   | Чака дете в process group `abs(pid)`      |

**Флагове за `options`:**

| Флаг         | Значение                                                |
| ------------ | ------------------------------------------------------- |
| `0`          | Блокира докато детето не завърши                        |
| `WNOHANG`    | Не блокира — ако детето не е завършило, връща 0 веднага |
| `WUNTRACED`  | Връща и при спряно (SIGSTOP) дете                       |
| `WCONTINUED` | Връща и при продължено (SIGCONT) дете                   |

```c
// Чакай конкретно дете
int status;
waitpid(child_pid, &status, 0);

// Провери без блокиране (polling)
pid_t result = waitpid(child_pid, &status, WNOHANG);
if (result == 0)
    dprintf(STDOUT_FILENO, "Детето все още работи\n");
else if (result == child_pid)
    dprintf(STDOUT_FILENO, "Детето завърши\n");

// Изчакай всичките деца едно по едно
while ((pid = waitpid(-1, &status, 0)) > 0) {
    dprintf(STDOUT_FILENO, "Дете %d завърши с код %d\n",
        pid, WEXITSTATUS(status));
}
```

### Макроси за четене на `status`

```c
WIFEXITED(status)    // true ако детето е завършило нормално (exit)
WEXITSTATUS(status)  // exit кодът (0–255), само ако WIFEXITED е true

WIFSIGNALED(status)  // true ако детето е убито от сигнал
WTERMSIG(status)     // номерът на сигнала, само ако WIFSIGNALED е true

WIFSTOPPED(status)   // true ако детето е спряно (SIGSTOP)
WSTOPSIG(status)     // номерът на сигнала, причинил спирането

WIFCONTINUED(status) // true ако детето е продължено (SIGCONT)
```

```c
// Пълна обработка на всички случаи
int status;
pid_t pid = waitpid(-1, &status, 0);

if (WIFEXITED(status)) {
    dprintf(STDOUT_FILENO, "PID %d: нормален изход, код %d\n",
        pid, WEXITSTATUS(status));
} else if (WIFSIGNALED(status)) {
    dprintf(STDOUT_FILENO, "PID %d: убит от сигнал %d (%s)\n",
        pid, WTERMSIG(status), strsignal(WTERMSIG(status)));
} else if (WIFSTOPPED(status)) {
    dprintf(STDOUT_FILENO, "PID %d: спрян от сигнал %d\n",
        pid, WSTOPSIG(status));
}
```

---

<a id="kill"></a>

## `kill()` — изпращане на сигнали

Въпреки името си, `kill()` не просто "убива" процеси — той **изпраща произволен сигнал** до процес или група процеси. Убиването е само един от вариантите.

### Сигнатура

```c
#include <signal.h>

int kill(pid_t pid, int sig);
// pid → кому да се изпрати сигналът (вижте таблицата)
// sig → номерът или константата на сигнала
// Връща: 0 при успех, -1 при грешка
```

**Стойности на `pid`:**

| Стойност | Изпраща до                                |
| -------- | ----------------------------------------- |
| `> 0`    | Конкретния процес с този PID              |
| `0`      | Всички процеси в същата process group     |
| `-1`     | Всички процеси (с права) освен init       |
| `< -1`   | Всички процеси в process group `abs(pid)` |

### Основните сигнали

| Сигнал    | Номер | Действие по подразбиране | Описание                                           |
| --------- | ----- | ------------------------ | -------------------------------------------------- |
| `SIGHUP`  | 1     | Terminate                | Терминалът е затворен / рестартирай конфигурацията |
| `SIGINT`  | 2     | Terminate                | Ctrl+C от клавиатурата                             |
| `SIGQUIT` | 3     | Core dump                | Ctrl+\\ — terminate с core dump                    |
| `SIGKILL` | 9     | Terminate                | **Не може да бъде прихванат или игнориран**        |
| `SIGUSR1` | 10    | Terminate                | Потребителски сигнал 1 (дефинирай сам)             |
| `SIGUSR2` | 12    | Terminate                | Потребителски сигнал 2 (дефинирай сам)             |
| `SIGPIPE` | 13    | Terminate                | Писане в затворен pipe                             |
| `SIGALRM` | 14    | Terminate                | Таймер (alarm()) е изтекъл                         |
| `SIGTERM` | 15    | Terminate                | Поискай учтиво спиране (може да се прихване)       |
| `SIGCHLD` | 17    | Ignore                   | Дете е завършило / спряно                          |
| `SIGSTOP` | 19    | Stop                     | Спри процеса (не може да се прихване)              |
| `SIGCONT` | 18    | Continue                 | Продължи спрян процес                              |
| `SIGTSTP` | 20    | Stop                     | Ctrl+Z от клавиатурата                             |

```c
#include <signal.h>
#include <unistd.h>
#include <err.h>

// Поискай учтиво спиране (процесът може да направи cleanup)
if (kill(pid, SIGTERM) == -1)
    err(1, "kill SIGTERM");

// Принудително спиране (не може да се прихване или игнорира)
if (kill(pid, SIGKILL) == -1)
    err(1, "kill SIGKILL");

// Спри процеса (като Ctrl+Z)
kill(pid, SIGSTOP);

// Продължи спрян процес
kill(pid, SIGCONT);

// Провери дали процес съществува (сигнал 0 не се изпраща реално)
if (kill(pid, 0) == 0)
    dprintf(STDOUT_FILENO, "Процес %d съществува\n", pid);
else if (errno == ESRCH)
    dprintf(STDOUT_FILENO, "Процес %d не съществува\n", pid);
else if (errno == EPERM)
    dprintf(STDOUT_FILENO, "Процес %d съществува, но нямаш права\n", pid);

// Изпрати сигнал на себе си
kill(getpid(), SIGUSR1);
```

### `SIGTERM` vs `SIGKILL`

```
SIGTERM (15):                    SIGKILL (9):
  → процесът го получава           → ядрото го убива директно
  → може да:                       → процесът НЯМА шанс да:
      - прибере ресурси               - запази данни
      - затвори файлове               - затвори файлове
      - логне изход                   - направи каквото и да е
      - игнорира сигнала!           → не може да се прихване

  Правилният ред: SIGTERM → изчакай → ако не спре → SIGKILL
```

```c
// Правилен pattern за спиране на процес
kill(pid, SIGTERM);

// Дай му 3 секунди да се справи сам
sleep(3);

// Провери дали все още е жив
if (kill(pid, 0) == 0) {
    // Все още работи — принудително
    kill(pid, SIGKILL);
}
```

---

<a id="sigaction"></a>

## `sigaction()` — прихващане на сигнали

`sigaction()` позволява да **дефинираш handler функция**, която да се извика когато процесът получи конкретен сигнал — вместо стандартното действие (terminate, core dump и т.н.).

### Защо `sigaction` вместо `signal()`

По-старият `signal()` работи, но поведението му е недефинирано в POSIX (различни системи се държат различно). `sigaction()` е надеждният, преносим начин.

### Структурата `sigaction`

```c
#include <signal.h>

struct sigaction {
    void     (*sa_handler)(int);   // прост handler: получава само номера на сигнала
    void     (*sa_sigaction)(int, siginfo_t *, void *);  // разширен handler
    sigset_t   sa_mask;            // сигнали, блокирани по време на handler-а
    int        sa_flags;           // флагове (SA_SIGINFO, SA_RESTART и др.)
};

int sigaction(int signum, const struct sigaction *act, struct sigaction *oldact);
// signum  → кой сигнал да прихванем
// act     → новото поведение
// oldact  → запазва старото поведение (може да е NULL)
// Връща: 0 при успех, -1 при грешка
```

### Основна употреба — прост handler

```c
#include <signal.h>
#include <unistd.h>
#include <stdio.h>

// Handler функцията — извиква се при SIGTERM
void handle_sigterm(int sig) {
    // ⚠️ В handler може да извикваш само async-signal-safe функции
    // write() е безопасна, printf() НЕ е
    write(STDOUT_FILENO, "Получих SIGTERM — приключвам...\n", 31);
    _exit(0);
}

int main(void) {
    struct sigaction sa = {0};
    sa.sa_handler = handle_sigterm;    // задай handler функцията
    sigemptyset(&sa.sa_mask);          // не блокирай допълнителни сигнали
    sa.sa_flags = 0;

    if (sigaction(SIGTERM, &sa, NULL) == -1)
        err(1, "sigaction");

    // Програмата продължава нормалната си работа
    // При SIGTERM → handle_sigterm() ще бъде извикана
    while (1) {
        dprintf(STDOUT_FILENO, "Работя... (PID %d)\n", getpid());
        sleep(1);
    }
    return 0;
}
```

### Игнориране на сигнал

```c
struct sigaction sa = {0};
sa.sa_handler = SIG_IGN;   // специална стойност: игнорирай сигнала
sigemptyset(&sa.sa_mask);
sigaction(SIGPIPE, &sa, NULL);
// Сега SIGPIPE (писане в затворен pipe) се игнорира вместо да убива процеса
```

### Възстановяване на стандартното поведение

```c
struct sigaction sa = {0};
sa.sa_handler = SIG_DFL;   // специална стойност: стандартното действие
sigemptyset(&sa.sa_mask);
sigaction(SIGTERM, &sa, NULL);
```

### Флагове `sa_flags`

| Флаг           | Значение                                                               |
| -------------- | ---------------------------------------------------------------------- |
| `SA_RESTART`   | Системните извиквания прекъснати от сигнала автоматично се рестартират |
| `SA_SIGINFO`   | Използвай `sa_sigaction` вместо `sa_handler` (повече информация)       |
| `SA_NODEFER`   | Не блокирай сигнала по време на handler-а                              |
| `SA_RESETHAND` | Възстанови стандартното действие след първото получаване               |

```c
// SA_RESTART — важен при long-running I/O
struct sigaction sa = {0};
sa.sa_handler = handle_sigterm;
sigemptyset(&sa.sa_mask);
sa.sa_flags = SA_RESTART;   // read()/write() ще се рестартират автоматично
sigaction(SIGTERM, &sa, NULL);
```

### Разширен handler с `SA_SIGINFO`

`sa_sigaction` получава допълнителна информация за сигнала — откой процес идва, причина:

```c
void handle_signal(int sig, siginfo_t *info, void *context) {
    dprintf(STDOUT_FILENO, "Сигнал %d от PID %d\n", sig, info->si_pid);
    // info->si_uid   → UID на изпращача
    // info->si_pid   → PID на изпращача
    // info->si_errno → errno стойност (ако е приложимо)
    // info->si_code  → причина (SI_USER = kill(), SI_KERNEL = ядрото)
}

struct sigaction sa = {0};
sa.sa_sigaction = handle_signal;
sigemptyset(&sa.sa_mask);
sa.sa_flags = SA_SIGINFO;   // ← задължително за sa_sigaction
sigaction(SIGUSR1, &sa, NULL);
```

> **`sig_atomic_t`** е специален тип, гарантиран за атомарен запис и четене от signal handler. Използвай го за споделени флагове между main() и handler-и — обикновените `int` и `bool` не са гарантирано безопасни.

---

<a id="context-switching"></a>

## Context Switching

### Какво е context switching

На един CPU ядро може да се изпълнява само един процес в даден момент. При много процеси ядрото ги **редува много бързо** — превключва между тях толкова бързо (стотици пъти в секунда), че изглежда, сякаш работят едновременно. Всяко такова превключване се нарича **context switch**.

### Какво е "context"

**Context** (контекст) на процеса е всичката информация, нужна на CPU да продължи изпълнението му от там, откъдето е спрял:

```
Context на процес:
  ┌─────────────────────────────────────┐
  │  Регистри на CPU:                   │
  │    rax, rbx, rcx, rdx, rsi, rdi    │
  │    rsp (stack pointer)              │
  │    rbp (base pointer)               │
  │    rip (instruction pointer)        │  ← кой ред от кода изпълняваме
  │    rflags (zero, carry, sign...)    │
  │                                     │
  │  Виртуална памет (page table base) │
  │  FD таблица                        │
  │  Signal mask                       │
  └─────────────────────────────────────┘
```

Когато ядрото превключи от процес A към процес B:

```
1. Запази контекста на A:
   Запиши всички CPU регистри в PCB (Process Control Block) на A

2. Зареди контекста на B:
   Презапиши регистрите от PCB на B
   Смени page table (виртуалното адресно пространство)

3. Продължи изпълнението на B от rip-а, записан в PCB-а му
```

### Кога се случва context switch

| Причина                | Описание                                                      |
| ---------------------- | ------------------------------------------------------------- |
| **Timer interrupt**    | Хардуерният таймер прекъсва на всеки ~1–10ms (квант от време) |
| **I/O блокиране**      | Процесът чака `read()`/`write()` → предай CPU на друг         |
| **`sleep()`**          | Процесът изрично се отказва от CPU за определено време        |
| **Системно извикване** | Преминаване user → kernel mode (не е пълен context switch)    |
| **По-висок приоритет** | По-приоритетен процес е готов за изпълнение (preemption)      |

### Scheduler — кой процес следва

**Scheduler-ът** е частта от ядрото, която решава кой процес получава CPU след context switch. Linux използва **CFS (Completely Fair Scheduler)** — опитва се да даде равно процесорно време на всички процеси, като взема предвид nice стойността.

```
Опростен цикъл на scheduler:

while (система работи) {
    избери следващия процес от run queue
    задай му квант от CPU време (~1-10ms)
    context switch → изпълни го
    когато квантът изтече (или процесът блокира):
        context switch обратно към ядрото
        → повтори
}
```

### Цената на context switch

Context switch не е безплатен — всяко превключване струва **микросекунди**:

- Запис и зареждане на регистри
- Смяна на page table → **TLB flush** (Translation Lookaside Buffer — кешът за адреси се изчиства)
- CPU кешовете ("топлите" данни на стария процес са безполезни за новия)

Затова прекалено много процеси или прекалено честите context switches **намаляват производителността** — CPU прекарва повече време в превключване, отколкото в реална работа. Можеш да видиш context switches с `vmstat`:

```bash
vmstat 1
# procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
#  r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
#  2  0      0 512000  10240 204800    0    0     0     0  450 1823  5  2 93  0  0
#                                                              └──── context switches/sec
```

### User mode vs Kernel mode

Когато процес извиква `fork()`, `exec()` или `open()`, той преминава от **user mode** в **kernel mode**:

```
User mode:                  Kernel mode:
  процесът изпълнява           ядрото изпълнява
  собствения си код            syscall handler-а
  (ограничени права)           (пълни права)

  syscall инструкция ──────►  изпълни заявката
                   ◄──────── върни резултата в rax
```

Превключването user→kernel е **по-евтино** от пълен context switch между два процеса — регистрите не се запазват напълно, само се сменя нивото на привилегии и stack pointer-ът.

### Пример — видимост на context switch

```c
#include <unistd.h>
#include <sys/wait.h>
#include <stdio.h>

int main(void) {
    for (int i = 0; i < 3; i++) {
        pid_t pid = fork();
        if (pid == 0) {
            // Всяко дете пише нещо — редът е недетерминиран
            // защото scheduler-ът решава кой получава CPU кога
            for (int j = 0; j < 5; j++) {
                dprintf(STDOUT_FILENO, "Дете %d: итерация %d\n", getpid(), j);
            }
            _exit(0);
        }
    }

    // Изчакай всичките деца
    while (wait(NULL) > 0)
        ;

    return 0;
}
// Изходът е различен при всяко изпълнение — context switch-овете не са предсказуеми
```

---

_Седмица 10 — Процеси, fork/exec и context switching в C_
