# Guía completa: Windows Server 2022 en LXD (Ubuntu 24.04) + acceso WebUI desde Mac

## 🧩 1. Prerrequisitos en Ubuntu 24.04

Asegúrate de tener actualizado el sistema:

``` bash
sudo apt update && sudo apt upgrade -y
```

Instala `lxd` (snap):

``` bash
sudo snap install lxd
sudo lxd init
```

> 💡 Si usas la instalación por defecto de Ubuntu 24.04, `lxd` ya viene
> preinstalado.\
> Verifica con `snap list lxd`.

------------------------------------------------------------------------

## ⚙️ 2. Configurar LXD (si es primera vez)

Durante `lxd init`: - Usa **dir** o **zfs** (recomendado) como storage
pool. - Habilita red **bridge (lxdbr0)** con DHCP. - No actives
clustering (a menos que lo necesites).

Verifica:

``` bash
lxc network list
lxc storage list
```

------------------------------------------------------------------------

## 💿 3. Descargar ISOs

Crea carpeta segura accesible por el snap:

``` bash
sudo mkdir -p /var/snap/lxd/common/lxd/disks
sudo chmod 755 /var/snap/lxd/common/lxd/disks
```

Descarga los ISOs:

``` bash
# Windows Server 2022 evaluation
wget -O /var/snap/lxd/common/lxd/disks/Windows_Server_2022.iso   "https://software-download.microsoft.com/download/pr/Windows_Server_2022_EVAL_x64.iso"

# VirtIO drivers (Fedora)
wget -O /var/snap/lxd/common/lxd/disks/virtio-win.iso   "https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso"

sudo chmod 644 /var/snap/lxd/common/lxd/disks/*.iso
```

------------------------------------------------------------------------

## 🖥️ 4. Crear la VM

``` bash
lxc launch images:windows/2022 win2022 --vm
```

> O usa tu propia imagen base si la tienes:
>
> ``` bash
> lxc launch win22-img win2022 --vm
> ```

Apaga la VM si arranca automáticamente:

``` bash
lxc stop win2022 --force
```

------------------------------------------------------------------------

## 🧩 5. Adjuntar los ISOs y corregir AppArmor

Permite lectura y file locking a QEMU:

``` bash
lxc config set win2022 raw.apparmor '
  /var/snap/lxd/common/lxd/disks/** rk,
  /var/lib/snapd/hostfs/etc/ssl/openssl.cnf r,
'
```

Adjunta los ISOs:

``` bash
lxc config device add win2022 winiso disk   source=/var/snap/lxd/common/lxd/disks/Windows_Server_2022.iso   boot.priority=10

# VirtIO por USB para evitar conflictos de bus
lxc config set win2022 raw.qemu -- " -device qemu-xhci,id=usbbus  -drive if=none,id=usbd,media=cdrom,readonly=on,file=/var/snap/lxd/common/lxd/disks/virtio-win.iso,file.locking=off  -device usb-storage,drive=usbd,bus=usbbus.0"
```

Desactiva Secure Boot (Windows lo reinstalará luego si quieres):

``` bash
lxc config set win2022 security.secureboot false
```

------------------------------------------------------------------------

## 🚀 6. Arrancar la VM

``` bash
lxc start win2022
```

Abre la consola VGA desde tu Mac:

    https://<IP_de_tu_host>:8443/ui

Ejemplo:

    https://192.168.55.43:8443/ui/project/default/instance/win2022/console

### 🧠 En tu Mac

No necesitas nada especial aparte de un navegador moderno (Safari,
Chrome, Edge).\
El frontend usa **SPICE over WebSockets**, gestionado por la Web UI de
LXD.\
Si prefieres, puedes usar también:

``` bash
lxc console win2022 --type=vga
```

------------------------------------------------------------------------

## 💽 7. En el instalador de Windows

1.  Cuando pida discos, presiona `Shift + F10` → ejecuta
    `diskpart → list volume`\
    → deberías ver:
    -   CD de instalación (Windows)
    -   CD USB (VirtIO)
2.  Si no tiene letra:\
    `select volume <N>` → `assign letter=E`
3.  Click **Browse → E:`\vioscsi`{=tex}\\2k22`\amd64`{=tex}** (o
    `w10\amd64` si no existe `2k22`)
4.  Instala el driver "Red Hat VirtIO SCSI Controller".

Después aparecerá tu disco y podrás seguir con la instalación normal.

------------------------------------------------------------------------

## 🧹 8. Limpieza post-instalación

Después de que Windows arranque por primera vez:

``` bash
lxc stop win2022
lxc config unset win2022 raw.qemu
lxc config device remove win2022 winiso
# (opcional)
# lxc config set win2022 security.secureboot true
lxc start win2022
```

------------------------------------------------------------------------

## 🛠️ 9. Drivers y extras dentro de Windows

Desde el ISO VirtIO (ahora montado): - **Red:** `NetKVM\w10\amd64` -
**Balloon:** `Balloon\w10\amd64` - **QEMU guest agent:** `qemu-ga` -
**VirtIO FS (opcional):** `viosf\w10\amd64`

------------------------------------------------------------------------

## ✅ Resumen final

  Componente                Estado
  ------------------------- -----------------------------------------
  ISO Windows Server 2022   ✅ Montado como CDROM
  ISO VirtIO                ✅ Montado por USB
  AppArmor                  ✅ Configurado con permisos `rk`
  Instalador                ✅ Detecta controladora `vioscsi`
  Consola Web               ✅ Accesible desde Mac sin dependencias


```bash
#!/usr/bin/env bash
# setup-win2022.sh
# Automática: LXD VM Windows Server 2022 + VirtIO en Ubuntu 24.04 (snap LXD)
set -euo pipefail

# ====== Parametrización ======
INSTANCE="${INSTANCE:-win2022}"
CPU="${CPU:-4}"
MEM="${MEM:-8GiB}"
ROOT_DISK="${ROOT_DISK:-60GiB}"

ISO_DIR="/var/snap/lxd/common/lxd/disks"
WIN_ISO="${WIN_ISO:-${ISO_DIR}/Windows_Server_2022.iso}"
VIRTIO_ISO="${VIRTIO_ISO:-${ISO_DIR}/virtio-win.iso}"

# URLs (puedes cambiarlas si ya tienes ISOs locales)
WIN_URL="${WIN_URL:-https://software-download.microsoft.com/download/pr/Windows_Server_2022_EVAL_x64.iso}"
VIRTIO_URL="${VIRTIO_URL:-https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso}"

# Puerto/UI de LXD
HTTPS_BIND="${HTTPS_BIND:-[::]:8443}"

# ====== Helpers ======
say() { printf "\n\033[1;32m==>\033[0m %s\n" "$*"; }
warn(){ printf "\n\033[1;33m!!\033[0m %s\n" "$*"; }
die() { printf "\n\033[1;31mXX\033[0m %s\n" "$*"; exit 1; }

need_bin() { command -v "$1" >/dev/null 2>&1 || die "Falta $1"; }

# ====== Comprobaciones básicas ======
need_bin snap
need_bin lxc
need_bin curl

# ====== Paso 0: Habilitar Web UI ======
say "Habilitando API/UI de LXD en ${HTTPS_BIND}"
sudo lxc config set core.https_address "${HTTPS_BIND}"

# ====== Paso 1: Carpeta y permisos para ISOs ======
say "Creando ${ISO_DIR} y permisos (snap confinement)"
sudo mkdir -p "${ISO_DIR}"
sudo chmod 755 "${ISO_DIR}"

# ====== Paso 2: Descargar ISOs si faltan ======
if [[ ! -f "${WIN_ISO}" ]]; then
  say "Descargando Windows Server 2022 → ${WIN_ISO}"
  sudo curl -L "${WIN_URL}" -o "${WIN_ISO}"
fi
if [[ ! -f "${VIRTIO_ISO}" ]]; then
  say "Descargando VirtIO drivers → ${VIRTIO_ISO}"
  sudo curl -L "${VIRTIO_URL}" -o "${VIRTIO_ISO}"
fi
sudo chmod 644 "${ISO_DIR}"/*.iso

# ====== Paso 3: Crear VM VACÍA (sin imagen), disco raíz y red ======
if lxc info "${INSTANCE}" >/dev/null 2>&1; then
  warn "La instancia ${INSTANCE} ya existe, se reutiliza."
else
  say "Creando VM vacía ${INSTANCE}"
  lxc init --empty "${INSTANCE}" --vm
fi

say "Configurando CPU/Memoria/Disco/Red"
lxc config set "${INSTANCE}" limits.cpu "${CPU}"
lxc config set "${INSTANCE}" limits.memory "${MEM}"

# root disk (idempotente)
if ! lxc config show "${INSTANCE}" | grep -q '^  root:'; then
  lxc config device add "${INSTANCE}" root disk pool=default size="${ROOT_DISK}" path=/
fi

# NIC sobre lxdbr0 (idempotente)
if ! lxc config show "${INSTANCE}" | grep -q '^  eth0:'; then
  lxc config device add "${INSTANCE}" eth0 nic network=lxdbr0 name=eth0
fi

# ====== Paso 4: AppArmor: lectura + file-lock de ISOs ======
say "Aplicando política AppArmor (rk) para permitir lectura y lock en ISOs"
lxc config set "${INSTANCE}" raw.apparmor $'
  /var/snap/lxd/common/lxd/disks/** rk,
  /var/lib/snapd/hostfs/etc/ssl/openssl.cnf r,
'

# ====== Paso 5: Adjuntar ISOs y forzar VirtIO por USB-CD ======
say "Adjuntando ISO de Windows como CD y VirtIO como USB-CD"
# Limpieza previa por si existe
lxc config device remove "${INSTANCE}" winiso  >/dev/null 2>&1 || true
lxc config unset  "${INSTANCE}" raw.qemu      >/dev/null 2>&1 || true

# CD de instalación (boot first)
lxc config device add "${INSTANCE}" winiso disk \
  source="${WIN_ISO}" boot.priority=10

# VirtIO como USB-CDROM (evita conflictos de bus) + sin file locking en QEMU
lxc config set "${INSTANCE}" raw.qemu -- "\
 -device qemu-xhci,id=usbbus \
 -drive if=none,id=usbd,media=cdrom,readonly=on,file=${VIRTIO_ISO},file.locking=off \
 -device usb-storage,drive=usbd,bus=usbbus.0"

# Secure Boot off durante instalación
lxc config set "${INSTANCE}" security.secureboot false

# ====== Paso 6: Arrancar e indicar URL de la consola ======
say "Arrancando ${INSTANCE}"
lxc start "${INSTANCE}"

HOST_IP=$(hostname -I | awk '{print $1}')
UI_URL="https://${HOST_IP}:8443/ui/project/default/instance/${INSTANCE}/console"
say "Abre la consola gráfica desde tu Mac en:
  ${UI_URL}

(usa Safari/Chrome; el certificado será autofirmado → continúa igualmente)"

cat <<'TIP'

== En el instalador de Windows ==
1) Cuando no detecte discos → "Load driver" → "Browse"
2) En el CD/USB de VirtIO selecciona:
   vioscsi\2k22\amd64   (o vioscsi\w10\amd64 si no existe 2k22)
3) Aceptar el “Red Hat VirtIO SCSI Controller” → aparecerá el disco.
4) Completa la instalación. Después instala desde el ISO de VirtIO:
   - NetKVM\w10\amd64  (red)
   - balloon\w10\amd64 (memoria)
   - qemu-ga           (agente invitado, opcional)
TIP

say "Listo."

```




Correccion paso 4 a mano

```bash
# 1) Crear VM vacía
lxc init --empty win2022 --vm

# 2) Recursos
lxc config set win2022 limits.cpu 4
lxc config set win2022 limits.memory 8GiB

# 3) Disco raíz (pool=default) y NIC
lxc config device add win2022 root disk pool=default size=60GiB path=/
lxc config device add win2022 eth0 nic network=lxdbr0 name=eth0

# 4) Permisos AppArmor (lectura/lock de ISOs en snap)
lxc config set win2022 raw.apparmor '
  /var/snap/lxd/common/lxd/disks/** rk,
  /var/lib/snapd/hostfs/etc/ssl/openssl.cnf r,
'

# 5) Adjuntar ISO de Windows como CD
lxc config device add win2022 winiso disk \
  source=/var/snap/lxd/common/lxd/disks/Windows_Server_2022.iso \
  boot.priority=10

# 6) Adjuntar VirtIO como USB-CD para evitar conflictos de bus
lxc config set win2022 raw.qemu -- "\
 -device qemu-xhci,id=usbbus \
 -drive if=none,id=usbd,media=cdrom,readonly=on,file=/var/snap/lxd/common/lxd/disks/virtio-win.iso,file.locking=off \
 -device usb-storage,drive=usbd,bus=usbbus.0"

# 7) Desactivar Secure Boot para la instalación
lxc config set win2022 security.secureboot false

# 8) Arrancar
lxc start win2022
```
