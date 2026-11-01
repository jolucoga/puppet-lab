# puppet-infra-lab

Repositorio de demostración de competencias en Puppet (OpenVox 8 / Puppet 8) sobre Red Hat Enterprise Linux 9 y VirtualBox. Cubre desde recursos básicos hasta infraestructura escalable con servidores de compilación, PuppetDB, Hiera 5, contenedores, orquestación con Bolt y extensión del lenguaje con código propio en Ruby.

Todo el contenido se construyó a partir de la guía práctica incluida en `docs/Puppet-Lab.md`, organizada en 12 fases. Cada directorio de este repositorio corresponde a una o varias de esas fases.

> **Estado de validación.** El código se redactó a partir de la documentación oficial vigente. Antes de publicarlo como evidencia, ejecutar cada fase en un entorno desechable y marcar en la tabla de avance (más abajo) lo que se haya comprobado. Los puntos señalados como "verificar" en la guía dependen de la versión exacta de los paquetes.

## Competencias demostradas

| Área | Evidencia |
|---|---|
| Lenguaje y recursos | `file`, `package`, `service`, `exec`, `cron`, metaparámetros y orden de aplicación |
| Módulos y buenas prácticas | Estructura con PDK, clases, parámetros tipados, datos de módulo, `puppet-lint` |
| Plantillas | EPP con parámetros y bucles |
| Escalado | Servidores de compilación, CA dedicada y balanceo con HAProxy |
| Datos y clasificación | Hiera 5 por entorno, ENC, `lookup` |
| Inventario y reporting | OpenVoxDB, recursos exportados, API REST |
| Contenedores | Servidor y agentes en contenedores, imágenes construidas con Puppet |
| Orquestación | OpenBolt: inventario, tareas y planes |
| Extensión | Hechos personalizados, tipos definidos, tipos nativos, proveedores y funciones |

## Entorno

| Elemento | Valor |
|---|---|
| Hipervisor | VirtualBox |
| Sistema operativo | RHEL 9 (suscripción Red Hat Developer) |
| Puppet | OpenVox 8 (`openvox-agent`, `openvox-server`, `openvoxdb`) |
| Orquestación | OpenBolt |
| Calidad | PDK, `puppet-lint`, `puppet parser validate`, `rspec-puppet` |

Las instrucciones para crear y clonar las VM (red NAT más solo anfitrión, direcciones `192.168.56.0/24`) están en la sección 0 y en la fase 1 de `docs/Puppet-Lab.md`.

## Estructura del repositorio

```text
puppet-infra-lab/
├── README.md
├── docs/
│   └── Puppet-Lab.md                 # guía completa, fases 1 a 12
├── environments/
│   └── production/
│       ├── environment.conf
│       ├── hiera.yaml                # Hiera 5 (fase 9)
│       ├── data/
│       │   ├── common.yaml
│       │   └── service/              # datos por rol: web.yaml, db.yaml (fases 9 y 10)
│       ├── manifests/
│       │   └── site.pp               # clasificación de nodos (fases 4, 9, 10, 11)
│       └── Puppetfile                # módulos de la Forge (fases 8 y 11)
├── site-modules/                     # módulos propios
│   ├── basics/                       # fases 2 y 3
│   │   └── manifests/                # files.pp, packages.pp, services.pp, exec.pp, cron.pp
│   ├── chrony/                       # fases 4, 5 y 6
│   │   ├── manifests/                # init.pp, install.pp, config.pp, service.pp, params.pp
│   │   ├── templates/chrony.conf.epp
│   │   ├── data/ y hiera.yaml        # datos de módulo
│   │   ├── examples/
│   │   └── spec/
│   ├── puppetserver_compiler/        # fase 7
│   │   └── manifests/                # ca.pp, compiler.pp, haproxy.pp
│   ├── nagios/                       # fase 8 (recursos exportados)
│   │   └── manifests/                # server.pp, client.pp, export.pp
│   ├── docker_prod/                  # fases 10 y 11
│   ├── git/                          # fase 11
│   ├── apps/                         # fases 11 y 12
│   │   ├── manifests/                # puppet_stack.pp, data_stack.pp, deploy.pp
│   │   └── files/                    # compose.yaml de cada pila
│   └── deploy_app/                   # fase 12
│       ├── lib/facter/server_type.rb
│       ├── lib/puppet/type/deploy_app.rb
│       ├── lib/puppet/provider/deploy_app/apache.rb
│       ├── lib/puppet/provider/deploy_app/tomcat.rb
│       └── lib/puppet/functions/metricbeat/check_elastic.rb
├── scripts/
│   ├── enc.rb                        # clasificador externo (fase 9)
│   └── puppetdb-queries.sh           # consultas curl + jq (fase 8)
├── containers/                       # fase 10
│   ├── server/Dockerfile
│   ├── agent/Dockerfile
│   ├── apache-image/{Dockerfile,manifests/,modules/}
│   ├── packer/apache.pkr.hcl
│   └── compose.yaml
├── bolt/                             # fase 11
│   ├── bolt-project.yaml
│   ├── inventory.yaml
│   └── site-modules/ops/plans/refresh.pp
├── infra/                            # fase 1 y 7
│   ├── virtualbox.md                 # parámetros de las VM y red
│   └── haproxy/haproxy.cfg
└── .github/workflows/lint.yml        # puppet-lint y validación de sintaxis
```

## Correspondencia con el laboratorio

| Fase | Tema de la guía | Ubicación en el repositorio |
|---|---|---|
| 0 y 1 | Entorno VirtualBox, RHEL 9, instalación de OpenVox, primer manifiesto | `infra/virtualbox.md` |
| 2 | Ficheros, paquetes, servicios, metaparámetros y orden | `site-modules/basics/` |
| 3 | `exec` y `cron` | `site-modules/basics/manifests/exec.pp` y `cron.pp` |
| 4 | Módulos, clases, tipos de datos y buenas prácticas | `site-modules/` (estructura PDK) |
| 5 | Módulo de sincronización horaria (chrony) | `site-modules/chrony/` |
| 6 | Plantillas EPP | `site-modules/chrony/templates/chrony.conf.epp` |
| 7 | Servidores de compilación, CA dedicada, balanceo | `site-modules/puppetserver_compiler/`, `infra/haproxy/` |
| 8 | OpenVoxDB, recursos exportados, API REST | `site-modules/nagios/`, `scripts/puppetdb-queries.sh` |
| 9 | Clasificación de nodos (ENC y Hiera 5) | `environments/production/hiera.yaml`, `data/`, `scripts/enc.rb` |
| 10 | Puppet con contenedores: servidor, agente, imágenes | `containers/`, `environments/production/data/service/` |
| 11 | Aplicaciones en contenedores y orquestación con Bolt | `site-modules/docker_prod/`, `git/`, `apps/`, `bolt/` |
| 12 | Hechos, tipos, proveedores y funciones | `site-modules/deploy_app/` |

## Cómo reproducirlo

1. Crear la VM base en VirtualBox y clonarla para cada nodo, según `infra/virtualbox.md`.
2. Registrar cada nodo con `subscription-manager` e instalar `openvox-agent` (y `openvox-server` en el servidor).
3. Copiar `environments/production` y `site-modules` a `/etc/puppetlabs/code/environments/production/` en el servidor, o clonar este repositorio allí.
4. Instalar los módulos de la Forge: `r10k puppetfile install` (o `puppet module install` para pruebas).
5. Ejecutar `puppet agent -t` en cada nodo y seguir el orden de las fases.
6. Validar antes de cada commit:

```bash
puppet parser validate $(git ls-files '*.pp')
puppet-lint --fail-on-warnings site-modules environments
pdk validate        # dentro de cada módulo
pdk test unit
```

## Seguimiento de avance

| Fase | Código escrito | Probado en VM | Notas |
|---|---|---|---|
| 1 a 3 | | | |
| 4 a 6 | | | |
| 7 | | | |
| 8 | | | |
| 9 y 10 | | | |
| 11 | | | |
| 12 | | | |

## Seguridad

- No se almacenan credenciales ni claves en el repositorio. Los secretos se gestionan con Hiera-eyaml o un gestor de secretos.
- La clave privada de la CA de Puppet no se versiona ni se copia entre servidores.
- Los ajustes solo para laboratorio (`autosign`, `host-key-check: false` en Bolt) están marcados en el código y no deben usarse en producción.

## Referencias

- Guía del laboratorio: `docs/Puppet-Lab.md`
- OpenVox: https://voxpupuli.org/openvox/install/
- Notas de versión de Puppet 8: https://help.puppet.com/core/current/Content/PuppetCore/PuppetReleaseNotes/release_notes_puppet_x-8-0-0.htm
