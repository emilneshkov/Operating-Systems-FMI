# Седмица 8 — Системно програмиране в C

## Съдържание

1. [Какво е C?](#какво-е-c)
2. [Как userland комуникира с ядрото — API и ABI](#api-abi)
3. [От C код до машинни инструкции](#компилатор)
4. [Syscall — системни извиквания](#syscall)
5. [Шестнадесетична бройна система и байтове](#hex)
6. [Little endian и big endian](#endian)
7. [Magic numbers](#magic-numbers)
8. [File Descriptors](#fd)
9. [open(), read(), write(), close()](#file-ops)
10. [Битови операции и shift](#битови-операции)

---

<a id="какво-е-c"></a>

## Какво е C?

C е компилируем, системен език за програмиране създаден от **Dennis Ritchie** в Bell Labs между 1969 и 1973 г. — почти едновременно с Unix, за чиято разработка е използван. Не е случайно: C е проектиран именно за да се пише операционна система.

За разлика от езиците с управлявана памет (Java, Python, Go), C **не скрива машината**. Ти сам управляваш паметта (`malloc`/`free`), ти избираш типовете данни и техния размер, ти решаваш кога и как да говориш с ядрото. Затова C е едновременно мощен и опасен.

```
Абстракция:   Python / Java / Go
              ───────────────────
              C / C++              ← близо до машината, но все още четимо
              ───────────────────
              Assembly             ← директни машинни инструкции
              ───────────────────
              Machine code / CPU
```

**Защо C ?**

- Ядрото на Linux е написано на C. Системните библиотеки са на C. Всеки език "отдолу" извиква C runtime.
- Syscall интерфейсът се извиква директно от C без посредник.

```c
#include <stdio.h>

int main(void) {
    printf("Hello, world!\n");
    return 0;
}
```

```bash
gcc hello.c -o hello    # компилирай
./hello                  # изпълни
```

---

<a id="api-abi"></a>

## Как userland комуникира с ядрото — API и ABI

Програмите, които пишеш (userland), работят в **потребителско пространство (user space)** — защитена среда, от която нямаш директен достъп до хардуера или ядрото. За да поискаш нещо от ядрото (отворен файл, мрежова връзка, памет), трябва да минеш през строго дефиниран интерфейс.

```
┌─────────────────────────────────────────┐
│           User Space                    │
│   програмата ти (C код, Python, Java)   │
│             │                           │
│         libc (glibc)                    │  ← стандартната C библиотека
│    open(), read(), malloc(), printf()   │  ← "обвивки" около syscall-и
│             │                           │
├─────────────┼───────────────────────────┤  ← граница user/kernel
│             ▼           Kernel Space    │
│          syscall                        │  ← единственият легален вход
│   sys_open, sys_read, sys_mmap...       │
│             │                           │
│          Hardware                       │
└─────────────────────────────────────────┘
```

### API vs ABI

**API (Application Programming Interface)** е договорът на ниво **изходен код** — какви функции съществуват, какви аргументи приемат, какво връщат. Когато пишеш `open(path, flags, mode)` в C, това е API.

**ABI (Application Binary Interface)** е договорът на ниво **машинен код** — как точно аргументите се предават (в кои регистри или на стека), как се кодират типовете данни, каква е наредбата на байтовете в структурите. ABI определя дали скомпилиран `.o` файл е съвместим с друга библиотека без прекомпилиране.

|                  | API                             | ABI                       |
| ---------------- | ------------------------------- | ------------------------- |
| Ниво             | Изходен код                     | Машинен код               |
| Пример           | `open(path, flags, mode)`       | аргументи в rdi, rsi, rdx |
| Промяна означава | прекомпилиране                  | преработване на всичко    |
| Стабилност       | може да се промени между версии | ядрото го гарантира       |

**Linux syscall ABI е умишлено стабилен** — програма, компилирана за Linux преди 20 години, продължава да работи на съвременно ядро. Linus Torvalds е известен с принципа "никога не чупим userspace".

---

<a id="компилатор"></a>

## От C код до машинни инструкции

Когато напишеш C код и го компилираш, минаваш през четири фази:

```
source.c
   │
   ▼  Preprocessor (cpp)     → разгъва #include, #define, #ifdef
preprocessed.c
   │
   ▼  Compiler (cc1)         → превежда C в assembly
source.s
   │
   ▼  Assembler (as)         → превежда assembly в машинен код
source.o  (object file)
   │
   ▼  Linker (ld)            → свързва .o файлове и библиотеки
executable
```

### C → Assembly → Machine code

Ето как прост C код се превръща в машинни инструкции:

```c
int add(int a, int b) {
    return a + b;
}
```

Компилаторът (GCC/Clang) превежда това до x86-64 assembly:

```asm
add:
    mov    eax, edi      ; eax = първи аргумент (a)
    add    eax, esi      ; eax = eax + втори аргумент (b)
    ret                  ; върни eax (return стойността)
```

Assembler-ът превежда assembly до машинен код — bytes:

```
8b 07        → mov eax, [rdi]
01 f0        → add eax, esi
c3           → ret
```

### Основни x86-64 инструкции

| Инструкция     | Значение                                                    |
| -------------- | ----------------------------------------------------------- |
| `mov dst, src` | копира стойност от src в dst                                |
| `add dst, src` | dst = dst + src                                             |
| `sub dst, src` | dst = dst - src                                             |
| `cmp a, b`     | изчислява a - b и задава флагове (без да запазва резултата) |
| `je  label`    | jump if equal — скача ако предишното `cmp` е дало 0         |
| `jne label`    | jump if not equal                                           |
| `jl  label`    | jump if less                                                |
| `jg  label`    | jump if greater                                             |
| `call label`   | извикване на функция                                        |
| `ret`          | връщане от функция                                          |
| `push reg`     | слага регистър на стека                                     |
| `pop  reg`     | взима от стека                                              |
| `syscall`      | системно извикване (специална инструкция)                   |

```c
// C:
if (a == b) {
    return 1;
}
return 0;
```

```asm
; Assembly:
    cmp  edi, esi       ; сравни a и b
    jne  .not_equal     ; ако не са равни → скочи
    mov  eax, 1         ; return 1
    ret
.not_equal:
    mov  eax, 0         ; return 0
    ret
```

```bash
# Виж генерирания assembly от GCC
gcc -O0 -S source.c -o source.s    # -S спира преди assembler
cat source.s

# Виж машинния код (disassembly)
objdump -d executable
```

---

<a id="syscall"></a>

## Syscall — системни извиквания

**Syscall** е единственият легален начин програма в user space да поиска услуга от ядрото. На x86-64 Linux, syscall се изпълнява с инструкцията `syscall`.

### Как работи syscall на ниско ниво

```
1. Програмата зарежда номера на syscall в регистъра rax
2. Зарежда аргументите в rdi, rsi, rdx, r10, r8, r9
3. Изпълнява инструкцията syscall
4. CPU превключва от user mode в kernel mode
5. Ядрото изпълнява функцията
6. Ядрото слага резултата в rax и се връща
7. CPU превключва обратно в user mode
```

```
User space регистри преди syscall:
  rax = номер на syscall
  rdi = 1-ви аргумент
  rsi = 2-ри аргумент
  rdx = 3-ти аргумент
  r10 = 4-ти аргумент
  r8  = 5-ти аргумент
  r9  = 6-ти аргумент

След syscall:
  rax = върната стойност (отрицателна = грешка)
```

### Таблица на syscall номерата (x86-64 Linux)

Всеки syscall е идентифициран с **номер** — фиксиран в ABI на ядрото. Таблицата се пази в `/usr/include/asm/unistd_64.h`.

| Номер | Syscall  | Описание                |
| ----- | -------- | ----------------------- |
| 0     | `read`   | чете от file descriptor |
| 1     | `write`  | пише в file descriptor  |
| 2     | `open`   | отваря файл             |
| 3     | `close`  | затваря file descriptor |
| 4     | `stat`   | информация за файл      |
| 9     | `mmap`   | маппира памет           |
| 11    | `munmap` | освобождава mmap памет  |
| 22    | `pipe`   | създава pipe            |
| 32    | `dup`    | дублира file descriptor |
| 39    | `getpid` | връща PID               |
| 57    | `fork`   | създава нов процес      |
| 59    | `execve` | изпълнява програма      |
| 60    | `exit`   | прекратява процес       |
| 62    | `kill`   | изпраща сигнал          |

```bash
# Виж всички syscall-и, извикани от програма
strace ./program
strace -e trace=open,read,write ./program    # само конкретни
strace -c ./program                          # статистика (брой, времe)
```

### Syscall в C vs директно в assembly

В C не извикваш `syscall` директно — използваш функциите на libc, които са wrapper-и:

```c
// Ти пишеш:
int fd = open("file.txt", O_RDONLY);

// libc прави:
// mov rax, 2          (syscall номер за open)
// mov rdi, "file.txt" (path)
// mov rsi, O_RDONLY   (flags)
// syscall
// → rax съдържа fd или отрицателна грешка
```

### Syscall в чист assembly (за разбиране)

```asm
section .data
    msg db "Hello, world!", 10    ; текст + newline

section .text
    global _start

_start:
    ; write(1, msg, 14)
    mov rax, 1          ; syscall: write
    mov rdi, 1          ; fd: stdout
    mov rsi, msg        ; buffer: адрес на текста
    mov rdx, 14         ; count: 14 байта
    syscall

    ; exit(0)
    mov rax, 60         ; syscall: exit
    mov rdi, 0          ; status: 0
    syscall
```

---

<a id="hex"></a>

## Шестнадесетична бройна система и байтове

### Защо hexadecimal?

Компютрите работят с двоична система — всичко са 0 и 1. Двоичният запис е труден за четене (`11111111` = 255). Шестнадесетичният (hex) е много по-удобен: **всеки hex цифра представя точно 4 бита** — един nibble.

```
Двоично:      1111 1111
Hex:           F    F    → 0xFF = 255
```

### Цифрите в hex

| Десетично | Двоично | Hex |
| --------- | ------- | --- |
| 0         | 0000    | 0   |
| 1         | 0001    | 1   |
| ...       | ...     | ... |
| 9         | 1001    | 9   |
| 10        | 1010    | A   |
| 11        | 1011    | B   |
| 12        | 1100    | C   |
| 13        | 1101    | D   |
| 14        | 1110    | E   |
| 15        | 1111    | F   |

```
0xFF   = 15*16 + 15 = 255
0x0F   = 15
0x10   = 16
0x100  = 256
0xFFFF = 65535
```

В C hex литералите се пишат с `0x` префикс: `0xFF`, `0x1A3F`.

### Байт — описване и размер

**Байтът** е фундаменталната единица памет — 8 бита. Един байт може да приема стойности от `0x00` до `0xFF` (0 до 255).

```
1 байт  =  8 бита  =  2 hex цифри
           ┌──┬──┬──┬──┬──┬──┬──┬──┐
           │b7│b6│b5│b4│b3│b2│b1│b0│
           └──┴──┴──┴──┴──┴──┴──┴──┘
           MSB  (most significant)   LSB (least significant)
```

### Типове данни и техния размер

| Тип в C          | Размер           | Минимум        | Максимум                   |
| ---------------- | ---------------- | -------------- | -------------------------- |
| `char`           | 1 байт           | -128           | 127                        |
| `unsigned char`  | 1 байт           | 0              | 255                        |
| `short`          | 2 байта          | -32,768        | 32,767                     |
| `unsigned short` | 2 байта          | 0              | 65,535                     |
| `int`            | 4 байта          | -2,147,483,648 | 2,147,483,647              |
| `unsigned int`   | 4 байта          | 0              | 4,294,967,295              |
| `long`           | 8 байта (64-bit) | -9.2 × 10¹⁸    | 9.2 × 10¹⁸                 |
| `uint8_t`        | 1 байт           | 0              | 255                        |
| `uint16_t`       | 2 байта          | 0              | 65,535                     |
| `uint32_t`       | 4 байта          | 0              | 4,294,967,295              |
| `uint64_t`       | 8 байта          | 0              | 18,446,744,073,709,551,615 |

`uint8_t`, `uint16_t` и т.н. са от `<stdint.h>` и гарантират точния размер на всяка платформа, за разлика от `int`, чийто размер зависи от архитектурата.

### `xxd` — преглед на файл в hex

`xxd` показва съдържанието на файл едновременно в hex и ASCII — изключително полезно за разбиране на бинарни файлове.

```bash
xxd file.bin
# 00000000: 7f45 4c46 0201 0100 0000 0000 0000 0000  .ELF............
# 00000010: 0200 3e00 0100 0000 4010 4000 0000 0000  ..>.....@.@.....
# └──────┘  └──────────────────────────────────────┘  └─────────────┘
# offset     hex байтове (16 на ред)                  ASCII (. = непечатаем)

xxd -l 32 file.bin          # само първите 32 байта
xxd -s 0x10 file.bin        # от offset 0x10 нататък
xxd -c 8 file.bin           # 8 байта на ред вместо 16
echo "Hello" | xxd          # hex на текст

# Обратно — hex към binary
xxd -r hex_dump.txt > output.bin
```

---

<a id="endian"></a>

## Little endian и Big endian

Когато число заема повече от 1 байт в паметта, има въпрос: в какъв ред се записват байтовете?

### Big endian — "естественият" ред

**Most significant byte** (най-значимият) е на **най-ниския адрес** — точно като при писане на числа.

```
Числото 0x12345678 в big endian памет:
Адрес:  0x00  0x01  0x02  0x03
Байт:   0x12  0x34  0x56  0x78
         MSB                LSB
```

### Little endian — обратният ред

**Least significant byte** е на **най-ниския адрес**. Изглежда "наобратно", но е по-ефективно за повечето CPU операции.

```
Числото 0x12345678 в little endian памет:
Адрес:  0x00  0x01  0x02  0x03
Байт:   0x78  0x56  0x34  0x12
         LSB                MSB
```

### Коя архитектура използва кое?

| Архитектура               | Endian                              |
| ------------------------- | ----------------------------------- |
| x86 / x86-64 (Intel, AMD) | **Little endian**                   |
| ARM                       | Little endian (по подразбиране)     |
| MIPS                      | Big endian                          |
| PowerPC                   | Big endian                          |
| Network (TCP/IP)          | **Big endian** (network byte order) |

**x86-64 е little endian** — затова когато гледаш памет с `xxd` на Intel/AMD, байтовете на многобайтовите числа са "наобратно".

```c
#include <stdio.h>
#include <stdint.h>

int main(void) {
    uint32_t n = 0x12345678;
    uint8_t *p = (uint8_t *)&n;

    // На little endian машина:
    printf("%02x\n", p[0]);    // 0x78  ← LSB е първи в паметта
    printf("%02x\n", p[1]);    // 0x56
    printf("%02x\n", p[2]);    // 0x34
    printf("%02x\n", p[3]);    // 0x12  ← MSB е последен
}
```

### Endian при мрежова комуникация

Когато пращаш числа по мрежата, трябва да конвертираш от host byte order (little endian на x86) в network byte order (big endian):

```c
#include <arpa/inet.h>

uint32_t htonl(uint32_t hostlong);    // host to network long
uint16_t htons(uint16_t hostshort);   // host to network short
uint32_t ntohl(uint32_t netlong);     // network to host long
uint16_t ntohs(uint16_t netshort);    // network to host short
```

---

<a id="magic-numbers"></a>

## Magic numbers

**Magic number** е специфична последователност от байтове в **началото на файл** (file header), по която можеш да идентифицираш формата — независимо от разширението на файла.

Операционната система и инструментите използват magic numbers, а не разширения, за да определят типа на файла. Затова `file` командата работи правилно дори ако преименуваш `image.png` на `document.txt`.

### Примери за magic numbers

| Формат                 | Hex байтове                   | ASCII      |
| ---------------------- | ----------------------------- | ---------- |
| ELF (Linux executable) | `7F 45 4C 46`                 | `.ELF`     |
| PNG                    | `89 50 4E 47 0D 0A 1A 0A`     | `.PNG....` |
| JPEG                   | `FF D8 FF`                    | `ÿØÿ`      |
| PDF                    | `25 50 44 46`                 | `%PDF`     |
| ZIP                    | `50 4B 03 04`                 | `PK..`     |
| GIF                    | `47 49 46 38`                 | `GIF8`     |
| Java `.class`          | `CA FE BA BE`                 | `ÊÞBA>`    |
| MP3                    | `49 44 33`                    | `ID3`      |
| SQLite                 | `53 51 4C 69 74 65`           | `SQLite`   |
| gzip                   | `1F 8B`                       | —          |
| tar                    | `75 73 74 61 72` (offset 257) | `ustar`    |

### Java magic number

Всеки компилиран Java `.class` файл започва с `0xCAFEBABE` — избран умишлено от James Gosling (създателят на Java) като "четимо" hex число. Следват два байта за minor version и два за major version:

```
CA FE BA BE  00 00  00 41
└──────────┘  └───┘  └───┘
magic number  minor  major (0x41 = 65 = Java 21)
```

```bash
# Провери magic number на файл
xxd file | head -1

# file командата използва magic number база
file image.png              # PNG image data, 1920 x 1080...
file /usr/bin/ls            # ELF 64-bit LSB executable...
file archive.zip            # Zip archive data
```

### ELF — Executable and Linkable Format

Изпълнимите файлове на Linux са в ELF формат. Header-ът съдържа:

```
Offset  Size  Значение
0x00    4     Magic: 7F 45 4C 46 (.ELF)
0x04    1     Class: 1=32bit, 2=64bit
0x05    1     Data: 1=little endian, 2=big endian
0x06    1     Version: 1
0x10    2     Type: 2=executable, 3=shared lib, 1=object
0x12    2     Machine: 0x3E=x86-64
0x18    8     Entry point address
```

```bash
readelf -h /usr/bin/ls      # четливо ELF header
xxd /usr/bin/ls | head -4   # raw bytes
```

---

<a id="fd"></a>

## File Descriptors (FD)

**File descriptor** е неотрицателно цяло число, което идентифицира отворен "ресурс" в контекста на процес. "Ресурсът" може да бъде файл, мрежова връзка, pipe, терминал — всичко, към което ядрото може да чете или пише.

### Таблицата на file descriptors

Всеки процес има своя собствена **fd таблица** — масив от указатели към ядрени структури, описващи отворени файлове. Индексът в масива е самият FD.

```
Процес PID 1234:
┌────────────────────────────────────────────┐
│  FD таблица                                │
│  ┌────┬──────────────────────────────────┐ │
│  │ 0  │ → stdin  (клавиатурата)          │ │
│  │ 1  │ → stdout (терминалът)            │ │
│  │ 2  │ → stderr (терминалът, за грешки) │ │
│  │ 3  │ → /home/ivan/notes.txt           │ │
│  │ 4  │ → socket (мрежова връзка)        │ │
│  │ 5  │ → (затворен)                     │ │
│  └────┴──────────────────────────────────┘ │
└────────────────────────────────────────────┘
```

### Стандартните file descriptors — 0, 1, 2

Всеки процес се ражда с три предварително отворени FD:

| FD  | Символно        | Посока | По подразбиране |
| --- | --------------- | ------ | --------------- |
| 0   | `STDIN_FILENO`  | четене | клавиатурата    |
| 1   | `STDOUT_FILENO` | писане | терминалът      |
| 2   | `STDERR_FILENO` | писане | терминалът      |

```c
#include <unistd.h>

// Дефинирани в unistd.h:
// STDIN_FILENO  = 0
// STDOUT_FILENO = 1
// STDERR_FILENO = 2

write(STDOUT_FILENO, "Hello\n", 6);   // пише в stdout
write(STDERR_FILENO, "Error\n", 6);   // пише в stderr
```

При **пренасочване** в shell:

```bash
./program > output.txt      # FD 1 се пренасочва към файла
./program 2> errors.txt     # FD 2 се пренасочва към файла
./program > out.txt 2>&1    # FD 2 → FD 1 → файла
./program < input.txt       # FD 0 се чете от файла
```

---

<a id="file-ops"></a>

## `open()`, `read()`, `write()`, `close()`

### `open()` — отваряне на файл

```c
#include <fcntl.h>

int open(const char *path, int flags, mode_t mode);
//                                    └── само при O_CREAT
// При успех → fd (неотрицателно цяло число)
// При грешка → -1 и errno се задава
```

### Флагове на `open()`

Флаговете се комбинират с битова операция `|` (OR):

**Задължителен режим на достъп — точно един от трите:**

| Флаг       | Стойност | Значение        |
| ---------- | -------- | --------------- |
| `O_RDONLY` | `0`      | само четене     |
| `O_WRONLY` | `1`      | само писане     |
| `O_RDWR`   | `2`      | четене и писане |

**Допълнителни флагове:**

| Флаг         | Значение                                                          |
| ------------ | ----------------------------------------------------------------- |
| `O_CREAT`    | Създай файла ако не съществува (изисква `mode`)                   |
| `O_TRUNC`    | Ако файлът съществува и се отваря за писане — изтрий съдържанието |
| `O_APPEND`   | Всяко писане добавя в края (не презаписва)                        |
| `O_EXCL`     | С `O_CREAT` — грешка ако файлът вече съществува (атомарно)        |
| `O_NONBLOCK` | Неблокиращ режим — ако операцията би блокирала, връща грешка      |
| `O_SYNC`     | Писането изчаква физически запис на диска                         |
| `O_CLOEXEC`  | Затвори FD автоматично при `exec()`                               |

```c
// Примери:
int fd;

// Отвори за четене
fd = open("notes.txt", O_RDONLY);

// Създай нов файл за писане (грешка ако съществува)
fd = open("new.txt", O_WRONLY | O_CREAT | O_EXCL, 0644);

// Отвори за писане, изтрий съдържанието, създай ако няма
fd = open("log.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);

// Добавяй в края
fd = open("log.txt", O_WRONLY | O_CREAT | O_APPEND, 0644);

if (fd == -1) {
    perror("open");     // отпечатва описанието на грешката
    return 1;
}
```

**`mode`** задава правата при създаване на нов файл (като `chmod`):

```c
0644    // rw-r--r--  (стандартен файл)
0755    // rwxr-xr-x  (изпълним файл)
0600    // rw-------  (личен файл)
```

### `read()` — четене

```c
#include <unistd.h>

ssize_t read(int fd, void *buf, size_t count);
// fd    → от кой file descriptor да чете
// buf   → адресът на буфера, в който да запише данните (&addr)
// count → максималният брой байтове за четене
//
// Връща: брой прочетени байтове
//         0 → край на файл (EOF)
//        -1 → грешка
```

```c
char buf[1024];
ssize_t n;

n = read(fd, buf, sizeof(buf));
if (n == -1) {
    perror("read");
}
if (n == 0) {
    // EOF — файлът е приключил
}

// buf е void* — ядрото записва директно в паметта на buf
// &addr означава "адресът на buf" — буфера, в памет
```

> **Важно:** `read()` **не гарантира** че ще прочете точно `count` байтове. Може да върне по-малко — при pipe, socket или край на файл. При нужда от точно N байта, трябва цикъл:

```c
ssize_t read_exactly(int fd, void *buf, size_t count) {
    size_t total = 0;
    while (total < count) {
        ssize_t n = read(fd, (char *)buf + total, count - total);
        if (n <= 0) return n;  // грешка или EOF
        total += n;
    }
    return total;
}
```

### `write()` — писане

```c
ssize_t write(int fd, const void *buf, size_t count);
// Връща: брой записани байтове или -1 при грешка
```

```c
const char *msg = "Hello, world!\n";
ssize_t n = write(STDOUT_FILENO, msg, 14);
if (n == -1) {
    perror("write");
}

// Пиши в файл
write(fd, &some_struct, sizeof(some_struct));  // записва struct като байтове
```

Като `read()`, и `write()` може да запише по-малко от поискания брой байтове (**partial write**) — особено при сокети. Добрата практика е да пишеш в цикъл.

### `close()` — затваряне

```c
int close(int fd);
// Връща: 0 при успех, -1 при грешка
```

`close()` освобождава FD и уведомява ядрото да приключи с ресурса. Незатворен FD е **изтичане на ресурс (resource leak)** — процесът изчерпва таблицата с FD (по подразбиране максимум ~1024 отворени FD на процес).

```c
close(fd);

// Добра практика — провери грешка дори при close:
if (close(fd) == -1) {
    perror("close");
}
```

### Пълен пример

```c
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>

int main(void) {
    // Отвори файл за четене
    int fd = open("input.txt", O_RDONLY);
    if (fd == -1) {
        perror("open");
        return 1;
    }

    // Прочети съдържанието
    char buf[256];
    ssize_t n;
    while ((n = read(fd, buf, sizeof(buf))) > 0) {
        // Запиши в stdout
        write(STDOUT_FILENO, buf, n);
    }
    if (n == -1) {
        perror("read");
    }

    // Затвори
    close(fd);
    return 0;
}
```

---

<a id="битови-операции"></a>

## Битови операции и shift

Битовите операции работят директно с **отделните битове** на числото. Използват се навсякъде в системното програмиране — флагове, маски, пакетиране на данни, оптимизации.

### Операторите

| Оператор | Название    | Пример             | Резултат    |
| -------- | ----------- | ------------------ | ----------- |
| `&`      | AND         | `0b1100 & 0b1010`  | `0b1000`    |
| `\|`     | OR          | `0b1100 \| 0b1010` | `0b1110`    |
| `^`      | XOR         | `0b1100 ^ 0b1010`  | `0b0110`    |
| `~`      | NOT         | `~0b1100`          | `0b...0011` |
| `<<`     | Left shift  | `1 << 3`           | `8`         |
| `>>`     | Right shift | `16 >> 2`          | `4`         |

### Shift — ляво и дясно

**Left shift `<<`** — мести битовете наляво, запълва с нули отдясно. Умножава по степен на 2:

```
1 << 0  =  00000001  =  1
1 << 1  =  00000010  =  2
1 << 2  =  00000100  =  4
1 << 3  =  00001000  =  8
1 << 4  =  00010000  =  16
n << k  =  n * 2^k
```

**Right shift `>>`** — мести битовете надясно. Дели по степен на 2 (целочислено):

```
16 >> 1  =  00001000  =  8
16 >> 2  =  00000100  =  4
16 >> 3  =  00000010  =  2
n  >> k  =  n / 2^k  (целочислено)
```

### Битови маски — флагове

Флаговете на `open()` са дефинирани именно с shift:

```c
// В <fcntl.h> (опростено):
#define O_RDONLY    0           // 000
#define O_WRONLY    1           // 001
#define O_RDWR      2           // 010
#define O_CREAT     (1 << 6)    // 01000000  = 0x40
#define O_TRUNC     (1 << 9)    // 1000000000 = 0x200
#define O_APPEND    (1 << 10)   // 10000000000 = 0x400
```

Комбинираш ги с `|` и проверяваш с `&`:

```c
int flags = O_WRONLY | O_CREAT | O_TRUNC;
// O_WRONLY = 000000001
// O_CREAT  = 001000000
// O_TRUNC  = 100000000
// OR       = 101000001

// Провери дали флаг е зададен:
if (flags & O_CREAT) {
    // O_CREAT е включен
}

// Изключи флаг:
flags = flags & ~O_TRUNC;
// ~O_TRUNC = ...011111111
// & премахва O_TRUNC бита

// Обърни флаг:
flags = flags ^ O_APPEND;
```

### Практически примери

```c
// Права на файл като bitfield (chmod)
// rwxr-xr-x = 0755
//   user:  rwx = 111 = 7
//   group: r-x = 101 = 5
//   other: r-x = 101 = 5

#define S_IRUSR (1 << 8)   // 0400 = user read
#define S_IWUSR (1 << 7)   // 0200 = user write
#define S_IXUSR (1 << 6)   // 0100 = user execute

// Провери дали файл е изпълним от собственика:
if (mode & S_IXUSR) { ... }

// Извлечи байт от uint32_t по позиция:
uint32_t ip = 0xC0A80101;   // 192.168.1.1
uint8_t octet3 = (ip >> 24) & 0xFF;   // 192
uint8_t octet2 = (ip >> 16) & 0xFF;   // 168
uint8_t octet1 = (ip >> 8)  & 0xFF;   // 1
uint8_t octet0 = (ip >> 0)  & 0xFF;   // 1
```

---

_Седмица 8 — Системно програмиране: syscall, памет и файлови операции_
