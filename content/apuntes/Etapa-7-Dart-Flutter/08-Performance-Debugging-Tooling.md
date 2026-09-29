# 08 - Performance, Debugging y Tooling

## Performance baseline

- Mide primero, optimiza despues.
- Vigila jank en animaciones y listas largas.

## Practicas utiles

- Usa `const` widgets cuando sea posible.
- Divide widgets grandes en subwidgets puros.
- Evita trabajo pesado en `build`.
- Usa paginacion/virtualizacion para listas grandes.

## DevTools

- Flutter Inspector: arbol de widgets.
- Performance: frame timeline.
- Memory: leaks y crecimiento anomalo.
- Network: latencias y fallos.

## Debugging profesional

- Logs estructurados con contexto (`feature`, `action`, `correlationId`).
- Manejo de errores unificado (mapper error -> mensaje UI).
- Reproducibilidad: documentar pasos y datos.

## Lints y calidad

- `flutter_lints` + reglas estrictas.
- Prohibir `print` en produccion.
- Revisar complejidad ciclomatica en metodos largos.

## Ejercicios

- [ ] Perf profile de pantalla productos con 500 items simulados.
- [ ] Reducir rebuilds innecesarios en lista.
- [ ] Crear guia corta de debugging para el equipo.
