# Bash Команди — Седмица 7

## Съдържание

1. [fork, exec и раждането на процесите](#fork-exec)
2. [Сигнали към процеси — `kill` и `SIGTERM`](#signals)
3. [Background режим — `&`, `jobs`, `fg`, `bg`](#background)
4. [Parameter expansion — `${}`](#parameter-expansion)
5. [`eval` — динамично изпълнение](#eval)
6. [Архивиране с `tar`](#tar)

---

<a id="fork-exec"></a>

## fork, exec и раждането на процесите

В Linux **няма начин да създадеш процес от нищото**. Всеки нов процес се ражда от съществуващ чрез два системни извиквания: `fork()` и `exec()`.

### `fork()` — клониране на процес

`fork()` прави **точно копие** на текущия процес. Новият процес (дете) получава:

- собствен PID
- копие на адресното пространство на родителя (код, данни, стек, heap)
- копие на file descriptors (stdin, stdout, stderr и всички отворени файлове)
- копие на environment variables

Единствената разлика между родителя и детето след `fork()` е **върнатата стойност**:

```
fork() връща:
  → в родителя: PID на детето (число > 0)
  → в детето:   0
  → при грешка: -1
```

```bash
# Всяка команда, която изпълняваш в bash, минава през fork:
ls -la
# bash fork-ва себе си → детето exec-ва /bin/ls → bash чака с wait()
```

### `exec()` — заместване на програмата

`exec()` **заменя** образа на текущия процес с нова програма. Кодът, данните и стекът се презаписват изцяло с новия изпълним файл. PID-ът **не се променя**.

```
Преди exec():          След exec():
┌─────────────┐        ┌─────────────┐
│  bash copy  │  ───►  │     ls      │
│  PID: 1235  │        │  PID: 1235  │  ← същият PID!
│  код: bash  │        │  код: ls    │
└─────────────┘        └─────────────┘
```

`exec()` семейството включва няколко варианта: `execve`, `execl`, `execle`, `execv`, `execvp` — всички правят едно и също, но се различават по начина на подаване на аргументи.

### fork → exec — пълният цикъл

Всяка команда в bash минава през този двустъпков процес:

```
bash (PID 1234)
    │
    ├─ fork() ──────────────► bash copy (PID 1235)
    │                               │
    │                          exec("/bin/ls")
    │                               │
    │                          ls (PID 1235)  ← вече не е bash
    │                               │
    │                          изпълнява се...
    │                               │
    │◄── wait(1235) ────────────────┘  exit()
    │
    └─ bash продължава (пише prompt)
```

Стъпките:

1. bash извиква `fork()` — ражда се копие на себе си
2. детето извиква `exec("/bin/ls")` — заменя образа си с `ls`
3. `ls` се изпълнява (със същия PID като детето)
4. `ls` извиква `exit()` — завършва
5. bash извиква `wait()` — прочита exit кода и почиства zombie-то
6. bash пише нов prompt

### Защо двустъпковият модел?

Разделението `fork` + `exec` дава на shell-а прозорец между двете стъпки, в който може да **конфигурира средата** на детето преди то да стане новата програма:

```bash
# Shell прави това зад кулисите при всяка команда с пренасочване:
ls > output.txt

# Реалната последователност:
# 1. fork()
# 2. В детето: затвори stdout (fd 1)
# 3. В детето: отвори output.txt като fd 1
# 4. В детето: exec("/bin/ls")
# ls "мисли", че пише на stdout, но всъщност пише в output.txt
```

Същото важи за pipe-овете, променливите на средата, `ulimit` и много други.

### `exec` в bash скрипт

В bash `exec` може да се използва директно — заменя текущия shell процес:

```bash
exec python3 myscript.py
# bash процесът се замества с python3 — не се връща обратно в bash

exec > output.log 2>&1
# Пренасочва stdout и stderr на целия скрипт към файл
# без да стартира нов процес
```

### Copy-on-Write (CoW) — fork без копиране

`fork()` не копира физически паметта веднага — това би било бавно. Вместо това ядрото използва **Copy-on-Write**: родителят и детето споделят едни и същи физически memory pages. Копие се прави само при **запис** в конкретна page.

```
Преди запис:                    При запис от детето:
┌──────────┐                   ┌──────────┐   ┌──────────┐
│  Родител │──► page A (ro)    │  Родител │──► page A    │  Дете  │──► page A'
│  Дете    │──► page A (ro)    └──────────┘   └──────────┘   (ново копие)
└──────────┘
```

Затова `fork()` е бърз дори при процеси с гигабайти памет — ако детето веднага извика `exec()`, почти нищо не се копира физически.

---

<a id="signals"></a>

## Сигнали към процеси — `kill` и `SIGTERM`

**Сигналите** са механизмът на Linux за изпращане на асинхронни уведомления към процеси. Те са числа между 1 и 64, всяко с конкретно значение.

### Основни сигнали

| Номер | Име       | Значение                 | Поведение по подразбиране                |
| ----- | --------- | ------------------------ | ---------------------------------------- |
| `1`   | `SIGHUP`  | Терминалът е затворен    | Прекратяване (или reload при daemon-и)   |
| `2`   | `SIGINT`  | Interrupt — `Ctrl+C`     | Прекратяване                             |
| `3`   | `SIGQUIT` | Quit — `Ctrl+\`          | Прекратяване + core dump                 |
| `9`   | `SIGKILL` | Принудително убиване     | Прекратяване — **не може да се блокира** |
| `15`  | `SIGTERM` | Молба за прекратяване    | Прекратяване (може да се хване)          |
| `17`  | `SIGCHLD` | Дете е завършило         | Игнориране                               |
| `18`  | `SIGCONT` | Продължи спрян процес    | Продължаване                             |
| `19`  | `SIGSTOP` | Спри процеса             | Спиране — **не може да се блокира**      |
| `20`  | `SIGTSTP` | Terminal stop — `Ctrl+Z` | Спиране (може да се хване)               |

```bash
# Виж всички сигнали
kill -l
```

### `kill` — изпращане на сигнал

Въпреки името си, `kill` не само убива — то изпраща **произволен сигнал** към процес:

```bash
kill PID              # изпраща SIGTERM (15) по подразбиране
kill -15 PID          # изрично SIGTERM
kill -SIGTERM PID     # същото, по-четимо
kill -9 PID           # SIGKILL — принудително убиване
kill -SIGKILL PID     # същото

# Изпращане към множество процеси
kill PID1 PID2 PID3

# Изпращане към process group (всички процеси в групата)
kill -SIGTERM -PGID
```

### `SIGTERM` vs `SIGKILL` — разликата

|                                 | `SIGTERM` (15)                            | `SIGKILL` (9)       |
| ------------------------------- | ----------------------------------------- | ------------------- |
| **Може да се хване от процеса** | **Да**                                    | Не                  |
| **Може да се игнорира**         | **Да**                                    | Не                  |
| **Процесът може да се почисти** | **Да** (затваря файлове, flush-ва буфери) | Не                  |
| **Кой го изпълнява**            | Процесът сам                              | **Ядрото директно** |
| **Кога се ползва**              | Нормално спиране                          | Последна мярка      |

```bash
# Правилна последователност за спиране на процес:
kill -SIGTERM PID          # 1. Помоли учтиво
sleep 5                    # 2. Изчакай да се почисти
kill -0 PID 2>/dev/null && kill -SIGKILL PID   # 3. Ако все още живее — принуди
```

`kill -0 PID` не изпраща сигнал — само проверява дали процесът съществува (exit 0 = да, exit 1 = не).

### `pkill` и `killall` — по име

```bash
# По-удобно от намирането на PID
pkill nginx              # SIGTERM към всички процеси с това име
pkill -9 nginx           # SIGKILL към всички
pkill -u ivan            # SIGTERM към всички процеси на потребител ivan

killall nginx            # SIGTERM към точно съвпадащото име
killall -9 nginx

# Провери преди да убиеш
pgrep nginx              # показва PID-овете без да изпраща сигнал
pgrep -l nginx           # показва PID + ime
```

### `trap` — прихващане на сигнали в скрипт

```bash
#!/usr/bin/env bash

# Почисти при SIGTERM или SIGINT
cleanup() {
    echo "Получен сигнал — почиствам..."
    rm -f /tmp/myapp.lock
    exit 0
}

trap cleanup SIGTERM SIGINT

# Основна логика
echo $$ > /tmp/myapp.lock
while true; do
    echo "Работя..."
    sleep 1
done
```

```bash
# Изпрати SIGTERM на скрипта — cleanup() ще се изпълни
kill $(cat /tmp/myapp.lock)
```

---

<a id="background"></a>

## Background режим — `&`, `jobs`, `fg`, `bg`

В Linux всеки процес работи или на **преден план (foreground)** — блокира терминала, или на **заден план (background)** — терминалът остава свободен.

### `&` — стартиране в background

```bash
./script.sh &           # стартира в background
sleep 100 &             # [1] 4821 — показва job номер и PID
```

Bash отпечатва `[1] 4821` — `[1]` е **job номер** (локален за сесията), `4821` е PID.

### `jobs` — преглед на background процесите

```bash
jobs                    # всички jobs в текущата сесия
jobs -l                 # + показва PID-овете

# Примерен изход:
# [1]  Running    sleep 100 &
# [2]- Running    ./backup.sh &
# [3]+ Stopped    vim file.txt
```

| Символ       | Значение                                 |
| ------------ | ---------------------------------------- |
| `+`          | **Current job** — default за `fg` и `bg` |
| `-`          | **Previous job**                         |
| (без символ) | Останалите jobs                          |

### `fg` и `bg` — преместване между foreground и background

```bash
fg              # върни current job на foreground
fg %1           # върни job номер 1 на foreground
fg %sleep       # върни job по името на командата

bg              # продължи current job в background
bg %2           # продължи job номер 2 в background
```

### `Ctrl+Z` — спиране и изпращане в background

```bash
vim file.txt           # стартира vim на foreground
# Ctrl+Z               # спира vim (SIGTSTP), връща контрол на терминала
# [1]+  Stopped    vim file.txt

bg                     # продължи vim в background (рядко полезно за vim)
fg                     # върни vim на foreground
```

Типичен workflow:

```bash
vim file.txt
# Ctrl+Z — временно излез от vim
ls -la              # провери нещо
fg                  # върни се в vim
```

### `nohup` — процесът оцелява след затваряне на терминала

Когато затвориш терминала, shell-ът изпраща `SIGHUP` на всички background процеси — те умират. `nohup` ги предпазва:

```bash
nohup ./script.sh &
# Изходът се пренасочва към nohup.out автоматично

nohup ./script.sh > output.log 2>&1 &
# Изходът отива в output.log
```

### `disown` — откачане на вече стартиран процес

```bash
./script.sh &
disown %1           # откачи job 1 от сесията — няма да получи SIGHUP
disown -a           # откачи всички jobs
```

### Пълен пример — background workflow

```bash
# Стартирай дълга операция в background
tar -caf backup.tar.xz /home/ivan/ &
echo "Архивирането тече с PID $!"

# Провери статуса
jobs -l

# Ако искаш да следиш изхода
nohup tar -caf backup.tar.xz /home/ivan/ > tar.log 2>&1 &
tail -f tar.log         # следи лога в реално време

# Спри и продължи
kill -SIGSTOP $!        # спри backup процеса
kill -SIGCONT $!        # продължи backup процеса
```

---

<a id="parameter-expansion"></a>

## Parameter expansion — `${}`

**Parameter expansion** е механизмът на bash за работа с променливи. Базовото `$var` е само началото — `${}` отключва цял набор от операции директно в shell-а, без нужда от външни команди.

### Основен синтаксис

```bash
var="hello"

echo $var          # hello  — основен достъп
echo ${var}        # hello  — еквивалентно, но по-безопасно
echo "${var}"      # hello  — с кавички (препоръчително винаги)
```

Фигурните скоби стават задължителни при:

```bash
file="report"
echo "$filelog"     # грешка — bash търси $filelog
echo "${file}log"   # правилно — reportlog
```

### Стойности по подразбиране

```bash
# ${var:-default}  — ако var е празна или незададена, използвай default
echo "${name:-anonymous}"       # anonymous (ако $name е празна)

# ${var:=default}  — ако var е празна, задай й стойността и я върни
echo "${name:=anonymous}"       # задава $name="anonymous" и връща "anonymous"

# ${var:+value}    — ако var Е зададена, върни value (иначе празно)
echo "${name:+hello $name}"     # "hello ivan" ако $name е зададена

# ${var:?message}  — ако var е празна, печатай грешка и излез
echo "${input:?Грешка: не е подаден вход}"
```

Типична употреба в скриптове:

```bash
#!/usr/bin/env bash

# Вземи аргумент или използвай стойност по подразбиране
output="${1:-output.txt}"
mode="${2:-644}"

echo "Записвам в: $output с права $mode"
```

### Дължина на низ

```bash
str="Hello, World"

echo "${#str}"          # 12 — брой символи
echo "${#array[@]}"     # брой елементи в масив
```

### Изрязване на низ (substring)

```bash
str="Hello, World"

echo "${str:7}"         # World — от позиция 7 до края
echo "${str:7:5}"       # World — 5 символа от позиция 7
echo "${str: -5}"       # orld! — последните 5 символа (с интервал пред -)
```

### Премахване на prefix и suffix (pattern matching)

```bash
file="archive.tar.gz"

# Премахни най-краткото съвпадение от НАЧАЛОТО (#)
echo "${file#*.}"       # tar.gz  (премахва "archive.")

# Премахни най-дългото съвпадение от НАЧАЛОТО (##)
echo "${file##*.}"      # gz      (премахва "archive.tar.")

# Премахни най-краткото съвпадение от КРАЯ (%)
echo "${file%.*}"       # archive.tar  (премахва ".gz")

# Премахни най-дългото съвпадение от КРАЯ (%%)
echo "${file%%.*}"      # archive      (премахва ".tar.gz")
```

Практически пример — вземи basename и разширение:

```bash
path="/home/ivan/report.pdf"

filename="${path##*/}"      # report.pdf  (премахва пътя)
basename="${filename%.*}"   # report      (премахва разширението)
ext="${filename##*.}"       # pdf         (само разширението)

echo "$basename"    # report
echo "$ext"         # pdf
```

### Замяна в низ

```bash
str="hello world hello"

echo "${str/hello/hi}"      # hi world hello      — замени първото съвпадение
echo "${str//hello/hi}"     # hi world hi          — замени ВСИЧКИ съвпадения
echo "${str/#hello/hi}"     # hi world hello       — замени само ако е в началото
echo "${str/%hello/hi}"     # hello world hi       — замени само ако е в края
```

### Промяна на регистър

```bash
str="Hello World"

echo "${str,,}"     # hello world   — всичко малки
echo "${str^^}"     # HELLO WORLD   — всичко главни
echo "${str,}"      # hEllo World   — само първата буква малка
echo "${str^}"      # Hello World   — само първата буква главна
```

### Индиректна референция — `${!var}`

```bash
# ${!var} — стойността на променливата, чието ИМЕ е в $var
fruit="apple"
var="fruit"

echo "${!var}"      # apple  — bash чете $fruit

# Полезно за динамичен достъп до променливи
for name in HOME USER SHELL; do
    echo "$name = ${!name}"
done
# HOME = /home/ivan
# USER = ivan
# SHELL = /bin/bash
```

### Бърза справка

| Синтаксис | Описание |
| --- | --- |
| `${var}` | Стойността на `var` |
| `${#var}` | Дължина на низа |
| `${var:-default}` | Стойност или default ако е празна |
| `${var:=default}` | Задай default ако е празна |
| `${var:?msg}` | Грешка и изход ако е празна |
| `${var:offset}` | Substring от offset |
| `${var:offset:len}` | Substring с дължина |
| `${var#pattern}` | Премахни кратък prefix |
| `${var##pattern}` | Премахни дълъг prefix |
| `${var%pattern}` | Премахни кратък suffix |
| `${var%%pattern}` | Премахни дълъг suffix |
| `${var/old/new}` | Замени първо съвпадение |
| `${var//old/new}` | Замени всички съвпадения |
| `${var,,}` | Малки букви |
| `${var^^}` | Главни букви |
| `${!var}` | Индиректна референция |

---

<a id="eval"></a>

## `eval` — динамично изпълнение

`eval` приема низ и го **изпълнява като bash команда**. Bash парсира и изпълнява низа сякаш е написан директно в скрипта.

```bash
eval "команда"
eval $variable
```

### Как работи

```bash
cmd="ls -la"
eval "$cmd"         # изпълнява: ls -la

# Без eval:
echo "$cmd"         # просто печата: ls -la
$cmd                # работи за прости команди, но не за сложни
```

```bash
# eval минава през два кръга на парсиране:
# 1. bash разгъва $variable
# 2. bash изпълнява резултата като команда

var="World"
eval echo "Hello, \$var"    # Hello, World
#          └── \$ — escape-ван за втория кръг на парсиране
```

### Динамично изграждане на команди

```bash
# Динамично избиране на команда
action="create"
target="backup"

if [ "$action" = "create" ]; then
    cmd="tar -caf ${target}.tar.xz ."
elif [ "$action" = "extract" ]; then
    cmd="tar -xf ${target}.tar.xz"
fi

eval "$cmd"
```

### Динамично задаване на променливи

```bash
# Задай променлива с динамично ime
for i in 1 2 3; do
    eval "VAR_$i=$((i * 10))"
done

echo $VAR_1     # 10
echo $VAR_2     # 20
echo $VAR_3     # 30
```

### `eval` с `getopts` резултати

```bash
# Типична употреба — парсиране на сложни аргументи
options=$(getopt -o vf: -- "$@")
eval set -- "$options"
# eval set -- преподрежда $@ с правилно quote-нати аргументи
```

### ⚠️ Сигурност — никога с потребителски вход

`eval` е **опасен** когато низът идва от потребителя или от ненадеждни източници:

```bash
# ОПАСНО — никога така
read -p "Въведи команда: " user_input
eval "$user_input"
# потребителят може да въведе: rm -rf / или всяка друга команда
```

```bash
# Алтернативи на eval за безопасно динамично поведение:

# 1. Масиви вместо eval за команди
cmd=(tar -caf backup.tar.xz .)
"${cmd[@]}"         # безопасно изпълнение

# 2. ${!var} за индиректен достъп до променливи (вместо eval)
name="HOME"
echo "${!name}"     # /home/ivan — без eval

# 3. declare за динамично задаване
declare "VAR_$i=$value"    # по-безопасно от eval "VAR_$i=$value"
```

### Кога `eval` е оправдан

- Парсиране на изхода на `getopt`/`getops` — стандартна употреба
- Генериране на код в build системи или metaprogramming скриптове
- Когато входът е **изцяло под твой контрол** и е невъзможно да дойде от потребител

---

<a id="tar"></a>

## Архивиране с `tar`

`tar` (**t**ape **ar**chive) е стандартният инструмент за архивиране в Linux. Важно е да се разбере разликата между **архивиране** и **компресиране**:

- `tar` сам по себе си само **обединява** файлове в един архив (без компресия)
- Компресията се добавя чрез допълнителен алгоритъм — `gzip`, `bzip2`, `xz` и др.

### Синтаксис

```bash
tar -caf АРХИВ.tar.xz  ФАЙЛОВЕ...    # създаване
tar -xf  АРХИВ.tar.xz                # разархивиране
tar -tf  АРХИВ.tar.xz                # преглед на съдържанието
```

### Флагове

| Флаг        | Значение                                                      |
| ----------- | ------------------------------------------------------------- |
| `-c`        | **c**reate — създай нов архив                                 |
| `-x`        | e**x**tract — разархивирай                                    |
| `-t`        | lis**t** — покажи съдържанието                                |
| `-f`        | **f**ile — следва името на архива (задължително)              |
| `-a`        | **a**uto-compress — избери компресия по разширението на файла |
| `-v`        | **v**erbose — показвай файловете при обработка                |
| `-z`        | компресия с **gzip** (`.tar.gz` / `.tgz`)                     |
| `-j`        | компресия с **bzip2** (`.tar.bz2`)                            |
| `-J`        | компресия с **xz** (`.tar.xz`)                                |
| `-C`        | смени директорията преди операцията                           |
| `-p`        | запази permissions                                            |
| `--exclude` | изключи файлове/директории                                    |

### `-a` — автоматичен избор на компресия

Флагът `-a` (auto-compress) кара `tar` да **разпознае компресията по разширението** на архива — не е нужно да помниш `-z`, `-j` или `-J`:

```bash
tar -caf backup.tar.gz  folder/    # → gzip компресия
tar -caf backup.tar.bz2 folder/    # → bzip2 компресия
tar -caf backup.tar.xz  folder/    # → xz компресия
```

### Създаване на архив

```bash
# Основен синтаксис
tar -caf archive.tar.xz files_or_dirs/

# Примери
tar -caf backup.tar.xz /home/ivan/           # архивирай домашната директория
tar -caf project.tar.gz ./project/           # архивирай проект с gzip
tar -caf configs.tar.xz /etc/nginx/ /etc/ssh/ # множество директории

# С verbose — виж кои файлове се добавят
tar -cavf backup.tar.xz /home/ivan/

# Изключи определени файлове
tar -caf backup.tar.xz /home/ivan/ --exclude='*.log' --exclude='.cache'

# Архивирай само файлове по-нови от дата (инкрементален backup)
tar -caf backup.tar.xz /home/ivan/ --newer-mtime="2024-01-01"
```

### Разархивиране

```bash
# Основен синтаксис — в текущата директория
tar -xf archive.tar.xz

# С verbose — виж кои файлове се разархивират
tar -xvf archive.tar.xz

# В конкретна директория (-C)
tar -xf archive.tar.xz -C /tmp/restore/

# Разархивирай само конкретен файл от архива
tar -xf archive.tar.xz path/to/file.txt

# Разархивирай само конкретна директория
tar -xf archive.tar.xz home/ivan/documents/
```

### Преглед на съдържанието

```bash
# Покажи файловете без разархивиране
tar -tf archive.tar.xz

# С детайли (permissions, размер, дата)
tar -tvf archive.tar.xz

# Провери дали файл е в архива
tar -tf archive.tar.xz | grep "filename"
```

### Формати и разширения

| Разширение         | Компресия     | Скорост | Размер        |
| ------------------ | ------------- | ------- | ------------- |
| `.tar`             | Без компресия | ⚡⚡⚡  | Голям         |
| `.tar.gz` / `.tgz` | gzip          | ⚡⚡    | Среден        |
| `.tar.bz2`         | bzip2         | ⚡      | По-малък      |
| `.tar.xz`          | xz            | 🐢      | **Най-малък** |
| `.tar.zst`         | zstd          | ⚡⚡    | Малък         |

`xz` дава най-добра компресия, но е най-бавен. За скрипти и бекъпи `.tar.gz` е добрият баланс. За дистрибуции на software — `.tar.xz`.

### Практически примери

```bash
# Backup на домашната директория
tar -caf ~/backup-$(date +%Y-%m-%d).tar.xz ~/ --exclude='~/.cache'

# Разархивирай в нова директория
mkdir restore && tar -xf backup.tar.xz -C restore/

# Копирай директория с tar (запазва permissions и symlinks)
tar -caf - source/ | tar -xf - -C destination/

# Провери интегритета на архива (без разархивиране)
tar -tf archive.tar.xz > /dev/null && echo "OK" || echo "Повреден архив"

# Архивирай и изпрати по мрежата
tar -caf - /home/ivan/ | ssh user@server "cat > backup.tar.xz"

# Виж размера преди и след компресия
du -sh folder/                          # оригинален размер
tar -caf archive.tar.xz folder/
du -sh archive.tar.xz                  # компресиран размер
```

### `tar` в комбинация с background

```bash
# Архивирай в background и следи прогреса
tar -cavf backup.tar.xz /home/ivan/ > tar.log 2>&1 &
echo "Архивирането тече... PID: $!"

# Следи в реално време
tail -f tar.log

# Или с pv за прогрес бар (ако е инсталиран)
tar -caf - /home/ivan/ | pv | xz > backup.tar.xz &
```

---

_Седмица 7 — Процеси, сигнали и архивиране_