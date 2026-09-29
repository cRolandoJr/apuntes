[[0. Índice DevOps]]

# Git

## 1. Configuración Inicial

```bash
# Identidad (obligatorio)
git config --global user.name "Rolando"
git config --global user.email "tu@email.com"

# Editor por defecto
git config --global core.editor vim

# Branch default
git config --global init.defaultBranch main

# Ver configuración
git config --list
```

---

## 2. Flujo Básico

```bash
# Inicializar repositorio
git init

# Clonar repositorio existente
git clone https://github.com/usuario/repo.git
git clone git@github.com:usuario/repo.git    # SSH (recomendado)

# Ver estado
git status
git status -s    # Formato corto

# Agregar archivos al staging
git add archivo.txt        # Un archivo
git add .                  # Todo
git add -p                 # Interactivo (elegir hunks)

# Commit
git commit -m "mensaje descriptivo"
git commit -am "mensaje"   # add + commit (solo archivos ya trackeados)

# Ver historial
git log
git log --oneline                     # Una línea por commit
git log --oneline --graph --all       # Gráfico con branches
git log --author="Rolando"            # Filtrar por autor
git log -5                            # Últimos 5 commits
git log --since="2 weeks ago"         # Últimas 2 semanas
git log -- archivo.txt                # Historial de un archivo
```

---

## 3. Branches (Ramas)

```bash
# Listar branches
git branch              # Locales
git branch -r           # Remotas
git branch -a           # Todas

# Crear branch
git branch nueva-feature

# Cambiar de branch
git checkout nueva-feature
git switch nueva-feature       # Forma moderna (Git 2.23+)

# Crear y cambiar en un paso
git checkout -b nueva-feature
git switch -c nueva-feature

# Borrar branch (local)
git branch -d rama-mergeada       # Solo si ya fue mergeada
git branch -D rama-sin-mergear    # Forzar borrado

# Borrar branch remota
git push origin --delete rama-vieja

# Renombrar branch actual
git branch -m nuevo-nombre
```

---

## 4. Merge y Rebase

### Merge (une el historial)

```bash
# Estando en main, mergear feature:
git checkout main
git merge feature-login

# Si hay conflictos:
# 1. Git marca los archivos con conflicto
# 2. Editar manualmente (buscar <<<<<<< ======= >>>>>>>)
# 3. Resolver y:
git add archivo-resuelto.txt
git commit    # Git genera mensaje de merge automáticamente

# Abortar merge si te arrepentís
git merge --abort
```

### Rebase (reescribe el historial)

```bash
# Estando en feature, rebasar sobre main:
git checkout feature
git rebase main

# Rebase interactivo (editar, squash, reordenar commits)
git rebase -i HEAD~3     # Últimos 3 commits

# En el editor interactivo:
# pick   = mantener commit
# squash = unir con el commit anterior
# reword = cambiar mensaje
# drop   = eliminar commit
# edit   = pausar para editar

# Abortar rebase
git rebase --abort

# Continuar después de resolver conflictos
git rebase --continue
```

> **Regla de oro:** Nunca hacer rebase de branches que ya pusheaste y otros usan. Rebase es para limpiar TU historial local antes de mergear.

---

## 5. Remotos

```bash
# Ver remotos configurados
git remote -v

# Agregar remoto
git remote add origin git@github.com:usuario/repo.git

# Cambiar URL del remoto
git remote set-url origin git@github.com:usuario/nuevo-repo.git

# Fetch (bajar cambios sin mergear)
git fetch origin
git fetch --all

# Pull (fetch + merge)
git pull origin main
git pull --rebase origin main    # Fetch + rebase (más limpio)

# Push
git push origin main
git push -u origin main    # -u establece upstream (después solo git push)
git push origin feature

# Push de todas las branches
git push --all origin

# Push de tags
git push origin --tags
```

---

## 6. Stash (Guardar Cambios Temporales)

```bash
# Guardar cambios sin commitear
git stash
git stash save "descripción de lo que guardé"

# Ver stashes guardados
git stash list

# Restaurar último stash
git stash pop           # Restaurar y borrar del stash
git stash apply         # Restaurar sin borrar del stash

# Restaurar un stash específico
git stash pop stash@{2}

# Borrar un stash
git stash drop stash@{0}

# Borrar todos los stashes
git stash clear
```

> ⚠️ El stash vive en el `.git`, no en tu carpeta de trabajo. Si tenés varios worktrees abiertos, **todos comparten la misma pila de stashes** y un `pop` puede llevarse lo del otro. Ver [[Git Worktrees]].

---

## 7. Deshacer Cosas

```bash
# Deshacer cambios en un archivo (antes de add)
git checkout -- archivo.txt
git restore archivo.txt         # Forma moderna

# Sacar archivo del staging (después de add, antes de commit)
git reset HEAD archivo.txt
git restore --staged archivo.txt    # Forma moderna

# Cambiar el último commit (mensaje o archivos)
git commit --amend -m "nuevo mensaje"
git add archivo-olvidado.txt && git commit --amend --no-edit

# Revertir un commit (crea un nuevo commit que deshace)
git revert abc1234

# Reset (mover HEAD hacia atrás)
git reset --soft HEAD~1     # Deshacer commit, mantener en staging
git reset --mixed HEAD~1    # Deshacer commit, sacar de staging (default)
git reset --hard HEAD~1     # Deshacer commit y BORRAR cambios (peligroso)

# Recuperar algo después de un reset --hard
git reflog                  # Ver historial de movimientos de HEAD
git reset --hard abc1234    # Volver a un punto del reflog
```

---

## 8. Tags

```bash
# Crear tag
git tag v1.0.0                            # Tag liviano
git tag -a v1.0.0 -m "Release 1.0.0"     # Tag anotado (recomendado)

# Tag en un commit específico
git tag -a v0.9.0 -m "Beta" abc1234

# Listar tags
git tag
git tag -l "v1.*"

# Push tags
git push origin v1.0.0
git push origin --tags         # Todos los tags

# Borrar tag
git tag -d v1.0.0              # Local
git push origin --delete v1.0.0  # Remoto
```

---

## 9. .gitignore

```gitignore
# Archivo: .gitignore (en la raíz del repo)

# Archivos compilados
*.exe
*.o
*.so
*.dll

# Dependencias
node_modules/
vendor/
__pycache__/
*.pyc

# IDEs
.vscode/
.idea/
*.swp
*.swo

# Sistema
.DS_Store
Thumbs.db

# Secrets (NUNCA commitear)
.env
*.pem
*.key
credentials.json

# Logs y temporales
*.log
tmp/
```

```bash
# Ignorar un archivo que ya fue trackeado
git rm --cached archivo.txt
echo "archivo.txt" >> .gitignore
git commit -am "Dejar de trackear archivo.txt"

# Ver qué archivos están siendo ignorados
git status --ignored
```

---

## 10. Branching Strategies

### Git Flow (equipos grandes, releases formales)

```
main ────────●──────────●──────── (releases)
              \        /
develop ───●───●──●───●────────── (integración)
            \    /
feature ─────●──● (features individuales)
```

- `main` = código en producción
- `develop` = integración
- `feature/*` = features nuevas (salen de develop)
- `release/*` = preparar release
- `hotfix/*` = fix urgente en producción

### GitHub Flow (equipos chicos, deploy continuo)

```
main ────●────●────●────●──── (siempre deployable)
          \  /      \  /
feature ───●── PR ── ●── PR
```

1. Branch desde main
2. Hacer commits
3. Abrir Pull Request
4. Code review
5. Merge a main
6. Deploy

> **Para empezar:** Usá GitHub Flow. Es simple y funciona bien con CI/CD.

---

## 11. Comandos Útiles

```bash
# Ver diferencias
git diff                       # Cambios no agregados
git diff --staged              # Cambios en staging
git diff main..feature         # Diferencia entre branches
git diff HEAD~3                # Últimos 3 commits

# Buscar quién modificó cada línea
git blame archivo.txt
git blame -L 10,20 archivo.txt    # Solo líneas 10-20

# Buscar texto en el historial
git log -S "texto_buscado"     # Commits donde apareció/desapareció
git grep "patron" HEAD         # Buscar en archivos trackeados

# Cherry-pick (traer un commit específico a otra branch)
git cherry-pick abc1234

# Ver un archivo en otro commit/branch
git show main:src/config.ts
git show HEAD~3:README.md

# Limpiar archivos no trackeados
git clean -n     # Dry run (mostrar qué borraría)
git clean -fd    # Borrar archivos y directorios no trackeados

# Bisect (encontrar qué commit introdujo un bug)
git bisect start
git bisect bad                # El commit actual tiene el bug
git bisect good v1.0.0        # Este commit estaba bien
# Git va haciendo checkout de commits intermedios
# En cada uno, testear y marcar:
git bisect good    # o
git bisect bad
# Al final te dice cuál commit introdujo el bug
git bisect reset   # Volver al estado normal
```

---

## 12. SSH Keys para GitHub/GitLab

```bash
# Generar clave SSH
ssh-keygen -t ed25519 -C "tu@email.com"

# Copiar clave pública
cat ~/.ssh/id_ed25519.pub
# Pegarla en GitHub → Settings → SSH and GPG keys → New SSH key

# Verificar conexión
ssh -T git@github.com

# Si tenés múltiples cuentas, configurar ~/.ssh/config:
Host github-personal
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_personal

Host github-trabajo
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_trabajo

# Uso:
git clone git@github-personal:usuario/repo.git
```

### El Protocolo Seguro de Sanitización (Clean-up)

**1. Sincroniza y Poda (Prune)** El primer paso de la limpieza no es borrar tus ramas locales, sino decirle a tu Git que olvide las ramas que ya fueron borradas en GitHub/GitLab (por ejemplo, las que se borran automáticamente al aceptar un Pull Request).

- Sitúate en tu rama principal: `git checkout develop`
    
- Ejecuta el comando de poda:
    
    Bash
    
    ```
    git fetch -p
    ```
    
    _(La `-p` es de prune. Esto limpia las referencias remotas "fantasmas" que ya no existen en internet)._
    

**2. Identifica qué es seguro borrar** Nunca borres a ciegas. Estando en `develop` (y después de un `git pull` para tener lo último), pregúntale a Git qué ramas locales ya están 100% integradas en el historial principal y son seguras de eliminar:

Bash

```
git branch --merged
```

_Todo lo que salga en esa lista (excepto develop, main o test) ya es código seguro y respaldado._

**3. El Borrado Seguro (La regla de oro)** Cuando vayas a borrar una rama local, **usa siempre la "d" minúscula**:

Bash

```
git branch -d nombre-de-la-rama
```

- **¿Por qué `-d` minúscula?** Es la opción de borrado "seguro". Git verificará si esos cambios ya están en tu rama principal. Si hay código en esa rama que no ha sido guardado o subido a ningún lado, **Git se negará a borrarla** y te lanzará una advertencia.
    
- **El peligro:** La `D` mayúscula (`git branch -D nombre-de-la-rama`) es un borrado forzado. Git aniquilará la rama sin hacer preguntas, ignorando si el código estaba a salvo o no. Resérvalo solo para experimentos fallidos de los que te quieres deshacer.
    

### Resumen para no romper nada

1. Nunca uses `git reset --hard` para limpiar ramas, eso es para reescribir la historia.
    
2. Usa `git fetch -p` para limpiar la basura de internet.
    
3. Usa `git branch -d` (minúscula) para borrar tus ramas locales terminadas; deja que Git te proteja de tus propios errores.