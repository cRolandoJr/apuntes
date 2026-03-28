# Gestión de Servidores — SSH y Automatización Remota

Para administrar servidores remotos desde Python, el stack estándar es `paramiko` (cliente SSH de bajo nivel) o `fabric` (capa de alto nivel sobre paramiko).

---

## Instalación

```bash
uv add paramiko fabric
```

---

## `paramiko` — SSH de bajo nivel

```python
import paramiko
from pathlib import Path

def get_ssh_client(
    host: str,
    user: str,
    key_path: Path | None = None,
    password: str | None = None,
    port: int = 22,
) -> paramiko.SSHClient:
    client = paramiko.SSHClient()
    # Cargar known_hosts del sistema
    client.load_system_host_keys()
    # Para entornos de CI o IPs dinámicas:
    # client.set_missing_host_key_policy(paramiko.AutoAddPolicy())

    if key_path:
        client.connect(
            hostname=host,
            port=port,
            username=user,
            key_filename=str(key_path),
            timeout=10,
        )
    else:
        client.connect(
            hostname=host,
            port=port,
            username=user,
            password=password,
            timeout=10,
        )
    return client

# Ejecutar comando remoto
def run_remote(client: paramiko.SSHClient, command: str) -> tuple[str, str, int]:
    """Retorna (stdout, stderr, exit_code)."""
    stdin, stdout, stderr = client.exec_command(command)
    exit_code = stdout.channel.recv_exit_status()  # espera a que termine
    return (
        stdout.read().decode("utf-8"),
        stderr.read().decode("utf-8"),
        exit_code,
    )

# Uso
with get_ssh_client("192.168.1.100", "admin", key_path=Path("~/.ssh/id_ed25519").expanduser()) as client:
    out, err, code = run_remote(client, "df -h /")
    if code != 0:
        print(f"Error: {err}")
    else:
        print(out)
```

### SFTP — transferencia de archivos

```python
def upload_file(
    client: paramiko.SSHClient,
    local_path: Path,
    remote_path: str,
) -> None:
    with client.open_sftp() as sftp:
        sftp.put(str(local_path), remote_path)

def download_file(
    client: paramiko.SSHClient,
    remote_path: str,
    local_path: Path,
) -> None:
    with client.open_sftp() as sftp:
        sftp.get(remote_path, str(local_path))

def list_remote_dir(client: paramiko.SSHClient, path: str) -> list[str]:
    with client.open_sftp() as sftp:
        return sftp.listdir(path)
```

---

## `fabric` — automatización SSH de alto nivel

Fabric simplifica la ejecución de comandos remotos, manejo de sudo y transferencia de archivos.

```python
# fabfile.py
from fabric import Connection, task
from invoke import run as local
from pathlib import Path

SERVERS = {
    "web-01": "192.168.1.10",
    "web-02": "192.168.1.11",
    "db-01": "192.168.1.20",
}

def get_connection(host_alias: str) -> Connection:
    return Connection(
        host=SERVERS[host_alias],
        user="deploy",
        connect_kwargs={
            "key_filename": str(Path.home() / ".ssh" / "deploy_key"),
        }
    )

@task
def deploy(ctx, server="web-01", branch="main"):
    """Despliega la aplicación en el servidor."""
    with get_connection(server) as c:
        print(f"Desplegando {branch} en {server}...")

        with c.cd("/opt/mi_app"):
            c.run(f"git fetch origin {branch}")
            c.run(f"git reset --hard origin/{branch}")
            c.run("uv sync --frozen")
            c.run("uv run alembic upgrade head")
            c.sudo("systemctl restart mi-app")

        # Verificar que el servicio quedó activo
        result = c.sudo("systemctl is-active mi-app", warn=True)
        if result.stdout.strip() != "active":
            raise Exception(f"Servicio no está activo en {server}!")

        print(f"Despliegue exitoso en {server}")

@task
def deploy_all(ctx, branch="main"):
    """Despliega en todos los web servers."""
    for server in ["web-01", "web-02"]:
        deploy(ctx, server=server, branch=branch)

@task
def restart_nginx(ctx, server="web-01"):
    with get_connection(server) as c:
        c.sudo("nginx -t")  # verificar config antes de reiniciar
        c.sudo("systemctl reload nginx")

@task
def backup_db(ctx, output_dir="/backups"):
    """Crea un backup de la base de datos."""
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    remote_file = f"/tmp/backup_{timestamp}.sql.gz"
    local_file = Path(output_dir) / f"backup_{timestamp}.sql.gz"

    with get_connection("db-01") as c:
        c.run(
            f"pg_dump -U postgres mi_db | gzip > {remote_file}",
            env={"PGPASSWORD": "password"},
        )
        c.get(remote_file, str(local_file))
        c.run(f"rm {remote_file}")  # limpiar del servidor

    print(f"Backup guardado en: {local_file}")
```

```bash
# Usar fabfile:
fab deploy server=web-01 branch=main
fab backup-db output_dir=/backups
```

---

## Monitoreo de servicios

```python
# scripts/monitor.py
import subprocess
import psutil  # uv add psutil
from dataclasses import dataclass
from datetime import datetime

@dataclass
class ServiceStatus:
    name: str
    active: bool
    memory_mb: float | None
    cpu_percent: float | None
    uptime: str | None

def check_systemd_service(name: str) -> ServiceStatus:
    result = subprocess.run(
        ["systemctl", "is-active", name],
        capture_output=True, text=True
    )
    is_active = result.stdout.strip() == "active"

    # Obtener PID para métricas de proceso
    if is_active:
        pid_result = subprocess.run(
            ["systemctl", "show", name, "--property=MainPID"],
            capture_output=True, text=True
        )
        pid = int(pid_result.stdout.split("=")[1].strip())
        if pid > 0:
            proc = psutil.Process(pid)
            return ServiceStatus(
                name=name,
                active=True,
                memory_mb=proc.memory_info().rss / 1024 / 1024,
                cpu_percent=proc.cpu_percent(interval=1),
                uptime=str(datetime.now() - datetime.fromtimestamp(proc.create_time())),
            )

    return ServiceStatus(name=name, active=False, memory_mb=None, cpu_percent=None, uptime=None)

def get_disk_usage(path: str = "/") -> dict:
    usage = psutil.disk_usage(path)
    return {
        "path": path,
        "total_gb": usage.total / (1024**3),
        "used_gb": usage.used / (1024**3),
        "free_gb": usage.free / (1024**3),
        "percent": usage.percent,
    }

def check_all():
    services = ["nginx", "postgresql", "mi-app"]

    for svc in services:
        status = check_systemd_service(svc)
        if not status.active:
            print(f"ALERTA: {svc} no está activo")
            send_slack_alert(webhook_url, f"{svc} caído en {socket.gethostname()}", "CRITICAL")
        else:
            print(f"OK: {svc} - {status.memory_mb:.1f} MB RAM")

    disk = get_disk_usage("/")
    if disk["percent"] > 85:
        print(f"ALERTA: disco al {disk['percent']}%")
```

---

## Gestión de paquetes del sistema

```python
import subprocess

def apt_update():
    subprocess.run(["sudo", "apt-get", "update", "-qq"], check=True)

def apt_install(*packages: str) -> None:
    subprocess.run(
        ["sudo", "apt-get", "install", "-y", "--no-install-recommends", *packages],
        check=True, env={**os.environ, "DEBIAN_FRONTEND": "noninteractive"}
    )

def apt_is_installed(package: str) -> bool:
    result = subprocess.run(
        ["dpkg", "-s", package],
        capture_output=True, text=True
    )
    return result.returncode == 0

# Uso
if not apt_is_installed("git"):
    apt_update()
    apt_install("git", "curl", "build-essential")
```

---

## Firewall con `ufw` (Ubuntu)

```python
def ufw_allow(port: int | str, proto: str = "tcp") -> None:
    subprocess.run(["sudo", "ufw", "allow", f"{port}/{proto}"], check=True)

def ufw_deny(port: int | str, proto: str = "tcp") -> None:
    subprocess.run(["sudo", "ufw", "deny", f"{port}/{proto}"], check=True)

def ufw_status() -> str:
    result = subprocess.run(["sudo", "ufw", "status"], capture_output=True, text=True, check=True)
    return result.stdout

# Configurar servidor web
def setup_web_server_fw():
    ufw_allow(22)    # SSH
    ufw_allow(80)    # HTTP
    ufw_allow(443)   # HTTPS
    subprocess.run(["sudo", "ufw", "--force", "enable"], check=True)
    print(ufw_status())
```

---

## Ansible vs Fabric vs scripts Python

| Herramienta       | Cuándo usar                                                                   |
| ----------------- | ----------------------------------------------------------------------------- |
| **Ansible**       | Infraestructura declarativa, múltiples servidores, idempotente, equipo grande |
| **Fabric**        | Scripts de deploy específicos, Python puro, proyectos pequeños/medianos       |
| **Paramiko**      | Integrar SSH en una aplicación Python existente (control total del protocolo) |
| **Scripts puros** | Tarea simple, único servidor, no necesita SSH                                 |

Para empezar: Fabric es la mejor opción para un Developer que ya sabe Python.
