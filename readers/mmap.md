# mmap

## Идея

До этого мы работали с файлами примерно так:

```c
read(fd, buf, size);
/* работаем с buf */
write(fd, buf, size);
```

`mmap` предлагает другой способ: не копировать файл в отдельный буфер,
а добавить в виртуальную память процесса диапазон адресов, связанный с файлом.

После этого файл выглядит как массив байт:

```c
char *data = mmap(...);
printf("%c\n", data[0]);
data[10] = 'x';
```

Программа делает обычное обращение к памяти, а ядро само подгружает нужные
страницы файла.

`mmap` используют, когда нужно:
* читать большой файл как массив байт;
* быстро обращаться к данным по случайным смещениям;
* разделять память между процессами;
* выделять большие области памяти;
* загружать бинарники и shared libraries.

## Виртуальная память процесса

Процесс не работает с физическими адресами RAM напрямую.
Он работает с *виртуальными* адресами, а процессор и ядро переводят их
в физические.

Современная виртуальная память устроена постранично.
Память делится на страницы, обычно по 4096 байт:

    $ getconf PAGE_SIZE
    4096

Из C:

```c
#include <unistd.h>

long page_size = sysconf(_SC_PAGESIZE);
```

Ядро хранит не один большой массив памяти процесса, а набор отображений:
какой диапазон виртуальных адресов существует, какие у него права и откуда брать
страницы.

Такие отображения бывают:
* *файловые* — страницы берутся из файла;
* *анонимные* — страницы не связаны с файлом и изначально заполнены нулями.

## Page fault и ленивая загрузка

При `mmap` ядро обычно не читает файл сразу.
Оно только запоминает: этот диапазон виртуальной памяти связан с этим файлом.

Дальше всё происходит лениво:

1. Программа обращается к адресу внутри отображения.
2. Процессор не находит страницу в таблице страниц.
3. Возникает page fault.
4. Ядро смотрит, что это за диапазон памяти.
5. Ядро читает нужную страницу из файла или выделяет нулевую страницу.
6. Таблица страниц обновляется, инструкция выполняется заново.

Поэтому `mmap` хорошо сочетается с большими файлами: если потрогали только
несколько страниц, только они реально и понадобились.

## `/proc/<PID>/maps`

Посмотреть отображения процесса можно через `/proc`:

    $ cat /proc/self/maps

Пример строки:

```text
55d5f6e4a000-55d5f6e4b000 r--p 00000000 08:01 123456 /usr/bin/cat
```

Колонки:
* диапазон виртуальных адресов;
* права доступа: `r` — read, `w` — write, `x` — execute;
* `p` или `s` — private или shared;
* смещение внутри файла;
* устройство и inode;
* путь к файлу, если отображение файловое.

Минимальный эксперимент:

```c
#include <unistd.h>
#include <stdio.h>

int main(void) {
    printf("pid = %d\n", getpid());
    getchar();
}
```

Запускаем:

    $ gcc maps.c -o maps
    $ ./maps
    pid = 12345

Во втором терминале:

    $ cat /proc/12345/maps

Там будут отображения самого бинарника, libc, стека, кучи, vdso и других областей.
Код программы в памяти — это тоже результат отображения файла.

## Системный вызов `mmap`

```c
#include <sys/mman.h>

void *mmap(void *addr, size_t len, int prot, int flags, int fd, off_t offset);
```

`mmap` создаёт новое отображение виртуальной памяти.

Аргументы:
* `addr` — желаемый адрес; почти всегда передаём `NULL`;
* `len` — длина отображения в байтах;
* `prot` — права доступа к памяти;
* `flags` — тип отображения;
* `fd` — файловый дескриптор;
* `offset` — смещение в файле.

Возвращаемое значение:
* при успехе — адрес начала отображения;
* при ошибке — `MAP_FAILED`, а не `NULL`!!!

```c
char *p = mmap(NULL, len, PROT_READ, MAP_PRIVATE, fd, 0);
if (p == MAP_FAILED) {
    perror("mmap");
    return 1;
}
```

Документация: `man 2 mmap`, `man 2 munmap`, `man 2 msync`,
[POSIX mmap](https://pubs.opengroup.org/onlinepubs/009604499/functions/mmap.html).

## Права доступа: `prot`

`prot` описывает, что процессу разрешено делать с памятью:

* `PROT_READ` — читать;
* `PROT_WRITE` — писать;
* `PROT_EXEC` — исполнять как код;
* `PROT_NONE` — не обращаться.

Флаги комбинируются через `|`:

```c
PROT_READ | PROT_WRITE
```

Права отображения должны согласовываться с тем, как открыт файл.
Файловое отображение требует дескриптор, открытый на чтение.
Если хотим `MAP_SHARED | PROT_WRITE`, файл должен быть открыт на чтение и запись:

```c
int fd = open("data.bin", O_RDWR);
char *p = mmap(NULL, size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
```

## Тип отображения: `flags`

В `flags` обязательно выбирается один из двух основных режимов:

* `MAP_PRIVATE` — изменения остаются приватными для процесса;
* `MAP_SHARED` — изменения разделяются с другими процессами и могут попасть в файл.

`MAP_PRIVATE` работает через copy-on-write.
Пока процесс только читает страницу, её можно разделять с файлом и другими процессами.
При первой записи ядро создаёт приватную копию страницы.
Файл при этом не меняется.

`MAP_SHARED` означает, что запись в память относится к общему объекту.
Если отображение файловое, меняется содержимое файла.
Если несколько процессов отобразили один и тот же файл через `MAP_SHARED`,
они могут видеть изменения друг друга.

`MAP_SHARED` делает изменение общим, но не говорит, в какой именно момент байты
попадут на диск. Для явной синхронизации используют `msync`.

## Размеры и выравнивание

Ядро работает целыми страницами.

`len` можно передавать не кратным размеру страницы, но в `/proc/<PID>/maps`
отображение всё равно будет выглядеть округлённым до страниц.

`offset` обязан быть кратен размеру страницы:

```c
long page = sysconf(_SC_PAGESIZE);
off_t offset = 2 * page;  // хорошо
```

Если передать `offset = 13`, `mmap` вернёт `MAP_FAILED` и выставит `errno = EINVAL`.

`len = 0` тоже ошибка.

## `msync`

```c
#include <sys/mman.h>

int msync(void *addr, size_t len, int flags);
```

Основные флаги:
* `MS_SYNC` — дождаться записи;
* `MS_ASYNC` — запланировать запись и не ждать;
* `MS_INVALIDATE` — попросить сбросить другие кэшированные копии.

`msync` не делает операции атомарными.
Если несколько процессов пишут в одну `MAP_SHARED`-область, нужна отдельная
синхронизация: mutex, semaphore, file lock, futex или атомарные операции.

## Освобождение: `munmap`

```c
#include <sys/mman.h>

int munmap(void *addr, size_t len);
```

`munmap` удаляет отображение из виртуальной памяти процесса.
После этого обращаться к этому адресу нельзя.

```c
if (munmap(p, len) < 0) {
    perror("munmap");
}
```

После успешного `mmap` файловый дескриптор можно закрыть:

```c
char *p = mmap(NULL, size, PROT_READ, MAP_PRIVATE, fd, 0);
close(fd);
```

Отображение не исчезнет.
Ядро держит нужную ссылку на файл, пока отображение существует.

## Сценарий 1: читаем файл через `mmap`

Это аналог простого `cat`, только без явного буфера для `read`.

```c
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>

int main(int argc, char *argv[]) {
    if (argc != 2) {
        return 1;
    }

    int fd = open(argv[1], O_RDONLY);
    if (fd < 0) {
        perror("open");
        return 1;
    }

    struct stat st;
    if (fstat(fd, &st) < 0) {
        perror("fstat");
        close(fd);
        return 1;
    }

    if (st.st_size == 0) {
        close(fd);
        return 0;
    }

    char *data = mmap(NULL, st.st_size, PROT_READ, MAP_PRIVATE, fd, 0);
    if (data == MAP_FAILED) {
        perror("mmap");
        close(fd);
        return 1;
    }

    write(STDOUT_FILENO, data, st.st_size);

    munmap(data, st.st_size);
    close(fd);
    return 0;
}
```

Запуск:

    $ gcc mmap_cat.c -o mmap_cat
    $ ./mmap_cat text.txt

Что важно:
* размер файла берём через `fstat`;
* пустой файл не отображаем;
* результат проверяем на `MAP_FAILED`;
* в конце вызываем `munmap`.

## Сценарий 2: `MAP_PRIVATE`

`MAP_PRIVATE` позволяет менять память, но не менять файл.

Пусть есть файл:

    $ printf 'hello\n' > text.txt

Программа:

```c
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>

int main(void) {
    int fd = open("text.txt", O_RDONLY);
    if (fd < 0) {
        perror("open");
        return 1;
    }

    struct stat st;
    if (fstat(fd, &st) < 0) {
        perror("fstat");
        close(fd);
        return 1;
    }

    char *p = mmap(NULL, st.st_size,
                   PROT_READ | PROT_WRITE,
                   MAP_PRIVATE, fd, 0);
    if (p == MAP_FAILED) {
        perror("mmap");
        close(fd);
        return 1;
    }

    p[0] = 'H';
    write(STDOUT_FILENO, p, st.st_size);

    munmap(p, st.st_size);
    close(fd);
    return 0;
}
```

**OUTPUT:**
```text
Hello
```

Файл не изменился:

    $ cat text.txt
    hello

При записи ядро сделало приватную копию страницы.

## Сценарий 3: `MAP_SHARED`

`MAP_SHARED` связывает запись в память с файлом.

```c
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>

int main(void) {
    int fd = open("text.txt", O_RDWR);
    if (fd < 0) {
        perror("open");
        return 1;
    }

    struct stat st;
    if (fstat(fd, &st) < 0) {
        perror("fstat");
        close(fd);
        return 1;
    }

    char *p = mmap(NULL, st.st_size,
                   PROT_READ | PROT_WRITE,
                   MAP_SHARED, fd, 0);
    if (p == MAP_FAILED) {
        perror("mmap");
        close(fd);
        return 1;
    }

    p[0] = 'H';

    if (msync(p, st.st_size, MS_SYNC) < 0) {
        perror("msync");
    }

    munmap(p, st.st_size);
    close(fd);
    return 0;
}
```

Запуск:

    $ printf 'hello\n' > text.txt
    $ gcc shared.c -o shared
    $ ./shared
    $ cat text.txt
    Hello

## Создаём файл и пишем через `mmap`

Если хотим создать файл и заполнить его через отображение, сначала задаём размер.
Само присваивание в память не расширяет файл.

```c
#include <sys/mman.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>
#include <stdio.h>

int main(void) {
    int fd = open("out.txt", O_RDWR | O_CREAT | O_TRUNC, 0600);
    if (fd < 0) {
        perror("open");
        return 1;
    }

    size_t size = 4096;
    if (ftruncate(fd, size) < 0) {
        perror("ftruncate");
        close(fd);
        return 1;
    }

    char *p = mmap(NULL, size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if (p == MAP_FAILED) {
        perror("mmap");
        close(fd);
        return 1;
    }

    strcpy(p, "hello from mmap\n");

    msync(p, size, MS_SYNC);
    munmap(p, size);
    close(fd);
    return 0;
}
```

`ftruncate` меняет размер файла.
После этого у отображения есть реальные байты файла, в которые можно писать.

## Анонимные отображения

Отображение не обязано быть связано с файлом.
Можно попросить у ядра просто нулевые страницы памяти:

```c
char *p = mmap(NULL, 4096,
               PROT_READ | PROT_WRITE,
               MAP_PRIVATE | MAP_ANONYMOUS,
               -1, 0);
```

Для анонимного отображения:
* `fd = -1`;
* `offset = 0`;
* память изначально заполнена нулями.

Пример:

```c
#include <sys/mman.h>
#include <unistd.h>
#include <stdio.h>

int main(void) {
    size_t size = sysconf(_SC_PAGESIZE);

    int *x = mmap(NULL, size,
                  PROT_READ | PROT_WRITE,
                  MAP_PRIVATE | MAP_ANONYMOUS,
                  -1, 0);
    if (x == MAP_FAILED) {
        perror("mmap");
        return 1;
    }

    printf("%d\n", *x);
    *x = 42;
    printf("%d\n", *x);

    munmap(x, size);
    return 0;
}
```

**OUTPUT:**
```text
0
42
```

Большие выделения в `malloc` часто делаются именно через `mmap`.
`calloc` тоже пользуется тем, что свежие страницы от ОС уже заполнены нулями.
Если аллокатор выдаёт старый освобождённый блок, ему всё равно придётся занулить
его вручную.

## `mmap` и `fork`

После `fork` отображения наследуются дочерним процессом.

Для `MAP_PRIVATE`:
* до записи страницы могут быть общими;
* после записи у процесса появляется своя копия;
* изменения не видны другому процессу.

Для `MAP_SHARED` изменения видны обоим процессам.

```c
#include <sys/mman.h>
#include <sys/wait.h>
#include <unistd.h>
#include <stdio.h>

int main(void) {
    int *value = mmap(NULL, sizeof(*value),
                      PROT_READ | PROT_WRITE,
                      MAP_SHARED | MAP_ANONYMOUS,
                      -1, 0);
    if (value == MAP_FAILED) {
        perror("mmap");
        return 1;
    }

    *value = 1;

    pid_t pid = fork();
    if (pid == 0) {
        *value = 100;
        _exit(0);
    }

    wait(NULL);
    printf("%d\n", *value);

    munmap(value, sizeof(*value));
    return 0;
}
```

**OUTPUT:**
```text
100
```

Это удобный способ получить shared memory между родственными процессами.
Но shared memory не делает `++x` атомарным.

## Что происходит у конца файла?

Файл может быть не кратен размеру страницы.
Например, файл занимает 10 байт, а страница — 4096.

POSIX требует, чтобы хвост последней страницы за концом файла был заполнен нулями.
Изменения этого хвоста не записываются обратно в файл.

Но доступ к целым страницам после конца файла может привести к `SIGBUS`.

Опасный пример:

```c
int fd = open("empty.txt", O_RDWR | O_CREAT | O_TRUNC, 0600);
char *p = mmap(NULL, 4096, PROT_READ, MAP_SHARED, fd, 0);

char c = p[0];  // файл пустой: на Linux обычно SIGBUS
```

Поэтому:
* перед чтением смотрим размер через `fstat`;
* пустой файл не отображаем;
* перед записью в новый файл расширяем его через `ftruncate`.

## `mprotect`

Права существующего отображения можно поменять:

```c
#include <sys/mman.h>

int mprotect(void *addr, size_t len, int prot);
```

Пример:

```c
char *p = mmap(NULL, 4096,
               PROT_READ | PROT_WRITE,
               MAP_PRIVATE | MAP_ANONYMOUS,
               -1, 0);

p[0] = 'x';
mprotect(p, 4096, PROT_READ);
p[0] = 'y';  // SIGSEGV
```

Так работают guard pages, JIT-компиляторы и некоторые отладочные техники.

## `MAP_FIXED`

Есть флаг `MAP_FIXED`, который просит ядро отобразить память ровно по адресу `addr`.

Почти всегда это плохая идея:
* можно случайно заменить существующее отображение;
* можно сломать `malloc`, стек или shared libraries;
* переносимость низкая.

Обычный код передаёт `addr = NULL` и даёт ядру выбрать адрес.

## `mmap` или `read`?

`mmap` не делает файл автоматически быстрее.
Это другой интерфейс к данным.

Плюсы:
* удобно обращаться к файлу как к массиву;
* не нужен явный буфер и цикл `read`;
* удобно читать структуры по смещениям;
* страницы файла могут разделяться между процессами;
* ОС подгружает только реально использованные страницы.

Минусы:
* ошибка может прийти не при `mmap`, а позже, при обращении к памяти;
* возможен `SIGBUS`, если файл стал меньше;
* сложнее контролировать момент записи;
* при смешивании с `read`/`write` нужна аккуратность;
* для маленького последовательного чтения обычный `read` проще.

## Частые ошибки

### Проверять результат на `NULL`

Неправильно:

```c
if (p == NULL) {
    perror("mmap");
}
```

Правильно:

```c
if (p == MAP_FAILED) {
    perror("mmap");
}
```

### Мапить пустой файл

```c
mmap(NULL, 0, PROT_READ, MAP_PRIVATE, fd, 0);  // EINVAL
```

Если файл пустой, не вызываем `mmap` или сначала расширяем файл через `ftruncate`.

### Забыть про выравнивание `offset`

```c
mmap(NULL, len, PROT_READ, MAP_PRIVATE, fd, 13);  // EINVAL
```

Смещение должно быть кратно размеру страницы.

### Писать в `MAP_SHARED`, открыв файл только на чтение

```c
int fd = open("data.txt", O_RDONLY);
mmap(NULL, size, PROT_WRITE, MAP_SHARED, fd, 0);  // EACCES
```

Для shared-записи нужен `O_RDWR`.

### Думать, что `mmap` копирует файл в память

`mmap` создаёт отображение.
Страницы читаются лениво, по page fault.

### Думать, что shared memory решает гонки

`MAP_SHARED` даёт общий адресуемый объект.
Порядок операций и атомарность всё ещё ваша ответственность.
