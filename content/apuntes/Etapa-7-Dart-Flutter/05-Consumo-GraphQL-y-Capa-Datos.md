# 05 - Consumo GraphQL y Capa de Datos

## Contexto backend gestion_productos

Schema con consultas y mutaciones CRUD de productos.

## Cliente GraphQL recomendado

- `graphql_flutter` o `ferry`.
- Para empezar rapido: `graphql_flutter`.
- Para tipado fuerte por generacion: evaluar codegen (graphql_codegen/ferry).

## Flujo de datos limpio

1. DataSource ejecuta query/mutation.
2. Mapper transforma JSON -> DTO -> Entity.
3. Repository retorna `Result<Entity>` al dominio.
4. ViewModel mapea resultado a estado UI.

## Query base (ejemplo)

```graphql
query Products {
  products {
    id
    name
    price
    stock
  }
}
```

## Mutation base (ejemplo)

```graphql
mutation CreateProduct($name: String!, $price: Float!, $stock: Int!) {
  createProduct(input: { name: $name, price: $price, stock: $stock }) {
    id
    name
    price
    stock
  }
}
```

## Estrategia de errores

- Error de red: sin conectividad o timeout.
- Error GraphQL: validacion o regla de negocio.
- Error parseo: contrato roto entre backend y front.

## Cache y sincronizacion

- MVP: politica simple de fetch + refresh manual.
- Version 2: cache normalizada + optimistic updates en mutaciones.

## Seguridad minima

- Sanitizar inputs de formularios.
- No loggear datos sensibles.
- Si luego agregan auth, centralizar token y refresh policy.

## Ejercicios

- [ ] Implementar `ProductRemoteDataSource` con query `products`.
- [ ] Implementar `createProduct` con manejo de errores por tipo.
- [ ] Agregar logging estructurado por operacion GraphQL.
