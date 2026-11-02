# Troubleshooting

Problemas reales encontrados al construir el pipeline, con su diagnóstico y solución.

## Resumen

| Síntoma | Causa | Solución |
|---|---|---|
| r10k despliega solo `README.md` | Los archivos no estaban en la rama `main` remota | Hacer commit y push desde el clon local, no desde el servidor |
| El agente aplica el catálogo sin mostrar el `notify` | La carpeta se llamó `manifest` (singular) | Renombrar a `manifests` |
| `validate` falla en segundos en el primer intento | `-v` en `gem install` se aplica a todos los gems de la línea | Separar en dos comandos |
| `shasum: command not found` | RHEL no incluye `shasum` | Usar `sha256sum -c` |
| `svc.sh: No such file or directory` | `svc.sh` lo genera `config.sh`; el registro no se completó o se está en otra carpeta | Ejecutar `config.sh` como `github-runner` en la carpeta correcta |
| El servicio del runner falla con `203/EXEC` | SELinux bloquea ejecutables con etiqueta `user_home_t` | Regla persistente con `semanage fcontext` y `restorecon` |
| `puppetserver` no arranca: `Permission denied - .../environments/production` | r10k elimina `production` y el servidor no puede recrearlo en un directorio de `root` | `chown puppet:puppet /etc/puppetlabs/code/environments` |
| Puerto 8140 rechaza conexiones tras reanudar la VM | Arranque en frío de Java y reloj desfasado | Esperar el arranque; `chronyc makestep` |
| El job `deploy` queda en cola | Runner desconectado con la VM apagada | Se ejecuta solo al reconectarse el runner |
| `fatal: not a git repository` en el equipo local | Se perdió la carpeta `.git` tras un fallo de VSCode | Volver a clonar y recuperar los cambios pendientes |

## Detalle

### r10k despliega solo el README

**Síntoma.** `ls /etc/puppetlabs/code/environments/main` muestra solo `README.md` y no existe `modules/`.

**Diagnóstico.** r10k despliega exactamente lo que hay en la rama remota. Se comprobó con `git ls-remote` que el hash de `main` no había cambiado.

**Causa.** Los archivos se habían creado, pero no se habían subido. El servidor no necesita un clon propio: r10k descarga el código por su cuenta.

**Solución.** Hacer `git add`, `git commit` y `git push origin main` desde el clon local, y repetir el despliegue.

### El `notify` no aparece

**Síntoma.** `puppet agent -t` compila y aplica el catálogo, pero sin recursos.

**Causa.** Puppet busca `manifests/` por defecto. La carpeta se había creado como `manifest/`, por lo que `site.pp` nunca se cargaba.

**Solución.** `git mv manifest manifests`, commit, push y nuevo despliegue.

### `validate` falla de inmediato

**Causa.** En `gem install puppet -v '~> 8' puppet-lint r10k`, la opción `-v` se aplica a todos los gems, por lo que pedía `puppet-lint ~> 8` y `r10k ~> 8`, versiones que no existen.

**Solución.**

```yaml
run: |
  gem install puppet -v '~> 8'
  gem install puppet-lint r10k
```

### Servicio del runner con `203/EXEC`

**Síntoma.** `svc.sh status` muestra `status=203/EXEC` y `Permission denied`.

**Diagnóstico.**

```bash
getenforce
ls -Z /home/github-runner/actions-runner/runsvc.sh
ausearch -m avc -ts recent | tail -n 5
```

**Causa.** Con SELinux en enforcing, systemd no puede ejecutar archivos con la etiqueta `user_home_t`.

**Solución.** Una regla persistente sobre toda la carpeta, ya que el servicio también ejecuta Node y los listeners:

```bash
semanage fcontext -a -t bin_t "/home/github-runner/actions-runner(/.*)?"
restorecon -Rv /home/github-runner/actions-runner
./svc.sh stop && ./svc.sh start
```

`chcon` funciona pero es temporal y se pierde con un `restorecon`. No se desactiva SELinux.

### `puppetserver` no arranca tras reiniciar la VM

**Síntoma.** El servicio entra en bucle de reinicios. El log indica:

```
File[/etc/puppetlabs/code/environments/production]: change from 'absent' to 'directory' failed:
Could not set 'directory' on ensure: Permission denied
```

**Diagnóstico.**

```bash
journalctl -u puppetserver --no-pager -o cat | grep -m1 "Could not set"
ls -ldZ /etc/puppetlabs/code/environments
ausearch -m avc -ts today
```

El directorio pertenecía a `root:root` y no había denegados de SELinux, lo que descarta esa causa.

**Causa.** Al arrancar, el servidor intenta crear el entorno `production` si no existe. r10k lo había eliminado por no estar administrado, y el servidor, que corre como `puppet`, no podía recrearlo.

**Solución.**

```bash
chown puppet:puppet /etc/puppetlabs/code/environments
systemctl start puppetserver
```

r10k seguirá eliminando `production` en cada despliegue y el servidor lo recreará al arrancar. Es inocuo porque los agentes usan `main`.

### Conexión rechazada en el puerto 8140

**Síntoma.** `puppet agent -t` falla con `Connection refused` y `ss -tlnp | grep 8140` no devuelve nada.

**Causa.** `puppetserver` seguía en `activating (start)`: el arranque en frío de Java tarda uno o dos minutos. Además, el reloj de la VM estaba atrasado tras reanudarla.

**Solución.** Sincronizar la hora y esperar:

```bash
chronyc makestep
timedatectl
systemctl status puppetserver
```

Para reducir el problema, apagar la VM con `poweroff` en lugar de guardar su estado.

### Job `deploy` en cola

**Causa.** El runner estaba desconectado porque la VM estaba apagada.

**Solución.** Verificar que el servicio del runner esté `active (running)` y que GitHub lo muestre como Idle. El trabajo pendiente arranca solo. Si ya había fallado, usar **Re-run jobs**.

### Carpeta `.git` perdida en el equipo local

**Síntoma.** `git status` responde `fatal: not a git repository`.

**Solución.** Lo ya subido sigue en GitHub. Renombrar la carpeta en lugar de borrarla, clonar de nuevo y copiar los cambios pendientes:

```bash
mv puppet-lab puppet-lab-old
git clone https://github.com/<usuario>/puppet-lab.git
diff puppet-lab-old/manifests/site.pp puppet-lab/manifests/site.pp
```
