# Resumen
```md
Perfecto. Aquí va la chuleta ultra corta (12 líneas) para usar siempre:

1. Elegir ID de tarea: T-XXX
2. Entrar al repo correcto (web/mobile/backend)
3. git checkout develop
4. git pull origin develop
5. git checkout -b feat/T-XXX-area
6. Hacer cambios + probar
7. git add .
8. git commit -m "feat(area): resumen [T-XXX]"
9. git push -u origin feat/T-XXX-area
10. Abrir PR hacia develop y mergear
11. git checkout develop && git pull origin develop
12. git branch -d feat/T-XXX-area

Regla de oro:

- Nunca crear rama nueva desde una feature; siempre desde develop actualizado.
```
---
---

Perfecto. Aquí tienes un template personal, corto y repetible, para usar siempre en web, mobile y backend.

TEMPLATE BASE POR TAREA (3 REPOS)

Nombre de tarea:

- ID: T-XXX
- Descripción corta: players-count, fix-login, etc.

Ramas por repo (mismo ID):

- web: feat/T-XXX-web
- mobile: feat/T-XXX-mobile
- backend: feat/T-XXX-backend

Flujo estándar por cada repo:

1. Ir a develop y actualizar
2. Crear rama desde develop
3. Hacer cambios
4. Commit con mensaje claro
5. Push de la rama
6. Abrir PR a develop
7. Merge PR
8. Volver a develop y pull
9. Borrar rama local (opcional recomendado)

COMANDOS TEMPLATE (copiar y reemplazar)

A. Preparar rama nueva

- git checkout develop
- git pull origin develop
- git checkout -b feat/T-XXX-AREA

B. Guardar trabajo

- git add .
- git commit -m "feat(AREA): descripcion corta [T-XXX]"
- git push -u origin feat/T-XXX-AREA

C. Después del merge

- git checkout develop
- git pull origin develop
- git branch -d feat/T-XXX-AREA
- git push origin --delete feat/T-XXX-AREA

REGLAS DE ORO (ANTI-ENREDO)

1. Nunca crear rama nueva desde una feature vieja; siempre desde develop actualizado.
2. Una tarea = una rama por repo.
3. No mezclar cambios de tareas distintas en la misma rama.
4. Si backend cambia contrato, primero merge/deploy backend; luego front.
5. Push frecuente para respaldo.
6. Si hay cambios locales sin commit y necesitas cambiar de rama: usa stash con nombre.

STASH TEMPLATE SEGURO

- git stash push -u -m "temp T-XXX motivo"
- git stash list
- git stash pop

RUTINA DIARIA (5 minutos)

1. Revisar rama actual

- git branch --show-current

2. Revisar cambios pendientes

- git status --short

3. Revisar si develop avanzó

- git fetch
- git log --oneline HEAD..origin/develop

CHECKLIST DE CIERRE DE TAREA

1. Rama pusheada
2. PR abierta
3. PR mergeada a develop
4. develop local actualizado
5. Rama feature eliminada
6. Nota breve de qué quedó pendiente