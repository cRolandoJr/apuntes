# Git — Workflow Profesional

Git se usa todos los días. Esta guía cubre lo que realmente necesitás en un equipo de trabajo: ramas, commits semánticos, rebase, resolución de conflictos y buenas prácticas.

---

## Configuración inicial

```bash
git config --global user.name "Rolando Apellido"
git config --global user.email "rolando@empresa.com"
git config --global core.editor "code --wait"   # VS Code como editor
git config --global init.defaultBranch main
git config --global pull.rebase true            # rebase en lugar de merge al hacer pull
git config --global rebase.autoStash true       # stash automático antes de rebase

# Ver configuración
git config --list --global
```

---

## Conventional Commits — el estándar de la industria

```
<tipo>(scope opcional): <descripción corta>

[cuerpo opcional]

[footer opcional: refs, breaking changes]
```

### Tipos

| Tipo       | Cuándo usar                            |
| ---------- | -------------------------------------- |
| `feat`     | Nueva funcionalidad                    |
| `fix`      | Corrección de bug                      |
| `refactor` | Cambio de código sin fix ni feature    |
| `test`     | Agregar o modificar tests              |
| `docs`     | Solo documentación                     |
| `chore`    | Tareas de mantenimiento (deps, config) |
| `perf`     | Mejora de performance                  |
| `ci`       | Cambios en CI/CD                       |
| `style`    | Formateo, sin cambio de lógica         |

```bash
# Ejemplos reales
git commit -m "feat(products): agregar paginación cursor-based"
git commit -m "fix(auth): corregir expiración de refresh token"
git commit -m "refactor(repo): extraer lógica de filtros a método privado"
git commit -m "chore: actualizar dependencias a versiones menores"
git commit -m "test(products): agregar tests de edge cases en precio"

# Breaking change — agrega ! y BREAKING CHANGE en el footer
git commit -m "feat(api)!: cambiar formato de respuesta de paginación

BREAKING CHANGE: el campo 'data' ahora se llama 'items'"
```

---

## Estrategia de ramas — GitHub Flow (la más común)

```
main           ← producción, siempre deployable
  └── feature/nombre-descriptivo   ← trabajo nuevo
  └── fix/descripcion-del-bug      ← hotfixes
  └── chore/actualizar-deps        ← mantenimiento
```

```bash
# Flujo completo para una nueva feature
git checkout main
git pull                                       # actualizar main
git checkout -b feature/productos-paginacion   # crear rama

# ... trabajar, hacer commits ...

git push -u origin feature/productos-paginacion  # push primera vez (-u setea upstream)
# Abrir PR en GitHub
# Code review → merge → eliminar rama

# Después del merge, limpiar local
git checkout main
git pull
git branch -d feature/productos-paginacion    # eliminar rama local
git remote prune origin                        # eliminar referencias remotas eliminadas
```

---

## Rebase — mantener historial limpio

```bash
# SITUACIÓN: tu rama feature está desactualizada respecto a main
# main:    A--B--C--D--E
# feature: A--B--X--Y

# Rebase: re-aplica tus commits encima de main
git checkout feature/mi-feature
git rebase main
# Resultado: A--B--C--D--E--X'--Y'

# Interactive rebase — reorganizar/limpiar commits ANTES del PR
git rebase -i HEAD~3    # últimos 3 commits

# En el editor aparece:
# pick abc123 feat: agregar endpoint
# pick def456 fix: corregir typo
# pick ghi789 fix: olvidé un campo

# Comandos disponibles:
# pick   — mantener el commit tal cual
# reword — cambiar el mensaje
# edit   — pausar para hacer más cambios
# squash — combinar con el anterior (mantiene ambos mensajes)
# fixup  — combinar con el anterior (descarta este mensaje)
# drop   — eliminar el commit

# Resultado deseado — combinar los 3 en uno limpio:
# reword abc123 feat: agregar endpoint de productos
# fixup  def456 fix: corregir typo
# fixup  ghi789 fix: olvidé un campo
```

### Cuándo NO hacer rebase

```bash
# NUNCA rebasar ramas que otros ya tienen
# Si ya hiciste push y alguien más trabaja en la misma rama → rebase rompe su historial

# Solo es seguro rebasar:
# 1. Commits que son solo tuyos (no pusheados)
# 2. Tu rama feature contra main (antes del PR, si nadie más la tiene)
```

---

## Stash — guardar trabajo sin commitear

```bash
# Guardar cambios temporalmente sin commit
git stash push -m "WIP: agregando validación de precio"

# Ver lista de stashes
git stash list
# stash@{0}: WIP: agregando validación de precio
# stash@{1}: WIP: refactor de usecase

# Recuperar el último stash
git stash pop

# Recuperar un stash específico
git stash apply stash@{1}

# Recuperar solo archivos específicos
git checkout stash@{0} -- src/usecase/product.go

# Eliminar stash sin aplicarlo
git stash drop stash@{0}
git stash clear    # eliminar todos
```

---

## Situaciones comunes en el trabajo

### "Commitié en main por error"

```bash
# Mover el último commit a una nueva rama (el código queda como está)
git branch feature/lo-que-iba-a-hacer    # crear rama con ese commit
git reset HEAD~1 --soft                  # deshacer el commit en main (código queda staged)
git stash                                # guardar cambios
git checkout feature/lo-que-iba-a-hacer  # ir a la rama correcta
git stash pop                            # recuperar cambios
```

### "Quiero deshacer cambios en un archivo"

```bash
# Descartar cambios no staged
git restore archivo.go

# Descartar todos los cambios no staged
git restore .

# Sacar archivo del staging area (sin perder cambios)
git restore --staged archivo.go

# Volver un archivo a como estaba en un commit específico
git checkout abc123 -- src/handler/product.go
```

### "El PR tiene conflictos"

```bash
git checkout feature/mi-feature
git fetch origin
git rebase origin/main    # o git merge origin/main si preferís merge

# Si hay conflictos, Git marca los archivos:
# <<<<<<< HEAD (tu código)
# tu versión
# =======
# versión de main
# >>>>>>> origin/main

# Editar el archivo para quedarte con la versión correcta
# Luego:
git add archivo_conflictuado.go
git rebase --continue    # si estabas en rebase
# o
git commit               # si estabas en merge
```

### "Necesito llevar un commit específico a mi rama" (cherry-pick)

```bash
# Ejemplo: hay un fix en otra rama que necesitás en la tuya
git log --oneline feature/otra-rama    # encontrar el hash del commit
# abc1234 fix: corregir validación de precio

git cherry-pick abc1234    # traer ese commit a tu rama actual
```

### "Quiero ver qué cambió y quién lo hizo"

```bash
# Ver historial visual
git log --oneline --graph --all

# Quién cambió qué línea en un archivo
git blame src/usecase/product.go
git blame -L 30,50 src/usecase/product.go    # solo líneas 30-50

# Buscar en qué commit se introdujo un bug (binary search)
git bisect start
git bisect bad                          # el commit actual tiene el bug
git bisect good v1.0.0                  # este tag estaba bien
# Git hace checkout en commits del medio, vos los probás
git bisect good    # o
git bisect bad
# ... repite hasta encontrar el commit culpable
git bisect reset   # terminar bisect
```

---

## `.gitignore` — qué nunca commitear

```gitignore
# Variables de entorno con secrets
.env
.env.local
.env.*.local
!.env.example    # el ejemplo SÍ se commitea

# Build outputs
/bin/
/dist/
__pycache__/
*.pyc
*.so

# Entornos virtuales
.venv/
venv/

# Editores
.vscode/settings.json    # preferencias personales (no del proyecto)
.idea/
*.swp

# Dependencias (nunca para Go, siempre para Python Node)
vendor/            # en Go, vendor/ puede commitearse en proyectos empresariales
node_modules/

# Coverage y artifacts de tests
.coverage
htmlcov/
*.test    # binarios de test de Go

# OS
.DS_Store
Thumbs.db
```

---

## Pull Request — buenas prácticas

````markdown
## ¿Qué hace este PR?

Agrega paginación cursor-based al endpoint de productos.

## ¿Por qué?

El endpoint devolvía todos los productos sin límite → timeout en producción con 50k registros.

## Cambios

- repo/product_repo.go: agrega método ListWithCursor
- handler/products.go: nuevos parámetros cursor/limit
- tests: tests de paginación con 0, 1 y N registros

## Testing

[ ] Tests unitarios pasan
[ ] Tests de integración pasan
[ ] Probado manualmente con Postman

## Screenshots / ejemplos (si aplica)

`GET /products?limit=20&cursor=eyJpZCI6MTAwfQ==`
````

### Para el reviewer

- Los PRs deben ser pequeños (< 400 líneas si es posible)
- Un PR = una cosa
- Si el PR está grande: agregar contexto en la descripción del por qué
