# puppet-lab

Laboratorio de demostración de competencias en Puppet (OpenVox 8) sobre Red Hat Enterprise Linux 9 y VirtualBox. El repositorio es a la vez el *control repo* que r10k despliega: lo que se fusiona a `main` se publica solo en el servidor mediante GitHub Actions y un runner self-hosted.

> **Estado.** La capa de despliegue continuo está implementada y operativa. Los contenidos de Puppet (fases 2 a 12) están planificados y se irán añadiendo; la hoja de ruta indica qué está hecho y qué no.

## Demostrado hoy

| Área | Evidencia |
|---|---|
| CI/CD de infraestructura | Pipeline con GitHub Actions: validación en PR y despliegue solo al fusionar a `main` |
| Gestión de código Puppet | Control repo con r10k, `Puppetfile` con versiones fijadas, un entorno por rama |
| Despliegue sin IP pública | Runner self-hosted que solo hace conexiones salientes |
| Mínimo privilegio | Usuario dedicado para el runner y `sudo` limitado a un único comando |
| Administración de RHEL 9 | SELinux en enforcing, servicios con systemd, diagnóstico con `journalctl` y `ausearch` |
| Control del flujo | Ruleset en `main`: PR obligatorio y check `validate` requerido |
| Diagnóstico | Problemas reales documentados con causa y solución en [`docs/troubleshooting.md`](docs/troubleshooting.md) |

## Arquitectura del despliegue

```
VSCode (local)
    │  rama de trabajo + Pull Request
    ▼
GitHub ── Job validate (ubuntu-latest)
    │        sintaxis .pp, puppet-lint, YAML, Puppetfile
    │
    │  merge a main (solo si validate pasa)
    ▼
GitHub ── Job deploy (runner self-hosted, etiqueta "puppet")
    │        sudo r10k deploy environment main -p
    ▼
Servidor OpenVox (RHEL 9, VM en VirtualBox detrás de NAT)
    │  r10k genera /etc/puppetlabs/code/environments/main
    ▼
Agente: puppet agent -t  (entorno main)
```

Detalle y alternativas evaluadas en [`docs/cicd-arquitectura.md`](docs/cicd-arquitectura.md).

## Flujo de trabajo

1. Crear una rama desde `main` y editar en VSCode.
2. Subir la rama y abrir un Pull Request hacia `main`.
3. Corre el job `validate`; `deploy` se omite.
4. Con `validate` en verde, fusionar el PR.
5. El push resultante a `main` ejecuta `validate` y después `deploy` en el runner.
6. Comprobar en el agente con `puppet agent -t`.

## Estructura del repositorio

Estructura actual:

```text
puppet-lab/
├── .github/workflows/deploy.yml   # validación y despliegue
├── data/common.yaml               # datos de Hiera
├── manifests/site.pp              # clasificación de nodos
├── site-modules/                  # módulos propios
├── environment.conf               # modulepath del entorno
├── hiera.yaml                     # jerarquía de Hiera
├── Puppetfile                     # módulos de la Forge
└── docs/                          # documentación detallada
```

r10k toma la raíz del repositorio como raíz del entorno, por lo que `manifests/`, `data/`, `Puppetfile`, `environment.conf` y `hiera.yaml` deben permanecer en la raíz. Los directorios que Puppet no usa (`docs/`, y los previstos `scripts/`, `containers/`, `bolt/`, `infra/`) pueden convivir con ellos; r10k los copia al servidor y Puppet los ignora.

## Entorno

| Elemento | Valor |
|---|---|
| Hipervisor | VirtualBox |
| Sistema operativo | RHEL 9 |
| Puppet | OpenVox 8 (`openvox-agent`, `openvox-server`) |
| Despliegue | r10k + GitHub Actions (runner self-hosted) |
| Entorno de Puppet | `main` (r10k crea un entorno por rama) |

## Hoja de ruta de contenidos

| Fase | Tema | Estado | Probado en VM |
|---|---|---|---|
| Transversal | Despliegue continuo con GitHub Actions y r10k | Implementada | Parcial (ver nota) |
| 1 | Entorno VirtualBox, RHEL 9, OpenVox, primer manifiesto | En curso | Sí |
| 2 | Ficheros, paquetes, servicios, metaparámetros y orden | Planificada | |
| 3 | `exec` y `cron` | Planificada | |
| 4 | Módulos, clases, tipos de datos y buenas prácticas (PDK) | Planificada | |
| 5 | Módulo de sincronización horaria (chrony) | Planificada | |
| 6 | Plantillas EPP | Planificada | |
| 7 | Servidores de compilación, CA dedicada, balanceo con HAProxy | Planificada | |
| 8 | OpenVoxDB, recursos exportados, API REST | Planificada | |
| 9 | Clasificación de nodos: ENC y Hiera 5 | Planificada | |
| 10 | Puppet con contenedores | Planificada | |
| 11 | Aplicaciones en contenedores y orquestación con OpenBolt | Planificada | |
| 12 | Hechos, tipos, proveedores y funciones propias | Planificada | |

*Nota:* el despliegue por el runner y la aplicación del catálogo en el agente están comprobados; falta registrar la prueba de extremo a extremo con un cambio visible en `manifests/site.pp`.

Limitación conocida: el pipeline despliega solo en el servidor donde corre el runner. Los servidores de compilación de la fase 7 necesitarán su propia sincronización del código.

## Cómo reproducirlo

1. Instalar OpenVox Server y r10k, y configurar el runner: [`docs/cicd-servidor-y-runner.md`](docs/cicd-servidor-y-runner.md).
2. Apuntar los agentes al entorno `main`:

```bash
puppet config set environment main --section agent
```

3. Validar localmente antes de cada commit:

```bash
puppet parser validate $(git ls-files '*.pp')
puppet-lint manifests site-modules
```

## Seguridad

- No se almacenan credenciales, claves, IPs ni hostnames internos en el repositorio.
- El despliegue nunca se ejecuta desde pull requests; los workflows de colaboradores externos requieren aprobación.
- El usuario del runner solo puede ejecutar `r10k deploy environment main -p` con privilegios de root.
- Los ajustes solo para laboratorio estarán marcados en el código y no deben usarse en producción.

## Documentación

- [`docs/cicd-arquitectura.md`](docs/cicd-arquitectura.md): diseño, decisiones y alternativas.
- [`docs/cicd-servidor-y-runner.md`](docs/cicd-servidor-y-runner.md): instalación y configuración paso a paso.
- [`docs/troubleshooting.md`](docs/troubleshooting.md): problemas reales y su solución.

## Referencias

- Instalación de OpenVox: https://voxpupuli.org/openvox/install/
- r10k: https://github.com/puppetlabs/r10k
