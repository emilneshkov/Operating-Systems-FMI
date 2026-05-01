# Седмица 9 — Грешки, файлови операции и структури

## Съдържание

1. [errno — как работят грешките в C](#errno)
2. [err() и errx() — репортване на грешки](#err-errx)
3. [lseek() — навигация в файл](#lseek)
4. [stat() и fstat() — метаданни на файл](#stat)
5. [lstat() — метаданни без следване на symlink](#lstat)
6. [struct — дефиниране на структури](#struct)
7. [Packed structs](#packed)

---

<a id="errno"></a>

## errno — как работят грешките в C

C няма изключения (exceptions) като Java или Python. Когато системна функция като `open()`, `read()` или `write()` се провали, тя не "хвърля" грешка — тя просто **връща `-1`** и записва причината за грешката в специална глобална променлива: **`errno`**.

### Какво е errno

`errno` е глобална целочислена променлива, дефинирана в `<errno.h>`. Всяка системна функция, която се проваля, записва в нея **номер на грешката** — цяло положително число, идентифициращо причината.

```c
#include <errno.h>    // дефинира errno и константите
#include <string.h>   // strerror()
#include <stdio.h>    // dprintf()
#include <fcntl.h>    // open()
#include <unistd.h>   // STDOUT_FILENO

int main(void) {
    int fd = open("nonexistent.txt", O_RDONLY);
    if (fd == -1) {
        // errno сега съдържа причината
        dprintf(STDOUT_FILENO, "errno = %d\n", errno);            // напр. 2
        dprintf(STDOUT_FILENO, "грешка: %s\n", strerror(errno));  // "No such file or directory"
    }
    return 0;
}
```

### Важни правила за errno

**`errno` е валиден само непосредствено след провалена функция.** Следващото системно извикване може да го презапише — дори ако то самото успее. Затова трябва да го прочетеш (или да го запазиш) преди да извикаш нещо друго:

```c
int fd = open("file.txt", O_RDONLY);
if (fd == -1) {
    int saved_errno = errno;        // запази ВЕДНАГА
    // ... тук може да правиш друго ...
    dprintf(STDOUT_FILENO, "грешка: %s\n", strerror(saved_errno));
}
```

**`errno` не се нулира автоматично.** Ако функцията успее, `errno` не се нулира до 0 — остава на стойността от последната грешка. Затова **никога не проверявай само `errno`** без да си проверил върнатата стойност:

```c
// ГРЕШНО — errno може да е от предишна грешка
open("file.txt", O_RDONLY);
if (errno != 0) { /* ненадежден */ }

// ПРАВИЛНО — първо провери върнатата стойност
int fd = open("file.txt", O_RDONLY);
if (fd == -1) { /* вече знаеш, че е грешка — сега погледни errno */ }
```

### Честите errno стойности

| Константа   | Номер | Значение                                    |
| ----------- | ----- | ------------------------------------------- |
| `EPERM`     | 1     | Операцията не е позволена                   |
| `ENOENT`    | 2     | Файлът или директорията не съществуват      |
| `ESRCH`     | 3     | Процесът не съществува                      |
| `EINTR`     | 4     | Системното извикване е прекъснато от сигнал |
| `EIO`       | 5     | I/O грешка                                  |
| `EBADF`     | 9     | Невалиден file descriptor                   |
| `ENOMEM`    | 12    | Недостатъчно памет                          |
| `EACCES`    | 13    | Отказан достъп (нямаш права)                |
| `EEXIST`    | 17    | Файлът вече съществува                      |
| `ENOTDIR`   | 20    | Не е директория                             |
| `EISDIR`    | 21    | Е директория (очаква се файл)               |
| `EINVAL`    | 22    | Невалиден аргумент                          |
| `EMFILE`    | 24    | Твърде много отворени файлове (за процеса)  |
| `ENOSPC`    | 28    | Няма свободно място на диска                |
| `EPIPE`     | 32    | Счупен pipe                                 |
| `ERANGE`    | 34    | Резултатът е извън диапазона                |
| `ENOTEMPTY` | 39    | Директорията не е празна                    |

```bash
# Виж всички errno стойности на системата
errno -l               # изисква: sudo apt install moreutils
```

### `strerror()` — четливо описание на грешката

```c
#include <string.h>

char *strerror(int errnum);
// Връща указател към статичен стринг с описанието на грешката
// НЕ трябва да го освобождаваш с free()
```

```c
dprintf(STDOUT_FILENO, "%s\n", strerror(ENOENT));   // "No such file or directory"
dprintf(STDOUT_FILENO, "%s\n", strerror(EACCES));   // "Permission denied"
dprintf(STDOUT_FILENO, "%s\n", strerror(EINVAL));   // "Invalid argument"
```

### `perror()` — бърз печат на грешка

`perror()` комбинира твоето съобщение с описанието на текущия `errno` и го отпечатва в **stderr**:

```c
#include <stdio.h>

void perror(const char *s);
// Отпечатва: "s: описание на errno\n" в stderr
```

```c
int fd = open("file.txt", O_RDONLY);
if (fd == -1) {
    perror("open");
    // Изход в stderr: "open: No such file or directory"
}
```

Форматът е винаги `"твоят_текст: системното_описание"`. Удобен за бързо дебъгване.

---

<a id="err-errx"></a>

## `err()` и `errx()` — репортване на грешки

`err()` и `errx()` са функции от `<err.h>` — BSD разширение, налично и в Linux. Те **отпечатват грешката и незабавно прекратяват програмата** с `exit()`. Много по-удобни от ръчната комбинация `perror()` + `exit()`.

```c
#include <err.h>

void err(int eval, const char *fmt, ...);
// eval → exit код (обикновено 1)
// fmt  → форматиращ стринг (като printf)
// Отпечатва: "program: fmt: strerror(errno)\n" и извиква exit(eval)

void errx(int eval, const char *fmt, ...);
// Същото, но БЕЗ да добавя strerror(errno) в края
// Използва се когато грешката не е свързана с errno
```

### Разликата между `err` и `errx`

```c
// err() — добавя автоматично strerror(errno)
int fd = open("file.txt", O_RDONLY);
if (fd == -1)
    err(1, "open: %s", "file.txt");
// Изход: "program: open: file.txt: No such file or directory"
//                                   └── добавено автоматично от errno

// errx() — само твоето съобщение, без errno
if (argc != 2)
    errx(1, "употреба: %s <файл>", argv[0]);
// Изход: "program: употреба: program <файл>"
// (грешката не е системна — errno не е релевантен)
```

### `warn()` и `warnx()` — като err/errx, но без exit

```c
#include <err.h>

void warn(const char *fmt, ...);   // като err(), но не спира програмата
void warnx(const char *fmt, ...);  // като errx(), но не спира програмата
```

```c
// Удобно когато искаш да логнеш грешка, но да продължиш
if (close(fd) == -1)
    warn("close");   // отпечатва предупреждение и продължава
```

### Пълен пример — правилна обработка на грешки

```c
#include <fcntl.h>
#include <unistd.h>
#include <err.h>
#include <stdint.h>
#include <stdio.h>

int main(int argc, char *argv[]) {
    if (argc != 2)
        errx(1, "употреба: %s <файл>", argv[0]);

    int fd = open(argv[1], O_RDONLY);
    if (fd == -1)
        err(1, "open: %s", argv[1]);

    char buf[512];
    ssize_t n;
    while ((n = read(fd, buf, sizeof(buf))) > 0) {
        if (write(STDOUT_FILENO, buf, n) != n)
            err(1, "write");
    }
    if (n == -1)
        err(1, "read");

    if (close(fd) == -1)
        err(1, "close");

    return 0;
}
```

> **Конвенция:** Exit код `0` означава успех, всяко друго число — грешка. `err(1, ...)` е стандартен начин да сигнализираш грешка от C програма към shell-а.

---

<a id="lseek"></a>

## `lseek()` — навигация в файл

### Какво е offset

Когато отвориш файл, ядрото поддържа **файлов офсет (file offset)** — цяло число, показващо на кое байтово място в файла се намираш в момента. Той започва от 0 при отваряне и автоматично се придвижва напред след всяко `read()` или `write()`.

```
Файл: [H][e][l][l][o][,][ ][W][o][r][l][d][!]
       0   1   2   3   4   5   6   7   8   9  10  11  12

След open():    offset = 0         (началото)
След read(5):   offset = 5         (прочетохме "Hello")
След read(7):   offset = 12        (прочетохме ", World")
След read():    offset = 13        (EOF)
```

`lseek()` ти позволява да **преместиш офсета** до произволна позиция — без да четеш или пишеш. Можеш да се върнеш назад, да скочиш напред, или да намериш края на файла.

### Сигнатура

```c
#include <unistd.h>

off_t lseek(int fd, off_t offset, int whence);
// fd     → file descriptor
// offset → отместване в байтове
// whence → от КЪДЕ се мери offset
//
// Връща: новата абсолютна позиция в байтове от началото
//        или -1 при грешка
```

### `whence` — точката на отместване

| Константа  | Значение                 | Пример                                    |
| ---------- | ------------------------ | ----------------------------------------- |
| `SEEK_SET` | от **началото** на файла | `lseek(fd, 0, SEEK_SET)` → начало         |
| `SEEK_CUR` | от **текущата** позиция  | `lseek(fd, 5, SEEK_CUR)` → напред 5 байта |
| `SEEK_END` | от **края** на файла     | `lseek(fd, 0, SEEK_END)` → краят          |

```c
// Върни се в началото на файла
lseek(fd, 0, SEEK_SET);

// Намери размера на файла (скочи в края, прочети позицията)
off_t size = lseek(fd, 0, SEEK_END);
dprintf(STDOUT_FILENO, "Размер: %ld байта\n", (long)size);

// Върни се в началото след намиране на размера
lseek(fd, 0, SEEK_SET);

// Прескочи напред 100 байта от текущата позиция
lseek(fd, 100, SEEK_CUR);

// Отиди 10 байта преди края
lseek(fd, -10, SEEK_END);

// Намери текущата позиция (без да се местиш)
off_t current = lseek(fd, 0, SEEK_CUR);
```

### Практически примери

```c
#include <fcntl.h>
#include <unistd.h>
#include <err.h>
#include <stdio.h>

// Намери размера на файл без stat()
off_t file_size(int fd) {
    off_t size = lseek(fd, 0, SEEK_END);
    if (size == -1)
        err(1, "lseek");
    lseek(fd, 0, SEEK_SET);   // върни се в началото
    return size;
}

// Прочети само последните N байта на файл
void read_last_n(int fd, int n) {
    lseek(fd, -n, SEEK_END);
    char buf[n];
    ssize_t bytes = read(fd, buf, n);
    write(STDOUT_FILENO, buf, bytes);
}

// Запиши на конкретна позиция (без да трием останалото)
void write_at(int fd, off_t pos, const char *data, size_t len) {
    lseek(fd, pos, SEEK_SET);
    write(fd, data, len);
}
```

### `off_t` — типът на офсета

`off_t` е целочислен тип, дефиниран в `<unistd.h>`. На 64-битови системи е `int64_t` (8 байта) — позволява файлове до 8 екзабайта. При печат използвай `(long)` cast или `%ld`:

```c
off_t pos = lseek(fd, 0, SEEK_END);
dprintf(STDOUT_FILENO, "Размер: %ld\n", (long)pos);
```

> **`lseek` не работи с всичко.** Pipes и сокети нямат офсет — `lseek()` върща `-1` с `errno = ESPIPE`. Работи само с "seekable" файлове: обикновени файлове и блокови устройства.

---

<a id="stat"></a>

## `stat()` и `fstat()` — метаданни на файл

`stat()` и `fstat()` извличат **метаданните на файл** (информацията, пазена в inode-а) — размер, права, собственик, timestamps, тип. Не четат съдържанието, само метаданните.

```c
#include <sys/stat.h>

int stat(const char *path, struct stat *buf);
// path → пътят до файла
// buf  → указател към struct stat, в която ядрото записва данните
// Връща: 0 при успех, -1 при грешка

int fstat(int fd, struct stat *buf);
// fd   → вече отворен file descriptor (вместо path)
// Връща: 0 при успех, -1 при грешка
```

**Разликата между `stat` и `fstat`:** `stat` приема **пътя** на файла и го търси в директорията. `fstat` приема **вече отворен FD** — по-ефективно, защото файлът вече е намерен. Ако файлът е отворен, предпочитай `fstat`. За symbolic links съществува `lstat()` — разгледано в следващата секция.

### `struct stat` — какво съдържа

Ядрото попълва тази структура с метаданните на файла:

```c
struct stat {
    dev_t     st_dev;     // ID на устройството, на което е файлът
    ino_t     st_ino;     // inode номер
    mode_t    st_mode;    // тип на файла + права за достъп
    nlink_t   st_nlink;   // брой hard links
    uid_t     st_uid;     // UID на собственика
    gid_t     st_gid;     // GID на групата
    off_t     st_size;    // размер в байтове (за обикновени файлове)
    blksize_t st_blksize; // препоръчителен I/O блок размер
    blkcnt_t  st_blocks;  // заети 512-байтови блока на диска
    time_t    st_atime;   // последен достъп (access time)
    time_t    st_mtime;   // последна промяна на съдържанието (modify time)
    time_t    st_ctime;   // последна промяна на метаданните (change time)
};
```

### Основна употреба

```c
#include <sys/stat.h>
#include <unistd.h>
#include <err.h>
#include <stdio.h>

int main(int argc, char *argv[]) {
    if (argc != 2)
        errx(1, "употреба: %s <файл>", argv[0]);

    struct stat st;
    if (stat(argv[1], &st) == -1)
        err(1, "stat: %s", argv[1]);

    dprintf(STDOUT_FILENO, "Файл:     %s\n",  argv[1]);
    dprintf(STDOUT_FILENO, "Размер:   %ld байта\n", (long)st.st_size);
    dprintf(STDOUT_FILENO, "inode:    %lu\n", (unsigned long)st.st_ino);
    dprintf(STDOUT_FILENO, "Links:    %lu\n", (unsigned long)st.st_nlink);
    dprintf(STDOUT_FILENO, "UID:      %u\n",  st.st_uid);
    dprintf(STDOUT_FILENO, "GID:      %u\n",  st.st_gid);
    dprintf(STDOUT_FILENO, "Блокове:  %ld\n", (long)st.st_blocks);

    return 0;
}
```

### `st_mode` — тип и права

`st_mode` е 16-битова стойност, в която са кодирани едновременно **типът на файла** и **правата за достъп**. Разделят се с битови маски:

```
st_mode:  0100644
           │└┘└┘└┘
           │  │  └── права: rw-r--r-- (644)
           │  └───── setuid/setgid/sticky bits
           └──────── тип: 010 = обикновен файл
```

**Макроси за тип на файла** (от `<sys/stat.h>`):

| Макрос        | Значение          |
| ------------- | ----------------- |
| `S_ISREG(m)`  | обикновен файл    |
| `S_ISDIR(m)`  | директория        |
| `S_ISLNK(m)`  | symbolic link     |
| `S_ISFIFO(m)` | named pipe (FIFO) |
| `S_ISSOCK(m)` | socket            |
| `S_ISBLK(m)`  | block device      |
| `S_ISCHR(m)`  | character device  |

**Битове за права** (комбинират се с `&`):

| Константа | Octal  | Значение      |
| --------- | ------ | ------------- |
| `S_IRUSR` | `0400` | user read     |
| `S_IWUSR` | `0200` | user write    |
| `S_IXUSR` | `0100` | user execute  |
| `S_IRGRP` | `0040` | group read    |
| `S_IWGRP` | `0020` | group write   |
| `S_IXGRP` | `0010` | group execute |
| `S_IROTH` | `0004` | other read    |
| `S_IWOTH` | `0002` | other write   |
| `S_IXOTH` | `0001` | other execute |

```c
struct stat st;
stat("file.txt", &st);

// Тип
if (S_ISREG(st.st_mode))
    dprintf(STDOUT_FILENO, "обикновен файл\n");
else if (S_ISDIR(st.st_mode))
    dprintf(STDOUT_FILENO, "директория\n");
else if (S_ISLNK(st.st_mode))
    dprintf(STDOUT_FILENO, "symbolic link\n");

// Права
if (st.st_mode & S_IRUSR)
    dprintf(STDOUT_FILENO, "собственикът може да чете\n");
if (st.st_mode & S_IXUSR)
    dprintf(STDOUT_FILENO, "собственикът може да изпълнява\n");

// Само правата (маскирай типа)
mode_t perms = st.st_mode & 0777;
dprintf(STDOUT_FILENO, "Права: %04o\n", perms);   // напр. 0644
```

### `fstat` — с вече отворен FD

```c
int fd = open("file.txt", O_RDONLY);
if (fd == -1)
    err(1, "open");

struct stat st;
if (fstat(fd, &st) == -1)
    err(1, "fstat");

dprintf(STDOUT_FILENO, "Размер: %ld\n", (long)st.st_size);

// Сега можем да четем точно толкова байта, колкото е файлът
char *buf = malloc(st.st_size);
read(fd, buf, st.st_size);
```

---

<a id="lstat"></a>

## `lstat()` — метаданни без следване на symlink

### Проблемът с `stat()` и symlinks

`stat()` **следва symbolic links** — когато подадеш пътя на symlink, тя прозрачно прескача и връща данните за **файла, към който сочи линкът**, не за самия линк. В повечето случаи това е желаното поведение, но понякога трябва да знаеш дали **самият path е symlink** и какви са неговите метаданни.

```
symlink "shortcut" → "/home/ivan/notes.txt"

stat("shortcut", &st)    → st описва notes.txt (следва линка)
lstat("shortcut", &st)   → st описва shortcut  (самия линк)
```

### Сигнатура

```c
#include <sys/stat.h>

int lstat(const char *path, struct stat *buf);
// path → пътят (може да е symlink)
// buf  → указател към struct stat
// Връща: 0 при успех, -1 при грешка
```

Сигнатурата е идентична с `stat()` — единствената разлика е поведението при symbolic links. Попълва същата `struct stat`.

### `stat` vs `lstat` vs `fstat` — кога кое

| Функция            | Аргумент     | При symlink                                 |
| ------------------ | ------------ | ------------------------------------------- |
| `stat(path, &st)`  | път (string) | следва линка → данни за целта               |
| `lstat(path, &st)` | път (string) | **не следва** → данни за линка              |
| `fstat(fd, &st)`   | отворен FD   | FD вече е отворен файл (без symlink въпрос) |

### Идентифициране на symlink с `lstat`

След `lstat()` полето `st_mode` ще покаже типа `S_IFLNK` ако пътят е symlink. Проверяваш с макроса `S_ISLNK()`:

```c
#include <sys/stat.h>
#include <unistd.h>
#include <err.h>
#include <stdio.h>

int main(int argc, char *argv[]) {
    if (argc != 2)
        errx(1, "употреба: %s <път>", argv[0]);

    struct stat st;
    if (lstat(argv[1], &st) == -1)
        err(1, "lstat: %s", argv[1]);

    if (S_ISLNK(st.st_mode)) {
        dprintf(STDOUT_FILENO, "%s е symbolic link\n", argv[1]);
        dprintf(STDOUT_FILENO, "Размер на линка: %ld байта\n", (long)st.st_size);
        // st_size на symlink = дължината на пътя, към който сочи
    } else if (S_ISREG(st.st_mode)) {
        dprintf(STDOUT_FILENO, "%s е обикновен файл (%ld байта)\n",
            argv[1], (long)st.st_size);
    } else if (S_ISDIR(st.st_mode)) {
        dprintf(STDOUT_FILENO, "%s е директория\n", argv[1]);
    }

    return 0;
}
```

### `readlink()` — накъде сочи symlink

След като установиш с `lstat` + `S_ISLNK` че пътят е symlink, можеш да прочетеш накъде сочи с `readlink()`:

```c
#include <unistd.h>

ssize_t readlink(const char *path, char *buf, size_t bufsiz);
// path   → пътят на symlink-а
// buf    → буфер, в който да се запише целевият път
// bufsiz → размерът на буфера
// Връща: брой байтове записани в buf (без \0), или -1 при грешка
// ⚠️  readlink НЕ добавя '\0' в края — трябва да го добавиш сам
```

```c
struct stat st;
lstat(argv[1], &st);

if (S_ISLNK(st.st_mode)) {
    // st.st_size е точно дължината на пътя → използваме го за буфера
    char *target = malloc(st.st_size + 1);  // +1 за '\0'
    ssize_t len = readlink(argv[1], target, st.st_size + 1);
    if (len == -1)
        err(1, "readlink");

    target[len] = '\0';  // задължително — readlink не го добавя
    dprintf(STDOUT_FILENO, "%s -> %s\n", argv[1], target);
    free(target);
}
```

---

<a id="struct"></a>

## `struct` — дефиниране на структури

### Какво е struct

В C можеш да групираш свързани данни от различни типове в **структура (struct)**. Структурата е съставен тип — тя описва как да наредиш в паметта набор от полета, всяко с различен тип и размер.

Вече видяхме структура в действие — `struct stat` е точно такава: ядрото ни връща цял набор от свързани данни в една структура вместо в десетина отделни параметъра.

### Дефиниране

```c
struct Point {
    int x;
    int y;
};

struct Person {
    char  name[64];
    int   age;
    float height;
};
```

Дефиницията описва **шаблон** — не заделя памет. Паметта се заделя когато декларираш **инстанция**:

```c
struct Point  p;           // заделя sizeof(struct Point) байта на стека
struct Person alice;       // заделя sizeof(struct Person) байта на стека
```

### Инициализация и достъп до полета

```c
// Инициализация при деклариране
struct Point p = { 10, 20 };           // по ред
struct Point q = { .x = 10, .y = 20 }; // по име (designated initializers)

// Достъп с . (dot operator)
p.x = 5;
p.y = 10;
dprintf(STDOUT_FILENO, "x=%d, y=%d\n", p.x, p.y);

// Достъп чрез указател с -> (arrow operator)
struct Point *ptr = &p;
ptr->x = 99;      // еквивалентно на (*ptr).x = 99
dprintf(STDOUT_FILENO, "x=%d\n", ptr->x);
```

### Размерът на struct в паметта

Структурата заема **поне сумата от размерите на полетата**, но компилаторът може да добавя **padding** (запълващи байтове) между полетата, за да ги подреди на адреси кратни на техния размер (alignment). Процесорът чете данните по-ефективно от изравнени адреси.

```c
struct Example {
    char  a;    // 1 байт на offset 0
                // [3 байта padding]
    int   b;    // 4 байта на offset 4  (int иска 4-байтово изравняване)
    char  c;    // 1 байт на offset 8
                // [3 байта padding]
};              // общо: 12 байта (не 6!)

dprintf(STDOUT_FILENO, "%zu\n", sizeof(struct Example));  // 12
```

Визуално в паметта:

```
Offset:  0    1    2    3    4    5    6    7    8    9   10   11
         [a] [pad][pad][pad][ b                  ][c] [pad][pad][pad]
```

Редът на полетата в дефиницията **влияе на размера**. Подреждането от по-голям към по-малък тип минимизира padding:

```c
// Лошо — много padding
struct Bad {
    char  a;    // 1 байт + 3 padding
    int   b;    // 4 байта
    char  c;    // 1 байт + 3 padding
};              // = 12 байта

// Добро — по-малко padding
struct Good {
    int   b;    // 4 байта
    char  a;    // 1 байт
    char  c;    // 1 байт + 2 padding
};              // = 8 байта
```

### `typedef` — по-кратко писане

С `typedef` можеш да дадеш псевдоним на типа и да спестиш `struct` при всяка употреба:

```c
typedef struct {
    int x;
    int y;
} Point;

// Вместо: struct Point p;
Point p = { 10, 20 };   // по-кратко
```

### Вложени структури

```c
typedef struct {
    float x;
    float y;
    float z;
} Vec3;

typedef struct {
    Vec3  position;
    Vec3  velocity;
    float mass;
} Particle;

Particle p;
p.position.x = 1.0f;
p.position.y = 2.0f;
p.velocity.z = -9.8f;
```

### Структури и функции

Структурите се предават по **стойност** (копират се) — при голяма структура е по-ефективно да подаваш указател:

```c
// По стойност — прави копие (неефективно за голяма структура)
void print_point(Point p) {
    dprintf(STDOUT_FILENO, "(%d, %d)\n", p.x, p.y);
}

// По указател — без копиране (предпочитано)
void print_point_ptr(const Point *p) {
    dprintf(STDOUT_FILENO, "(%d, %d)\n", p->x, p->y);
}

// Връщане на структура
Point make_point(int x, int y) {
    Point p = { x, y };
    return p;
}
```

---

<a id="packed"></a>

## Packed structs

### Проблемът с padding

Видяхме, че компилаторът добавя padding байтове за изравняване. Това е добре за производителност, но проблематично когато структурата трябва да **съответства точно на бинарен формат** — например file header, мрежов пакет, или хардуерен регистър.

Ако `struct stat` имаше padding, данните от ядрото щяха да попадат на грешните полета. Ядрото и libc гарантират, че системните структури са правилно дефинирани — но когато **ти дефинираш структура за бинарен протокол**, трябва да контролираш layout-а.

### `__attribute__((packed))`

GCC разширението `__attribute__((packed))` инструктира компилатора да **не добавя padding** — всяко поле следва директно след предишното:

```c
struct __attribute__((packed)) PackedExample {
    char  a;    // 1 байт на offset 0
    int   b;    // 4 байта на offset 1  (не е изравнен!)
    char  c;    // 1 байт на offset 5
};              // общо: 6 байта (не 12!)

dprintf(STDOUT_FILENO, "%zu\n", sizeof(struct PackedExample));  // 6
```

Визуално в паметта:

```
Offset:  0    1    2    3    4    5
         [a][ b                  ][c]
              └── неизравнен int (може да е по-бавен за четене)
```

### Кога се използва packed struct

**Бинарни файлови формати** — когато четеш или пишеш файл с фиксиран формат, структурата трябва да съответства байт по байт. Например ELF header:

```c
// ELF header — точно 64 байта, без padding
struct __attribute__((packed)) ElfHeader {
    uint8_t  magic[4];    // 0x7F 'E' 'L' 'F'
    uint8_t  class;       // 1=32bit, 2=64bit
    uint8_t  data;        // 1=little endian, 2=big endian
    uint8_t  version;
    uint8_t  padding[9];
    uint16_t type;        // 2=executable
    uint16_t machine;     // 0x3E=x86-64
    uint32_t elf_version;
    uint64_t entry;       // entry point адрес
};                        // = точно 32 байта за тази версия

// Прочети директно от файл в структурата
struct ElfHeader hdr;
read(fd, &hdr, sizeof(hdr));
dprintf(STDOUT_FILENO, "Entry point: 0x%lX\n", (unsigned long)hdr.entry);
```

**Мрежови пакети** — протоколните хедъри имат фиксиран размер:

```c
struct __attribute__((packed)) IpHeader {
    uint8_t  version_ihl;   // 4 бита версия + 4 бита header length
    uint8_t  tos;
    uint16_t total_length;
    uint16_t id;
    uint16_t flags_offset;
    uint8_t  ttl;
    uint8_t  protocol;
    uint16_t checksum;
    uint32_t src_ip;
    uint32_t dst_ip;
};  // = точно 20 байта
```

### Packed structs и endian

При работа с бинарни формати packed structs и endian проблемите вървят ръка за ръка — мрежовите протоколи използват big endian, докато x86 е little endian:

```c
struct __attribute__((packed)) DnsHeader {
    uint16_t id;           // в паметта: big endian (мрежов ред)
    uint16_t flags;
    uint16_t qdcount;
    uint16_t ancount;
    uint16_t nscount;
    uint16_t arcount;
};

struct DnsHeader hdr;
read(fd, &hdr, sizeof(hdr));

// Трябва да конвертираш от network byte order (big endian) към host (little endian)
uint16_t id = ntohs(hdr.id);
dprintf(STDOUT_FILENO, "DNS ID: %u\n", id);
```

### `offsetof` — намери offset на поле

Макросът `offsetof` от `<stddef.h>` връща байтовия офсет на поле в структура — полезно при дебъгване и при работа с бинарни формати:

```c
#include <stddef.h>
#include <stdio.h>

struct Example {
    char  a;
    int   b;
    char  c;
};

dprintf(STDOUT_FILENO, "offset of a: %zu\n", offsetof(struct Example, a));  // 0
dprintf(STDOUT_FILENO, "offset of b: %zu\n", offsetof(struct Example, b));  // 4 (с padding)
dprintf(STDOUT_FILENO, "offset of c: %zu\n", offsetof(struct Example, c));  // 8

// Размерите
dprintf(STDOUT_FILENO, "sizeof(struct Example): %zu\n", sizeof(struct Example));  // 12
```

---

_Седмица 9 — Грешки, файлови операции и структури_
