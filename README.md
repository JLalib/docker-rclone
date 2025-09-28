# Rclone WebUI en Docker + OneDrive (headless)

Guía práctica para desplegar **rclone WebUI** en Docker, crear un remoto **OneDrive** sin navegador (modo *headless*), y (opcionalmente) **montarlo** como carpeta local. Incluye ejemplos de uso de la **API RC** para lanzar copias y sincronizaciones desde tu LAN.

> Probado con `rclone v1.71.1`, usuario `UID:GID = 1000:1000`, volumen de configuración en `/rclone/config` y acceso LAN vía `http://IP_DEL_HOST:5670`.

---

## Índice
1. [Requisitos](#requisitos)
2. [Estructura y permisos](#estructura-y-permisos)
3. [Docker Compose: WebUI (sin healthcheck)](#docker-compose-webui-sin-healthcheck)
4. [Arranque y acceso](#arranque-y-acceso)
5. [Crear remoto OneDrive (headless)](#crear-remoto-onedrive-headless)
6. [Verificar acceso](#verificar-acceso)
7. [Montar OneDrive como carpeta (opcional)](#montar-onedrive-como-carpeta-opcional)
8. [Ejecutar operaciones vía API RC](#ejecutar-operaciones-vía-api-rc)
9. [Añadir /data al servicio (opcional)](#añadir-data-al-servicio-opcional)
10. [Buenas prácticas y seguridad](#buenas-prácticas-y-seguridad)
11. [Solución de problemas](#solución-de-problemas)

---

## Requisitos
- **Docker** y (opcionalmente) **Docker Compose v2**.
- **Acceso a Internet** la primera vez (para descargar los assets de la WebUI).
- (Opcional para montar): Soporte **FUSE** en el host.

---

## Estructura y permisos
Crea la carpeta de configuración y la caché de la WebUI **persistentes** en el host:

```bash
mkdir -p /rclone/config/.cache/rclone/webgui
chown -R 1000:1000 /rclone/config
chmod -R 700 /rclone/config
```

> Guardaremos `rclone.conf` y la caché de la WebUI en ese volumen para evitar errores 404 por permisos.

---

## Docker Compose: WebUI (sin healthcheck)
`docker-compose.yml` mínimo y seguro para exponer la WebUI en la LAN.

```yaml
services:
  rclone-webui:
    image: rclone/rclone:latest
    container_name: rclone-webui
    user: "1000:1000"                 # tu UID:GID
    environment:
      - TZ=Europe/Madrid
      - XDG_CACHE_HOME=/config/rclone/.cache
    ports:
      - "5670:5670"                   # LAN: http://IP_DEL_HOST:5670
    volumes:
      - /rclone/config:/config/rclone  # aquí vive rclone.conf y la caché
    command: >
      rcd
      --rc-addr=:5670
      --rc-user=admin
      --rc-pass=CAMBIA_ESTA_PASSWORD   # pon una contraseña fuerte
      --rc-web-gui
      --rc-web-gui-update
      --rc-web-gui-no-open-browser
      --log-level=INFO
    restart: unless-stopped
```

> **Sin healthcheck**: así evitamos ventanas de login recurrentes por respuestas 401 en `/rc/*`.

---

## Arranque y acceso

```bash
docker compose up -d
docker logs -f rclone-webui
```

Deberías ver `Serving Web GUI` y el binding `http://[::]:5670/`.

Accede desde tu LAN (usa HTTP):
```
http://IP_DEL_HOST:5670/gui/
```

**Usuario**: `admin`  ·  **Contraseña**: la de `--rc-pass`

---

## Crear remoto OneDrive (headless)
Como no hay navegador en el contenedor, haremos la autorización desde tu PC.

1) Lanza el asistente interactivo dentro de un contenedor temporal:
```bash
docker run --rm -it   -v /rclone/config:/config/rclone   --user 1000:1000   rclone/rclone:latest config
```
2) Pasos en el asistente:
- `n` (new remote) → nombre: **onedrive**
- Backend: **Microsoft OneDrive** (elige en la lista)
- `client_id` / `client_secret`: deja vacíos si no tienes propios
- Región: **global** (habitual)
- Advanced config: **n**
- `Use web browser to automatically authenticate?` → **n** (No)

3) En tu **PC con navegador**, ejecuta:
```bash
rclone authorize "onedrive"
```
- Se abrirá el login de Microsoft en tu navegador.
- Al terminar, en la **terminal del PC** verás un **JSON** con los tokens.

4) Copia ese JSON y **pégalo** en `config_token>` donde espera el contenedor (TTY). Acepta guardar.

---

## Verificar acceso
Lista el contenido raíz de OneDrive:
```bash
docker run --rm -it   -v /rclone/config:/config/rclone   --user 1000:1000   rclone/rclone:latest lsd onedrive:
```

---

## Montar OneDrive como carpeta (opcional)
Requiere FUSE. Crea un servicio **separado** para el montaje.

1) Prepara la carpeta en el host:
```bash
sudo mkdir -p /mnt/onedrive
sudo chown 1000:1000 /mnt/onedrive
```

2) Añade este servicio al `docker-compose.yml`:
```yaml
  rclone-onedrive-mount:
    image: rclone/rclone:latest
    container_name: rclone-onedrive-mount
    user: "1000:1000"
    cap_add:
      - SYS_ADMIN
    devices:
      - /dev/fuse
    security_opt:
      - apparmor:unconfined
    environment:
      - TZ=Europe/Madrid
    volumes:
      - /rclone/config:/config/rclone
      - /mnt/onedrive:/mnt/onedrive:shared
    command: >
      mount onedrive: /mnt/onedrive
      --allow-other
      --dir-cache-time 72h
      --vfs-cache-mode writes    # recomendado para OneDrive
      --log-level=INFO
    restart: unless-stopped
```

> **No** montes en `/` ni uses `--allow-non-empty`. Para OneDrive, `--vfs-cache-mode writes` (o `full`) es recomendable.

---

## Ejecutar operaciones vía API RC
La WebUI usa la **API RC**. Puedes invocarla desde cualquier equipo de la LAN con `curl` (o `rclone rc`). Si quieres operar con rutas locales, mapea un volumen `/data` (ver siguiente sección).

**Listar la raíz de OneDrive**
```bash
curl -u admin:ADMIN_PASS -H 'Content-Type: application/json'   -d '{"fs":"onedrive:","remote":""}'   http://IP_DEL_HOST:5670/rc/operations/list
```

**Sincronizar carpeta local → OneDrive (en background)**
```bash
curl -u admin:ADMIN_PASS -H 'Content-Type: application/json'   -d '{"srcFs":"/data","dstFs":"onedrive:Backup","_async":true}'   http://IP_DEL_HOST:5670/rc/sync/sync
```

**Copiar un archivo suelto**
```bash
curl -u admin:ADMIN_PASS -H 'Content-Type: application/json'   -d '{"srcFs":"/data","srcRemote":"mi.txt","dstFs":"onedrive:","dstRemote":"mi.txt"}'   http://IP_DEL_HOST:5670/rc/operations/copyfile
```

**Consultar jobs async**
```bash
curl -u admin:ADMIN_PASS http://IP_DEL_HOST:5670/rc/job/list
```

> Con `rclone rc` también puedes invocar la API:
> ```bash
> rclone rc --url http://IP_DEL_HOST:5670/ --user admin --pass ADMIN_PASS sync/sync srcFs=/data dstFs=onedrive:Backup _async=true
> ```

---

## Añadir /data al servicio (opcional)
Si vas a usar la API RC para copiar/sincronizar desde rutas locales, añade un volumen `/data` al servicio **rclone-webui**:

```yaml
volumes:
  - /rclone/config:/config/rclone
  - /ruta/local/que/quieres/subir:/data
```

Así podrás referenciar `/data` en las llamadas RC (`srcFs=/data`).

---

## Buenas prácticas y seguridad
- Usa `--rc-user` y `--rc-pass` con **contraseñas largas**.
- Restringe el puerto **5670** en el **firewall** a tu subred.
- Haz **backup** de `/rclone/config/rclone.conf` (contiene tokens OAuth).
- Para cargas grandes en OneDrive, usa `--vfs-cache-mode writes/full`.
- Si más adelante pones proxy inverso (Traefik/Nginx) y **HTTPS**, mantén la auth de rclone y añade otra capa (BasicAuth/OIDC).

---

## Solución de problemas
- **404 en la WebUI al arrancar**: faltaban assets por permisos. Solución: definir `XDG_CACHE_HOME=/config/rclone/.cache` y crear `.cache` en el volumen (ver [Estructura y permisos](#estructura-y-permisos)).
- **Pide usuario/contraseña todo el rato**: suele venir de un *healthcheck* que llama a `/rc/*` sin credenciales → desactiva healthcheck o pásale `-u admin:pass`.
- **`mount FAILED ... directory already mounted`**: estabas montando en `/` o en un directorio no vacío → monta en `/mnt/onedrive` y no uses `--allow-non-empty`.
- **Aviso OneDrive `--vfs-cache-mode`**: usa `writes` o `full` para un comportamiento estable en montajes.

---


Recuerda:

Cambia --rc-pass=CAMBIA_ESTA_PASSWORD por una contraseña fuerte.

Antes del montaje, crea la carpeta del host y ajusta permisos:
```
sudo mkdir -p /mnt/onedrive
```
```
sudo chown 1000:1000 /mnt/onedrive
```

