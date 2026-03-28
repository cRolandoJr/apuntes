# Recursos Recomendados

Lista curada de recursos para escalar de junior a senior en Go y backend. Calidad sobre cantidad.

---

## Libros — leer en este orden

### Nivel 1 — Fundamentos

**The Go Programming Language** — Donovan & Kernighan

- El libro definitivo del lenguaje. Escrito por un autor del lenguaje.
- Leer completo, hacer los ejercicios.
- PDF disponible legalmente online.

**Go in Action** — Kennedy, Ketelsen, Martin

- Más práctico que el anterior. Buenos ejemplos del mundo real.
- Complementa bien al Donovan.

### Nivel 2 — Intermedio-Avanzado

**100 Go Mistakes and How to Avoid Them** — Teiva Harsanyi

- El libro más recomendado por la comunidad Go actualmente.
- Cubre exactamente los errores que cometés en código real.
- Disponible en Manning.

**Concurrency in Go** — Katherine Cox-Buday

- Deep dive en goroutines, channels, Context, patrones de concurrencia.
- Indispensable cuando trabajás en sistemas concurrentes.

### Nivel 3 — Arquitectura

**Clean Architecture** — Robert C. Martin (Uncle Bob)

- La base teórica de lo que aplicás en `gestion_productos`.
- Agnóstico al lenguaje, los principios aplican a Go.

**Designing Data-Intensive Applications** — Martin Kleppmann

- El mejor libro de ingeniería de software de la última década.
- Cubre: bases de datos, replicación, particionamiento, consistencia, Kafka.
- Esencial para entrevistas de sistema design.

**Database Internals** — Alex Petrov

- Cómo funcionan las bases de datos por dentro.
- Útil para entender por qué ciertos patrones funcionan.

---

## Blogs y sitios — leer regularmente

**go.dev/blog** — blog oficial del equipo de Go

- Anuncios de nuevas versiones, artículos técnicos.
- Especialmente: "Go Memory Model", "Share Memory By Communicating"

**Dave Cheney** — dave.cheney.net

- Uno de los mayores contribuyentes al ecosistema Go.
- Artículos sobre error handling, concurrencia, interfaces.
- Must-read: "Practical Go", "Errors are values"

**Ardan Labs Blog** — ardanlabs.com/blog

- Material de nivel avanzado, escrito por los autores de "Go in Action".
- Series completas sobre concurrencia, memory model, profiling.

**The Gopher Academy Blog** — blog.gopheracademy.com

- Advent of Go cada diciembre (artículos de la comunidad).
- Buenas prácticas y casos de uso reales.

---

## Canales de YouTube

**Nic Jackson (HashiCorp)** — Tutoriales de Go + microservicios en Go.
**Anthony GG** — Proyectos completos en Go, muy práctico.
**Fireship** — Videos cortos de conceptos, bueno para repaso rápido.
**Hussein Nasser** — System design y backend fundamentals, excelente.
**TechWorld with Nana** — Kubernetes, Docker, DevOps práctico.

---

## Repositorios — estudiar el código

**github.com/avelino/awesome-go** — Lista curada de librerías Go.
**github.com/golang/go** — El código fuente del lenguaje.
**github.com/gin-gonic/gin** — Ver cómo está construido un framework popular.
**github.com/gofiber/fiber** — Framework inspirado en Express.js.
**github.com/99designs/gqlgen** — El código de gqlgen, útil para entender cómo funciona.

---

## Herramientas — instalar ahora

```bash
# Linter — indispensable en todo proyecto
go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest

# Hot reload para desarrollo
go install github.com/air-verse/air@latest

# Generar mocks
go install go.uber.org/mock/mockgen@latest

# Formateo de código (ya incluido en Go)
gofmt -w .

# Ver todo el árbol de dependencias
go mod graph

# Actualizar dependencias
go get -u ./...
go mod tidy

# Profiling — ya incluido
go tool pprof http://localhost:6060/debug/pprof/heap
```

---

## Boot.dev — orden recomendado para Rolando

```
Prioridad inmediata (antes del trabajo):
1. Learn Go → completar 100%
2. Build a Social Media Backend → primer proyecto real

Post trabajo (primeros 3 meses):
3. Learn HTTP Clients and Servers
4. Learn SQL
5. Build a Blog Aggregator
6. Learn Docker
7. Learn CI/CD

Después:
8. Learn Kubernetes
9. Learn gRPC
```

---

## Recursos gratuitos online

**go.dev/tour** — Tour interactivo oficial, 76 lecciones.
**gobyexample.com** — Ejemplos de cada feature del lenguaje.
**exercism.io/tracks/go** — 130+ ejercicios con mentores real.
**leetcode.com** — Algoritmos (filtrar por Go).

---

## Comunidades

**gophers.slack.com** — El Slack oficial de la comunidad Go.
**reddit.com/r/golang** — Noticias y preguntas.
**github.com/golang/go/issues** — Issues del lenguaje (para entender decisiones de diseño).

---

## Lista de conceptos para dominar antes de tu primera entrevista

- [ ] Interfaces implícitas y dónde viven
- [ ] Value vs pointer receivers (y cuándo usar cada uno)
- [ ] Goroutines + channels (sin data races)
- [ ] Context propagation (cancelación + timeout)
- [ ] Error wrapping con %w y errors.Is/As
- [ ] Clean Architecture (las 4 capas y la Dependency Rule)
- [ ] SQL: SELECT/JOIN/INDEX/transacciones
- [ ] Docker multi-stage build
- [ ] Table-driven tests con testify
- [ ] JWT authentication middleware
- [ ] HTTP middleware chain
- [ ] Los 4 Golden Signals (Prometheus)
