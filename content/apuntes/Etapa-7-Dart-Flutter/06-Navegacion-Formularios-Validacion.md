# 06 - Navegacion, Formularios y Validacion

## Navegacion

- Recomendado para escalar: `go_router`.
- Define rutas por feature y evita rutas sueltas en toda la app.

## Rutas sugeridas para gestion_productos

- `/products`
- `/products/new`
- `/products/:id`
- `/products/:id/edit`

## Formularios robustos

- Usa `Form` + `GlobalKey<FormState>`.
- Validacion sincrona para formato.
- Validacion asincrona para reglas remotas (si aplica).

```dart
String? validatePrice(String? value) {
  final parsed = double.tryParse(value ?? '');
  if (parsed == null) return 'Precio invalido';
  if (parsed <= 0) return 'Debe ser mayor a 0';
  return null;
}
```

## UX de formularios

- Modo create/edit con mismo componente base.
- Boton submit deshabilitado durante loading.
- Mensajes de error cerca del campo.
- Toast/snackbar solo para eventos globales.

## Focus y teclado

- `TextInputAction.next` para flujo.
- Mover focus automaticamente entre campos.
- Compatibilidad web/desktop con teclado.

## Ejercicios

- [ ] Implementar formulario create product con validacion completa.
- [ ] Reutilizar formulario para edit product.
- [ ] Manejar submit doble (doble click) sin request duplicado.
