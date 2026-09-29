# 07 - Testing Flutter: Unit, Widget, Integration

## Piramide pragmatica

- Unit tests: casos de uso, mappers, validadores.
- Widget tests: estados clave de pantallas y widgets criticos.
- Integration tests: flujos E2E minimos.

## Que testear primero en gestion_productos

1. Validaciones de producto (name, price, stock).
2. ViewModel de lista y formulario (loading/error/success).
3. Pantalla lista con estados vacio/error/cargado.
4. Flujo create -> aparece en lista.

## Unit test ejemplo (idea)

- Given: repo mock devuelve lista.
- When: `GetProductsUseCase()`.
- Then: resultado `Ok` con elementos esperados.

## Widget test ejemplo (idea)

- Given: ViewModel en `ProductsLoading`.
- Then: se ve `CircularProgressIndicator`.

## Integration test minimo

- Levantar app con backend local.
- Crear producto.
- Verificar en lista.
- Editar y eliminar.

## Cobertura util

- No persigas 100% por vanity metric.
- Cubre caminos de alto riesgo y negocio.

## Ejercicios

- [ ] Crear suite de unit tests para casos de uso CRUD.
- [ ] Crear 3 widget tests para estados de lista.
- [ ] Crear 1 integration test de flujo feliz CRUD.
