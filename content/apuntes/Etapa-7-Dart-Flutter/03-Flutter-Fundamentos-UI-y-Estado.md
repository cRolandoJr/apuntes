# 03 - Flutter Fundamentos: UI y Estado

## Como piensa Flutter

- Todo es widget.
- UI = funcion del estado.
- Rebuild no es redraw completo bruto; Flutter optimiza por arbol.

## Stateless vs Stateful

- `StatelessWidget`: UI derivada solo de parametros.
- `StatefulWidget`: mantiene estado local efimero.

## Layout esencial

- `Row`, `Column`, `Expanded`, `Flexible`, `Stack`, `ListView`, `GridView`.
- Evita nested widgets gigantes: extrae subwidgets con nombre.

## Estado: local vs global

- Local: toggles, focus, animation chica.
- Pantalla/modulo: ViewModel/StateNotifier/Bloc/etc.
- App-wide: auth session, tema, configuracion global.

## Ciclo de vida importante

- `initState`: inicializacion unica.
- `didUpdateWidget`: reaccion a cambios de props.
- `dispose`: liberar controllers/listeners.

## Errores comunes

- setState despues de dispose.
- Hacer llamadas async en build.
- Guardar logica de negocio en widget.

## Accesibilidad minima

- Labels claros en botones/campos.
- Contraste y tamano de fuente razonable.
- Soporte de teclado para web/desktop.

## Ejercicio practico

Construye una pantalla de lista de productos con:

- Loading.
- Empty state.
- Error state con retry.
- Lista con cards y boton "Crear producto".

Checklist:

- [ ] Widget tree legible.
- [ ] Estado desacoplado de la vista.
- [ ] Manejo de errores visible al usuario.
