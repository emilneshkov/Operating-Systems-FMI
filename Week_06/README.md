# Bash Команди — Седмица 6

## Съдържание

1. [Shebang — `#!/bin/bash`](#shebang)
2. [Как се стартира скрипт](#running)
3. [Променливи](#variables)
4. [Environment variables](#env-variables)
5. [Специални променливи — `$?`, `$#`, `$N`](#special-variables)
6. [Exit кодове](#exit-codes)
7. [Аргументи на скрипта](#arguments)
8. [if / else / fi](#if-else)
9. [for](#for)
10. [while](#while)
11. [Четене на редове — `while read`](#read-lines)
12. [Разлика между `for` и `while` с pipe и редирекция](#for-vs-while)
13. [mktemp — временни файлове](#mktemp)

---

<a id="shebang"></a>

## Shebang — `#!/bin/bash`

**Shebang** (от **sh**arp `#` + **bang** `!`) е специален коментар на първия ред на скрипта. Той казва на ядрото **кой интерпретатор** да използва за изпълнение на файла.

```bash
#!/bin/bash
```

### Как работи вътрешно

Когато изпълниш `./script.sh`, ядрото чете първите два байта. Ако са `#!`, то извиква посочения интерпретатор и му подава скрипта като аргумент:

```
./script.sh аргумент
      │
      ▼
ядрото чете: #!/bin/bash
      │
      ▼
/bin/bash ./script.sh аргумент
```

Без shebang ядрото не знае как да изпълни файла — shell-ът ще се опита да го интерпретира като команди на текущия shell, което може да доведе до неочаквано поведение.

### Видове shebang

| Shebang                  | Кога се ползва                                              |
| ------------------------ | ----------------------------------------------------------- |
| `#!/bin/bash`            | Bash скриптове — най-честата употреба                       |
| `#!/bin/sh`              | POSIX-съвместими скриптове (по-преносими, по-малко функции) |
| `#!/usr/bin/env bash`    | Търси `bash` в `$PATH` — по-преносимо между системи         |
| `#!/usr/bin/env python3` | Python скриптове                                            |
| `#!/usr/bin/env node`    | Node.js скриптове                                           |

```bash
#!/usr/bin/env bash
# По-добър избор за преносимост — работи дори ако bash е в /usr/local/bin
```

### Добри практики за началото на скрипта

```bash
#!/usr/bin/env bash
set -e          # спри при грешка (exit on error)
set -u          # грешка при неинициализирана променлива
set -o pipefail # pipe-ът връща грешка ако някоя команда в него се е провалила
set -x          # дебъг режим — печата всяка команда преди изпълнение
```

Или накратко: `set -euo pipefail` — стандартна "defensive" конфигурация за продукционни скриптове.

---

<a id="running"></a>

## Как се стартира скрипт

### Стъпки за стартиране

```bash
# 1. Създай файла
vim script.sh

# 2. Направи го изпълним
chmod +x script.sh

# 3. Изпълни го
./script.sh              # изрично — текущата директория
bash script.sh           # без chmod +x, bash го изпълнява директно
sh script.sh             # с POSIX sh интерпретатор
```

### Защо `./` е задължително

Bash не търси изпълними файлове в текущата директория (`./`) по подразбиране — само в директориите от `$PATH`. Затова `script.sh` се проваля, но `./script.sh` работи.

```bash
echo $PATH
# /usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin

# Ако искаш да стартираш без ./ — добави директорията в PATH (не препоръчително)
export PATH="$PATH:$(pwd)"
script.sh   # сега работи
```

### Source vs изпълнение

| Начин              | Нов процес?                           | Промените в среда важат ли за родителя? |
| ------------------ | ------------------------------------- | --------------------------------------- |
| `./script.sh`      | **Да** — fork + exec                  | Не                                      |
| `bash script.sh`   | **Да** — fork + exec                  | Не                                      |
| `source script.sh` | **Не** — изпълнява се в текущия shell | **Да**                                  |
| `. script.sh`      | **Не** — същото като `source`         | **Да**                                  |

```bash
# Пример — скрипт, който задава променлива
cat set_var.sh
# export MY_VAR="hello"

./set_var.sh
echo $MY_VAR   # празно — изпълни се в подпроцес

source set_var.sh
echo $MY_VAR   # hello — изпълни се в текущия shell
```

### Тестване с `bash -x` (дебъг)

```bash
bash -x script.sh        # печата всяка команда с + преди изпълнение
bash -n script.sh        # само проверява синтаксиса, не изпълнява
bash -v script.sh        # печата всеки ред при четене
```

---

<a id="variables"></a>

## Променливи

В bash променливите нямат тип — всичко е низ. Присвояването **не допуска интервали** около `=`.

```bash
name="ivan"          # низ
count=42             # число (също е низ за bash)
result=$(whoami)     # команден изход

echo $name           # ivan
echo ${name}         # ivan — фигурните скоби са по-безопасни
echo "${name}"       # ivan — кавичките предпазват от word splitting
```

### Кавички — важна разлика

| Синтаксис    | Интерполация               | Glob разширение         |
| ------------ | -------------------------- | ----------------------- |
| `"двойни"`   | **Да** — `$var` се разгъва | Не                      |
| `'единични'` | **Не** — буквален низ      | Не                      |
| без кавички  | Да                         | **Да** — `*` се разгъва |

```bash
name="ivan"
echo "Здравей, $name"    # Здравей, ivan
echo 'Здравей, $name'    # Здравей, $name  (буквално)
echo Здравей, $name      # Здравей, ivan (но опасно с интервали)
```

### Операции с низове

```bash
str="Hello, World"

echo ${#str}             # 13 — дължина
echo ${str:7}            # World — от позиция 7
echo ${str:7:5}          # World — 5 символа от позиция 7
echo ${str/World/Bash}   # Hello, Bash — замяна
echo ${str,,}            # hello, world — малки букви
echo ${str^^}            # HELLO, WORLD — главни букви
```

### Аритметика

```bash
a=5
b=3

echo $((a + b))          # 8
echo $((a * b))          # 15
echo $((a % b))          # 2
echo $((a ** b))         # 125

# Инкремент
((count++))
((count += 5))
```

### Масиви

```bash
fruits=("apple" "banana" "cherry")

echo ${fruits[0]}        # apple
echo ${fruits[@]}        # всички елементи
echo ${#fruits[@]}       # брой елементи
fruits+=("date")         # добавяне на елемент
```

<a id="env-variables"></a>

## Environment variables

**Environment variables** са именовани стойности, достъпни за всеки процес и неговите деца. За разлика от обикновените shell променливи, environment variables се **наследяват** от child процесите.

```
Shell
  │
  ├── обикновена променлива: name="ivan"     ← само в текущия shell
  └── environment variable:  export NAME="ivan" ← наследява се от децата
```

### `export` — превръщане в environment variable

```bash
name="ivan"          # обикновена shell променлива
export name          # вече е environment variable

export AGE=30        # директно деклариране и експортиране
```

### Преглед на environment variables

```bash
env                  # всички environment variables
printenv             # същото
printenv HOME        # стойността на конкретна променлива
echo $HOME           # достъп директно

# Подредено
env | sort
```

### Важни стандартни environment variables

| Променлива | Значение                                           |
| ---------- | -------------------------------------------------- |
| `$HOME`    | Домашната директория на потребителя (`/home/ivan`) |
| `$USER`    | Текущото потребителско име                         |
| `$PATH`    | Директориите, в които bash търси команди           |
| `$SHELL`   | Пътят до текущия shell (`/bin/bash`)               |
| `$PWD`     | Текущата директория                                |
| `$OLDPWD`  | Предишната директория (`cd -` ползва тази)         |
| `$LANG`    | Локалът на системата (`en_US.UTF-8`)               |
| `$EDITOR`  | Предпочитаният текстов редактор                    |
| `$TERM`    | Типът на терминала (`xterm-256color`)              |
| `$PS1`     | Prompt стрингът на bash                            |

### `$PATH` — как bash намира команди

```bash
echo $PATH
# /usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin

# Bash търси ls в:
# 1. /usr/local/bin/ls
# 2. /usr/bin/ls       ← намира го тук
# 3. /bin/ls
# ...

# Добавяне на директория в PATH
export PATH="$PATH:/opt/myapp/bin"

# За да е постоянно — добави в ~/.bashrc или ~/.bash_profile
echo 'export PATH="$PATH:/opt/myapp/bin"' >> ~/.bashrc
```

### Задаване на environment variable само за една команда

```bash
# Без export — важи само за тази команда
DEBUG=true ./script.sh
LANG=en_US.UTF-8 sort file.txt

# Проверка вътре в скрипта
echo $DEBUG     # true (само за тази команда)
```

### `unset` — изтриване на променлива

```bash
export MY_VAR="hello"
echo $MY_VAR    # hello

unset MY_VAR
echo $MY_VAR    # (празно)
```

### `.bashrc` vs `.bash_profile`

| Файл              | Кога се зарежда                                          |
| ----------------- | -------------------------------------------------------- |
| `~/.bash_profile` | При **login** shell (SSH вход, tty вход)                 |
| `~/.bashrc`       | При **interactive non-login** shell (нов терминал в GUI) |
| `~/.profile`      | При login shell — за всички POSIX shells                 |

```bash
# Добри практики
# ~/.bash_profile — зарежда .bashrc
if [ -f ~/.bashrc ]; then
    source ~/.bashrc
fi

# ~/.bashrc — дефинирай PATH и aliases тук
export PATH="$PATH:$HOME/.local/bin"
alias ll='ls -la'
```

---

<a id="special-variables"></a>

## Специални променливи — `$?`, `$#`, `$N`

Bash поддържа набор от вградени променливи, които се задават автоматично от shell-а:

### Позиционни параметри

| Променлива           | Значение                                                   |
| -------------------- | ---------------------------------------------------------- |
| `$0`                 | Името на скрипта (или shell-а)                             |
| `$1`, `$2`, ... `$N` | Аргументите, подадени на скрипта                           |
| `$@`                 | Всички аргументи като **отделни низове** — `"$1" "$2" ...` |
| `$*`                 | Всички аргументи като **един низ** — `"$1 $2 ..."`         |
| `$#`                 | **Броят** на аргументите                                   |

```bash
#!/usr/bin/env bash
# script.sh

echo "Скрипт: $0"
echo "Първи аргумент: $1"
echo "Втори аргумент: $2"
echo "Всички аргументи: $@"
echo "Брой аргументи: $#"
```

```bash
./script.sh foo bar baz
# Скрипт: ./script.sh
# Първи аргумент: foo
# Втори аргумент: bar
# Всички аргументи: foo bar baz
# Брой аргументи: 3
```

### Системни специални променливи

| Променлива | Значение                                        |
| ---------- | ----------------------------------------------- |
| `$?`       | Exit кодът на **последната** изпълнена команда  |
| `$$`       | PID на **текущия** shell/скрипт                 |
| `$!`       | PID на **последния фонов** процес (`command &`) |
| `$-`       | Текущите опции на shell-а (от `set`)            |
| `$_`       | Последният аргумент на предишната команда       |

```bash
ls /tmp
echo $?      # 0 — успех

ls /няма
echo $?      # 2 — грешка (файлът не съществува)

sleep 10 &
echo $!      # PID на sleep процеса

echo $$      # PID на текущия bash
```

### `$@` vs `$*` — важна разлика

```bash
#!/usr/bin/env bash
print_args() {
    echo "Брой аргументи: $#"
    for arg in "$@"; do echo "  -> $arg"; done
}

print_args "hello world" "foo"
# Брой аргументи: 2
#   -> hello world    ← "hello world" е ЕДИН аргумент
#   -> foo

# Ако използваш $* вместо $@:
for arg in "$*"; do echo "  -> $arg"; done
# -> hello world foo  ← всичко е ЕДИН низ
```

Правилото: **винаги ползвай `"$@"`** когато искаш да предаваш или итерираш аргументи.

---

<a id="exit-codes"></a>

## Exit кодове

Всяка команда в Linux завършва с **exit код** — число между 0 и 255, което показва дали е успяла.

| Exit код | Значение                                                |
| -------- | ------------------------------------------------------- |
| `0`      | **Успех**                                               |
| `1`      | Обща грешка                                             |
| `2`      | Неправилна употреба (грешен синтаксис)                  |
| `126`    | Файлът не е изпълним                                    |
| `127`    | Командата не е намерена                                 |
| `128+N`  | Прекратен от сигнал N (напр. `130` = `Ctrl+C` = SIGINT) |

```bash
# Провери exit кода веднага след команда
ls /tmp
echo $?     # 0

ls /няма
echo $?     # 2

# ВАЖНО: $? се презаписва след всяка команда
ls /няма
status=$?   # запази го веднага!
echo "Exit код: $status"
```

### `exit` в скриптове

```bash
#!/usr/bin/env bash

if [ ! -f "$1" ]; then
    echo "Грешка: файлът '$1' не съществува" >&2
    exit 1    # излез с код за грешка
fi

# ... обработка на файла ...

exit 0        # излез с успех (по избор — bash го прави автоматично)
```

### Верижни команди с `&&` и `||`

```bash
# && — изпълни втората само ако първата е успешна (exit 0)
mkdir /tmp/test && cd /tmp/test && echo "Готово"

# || — изпълни втората само ако първата е НЕУСПЕШНА (exit != 0)
cd /няма || echo "Директорията не съществува"

# Комбинация — типичен pattern за error handling
cp file.txt backup.txt || { echo "Копирането се провали" >&2; exit 1; }
```

---

<a id="arguments"></a>

## Аргументи на скрипта

### Достъп до аргументи

```bash
#!/usr/bin/env bash
# usage: ./script.sh <вход> <изход>

input="$1"
output="$2"

echo "Вход: $input"
echo "Изход: $output"
```

### Проверка на аргументи

```bash
#!/usr/bin/env bash

# Провери дали са подадени достатъчно аргументи
if [ $# -lt 2 ]; then
    echo "Употреба: $0 <вход> <изход>" >&2
    exit 1
fi

# Провери дали файлът съществува
if [ ! -f "$1" ]; then
    echo "Грешка: '$1' не е файл" >&2
    exit 1
fi
```

### `shift` — изместване на аргументите

`shift` премахва `$1` и измества всички останали аргументи наляво: `$2` става `$1`, `$3` става `$2` и т.н.

```bash
#!/usr/bin/env bash

while [ $# -gt 0 ]; do
    echo "Обработвам: $1"
    shift        # $2 → $1, $3 → $2, ...
done
```

### Парсиране на флагове с `getopts`

```bash
#!/usr/bin/env bash

verbose=false
output="default.txt"

while getopts "vo:" opt; do
    case $opt in
        v) verbose=true ;;
        o) output="$OPTARG" ;;
        *) echo "Непознат флаг" >&2; exit 1 ;;
    esac
done

shift $((OPTIND - 1))   # премахни флаговете, остави позиционните аргументи

$verbose && echo "Verbose режим включен"
echo "Изход: $output"
echo "Останали аргументи: $@"
```

```bash
./script.sh -v -o result.txt file1 file2
# Verbose режим включен
# Изход: result.txt
# Останали аргументи: file1 file2
```

---

<a id="if-else"></a>

## `if` / `else` / `fi`

```bash
if [ условие ]; then
    # команди при истина
elif [ друго условие ]; then
    # команди при друга истина
else
    # команди при лъжа
fi
```

`fi` е просто `if` наобратно — затваря блока. Всеки `if` **задължително** завършва с `fi`.

### Оператори за сравнение

**Числа:**

| Оператор | Значение                 |
| -------- | ------------------------ |
| `-eq`    | равно (equal)            |
| `-ne`    | различно (not equal)     |
| `-lt`    | по-малко (less than)     |
| `-le`    | по-малко или равно       |
| `-gt`    | по-голямо (greater than) |
| `-ge`    | по-голямо или равно      |

**Низове:**

| Оператор     | Значение                         |
| ------------ | -------------------------------- |
| `=` или `==` | равни низове                     |
| `!=`         | различни низове                  |
| `-z "$str"`  | низът е **празен** (zero length) |
| `-n "$str"`  | низът **не е** празен            |

**Файлове:**

| Оператор     | Значение                             |
| ------------ | ------------------------------------ |
| `-f "$file"` | съществува и е **файл**              |
| `-d "$dir"`  | съществува и е **директория**        |
| `-e "$path"` | **съществува** (файл или директория) |
| `-r "$file"` | съществува и е **четим**             |
| `-w "$file"` | съществува и е **записваем**         |
| `-x "$file"` | съществува и е **изпълним**          |
| `-s "$file"` | съществува и **не е празен**         |

```bash
# Числа
if [ $count -gt 10 ]; then
    echo "Повече от 10"
fi

# Низове
if [ "$name" = "ivan" ]; then
    echo "Здравей, ivan"
fi

if [ -z "$input" ]; then
    echo "Не е подаден вход" >&2
    exit 1
fi

# Файлове
if [ -f "/etc/passwd" ]; then
    echo "Файлът съществува"
fi

if [ ! -d "/tmp/mydir" ]; then
    mkdir /tmp/mydir
fi
```

### `[[ ]]` vs `[ ]` — разширени условия

`[[ ]]` е bash разширение на `[ ]` — по-мощно и по-безопасно:

```bash
# [[ ]] поддържа && и || вътре
if [[ $a -gt 5 && $b -lt 10 ]]; then
    echo "И двете условия са верни"
fi

# [[ ]] поддържа regex съвпадение с =~
if [[ "$email" =~ ^[a-z]+@[a-z]+\.[a-z]+$ ]]; then
    echo "Валиден имейл"
fi

# [[ ]] е по-безопасен с празни низове
if [[ -z $var ]]; then   # работи дори $var е недефиниран
    echo "Празно"
fi
```

### Кратък синтаксис

```bash
[ -f "$file" ] && echo "Файлът съществува"
[ -d "$dir" ]  || mkdir "$dir"
```

---

<a id="for"></a>

## `for`

`for` итерира през списък от стойности и изпълнява блока за всяка от тях.

```bash
for променлива in списък; do
    # команди
done
```

### Итерация през списък

```bash
for fruit in apple banana cherry; do
    echo "Плод: $fruit"
done
# Плод: apple
# Плод: banana
# Плод: cherry
```

### Итерация през файлове

```bash
# Всички .txt файлове в текущата директория
for file in *.txt; do
    echo "Обработвам: $file"
    wc -l "$file"
done

# Всички файлове рекурсивно
for file in $(find . -name "*.log"); do
    echo "$file"
done
```

### C-стил `for` — числов диапазон

```bash
for ((i = 0; i < 10; i++)); do
    echo "i = $i"
done

# С seq
for i in $(seq 1 5); do
    echo "i = $i"
done

# С brace expansion
for i in {1..10}; do
    echo "i = $i"
done

# С стъпка
for i in {0..20..5}; do   # 0, 5, 10, 15, 20
    echo "i = $i"
done
```

### Итерация през масив

```bash
fruits=("apple" "banana" "cherry")

for fruit in "${fruits[@]}"; do
    echo "$fruit"
done

# С индекс
for i in "${!fruits[@]}"; do
    echo "$i: ${fruits[$i]}"
done
```

### `break` и `continue`

```bash
for i in {1..10}; do
    [ $i -eq 5 ] && continue   # прескочи 5
    [ $i -eq 8 ] && break      # спри при 8
    echo $i
done
# 1 2 3 4 6 7
```

---

<a id="while"></a>

## `while`

`while` изпълнява блока **докато** условието е вярно.

```bash
while [ условие ]; do
    # команди
done
```

### Основна употреба

```bash
count=1
while [ $count -le 5 ]; do
    echo "count = $count"
    ((count++))
done
```

### Безкраен цикъл

```bash
while true; do
    echo "Работя..."
    sleep 1
done

# Прекъсва се с Ctrl+C или break
while true; do
    read -p "Команда (quit за изход): " cmd
    [ "$cmd" = "quit" ] && break
    eval "$cmd"
done
```

### `until` — обратното на `while`

`until` изпълнява докато условието е **лъжа** (т.е. докато не стане вярно):

```bash
until [ -f "/tmp/ready" ]; do
    echo "Чакам файла..."
    sleep 2
done
echo "Файлът се появи!"
```

---

<a id="read-lines"></a>

## Четене на редове — `while read`

Стандартният начин за четене ред по ред в bash е комбинацията `while read`:

```bash
while IFS= read -r line; do
    echo "$line"
done < file.txt
```

### Защо `IFS=` и `-r`?

| Опция  | Без нея                                        | С нея                                     |
| ------ | ---------------------------------------------- | ----------------------------------------- |
| `IFS=` | Водещите/крайните интервали се премахват       | Редът се запазва **буквално**             |
| `-r`   | `\n`, `\t` в реда се интерпретират като escape | Обратните наклонени черти са **буквални** |

Правилото: **`while IFS= read -r line`** е правилният шаблон за четене на произволни файлове.

### Четене от файл

```bash
# Основен шаблон
while IFS= read -r line; do
    echo "Ред: $line"
done < input.txt

# Обработка на CSV — разбива по ":"
while IFS=: read -r user pass uid gid info home shell; do
    echo "Потребител: $user, Shell: $shell"
done < /etc/passwd

# Прескачане на коментари и празни редове
while IFS= read -r line; do
    [[ "$line" =~ ^#  ]] && continue   # коментар
    [[ -z "$line"     ]] && continue   # празен ред
    echo "Обработвам: $line"
done < config.txt
```

### Четене от команда

```bash
# Pipe към while read
ps aux | while IFS= read -r line; do
    echo "$line"
done

# Process substitution (запазва средата на цикъла)
while IFS= read -r line; do
    echo "$line"
done < <(ps aux)
```

### Четене с `read` интерактивно

```bash
# Прочети един ред от потребителя
read -p "Въведи името си: " name
echo "Здравей, $name"

# С таймаут
read -t 10 -p "Имаш 10 секунди: " answer || echo "Времето изтече"

# Без echo (за пароли)
read -s -p "Парола: " password
echo ""   # нов ред след скритото въвеждане
```

---

<a id="for-vs-while"></a>

## Разлика между `for` и `while` с pipe и редирекция

Това е **една от най-важните тънкости в bash**.

### `for` — итерира през думи, не редове

```bash
# for LINE in $(cat $1) — итерира през думи, не редове!
for LINE in 1 2 3 pesho ivan gosho; do
    echo $LINE
done
# 1
# 2
# 3
# pesho
# ivan
# gosho
```

`for LINE in $(cat $1)` изглежда примамливо, но **разбива по whitespace**, не по редове — ако един ред съдържа интервали, той се третира като няколко елемента.

### `while` с pipe — изгубени променливи

```bash
COUNT=0
cat $1 | while read LINE; do
    echo "LINE $COUNT"
    echo $LINE
    COUNT=$(echo $COUNT + 1 | bc)
    # Алтернативно: COUNT=$((COUNT + 1))
done
# Проблем: COUNT остава 0 след цикъла!
```

Причината: **pipe стартира `while` в subshell**. Всички промени на `COUNT` и други променливи вътре в цикъла се губят при излизане от subshell-а.

```
cat $1  |  while read LINE; do ...   ← subshell (нов bash процес)
   │                │
   │         COUNT++, SOME_VAR="Result"   — работи тук
   │                │
   └── pipe ────────┘
                     COUNT и SOME_VAR се губят тук!

echo $SOME_VAR   # празно
```

### `while` с редирекция — правилният начин

```bash
# With Redirection
COUNT=0
while read LINE; do
    echo "LINE $COUNT"
    echo $LINE
    COUNT=$(echo $COUNT + 1 | bc)
    # Алтернативно: COUNT=$((COUNT + 1))
    SOME_VAR=Result
done < $1

echo $SOME_VAR   # Result — работи! Променливата е запазена
```

Редирекцията `< $1` **не създава subshell** — `while` блокът се изпълнява в текущия shell и всички промени на `COUNT` и `SOME_VAR` остават видими след `done`.

```
while read LINE; do   ← изпълнява се в ТЕКУЩИЯ shell
    COUNT++
    SOME_VAR=Result
done < $1             ← файлът се подава като stdin

echo $SOME_VAR        # Result ✓
```

### Пълно сравнение

|                                 | `for LINE in $(cat $1)` | `cat $1 \| while read` | `while read < $1` |
| ------------------------------- | ----------------------- | ---------------------- | ----------------- |
| **Разбива по**                  | whitespace (думи)       | редове                 | редове            |
| **Subshell**                    | Не                      | **Да**                 | Не                |
| **Променливите се запазват**    | Да                      | **Не**                 | **Да**            |
| **Работи с редове с интервали** | **Не**                  | Да                     | Да                |
| **Брояч работи след цикъла**    | Да                      | **Не**                 | **Да**            |

### Process substitution — алтернатива на pipe без subshell проблема

```bash
# Ако входът идва от команда, не от файл:
COUNT=0
while read LINE; do
    echo $LINE
    COUNT=$((COUNT + 1))
    SOME_VAR=Result
done < <(grep "ERROR" app.log)

echo $COUNT     # правилният брой ✓
echo $SOME_VAR  # Result ✓
```

`< <(команда)` изглежда странно, но е комбинация от редирекция (`<`) и process substitution (`<()`). `while` не е в subshell и вижда всички промени.

### Кога `for` vs `while`

```bash
# for — за списъци и диапазони, известни предварително
for LINE in 1 2 3 pesho ivan gosho; do
    echo $LINE
done

for i in {1..100}; do
    echo $i
done

for file in *.log; do
    gzip "$file"
done

# while read — за четене на файл ред по ред с редирекция
COUNT=0
while read LINE; do
    echo $LINE
    COUNT=$((COUNT + 1))
done < "$1"

echo "Общо редове: $COUNT"
```

---

<a id="mktemp"></a>

## `mktemp` — временни файлове

`mktemp` създава **уникален временен файл или директория** и отпечатва пътя му. За разлика от ръчното `/tmp/myfile`, `mktemp` гарантира уникалност и избягва race conditions.

```bash
tmpfile=$(mktemp)           # /tmp/tmp.XjK8mN2 (случайно суфикс)
tmpdir=$(mktemp -d)         # временна директория
```

### Синтаксис

```bash
mktemp                      # файл в /tmp с случайно име
mktemp -d                   # директория вместо файл
mktemp /tmp/myapp.XXXXXX    # собствен шаблон (минимум 3 X)
mktemp -t myapp.XXXXXX      # шаблон в системната tmp директория
mktemp --suffix=.log        # с конкретно разширение
```

`XXXXXX` — bash заменя X-овете с случайни символи. Трябват поне 3.

### Правилен pattern с `trap`

Временните файлове трябва да се почистват дори при грешка. `trap` изпълнява команда при излизане от скрипта:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Създай временен файл и го почисти при излизане
tmpfile=$(mktemp)
trap "rm -f $tmpfile" EXIT    # EXIT се изпълнява ВИНАГИ при край на скрипта

# Използвай временния файл
some_command > "$tmpfile"
process_results "$tmpfile"

# rm се извиква автоматично при EXIT — дори при грешка
```

```bash
# За директория
tmpdir=$(mktemp -d)
trap "rm -rf $tmpdir" EXIT

# Работи вътре в директорията
cd "$tmpdir"
# ...
```

### Практически примери

```bash
# Редактирай конфиг безопасно — само ако всичко е наред го приложи
tmpfile=$(mktemp)
trap "rm -f $tmpfile" EXIT

cp /etc/nginx/nginx.conf "$tmpfile"
vim "$tmpfile"

if nginx -t -c "$tmpfile"; then
    cp "$tmpfile" /etc/nginx/nginx.conf
    systemctl reload nginx
    echo "Конфигурацията е приложена"
else
    echo "Грешка в конфигурацията — промените НЕ са приложени" >&2
fi

# Сортирай голям файл без да го презаписваш директно
tmpfile=$(mktemp)
trap "rm -f $tmpfile" EXIT
sort input.txt > "$tmpfile" && mv "$tmpfile" input.txt

# Сравни изходи на две команди
tmp1=$(mktemp)
tmp2=$(mktemp)
trap "rm -f $tmp1 $tmp2" EXIT

command1 > "$tmp1"
command2 > "$tmp2"
diff "$tmp1" "$tmp2"
```

### Защо не `/tmp/myfile` директно?

```bash
# ОПАСНО — race condition и предсказуемо име
tmpfile="/tmp/myapp.tmp"
echo "данни" > $tmpfile     # атакуващ може да създаде symlink преди нас

# БЕЗОПАСНО — mktemp
tmpfile=$(mktemp)
echo "данни" > "$tmpfile"   # уникален файл, само ние знаем името му
```

---

_Седмица 6 — Shell Scripting_
