# Scripting con Python — SysAdmin

Python es el lenguaje estándar para automatización de sistemas, scripting de infraestructura y administración de servidores. Reemplaza bash para scripts complejos.

---

## `pathlib` — manejo de rutas (siempre esto, nunca `os.path`)

```python
from pathlib import Path

# Construir rutas
home = Path.home()                    # /home/rolando
config = Path("/etc/nginx/nginx.conf")
project = Path(__file__).parent.parent  # dos directorios arriba del script

# Operador / para construir rutas
log_dir = Path("/var/log") / "nginx"
access_log = log_dir / "access.log"

# Información de la ruta
print(config.name)        # "nginx.conf"
print(config.stem)        # "nginx"
print(config.suffix)      # ".conf"
print(config.parent)      # Path("/etc/nginx")
print(config.parts)       # ('/', 'etc', 'nginx', 'nginx.conf')

# Verificar existencia y tipo
config.exists()           # True/False
config.is_file()          # es archivo
config.is_dir()           # es directorio
config.is_symlink()       # es symlink

# Leer y escribir
text = config.read_text(encoding="utf-8")
config.write_text("nuevo contenido\n", encoding="utf-8")
bytes_data = config.read_bytes()

# Crear directorios
Path("/tmp/mi_app/logs").mkdir(parents=True, exist_ok=True)

# Listar archivos
for f in Path("/var/log").iterdir():
    print(f.name, f.stat().st_size)

# Glob — buscar patrones
logs = list(Path("/var/log").glob("*.log"))           # solo este dir
all_logs = list(Path("/var/log").rglob("*.log"))      # recursivo
py_files = list(Path(".").rglob("*.py"))

# Mover, copiar, eliminar
import shutil
shutil.copy2("/etc/nginx/nginx.conf", "/tmp/nginx.conf.bak")  # preserva metadata
shutil.copytree("/etc/nginx", "/tmp/nginx-backup")              # copia directorio
Path("/tmp/archivo").rename("/tmp/archivo_nuevo")              # mover/renombrar
Path("/tmp/archivo").unlink(missing_ok=True)                    # eliminar archivo
shutil.rmtree("/tmp/viejo_dir", ignore_errors=True)            # eliminar directorio

# Obtener información del archivo
stat = config.stat()
print(stat.st_size)     # bytes
print(stat.st_mtime)    # timestamp modificación
```

---

## `os` y `os.path` — operaciones del sistema

```python
import os
from pathlib import Path

# Variables de entorno
os.environ["PATH"]                        # variable PATH
os.getenv("DATABASE_URL", "default")      # con fallback
os.environ.setdefault("DEBUG", "false")   # solo si no existe

# Información del sistema
os.getpid()     # PID del proceso actual
os.getuid()     # UID del usuario (Unix)
os.getcwd()     # directorio actual (preferir Path.cwd())
os.listdir(".")  # listar dir (preferir Path.iterdir())

# Operaciones de archivos
os.chmod(path, 0o644)  # permisos
os.chown(path, uid=1000, gid=1000)  # propietario (necesita root o sudo)
os.stat(path).st_mode  # bits de modo

# Trabajar con permisos
import stat
file_stat = os.stat("/etc/nginx/nginx.conf")
perms = stat.filemode(file_stat.st_mode)  # '-rw-r--r--'
is_executable = bool(file_stat.st_mode & stat.S_IXUSR)

# Expandir ~ y variables de entorno
path = os.path.expanduser("~/Documents")  # /home/rolando/Documents
path = os.path.expandvars("$HOME/logs")   # /home/rolando/logs
# Con pathlib: Path("~/Documents").expanduser()
```

---

## `subprocess` — ejecutar comandos del sistema

```python
import subprocess
from pathlib import Path

# run() — ejecutar y esperar a que termine (versión moderna)
result = subprocess.run(
    ["ls", "-la", "/etc"],
    capture_output=True,   # captura stdout y stderr
    text=True,             # decode como string (no bytes)
    check=True,            # lanza CalledProcessError si returncode != 0
    timeout=30,            # segundos máximos
)
print(result.stdout)
print(result.returncode)  # 0 = éxito

# Con check=False — manejar errores manualmente
result = subprocess.run(["systemctl", "status", "nginx"], capture_output=True, text=True)
if result.returncode != 0:
    print(f"nginx no está corriendo: {result.stderr}")

# Pasar input por stdin
result = subprocess.run(
    ["python3", "-c", "import sys; print(sys.stdin.read().upper())"],
    input="hola mundo\n",
    capture_output=True,
    text=True,
)
print(result.stdout)  # "HOLA MUNDO\n"

# NUNCA pasar shell=True con datos del usuario — vulnerabilidad de inyección
# MAL:
def bad_disk_usage(path: str) -> str:
    result = subprocess.run(f"du -sh {path}", shell=True, capture_output=True, text=True)
    return result.stdout  # si path = "/ && rm -rf /" → desastre

# BIEN: lista de argumentos
def disk_usage(path: str) -> str:
    result = subprocess.run(
        ["du", "-sh", path],   # lista, no string
        capture_output=True, text=True, check=True
    )
    return result.stdout.strip()

# Procesar output línea a línea (para outputs grandes)
result = subprocess.run(["journalctl", "-n", "100"], capture_output=True, text=True)
for line in result.stdout.splitlines():
    if "ERROR" in line:
        print(line)
```

### Ejecutar proceso en background

```python
# Popen — para control más granular o procesos no bloqueantes
process = subprocess.Popen(
    ["tail", "-f", "/var/log/nginx/access.log"],
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE,
    text=True,
)

# Leer output a medida que llega
try:
    for line in process.stdout:
        print(line.strip())
        if "CRITICAL" in line:
            process.terminate()
            break
except KeyboardInterrupt:
    process.terminate()

process.wait()  # esperar que termine
```

---

## Lectura y escritura de archivos comunes

```python
import json
import csv
import configparser
from pathlib import Path

# JSON
config = json.loads(Path("config.json").read_text())
Path("output.json").write_text(json.dumps(data, indent=2, ensure_ascii=False))

# CSV
import csv

# Leer
with open("products.csv", newline="", encoding="utf-8") as f:
    reader = csv.DictReader(f)
    rows = list(reader)  # [{header: value, ...}, ...]

# Escribir
rows = [{"name": "Laptop", "price": "1500"}, {"name": "Mouse", "price": "30"}]
with open("output.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.DictWriter(f, fieldnames=["name", "price"])
    writer.writeheader()
    writer.writerows(rows)

# INI / config files
config = configparser.ConfigParser()
config.read("/etc/myapp/app.conf")
db_host = config.get("database", "host", fallback="localhost")

# YAML — con pyyaml
import yaml
with open("docker-compose.yml") as f:
    compose = yaml.safe_load(f)  # safe_load, nunca yaml.load()

# .env files — con python-dotenv
from dotenv import load_dotenv
load_dotenv("/etc/myapp/.env")
```

---

## Logging para scripts

```python
import logging
import sys
from pathlib import Path

def setup_logger(name: str, log_file: Path | None = None) -> logging.Logger:
    logger = logging.getLogger(name)
    logger.setLevel(logging.DEBUG)

    formatter = logging.Formatter(
        "%(asctime)s %(levelname)-8s %(name)s: %(message)s",
        datefmt="%Y-%m-%d %H:%M:%S"
    )

    # Handler para consola
    console_handler = logging.StreamHandler(sys.stdout)
    console_handler.setLevel(logging.INFO)
    console_handler.setFormatter(formatter)
    logger.addHandler(console_handler)

    # Handler para archivo (opcional)
    if log_file:
        from logging.handlers import RotatingFileHandler
        file_handler = RotatingFileHandler(
            log_file,
            maxBytes=10 * 1024 * 1024,  # 10 MB
            backupCount=5,
        )
        file_handler.setLevel(logging.DEBUG)
        file_handler.setFormatter(formatter)
        logger.addHandler(file_handler)

    return logger

logger = setup_logger("mi_script", log_file=Path("/var/log/mi_script.log"))
logger.info("Iniciando proceso de backup")
logger.error("Error al conectar con %s: %s", host, error)
logger.exception("Excepción no esperada")  # incluye traceback
```

---

## Práctica: Novato vs Profesional

### Novato

```python
# Script bash heredado en Python — peor de ambos mundos
import os
os.system(f"cp {source} {dest}")          # sin manejo de errores, sin captura
os.system(f"rm -rf {path}")               # sin verificación, puede ser ""
os.path.join("/var", "log", "app.log")    # os.path — anticuado

# Usar print en lugar de logging
print(f"procesando {file}")  # no tiene timestamp, no va a archivo, no tiene level
```

### Profesional

```python
import subprocess
from pathlib import Path

def safe_copy(source: Path, dest: Path) -> None:
    """Copia un archivo con manejo de errores."""
    if not source.exists():
        raise FileNotFoundError(f"Origen no existe: {source}")
    dest.parent.mkdir(parents=True, exist_ok=True)
    shutil.copy2(source, dest)
    logger.info("Copiado %s → %s", source, dest)

def safe_delete(path: Path) -> None:
    """Elimina un archivo con verificación."""
    if not path.exists():
        logger.warning("Archivo no existe, omitiendo: %s", path)
        return
    if path.is_dir():
        raise IsADirectoryError(f"Use rmtree para directorios: {path}")
    path.unlink()
    logger.info("Eliminado: %s", path)

# subprocess con lista (no shell=True con input del usuario)
subprocess.run(["systemctl", "reload", "nginx"], check=True)
```
