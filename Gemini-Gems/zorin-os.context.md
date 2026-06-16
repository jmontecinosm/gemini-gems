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

## Quirks & Hacks de Entorno (Estado Actual)

### 1. Video / KVM (Headless Bug & Chromium Spin-Loop)
El KVM bloquea físicamente el bus I2C/DDC al cambiar de canal. La pérdida de EDID/Framebuffer causa que procesos Chromium (Brave, VSCode) entren en spin-loop (100% CPU `gpu-process`). 
- **REGLA 1:** NO intentar leer EDID vía `sysfs` o `get-edid`, ni reconfigurar `xorg.conf`.
- **REGLA 2 (KMS Override Limpio):**
  - **Módulo:** `/etc/modprobe.d/99-nvidia-kvm.conf` tiene `options nvidia-drm modeset=1 fbdev=1`.
  - **GRUB:** Archivo principal limpio. Override en `/etc/default/grub.d/99-kvm-nvidia.cfg` con: `GRUB_CMDLINE_LINUX_DEFAULT="${GRUB_CMDLINE_LINUX_DEFAULT} video=HDMI-A-1:1920x1080@60D processor.max_cstate=5 rcu_nocbs=0-15"`.
- **Aviso:** `nvidia-smi` arrojará deprecación en `display_mode` (esperado).

### 2. Estabilidad CPU/VRM (Ryzen C6 vs UPS Sags)
El kernel restringe el CPU a `processor.max_cstate=5 rcu_nocbs=0-15` para evitar *hardware halt* provocado por micro-caídas de tensión de la UPS cuando el procesador entra en reposo profundo (C6). 

## NUT — Gestión de Energía UPS
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