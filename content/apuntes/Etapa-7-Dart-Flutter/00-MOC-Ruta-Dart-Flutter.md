# MOC - Ruta de Aprendizaje Dart/Flutter (orientada a Gama y gestion_productos)

## Objetivo macro

Pasar de backend dev (Go) a frontend Flutter productivo, entendiendo que hace la IA y pudiendo revisar, aceptar, rechazar o mejorar cambios con criterio tecnico.

## Contexto del proyecto practico

- Backend objetivo: `~/gestion_productos`
- Stack backend detectado: Go + GraphQL (gqlgen), clean architecture, repositorio en memoria.
- Endpoints: `/` (playground), `/query` (GraphQL).
- Operaciones clave: `products`, `product(id)`, `createProduct`, `updateProduct`, `deleteProduct`.

## Como usar estos apuntes

1. Estudia 1 modulo por dia laboral (60-90 min).
2. Aplica lo estudiado en una tarea real del front de `gestion_productos`.
3. Cierra cada sesion con evidencia:
   - una nota corta de decisiones,
   - un commit pequeno,
   - una pregunta tecnica respondida.

## Indice de modulos

- [01-Dart-Esencial-para-Backend-Dev](01-Dart-Esencial-para-Backend-Dev.md)
- [02-POO-Funcional-Asincronia-Dart](02-POO-Funcional-Asincronia-Dart.md)
- [03-Flutter-Fundamentos-UI-y-Estado](03-Flutter-Fundamentos-UI-y-Estado.md)
- [04-Arquitectura-Flutter-MVVM-Clean](04-Arquitectura-Flutter-MVVM-Clean.md)
- [05-Consumo-GraphQL-y-Capa-Datos](05-Consumo-GraphQL-y-Capa-Datos.md)
- [06-Navegacion-Formularios-Validacion](06-Navegacion-Formularios-Validacion.md)
- [07-Testing-Flutter-Unit-Widget-Integration](07-Testing-Flutter-Unit-Widget-Integration.md)
- [08-Performance-Debugging-Tooling](08-Performance-Debugging-Tooling.md)
- [09-Seguridad-Errores-Observabilidad](09-Seguridad-Errores-Observabilidad.md)
- [10-Proyecto-Gestion-Productos-Frontend-Plan](10-Proyecto-Gestion-Productos-Frontend-Plan.md)
- [11-Habitos-de-Estudio-y-Metodo-Deliberado](11-Habitos-de-Estudio-y-Metodo-Deliberado.md)
- [12-Checklist-Mastery-Dart-Flutter](12-Checklist-Mastery-Dart-Flutter.md)

## Ruta recomendada en 6 semanas

- Semana 1: Modulos 01, 02, 03.
- Semana 2: Modulos 04, 05.
- Semana 3: Modulos 06, 07.
- Semana 4: Modulos 08, 09.
- Semana 5: Modulo 10 (MVP funcional).
- Semana 6: Modulos 11 y 12 + hardening del MVP.

## Criterio de completitud

No basta "que funcione".
Debe cumplir:

- Correctitud funcional.
- Legibilidad y mantenibilidad.
- Cobertura de pruebas clave.
- Observabilidad minima (logs, errores mapeados, estados UI claros).
