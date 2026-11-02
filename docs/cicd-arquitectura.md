# Arquitectura del despliegue continuo

Este documento describe cómo llega el código desde VSCode hasta el servidor OpenVox, qué decisiones se tomaron y qué alternativas se evaluaron.

## Objetivo

Que cualquier cambio fusionado a `main` se despliegue automáticamente en el servidor, con validación previa y sin exponer el servidor a internet.

## Componentes

| Componente | Función |
|---|---|
| VSCode + extensión Puppet | Edición local y resaltado de sintaxis |
| GitHub (repositorio público) | Control de versiones, PR, ruleset y Actions |
| Job `validate` | Corre en `ubuntu-latest`: `puppet parser validate`, `puppet-lint`, validación de YAML y `r10k puppetfile check` |
| Job `deploy` | Corre en el runner self-hosted (etiqueta `puppet`) y ejecuta r10k |
| r10k | Clona el repositorio y genera un entorno de Puppet por rama |
| OpenVox Server | Compila catálogos a partir del entorno desplegado |
| Agente | Solicita el catálogo con `puppet agent -t` |

## Flujo

```
VSCode → rama + PR → validate → merge a main → deploy (runner) → r10k → entorno main → agente
```

El job `deploy` depende de `validate` y solo se ejecuta en `push` a `main`. Los pull requests solo ejecutan la validación.

## Decisiones de diseño

- **Un entorno por rama.** r10k crea el entorno `main` a partir de la rama `main`. Los agentes se configuran con `environment = main`. Se evitó renombrar la rama a `production` para mantener la convención de GitHub.
- **El job `deploy` no hace `checkout`.** No necesita el código en el runner, porque r10k lo descarga directamente de GitHub. Reduce la superficie de ataque y el tiempo de ejecución.
- **`concurrency` en el despliegue.** Evita dos despliegues simultáneos sobre el mismo directorio.
- **`sudo` restringido al comando exacto.** `sudo` compara el comando de forma literal, por lo que el workflow invoca exactamente `r10k deploy environment main -p`.
- **Validación obligatoria.** Sin `|| true` en los pasos: cualquier fallo detiene el despliegue.
- **Versiones fijadas en el `Puppetfile`.** Los despliegues son reproducibles.

## Alternativas evaluadas

El servidor es una VM detrás de NAT, sin IP pública fija.

| Opción | Ventajas | Desventajas |
|---|---|---|
| **Runner self-hosted en el servidor (elegida)** | Sin SSH ni IP pública, sin secretos de acceso remoto, despliegue local | Hay que mantener el runner; con la VM apagada el job espera en cola |
| Cloudflare Tunnel + Access para SSH | Mantiene runners de GitHub; funciona con IP dinámica | Requiere dominio, túnel y política de Access; más piezas que mantener |
| Tailscale | Configuración simple, sin dominio | Servicio adicional; sigue necesitando SSH |
| Webhook de r10k | Despliegue casi instantáneo | Expone un endpoint HTTP y no valida antes de desplegar |
| Code Manager | Integrado y soportado | Solo en Puppet Enterprise, no aplica a OpenVox |

## Seguridad

- El repositorio es público y el runner vive en una VM propia, por lo que se exige aprobación para los workflows de colaboradores externos (*Settings → Actions → General*).
- El despliegue nunca se ejecuta desde pull requests.
- El runner usa un usuario sin privilegios y sin `sudo` general.
- El token de registro del runner es de un solo uso, caduca en una hora y no se almacena en el repositorio.
- La rama `main` está protegida con un ruleset: PR obligatorio, check `validate` requerido, sin force push ni borrado.

## Limitaciones y próximos pasos

- El pipeline despliega solo en el servidor donde corre el runner. Los servidores de compilación (fase 7) necesitarán su propia sincronización.
- Si la VM está apagada, los jobs `deploy` quedan en cola y se ejecutan al reconectarse el runner.
- r10k elimina los entornos que no provienen de ramas, incluido `production`. El servidor lo recrea al arrancar; es inocuo porque los agentes usan `main`. Se puede configurar r10k para excluirlo de la purga.
- Posible mejora: `paths-ignore` en el disparador `push` para que los cambios solo de documentación no desplieguen. No debe aplicarse al disparador `pull_request`, porque dejaría el check `validate` pendiente y bloquearía la fusión.
