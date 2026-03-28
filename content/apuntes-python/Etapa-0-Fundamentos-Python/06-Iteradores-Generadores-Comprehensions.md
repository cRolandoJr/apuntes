# Iteradores, Generadores y Comprehensions

Las comprehensions y los generadores son el corazón del Python idiomático. Los generadores permiten procesar colecciones de millones de elementos con memoria constante.

---

## Comprehensions

### List comprehension

```python
# Forma básica: [expresión for elemento in iterable if condición]
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# filtrar y transformar
evens_squared = [x**2 for x in numbers if x % 2 == 0]
# [4, 16, 36, 64, 100]

# sin comprehension (peor)
evens_squared = []
for x in numbers:
    if x % 2 == 0:
        evens_squared.append(x**2)
```

### Dict comprehension

```python
products = [
    {"id": 1, "name": "Laptop", "price": 1500},
    {"id": 2, "name": "Mouse", "price": 30},
    {"id": 3, "name": "Monitor", "price": 400},
]

# Índice rápido id → producto
by_id = {p["id"]: p for p in products}
# {1: {"id": 1, ...}, 2: {...}, 3: {...}}

# Transformar valores
prices_by_name = {p["name"]: p["price"] for p in products}
# {"Laptop": 1500, "Mouse": 30, "Monitor": 400}
```

### Set comprehension

```python
# Valores únicos
names = ["Alice", "Bob", "Alice", "Charlie", "Bob"]
unique_names = {name.lower() for name in names}
# {"alice", "bob", "charlie"}
```

### Generator expression — sin corchetes, con paréntesis

```python
# Genera valores de a uno — no crea la lista completa en memoria
gen = (x**2 for x in range(1_000_000))

# sum/max/min aceptan generadores directamente
total = sum(x**2 for x in range(1_000_000))   # sin [], crea generador
maximum = max(len(line) for line in open("log.txt"))

# Cuándo usar generator expression vs list comprehension:
# - generator: necesitás iterar UNA vez, o la colección es grande
# - list: necesitás indexar, iterar múltiples veces, len(), etc.
result = list(x**2 for x in range(10))  # si igual necesitás la lista
```

---

## El protocolo Iterator

Python usa _duck typing_ para iterables. Cualquier objeto con `__iter__` y `__next__` es un iterador.

```python
# Cómo funciona for internamente:
for item in [1, 2, 3]:
    print(item)

# Equivale a:
it = iter([1, 2, 3])      # llama __iter__()
while True:
    try:
        item = next(it)   # llama __next__()
        print(item)
    except StopIteration:
        break
```

### Crear un iterador propio

```python
class Countdown:
    def __init__(self, start: int) -> None:
        self.current = start

    def __iter__(self) -> "Countdown":
        return self  # el objeto mismo es el iterador

    def __next__(self) -> int:
        if self.current <= 0:
            raise StopIteration
        value = self.current
        self.current -= 1
        return value

for n in Countdown(5):
    print(n)  # 5, 4, 3, 2, 1
```

---

## Generadores con `yield`

Un generador es una función que usa `yield`. Cuando se llama, retorna un objeto generador sin ejecutar el cuerpo. El cuerpo se ejecuta de a pasos, un `yield` a la vez.

```python
def fibonacci():
    """Generador infinito de Fibonacci."""
    a, b = 0, 1
    while True:
        yield a
        a, b = b, a + b

fib = fibonacci()
print(next(fib))  # 0
print(next(fib))  # 1
print(next(fib))  # 1
print(next(fib))  # 2

# Con itertools.islice para tomar los primeros N
from itertools import islice
first_10 = list(islice(fibonacci(), 10))
# [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]

# Generador que lee un archivo línea a línea (memoria constante)
def read_log_lines(path: str):
    with open(path) as f:
        for line in f:
            line = line.strip()
            if line:
                yield line

# Procesar archivo de 10GB sin cargarlo en memoria
for line in read_log_lines("/var/log/nginx/access.log"):
    if "ERROR" in line:
        print(line)
```

### `yield from` — delegar en otro generador

```python
def gen_a():
    yield 1
    yield 2

def gen_b():
    yield 3
    yield 4

# Sin yield from
def combined_bad():
    for item in gen_a():
        yield item
    for item in gen_b():
        yield item

# Con yield from (preferido)
def combined():
    yield from gen_a()
    yield from gen_b()
    yield from range(5, 8)

list(combined())  # [1, 2, 3, 4, 5, 6, 7]

# yield from es esencial para generadores recursivos
def flatten(nested):
    for item in nested:
        if isinstance(item, list):
            yield from flatten(item)  # recursión
        else:
            yield item

list(flatten([1, [2, [3, 4]], 5]))  # [1, 2, 3, 4, 5]
```

---

## `itertools` — la caja de herramientas

```python
import itertools

# chain — concatenar iterables
from itertools import chain
combined = list(chain([1, 2], [3, 4], [5]))
# [1, 2, 3, 4, 5]

# chain.from_iterable — aplanar un nivel
nested = [[1, 2], [3, 4], [5, 6]]
flat = list(chain.from_iterable(nested))
# [1, 2, 3, 4, 5, 6]

# islice — slice sobre un generador
from itertools import islice
first_5 = list(islice(fibonacci(), 5))  # [0, 1, 1, 2, 3]

# groupby — agrupar elementos consecutivos iguales
from itertools import groupby
data = [("A", 1), ("A", 2), ("B", 3), ("B", 4), ("A", 5)]
# Importante: groupby requiere que esté ORDENADO por la clave
for key, group in groupby(data, key=lambda x: x[0]):
    print(key, list(group))
# A [('A', 1), ('A', 2)]
# B [('B', 3), ('B', 4)]
# A [('A', 5)]

# product — producto cartesiano (como nested for loops)
from itertools import product
for x, y in product([1, 2], ["a", "b"]):
    print(x, y)
# 1 a | 1 b | 2 a | 2 b

# combinations y permutations
from itertools import combinations, permutations
list(combinations([1, 2, 3], 2))  # [(1,2), (1,3), (2,3)]
list(permutations([1, 2, 3], 2))  # [(1,2), (1,3), (2,1), (2,3), ...]

# takewhile / dropwhile
from itertools import takewhile, dropwhile
list(takewhile(lambda x: x < 5, [1, 2, 3, 5, 4, 6]))
# [1, 2, 3]  ← se detiene cuando falla la condición

# count — contador infinito
from itertools import count
for i, item in zip(count(1), ["a", "b", "c"]):
    print(i, item)  # 1 a | 2 b | 3 c

# batched (Python 3.12+) — dividir en chunks
from itertools import batched
list(batched([1,2,3,4,5,6,7], 3))  # [(1,2,3), (4,5,6), (7,)]
```

---

## map, filter, reduce — y cuándo NO usarlos

```python
from functools import reduce

numbers = [1, 2, 3, 4, 5]

# map — aplicar función a cada elemento
squared_map = list(map(lambda x: x**2, numbers))
squared_comp = [x**2 for x in numbers]  # más legible en Python

# filter — filtrar elementos
evens_filter = list(filter(lambda x: x % 2 == 0, numbers))
evens_comp = [x for x in numbers if x % 2 == 0]  # más legible

# reduce — acumular valor (functools)
total = reduce(lambda acc, x: acc + x, numbers, 0)
total = sum(numbers)  # sum es siempre mejor que reduce para sumar

# Cuándo SÍ usar map/filter:
# - map: cuando tenés una función ya definida (sin lambda)
from pathlib import Path
paths = ["/home/user/.bashrc", "/etc/nginx/nginx.conf"]
path_objects = list(map(Path, paths))  # limpio, sin lambda

# - filter: con función predefinida
def is_valid(email: str) -> bool: ...
valid = list(filter(is_valid, emails))
```

---

## Generadores en pipeline — procesamiento en streaming

```python
# Pipeline de procesamiento de logs sin cargar todo en memoria
def parse_log_line(line: str) -> dict | None:
    """Parsea una línea de log, retorna None si no es válida."""
    parts = line.split()
    if len(parts) < 4:
        return None
    return {
        "ip": parts[0],
        "method": parts[5].strip('"'),
        "path": parts[6],
        "status": int(parts[8]),
    }

def read_lines(path: str):
    with open(path) as f:
        yield from (line.strip() for line in f if line.strip())

def parse_lines(lines):
    for line in lines:
        parsed = parse_log_line(line)
        if parsed:
            yield parsed

def filter_errors(entries):
    return (e for e in entries if e["status"] >= 400)

# Componer el pipeline — nada se ejecuta hasta que iteras
lines = read_lines("/var/log/nginx/access.log")
parsed = parse_lines(lines)
errors = filter_errors(parsed)

# Solo aquí se procesa de a una línea a la vez
error_count = sum(1 for _ in errors)
print(f"Total errores: {error_count}")
```

---

## Práctica: Novato vs Profesional

### Novato

```python
# Cargar todo en memoria innecesariamente
def get_all_errors(log_path):
    lines = open(log_path).readlines()          # todo en memoria
    parsed = [parse(l) for l in lines]           # otra lista completa
    errors = [e for e in parsed if e["status"] >= 400]  # otra lista
    return errors  # 3x el tamaño del log en memoria

# map con lambda cuando hay comprehension más clara
result = list(map(lambda x: x * 2, items))
```

### Profesional

```python
# Pipeline lazy — memoria constante independientemente del tamaño del archivo
def get_all_errors(log_path: str):
    lines = read_lines(log_path)       # generador
    parsed = parse_lines(lines)        # generador
    return (e for e in parsed if e["status"] >= 400)  # generador

# Comprehension cuando es más legible
result = [x * 2 for x in items]

# Generator expression cuando solo se necesita iterar una vez
total = sum(x * 2 for x in items)

# yield from para generadores recursivos y delegación
def all_files(directory: Path):
    for item in directory.iterdir():
        if item.is_dir():
            yield from all_files(item)
        else:
            yield item
```
