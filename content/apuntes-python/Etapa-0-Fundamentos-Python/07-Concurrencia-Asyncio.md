# Concurrencia en Python: asyncio, threading y multiprocessing

Python tiene el GIL (Global Interpreter Lock) que limita un proceso a ejecutar código Python en un solo hilo a la vez. Esto cambia drásticamente cómo se maneja la concurrencia comparado con Go.

---

## El GIL — qué es y qué implica

```
GIL = Global Interpreter Lock

Un solo hilo ejecuta código Python a la vez en CPython.
Libera el GIL solo para I/O (red, disco) y operaciones C nativas.

Consecuencia:
- threading: útil para I/O-bound (web requests, queries de DB)
- threading: INÚTIL para CPU-bound (cálculos matemáticos)
- multiprocessing: necesario para CPU-bound (evita el GIL)
- asyncio: ideal para I/O-bound concurrente (un solo hilo)

Python 3.13 tiene GIL opcional (experimental) — no productivo aún.
```

---

## ¿Cuándo usar qué?

| Operación                                | Recomendación                        |
| ---------------------------------------- | ------------------------------------ |
| Web API (FastAPI)                        | asyncio (async/await)                |
| Múltiples requests HTTP a APIs externas  | asyncio + httpx                      |
| Queries de DB (PostgreSQL, MySQL)        | asyncio + asyncpg / SQLAlchemy async |
| Scripts SysAdmin (I/O archivos, disk)    | asyncio o threading                  |
| Procesamiento de imágenes, pandas, numpy | multiprocessing                      |
| Web scraping (muchas URLs)               | asyncio + httpx o aiohttp            |
| Subprocess en paralelo                   | asyncio subprocess                   |

---

## asyncio — el modelo de concurrencia principal

asyncio usa un **event loop** (bucle de eventos). Hay un solo hilo, pero puede tener miles de "tareas" (corrutinas) esperando I/O sin bloquear.

### async/await básico

```python
import asyncio

# coroutine — función que puede pausarse con await
async def fetch_user(user_id: int) -> dict:
    # await pausa ESTA coroutina y deja que otras corran
    await asyncio.sleep(1)  # simula I/O (red, DB)
    return {"id": user_id, "name": f"User {user_id}"}

# Para llamar una coroutine, hay que awaiterla o crear una Task
async def main():
    user = await fetch_user(1)
    print(user)

# Punto de entrada
asyncio.run(main())  # crea el event loop, corre main(), lo cierra
```

### asyncio.gather — ejecutar coroutines en paralelo

```python
import asyncio
import httpx

async def fetch_product(client: httpx.AsyncClient, product_id: int) -> dict:
    response = await client.get(f"https://api.example.com/products/{product_id}")
    response.raise_for_status()
    return response.json()

async def main():
    async with httpx.AsyncClient() as client:
        # Ejecutar 5 requests EN PARALELO (no secuencial)
        results = await asyncio.gather(
            fetch_product(client, 1),
            fetch_product(client, 2),
            fetch_product(client, 3),
            fetch_product(client, 4),
            fetch_product(client, 5),
        )
    # results es una lista con los resultados en orden
    for product in results:
        print(product)

asyncio.run(main())
```

### asyncio.gather con lista dinámica

```python
async def fetch_many(ids: list[int]) -> list[dict]:
    async with httpx.AsyncClient(timeout=10.0) as client:
        tasks = [fetch_product(client, id) for id in ids]
        results = await asyncio.gather(*tasks, return_exceptions=True)

    # Si return_exceptions=True, los errores no propagan, están en results
    products = []
    for r in results:
        if isinstance(r, Exception):
            print(f"Error: {r}")
        else:
            products.append(r)
    return products
```

### Tasks — scheduling explícito

```python
async def background_cleanup():
    """Se ejecuta en segundo plano."""
    while True:
        await asyncio.sleep(60)
        await purge_expired_sessions()

async def main():
    # Crear task — se ejecuta "en segundo plano" dentro del event loop
    cleanup_task = asyncio.create_task(background_cleanup())

    # Hacer otras cosas...
    await start_server()

    # Cancelar cuando ya no se necesite
    cleanup_task.cancel()
    try:
        await cleanup_task
    except asyncio.CancelledError:
        pass  # esperado
```

---

## Async context managers y async iterators

```python
# Async context manager — para resources async (conexiones, locks)
async def get_user(user_id: int) -> User:
    async with get_db_session() as session:  # async with
        result = await session.execute(
            select(User).where(User.id == user_id)
        )
        return result.scalar_one_or_none()

# Async for — para iterar sobre fuentes de datos async
async def stream_logs():
    async with aiofiles.open("app.log") as f:
        async for line in f:   # async for
            yield line.strip()

async def process_logs():
    async for line in stream_logs():  # async for en generador async
        if "ERROR" in line:
            await alert(line)
```

---

## asyncio en FastAPI

En FastAPI, cualquier path operation puede ser `async def` o `def`:

```python
from fastapi import FastAPI, Depends
from sqlalchemy.ext.asyncio import AsyncSession

app = FastAPI()

# async def — corre en el event loop (ideal para I/O)
@app.get("/products/{id}")
async def get_product(id: int, session: AsyncSession = Depends(get_session)):
    product = await session.get(Product, id)
    if not product:
        raise HTTPException(status_code=404)
    return product

# def — FastAPI lo corre en un thread pool (para código síncrono bloqueante)
@app.get("/health")
def health_check():
    return {"status": "ok"}  # no hace I/O, def está bien

# NUNCA mezclar — no hacer I/O bloqueante en async def
@app.get("/bad")
async def bad_endpoint():
    import time
    time.sleep(1)     # MAL — bloquea el event loop entero
    return {}

@app.get("/good")
async def good_endpoint():
    await asyncio.sleep(1)  # BIEN — pausa sin bloquear
    return {}
```

---

## Timeout y cancellation

```python
import asyncio

async def slow_operation():
    await asyncio.sleep(30)
    return "done"

async def main():
    # Timeout con asyncio.wait_for
    try:
        result = await asyncio.wait_for(slow_operation(), timeout=5.0)
    except asyncio.TimeoutError:
        print("La operación tardó más de 5 segundos")

    # Python 3.11+: asyncio.timeout (context manager)
    try:
        async with asyncio.timeout(5.0):
            result = await slow_operation()
    except TimeoutError:
        print("Timeout")
```

---

## threading — para I/O legacy síncrono

```python
import threading
from concurrent.futures import ThreadPoolExecutor
import requests  # librería síncrona — si no podés usar httpx async

def fetch_url(url: str) -> str:
    response = requests.get(url, timeout=10)
    return response.text

urls = [
    "https://api.example.com/1",
    "https://api.example.com/2",
    "https://api.example.com/3",
]

# ThreadPoolExecutor — manejo automático del pool
with ThreadPoolExecutor(max_workers=10) as executor:
    results = list(executor.map(fetch_url, urls))

# submit — para tareas individuales con manejo de errores
with ThreadPoolExecutor(max_workers=5) as executor:
    futures = [executor.submit(fetch_url, url) for url in urls]
    for future in futures:
        try:
            result = future.result(timeout=15)
            print(result[:100])
        except Exception as e:
            print(f"Error: {e}")
```

### Locks para datos compartidos

```python
import threading

class Counter:
    def __init__(self):
        self._value = 0
        self._lock = threading.Lock()

    def increment(self):
        with self._lock:  # context manager — adquiere y libera automáticamente
            self._value += 1

    @property
    def value(self):
        with self._lock:
            return self._value
```

---

## multiprocessing — para CPU-bound

```python
from concurrent.futures import ProcessPoolExecutor
from multiprocessing import Pool
import numpy as np

def process_chunk(data: list[float]) -> float:
    """Operación costosa en CPU."""
    arr = np.array(data)
    return float(np.std(arr) * np.mean(arr))

data = list(range(1_000_000))
chunk_size = 100_000
chunks = [data[i:i+chunk_size] for i in range(0, len(data), chunk_size)]

# ProcessPoolExecutor — interfaz similar a ThreadPoolExecutor
with ProcessPoolExecutor(max_workers=4) as executor:
    results = list(executor.map(process_chunk, chunks))

total = sum(results)
print(f"Resultado: {total}")

# IMPORTANTE: los procesos no comparten memoria
# Los argumentos y resultados se serializan (pickle)
# Solo pasar tipos simples: int, float, str, list, dict
```

---

## asyncio.to_thread — correr código síncrono sin bloquear

```python
import asyncio

def read_config_file(path: str) -> dict:
    """Función síncrona bloqueante."""
    import json
    with open(path) as f:
        return json.load(f)

async def main():
    # Corre la función bloqueante en un thread pool sin bloquear el event loop
    config = await asyncio.to_thread(read_config_file, "/etc/app/config.json")
    print(config)
```

---

## Semaphore — limitar concurrencia

```python
import asyncio
import httpx

async def fetch_limited(
    semaphore: asyncio.Semaphore,
    client: httpx.AsyncClient,
    url: str
) -> dict:
    async with semaphore:  # máximo N requests simultáneos
        response = await client.get(url)
        return response.json()

async def fetch_all(urls: list[str], max_concurrent: int = 10):
    semaphore = asyncio.Semaphore(max_concurrent)
    async with httpx.AsyncClient() as client:
        tasks = [fetch_limited(semaphore, client, url) for url in urls]
        return await asyncio.gather(*tasks)
```

---

## Práctica: Novato vs Profesional

### Novato

```python
# I/O síncrono y secuencial en async def — bloquea el event loop
async def get_users():
    users = []
    for id in range(100):
        # requests es síncrono — bloquea el event loop completo
        r = requests.get(f"https://api.com/users/{id}")
        users.append(r.json())
    return users  # 100 requests secuenciales = lentísimo

# Crear event loop manualmente (incorrecto en producción)
loop = asyncio.get_event_loop()  # deprecado en 3.10+
loop.run_until_complete(main())
```

### Profesional

```python
# Async nativo + concurrencia real con semaphore para no sobrecargar la API
async def get_users(ids: list[int]) -> list[dict]:
    semaphore = asyncio.Semaphore(20)  # máx 20 requests simultáneos

    async def fetch_one(client: httpx.AsyncClient, id: int) -> dict:
        async with semaphore:
            r = await client.get(f"https://api.com/users/{id}")
            r.raise_for_status()
            return r.json()

    async with httpx.AsyncClient(timeout=10.0) as client:
        tasks = [fetch_one(client, id) for id in ids]
        results = await asyncio.gather(*tasks, return_exceptions=True)

    return [r for r in results if not isinstance(r, Exception)]

# Punto de entrada correcto
asyncio.run(main())
```
