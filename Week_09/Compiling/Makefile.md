> **Забележка:** Този Makefile може директно да бъде копиран в твоя home directory в `astero` от следното място:
>
> ```
> /home/rado/28.04_02/2017_IN_01/Makefile
> ```

### Съдържание:

```c
#!/usr/bin/make -f

CC= gcc
CFLAGS_OPT?= -O2 -g -pipe
CFLAGS_STD?= -std=c11
CFLAGS_WARN?= -Wall -W -Wextra -Wpedantic

CFLAGS?= ${CFLAGS_OPT}
CFLAGS+= ${CFLAGS_STD} ${CFLAGS_WARN}

#CFLAGS+= -Werror
CFLAGS+= -Wbad-function-cast -Wcast-align -Wcast-qual -Wchar-subscripts \
 -Winline -Wmissing-prototypes -Wnested-externs -Wpointer-arith \
 -Wredundant-decls -Wshadow -Wstrict-prototypes -Wwrite-strings

CPPFLAGS_STD?= -D_POSIX_C_SOURCE=200809L -D_XOPEN_SOURCE=700
CPPFLAGS+= ${CPPFLAGS_STD}

RM?= rm -f

PROG= main
OBJS= main.o
SRCS= main.c

all:        ${PROG}

${PROG}:    ${OBJS}
            ${CC} ${LDFLAGS} -o ${PROG} ${OBJS}

${OBJS}:    ${SRCS}
            ${CC} ${CPPFLAGS} ${CFLAGS} -c -o ${OBJS} ${SRCS}

clean:
            ${RM} -- ${OBJS} ${PROG}

.PHONY: all clean
```
