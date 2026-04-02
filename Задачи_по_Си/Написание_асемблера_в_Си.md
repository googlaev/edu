# Встроенный ассемблер в языке Си
### Полный курс с примерами кода

---

## Содержание

1. [Зачем нужна вставка ассемблера](#1-зачем-нужна-вставка-ассемблера)
2. [Два синтаксиса: AT&T и Intel](#2-два-синтаксиса-att-и-intel)
3. [GCC Extended Inline Assembly](#3-gcc-extended-inline-assembly)
4. [Ограничения операндов (constraints)](#4-ограничения-операндов-constraints)
5. [Затираемые регистры (clobbers)](#5-затираемые-регистры-clobbers)
6. [Примеры: от простого к сложному](#6-примеры-от-простого-к-сложному)
7. [MSVC: блочный `__asm`](#7-msvc-блочный-__asm)
8. [Именованные операнды](#8-именованные-операнды)
9. [Барьеры памяти и синхронизация](#9-барьеры-памяти-и-синхронизация)
10. [Типичные ошибки](#10-типичные-ошибки)
11. [Когда НЕ использовать inline asm](#11-когда-не-использовать-inline-asm)
12. [Шпаргалка](#12-шпаргалка)

---

## 1. Зачем нужна вставка ассемблера

Си компилируется в машинный код автоматически. Но бывают ситуации, когда компилятор не справляется или не имеет доступа к нужным инструкциям:

| Сценарий | Пример |
|----------|--------|
| Критическая оптимизация | Ручной SIMD-цикл быстрее автовекторизации |
| Привилегированные инструкции | `hlt`, `cli`, `sti` — только в ядре ОС |
| Специальные регистры | Чтение `cr0`, `msr`, `cpuid` |
| Счётчик тактов процессора | `rdtsc` — точное профилирование |
| Порты ввода/вывода | `in`/`out` — работа с железом напрямую |
| Атомарные операции | `lock xchg`, `lock cmpxchg` |

> **Правило:** Используйте inline asm только тогда, когда компилятор объективно
> не может сгенерировать нужный код или делает это значительно хуже.

---

## 2. Два синтаксиса: AT&T и Intel

В мире x86 существуют два несовместимых синтаксиса.

### AT&T (GNU, Linux, GCC по умолчанию)

```asm
movl %eax, %ebx     # ebx = eax  (источник слева, приёмник справа)
addl $5,   %eax     # eax += 5   (константы с $)
movl 8(%rbp), %eax  # eax = *(rbp + 8)  (смещение в скобках)
```

- Регистры пишутся с `%`: `%eax`, `%rbx`
- Константы пишутся с `$`: `$5`, `$0xFF`
- Мнемоники имеют суффикс размера: `b` (byte), `w` (word), `l` (long/dword), `q` (qword)
- **Источник → Приёмник** (слева направо)

### Intel (MSVC, NASM, Windows)

```asm
mov ebx, eax        ; ebx = eax  (приёмник слева, источник справа)
add eax, 5          ; eax += 5   (константы без знаков)
mov eax, [rbp + 8]  ; eax = *(rbp + 8)
```

- Регистры и константы без префиксов
- Размер через `DWORD PTR`, `QWORD PTR`
- **Приёмник ← Источник** (справа налево)

### Переключение синтаксиса в GCC

```bash
# Компилировать с Intel-синтаксисом в выходном коде
gcc -masm=intel -O2 -o prog prog.c

# Посмотреть сгенерированный asm с Intel-синтаксисом
gcc -S -masm=intel -O2 prog.c
```

---

## 3. GCC Extended Inline Assembly

Расширенный встроенный ассемблер GCC — основной механизм для GCC и Clang.

### Полная форма

```c
asm [volatile] (
    "инструкции"              /* строка с кодом              */
    : выходные_операнды       /* output constraints (опц.)   */
    : входные_операнды        /* input  constraints (опц.)   */
    : затираемые_ресурсы      /* clobbers           (опц.)   */
    : goto_метки              /* только для asm goto (редко) */
);
```

### Правила записи инструкций

```c
asm volatile (
    "movl $0, %%eax\n\t"   /* \n\t обязателен — разделитель строк */
    "addl %1,  %%eax\n\t"  /* %% — экранированный % для регистра  */
    "movl %%eax, %0"        /* %0, %1 — ссылки на операнды        */
    : "=r"(result)
    : "r"(value)
    : "%eax"
);
```

- `\n\t` — перевод строки + табуляция (для читаемого листинга)
- `%%reg` — двойной `%` нужен, потому что одиночный `%N` — ссылка на операнд
- `%0`, `%1`, `%2` — позиционные ссылки на операнды в порядке: сначала выходные, потом входные

### Ключевое слово `volatile`

```c
/* БЕЗ volatile — компилятор МОЖЕТ удалить блок! */
asm ("rdtsc" : "=a"(lo), "=d"(hi));   /* не безопасно */

/* С volatile — блок всегда выполнится */
asm volatile ("rdtsc" : "=a"(lo), "=d"(hi));   /* правильно */
```

`volatile` запрещает:
- удаление блока (если компилятор считает выходы неиспользуемыми)
- перестановку блока с другими инструкциями
- дублирование блока (например, при разворачивании циклов)

> **Практическое правило:** Всегда ставьте `volatile`, если блок имеет
> побочные эффекты — изменяет флаги, обращается к портам, меняет память
> не через операнды.

---

## 4. Ограничения операндов (constraints)

Ограничения говорят компилятору, **где** хранить операнд.

### Основные буквы ограничений

| Буква | Где хранится | Пример |
|-------|-------------|--------|
| `r` | Любой регистр общего назначения | `"r"(x)` |
| `m` | Ячейка памяти | `"m"(arr[i])` |
| `i` | Непосредственная константа (compile-time) | `"i"(42)` |
| `g` | Регистр, память или константа | `"g"(x)` |
| `a` | Конкретно `eax`/`rax` | `"a"(x)` |
| `b` | Конкретно `ebx`/`rbx` | `"b"(x)` |
| `c` | Конкретно `ecx`/`rcx` | `"c"(x)` |
| `d` | Конкретно `edx`/`rdx` | `"d"(x)` |
| `S` | Конкретно `esi`/`rsi` | `"S"(x)` |
| `D` | Конкретно `edi`/`rdi` | `"D"(x)` |
| `N` | 8-битная беззнаковая константа (0–255) | `"N"(port)` |
| `n` | Числовая константа (compile-time) | `"n"(x)` |

### Модификаторы ограничений

| Символ | Смысл |
|--------|-------|
| `=` | Операнд только пишется (выходной) |
| `+` | Операнд читается И пишется |
| `&` | Early-clobber: записывается до чтения входов |
| `%` | Операнд коммутативен со следующим |

### Совмещение операндов (`"0"`, `"1"` ...)

```c
/* %0 сначала содержит newval, потом будет перезаписан — xchg */
asm volatile (
    "xchgl %0, %1"
    : "=r"(old), "+m"(*ptr)
    : "0"(newval)          /* "0" = тот же регистр, что и %0 */
);
```

---

## 5. Затираемые регистры (clobbers)

Если asm-блок меняет регистры или память, не объявленные в операндах, их нужно перечислить в clobbers.

### Три вида clobbers

```c
asm volatile (
    "cpuid"
    : "=a"(eax), "=b"(ebx), "=c"(ecx), "=d"(edx)
    : "a"(leaf)
    /* clobbers не нужны — все изменённые рег. объявлены как выходные */
);
```

```c
asm volatile (
    "call some_function"   /* call затирает rax, rcx, rdx, r8-r11 */
    :
    :
    : "rax", "rcx", "rdx", "r8", "r9", "r10", "r11", "memory"
);
```

### Специальные clobbers

```c
/* "memory" — программный барьер памяти */
asm volatile ("mfence" ::: "memory");
/* Компилятор обязан перечитать ВСЕ переменные из памяти после этой строки */

/* "cc" — блок меняет флаги (EFLAGS/RFLAGS) */
asm volatile ("addl %1, %0" : "+r"(a) : "r"(b) : "cc");
/* На x86 "cc" подразумевается для большинства арифметических инструкций */
```

---

## 6. Примеры: от простого к сложному

### 6.1 Сложение двух целых чисел

```c
#include <stdio.h>

int add_asm(int a, int b) {
    int result;
    asm volatile (
        "addl %2, %1\n\t"    /* %1 (a) += %2 (b) */
        "movl %1, %0"        /* result = %1       */
        : "=r"(result)       /* %0 — выходной     */
        : "r"(a), "r"(b)    /* %1 = a, %2 = b    */
    );
    return result;
}

int main(void) {
    printf("3 + 4 = %d\n", add_asm(3, 4));   /* 7 */
    return 0;
}
```

### 6.2 Инкремент переменной в памяти

```c
int x = 41;

/* Вариант 1: прямо в памяти */
asm volatile ("incl %0" : "+m"(x));
printf("x = %d\n", x);   /* 42 */

/* Вариант 2: через регистр */
asm volatile (
    "movl %1, %%eax\n\t"
    "incl %%eax\n\t"
    "movl %%eax, %0"
    : "=m"(x)
    : "m"(x)
    : "%eax"
);
```

### 6.3 Умножение 32-бит → 64-бит результат

```c
#include <stdint.h>

uint64_t mul32to64(uint32_t a, uint32_t b) {
    uint32_t lo, hi;
    asm volatile (
        "mull %3"            /* eax * %3 → edx:eax */
        : "=a"(lo), "=d"(hi)
        : "a"(a), "r"(b)
    );
    return (uint64_t)hi << 32 | lo;
}
```

### 6.4 Чтение счётчика тактов (RDTSC)

```c
#include <stdint.h>

uint64_t rdtsc(void) {
    uint32_t lo, hi;
    asm volatile (
        "rdtsc"
        : "=a"(lo), "=d"(hi)   /* eax = lo, edx = hi */
    );
    return (uint64_t)hi << 32 | lo;
}

/* Более строгий вариант с сериализацией */
uint64_t rdtsc_serialized(void) {
    uint32_t lo, hi;
    asm volatile (
        "cpuid\n\t"    /* сериализация — дождаться завершения предыдущих инструкций */
        "rdtsc"
        : "=a"(lo), "=d"(hi)
        :
        : "%rbx", "%rcx"   /* cpuid портит rbx и rcx */
    );
    return (uint64_t)hi << 32 | lo;
}

/* Пример использования: измерение времени */
void benchmark(void) {
    uint64_t start = rdtsc();
    /* ... измеряемый код ... */
    uint64_t end   = rdtsc();
    printf("Тактов: %llu\n", (unsigned long long)(end - start));
}
```

### 6.5 Порты ввода/вывода (только ring 0 — ядро ОС)

```c
#include <stdint.h>

/* Чтение байта из порта */
static inline uint8_t inb(uint16_t port) {
    uint8_t val;
    asm volatile (
        "inb %1, %0"
        : "=a"(val)       /* результат в al */
        : "Nd"(port)      /* N = 8-бит константа, d = dx */
    );
    return val;
}

/* Запись байта в порт */
static inline void outb(uint16_t port, uint8_t val) {
    asm volatile (
        "outb %0, %1"
        :
        : "a"(val), "Nd"(port)
    );
}

/* Чтение 16-бит слова */
static inline uint16_t inw(uint16_t port) {
    uint16_t val;
    asm volatile ("inw %1, %0" : "=a"(val) : "Nd"(port));
    return val;
}

/* Чтение 32-бит двойного слова */
static inline uint32_t inl(uint16_t port) {
    uint32_t val;
    asm volatile ("inl %1, %0" : "=a"(val) : "Nd"(port));
    return val;
}

/* Пример: чтение статуса клавиатуры */
void example_kbd(void) {
    uint8_t status = inb(0x64);   /* порт контроллера клавиатуры */
    if (status & 0x01)
        printf("Данные от клавиатуры готовы\n");
}
```

### 6.6 Атомарный обмен (xchg)

```c
#include <stdint.h>

/* xchg неявно добавляет LOCK — самостоятельно атомарна */
static inline int atomic_xchg(volatile int *ptr, int newval) {
    int old;
    asm volatile (
        "xchgl %0, %1"
        : "=r"(old), "+m"(*ptr)
        : "0"(newval)
        : "memory"
    );
    return old;
}

/* Атомарное сравнение и обмен (CAS) */
static inline int atomic_cmpxchg(volatile int *ptr,
                                   int expected, int desired) {
    int old;
    asm volatile (
        "lock cmpxchgl %2, %1"
        : "=a"(old), "+m"(*ptr)
        : "r"(desired), "0"(expected)
        : "memory"
    );
    return old;
}

/* Атомарное сложение, возвращает старое значение */
static inline int atomic_fetch_add(volatile int *ptr, int val) {
    asm volatile (
        "lock xaddl %0, %1"
        : "+r"(val), "+m"(*ptr)
        :
        : "memory"
    );
    return val;   /* val теперь содержит СТАРОЕ значение *ptr */
}
```

### 6.7 CPUID — идентификация процессора

```c
#include <stdint.h>
#include <string.h>
#include <stdio.h>

typedef struct {
    uint32_t eax, ebx, ecx, edx;
} CpuidResult;

CpuidResult cpuid(uint32_t leaf, uint32_t subleaf) {
    CpuidResult r;
    asm volatile (
        "cpuid"
        : "=a"(r.eax), "=b"(r.ebx), "=c"(r.ecx), "=d"(r.edx)
        : "a"(leaf),   "c"(subleaf)
    );
    return r;
}

void print_cpu_vendor(void) {
    CpuidResult r = cpuid(0, 0);
    char vendor[13];
    memcpy(vendor + 0, &r.ebx, 4);
    memcpy(vendor + 4, &r.edx, 4);
    memcpy(vendor + 8, &r.ecx, 4);
    vendor[12] = '\0';
    printf("Производитель CPU: %s\n", vendor);
    /* GenuineIntel или AuthenticAMD */
}

void check_sse2(void) {
    CpuidResult r = cpuid(1, 0);
    int has_sse2 = (r.edx >> 26) & 1;
    printf("SSE2: %s\n", has_sse2 ? "есть" : "нет");
}
```

### 6.8 Простой SIMD-пример (SSE2)

```c
#include <stdint.h>

/* Сложение 4 пар float одной инструкцией */
void sse2_add_floats(float *a, const float *b, int n) {
    int i = 0;
    /* Обрабатываем по 4 элемента за итерацию */
    for (; i + 4 <= n; i += 4) {
        asm volatile (
            "movups (%1),    %%xmm0\n\t"   /* загрузить a[i..i+3] */
            "movups (%2),    %%xmm1\n\t"   /* загрузить b[i..i+3] */
            "addps  %%xmm1,  %%xmm0\n\t"   /* xmm0 += xmm1        */
            "movups %%xmm0,  (%1)"          /* сохранить результат  */
            :
            : "r"(a + i), "r"(a + i), "r"(b + i)
            : "xmm0", "xmm1", "memory"
        );
    }
    /* Оставшиеся элементы — скалярно */
    for (; i < n; i++)
        a[i] += b[i];
}
```

> **Замечание:** На практике для SIMD лучше использовать intrinsics
> (`#include <immintrin.h>`), которые переносимее и читаемее.

### 6.9 Управление флагами прерываний (ядро ОС)

```c
/* Запретить прерывания (cli) */
static inline void local_irq_disable(void) {
    asm volatile ("cli" ::: "memory");
}

/* Разрешить прерывания (sti) */
static inline void local_irq_enable(void) {
    asm volatile ("sti" ::: "memory");
}

/* Сохранить флаги и запретить прерывания */
static inline unsigned long local_irq_save(void) {
    unsigned long flags;
    asm volatile (
        "pushfq\n\t"
        "popq  %0\n\t"
        "cli"
        : "=r"(flags)
        :
        : "memory"
    );
    return flags;
}

/* Восстановить сохранённые флаги */
static inline void local_irq_restore(unsigned long flags) {
    asm volatile (
        "pushq %0\n\t"
        "popfq"
        :
        : "r"(flags)
        : "memory", "cc"
    );
}
```

### 6.10 Чтение управляющего регистра CR0

```c
#include <stdint.h>

/* Только в режиме ядра! */
static inline uint64_t read_cr0(void) {
    uint64_t val;
    asm volatile ("movq %%cr0, %0" : "=r"(val));
    return val;
}

static inline void write_cr0(uint64_t val) {
    asm volatile ("movq %0, %%cr0" : : "r"(val));
}

/* Пример: проверить бит защиты (PE) */
void check_protected_mode(void) {
    uint64_t cr0 = read_cr0();
    if (cr0 & 1)
        printf("Процессор в защищённом режиме\n");
}
```

---

## 7. MSVC: блочный `__asm`

На 32-битном x86 MSVC поддерживает блочный синтаксис с Intel-нотацией.

```c
/* Только MSVC, только x86-32 */

int add_msvc(int a, int b) {
    __asm {
        mov eax, a    ; переменные Си доступны по имени
        add eax, b    ; результат в eax = возвращаемое значение
    }
    /* return не нужен — eax возвращается автоматически */
}

void swap_msvc(int *a, int *b) {
    __asm {
        mov  eax, a
        mov  ecx, b
        mov  edx, [eax]
        xchg edx, [ecx]
        mov  [eax], edx
    }
}
```

### Ограничения MSVC

- Только x86-32. На x86-64 встроенного asm нет вовсе.
- Для x86-64 под MSVC нужен отдельный `.asm`-файл + компилятор MASM.
- Нет аналога constraints — управление регистрами полностью ручное.

### Отдельный asm-файл для MSVC x64

```asm
; myasm.asm — компилировать: ml64 /c myasm.asm
PUBLIC my_add
.CODE
my_add PROC
    ; rcx = a, rdx = b (соглашение __fastcall / Microsoft ABI)
    lea  rax, [rcx + rdx]
    ret
my_add ENDP
END
```

```c
/* Объявление в Си */
extern long long my_add(long long a, long long b);
```

---

## 8. Именованные операнды

Вместо позиционных `%0`, `%1` можно давать операндам имена — код читается нагляднее.

```c
int multiply(int a, int b) {
    int result;
    asm volatile (
        "imull %[src], %[dst]"
        : [dst] "=r"(result)   /* выходной операнд, имя dst */
        : [src]  "r"(b),       /* входной операнд, имя src  */
                 "0"(a)        /* result начинает жизнь = a */
    );
    return result;
}
```

```c
/* Ещё один пример: побитовое AND */
unsigned mask_bits(unsigned val, unsigned mask) {
    unsigned out;
    asm volatile (
        "andl %[mask], %[val]"
        : [val] "=r"(out)
        : [mask] "r"(mask), "0"(val)
    );
    return out;
}
```

---

## 9. Барьеры памяти и синхронизация

### Зачем нужны барьеры

Компилятор и процессор оба могут менять порядок инструкций. Барьеры запрещают это.

```c
/* Компиляторный барьер — только запрещает перестановку в C-коде */
static inline void compiler_barrier(void) {
    asm volatile ("" ::: "memory");
}

/* Барьер записи: все предыдущие store видны до последующих */
static inline void store_barrier(void) {
    asm volatile ("sfence" ::: "memory");
}

/* Барьер чтения: все предыдущие load завершены до последующих */
static inline void load_barrier(void) {
    asm volatile ("lfence" ::: "memory");
}

/* Полный барьер памяти */
static inline void memory_barrier(void) {
    asm volatile ("mfence" ::: "memory");
}
```

### Пример: спин-блокировка

```c
#include <stdint.h>

typedef volatile int spinlock_t;

static inline void spin_lock(spinlock_t *lock) {
    int tmp = 1;
    asm volatile (
        "1:\n\t"
        "xchgl %0, %1\n\t"    /* атомарный обмен */
        "testl %0, %0\n\t"    /* проверить старое значение */
        "jnz   1b"            /* если было 1 — ждать       */
        : "=r"(tmp), "+m"(*lock)
        : "0"(tmp)
        : "memory", "cc"
    );
}

static inline void spin_unlock(spinlock_t *lock) {
    asm volatile (
        "movl $0, %0"
        : "=m"(*lock)
        :
        : "memory"
    );
}
```

### Пауза в цикле ожидания

```c
/* pause — подсказка процессору: мы в спин-петле */
/* Снижает энергопотребление, помогает HyperThreading */
static inline void cpu_relax(void) {
    asm volatile ("pause" ::: "memory");
}

/* Правильный спин-wait */
void wait_for_flag(volatile int *flag) {
    while (!*flag)
        cpu_relax();
}
```

---

## 10. Типичные ошибки

### Ошибка 1: Забыть `volatile`

```c
/* НЕПРАВИЛЬНО — компилятор может удалить этот блок при оптимизации */
uint64_t t;
uint32_t lo, hi;
asm ("rdtsc" : "=a"(lo), "=d"(hi));
t = (uint64_t)hi << 32 | lo;

/* ПРАВИЛЬНО */
asm volatile ("rdtsc" : "=a"(lo), "=d"(hi));
```

### Ошибка 2: Перепутать направление AT&T

```c
/* НЕПРАВИЛЬНО: в AT&T movl A, B означает B = A */
asm ("movl %%eax, %%ebx");   /* ebx = eax */

/* Если хотели eax = ebx — надо наоборот: */
asm ("movl %%ebx, %%eax");
```

### Ошибка 3: Забыть clobber для "memory"

```c
int arr[4] = {1, 2, 3, 4};
int sum = 0;

/* НЕПРАВИЛЬНО: компилятор не знает, что asm меняет arr */
asm volatile (
    "addl (%1), %0\n\t"
    "addl 4(%1), %0"
    : "+r"(sum)
    : "r"(arr)
    /* нет "memory" — компилятор может не перечитать arr */
);

/* ПРАВИЛЬНО */
asm volatile (
    "addl (%1), %0\n\t"
    "addl 4(%1), %0"
    : "+r"(sum)
    : "r"(arr)
    : "memory"           /* теперь компилятор знает о чтении памяти */
);
```

### Ошибка 4: Не указать затёртый регистр

```c
/* НЕПРАВИЛЬНО: div использует edx, но он не объявлен */
asm volatile (
    "xorl %%edx, %%edx\n\t"   /* edx портится */
    "divl %1"
    : "=a"(quotient)
    : "r"(divisor), "a"(dividend)
    /* нет "%edx" в clobbers! */
);

/* ПРАВИЛЬНО */
asm volatile (
    "xorl %%edx, %%edx\n\t"
    "divl %2"
    : "=a"(quotient), "=d"(remainder)   /* edx объявлен как выходной */
    : "r"(divisor), "a"(dividend)
);
```

### Ошибка 5: Ожидать переносимость

```c
/* Это работает только на x86-64 с GCC/Clang */
static inline uint64_t rdtsc(void) {
    uint32_t lo, hi;
    asm volatile ("rdtsc" : "=a"(lo), "=d"(hi));
    return (uint64_t)hi << 32 | lo;
}

/* ПРАВИЛЬНО: обернуть в #ifdef */
#if defined(__GNUC__) && (defined(__x86_64__) || defined(__i386__))
static inline uint64_t rdtsc(void) {
    uint32_t lo, hi;
    asm volatile ("rdtsc" : "=a"(lo), "=d"(hi));
    return (uint64_t)hi << 32 | lo;
}
#elif defined(__aarch64__)
static inline uint64_t rdtsc(void) {
    uint64_t val;
    asm volatile ("mrs %0, cntvct_el0" : "=r"(val));
    return val;
}
#else
#   include <time.h>
static inline uint64_t rdtsc(void) {
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    return (uint64_t)ts.tv_sec * 1000000000ULL + ts.tv_nsec;
}
#endif
```

---

## 11. Когда НЕ использовать inline asm

### Компилятор справляется сам

```c
/* Это GCC превратит в единственный addl */
int sum(int a, int b) { return a + b; }

/* Это тоже оптимизируется отлично */
int clz(unsigned x) { return __builtin_clz(x); }   /* bsr/lzcnt */
```

### Используйте intrinsics для SIMD

```c
/* Вместо asm SSE: */
asm ("addps %%xmm1, %%xmm0" ...);

/* Гораздо лучше — intrinsics, переносимее и читаемее: */
#include <immintrin.h>
__m128 a = _mm_load_ps(pa);
__m128 b = _mm_load_ps(pb);
__m128 c = _mm_add_ps(a, b);
_mm_store_ps(pc, c);
```

### Используйте `<stdatomic.h>` вместо asm-атомик

```c
/* Вместо: */
asm volatile ("lock xaddl %0, %1" ...);

/* Лучше: */
#include <stdatomic.h>
atomic_int counter = 0;
atomic_fetch_add(&counter, 1);
```

### Используйте `__builtin_*` для аппаратных операций

```c
/* Паузу в цикле ожидания: */
__builtin_ia32_pause();          /* вместо asm("pause") */

/* Подсчёт единичных бит: */
__builtin_popcount(x);           /* вместо asm("popcnt") */

/* Ведущие нули: */
__builtin_clz(x);                /* вместо asm("bsr") */
```

---

## 12. Шпаргалка

### Быстрая ссылка по синтаксису GCC

```
asm volatile (
    "инструкция %0, %1\n\t"
    "инструкция %[имя]"
    : "=r"(out1), "+m"(inout),   [имя] "=r"(out2)   // outputs
    : "r"(in1),    "m"(in2),     "0"(совместить_с_out1)  // inputs
    : "rax", "memory", "cc"      // clobbers
);
```

### Таблица: когда что использовать

| Задача | Рекомендация |
|--------|-------------|
| Сложение, умножение, логика | Компилятор сам |
| SIMD-вычисления | Intrinsics (`<immintrin.h>`) |
| Атомарные операции | `<stdatomic.h>` |
| `rdtsc`, `cpuid` | Inline asm или `__builtin_*` |
| Порты ввода/вывода | Inline asm (только ядро) |
| Управление прерываниями | Inline asm (только ядро) |
| Спин-блокировки | Inline asm или `<stdatomic.h>` |
| Барьеры памяти | Inline asm или `atomic_thread_fence` |

### Минимальный рабочий шаблон

```c
#include <stdint.h>

#if defined(__GNUC__) && defined(__x86_64__)

static inline uint64_t read_tsc(void) {
    uint32_t lo, hi;
    asm volatile ("rdtsc" : "=a"(lo), "=d"(hi));
    return (uint64_t)hi << 32 | lo;
}

static inline void cpu_pause(void) {
    asm volatile ("pause" ::: "memory");
}

static inline void full_barrier(void) {
    asm volatile ("mfence" ::: "memory");
}

#endif /* __GNUC__ && __x86_64__ */
```

---

*Лекция охватывает x86/x86-64. Для ARM (AArch64) синтаксис аналогичен
(GCC Extended Inline ASM), но инструкции и ограничения регистров отличаются.*
