# 10 - Proyecto Practico: Front Flutter para gestion_productos

## Objetivo

Construir un front Flutter (web + mobile ready) para el backend GraphQL de `~/gestion_productos`.

## Alcance MVP

- Lista productos.
- Ver detalle producto.
- Crear producto.
- Editar producto.
- Eliminar producto.
- Estados de loading/error/empty consistentes.

## Arquitectura objetivo

- Feature-based + MVVM + clean.
- Data source GraphQL.
- Repositorio de dominio.
- Casos de uso por operacion.

## Hitos

### Hito 1 - Bootstrap tecnico

- Configurar proyecto Flutter base.
- Configurar cliente GraphQL apuntando a `/query`.
- Definir carpetas y convenciones.

### Hito 2 - Lectura

- Implementar `products` y `product(id)`.
- Pantallas lista y detalle.

### Hito 3 - Escritura

- Implementar `create`, `update`, `delete`.
- Formularios reutilizables create/edit.

### Hito 4 - Calidad

- Unit + widget tests basicos.
- Manejo uniforme de errores.
- Mejora de UX y accesibilidad.

## Definicion de terminado por feature

- Funciona segun criterio de negocio.
- Tiene pruebas minimas.
- Tiene manejo de error y loading.
- Queda documentada una decision tecnica.

## Riesgos esperados

- Cambio de contrato GraphQL sin aviso.
- Inconsistencias de tipos numericos (Float/int).
- Deuda en estado UI por mezcla de responsabilidades.

## Mitigaciones

- Introducir mapper centralizado.
- Contratos tipados + tests de mapping.
- Revisiones pequenas y frecuentes.
