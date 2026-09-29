# 04 - Arquitectura Flutter MVVM + Clean

## Objetivo

Mantener velocidad de entrega sin perder mantenibilidad.

## Capas recomendadas

- Presentation: pantallas, widgets, view models, UI state.
- Domain: entidades, casos de uso, contratos de repositorio.
- Data: repositories concretos, datasources, mappers, DTOs.

## Estructura sugerida

```text
lib/
  features/products/
    presentation/
      screens/
      widgets/
      viewmodels/
      states/
    domain/
      entities/
      repositories/
      usecases/
    data/
      datasources/
      models/
      mappers/
      repositories/
```

## MVVM en practico

- View: renderiza estado y emite eventos.
- ViewModel: orquesta casos de uso y decide estado.
- Model: entidades de dominio + DTOs de data.

## Reglas de oro

- ViewModel no conoce widgets concretos.
- UseCase no conoce GraphQL/HTTP.
- Repository de dominio no depende de libreria externa.

## Estado de pantalla recomendado

```dart
sealed class ProductsUiState {
  const ProductsUiState();
}

class ProductsInitial extends ProductsUiState { const ProductsInitial(); }
class ProductsLoading extends ProductsUiState { const ProductsLoading(); }
class ProductsLoaded extends ProductsUiState {
  final List<Product> items;
  const ProductsLoaded(this.items);
}
class ProductsError extends ProductsUiState {
  final String message;
  const ProductsError(this.message);
}
```

## Dependency injection

- Puedes usar GetIt, Riverpod, Provider, o constructor injection manual.
- Para comenzar rapido y claro, constructor injection + providers por feature.

## Antipatrones a evitar

- "God ViewModel" con 1500 lineas.
- DTOs mezclados con entidades de dominio.
- Widgets llamando directo a cliente GraphQL.

## Ejercicios

- [ ] Definir contrato `ProductRepository` (domain).
- [ ] Implementar `GetProductsUseCase`.
- [ ] Crear `ProductsViewModel` con estados sellados.
