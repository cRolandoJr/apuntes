# Microservicios y System Design

El tema más importante en entrevistas de backend senior y la base del trabajo en empresas grandes. Entender cuándo usar microservicios (y más importante: cuándo NO usarlos).

---

## Monolito vs Microservicios — la decisión real

### Monolito bien estructurado

```
cmd/
└── main.go

internal/
├── domain/          ← entities + interfaces
├── product/         ← bounded context de productos
│   ├── repository.go
│   ├── usecase.go
│   └── handler.go
├── user/            ← bounded context de usuarios
│   ├── repository.go
│   ├── usecase.go
│   └── handler.go
├── order/           ← bounded context de órdenes
│   └── ...
└── infra/
    ├── postgres/
    └── http/
```

Un monolito modular bien separado puede soportar cientos de miles de usuarios y equipos de 5-20 ingenieros sin problemas.

### Cuándo migrar a microservicios

| Señal                                                   | Descripción                                           |
| ------------------------------------------------------- | ----------------------------------------------------- |
| Equipos independientes necesitan deploys independientes | 3+ equipos pisan código del otro constantemente       |
| Escalado diferencial                                    | Módulo de búsqueda necesita 50 instancias, el resto 2 |
| Diferente tecnología requerida                          | ML en Python, CRUD en Go, streaming en Kafka          |
| SLA diferencial                                         | Pagos necesita 99.99%, catálogo puede tener downtime  |
| El monolito es demasiado lento para cambiar             | Deploy de 2 horas para cambiar un campo               |

**No migres porque "es lo que hacen las empresas grandes"** — Netflix, Amazon, Google empezaron con monolitos.

---

## Bounded Contexts — DDD básico

Un bounded context es un límite lógico donde un término tiene un significado específico.

```
Contexto de Catálogo:
  Product = { id, name, description, price, stock }

Contexto de Órdenes:
  Product = { productId, name, price }  ← solo lo que le importa a órdenes

Contexto de Envío:
  Product = { productId, weight, dimensions }  ← solo lo que importa para envío
```

Cada bounded context puede ser un microservicio — o un módulo dentro de un monolito.

---

## Comunicación entre servicios

### Síncrona — gRPC

Para comunicación request/response entre servicios internos. Más eficiente que REST (Protobuf binario, HTTP/2, multiplexing).

```protobuf
// product.proto
syntax = "proto3";
package product.v1;
option go_package = "gestion_productos/proto/product/v1";

service ProductService {
  rpc GetProduct(GetProductRequest) returns (GetProductResponse);
  rpc ListProducts(ListProductsRequest) returns (ListProductsResponse);
}

message GetProductRequest {
  string id = 1;
}

message GetProductResponse {
  Product product = 1;
}

message Product {
  string id = 1;
  string name = 2;
  double price = 3;
  int32 stock = 4;
}
```

```bash
# Generar código Go desde .proto
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

protoc --go_out=. --go-grpc_out=. proto/product.proto
```

```go
// Servidor gRPC
type productServer struct {
    pb.UnimplementedProductServiceServer
    uc domain.ProductUseCase
}

func (s *productServer) GetProduct(ctx context.Context, req *pb.GetProductRequest) (*pb.GetProductResponse, error) {
    product, err := s.uc.GetByID(ctx, req.Id)
    if err != nil {
        return nil, status.Errorf(codes.NotFound, "producto no encontrado: %v", err)
    }
    return &pb.GetProductResponse{
        Product: &pb.Product{
            Id:    product.ID,
            Name:  product.Name,
            Price: product.Price,
            Stock: int32(product.Stock),
        },
    }, nil
}

// Cliente gRPC
conn, err := grpc.Dial("product-service:50051", grpc.WithTransportCredentials(insecure.NewCredentials()))
client := pb.NewProductServiceClient(conn)
resp, err := client.GetProduct(ctx, &pb.GetProductRequest{Id: "product-id"})
```

### Asíncrona — mensajes con Kafka/RabbitMQ

Para comunicación donde no necesitás respuesta inmediata (email, notificaciones, analytics).

```go
// Publicar evento cuando se crea un producto
type ProductCreatedEvent struct {
    ProductID string    `json:"productId"`
    Name      string    `json:"name"`
    Price     float64   `json:"price"`
    CreatedAt time.Time `json:"createdAt"`
}

// En el usecase
func (uc *productUseCase) Create(ctx context.Context, input domain.CreateProductInput) (*domain.Product, error) {
    product, err := uc.repo.Create(ctx, &domain.Product{...})
    if err != nil {
        return nil, err
    }

    // Publicar evento — los servicios interesados (analytics, email) lo consumen
    event := ProductCreatedEvent{
        ProductID: product.ID,
        Name:      product.Name,
        Price:     product.Price,
        CreatedAt: product.CreatedAt,
    }

    // Publicación asíncrona para no bloquear el request
    go func() {
        if err := uc.eventBus.Publish(ctx, "product.created", event); err != nil {
            // Loguear pero no fallar — el evento puede reintentar
            log.FromContext(ctx).Error("publicar evento product.created", "error", err)
        }
    }()

    return product, nil
}
```

---

## Patrones claves en microservicios

### Circuit Breaker

Protege contra cascadas de fallos. Si un servicio está fallando, deja de llamarlo temporalmente.

```go
import "github.com/sony/gobreaker"

cb := gobreaker.NewCircuitBreaker(gobreaker.Settings{
    Name:        "product-service",
    MaxRequests: 5,   // requests permitidos en estado half-open
    Interval:    10 * time.Second,
    Timeout:     30 * time.Second, // tiempo en open antes de intentar de nuevo
    ReadyToTrip: func(counts gobreaker.Counts) bool {
        // Abrir el circuit si >50% de los últimos 10 requests fallaron
        failureRatio := float64(counts.TotalFailures) / float64(counts.Requests)
        return counts.Requests >= 10 && failureRatio >= 0.5
    },
})

result, err := cb.Execute(func() (interface{}, error) {
    return productClient.GetProduct(ctx, req)
})
```

### Idempotency — operaciones seguras de reintentar

```go
// Cada mutación acepta un idempotency key
// Si el cliente reintenta con el mismo key, retorna el mismo resultado

func (uc *orderUseCase) Create(ctx context.Context, input CreateOrderInput, idempotencyKey string) (*Order, error) {
    // Verificar si ya procesamos este key
    if existing, err := uc.idempotencyRepo.Get(ctx, idempotencyKey); err == nil {
        return existing, nil // ya fue procesado, retornar el resultado anterior
    }

    order, err := uc.processOrder(ctx, input)
    if err != nil {
        return nil, err
    }

    // Guardar resultado para futuros reintentos
    _ = uc.idempotencyRepo.Save(ctx, idempotencyKey, order, 24*time.Hour)

    return order, nil
}
```

---

## System Design — preguntas frecuentes de entrevistas

### Framework para responder

```
1. Clarificar requisitos (5 min)
   - ¿Cuántos usuarios? ¿QPS? ¿Datos?
   - ¿Lectura o escritura intensiva?
   - ¿Consistencia fuerte o eventual?

2. Estimaciones de escala (2 min)
   - 1M usuarios × 10 requests/día = 115 QPS (baja carga)
   - 100M DAU × 100 requests/día = 115K QPS (alta carga)
   - 1KB por request × 115K QPS = 115MB/s de datos

3. API (5 min)
   - Definir endpoints principales

4. Diseño de datos (10 min)
   - Entidades y relaciones
   - ¿SQL o NoSQL? ¿Por qué?

5. Arquitectura de alto nivel (15 min)
   - Load balancer → API Gateway → Servicios → DB
   - Caché, CDN, cola de mensajes

6. Deep dive en componentes críticos (10 min)
   - El que el entrevistador pida

7. Identificar problemas y trade-offs (5 min)
   - Single points of failure
   - Bottlenecks
   - ¿Qué sacrificamos?
```

### Números para memorizar

| Operación             | Latencia aproximada |
| --------------------- | ------------------- |
| L1 cache hit          | 0.5 ns              |
| L2 cache hit          | 7 ns                |
| RAM access            | 100 ns              |
| SSD read              | 150 µs              |
| HDD seek              | 10 ms               |
| Network (mismo DC)    | 0.5 ms              |
| Network (EEUU↔Europa) | 150 ms              |

---

## Práctica: Novato vs Profesional

### Novato

```
"Necesito escalar mi app → voy a hacer microservicios"
Sin considerar la complejidad operacional:
- Service discovery
- Distributed tracing
- Eventual consistency
- Network partitions
- Testing de integración entre servicios
```

### Profesional

```
"Empiezo con monolito modular bien estructurado.
Los bounded contexts ya están definidos como módulos.
Si necesito escalar un módulo independientemente,
lo extraigo a un servicio — pero el código ya está preparado
porque la separación de responsabilidades estaba bien hecha."

Regla: si no tenés 3+ equipos trabajando en el mismo repo
y tenés <10M usuarios, un monolito bien escrito es la respuesta correcta.
```
