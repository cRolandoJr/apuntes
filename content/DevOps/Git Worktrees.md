[[0. Índice DevOps]]

# Git Worktrees

> Varias carpetas de trabajo, un solo repositorio.
> Complemento de [[Git]] — asume que ya manejás branches, merge y stash.

## 1. El problema que resuelve

Un repo git normal tiene **una sola carpeta de trabajo**. Eso significa **una sola rama abierta a la vez**. Si dos cosas necesitan pasar en paralelo, se pisan:

- Estás a mitad de una feature y entra un hotfix urgente en `main`.
- Querés compilar la versión vieja para comparar, sin perder lo que estás editando.
- **Dos agentes de IA (o dos personas en la misma máquina) trabajando en el mismo repo al mismo tiempo.**

El reflejo habitual es `git stash` + `git checkout` + volver. Funciona, pero tiene tres costos: se pierde el estado de compilación, es fácil olvidarse un stash, y no sirve si necesitás las dos ramas **vivas a la vez**.

## 2. Qué es un worktree

Un worktree es una **segunda carpeta de trabajo conectada al mismo `.git`**.

La analogía: el `.git` es la **biblioteca** (todos los commits, todas las ramas, todo el historial). La carpeta de trabajo es tu **escritorio**, con unos pocos libros abiertos. Un worktree te da **otro escritorio, misma biblioteca**.

Lo que NO es: no es un clon. No se duplica el historial, no hay que volver a hacer `fetch`, y un commit hecho en un worktree lo ven todos los demás al instante.

```bash
# Cómo distinguir si estás dentro de un worktree:
git rev-parse --git-dir          # .git/worktrees/<nombre>   ← worktree
git rev-parse --git-common-dir   # .git                      ← la biblioteca compartida

# Si los dos dan lo mismo, estás en el checkout principal.
```

## 3. La red de seguridad

**Git prohíbe tener la misma rama abierta en dos worktrees.**

```bash
git worktree add ../otra-carpeta develop
# fatal: 'develop' is already checked out at '/ruta/del/repo'
```

Esto no es una convención ni una buena práctica: es una **negativa del programa**. Y es exactamente lo que hace imposible que dos trabajos paralelos se pisen — te obliga a ramas distintas, y ramas distintas en carpetas distintas no comparten ni un archivo.

## 4. Comandos

```bash
# Crear un worktree con una rama NUEVA, desde un punto de partida explícito
git worktree add <carpeta> -b <rama-nueva> <base>
#   add <carpeta>    → dónde se crea el segundo escritorio
#   -b <rama-nueva>  → crea la rama y la deja abierta ahí
#   <base>           → desde dónde arranca (develop, main, un tag, un commit)

# Ejemplo real
git worktree add .worktrees/hotfix-login -b fix/login-timeout develop

# Crear uno sobre una rama que YA existe (ojo: no puede estar abierta en otro lado)
git worktree add .worktrees/revision feature/algo-ya-existente

# Ver todos los escritorios abiertos y en qué rama está cada uno
git worktree list

# Borrar un worktree cuando terminaste (borra la carpeta, NO la rama)
git worktree remove .worktrees/hotfix-login

# Borrar también la rama, una vez mergeada
git branch -d fix/login-timeout

# Si borraste la carpeta a mano, git queda con una referencia muerta:
git worktree prune
```

**Siempre poné la base explícita** (`develop`, `main`) en el `add`. Sin ella, git rama desde donde estés parado — y si estabas en un experimento a medio terminar, tu rama nueva nace contaminada.

## 5. El caso "dos sesiones en la misma PC"

Este es el escenario que más duele, porque falla en silencio. Dos agentes (o dos terminales) en la misma carpeta:

| Riesgo | Qué pasa |
|---|---|
| **El piso se mueve** | Una sesión hace `git checkout otra-rama` y los archivos de la otra cambian debajo. Creés estar editando una versión y estás editando otra. |
| **`git status` se mezcla** | Los cambios de ambos aparecen revueltos. Un `git add -A` se lleva el trabajo del otro sin avisar. |
| **Una sola rama** | Físicamente no podés tener dos ramas abiertas en una carpeta. |

Solución: cada sesión en su worktree.

```bash
# Sesión A se queda en el checkout principal, en develop
# Sesión B:
git worktree add .worktrees/mi-tarea -b feature/mi-tarea develop
cd .worktrees/mi-tarea
```

A partir de ahí las dos pueden commitear, cambiar de rama y romper cosas sin tocarse.

## 6. Trampas que sí muerden

### El stash es COMPARTIDO

`git stash` vive en el `.git` — o sea, **en la biblioteca, no en tu escritorio**. Un `git stash pop` desde tu worktree puede sacar el stash que guardó la otra sesión.

```bash
# En vez de `git stash` pelado, cuando hay varios worktrees:
git stash push -u -m "wip-lo-mio"     # con etiqueta propia
git stash list                        # buscá la tuya por la etiqueta
git stash apply stash@{2}             # apply, NO pop (no lo borra si te equivocaste)
```

Mejor todavía: en vez de stashear, hacé un commit temporal (`git commit -m "wip"`) y después `git reset --soft HEAD~1`. Es tuyo, está en tu rama, nadie te lo puede robar.

### Las dependencias NO se comparten

El worktree comparte el historial, pero **no los archivos ignorados**. `node_modules/`, `.dart_tool/`, `build/`, `venv/` no existen en el worktree nuevo. Cada uno necesita su propia instalación:

```bash
flutter pub get     # o npm install, cargo build, go mod download...
```

Es el precio del aislamiento — y también su ventaja: podés tener dos builds distintos sin que se peleen.

### Si el worktree vive dentro del repo, tiene que estar ignorado

Si lo creás en `.worktrees/` adentro del proyecto, agregalo al `.gitignore` **antes** de crearlo. Si no, git ve todos esos archivos como nuevos y podés commitear una copia entera del repo dentro del repo.

```bash
echo ".worktrees/" >> .gitignore
git check-ignore -v .worktrees    # verificá que quedó ignorado
```

La alternativa es ponerlo **fuera** del repo (`../repo-hotfix/`) y no tocar el `.gitignore`.

### No podés borrar una rama abierta en otro worktree

```bash
git branch -d feature/x
# error: Cannot delete branch 'feature/x' checked out at '/otra/carpeta'
```

Primero `git worktree remove`, después `git branch -d`.

### `remove` se niega si hay trabajo sin guardar

Es una protección, no un error. Commiteá (o descartá a conciencia) antes de remover.

## 7. Cómo probar el trabajo que vive en un worktree

La confusión más común, y es de modelo mental, no de comandos:

> **"Ya commiteé y pusheé, ¿por qué no lo veo en mi carpeta original?"**

Porque el commit está en la **biblioteca** (el `.git`, compartido), pero tu escritorio original tiene abierta otra rama. El trabajo existe; simplemente no está desplegado ahí.

Tres formas de probarlo, de menos a más invasiva:

### 1. Correrlo desde el worktree (lo más limpio)

No hay que mergear ni tocar la carpeta original. Sólo acordate de que **las dependencias no se comparten** (§6), así que la primera vez hay que instalarlas:

```bash
cd <ruta-del-worktree>
flutter pub get          # o npm install, cargo build, go mod download
flutter run -d chrome    # o el comando que levante tu app
```

Si la app necesita config de entorno, se la pasás igual que siempre — el worktree tiene su propia copia de los archivos versionados:

```bash
flutter run -d chrome --dart-define-from-file=.env/test.json
```

### 2. Mergear a la rama de integración

Recién ahí aparece en la carpeta original. Es la vía normal **cuando el trabajo ya está validado**, y tiene un costo que conviene ver antes: si otra persona (u otra sesión) está editando en esa carpeta, el merge le mueve el piso mientras trabaja.

### 3. Checkoutear la rama en la carpeta original

**No se puede mientras el worktree exista.** Git se niega a tener la misma rama en dos escritorios (§3). Habría que sacar el worktree primero:

```bash
git worktree remove <ruta-del-worktree>
git checkout <la-rama>
```

Es la más invasiva y casi nunca hace falta.

### La regla corta

**Probar → opción 1. Integrar → opción 2.** La 3 es para cuando ya terminaste con el worktree, no para probar.

## 8. Cuándo NO usarlo

- **Para cambiar de rama y volver.** Eso es `git checkout` y ya. El worktree es para cosas **simultáneas**, no secuenciales.
- **Si el proyecto tarda muchísimo en instalar dependencias.** Cada worktree paga ese costo de nuevo.
- **Si no vas a limpiar.** Worktrees olvidados llenan el disco y confunden. `git worktree list` cada tanto.

## 9. Resumen

```bash
git worktree add <carpeta> -b <rama> <base>   # crear
git worktree list                              # ver
git worktree remove <carpeta>                  # borrar carpeta
git branch -d <rama>                           # borrar rama (después)
git worktree prune                             # limpiar referencias muertas
```

Tres ideas para retener:

1. **Misma biblioteca, otro escritorio.** No es un clon.
2. **La misma rama no puede estar en dos escritorios.** Esa negativa es la que te protege.
3. **El stash y las ramas son compartidos; los archivos ignorados y las dependencias, no.**

---

Relacionado: [[Git]] · [[CI-CD con GitHub Actions]]
