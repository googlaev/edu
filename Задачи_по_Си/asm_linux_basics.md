# Основы ассемблера под Linux
### x86-64 NASM — с нуля до первых программ

---

## Содержание

1. [Что такое ассемблер](#1-что-такое-ассемблер)
2. [Инструменты: NASM, ld, gcc](#2-инструменты-nasm-ld-gcc)
3. [Структура программы](#3-структура-программы)
4. [Регистры процессора](#4-регистры-процессора)
5. [Основные команды пересылки данных](#5-основные-команды-пересылки-данных)
6. [Арифметические операции](#6-арифметические-операции)
7. [Логические операции](#7-логические-операции)
8. [Сравнение и условные переходы](#8-сравнение-и-условные-переходы)
9. [Безусловный переход и метки](#9-безусловный-переход-и-метки)
10. [Циклы](#10-циклы)
11. [Стек](#11-стек)
12. [Системные вызовы Linux](#12-системные-вызовы-linux)
13. [Работа с памятью](#13-работа-с-памятью)
14. [Первые программы](#14-первые-программы)
15. [Шпаргалка](#15-шпаргалка)

---

## 1. Что такое ассемблер

Ассемблер — язык программирования, в котором каждая строка кода
соответствует **одной машинной инструкции** процессора.

```
Ваш код:    mov rax, 5       ← одна строка
Процессор:  48 C7 C0 05 00 00 00   ← 7 байт машинного кода
```

### Зачем его учить

- Понять, как процессор на самом деле выполняет программы
- Писать ядра ОС, драйверы, загрузчики
- Разбирать скомпилированный код (reverse engineering)
- Писать критически быстрый код вручную
- Понимать уязвимости (buffer overflow, ROP-цепочки)

### Уровни абстракции

```
Python      →  интерпретатор + байткод
C           →  компилятор генерирует ассемблер
Ассемблер   →  ассемблер (NASM) генерирует машинный код
Машинный код →  процессор исполняет напрямую
```

---

## 2. Инструменты: NASM, ld, gcc

### Установка

```bash
sudo apt update
sudo apt install nasm binutils gcc
```

### Что это

| Программа | Роль |
|-----------|------|
| `nasm` | Ассемблер: текст `.asm` → объектный файл `.o` |
| `ld` | Линковщик: `.o` → исполняемый файл |
| `gcc` | Можно использовать вместо `ld` — проще |
| `objdump` | Просмотр машинного кода |
| `strace` | Трассировка системных вызовов |

### Цикл сборки

```bash
# 1. Написать код
nano hello.asm

# 2. Ассемблировать: .asm → .o
nasm -f elf64 hello.asm -o hello.o

# 3. Слинковать: .o → исполняемый файл
ld hello.o -o hello

# 4. Запустить
./hello

# Или через gcc (проще, подключает стандартную библиотеку С):
gcc -no-pie hello.o -o hello
```

### Посмотреть сгенерированный машинный код

```bash
objdump -d hello
```

---

## 3. Структура программы

```asm
; Это комментарий — всё после точки с запятой игнорируется

section .data           ; Секция данных — инициализированные переменные
    msg db "Привет!", 10    ; строка + символ новой строки (10 = '\n')
    len equ $ - msg         ; длина строки ($ = текущий адрес)

section .bss            ; Секция для неинициализированных переменных
    buf resb 64             ; зарезервировать 64 байта

section .text           ; Секция кода
    global _start           ; объявить точку входа (нужно для ld)

_start:                 ; Метка — точка входа программы
    ; ... инструкции ...

    ; Завершить программу (системный вызов exit)
    mov rax, 60         ; номер системного вызова: exit
    mov rdi, 0          ; код возврата: 0 = успех
    syscall             ; вызвать ядро Linux
```

### Директивы объявления данных

```asm
section .data
    byte_val  db  42            ; 1 байт (define byte)
    word_val  dw  1000          ; 2 байта (define word)
    dword_val dd  100000        ; 4 байта (define dword)
    qword_val dq  1000000000    ; 8 байт (define qword)

    str1  db  "Hello", 0        ; строка с нулём в конце (C-style)
    str2  db  "World", 10       ; строка с переводом строки
    array db  1, 2, 3, 4, 5    ; массив байт

section .bss
    buf8   resb  8      ; зарезервировать 8 байт
    buf16  resw  4      ; зарезервировать 4 слова (= 8 байт)
    buf32  resd  2      ; зарезервировать 2 двойных слова
    buf64  resq  1      ; зарезервировать 1 восьмибайтовое слово
```

---

## 4. Регистры процессора

В x86-64 есть 16 регистров общего назначения, каждый 64-битный.

### Карта регистров

```
64-бит   32-бит   16-бит   8-бит (младший)
rax      eax      ax       al        ← Аккумулятор, результат вызовов
rbx      ebx      bx       bl        ← Базовый (сохраняемый)
rcx      ecx      cx       cl        ← Счётчик циклов
rdx      edx      dx       dl        ← Данные, делитель
rsi      esi      si       sil       ← Источник строковых операций
rdi      edi      di       dil       ← Приёмник; 1-й аргумент syscall
rbp      ebp      bp       bpl       ← Указатель базы фрейма
rsp      esp      sp       spl       ← Указатель стека (stack pointer)
r8       r8d      r8w      r8b       ← Дополнительные регистры
r9       r9d      r9w      r9b
r10      r10d     r10w     r10b
r11      r11d     r11w     r11b
r12      r12d     r12w     r12b
r13      r13d     r13w     r13b
r14      r14d     r14w     r14b
r15      r15d     r15w     r15b
```

### Специальные регистры

```
rip   — счётчик команд (instruction pointer): адрес текущей инструкции
rflags — регистр флагов: ZF, SF, CF, OF, PF...
```

### Соглашение о системных вызовах Linux (x86-64)

```
rax  — номер системного вызова
rdi  — 1-й аргумент
rsi  — 2-й аргумент
rdx  — 3-й аргумент
r10  — 4-й аргумент
r8   — 5-й аргумент
r9   — 6-й аргумент
```

После `syscall` результат возвращается в `rax`.

---

## 5. Основные команды пересылки данных

### mov — переместить данные

```asm
; Синтаксис: mov приёмник, источник
; Приёмник получает значение источника (источник не меняется)

mov rax, 42         ; rax = 42  (константа → регистр)
mov rbx, rax        ; rbx = rax (регистр → регистр)
mov rax, [rbx]      ; rax = *rbx (чтение из памяти по адресу в rbx)
mov [rbx], rax      ; *rbx = rax (запись в память)
mov [rbx], 5        ; *rbx = 5  (константа → память)

; Размеры операндов
mov al,  42         ; 8-бит
mov ax,  1000       ; 16-бит
mov eax, 100000     ; 32-бит (также обнуляет старшие 32 бита rax!)
mov rax, 1000000    ; 64-бит
```

> **Важно:** нельзя делать `mov [mem], [mem]` — память → память
> напрямую. Нужен промежуточный регистр.

### movzx — пересылка с расширением нулями

```asm
; Кладёт меньшее значение в больший регистр, дополняя нулями
movzx rax, byte [rbx]   ; rax = (uint64_t)(uint8_t)*rbx
movzx eax, word [rbx]   ; eax = (uint32_t)(uint16_t)*rbx
movzx rax, ax           ; rax = (uint64_t)(uint16_t)ax
```

### movsx — пересылка с расширением знака

```asm
; Дополняет знаковым битом (для знаковых чисел)
movsx rax, byte [rbx]   ; rax = (int64_t)(int8_t)*rbx
movsx rax, eax          ; rax = (int64_t)(int32_t)eax
```

### xchg — обменять значения

```asm
xchg rax, rbx    ; поменять rax и rbx местами (без temp-переменной)
```

### lea — загрузить эффективный адрес

```asm
; lea вычисляет адрес, но НЕ читает память
lea rax, [rbx + 8]      ; rax = rbx + 8 (адрес, не значение)
lea rax, [rbx + rcx*4]  ; rax = rbx + rcx*4

; Трюк: быстрое умножение на 2, 3, 5, 9
lea rax, [rax + rax]    ; rax *= 2
lea rax, [rax + rax*2]  ; rax *= 3
lea rax, [rax*4 + rax]  ; rax *= 5
```

---

## 6. Арифметические операции

### Сложение и вычитание

```asm
add rax, rbx        ; rax = rax + rbx
add rax, 10         ; rax = rax + 10
add [var], rax      ; *var += rax
add [var], 5        ; *var += 5

sub rax, rbx        ; rax = rax - rbx
sub rax, 10         ; rax = rax - 10

; Инкремент и декремент (на 1)
inc rax             ; rax++
dec rax             ; rax--
inc byte [rbx]      ; (*rbx)++  (байт по адресу rbx)
```

### Умножение

```asm
; Беззнаковое умножение: mul источник
; Результат: rdx:rax = rax * источник

mov rax, 6
mov rbx, 7
mul rbx             ; rdx:rax = rax * rbx = 42
; rax = 42, rdx = 0 (старшие 64 бита)

; Знаковое умножение: imul
imul rbx            ; rdx:rax = rax * rbx (знаковое)

; imul с двумя операндами (удобнее):
imul rax, rbx       ; rax *= rbx
imul rax, rcx, 10   ; rax = rcx * 10
```

### Деление

```asm
; Беззнаковое деление: div делитель
; Делится rdx:rax на делитель
; Частное → rax, остаток → rdx

mov rax, 17
mov rdx, 0      ; ОБЯЗАТЕЛЬНО обнулить rdx перед div!
mov rbx, 5
div rbx         ; rax = 17/5 = 3, rdx = 17%5 = 2

; Знаковое деление: idiv
; Перед idiv нужно знаково расширить rax в rdx:rax
mov rax, -17
cqo             ; sign-extend rax → rdx:rax  (Convert Quadword to Octword)
mov rbx, 5
idiv rbx        ; rax = -3, rdx = -2
```

### Смена знака

```asm
neg rax         ; rax = -rax
```

---

## 7. Логические операции

```asm
; AND — логическое И (побитово)
and rax, rbx        ; rax &= rbx
and rax, 0xFF       ; оставить только младший байт

; OR — логическое ИЛИ
or  rax, rbx        ; rax |= rbx
or  rax, 0x80       ; установить бит 7

; XOR — исключающее ИЛИ
xor rax, rbx        ; rax ^= rbx
xor rax, rax        ; rax = 0  ← идиома обнуления регистра (быстрее mov rax,0)
xor rax, 1          ; перевернуть бит 0

; NOT — инверсия всех битов
not rax             ; rax = ~rax

; Сдвиги влево
shl rax, 1          ; rax <<= 1  (умножение на 2)
shl rax, 3          ; rax <<= 3  (умножение на 8)
shl rax, cl         ; rax <<= cl (на cl бит, cl = регистр счётчика)

; Сдвиги вправо (беззнаковые)
shr rax, 1          ; rax >>= 1  (деление на 2)
shr rax, cl         ; rax >>= cl

; Арифметический сдвиг вправо (сохраняет знак)
sar rax, 1          ; rax >>= 1  (знаковое деление на 2)

; Циклический сдвиг
rol rax, 1          ; сдвиг влево по кругу
ror rax, 1          ; сдвиг вправо по кругу
```

### Проверка бит (test)

```asm
; test: выполняет AND, но не сохраняет результат — только ставит флаги
test rax, rax       ; проверить: rax == 0? (ставит ZF если rax=0)
test rax, 1         ; проверить младший бит (ставит ZF если бит = 0)
test al,  0x80      ; проверить бит 7 в al
```

---

## 8. Сравнение и условные переходы

### cmp — сравнение

```asm
; cmp A, B  вычисляет A - B и ставит флаги, результат не сохраняет
cmp rax, rbx        ; сравнить rax и rbx
cmp rax, 0          ; сравнить rax с нулём
cmp byte [rbx], 10  ; сравнить байт памяти с 10
```

### Флаги после cmp/test

| Флаг | Название | Когда установлен |
|------|----------|-----------------|
| `ZF` | Zero Flag | Результат = 0 (равны) |
| `SF` | Sign Flag | Результат отрицательный |
| `CF` | Carry Flag | Беззнаковое переполнение |
| `OF` | Overflow Flag | Знаковое переполнение |

### Условные переходы (после cmp)

```asm
; Для знаковых чисел:
je   метка   ; jump if equal          (ZF=1)
jne  метка   ; jump if not equal      (ZF=0)
jl   метка   ; jump if less           (SF≠OF)
jle  метка   ; jump if less or equal  (ZF=1 или SF≠OF)
jg   метка   ; jump if greater        (ZF=0 и SF=OF)
jge  метка   ; jump if greater/equal  (SF=OF)

; Для беззнаковых чисел:
jb   метка   ; jump if below          (CF=1)   — аналог jl
jbe  метка   ; jump if below or equal (CF=1 или ZF=1)
ja   метка   ; jump if above          (CF=0 и ZF=0)
jae  метка   ; jump if above or equal (CF=0)

; Проверка флагов напрямую:
jz   метка   ; jump if zero   (ZF=1) — то же что je
jnz  метка   ; jump if not zero       — то же что jne
js   метка   ; jump if sign (отрицательный результат)
jns  метка   ; jump if not sign
jc   метка   ; jump if carry
jnc  метка   ; jump if no carry
```

### Пример: if-else на ассемблере

```asm
; Аналог: if (rax > rbx) { rax = rax - rbx; } else { rax = 0; }

    cmp rax, rbx
    jle .else_branch        ; если rax <= rbx — перейти в else
    sub rax, rbx            ; then: rax -= rbx
    jmp .end_if
.else_branch:
    xor rax, rax            ; else: rax = 0
.end_if:
    ; продолжаем...
```

---

## 9. Безусловный переход и метки

```asm
; Метка — просто имя для адреса в коде
my_label:
    mov rax, 1

; Безусловный переход (goto)
    jmp my_label        ; перейти на my_label

; Метки принято именовать:
;   .local_label    — локальная (видна только в пределах процедуры)
;   global_label    — глобальная
```

---

## 10. Циклы

### Простой цикл со счётчиком

```asm
; Аналог: for (rcx = 5; rcx > 0; rcx--)

    mov rcx, 5          ; счётчик = 5
.loop:
    ; ... тело цикла ...
    dec rcx
    jnz .loop           ; если rcx ≠ 0 — повторить
```

### Команда loop (удобный вариант)

```asm
; loop метка: dec rcx + jnz метка одной командой

    mov rcx, 10         ; счётчик итераций
.count_loop:
    ; ... тело ...
    loop .count_loop    ; rcx--, если rcx≠0 — перейти
```

### Цикл while

```asm
; Аналог: while (rax < 100) { rax += rax; }

.while_start:
    cmp rax, 100
    jge .while_end      ; если rax >= 100 — выход
    add rax, rax        ; rax *= 2
    jmp .while_start
.while_end:
```

### Цикл do-while

```asm
; Аналог: do { dec rax; } while (rax > 0);

.do_start:
    dec rax
    cmp rax, 0
    jg  .do_start
```

### Пример: сумма чисел от 1 до N

```asm
; rdi = N, результат в rax

    xor rax, rax        ; sum = 0
    mov rcx, rdi        ; счётчик = N
    xor rbx, rbx        ; i = 0
.sum_loop:
    inc rbx             ; i++
    add rax, rbx        ; sum += i
    loop .sum_loop      ; rcx--; if rcx≠0 goto loop
    ; rax = N*(N+1)/2
```

---

## 11. Стек

Стек — область памяти, растущая **вниз** (от старших адресов к младшим).
`rsp` всегда указывает на вершину стека.

### push и pop

```asm
push rax        ; rsp -= 8; *rsp = rax   (положить на стек)
pop  rbx        ; rbx = *rsp; rsp += 8  (снять со стека)

push 42         ; можно класть константу
push qword [var]; кладёт 8 байт из памяти
```

### Ручная работа со стеком

```asm
; Сохранить rax без pop
push rax
; ... другой код ...
pop  rax    ; восстановить

; Чтение со стека без извлечения
push rax
mov rbx, [rsp]      ; rbx = вершина стека (= rax)
mov rcx, [rsp + 8]  ; следующий элемент
pop rax

; Выровнять стек (нужно для некоторых вызовов)
sub rsp, 8      ; зарезервировать 8 байт
; ...
add rsp, 8      ; освободить
```

### Зачем нужен стек

```asm
; 1. Сохранять регистры перед их использованием
push rbx
push rcx
    mov rbx, 10
    mov rcx, 20
    ; ... используем rbx, rcx ...
pop  rcx
pop  rbx        ; восстанавливаем в обратном порядке!

; 2. Передавать параметры (старый способ, до x86-64)
; 3. Хранить локальные переменные
sub rsp, 32     ; зарезервировать 32 байта под локальные переменные
mov qword [rsp],    0   ; первая переменная
mov qword [rsp+8],  0   ; вторая переменная
; ...
add rsp, 32     ; освободить
```

---

## 12. Системные вызовы Linux

Linux предоставляет услуги ядра через инструкцию `syscall`.

### Механизм

```asm
mov rax, <номер_syscall>   ; что делать
mov rdi, <аргумент 1>
mov rsi, <аргумент 2>
mov rdx, <аргумент 3>
syscall                    ; передать управление ядру
; результат в rax (отрицательный = код ошибки)
```

### Важнейшие системные вызовы

| № | Название | rdi | rsi | rdx | Что делает |
|---|----------|-----|-----|-----|------------|
| 0 | `read` | fd | buf | count | Читать из файла/stdin |
| 1 | `write` | fd | buf | count | Писать в файл/stdout |
| 60 | `exit` | code | — | — | Завершить процесс |
| 39 | `getpid` | — | — | — | Получить PID |
| 57 | `fork` | — | — | — | Создать дочерний процесс |
| 2 | `open` | path | flags | mode | Открыть файл |
| 3 | `close` | fd | — | — | Закрыть файл |

### Дескрипторы файлов (fd)

```
0 — stdin  (стандартный ввод)
1 — stdout (стандартный вывод)
2 — stderr (стандартный вывод ошибок)
```

### Пример: write в stdout

```asm
; write(1, "hello\n", 6)
mov rax, 1          ; syscall: write
mov rdi, 1          ; fd = stdout
mov rsi, msg        ; адрес строки
mov rdx, 6          ; количество байт
syscall
```

### Пример: read из stdin

```asm
; read(0, buf, 64)
mov rax, 0          ; syscall: read
mov rdi, 0          ; fd = stdin
mov rsi, buf        ; куда писать
mov rdx, 64         ; максимум байт
syscall
; в rax — сколько байт реально прочитано
```

---

## 13. Работа с памятью

### Адресация

```asm
; Формат: [база + индекс * масштаб + смещение]
; масштаб может быть: 1, 2, 4, 8

mov rax, [rbx]              ; *(rbx)
mov rax, [rbx + 8]          ; *(rbx + 8)
mov rax, [rbx + rcx]        ; *(rbx + rcx)
mov rax, [rbx + rcx*8]      ; *(rbx + rcx*8)
mov rax, [rbx + rcx*8 + 4]  ; *(rbx + rcx*8 + 4)
mov rax, [var]              ; значение по метке var
```

### Указание размера операнда

```asm
; Когда компилятор не может определить размер — указываем явно
mov byte  [rbx], 42     ; записать 1 байт
mov word  [rbx], 1000   ; записать 2 байта
mov dword [rbx], 70000  ; записать 4 байта
mov qword [rbx], 0      ; записать 8 байт
```

### Массивы

```asm
section .data
    arr dq 10, 20, 30, 40, 50   ; массив из 5 элементов по 8 байт

section .text
_start:
    ; arr[0]
    mov rax, [arr]

    ; arr[2]  (индекс 2, размер элемента 8)
    mov rax, [arr + 2*8]

    ; arr[rcx]  (rcx = индекс)
    mov rax, [arr + rcx*8]
```

### Строки (байт за байтом)

```asm
section .data
    str db "Hello", 0       ; строка, завершённая нулём

section .text
_start:
    ; Чтение байт за байтом
    lea rsi, [str]          ; rsi = адрес строки
.read_char:
    movzx rax, byte [rsi]   ; rax = текущий символ
    test  rax, rax          ; проверить на ноль (конец строки)
    jz    .done
    ; ... обработка символа в rax ...
    inc   rsi               ; следующий символ
    jmp   .read_char
.done:
```

---

## 14. Первые программы

### Программа 1: "Hello, World!"

```asm
; hello.asm
; Сборка:
;   nasm -f elf64 hello.asm -o hello.o
;   ld hello.o -o hello
;   ./hello

section .data
    msg db "Hello, World!", 10   ; строка + '\n'
    len equ $ - msg              ; длина строки

section .text
    global _start

_start:
    ; write(1, msg, len)
    mov rax, 1          ; системный вызов: write
    mov rdi, 1          ; fd = stdout
    mov rsi, msg        ; адрес строки
    mov rdx, len        ; длина
    syscall

    ; exit(0)
    mov rax, 60         ; системный вызов: exit
    mov rdi, 0          ; код возврата
    syscall
```

### Программа 2: Вывести цифру

```asm
; print_digit.asm — вывести одну цифру (0–9)

section .data
    digit db '0', 10    ; символ '0' + перевод строки

section .text
    global _start

_start:
    mov rax, 7              ; число, которое хотим вывести (0–9)
    add al, '0'             ; преобразовать в ASCII ('0' = 48)
    mov [digit], al         ; сохранить символ

    mov rax, 1
    mov rdi, 1
    mov rsi, digit
    mov rdx, 2
    syscall

    mov rax, 60
    xor rdi, rdi
    syscall
```

### Программа 3: Сложение двух чисел + вывод

```asm
; add_print.asm — сложить 3 + 4 и вывести результат

section .data
    result db '0', 10    ; буфер для одной цифры + '\n'

section .text
    global _start

_start:
    mov rax, 3          ; a = 3
    mov rbx, 4          ; b = 4
    add rax, rbx        ; rax = a + b = 7

    add al, '0'         ; ASCII-код цифры
    mov [result], al    ; сохранить

    mov rax, 1
    mov rdi, 1
    mov rsi, result
    mov rdx, 2
    syscall

    mov rax, 60
    xor rdi, rdi
    syscall
```

### Программа 4: Цикл — вывести числа от 1 до 9

```asm
; count.asm — вывести "123456789"

section .data
    digit db '0', 10    ; текущий символ + '\n'

section .text
    global _start

_start:
    mov rcx, 1          ; счётчик = 1

.loop:
    ; Вывести текущее число
    mov rax, rcx
    add al, '0'         ; в ASCII
    mov [digit], al

    mov rax, 1
    mov rdi, 1
    mov rsi, digit
    mov rdx, 2
    syscall

    inc rcx             ; rcx++
    cmp rcx, 10
    jl  .loop          ; если rcx < 10 — повторить

    ; exit
    mov rax, 60
    xor rdi, rdi
    syscall
```

### Программа 5: Прочитать символ и вывести его

```asm
; echo_char.asm — прочитать один символ и вывести его обратно

section .bss
    ch resb 1           ; буфер для одного символа

section .text
    global _start

_start:
    ; read(0, ch, 1)
    mov rax, 0
    mov rdi, 0
    mov rsi, ch
    mov rdx, 1
    syscall

    ; write(1, ch, 1)
    mov rax, 1
    mov rdi, 1
    mov rsi, ch
    mov rdx, 1
    syscall

    ; exit(0)
    mov rax, 60
    xor rdi, rdi
    syscall
```

### Программа 6: Вычислить факториал

```asm
; factorial.asm — вычислить 5! = 120 и вывести результат

section .data
    ; Готовая строка для вывода трёхзначного числа + '\n'
    out_buf db "   ", 10
    out_len equ 4

section .text
    global _start

; Процедура: вычислить n! в rax
; Вход: rdi = n
; Выход: rax = n!
factorial:
    mov rax, 1
    cmp rdi, 1
    jle .done           ; 0! = 1! = 1
.loop:
    imul rax, rdi
    dec rdi
    cmp rdi, 1
    jg  .loop
.done:
    ret

; Процедура: записать число (0..999) в out_buf
; Вход: rax = число
store_number:
    ; сотни
    mov rbx, 100
    xor rdx, rdx
    div rbx             ; rax = сотни, rdx = остаток
    add al, '0'
    mov [out_buf], al

    ; десятки
    mov rax, rdx
    xor rdx, rdx
    mov rbx, 10
    div rbx
    add al, '0'
    mov [out_buf+1], al

    ; единицы
    add dl, '0'
    mov [out_buf+2], dl
    ret

_start:
    mov rdi, 5          ; считаем 5!
    call factorial      ; rax = 120

    call store_number   ; записать 120 в out_buf

    mov rax, 1
    mov rdi, 1
    mov rsi, out_buf
    mov rdx, out_len
    syscall

    mov rax, 60
    xor rdi, rdi
    syscall
```

### Программа 7: call и ret — простые процедуры

```asm
; procedures.asm — структура с вызовами функций

section .data
    str1 db "Вызов первой процедуры", 10
    len1 equ $ - str1
    str2 db "Вызов второй процедуры", 10
    len2 equ $ - str2

section .text
    global _start

; Процедура: вывести строку
; Вход: rsi = адрес, rdx = длина
print_str:
    mov rax, 1
    mov rdi, 1
    syscall
    ret

; Процедура: выход
do_exit:
    mov rax, 60
    xor rdi, rdi
    syscall

_start:
    mov rsi, str1
    mov rdx, len1
    call print_str      ; вызвать процедуру

    mov rsi, str2
    mov rdx, len2
    call print_str

    call do_exit
```

---

## 15. Шпаргалка

### Основные инструкции одним взглядом

```asm
; ПЕРЕСЫЛКА
mov  dst, src       ; dst = src
movzx dst, src      ; dst = (unsigned)src  (с расширением нулями)
movsx dst, src      ; dst = (signed)src    (с расширением знака)
lea  dst, [expr]    ; dst = адрес выражения (без чтения памяти)
xchg dst, src       ; swap(dst, src)

; АРИФМЕТИКА
add  dst, src       ; dst += src
sub  dst, src       ; dst -= src
inc  dst            ; dst++
dec  dst            ; dst--
imul dst, src       ; dst *= src    (знаковое)
imul dst, src, imm  ; dst = src * imm
mul  src            ; rdx:rax = rax * src (беззнаковое)
idiv src            ; rax = rdx:rax / src, rdx = остаток (знаковое)
div  src            ; беззнаковое деление
neg  dst            ; dst = -dst
cqo                 ; sign-extend rax → rdx:rax  (перед idiv)

; ЛОГИКА И СДВИГИ
and  dst, src       ; dst &= src
or   dst, src       ; dst |= src
xor  dst, src       ; dst ^= src
not  dst            ; dst = ~dst
shl  dst, n         ; dst <<= n
shr  dst, n         ; dst >>= n  (беззнаковый)
sar  dst, n         ; dst >>= n  (знаковый, сохраняет знак)

; СРАВНЕНИЕ И ПЕРЕХОДЫ
cmp  a, b           ; установить флаги по a - b
test a, b           ; установить флаги по a & b
jmp  label          ; безусловный переход
je/jz   label       ; == 0
jne/jnz label       ; != 0
jl   label          ; <  (знаковое)
jle  label          ; <= (знаковое)
jg   label          ; >  (знаковое)
jge  label          ; >= (знаковое)
jb   label          ; <  (беззнаковое)
ja   label          ; >  (беззнаковое)

; СТЕК
push src            ; rsp -= 8; *rsp = src
pop  dst            ; dst = *rsp; rsp += 8

; ПРОЦЕДУРЫ
call label          ; push rip; jmp label
ret                 ; pop rip

; СИСТЕМНЫЙ ВЫЗОВ
syscall             ; вызов ядра Linux

; ИДИОМЫ
xor rax, rax        ; обнулить rax (быстрее mov rax, 0)
test rax, rax       ; проверить rax на ноль
loop label          ; dec rcx; jnz label
```

### Директивы NASM

```asm
db   42             ; 1 байт
dw   1000           ; 2 байта
dd   100000         ; 4 байта
dq   1000000000     ; 8 байт
resb N              ; зарезервировать N байт (в .bss)
equ                 ; символьная константа: LEN equ 10
$                   ; текущий адрес
global _start       ; сделать метку видимой для линковщика
```

### Команды сборки

```bash
nasm -f elf64 prog.asm -o prog.o   # ассемблировать
ld prog.o -o prog                   # слинковать
./prog                              # запустить
echo $?                             # код возврата (exit code)
objdump -d prog                     # дизассемблировать
strace ./prog                       # трассировка syscall
```

### Таблица системных вызовов (основные)

```
rax   Имя        rdi          rsi        rdx
───────────────────────────────────────────────
0     read        fd           buf        count
1     write       fd           buf        count
2     open        path         flags      mode
3     close       fd
39    getpid
57    fork
60    exit        code
```

---

*Архитектура: x86-64 Linux. Ассемблер: NASM. Для изучения ARM/AArch64
синтаксис и инструкции будут другими, но принципы работы со стеком,
регистрами и системными вызовами — те же.*



section .data
    num1 db 5
    num2 db 3
    result db 0
    newline db 10

section .bss
    output resb 2      ; место под строковое представление результата

section .text
    global _start

_start:
    ; Сложение
    mov al, [num1]
    add al, [num2]
    mov [result], al

    ; Преобразование числа в ASCII
    mov al, [result]
    add al, '0'        ; перевод в символ цифры
    mov [output], al
    mov byte [output+1], 10   ; перевод строки

    ; Вывод результата
    mov eax, 4         ; sys_write
    mov ebx, 1         ; stdout
    mov ecx, output    ; адрес строки
    mov edx, 2         ; длина (цифра + \n)
    int 0x80

    ; Выход
    mov eax, 1         ; sys_exit
    xor ebx, ebx
    int 0x80
