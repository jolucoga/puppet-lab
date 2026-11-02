# Configuración del servidor y del runner

Guía de instalación sobre RHEL 9. Todos los comandos se ejecutan como root en el servidor, salvo indicación contraria.

## 1. OpenVox Server

```bash
dnf install -y https://yum.voxpupuli.org/openvox8-release-el-$(rpm -E %rhel).noarch.rpm
dnf install -y openvox-server git
```

El servidor y el agente deben ser de la misma serie (8.x). Si el repositorio `openvox8-release` ya estaba instalado, basta con `dnf install -y openvox-server`.

Limitar el heap de Java en una VM con poca memoria, en `/etc/sysconfig/puppetserver`:

```
JAVA_ARGS="-Xms1g -Xmx1g"
```

Permisos del directorio de entornos. El servidor corre como usuario `puppet` y debe poder crear subdirectorios (por ejemplo `production`) al arrancar:

```bash
chown puppet:puppet /etc/puppetlabs/code/environments
```

Configurar el nombre del servidor y arrancar:

```bash
/opt/puppetlabs/bin/puppet config set server <fqdn-del-servidor> --section main
systemctl enable --now puppetserver
```

El primer arranque genera la CA y tarda uno o dos minutos. Verificación:

```bash
systemctl status puppetserver
ss -tlnp | grep 8140
/opt/puppetlabs/bin/puppet agent -t
```

## 2. Sincronización de hora

Un reloj desfasado impide que el runner se autentique con GitHub y puede invalidar certificados.

```bash
systemctl enable --now chronyd
chronyc makestep
timedatectl
```

`timedatectl` debe indicar `System clock synchronized: yes`.

## 3. r10k

```bash
/opt/puppetlabs/puppet/bin/gem install r10k
mkdir -p /etc/puppetlabs/r10k /var/cache/r10k
```

Se instala con el Ruby de OpenVox para que el binario quede en `/opt/puppetlabs/puppet/bin/r10k`, la ruta autorizada en el sudoers.

`/etc/puppetlabs/r10k/r10k.yaml` (repositorio público, clonado por HTTPS):

```yaml
cachedir: /var/cache/r10k
sources:
  lab:
    remote: https://github.com/<usuario>/puppet-lab.git
    basedir: /etc/puppetlabs/code/environments
```

Despliegue manual de prueba:

```bash
/opt/puppetlabs/puppet/bin/r10k deploy environment main -p -v
ls -A /etc/puppetlabs/code/environments/main
```

Deben verse `manifests`, `data`, `Puppetfile`, `environment.conf` y `modules/stdlib`.

Apuntar el agente al entorno `main`:

```bash
/opt/puppetlabs/bin/puppet config set environment main --section agent
/opt/puppetlabs/bin/puppet agent -t
```

## 4. Usuario del runner y sudoers

```bash
useradd -m -s /bin/bash github-runner
echo 'github-runner ALL=(root) NOPASSWD: /opt/puppetlabs/puppet/bin/r10k deploy environment main -p' \
  > /etc/sudoers.d/github-runner
chmod 440 /etc/sudoers.d/github-runner
visudo -cf /etc/sudoers.d/github-runner
```

Prueba (debe terminar sin pedir contraseña y con código 0):

```bash
sudo -u github-runner sudo -n /opt/puppetlabs/puppet/bin/r10k deploy environment main -p; echo "codigo: $?"
```

## 5. Registro del runner

En el repositorio: *Settings → Actions → Runners → New self-hosted runner*, Linux x64. La página muestra los comandos de descarga con la versión vigente y el token de registro.

Descargar y verificar la integridad como `github-runner` (en RHEL se usa `sha256sum`, no `shasum`):

```bash
su - github-runner
mkdir actions-runner && cd actions-runner
# ... comandos de descarga de la página de GitHub ...
echo "<hash>  <archivo>.tar.gz" | sha256sum -c
tar xzf ./<archivo>.tar.gz
./config.sh --url https://github.com/<usuario>/puppet-lab --token <TOKEN> --labels puppet --unattended
exit
```

El script `svc.sh` lo genera `config.sh` al registrarse con éxito; no viene en el tarball.

## 6. SELinux

Con SELinux en enforcing, los archivos del runner en `/home` tienen la etiqueta `user_home_t` y systemd no puede ejecutarlos (`status=203/EXEC`). Se resuelve con una regla persistente:

```bash
dnf install -y policycoreutils-python-utils
semanage fcontext -a -t bin_t "/home/github-runner/actions-runner(/.*)?"
restorecon -Rv /home/github-runner/actions-runner
```

## 7. Servicio del runner

```bash
cd /home/github-runner/actions-runner
./bin/installdependencies.sh
./svc.sh install github-runner
./svc.sh start
./svc.sh status
```

Debe aparecer `active (running)` y el log debe mostrar `Connected to GitHub` y `Listening for Jobs`. En GitHub, el runner aparece como **Idle** con la etiqueta `puppet`.

## 8. Protección del repositorio

- *Settings → Actions → General → Fork pull request workflows*: **Require approval for all outside collaborators**.
- *Settings → Rules → Rulesets → New branch ruleset*, sobre la rama por defecto:
  - Restrict deletions y Block force pushes.
  - Require a pull request before merging (0 aprobaciones si hay un solo colaborador).
  - Require status checks to pass: `validate`, con la rama actualizada.
  - No incluir `deploy` como check obligatorio: corre después de fusionar y bloquearía todas las fusiones.

## Verificación de extremo a extremo

1. Cambiar el mensaje del `notify` en `manifests/site.pp` en una rama nueva.
2. Abrir el PR: corre solo `validate`.
3. Fusionar: corren `validate` y `deploy`.
4. En el servidor: `grep notify /etc/puppetlabs/code/environments/main/manifests/site.pp` y `puppet agent -t`. La salida debe mostrar el mensaje nuevo.
