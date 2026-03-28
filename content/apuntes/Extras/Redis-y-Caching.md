# Redis — Caching y Estructuras de Datos

Redis es una base de datos en memoria key-value. Se usa para cachear respuestas de API, almacenar sesiones, implementar rate limiting, y como broker de colas de tareas.

---

## Instalación y conceptos básicos

```bash
# Levantar Redis con Docker (lo más rápido para desarrollo)
docker run -d -p 6379:6379 --name redis redis:7-alpine

# Con contraseña
docker run -d -p 6379:6379 --name redis redis:7-alpine redis-server --requirepass "mi_password"

# Conectar con redis-cli
docker exec -it redis redis-cli
# o con contraseña:
docker exec -it redis redis-cli -a mi_password

# En docker-compose:
# services:
#   redis:
#     image: redis:7-alpine
#     command: redis-server --requirepass mi_password
#     ports: ["6379:6379"]
```

### Tipos de datos principales

| Tipo                  | Caso de uso                                        |
| --------------------- | -------------------------------------------------- |
| **String**            | Cache de cualquier valor (JSON, contadores)        |
| **Hash**              | Objetos con campos (sesiones de usuario)           |
| **List**              | Cola de tareas, logs recientes                     |
| **Set**               | IDs únicos, tags                                   |
| **Sorted Set** (ZSet) | Leaderboards, tareas con prioridad                 |
| **Expire (TTL)**      | Se aplica a cualquier tipo — expiración automática |

---

## Redis con Go — `go-redis`

```bash
go get github.com/redis/go-redis/v9
```

### Conexión

```go
// infrastructure/cache/redis.go
package cache

import (
    "context"
    "time"
    "github.com/redis/go-redis/v9"
)

func NewRedisClient(addr, password string, db int) *redis.Client {
    return redis.NewClient(&redis.Options{
        Addr:         addr,       // "localhost:6379"
        Password:     password,
        DB:           db,         // 0 = default
        DialTimeout:  5 * time.Second,
        ReadTimeout:  3 * time.Second,
        WriteTimeout: 3 * time.Second,
        PoolSize:     10,
    })
}

// Verificar conexión
func Ping(ctx context.Context, rdb *redis.Client) error {
    return rdb.Ping(ctx).Err()
}
```

### Cache de respuestas de API (patrón cache-aside)

```go
// usecase/product_usecase.go
package usecase

import (
    "context"
    "encoding/json"
    "fmt"
    "time"
    "github.com/redis/go-redis/v9"
)

const productCacheTTL = 5 * time.Minute

func (u *ProductUsecase) GetProduct(ctx context.Context, id int) (*Product, error) {
    cacheKey := fmt.Sprintf("product:%d", id)

    // 1. Intentar cache
    cached, err := u.cache.Get(ctx, cacheKey).Bytes()
    if err == nil {
        var product Product
        if err := json.Unmarshal(cached, &product); err == nil {
            return &product, nil  // cache hit
        }
    }

    // 2. Cache miss — ir a la DB
    product, err := u.repo.GetByID(ctx, id)
    if err != nil {
        return nil, err
    }

    // 3. Guardar en cache
    data, err := json.Marshal(product)
    if err == nil {
        // SetNX = set if not exists, evita race conditions
        u.cache.Set(ctx, cacheKey, data, productCacheTTL)
    }

    return product, nil
}

// Invalidar cache cuando el producto se actualiza
func (u *ProductUsecase) UpdateProduct(ctx context.Context, id int, input UpdateInput) (*Product, error) {
    product, err := u.repo.Update(ctx, id, input)
    if err != nil {
        return nil, err
    }

    // Invalidar la entrada cacheada
    cacheKey := fmt.Sprintf("product:%d", id)
    u.cache.Del(ctx, cacheKey)

    return product, nil
}
```

### Rate limiting con Redis

```go
// middleware/rate_limit.go
func RateLimitMiddleware(rdb *redis.Client, limit int, window time.Duration) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            ctx := r.Context()
            ip := r.RemoteAddr
            key := fmt.Sprintf("rate_limit:%s", ip)

            // Incrementar contador con expiración
            count, err := rdb.Incr(ctx, key).Result()
            if err != nil {
                next.ServeHTTP(w, r)  // fallo silencioso en Redis → no bloquear
                return
            }

            if count == 1 {
                // Primera request en la ventana — setear TTL
                rdb.Expire(ctx, key, window)
            }

            if count > int64(limit) {
                w.Header().Set("Retry-After", fmt.Sprintf("%d", int(window.Seconds())))
                http.Error(w, `{"error":"rate_limit_exceeded"}`, http.StatusTooManyRequests)
                return
            }

            w.Header().Set("X-RateLimit-Remaining", fmt.Sprintf("%d", int64(limit)-count))
            next.ServeHTTP(w, r)
        })
    }
}
```

### Sesiones con Hash

```go
func (s *SessionStore) SaveSession(ctx context.Context, sessionID string, userID int, data map[string]string) error {
    key := fmt.Sprintf("session:%s", sessionID)

    fields := map[string]interface{}{
        "user_id": userID,
    }
    for k, v := range data {
        fields[k] = v
    }

    if err := s.rdb.HMSet(ctx, key, fields).Err(); err != nil {
        return err
    }
    return s.rdb.Expire(ctx, key, 24*time.Hour).Err()
}

func (s *SessionStore) GetSession(ctx context.Context, sessionID string) (map[string]string, error) {
    key := fmt.Sprintf("session:%s", sessionID)
    result, err := s.rdb.HGetAll(ctx, key).Result()
    if err != nil {
        return nil, err
    }
    if len(result) == 0 {
        return nil, ErrSessionNotFound
    }
    return result, nil
}

func (s *SessionStore) DeleteSession(ctx context.Context, sessionID string) error {
    return s.rdb.Del(ctx, fmt.Sprintf("session:%s", sessionID)).Err()
}
```

---

## Redis con Python — `redis-py`

```bash
uv add redis
# Para async:
uv add "redis[hiredis]"  # hiredis es el parser C más rápido
```

### Conexión

```python
# infrastructure/cache.py
import redis.asyncio as redis
from functools import lru_cache
from .config import settings

@lru_cache
def get_redis_client() -> redis.Redis:
    return redis.Redis(
        host=settings.redis_host,
        port=settings.redis_port,
        password=settings.redis_password,
        db=0,
        decode_responses=True,   # retorna str en lugar de bytes
        socket_timeout=3,
        socket_connect_timeout=5,
        health_check_interval=30,
    )
```

### Cache de respuestas con decorador

```python
# infrastructure/cache_decorator.py
import json
import functools
from typing import Callable, Any
import redis.asyncio as redis

def cached(ttl: int = 300, key_prefix: str = ""):
    """Decorador para cachear el resultado de una función async."""
    def decorator(func: Callable) -> Callable:
        @functools.wraps(func)
        async def wrapper(*args, **kwargs) -> Any:
            # Construir clave de cache a partir de los argumentos
            cache_key = f"{key_prefix or func.__name__}:{args}:{sorted(kwargs.items())}"

            rdb = get_redis_client()

            # Intentar cache
            cached_value = await rdb.get(cache_key)
            if cached_value:
                return json.loads(cached_value)

            # Cache miss
            result = await func(*args, **kwargs)

            # Guardar (permitir None — no cachear si es None)
            if result is not None:
                await rdb.setex(cache_key, ttl, json.dumps(result, default=str))

            return result

        # Exponer método para invalidar manualmente
        async def invalidate(*args, **kwargs):
            cache_key = f"{key_prefix or func.__name__}:{args}:{sorted(kwargs.items())}"
            await get_redis_client().delete(cache_key)

        wrapper.invalidate = invalidate
        return wrapper
    return decorator

# Uso
@cached(ttl=300, key_prefix="product")
async def get_product_by_id(product_id: int) -> dict | None:
    return await repo.get_by_id(product_id)
```

### Rate limiting con Python

```python
# middleware/rate_limit.py
from fastapi import Request, HTTPException, status
import redis.asyncio as redis
from .infrastructure.cache import get_redis_client

async def rate_limit_middleware(
    request: Request,
    limit: int = 60,
    window: int = 60,  # segundos
) -> None:
    """Dependencia de FastAPI para rate limiting."""
    rdb = get_redis_client()
    client_ip = request.client.host
    key = f"rate_limit:{client_ip}:{request.url.path}"

    # Pipeline para atomicidad
    async with rdb.pipeline(transaction=True) as pipe:
        try:
            await pipe.incr(key)
            await pipe.ttl(key)
            count, ttl = await pipe.execute()

            if ttl == -1:  # no tiene expiración (primera vez)
                await rdb.expire(key, window)

            if count > limit:
                raise HTTPException(
                    status_code=status.HTTP_429_TOO_MANY_REQUESTS,
                    detail="Demasiadas solicitudes. Intentá de nuevo en un momento.",
                    headers={"Retry-After": str(window)},
                )
        except redis.RedisError:
            pass  # Redis caído → no bloquear requests

# En el router o como dependencia global
from fastapi import Depends
from functools import partial

rate_limit_strict = partial(rate_limit_middleware, limit=10, window=60)

@router.post("/auth/login")
async def login(
    credentials: LoginRequest,
    _: None = Depends(rate_limit_strict),  # 10 intentos por minuto
):
    ...
```

### Pub/Sub — notificaciones en tiempo real

```python
# Con Redis Pub/Sub para notificar a workers
async def publish_event(channel: str, event: dict) -> None:
    rdb = get_redis_client()
    await rdb.publish(channel, json.dumps(event))

# Subscriber (en un worker separado)
async def listen_for_events(channel: str):
    rdb = get_redis_client()
    async with rdb.pubsub() as pubsub:
        await pubsub.subscribe(channel)
        async for message in pubsub.listen():
            if message["type"] == "message":
                event = json.loads(message["data"])
                await handle_event(event)
```

---

## Patrones de cache

### Cache-aside (el más común)

```
Leer:    App → Redis (hit? → retornar) → DB → guardar en Redis → retornar
Escribir: App → DB → invalidar Redis (DEL)
```

### Write-through

```
Escribir: App → DB + Redis al mismo tiempo
Leer: App → Redis (siempre tiene el valor más reciente)
Riesgo: si DB falla, Redis y DB quedan desincronizados
```

### TTL — qué tiempo elegir

| Tipo de dato                        | TTL sugerido           |
| ----------------------------------- | ---------------------- |
| Datos que cambian poco (categorías) | 1 hora - 24 horas      |
| Perfil de usuario                   | 5-15 minutos           |
| Resultado de API externa            | 1-5 minutos            |
| Rate limit counter                  | Duración de la ventana |
| Sesión de usuario                   | 24 horas - 7 días      |
| Token de verificación de email      | 24 horas               |

### Cache stampede — problema y solución

```python
# PROBLEMA: miles de requests llegan cuando el cache expira → todas van a DB
# SOLUCIÓN: probabilistic early expiration

import random
import time

async def get_with_jitter(key: str, fetch_fn, ttl: int):
    """Renueva el cache un poco antes de que expire para evitar stampede."""
    rdb = get_redis_client()

    # Guardar con metadata
    raw = await rdb.get(f"{key}:meta")
    if raw:
        meta = json.loads(raw)
        # Si quedan menos del 10% del TTL, renovar con 10% de probabilidad
        remaining = meta["expires_at"] - time.time()
        if remaining < ttl * 0.1 and random.random() < 0.1:
            raw = None  # forzar renovación

    if raw:
        return json.loads(await rdb.get(key))

    # Fetch y guardar
    value = await fetch_fn()
    expires_at = time.time() + ttl
    await rdb.setex(key, ttl, json.dumps(value))
    await rdb.setex(f"{key}:meta", ttl, json.dumps({"expires_at": expires_at}))
    return value
```
