# Ruta de Aprendizaje Go — Backend Profesional

Este vault cubre Go desde cero hasta nivel profesional en equipos reales.
Cada archivo tiene sección "novato" (cómo lo hace alguien que recién aprende) y "profesional" (cómo se hace en producción con Clean Architecture).

---

## Cómo usar este vault

1. Seguí el orden de las Etapas.
2. En cada archivo: primero leé el concepto, luego el código novato, luego el profesional.
3. Reproducí los ejemplos en el proyecto `gestion_productos`.
4. Los archivos de Etapa-1 en adelante se retoman muchas veces — están diseñados como referencia permanente.

---

## Etapa 0 — Fundamentos del Lenguaje

| Archivo                     | Qué cubre                                                             |
| --------------------------- | --------------------------------------------------------------------- |
| [[01-Tipos-y-Variables]]    | Tipos primitivos, custom types, iota, zero values                     |
| [[02-Funciones-y-Metodos]]  | Funciones, múltiples retornos, closures, métodos, receptores          |
| [[03-Structs-e-Interfaces]] | Structs, embedding, interfaces implícitas, composición                |
| [[04-Punteros]]             | `&`, `*`, cuándo usar valor vs puntero, nil safety                    |
| [[05-Concurrencia-Basica]]  | Goroutines, channels, select, WaitGroup, Mutex, Context, worker pools |
| [[06-Manejo-de-Errores]]    | error interface, errores tipados, wrapping, sentinel errors           |
| [[07-Paquetes-y-Modulos]]   | go.mod, visibilidad, internal/, convenciones                          |
| [[08-Slices-y-Maps]]        | Internals, capacity, copy, patrones comunes                           |

---

## Etapa 1 — Arquitectura

| Archivo                     | Qué cubre                                           |
| --------------------------- | --------------------------------------------------- |
| [[01-Clean-Architecture]]   | Capas, Dependency Rule, estructura de proyecto real |
| [[02-Principios-SOLID]]     | Los 5 principios con ejemplos Go                    |
| [[03-Dependency-Injection]] | DI manual, Wire, testing con mocks                  |
| [[04-Patrones-de-Diseno]]   | Repository, Factory, Options, Observer, Strategy    |

---

## Etapa 2 — APIs y Protocolos

| Archivo                             | Qué cubre                                                |
| ----------------------------------- | -------------------------------------------------------- |
| [[01-HTTP-Fundamentos]]             | net/http, middleware, ServeMux, timeouts                 |
| [[02-REST-API-Design]]              | Diseño de recursos, status codes, versioning, pagination |
| [[03-GraphQL]]                      | gqlgen, schema-first, resolvers, N+1 problem             |
| [[04-Autenticacion-y-Autorizacion]] | JWT, middleware de auth, RBAC                            |

---

## Etapa 3 — Bases de Datos

| Archivo                  | Qué cubre                                                |
| ------------------------ | -------------------------------------------------------- |
| [[01-SQL-Fundamentos]]   | SQL esencial para backend, joins, índices, transacciones |
| [[02-PostgreSQL-con-Go]] | pgx, sqlx, migrations, pool de conexiones                |

---

## Etapa 4 — Testing

| Archivo              | Qué cubre                                                      |
| -------------------- | -------------------------------------------------------------- |
| [[01-Testing-en-Go]] | Unit tests, table-driven, mocks manuales, testify, integración |

---

## Etapa 5 — DevOps básico del dev

| Archivo               | Qué cubre                                                |
| --------------------- | -------------------------------------------------------- |
| [[01-Docker]]         | Dockerfile multi-stage, docker-compose, buenas prácticas |
| [[02-CI-CD]]          | GitHub Actions para Go: test, lint, build, deploy        |
| [[03-Observabilidad]] | Prometheus + slog, structured logging, métricas          |

---

## Etapa 6 — Arquitectura Avanzada

| Archivo                               | Qué cubre                                      |
| ------------------------------------- | ---------------------------------------------- |
| [[01-Microservicios-y-System-Design]] | Bounded contexts, gRPC, eventos, escalabilidad |

---

## Etapa 7 — Dart/Flutter (Frontend Profesional)

| Archivo                                         | Qué cubre                                                                       |
| ----------------------------------------------- | ------------------------------------------------------------------------------- |
| [[00-MOC-Ruta-Dart-Flutter]]                    | Ruta completa para pasar de backend dev a frontend Flutter con criterio tecnico |
| [[01-Dart-Esencial-para-Backend-Dev]]           | Dart esencial para productividad real                                           |
| [[02-POO-Funcional-Asincronia-Dart]]            | Modelado, asincronia y manejo de errores                                        |
| [[03-Flutter-Fundamentos-UI-y-Estado]]          | UI, ciclo de vida, estado y accesibilidad                                       |
| [[04-Arquitectura-Flutter-MVVM-Clean]]          | Arquitectura escalable MVVM + Clean                                             |
| [[05-Consumo-GraphQL-y-Capa-Datos]]             | Integracion GraphQL orientada a `gestion_productos`                             |
| [[06-Navegacion-Formularios-Validacion]]        | Navegacion y formularios robustos                                               |
| [[07-Testing-Flutter-Unit-Widget-Integration]]  | Testing pragmatico de front                                                     |
| [[08-Performance-Debugging-Tooling]]            | Rendimiento, debugging y calidad de entrega                                     |
| [[09-Seguridad-Errores-Observabilidad]]         | Errores, seguridad y observabilidad minima                                      |
| [[10-Proyecto-Gestion-Productos-Frontend-Plan]] | Plan practico para construir el front                                           |
| [[11-Habitos-de-Estudio-y-Metodo-Deliberado]]   | Metodo de estudio aplicable mientras trabajas                                   |
| [[12-Checklist-Mastery-Dart-Flutter]]           | Checklist de dominio real                                                       |

---

## Extras

| Archivo                   | Qué cubre                                  |
| ------------------------- | ------------------------------------------ |
| [[Recursos-Recomendados]] | Libros, posts, repos y channels esenciales |

---

## Mapa mental de dependencias

```
Etapa-0 (lenguaje)
    └── Etapa-1 (arquitectura)
            ├── Etapa-2 (APIs)
            ├── Etapa-3 (DB)
            └── Etapa-4 (Testing)
                    ├── Etapa-5 (DevOps)
                    ├── Etapa-6 (Avanzado)
                    └── Etapa-7 (Dart/Flutter)
```

---

## Señales de que ya dominás cada etapa

- **Etapa 0:** Podés leer cualquier código Go y entender qué hace sin buscar en Google.
- **Etapa 1:** Podés leer código ajeno mal estructurado e identificar qué viola la Dependency Rule en 5 minutos.
- **Etapa 2:** Podés diseñar una API REST sensata sin consultar nada.
- **Etapa 3:** Podés escribir queries con JOIN, evitar N+1, y usar transacciones correctamente.
- **Etapa 4:** Podés testear cualquier capa sin base de datos ni red real.
- **Etapa 5:** Podés Dockerizar cualquier app Go y armar un pipeline básico de CI.
- **Etapa 6:** Podés hablar de trade-offs de arquitectura en una entrevista.
- **Etapa 7:** Podés construir frontends Flutter mantenibles, consumir GraphQL con criterio, y revisar cambios de IA con fundamento tecnico.
