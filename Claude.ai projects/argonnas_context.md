# Infraestructura y Entorno — ArgonNas
> Última verificación por diagnóstico completo: 2026-06-27

## Hardware
| Componente  | Detalle                                                          |
|-------------|-------------------------------------------------------------------|
| Host/User   | `ArgonNas` / `jorge`                                              |
| SBC         | Raspberry Pi 4 — 8GB RAM (7.6Gi visibles)                         |
| Carcasa     | Argon ONE (M.2 NVMe/SATA + ventilador)                            |
| OS          | **Debian GNU/Linux 13 (trixie)** — kernel `6.18.29+rpt-rpi-v8`    |
| Almacenamiento | SSD SATA M.2 1TB **WDC WDS100T2B0B-00YS70**, vía adaptador USB-SATA del Argon ONE (`/dev/disk/by-id/usb-Argon_Forty_...`) |

> **Corrección vs. doc anterior:** el sistema NO corre Raspberry Pi OS Bookworm — es Debian 13 trixie puro (probablemente reinstalado/migrado). Cualquier referencia a paquetes o rutas específicas de Raspberry Pi OS debe verificarse contra Debian trixie.

### Particionado real (verificado)
| Partición | FS   | Tamaño | Uso  | Mountpoint           |
|-----------|------|--------|------|----------------------|
| sda1      | vfat | 505M   | 13%  | `/boot/firmware`     |
| sda2      | ext4 | 63G    | 14%  | `/` (rootfs)          |
| sda3      | ext4 | 853G   | 86%  | `/srv/NAS/data`       |

**Atención:** `/srv/NAS/data` está al 86% de uso (696G/853G, ~114G libres). Vigilar crecimiento, especialmente Jellyfin/Transmission.

### Salud SSD (SMART — `smartctl`)
- Resultado: **PASSED**, 0 errores registrados, 0 sectores reasignados.
- Power-On Hours: 5827 h · Power Cycles: 379 · Pérdidas de energía inesperadas: 332.
- Temperatura: 34°C actual (rango histórico 19–75°C, umbral de fallo no alcanzado).
- Desgaste (Media_Wearout_Indicator): 1/100 — mínimo desgaste, disco sano.
- `Host_Writes_GiB`: 5111 · `Host_Reads_GiB`: 9212.

---

## Quirks & Hacks de Entorno

### 1. Térmico — Ventilador Argon ONE
| Umbral  | Velocidad |
|---------|-----------|
| < 55°C  | 0%        |
| 55°C    | 10%       |
| 60°C    | 55%       |
| ≥ 65°C  | 100%      |

Temperatura actual: 29–30°C, `throttled=0x0` (sin throttling). Curva funcionando correctamente en la práctica.

> **Pendiente de verificar:** el diagnóstico no encontró un servicio systemd `argononed` activo ni archivos de config bajo `/etc` con ese nombre — solo aparecieron rutas del disco (`usb-Argon_Forty_*`) y coincidencias irrelevantes (logs de Samba, stubs de Python). Confirmar manualmente con `systemctl list-units | grep -i fan`, `ps aux | grep -i argon`, o revisar `/boot/firmware/config.txt` para saber qué proceso controla realmente el ventilador.

### 2. Red y Seguridad

**IP/Red:** `eth0` → `10.0.0.182/24`, gateway `10.0.0.1`, DNS `10.0.0.1`, dominio de búsqueda `lan` (gestionado por NetworkManager, no solo `dhcpcd`).

**SSH (config activa vía `sshd -T`):**
| Parámetro              | Valor                  |
|-------------------------|------------------------|
| Port                    | 22                     |
| PermitRootLogin         | without-password (solo con llave) |
| PubkeyAuthentication    | yes                    |
| PasswordAuthentication  | **yes** (decisión deliberada del usuario) |

> Confirmado con Jorge: `PasswordAuthentication yes` es intencional, se mantiene por preferencia propia junto con auth por llave ED25519. No es un hallazgo a corregir.

**UFW (reglas activas, verificadas):**
| Regla                    | Acción   | Origen          | Notas                              |
|--------------------------|----------|-----------------|-------------------------------------|
| 22/tcp                   | ALLOW IN | 10.0.0.0/24      | SSH                                  |
| 137,138/udp              | ALLOW IN | 10.0.0.0/24      | Samba NetBIOS                        |
| 139,445/tcp              | ALLOW IN | 10.0.0.0/24      | Samba SMB                            |
| 8096/tcp                 | ALLOW IN | 10.0.0.0/24      | Jellyfin                             |
| 4533/tcp                 | ALLOW IN | 10.0.0.0/24      | Navidrome                            |
| 9443/tcp                 | ALLOW IN | 10.0.0.0/24      | Portainer (UI HTTPS)                 |
| 8000/tcp                 | ALLOW IN | 10.0.0.0/24      | Portainer (Edge Agent)               |
| 7878/tcp                 | ALLOW IN | 10.0.0.0/24      | Radarr                               |
| 8989/tcp                 | ALLOW IN | 10.0.0.0/24      | Sonarr                               |
| 9696/tcp                 | ALLOW IN | 10.0.0.0/24      | Prowlarr                             |
| 9091/tcp                 | ALLOW IN | 10.0.0.0/24      | Transmission (UI web)                |
| 51414/tcp, udp           | ALLOW IN | **Anywhere** (+v6) | Transmission P2P — expuesto a Internet a propósito |
| 1900/udp, 7359/udp       | ALLOW IN | 10.0.0.0/24      | Jellyfin (descubrimiento DLNA/cliente) |

Política default: `deny incoming`, `allow outgoing`, `deny routed`. Logging activo (low).

**Puertos reales en escucha (`ss -tulnp`):** coinciden 1:1 con las reglas UFW de arriba; sin servicios "fantasma" escuchando sin regla de firewall correspondiente.

### 3. Almacenamiento
- **Ruta base:** `/srv/NAS` — `775`, propietario `jorge:jorge`. Subcarpetas: `appdata/`, `config/`, `data/`.
- **Recurso Samba:** `[Media_SSD]` → `/srv/NAS/data/media`, `force user = jorge`, lectura/escritura.
- Samba 4.22.8-Debian, rol standalone server. `unix password sync = yes` — cuidado al usar `sudo` interactivo en la misma sesión donde se prueba Samba: un intento de contraseña incorrecta puede disparar el flujo de `passwd` vía PAM (visto en los logs durante el diagnóstico).

### 4. Stack Docker — Servicios Multimedia
> Engine: Docker 29.4.3 (build 055a478) · Compose v5.1.3 · Driver: overlayfs/containerd · CGroup v2.
> **Actualización disponible:** Docker CE 29.5.0 (ver sección Mantenimiento).
> Compose file activo: `/home/jorge/docker/docker-compose.yml`.
> Uso real de RAM del stack: **~1.1GB** (muy por debajo del límite de 3GB documentado).

| Servicio     | Imagen                                      | Puerto(s)                          | Mounts (host → contenedor)                                          |
|--------------|----------------------------------------------|-------------------------------------|-----------------------------------------------------------------------|
| Jellyfin     | `lscr.io/linuxserver/jellyfin:latest`        | 8096/tcp, 1900/udp, 7359/udp        | `/srv/NAS/config/jellyfin → /config`, `/srv/NAS/data/media → /data/media` |
| Navidrome    | `deluan/navidrome:latest`                    | 4533/tcp                            | `/srv/NAS/config/navidrome → /data`, `/srv/NAS/data/media/music → /music (ro)` |
| Sonarr       | `lscr.io/linuxserver/sonarr:latest`           | 8989/tcp                            | `/srv/NAS/config/sonarr → /config`, `/srv/NAS/data/media → /media`    |
| Radarr       | `lscr.io/linuxserver/radarr:latest`           | 7878/tcp                            | `/srv/NAS/config/radarr → /config`, `/srv/NAS/data/media → /media`    |
| Prowlarr     | `lscr.io/linuxserver/prowlarr:latest`         | 9696/tcp                            | `/srv/NAS/config/prowlarr → /config`                                  |
| Transmission | `lscr.io/linuxserver/transmission:latest`     | 9091/tcp (web), 51414/tcp+udp (P2P) | `/srv/NAS/config/transmission → /config`, `/srv/NAS/data/media/downloads → /downloads` |
| Portainer    | `portainer/portainer-ce:latest`               | 9443/tcp (UI), 8000/tcp (edge)      | volumen `portainer_data → /data`, `/var/run/docker.sock` (acceso total al daemon) |

**Redes Docker:**
- `docker-yml_default` (172.20.0.0/16) — **red activa**, todos los contenedores del compose actual cuelgan de aquí.
- `jorge_default` (172.19.0.0/16) y `servidores_medios_default` (172.18.0.0/16) — bridges **huérfanos**, sin interfaces activas (`NO-CARRIER`/`DOWN`), restos de proyectos compose anteriores. Candidatos a `docker network prune` tras confirmar que no los referencia nada.
- `bridge` (172.17.0.0/16) — red default de Docker, sin uso activo visible.

**Procedimiento reset admin Jellyfin** (sin cambios):
```bash
docker compose stop jellyfin
# Editar /srv/NAS/config/jellyfin/system.xml: vaciar o eliminar el bloque <DefaultUserId>
docker compose start jellyfin
# Acceder a http://argonnas:8096 y crear nuevo admin
```

> **Pendiente de confirmar:** no se pudo extraer límites de memoria/CPU, restart policy ni `privileged` por contenedor (error de template en `docker inspect` durante el diagnóstico). El compose file no parece definir límites explícitos — a confirmar leyendo directamente `/home/jorge/docker/docker-compose.yml`.

### 5. Audio
ArgonNas **no tiene salida de audio propia** ni requiere configuración "bit-perfect" local — confirmado con Jorge. La reproducción bit-perfect ocurre en el equipo de sonido externo del usuario, que lee los archivos directamente vía:
- **Samba** (`[Media_SSD]` → `/srv/NAS/data/media`), acceso directo a archivos sin transcodificación, o
- **Navidrome** (API Subsonic, puerto 4533), con transcodificación deshabilitada/opcional según el reproductor.

No hace falta `/etc/asound.conf` ni ajustes de ALSA en el host — los dispositivos de audio detectados (bcm2835 Headphones + 2x HDMI) son los de fábrica del Pi y no forman parte de la ruta de escucha real. Esta sección queda solo como referencia de que el host es agnóstico a audio; toda la responsabilidad de "bit-perfect" recae en el reproductor externo del usuario.

### 6. UPS / NUT
**No hay NUT instalado ni configurado en ArgonNas** — confirmado. Existe un UPS físico respaldando el equipo, pero actualmente **sin conexión USB** al Pi, por lo que no hay monitoreo ni apagado automático ante corte de energía. Si se quiere apagado seguro automatizado, sería necesario conectar el USB del UPS al Pi e instalar/configurar `nut-server`/`nut-client`. Mientras tanto, no hay protección por software contra cortes prolongados.

---

## Mantenimiento pendiente (detectado 2026-06-27)
- **62 paquetes con actualización disponible** vía `apt`, incluyendo paquetes de seguridad: `openssh-server/client` (10.0p1-7+deb13u2 → u4), `openssl` (3.5.5 → 3.5.6), `sudo` (1.9.16p2-3+deb13u1 → u2), `systemd` (257.9 → 257.13), y `docker-ce`/`docker-ce-cli` (29.4.3 → 29.5.0). Recomendado: `sudo apt update && sudo apt full-upgrade` en ventana de mantenimiento.
- **Sin backups automatizados detectados**: `crontab -l` del usuario vacío, `/etc/cron.d` solo contiene el job default `e2scrub_all` del sistema. No hay tarea programada para respaldar `/srv/NAS/config` (configs de todos los servicios) ni `appdata/`.
- **Redes Docker huérfanas** (`jorge_default`, `servidores_medios_default`) — limpieza recomendada con `docker network prune` tras verificar que no se usan.

## Especialización
RPi4 thermals · Docker ARM64 (7 contenedores LinuxServer.io + Navidrome + Portainer) · Samba/SMB · Jellyfin · Navidrome · Sonarr/Radarr/Prowlarr/Transmission · UFW · SSH hardening · Debian 13 trixie

## Referencias Oficiales
- Debian: https://www.debian.org/doc/
- Docker Engine: https://docs.docker.com/engine/install/debian/
- Jellyfin: https://jellyfin.org/docs/
- Navidrome: https://www.navidrome.org/docs/
- Sonarr/Radarr/Prowlarr (Servarr wiki): https://wiki.servarr.com/
- smartmontools: https://www.smartmontools.org/
