# Python para Backend y SysAdmin — Mapa de Aprendizaje

Guía profesional de Python enfocada en dos roles:

- **Developer backend**: FastAPI, SQLAlchemy, APIs REST/GraphQL, testing
- **SysAdmin/DevOps**: scripting, automatización, gestión de servidores, CLI tools

> Complementa los apuntes de Go: `[[00-MOC-Ruta-de-Aprendizaje]]`. Los conceptos de Clean Architecture, SOLID, y patrones aplican igual.

---

## Estructura de archivos

```
apuntes-python/
├── 00-MOC-Python-Ruta-de-Aprendizaje.md   ← este archivo
│
├── Etapa-0-Fundamentos-Python/
│   ├── 01-Tipos-y-Variables.md
│   ├── 02-Funciones-y-Decoradores.md
│   ├── 03-Clases-y-OOP.md
│   ├── 04-Modulos-y-Paquetes.md
│   ├── 05-Manejo-de-Errores.md
│   ├── 06-Iteradores-Generadores-Comprehensions.md
│   └── 07-Concurrencia-Asyncio.md
│
├── Etapa-1-Backend-FastAPI/
│   ├── 01-FastAPI-Fundamentos.md
│   ├── 02-Pydantic-y-Validacion.md
│   └── 03-Autenticacion-JWT.md
│
├── Etapa-2-Bases-de-Datos/
│   ├── 01-SQLAlchemy-ORM.md
│   └── 02-Alembic-Migraciones.md
│
├── Etapa-3-SysAdmin-Scripting/
│   ├── 01-Scripting-con-Python.md
│   ├── 02-Automatizacion-y-CLI.md
│   └── 03-Gestion-de-Servidores.md
│
├── Etapa-4-Testing/
│   └── 01-Testing-con-Pytest.md
│
├── Etapa-5-DevOps/
│   └── 01-Docker-Python.md
│
└── Extras/
    └── Recursos-Python.md
```

---

## Tabla de progresión

| Etapa | Tema               | Lo que podés hacer al dominarlo                                |
| ----- | ------------------ | -------------------------------------------------------------- |
| 0     | Fundamentos Python | Escribir código Python idiomático, type hints, async           |
| 1     | Backend FastAPI    | Construir APIs REST profesionales con validación y auth        |
| 2     | Bases de Datos     | CRUD con SQLAlchemy, migraciones con Alembic                   |
| 3     | SysAdmin Scripting | Automatizar tareas del servidor, gestionar archivos y procesos |
| 4     | Testing            | Pytest con fixtures, mocks, cobertura                          |
| 5     | DevOps             | Docker para apps Python, CI/CD                                 |

---

## Python vs Go — cuándo usar cada uno

| Tarea                          | Python                | Go                                       |
| ------------------------------ | --------------------- | ---------------------------------------- |
| Scripts de automatización      | ✅ Primera opción     | Posible pero verbose                     |
| APIs REST/GraphQL              | ✅ FastAPI, Django    | ✅ idéntico rendimiento con menos código |
| Data science / ML              | ✅ Único estándar     | No aplica                                |
| Herramientas CLI               | ✅ Click, Typer       | ✅ binario estático, más portable        |
| Servicios de alta concurrencia | Limitado por GIL      | ✅ goroutines                            |
| Scripting en servidores        | ✅ Siempre disponible | Requiere deploy del binario              |
| Interoperar con librerías C/ML | ✅ ctypes, cffi       | Más difícil                              |

---

## Setup del entorno

```bash
# Verificar versión (usar Python 3.11+ en proyectos nuevos)
python3 --version

# uv — el gestor de paquetes moderno (reemplaza pip + venv + poetry)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Crear proyecto nuevo con uv
uv init mi-proyecto
cd mi-proyecto
uv add fastapi uvicorn sqlalchemy psycopg2-binary

# Crear virtualenv con pip (alternativa clásica)
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# pyenv — manejar múltiples versiones de Python
curl https://pyenv.run | bash
pyenv install 3.12.0
pyenv local 3.12.0  # versión para este directorio
```

---

## Señales de que dominaste cada etapa

**Etapa 0**: Escribís type hints en todas las funciones. Entendés la diferencia entre lista/generador. Usás `async/await` correctamente sin bloquear el event loop.

**Etapa 1**: Podés construir una API con autenticación, validación y manejo de errores sin mirar documentación. Entendés cómo funcionan los middlewares en FastAPI.

**Etapa 2**: Podés modelar relaciones complejas en SQLAlchemy y escribir migraciones con Alembic. Sabés cuándo usar ORM vs SQL puro.

**Etapa 3**: Podés escribir scripts que automaticen tareas de servidor, interactúen con la CLI del OS y se ejecuten en remoto via SSH.

**Etapa 4**: Escribís tests con fixtures, mocks y cobertura >80% sin esfuerzo.

**Etapa 5**: Tus apps Python corren en Docker multi-stage con el menor tamaño posible.
