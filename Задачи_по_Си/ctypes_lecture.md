# Лекция: ctypes — вызов C из Python

## Что такое ctypes?

`ctypes` — стандартная библиотека Python, которая позволяет вызывать функции
из обычных C-библиотек (`.so` на Linux, `.dll` на Windows).

**Главное преимущество:** не нужно писать никакого Python C API кода.
Пишешь чистый C, компилируешь, вызываешь из Python.

---

## Установка

`ctypes` входит в стандартную библиотеку Python — устанавливать ничего не нужно.

```python
import ctypes  # просто импортируй — и всё готово
```

> ⚠️ Частая ошибка новичков:
> ```python
> import crypes   # ❌ ModuleNotFoundError
> import ctypes   # ✅ правильно
> ```

---

## Пример 1 — Простые функции (сложение, квадрат)

### mylib.c
```c
int add(int a, int b) {
    return a + b;
}

double square(double x) {
    return x * x;
}
```

### Компиляция
```bash
gcc -shared -fPIC -o mylib.so mylib.c
```

### main.py
```python
import ctypes

lib = ctypes.CDLL("./mylib.so")

# Указываем типы — Python иначе не знает, что передавать
lib.add.argtypes = [ctypes.c_int, ctypes.c_int]
lib.add.restype  = ctypes.c_int

lib.square.argtypes = [ctypes.c_double]
lib.square.restype  = ctypes.c_double

print(lib.add(3, 4))      # 7
print(lib.square(2.5))    # 6.25
```

**Зачем указывать argtypes и restype?**
Без этого ctypes не знает типы и может передать неправильные байты.
Например, `double` без `restype` вернёт мусор вместо числа.

---

## Пример 2 — Строки

### mylib.c
```c
#include <string.h>
#include <stdio.h>

void greet(const char *name) {
    printf("Привет, %s!\n", name);
}

int count_chars(const char *str) {
    return (int)strlen(str);
}
```

### main.py
```python
import ctypes

lib = ctypes.CDLL("./mylib.so")

lib.greet.argtypes = [ctypes.c_char_p]
lib.greet.restype  = None  # void

lib.count_chars.argtypes = [ctypes.c_char_p]
lib.count_chars.restype  = ctypes.c_int

# Строки передаются как bytes, не str!
lib.greet(b"Alice")               # Привет, Alice!
print(lib.count_chars(b"hello"))  # 5
```

> ⚠️ Строки в ctypes — это `bytes`, не `str`.
> Если есть обычная строка, конвертируй: `"hello".encode()`

---

## Пример 3 — Массивы

### mylib.c
```c
// Сумма элементов массива
int array_sum(int *arr, int n) {
    int sum = 0;
    for (int i = 0; i < n; i++)
        sum += arr[i];
    return sum;
}

// Умножить каждый элемент на множитель (изменяет массив на месте)
void array_scale(double *arr, int n, double factor) {
    for (int i = 0; i < n; i++)
        arr[i] *= factor;
}
```

### main.py
```python
import ctypes

lib = ctypes.CDLL("./mylib.so")

lib.array_sum.argtypes = [ctypes.POINTER(ctypes.c_int), ctypes.c_int]
lib.array_sum.restype  = ctypes.c_int

lib.array_scale.argtypes = [ctypes.POINTER(ctypes.c_double),
                             ctypes.c_int, ctypes.c_double]
lib.array_scale.restype  = None

# Создаём C-массив из Python-списка
arr = (ctypes.c_int * 5)(10, 20, 30, 40, 50)
print(lib.array_sum(arr, 5))  # 150

# Массив double, изменяется на месте
data = (ctypes.c_double * 4)(1.0, 2.0, 3.0, 4.0)
lib.array_scale(data, 4, 2.5)
print(list(data))  # [2.5, 5.0, 7.5, 10.0]
```

---

## Пример 4 — Структуры

### mylib.c
```c
#include <math.h>

typedef struct {
    double x;
    double y;
} Point;

double distance(Point a, Point b) {
    double dx = a.x - b.x;
    double dy = a.y - b.y;
    return sqrt(dx*dx + dy*dy);
}

Point midpoint(Point a, Point b) {
    Point m;
    m.x = (a.x + b.x) / 2.0;
    m.y = (a.y + b.y) / 2.0;
    return m;
}
```

```bash
gcc -shared -fPIC -o mylib.so mylib.c -lm
```

### main.py
```python
import ctypes

class Point(ctypes.Structure):
    _fields_ = [("x", ctypes.c_double),
                ("y", ctypes.c_double)]

lib = ctypes.CDLL("./mylib.so")

lib.distance.argtypes = [Point, Point]
lib.distance.restype  = ctypes.c_double

lib.midpoint.argtypes = [Point, Point]
lib.midpoint.restype  = Point

a = Point(0.0, 0.0)
b = Point(3.0, 4.0)

print(lib.distance(a, b))           # 5.0

m = lib.midpoint(a, b)
print(f"Середина: ({m.x}, {m.y})")  # Середина: (1.5, 2.0)
```

---

## Пример 5 — Указатели (возврат значения через параметр)

В C часто возвращают несколько значений через указатели.

### mylib.c
```c
// Возвращает min и max через указатели
void minmax(int *arr, int n, int *out_min, int *out_max) {
    *out_min = arr[0];
    *out_max = arr[0];
    for (int i = 1; i < n; i++) {
        if (arr[i] < *out_min) *out_min = arr[i];
        if (arr[i] > *out_max) *out_max = arr[i];
    }
}
```

### main.py
```python
import ctypes

lib = ctypes.CDLL("./mylib.so")

lib.minmax.argtypes = [ctypes.POINTER(ctypes.c_int), ctypes.c_int,
                       ctypes.POINTER(ctypes.c_int),
                       ctypes.POINTER(ctypes.c_int)]
lib.minmax.restype  = None

arr = (ctypes.c_int * 6)(4, 1, 9, 2, 7, 3)

mn = ctypes.c_int()
mx = ctypes.c_int()

lib.minmax(arr, 6, ctypes.byref(mn), ctypes.byref(mx))

print(f"min={mn.value}, max={mx.value}")  # min=1, max=9
```

`ctypes.byref(x)` — передаёт указатель на переменную `x`, как `&x` в C.

---

## Таблица типов C → ctypes

| C-тип         | ctypes              |
|---------------|---------------------|
| `int`         | `c_int`             |
| `long`        | `c_long`            |
| `double`      | `c_double`          |
| `float`       | `c_float`           |
| `char *`      | `c_char_p`          |
| `void *`      | `c_void_p`          |
| `void`        | `None` (restype)    |
| `int *`       | `POINTER(c_int)`    |
| `int[5]`      | `c_int * 5`         |

---

## Итог — алгоритм работы с ctypes

```
1. Пишешь обычный C-код (без Python.h)
         ↓
2. Компилируешь: gcc -shared -fPIC -o mylib.so mylib.c
         ↓
3. В Python: lib = ctypes.CDLL("./mylib.so")
         ↓
4. Указываешь типы: lib.func.argtypes = [...], lib.func.restype = ...
         ↓
5. Вызываешь как обычную Python-функцию
```

Это всё. Никакого `setup.py`, никакого `PyInit_*`, никакого управления памятью вручную.
