# GNU Debugger tutorial
Во время написания кода неизбежно возникают ошибки: неправильный вывод, неправильное значение
переменной, ошибка памяти (segmentation fault). И когда такое происходит, сразу хочется натыкать
print-ов и так посмотреть, что пошло не так. Но такой подход может отловить не все ошибки да и потом
придётся удалять все выводы в терминал (а в большой программе их не всегда просто найти).
Более правильным и универсальным способом является ильзование отладчика (debugger). Здесь мы
рассмотрим использование gdb на примере работы с языком C.

## 1. Запуск отладки
Прежде чем начать работу с отладчиком, надо подготовить программу на C к работе с ним. Для этого
надо добавить два простых флага: 
- `-g`: включение отладочной информации;
- `-O0`: отключение оптимизаций.

Теперь, например, компиляция программы может выглядеть так:
```bash
$ gcc -Wall -Werror -Wextra -std=c11 -O0 -g filename.c -o out
```

Терерь мы готовы включать отладчик, это делаеться командой:
```bash
$ gdb ./out
```

Чтобы запустить программу, нужно в отладчике ввести команду 
```gdb
(gdb) run
```
Также можно передать параметры командной строки: `run 1 2`.

Чтобы закрыть gdb надо ввести команду:
```gdb
(gdb) quit
```

## 2. Вывод информации, команда print
`print` - встроенная команда GDB, которая вычисляет значение выражения на C. Рассмотрим несколько
примеров:

```gdb
(gdb) print 1 + 2
$1 = 3
```
Здесь мы просто вычисляем значение выражения `1 + 2`.

```gdb
(gdb) print (int)2147483648
$2 = -2147483648
```
Мы преобразовываем большое число к типу `int`. Происходит переполнение типа, поэтому мы получаем
отрицательное число.

```gdb
(gdb) print $1
$3 = 3
```
Выводим результат первого вычисления (`$1 = 3`).

Это ещё не все возможности данной команды, но остальные мы рассмотрим немного позже.

## 3. Базовая отладка, команды break и next
- `break` - добавляет точку остановки в программе. То есть, когда программа в процессе выполнения
  доходит до этой точки, то она останавливается и ожидает следующего действия;
- `next` - выполняет текущую строку кода и переходит на вторую.

Рассмотрим вышеуказанные команды на простом примере:
```c
int main(void) {
    int i = 1432;
    int x = 4329;
    x -= i;
    return 0;
}
```
Скомпилируем эту программу и запустим её с помощью отладчика.
В первую очередь добавим точку остановки (breakpoint) в функции `main`:
```gdb
(gdb) break main
Breakpoint 1 at 0x111d: file test.c, line 2.
```
Мы успешно создали точку остановки в начале функции `main`, теперь можно запустить программу:
```gdb
(gdb) run
Breakpoint 1, main () at test.c:2
2           int i = 1432;
```
Мы достигли ранее созданной точки остановки, можно попробовать вывести переменную `i`:
```gdb
(gdb) print i
$1 = -134329952
```
Переменная ещё не инициализирована, поэтому в ней хранится случайное значение.

Далее будем построчно выполнять программу (для этого используем команду `next`) и следить за 
изменением переменных.
```gdb
(gdb) next
3           int x = 4329;
```
Мы перешли на строку с инициализацией переменной `x`. Попробуем вывести обе переменных:
```gdb
(gdb) print i
$2 = 1432
(gdb) print x
$3 = 32767
```
Переменная `i` теперь имеет заданное в программе значение, а `x` ещё не инициализирована.

Переходим на следующую строку:
```gdb
(gdb) next
4           x -= i;
(gdb) print x
$4 = 4329
```
Теперь и `x` тоже инициализирована.

Выполним предпоследнюю строку с вычитанием:
```gdb
(gdb) next
5           return 0;
(gdb) print x
$5 = 2897
```
Мы выполнили вычитание.

Отладка закончена, можно ввести команду `continue`, чтобы дать программе завершиться.
```gdb
(gdb) continue
Continuing.
[Inferior 1 (process 9002) exited normally]
```
Процесс успешно завершился.

## 4. Добавляем больше break-ов, команда continue
- `continue` - продолжает выполнение программы до следующей точки остановки или конца программы.

Рассмотрим более сложную программу:
```c
int test_func(void) {
    int i = 0;
    for (; i < 55; i += 2) {
        i *= 2;
    }
    return i;
}

int main(void) {
    int res = test_func();
    return 0;
}
```
Скомпилируем её и запустим дебаггер. Добавим две точки остановки: на 4 и на 11 строке:
```gdb
(gdb) break test.c:4
Breakpoint 1 at 0x1126: file test.c, line 4.
(gdb) break test.c:11
Breakpoint 2 at 0x1148: file test.c, line 11.
```
Запускаем программу командой `run`.

GDB остановится, когда дойдёт до первой точки остановки:
```gdb
Breakpoint 1, test_func () at test.c:4
4               i *= 2;
(gdb) print i
$1 = 0
```

Чтобы продолжить выполнение программы, нужно ввести команду:
```gdb
(gdb) continue
Continuing.

Breakpoint 1, test_func () at test.c:4
4               i *= 2;
(gdb) print i
$2 = 2
```
Мы снова попали в точку остановки на 4-ой линии. Так будет продолжаться пока мы не выйдем из цикла.
А когда мы выйдем из цикла, то попадём в следующую точку остановки на линии 11:
```gdb
(gdb) continue
Continuing.

Breakpoint 2, main () at test.c:11
11          return 0;
(gdb) print res
$6 = 62
(gdb) continue
Continuing.
[Inferior 1 (process 11972) exited normally]
```
Таким образом мы отладили программу.

## 5. Наблюдаем за изменением переменной, команда watch
В прошлом пункте мы отслеживали изменениие значения переменной `i` с помощью точки остановки и
команды `print`, но есть более простой способ сделать это.
```gdb
(gdb) break test.c:3
Breakpoint 1 at 0x1124: file test.c, line 3.
(gdb) run
Breakpoint 1, test_func () at test.c:3
3           for (; i < 55; i += 2) {
(gdb) watch i
Hardware watchpoint 2: i
(gdb) continue
Continuing.

Hardware watchpoint 2: i

Old value = 0
New value = 2
0x000055555555512d in test_func () at test.c:3
3           for (; i < 55; i += 2) {
```
Здесь мы ставим точку остановки перед циклом, и когда достигаем её, то начинаем отслеживать
переменную `i` (мы не можем начать отслеживать переменную до её объявления). 
Когда переменная изменяется, программа останавливается. Продолжить выполнение программы можно
командой `continue`. Так мы продолжаем выполнение программы, пока цикл не закончится, когда это
произойдёт, мы увидим следующее сообщение:
```gdb
(gdb) continue
Continuing.

Watchpoint 2 deleted because the program has left the block in
which its expression is valid.
0x0000555555555145 in main () at test.c:10
10          int res = test_func();
(gdb) print res
$1 = 32767
(gdb) next
11          return 0;
(gdb) print res
$2 = 62
```

## 6. Удаление точек остановки, команды info breakpoints и delete
Если точка остановик или отслеживаемая переменная больше не нужна, то её можно удалить. Для этого
нужно узнать идентификатор, его можно узнать с помощью команды `info breakpoints`, и после этого
удалить командой `delete <id>`. Например:
```gdb
(gdb) info breakpoints
Num     Type           Disp Enb Address            What
1       breakpoint     keep y   0x0000555555555126 in test_func at test.c:4
        breakpoint already hit 1 time
2       breakpoint     keep y   0x0000555555555148 in main at test.c:11
3       hw watchpoint  keep y                      i
(gdb) delete 3
(gdb) info breakpoints
Num     Type           Disp Enb Address            What
1       breakpoint     keep y   0x0000555555555126 in test_func at test.c:4
        breakpoint already hit 1 time
2       breakpoint     keep y   0x0000555555555148 in main at test.c:11
```

## 7. Ловим ошибки памяти, команда backtrace
Рассмотрим программу:
```c
#include <stdlib.h>

void segfault(void) {
    char *data = malloc(10);
    free(data);
    free(data);
}

int main(void) {
    segfault();
    return 0;
}
```
Здесь мы два раза освобождаем память.
Когда мы запустим программу, то увидим следующее сообщение:
```gdb
(gdb) run
Using host libthread_db library "/usr/lib/libthread_db.so.1".
free(): double free detected in tcache 2

Program received signal SIGABRT, Aborted.
Downloading 4.48 K source file /usr/src/debug/glibc/glibc/nptl/pthread_kill.c
__pthread_kill_implementation (threadid=<optimized out>, signo=signo@entry=6, no_tid=no_tid@entry=0)
    at pthread_kill.c:44
44            return INTERNAL_SYSCALL_ERROR_P (ret) ? INTERNAL_SYSCALL_ERRNO (ret) : 0;
```
В таком случае можно использовать команду `backtrace`:
```gdb
(gdb) backtrace
#0  __pthread_kill_implementation (threadid=<optimized out>, signo=signo@entry=6, no_tid=no_tid@entry=0)
    at pthread_kill.c:44
#1  0x00007ffff7e4b813 in __pthread_kill_internal (threadid=<optimized out>, signo=6)
    at pthread_kill.c:89
#2  0x00007ffff7df1dc0 in __GI_raise (sig=sig@entry=6) at ../sysdeps/posix/raise.c:26
#3  0x00007ffff7dd957a in __GI_abort () at abort.c:73
#4  0x00007ffff7dda5c9 in __libc_message_impl (fmt=fmt@entry=0x7ffff7f6639f "%s\n")
    at ../sysdeps/posix/libc_fatal.c:134
#5  0x00007ffff7e55a35 in malloc_printerr (
    str=str@entry=0x7ffff7f68f98 "free(): double free detected in tcache 2") at malloc.c:5829
#6  0x00007ffff7e55ac3 in tcache_double_free_verify (e=e@entry=0x5555555592a0, tc_idx=tc_idx@entry=0)
    at malloc.c:3240
#7  0x00007ffff7e5af6a in tcache_free (p=<optimized out>, size=32) at malloc.c:3263
#8  _int_free (av=0x7ffff7f9aac0 <main_arena>, p=<optimized out>, have_lock=0) at malloc.c:4695
#9  __GI___libc_free (mem=<optimized out>) at malloc.c:3476
#10 0x0000555555555177 in segfault () at test.c:6
#11 0x0000555555555183 in main () at test.c:10
```
В строке `#10 0x0000555555555177 in segfault () at test.c:6` можно обнаружить, где произошла ошибка.

## 8. Сокращение команд
У многих команд в gdb есть свои сокращения, далее я их перечислю:
- `run` - `r`;
- `next` - `n`;
- `quit` - `q`;
- `help` - `h`.
- `print` - `p`;
- `break` - `b`;
- `continue` - `c`;
- `backtrace` - `bt`;
- `info breakpoints` - `i b`.

## 9. Другие интересные команды
- `print sizeof(var)` - выводит размер переменной в байтах;
- `ptype var` - выводит тип данных переменной `var`;
- `x/4xb var` - выводит 4 байта переменной `var` в шестнадцатеричном формате;
- `gdb -tui ./out` - запускает текстовый пользовательский интерфейс (text user interface).
- `gdb -p 12345` - присоединяется к уже существующему процессу по его id;
- `until 100` - выполнить программу до указаной линии;

