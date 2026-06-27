# Infraestructura y Entorno - Zorin OS 18 Pro

## Hardware
| Componente | Detalle |
|---|---|
| CPU/MB | AMD Ryzen 7 2700 / Asus TUF X470-Plus Gaming |
| RAM/GPU | 32GB DDR4 / NVIDIA GTX 1080 Ti (drivers privativos + CUDA, Rama 580+) |
| Boot/Root | Samsung NVMe 1TB |
| NAS Pool | 6× SSD SATA 1TB → Zpool ZFS (`nas_pool`) |
| Windows 11 | 1× NVMe 1TB (PCIe) + 1× M.2 SATA 500GB (boot independiente) |
| UPS | APC Back-UPS BX2200MI (USB `051d:0002` - Batería degradada, genera micro-sags) |
| Video | Emulador EDID HDMI (Passthrough) intercalado entre GPU y KVM. |

## Quirks & Hacks de Entorno (Estado Actual)

### 1. Video / KVM (Headless Bug, DPMS Off & Spin-Loop)
El KVM corta físicamente el pin HPD (Hot-Plug Detect) y el bus I2C/DDC al cambiar de canal. Esto causaba que procesos Chromium entraran en spin-loop, que Plymouth bloqueara el DRM Master, y que el driver de NVIDIA apagara el puerto (`Disp.A: Off`), matando el servidor Xorg sin posibilidad de recuperación en caliente.
- **Hardware Fix:** Se instaló un **Emulador EDID HDMI con passthrough** en la salida de la GPU para mantener el pin HPD siempre energizado y engañar al driver.
- **REGLA 1:** NO intentar leer EDID vía `sysfs` o `get-edid`, ni reconfigurar `xorg.conf`.
- **REGLA 2 (KMS Override & VRAM Preservation):**
  - **Módulo KMS (`/etc/modprobe.d/99-nvidia-kvm.conf`):** `options nvidia-drm modeset=1 fbdev=1`
  - **Módulo Power (`/etc/modprobe.d/99-nvidia-power.conf`):** `options nvidia NVreg_PreserveVideoMemoryAllocations=1`
  - **Servicios Systemd Activos:** `nvidia-suspend.service`, `nvidia-hibernate.service`, `nvidia-resume.service` (Cruciales para evitar corrupción de VRAM tras bloqueos largos de pantalla).
  - **GRUB (`/etc/default/grub.d/99-kvm-nvidia.cfg`):** `GRUB_CMDLINE_LINUX_DEFAULT="${GRUB_CMDLINE_LINUX_DEFAULT} video=HDMI-A-1:1920x1080@60D processor.max_cstate=5 rcu_nocbs=0-15"`.
- **Mitigación Plymouth:** Drop-in en `/etc/systemd/system/plymouth-quit-wait.service.d/override.conf` con `TimeoutSec=30` y `KillMode=control-group` para evitar bloqueos del nodo DRM en el arranque.

### 2. Estabilidad CPU/VRM (Ryzen C6 vs UPS Sags)
El kernel restringe el CPU a `processor.max_cstate=5 rcu_nocbs=0-15` para evitar *hardware halt* provocado por micro-caídas de tensión de la UPS cuando el procesador entra en reposo profundo (C6). 

### 3. NUT — Gestión de Energía UPS
Framework: NUT v2.8.1+ modo `standalone`. Reemplaza UPower y `apcupsd`. 
**Filtro Anti-Rebote (Debounce) Activo:** Configurado para ignorar caídas de voltaje <15s por degradación de batería.

**`/etc/nut/ups.conf`:**
- `pollinterval = 15` / `maxage = 25` — filtro de micro-lags y sags.
- `ignorelb` — ignora flag de batería baja mentiroso del hardware.
- `override.battery.charge.low = 10` — umbral real al 10%.

**`/etc/nut/upsmon.conf` (Lógica de apagado):**
- `POLLFREQ 15` / `POLLFREQALERT 15` / `DEADTIME 45`
- `POWERDOWNFLAG /etc/killpower`
- **Auto (10% carga):** detiene Docker → desmonta Zpool → `killpower` a la UPS.
- **Manual (`sudo poweroff`):** apagado estándar; UPS sigue energizando router.

## Especialización
ZFS/LVM/mounts · drivers NVIDIA para workloads IA · kernel/systemd para Ryzen/X470 · automatización NUT/UDEV/systemd para apagados críticos.

## Referencias Oficiales
- OpenZFS 2.x: https://openzfs.github.io/openzfs-docs/
- NUT Manual: https://networkupstools.org/documentation.html
- Ubuntu 24.04 Server: https://ubuntu.com/server/docs