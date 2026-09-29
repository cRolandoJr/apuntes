# 01 - Dart Esencial para Backend Dev

## Mental model para alguien que viene de Go

- Go: minimalismo + interfaces implicitas + concurrencia por goroutines.
- Dart: OO + tipado fuerte + null safety + asincronia por event loop/Future/Stream.
- En Flutter, entender Dart bien evita deuda tecnica en UI state y arquitectura.

## Tipos y null safety

- Todo tipo puede ser nullable (`String?`) o non-null (`String`).
- Usa `late` solo cuando tengas garantia real de inicializacion.
- Evita abuso de `!` (null assertion).

```dart
class Product {
  final String id;
  final String name;
  final double price;

  const Product({required this.id, required this.name, required this.price});
}
```

## Inmutabilidad por defecto

- Prefiere `final` para campos y variables.
- Modelos inmutables facilitan testeo y evitan bugs de estado compartido.

## Colecciones y operadores utiles

- `map`, `where`, `fold`, `any`, `every`, `firstWhere`.
- Spread: `[...]`, null-aware spread: `...?`.

## Errores y control de flujo

- Usa `Exception` para errores esperables de negocio.
- Usa `Error` para fallas de programacion (no recuperables).

```dart
Product parseProduct(Map<String, dynamic> json) {
  try {
    return Product(
      id: json['id'] as String,
      name: json['name'] as String,
      price: (json['price'] as num).toDouble(),
    );
  } catch (e) {
    throw FormatException('Invalid product payload: $e');
  }
}
```

## Extensiones, mixins y sealed

- Extensions: agrega API a tipos existentes.
- Mixins: comparte comportamiento.
- `sealed class`: modela estados cerrados (ideal para UI state).

## Concurrencia y asincronia

- `Future`: 1 resultado futuro.
- `Stream`: secuencia de resultados.
- `Isolate`: trabajo CPU heavy fuera del hilo principal.

## Buenas practicas iniciales

- Regla 1: sin `dynamic` salvo frontera externa (JSON, plugin, etc).
- Regla 2: mapea JSON a modelos tipados rapidamente.
- Regla 3: no mezclar parsing de datos con widgets.

## Ejercicios

- [ ] Modelar entidad `Product` inmutable con `copyWith`.
- [ ] Parsear lista JSON a `List<Product>` con manejo de errores.
- [ ] Crear `sealed class ProductFailure` con 4 tipos de error.
