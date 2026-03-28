# Automatización y CLI con Python

Typer (construido sobre Click) es el estándar moderno para CLI tools en Python. Usa type hints para definir argumentos y opciones automáticamente.

---

## Typer — CLI moderno con type hints

```bash
uv add typer[all]   # incluye rich para output bonito
```

### CLI básico

```python
# scripts/backup.py
import typer
from pathlib import Path
from typing import Annotated
import subprocess
import shutil
from datetime import datetime

app = typer.Typer(
    name="backup",
    help="Herramienta de backup para servidores",
    no_args_is_help=True,
)

@app.command()
def create(
    source: Annotated[Path, typer.Argument(help="Directorio a respaldar")],
    dest: Annotated[Path, typer.Argument(help="Destino del backup")],
    compress: Annotated[bool, typer.Option("--compress", "-c", help="Comprimir")] = True,
    name: Annotated[str | None, typer.Option(help="Nombre del backup")] = None,
):
    """Crea un backup de un directorio."""
    if not source.exists():
        typer.echo(f"Error: {source} no existe", err=True)
        raise typer.Exit(code=1)

    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    backup_name = name or f"backup_{timestamp}"

    dest.mkdir(parents=True, exist_ok=True)
    backup_path = dest / backup_name

    typer.echo(f"Creando backup: {source} → {backup_path}")

    with typer.progressbar(range(3), label="Progreso") as progress:
        shutil.copytree(source, backup_path)
        progress.update(1)
        if compress:
            shutil.make_archive(str(backup_path), "gztar", str(backup_path))
            shutil.rmtree(backup_path)
            backup_path = Path(str(backup_path) + ".tar.gz")
        progress.update(2)

    typer.secho(f"Backup completado: {backup_path}", fg=typer.colors.GREEN)

@app.command()
def list(
    backup_dir: Annotated[Path, typer.Argument()] = Path("/backups"),
):
    """Lista los backups disponibles."""
    if not backup_dir.exists():
        typer.echo("No hay backups.")
        return

    for f in sorted(backup_dir.iterdir()):
        size = f.stat().st_size / 1024 / 1024
        typer.echo(f"{f.name:40} {size:8.2f} MB")

if __name__ == "__main__":
    app()

# Uso:
# python backup.py create /var/app/data /backups --compress
# python backup.py list /backups
# python backup.py --help
```

---

## Subcomandos y apps anidados

```python
# cli/main.py — estructura de proyecto CLI más complejo
import typer

app = typer.Typer(help="Herramienta de administración")

# Sub-apps para cada dominio
db_app = typer.Typer(help="Comandos de base de datos")
server_app = typer.Typer(help="Comandos de servidor")

app.add_typer(db_app, name="db")
app.add_typer(server_app, name="server")

@db_app.command("migrate")
def db_migrate():
    """Aplica las migraciones pendientes."""
    subprocess.run(["alembic", "upgrade", "head"], check=True)
    typer.secho("Migraciones aplicadas.", fg=typer.colors.GREEN)

@db_app.command("seed")
def db_seed():
    """Carga datos iniciales."""
    ...

@server_app.command("restart")
def server_restart(
    service: str = typer.Argument(help="Nombre del servicio"),
    force: bool = typer.Option(False, "--force", "-f"),
):
    """Reinicia un servicio del sistema."""
    ...

if __name__ == "__main__":
    app()

# Uso:
# python main.py db migrate
# python main.py server restart nginx --force
```

---

## Automatización de tareas programadas

```python
# tasks/cleanup.py — script de limpieza
import logging
from pathlib import Path
from datetime import datetime, timedelta
import typer

logger = logging.getLogger(__name__)

def cleanup_old_logs(
    log_dir: Path,
    max_age_days: int = 30,
    dry_run: bool = False,
) -> int:
    """
    Elimina logs más viejos que max_age_days.
    Retorna la cantidad de archivos eliminados.
    """
    cutoff = datetime.now() - timedelta(days=max_age_days)
    deleted = 0

    for log_file in log_dir.rglob("*.log"):
        mtime = datetime.fromtimestamp(log_file.stat().st_mtime)
        if mtime < cutoff:
            if dry_run:
                logger.info("[DRY-RUN] Eliminaría: %s", log_file)
            else:
                size_mb = log_file.stat().st_size / 1024 / 1024
                log_file.unlink()
                logger.info("Eliminado: %s (%.2f MB)", log_file, size_mb)
            deleted += 1

    return deleted

app = typer.Typer()

@app.command()
def cleanup(
    log_dir: Path = typer.Argument("/var/log/app"),
    max_age: int = typer.Option(30, "--max-age", "-d", help="Días máximos"),
    dry_run: bool = typer.Option(False, "--dry-run", "-n", help="Solo mostrar, no eliminar"),
):
    count = cleanup_old_logs(log_dir, max_age, dry_run)
    msg = f"{'Eliminaría' if dry_run else 'Eliminados'} {count} archivos"
    typer.echo(msg)
```

### Configurar cron para el script

```bash
# Ver crontab actual
crontab -l

# Editar
crontab -e

# Ejemplos de cron entries:
# Ejecutar cada día a las 3:00 AM
0 3 * * * /usr/bin/python3 /opt/scripts/cleanup.py /var/log/app --max-age 30

# Usar uv run para respetar el entorno virtual
0 3 * * * cd /opt/mi_proyecto && /home/rolando/.local/bin/uv run python scripts/cleanup.py

# Redirigir logs del cron
0 3 * * * /opt/scripts/cleanup.py >> /var/log/cleanup.log 2>&1
```

---

## `schedule` — cron programático en Python

```python
# Para scripts que corren continuamente (como servicios)
# uv add schedule

import schedule
import time
import logging

logger = logging.getLogger(__name__)

def job_cleanup():
    logger.info("Iniciando limpieza...")
    deleted = cleanup_old_logs(Path("/var/log/app"))
    logger.info("Limpieza completada: %d archivos", deleted)

def job_backup():
    logger.info("Iniciando backup...")
    # ...

def job_health_check():
    result = subprocess.run(
        ["systemctl", "is-active", "nginx"],
        capture_output=True, text=True
    )
    if result.stdout.strip() != "active":
        logger.critical("nginx no está activo!")
        # enviar alerta...

# Configurar schedule
schedule.every().day.at("03:00").do(job_cleanup)
schedule.every().day.at("02:00").do(job_backup)
schedule.every(5).minutes.do(job_health_check)

if __name__ == "__main__":
    logger.info("Scheduler iniciado")
    while True:
        schedule.run_pending()
        time.sleep(30)
```

---

## Envío de alertas

```python
# Alertas por email
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart

def send_alert(subject: str, body: str, to_emails: list[str]) -> None:
    msg = MIMEMultipart()
    msg["From"] = "alerts@mi-empresa.com"
    msg["To"] = ", ".join(to_emails)
    msg["Subject"] = f"[ALERTA] {subject}"
    msg.attach(MIMEText(body, "plain"))

    with smtplib.SMTP(settings.smtp_host, settings.smtp_port) as server:
        server.starttls()
        server.login(settings.smtp_user, settings.smtp_password)
        server.send_message(msg)

# Alertas por Slack webhook (más común en equipos)
import httpx

def send_slack_alert(webhook_url: str, message: str, level: str = "INFO") -> None:
    emoji = {"INFO": ":information_source:", "ERROR": ":x:", "CRITICAL": ":rotating_light:"}
    payload = {
        "text": f"{emoji.get(level, '')} *[{level}]* {message}"
    }
    try:
        response = httpx.post(webhook_url, json=payload, timeout=5)
        response.raise_for_status()
    except Exception as e:
        # No queremos que una falla en la alerta mate nuestro script
        logger.error("No se pudo enviar alerta a Slack: %s", e)
```

---

## Leer y procesar archivos de configuración del servidor

```python
import re
from pathlib import Path

def parse_nginx_access_log(log_path: Path):
    """Parsea el log de acceso de nginx."""
    # Combined Log Format: IP - - [date] "METHOD PATH HTTP" status bytes
    pattern = re.compile(
        r'(?P<ip>\S+) \S+ \S+ \[(?P<time>[^\]]+)\] '
        r'"(?P<method>\S+) (?P<path>\S+) \S+" '
        r'(?P<status>\d+) (?P<bytes>\d+)'
    )

    with open(log_path) as f:
        for line in f:
            m = pattern.match(line)
            if m:
                yield m.groupdict()

def analyze_traffic(log_path: Path) -> dict:
    """Estadísticas del log de nginx."""
    from collections import Counter, defaultdict

    status_counts = Counter()
    path_counts = Counter()
    ip_counts = Counter()
    error_paths = defaultdict(list)

    for entry in parse_nginx_access_log(log_path):
        status = int(entry["status"])
        status_counts[status] += 1
        path_counts[entry["path"]] += 1
        ip_counts[entry["ip"]] += 1
        if status >= 400:
            error_paths[status].append(entry["path"])

    return {
        "total_requests": sum(status_counts.values()),
        "status_breakdown": dict(status_counts),
        "top_paths": path_counts.most_common(10),
        "top_ips": ip_counts.most_common(10),
        "error_paths": dict(error_paths),
    }
```
