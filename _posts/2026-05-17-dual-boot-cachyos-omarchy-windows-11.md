---
title: Dual boot Windows 11 + CachyOS con Omarchy en dos NVMe sin perder nada
date: 2026-05-17 12:00:00 +0200
categories:
  - Linux
  - Tutoriales
tags:
  - arch
  - cachyos
  - omarchy
  - dual-boot
  - windows-11
  - hyprland
  - limine
  - btrfs
  - snapper
  - nvme
description: Guía paso a paso para instalar CachyOS con la capa Omarchy en un segundo NVMe junto a Windows 11, con verificación a prueba de fallos para no romper el arranque de Windows.
image:
  path: /assets/img/posts/cachyos-omarchy-dualboot.png
  alt: Dual boot Windows 11 + CachyOS con Omarchy en dos NVMe sin perder nada
pin: false
math: false
mermaid: false
share: true
---

## El setup

- **Placa:** ASUS TUF Gaming B650M-PLUS (AM5)
- **Disco 1:** Samsung SSD 980 PRO 1 TB → **Windows 11** (no se toca)
- **Disco 2:** Lexar SSD NM790 2 TB (nuevo) → **CachyOS + Omarchy**
- **Objetivo:** dual boot limpio, sin perder datos ni usuarios de Windows, sin romper el arranque.

La estrategia es la más segura posible: **cada SO en un disco distinto**. Dejamos el disco de Windows completamente intacto y trabajamos solo sobre el nuevo Lexar. Lo único compartido será la NVRAM UEFI y el menú de Limine.

## Lecciones aprendidas (léelas antes de empezar)

Estas son las cosas que descubrí por el camino y que casi nadie cuenta hasta que te explotan:

1. **Limine en CachyOS (enero 2026+) pide ESP de mínimo 4 GiB montado en `/boot`** (no `/boot/efi`, no 1 GiB). El cambio se hizo para que `limine-snapper-sync` quepa con sus snapshots de kernel. Si pones 1 GiB o lo montas en `/boot/efi`, Calamares te frenará.
2. **No uses "Erase Disk" en Calamares con dos discos puestos.** Usa **particionado manual** apuntando explícitamente al disco nuevo. Es la única forma de garantizar que no se toca nada del Samsung.
3. **Hay reportes de Omarchy/Limine sobrescribiendo la entrada UEFI de Windows** al instalar a un segundo disco. CachyOS es más cuidadoso, pero la verificación post-install no es opcional.
4. **BitLocker puede activarse solo en Windows 11** con cuenta Microsoft. Cambiar Secure Boot o el orden de arranque dispara la pantalla de recuperación. **Guarda la clave antes de tocar nada.**
5. **Fast Startup de Windows 11 corrompe NTFS** si Linux monta el disco de Windows. Hay que desactivarlo sí o sí.
6. **No instales GNOME ni KDE** si vas a usar el script `omarchy-on-cachyos`. Elige **Hyprland** o ningún DE.

## Fase 0 — Preparativos en Windows

### 0.1. Guardar la clave de BitLocker

```powershell
# PowerShell admin
manage-bde -status
```

Si el C: aparece cifrado, descarga la clave desde <https://account.microsoft.com/devices/recoverykey> e imprímela o guárdala fuera del PC (móvil, otro USB, papel).

### 0.2. Backup imagen del Samsung

Con [Macrium Reflect Free](https://www.macrium.com/reflectfree) o [Clonezilla](https://clonezilla.org/) clona el Samsung a un disco externo. **No es opcional** cuando los dos discos están dentro del PC durante la instalación.

### 0.3. Desactivar Fast Startup

Panel de control → Opciones de energía → "Elegir el comportamiento de los botones de inicio/apagado" → "Cambiar la configuración actualmente no disponible" → desmarcar **"Activar inicio rápido"** → Guardar.

### 0.4. Desactivar hibernación

```powershell
powercfg /h off
```

### 0.5. Apaga con Apagar de verdad

No reinicies. No cierres sesión. **Apaga.** Y ya que estás, ponle el Shift pulsado al hacer click en Apagar para forzar apagado completo.

## Fase 1 — BIOS (Supr al encender)

En la ASUS TUF B650M-PLUS, pulsa **F7** para pasar a Advanced Mode si entras en EZ Mode.

| Opción          | Ruta                                | Valor                                |
| --------------- | ----------------------------------- | ------------------------------------ |
| Secure Boot     | Boot → Secure Boot → OS Type        | **Other OS**                         |
| CSM             | Boot → CSM → Launch CSM             | **Disabled**                         |
| TPM             | Advanced → AMD fTPM                 | **Enabled**                          |
| Detección discos | Advanced → NVMe Configuration       | Comprobar que aparecen ambos         |

F10 → guardar → salir.

> Si al volver a Windows te pide la clave de BitLocker, es por el cambio de Secure Boot. Introduces la clave una vez y se reajusta solo.
{: .prompt-warning }

## Fase 2 — USB de CachyOS

1. Descargar ISO desde <https://cachyos.org/download/> (Desktop Edition).
2. Verificar SHA256.
3. Flashear con **Rufus** (modo DD) o **Ventoy**.

## Fase 3 — Pre-flight checks en el live USB

Arranca el live de CachyOS (**F8** al encender → elige el USB). **Antes de lanzar Calamares**, abre terminal:

### 3.1. Identifica los discos por modelo

```bash
lsblk -d -o NAME,SIZE,MODEL,SERIAL
```

Apunta en papel:

- `nvme0n1` → Samsung SSD 980 PRO → **NO TOCAR**
- `nvme1n1` → Lexar NM790 → **AQUÍ INSTALAMOS**

### 3.2. Inspecciona la estructura del Samsung

```bash
lsblk -o NAME,SIZE,FSTYPE,LABEL /dev/nvme0n1
```

Algo así:

```
NAME          SIZE FSTYPE LABEL
nvme0n1     931.5G
├─nvme0n1p1   100M vfat
├─nvme0n1p2    16M
├─nvme0n1p3 930.6G ntfs   NVME1-SO
└─nvme0n1p4   767M ntfs
```

Memoriza esta foto. Cuando termines la instalación, debe seguir exactamente igual.

### 3.3. Guarda el estado de la NVRAM

```bash
sudo efibootmgr -v > /tmp/nvram-antes.txt
cat /tmp/nvram-antes.txt
```

Apunta el UUID de la entrada **"Windows Boot Manager"**. Es tu seguro de vida si algo se descoloca.

## Fase 4 — Calamares con particionado manual

Lanza el instalador. Configura idioma, zona horaria, teclado.

### 4.1. En el paso de particiones, elige **Manual partitioning**

NO "Erase Disk", NO "Install alongside".

### 4.2. En el desplegable de disco, selecciona el **Lexar SSD NM790**

Verifica modelo en pantalla. Si seleccionas el Samsung aquí y le das a "Nueva tabla de particiones", habrás destruido Windows. No lo hagas.

### 4.3. Crea las particiones

**Partición 1 (ESP/boot):**

| Campo                | Valor                            |
| -------------------- | -------------------------------- |
| Tamaño               | **4500 MiB** (mínimo 4096)       |
| Sistema de archivos  | **FAT32**                        |
| Punto de montaje     | **`/boot`** (no `/boot/efi`)     |
| Flags                | **boot, esp** ← imprescindibles  |

> Si olvidas marcar `boot`, Calamares te suelta el error _"El sistema de archivos debe tener establecido el indicador boot"_. Edita la partición y márcalo.
{: .prompt-tip }

**Partición 2 (raíz):**

| Campo                | Valor                  |
| -------------------- | ---------------------- |
| Tamaño               | el restante (~1.86 TiB)|
| Sistema de archivos  | **Btrfs**              |
| Punto de montaje     | **`/`**                |
| Cifrado LUKS         | opcional               |

Las subvolúmenes (`@`, `@home`, `@snapshots`) las gestiona Calamares automáticamente para Snapper.

### 4.4. Elige bootloader: **Limine**

Es el default actual de CachyOS. Trae integración con Snapper y `limine-scan` para detectar Windows automáticamente.

### 4.5. Resto del instalador

- **Shell:** Fish
- **Desktop:** CachyOS Hyprland Desktop (instala SDDM)
- **Packages:** lo mínimo posible (Omarchy traerá lo suyo)
- **Usuario:** el tuyo

### 4.6. Resumen antes de Install

Lee el resumen entero. **Debe mencionar solo `/dev/nvme1n1`** (Lexar). Si aparece cualquier referencia a `nvme0n1`, **cancela**.

## Fase 5 — Verificación post-install (todavía en el live)

**Antes de reiniciar**, vuelve al terminal:

### 5.1. NVRAM intacta

```bash
sudo efibootmgr -v
```

Debe seguir apareciendo **"Windows Boot Manager"** con el mismo UUID que apuntaste en 3.3, además de la entrada nueva de Limine/CachyOS.

### 5.2. ESP de Windows intacto

```bash
sudo mkdir -p /mnt/win-esp
sudo mount /dev/nvme0n1p1 /mnt/win-esp
ls /mnt/win-esp/EFI/
sudo umount /mnt/win-esp
```

Salida esperada:

```
Boot  Microsoft
```

Si ves `Microsoft`, Windows arrancará perfecto. Si no, **avisa antes de reiniciar** — desde el live es mucho más fácil de arreglar.

## Fase 6 — Verificar dual boot ANTES de tocar Omarchy

Al reiniciar, **F8** en la BIOS para el menú UEFI. Deben aparecer:

- Limine (CachyOS)
- Windows Boot Manager

### 6.1. Arranca Windows primero

Si BitLocker pide clave, introduce la del paso 0.1. Verifica que entras, tus archivos están, tus usuarios están.

### 6.2. Arranca CachyOS

Login normal, red, etc.

> Si Windows no arranca aquí, **detente**. Restaura la imagen de Macrium del paso 0.2 y replantea. No instales Omarchy hasta confirmar que el dual boot funciona limpio.
{: .prompt-danger }

## Fase 7 — Integrar Windows en el menú de Limine

```bash
sudo limine-scan
```

Selecciona Windows cuando lo detecte. Esto añade la entrada al `/boot/limine.conf`.

```bash
sudo cat /boot/limine.conf
```

Debe aparecer una sección de Windows. Reinicia y verás Limine con ambas opciones sin necesidad de F8.

En BIOS, pon **Limine** como primer arranque si quieres que sea el menú por defecto.

## Fase 8 — Reloj UTC

Para que Windows y Linux no se peleen con la hora:

```bash
# En CachyOS
sudo timedatectl set-local-rtc 0
sudo hwclock --systohc --utc
```

Próxima vez que arranques Windows, PowerShell admin:

```powershell
reg add "HKLM\System\CurrentControlSet\Control\TimeZoneInformation" /v RealTimeIsUniversal /t REG_DWORD /d 1 /f
```

## Fase 9 — Update y luego Omarchy

```bash
sudo pacman -Syu
# reinicia si actualizó kernel
```

Solo cuando todo lo anterior está sólido:

```bash
git clone https://github.com/mroboff/omarchy-on-cachyos.git
cd omarchy-on-cachyos/bin
chmod +x install-omarchy-on-cachyos.sh
# Revisa el contenido antes de ejecutar
less install-omarchy-on-cachyos.sh
./install-omarchy-on-cachyos.sh
```

El script tarda 5–30 min. Si tienes NVIDIA, instala drivers 580xx propietarios automáticamente. Tras terminar, reinicia y ya tienes Omarchy + CachyOS + dual boot con Windows.

## Resumen de los puntos donde está toda la seguridad

1. **Particionado manual**, no Erase Disk. Tú decides qué se toca.
2. **ESP en `/boot`, 4 GiB mínimo, flags `boot,esp`** — los tres juntos.
3. **Verificación `efibootmgr` + ESP de Windows montado** antes de reiniciar.
4. **Probar dual boot completo antes de instalar Omarchy.**

## Referencias

- [omarchy-on-cachyos (mroboff)](https://github.com/mroboff/omarchy-on-cachyos)
- [Omarchy oficial](https://omarchy.org/)
- [CachyOS docs](https://wiki.cachyos.org/)
- [Limine bootloader](https://github.com/limine-bootloader/limine)
- [ASUS — Enable/Disable Secure Boot](https://www.asus.com/support/faq/1050047/)
