# 09 - Seguridad, Errores y Observabilidad

## Seguridad de aplicacion cliente

- No hardcodear secretos en app.
- Configurar endpoints por ambiente (dev/stg/prod).
- Sanitizar y validar entradas del usuario.

## Manejo de errores por capas

- Data: clasifica errores tecnicos (network, parse, server).
- Domain: traduce a errores de negocio.
- Presentation: mensajes claros + acciones (`retry`, `contact support`).

## Observabilidad minima para equipos chicos

- Logging estructurado.
- Correlation ID (alineado con backend si existe).
- Captura de errores no controlados en zona global.

## Politica de mensajes UI

- Usuario final: mensaje claro y accionable.
- Logs internos: detalle tecnico.
- Nunca mostrar stack trace crudo al usuario.

## Checklist de release

- [ ] Variables de entorno correctas.
- [ ] Errores globales capturados.
- [ ] Logs sin datos sensibles.
- [ ] Flujos criticos probados manualmente.
