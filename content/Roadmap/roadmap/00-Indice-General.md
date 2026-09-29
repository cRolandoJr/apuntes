# De SysAdmin Jr a Senior Platform Engineer — 24 Meses

> ⚠️ **HISTÓRICO (marcado 2026-07-20).** Este roadmap (mar-2026) fue superado por
> `~/plan-carrera-it.md`, que reprioriza por velocidad-a-ingreso (Wazuh/SOC y
> cloud+Terraform+CI/CD antes que la escalera completa de 8 fases, proyectos públicos
> antes que certs). Sirve como visión larga y material de referencia por fase;
> **el tracking y las decisiones viven en plan-carrera-it.md.**

> **Para**: Rolando, 26 años, Viedma, Río Negro
> **Punto de partida**: Tecnicatura SysAdmin + Software Libre, Tecnicatura en Desarrollo Web (UNCO). Primer trabajo IT. Conocimiento básico de Go y Docker.
> **Meta**: Senior Platform Engineer / DevOps Engineer
> **Duración**: 24 meses de estudio disciplinado (2-3 horas/día mínimo)

---

## ¿Qué es un Platform Engineer?

Un Platform Engineer construye y mantiene la **plataforma interna** que usan los developers para deployar, monitorear y operar sus aplicaciones. Es la evolución natural de DevOps: en vez de ser un "DevOps" que hace pipelines para cada equipo, creás la plataforma que permite a los equipos **ser autónomos**.

**El stack mental de un Platform Engineer**:

```
┌─────────────────────────────────────────┐
│ 8. Platform Engineering & Diseño        │  ← Diseñar la plataforma interna
│ 7. Observabilidad & SRE                 │  ← Garantizar confiabilidad
│ 6. IaC & Cloud                          │  ← Infraestructura como código
│ 5. Kubernetes & Orquestación            │  ← Orquestar contenedores
│ 4. Containers & CI/CD                   │  ← Empaquetar y entregar software
│ 3. Programación & Automatización        │  ← Escribir código de infra y herramientas
│ 2. SysAdmin Profesional                 │  ← Administrar sistemas en producción
│ 1. Linux & Redes (Fundamentos)          │  ← La base de todo
└─────────────────────────────────────────┘
```

Cada capa depende de las anteriores. No podés hacer Kubernetes bien si no entendés Linux. No podés hacer Platform Engineering si no entendés todo lo de abajo.

---

## Estructura de cada Fase

Cada fase dura **3 meses** y contiene:

- **Conceptos**: Explicación detallada de cada tema
- **Recursos**: Libros, cursos, documentación oficial (priorizando gratuitos)
- **Ejercicios**: Tareas verificables en terminal/playground
- **Proyecto de fase**: Un proyecto integrador que demuestra dominio
- **Checkpoint**: Cómo saber si estás listo para la siguiente fase

---

## Las 8 Fases

| Fase | Meses | Tema                           | Archivo                                   |
| ---- | ----- | ------------------------------ | ----------------------------------------- |
| 1    | 1-3   | Linux & Redes                  | [[Fase-01-Linux-y-Redes]]                 |
| 2    | 4-6   | SysAdmin Profesional           | [[Fase-02-SysAdmin-Profesional]]          |
| 3    | 7-9   | Programación & Automatización  | [[Fase-03-Programacion-y-Automatizacion]] |
| 4    | 10-12 | Containers & CI/CD             | [[Fase-04-Containers-y-CICD]]             |
| 5    | 13-15 | Kubernetes & Orquestación      | [[Fase-05-Kubernetes]]                    |
| 6    | 16-18 | Infrastructure as Code & Cloud | [[Fase-06-IaC-y-Cloud]]                   |
| 7    | 19-21 | Observabilidad & SRE           | [[Fase-07-Observabilidad-y-SRE]]          |
| 8    | 22-24 | Platform Engineering Senior    | [[Fase-08-Platform-Engineering]]          |

---

## Certificaciones Recomendadas (por orden)

Estas no son obligatorias pero validan tu conocimiento ante empleadores:

| Orden | Certificación                            | Cuándo | Costo aprox |
| ----- | ---------------------------------------- | ------ | ----------- |
| 1     | LPIC-1 (Linux Professional Institute)    | Mes 6  | ~$200 USD   |
| 2     | CKA (Certified Kubernetes Administrator) | Mes 15 | ~$395 USD   |
| 3     | Terraform Associate (HashiCorp)          | Mes 18 | ~$70 USD    |
| 4     | AWS Solutions Architect Associate        | Mes 20 | ~$150 USD   |
| 5     | CKS (Certified Kubernetes Security)      | Mes 24 | ~$395 USD   |

---

## Herramientas que Necesitás (instalá desde el mes 1)

```bash
# Tu lab personal - todo esto corre en tu PC
# Sistema operativo
- Linux (ya lo tenés)
- VirtualBox o KVM para crear VMs de práctica

# Herramientas base
- git, curl, wget, jq, vim/neovim
- tmux (multiplexor de terminal)
- htop, iotop, iftop (monitoreo)

# Containers
- Docker, docker-compose (ya lo tenés)
- Podman (alternativa sin daemon)

# Infraestructura
- Vagrant (para crear VMs automatizadas)
- Ansible (automatización)
- Terraform (infraestructura como código)

# Kubernetes
- minikube o k3s (cluster local)
- kubectl, helm

# Cloud
- Cuenta free tier de AWS (o GCP)
```

---

## Método de Estudio Recomendado

1. **Leer/ver** el concepto (30 min)
2. **Practicar** en terminal inmediatamente (60 min)
3. **Anotar** en tus apuntes de Obsidian lo que aprendiste (15 min)
4. **Repetir** el ejercicio al día siguiente sin mirar notas (30 min)
5. **Proyecto** semanal que integre lo aprendido

**Regla de oro**: Si no lo hiciste en la terminal, no lo aprendiste.

---

## Recursos Generales (los mejores, gratuitos)

| Recurso                                            | Qué es                                        | URL                                                |
| -------------------------------------------------- | --------------------------------------------- | -------------------------------------------------- |
| **The Linux Documentation Project**                | Guías profundas de Linux                      | tldp.org                                           |
| **Linux From Scratch**                             | Construir tu propio Linux                     | linuxfromscratch.org                               |
| **Networking Fundamentals - Practical Networking** | Videos de redes                               | practicalnetworking.net                            |
| **Go by Example**                                  | Ejercicios de Go                              | gobyexample.com                                    |
| **KillerCoda**                                     | Labs interactivos gratis (K8s, Docker, Linux) | killercoda.com                                     |
| **Kubernetes The Hard Way**                        | El tutorial definitivo de K8s                 | github.com/kelseyhightower/kubernetes-the-hard-way |
| **roadmap.sh/devops**                              | Roadmap visual de DevOps                      | roadmap.sh/devops                                  |
| **SadServers**                                     | Troubleshooting Linux en VMs reales           | sadservers.com                                     |
| **The SRE Book (Google)**                          | Biblia de Site Reliability                    | sre.google/sre-book                                |

---

Empezá por la **Fase 1**. No saltes fases aunque algo te parezca "fácil" — los fundamentos son lo que separa a un senior de un junior.
