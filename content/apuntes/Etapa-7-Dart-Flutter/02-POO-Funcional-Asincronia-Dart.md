# 02 - POO, Funcional y Asincronia en Dart

## POO bien usada (sin sobreingenieria)

- Clases pequenas con responsabilidad unica.
- Constructores `const` cuando aplique.
- Dependencias por abstraccion (interfaces/contratos).

## Programacion funcional util

- Funciones puras para transformaciones.
- Evitar side effects en capas de dominio.
- `Either`/`Result` style (manual o con libreria) para errores explicitos.

```dart
sealed class Result<T> {
  const Result();
}

class Ok<T> extends Result<T> {
  final T value;
  const Ok(this.value);
}

class Fail<T> extends Result<T> {
  final String message;
  const Fail(this.message);
}
```

## Futures

- `async/await` para codigo secuencial legible.
- `Future.wait` para paralelismo de IO.
- Usa timeout en llamadas remotas.

## Streams

- Ideal para estados continuos (busquedas, sockets, polling).
- Cierra controladores (`StreamController.close`).

## Isolates (cuando si, cuando no)

- Si: parsing pesado, compresion, procesamiento grande.
- No: peticiones HTTP normales o logica pequena.

## Patron de manejo de errores recomendado

1. Data source lanza errores tecnicos.
2. Repository mapea a errores de dominio.
3. ViewModel traduce a estado UI (mensaje, retry, loading).

## Ejercicios

- [ ] Crear un `Result<T>` y usarlo en `getProducts()`.
- [ ] Simular latencia y timeout en una llamada.
- [ ] Implementar retry exponencial simple para fallo de red.
