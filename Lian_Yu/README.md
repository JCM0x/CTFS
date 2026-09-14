# <div align="center">[Lian_Yu - Writeup de TryHackMe](https://tryhackme.com/room/lianyu)</div>
<div align="center">Una sala de nivel principiante, con toda la onda de Arrow</div>
<div align="center">
  <img src="https://github.com/user-attachments/assets/af788293-2527-4656-8dc2-2db94d4e01b7" height="200" width="200"></img>
</div>

## Antes de empezar

Vamos a resolver juntos esta máquina. Se llama Lian_Yu y está inspirada en el universo de Arrow, así que si eres fan del show vas a notar varias referencias en el camino: nombres de archivos, usuarios, contraseñas. Todo tiene su guiño.

Es una sala pensada para quien apenas está empezando en esto de los CTF, pero no por eso es aburrida. Vamos a tocar un poco de todo: enumeración web, FTP, esteganografía, edición de archivos a mano con un editor hexadecimal, y al final una escalada de privilegios bien clásica. Perfecta para practicar la metodología de principio a fin.

### Resumen rápido de lo que vamos a encontrar

| Pregunta | Respuesta |
|---|---|
| Directorio web oculto | `2100` |
| Archivo con el ticket | `green_arrow.ticket` |
| Contraseña de FTP | `!#th3h00d` |
| Archivo con la contraseña SSH | `shado` |
| user.txt | `THM{P30P7E_K33P_53CRET5__C0MPUT3R5_D0N'T}` |
| root.txt | `THM{MY_W0RD_I5_MY_B0ND_IF_I_ACC3PT_YOUR_CONTRACT_THEN_IT_WILL_BE_COMPL3TED_OR_I'LL_BE_D34D}` |

Bueno, ahora sí, vamos paso a paso.

---

## Paso 1: reconocimiento inicial

Como siempre, lo primero es entender contra qué estamos jugando. Nada de lanzarse a probar cosas al azar. Primero hay que mapear bien la superficie de ataque.

Para eso corrí un escaneo con detección de versiones y scripts por defecto, así saco el máximo de información posible en una sola pasada, en vez de hacer algo básico y tener que volver después.

```bash
nmap -sV -sC 10.49.161.195
```

El resultado fue corto, pero bastante interesante. Cuatro servicios abiertos:

- **FTP** en el puerto 21
- **SSH** en el puerto 22
- Un **servidor web** en el puerto 80
- **RPC** en el puerto 111

Apenas vi FTP conviviendo con un servicio web, se me prendió el foco. Esa combinación suele ser terreno fértil para credenciales mal guardadas. Lo anoté mentalmente y seguí adelante.

---

## Paso 2: metiendo mano al sitio web

Con el puerto 80 abierto, lo lógico era abrir el navegador y ver qué había ahí.

<img width="1891" height="957" alt="image" src="https://github.com/user-attachments/assets/a483707b-ac54-47f3-a7f9-8b6c578a6ead" />

Y, como pasa casi siempre en estas salas, a simple vista no había nada. Ni formularios, ni enlaces raros, ni un comentario suelto por ahí. Una página bastante callada. Cuando pasa eso, ya uno sabe lo que toca hacer: cavar un poco más abajo de la superficie.

---

## Paso 3: fuerza bruta de directorios

Usé `dirsearch` con una wordlist conocida, de esas que casi nunca fallan para un primer barrido.

dirsearch -u http://10.49.161.195 -w SecLists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt

Target: http://10.49.161.195/

[20:13:50] Starting:
[20:14:30] 301 - 236B - /island -> http://10.49.161.195/island/
[20:18:22] 403 - 199B - /server-status

Task Completed


Y ahí apareció algo interesante:

/island


Un nuevo camino para explorar. El `/server-status` con 403 lo descarté rápido, ese tipo de rutas suele estar bien bloqueado y no valía la pena gastar tiempo ahí.

---

## Paso 4: pistas escondidas en el código fuente

Entré a `/island` y, en lugar de quedarme solo con lo que se veía en pantalla, fui directo a revisar el código fuente de la página. Es un hábito que vale la pena mantener siempre. Muchas veces el dato bueno está en un comentario HTML que nadie se acordó de borrar.

<img width="1920" height="406" alt="image" src="https://github.com/user-attachments/assets/ea6d9390-00e4-4cda-838c-a195bfe9d486" />

Y ahí estaba, escondida, esta palabra:

vigilante


Mi primera sospecha fue que se trataba de un nombre de usuario. Podía servir para FTP, para SSH, todavía no se sabía. Pero sin contraseña, por ahora no servía de mucho. Aun así, lo guardé como una pieza importante del rompecabezas y seguí enumerando.

---

## Paso 5: cavando un poco más hondo

Repetí la fuerza bruta, esta vez apuntando directamente a `/island`.

dirsearch -u http://10.49.161.195/island/ -w SecLists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt

Target: http://10.49.161.195/

[20:25:42] Starting: island/
[20:26:01] 301 - 241B - /island/2100 -> http://10.49.161.195/island/2100/

Task Completed


Nueva ruta encontrada:

/island/2100


La abrí en el navegador y, otra vez, a primera vista no había nada llamativo.

<img width="1640" height="565" alt="image" src="https://github.com/user-attachments/assets/dde7c989-0fd3-4bdf-9baa-5bcdaeedd7b1" />

Pero al revisar el código fuente, encontré una referencia a un archivo con extensión `.ticket`. Eso ya hizo sospechar que había algo escondido a propósito. Así que repetí la fuerza bruta, pero esta vez pidiéndole a la herramienta que probara específicamente esa extensión.

dirsearch -u http://10.49.161.195/island/2100 -w SecLists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -e .ticket

Target: http://10.49.161.195/

[20:25:42] Starting: island/2100
[20:26:01] 301 - 241B - /green_arrow.ticket -> http://10.49.161.195/island/2100/green_arrow.ticket

Task Completed


Y apareció el archivo que se estaba buscando:

/island/2100/green_arrow.ticket


---

## Paso 6: sacando las primeras credenciales

Al abrir el archivo, apareció una cadena codificada.

<img width="455" height="124" alt="image" src="https://github.com/user-attachments/assets/30091d5a-ec70-4d66-a0f1-d4f16ff20d3d" />

RTy8yhBQdscX


No parecía texto al azar, así que tocaba mirar el patrón con calma. Terminó identificándose como codificación Base58, la misma familia que usa Bitcoin para sus direcciones, por si querías el dato curioso. Al decodificarla salió el valor original.

<img width="1920" height="935" alt="image" src="https://github.com/user-attachments/assets/ac068391-589a-4e90-b50b-5517b49adad7" />

!#th3h00d


Con esto ya había en mano un posible usuario (`vigilante`) y una posible contraseña. Lo que seguía era simple: probar dónde encajaban.

---

## Paso 7: entrando por FTP

Con usuario y contraseña en mano, fui directo a probar el servicio FTP.

ftp 10.49.161.195

Name: vigilante
Password: !#th3h00d


Y adentro. El login funcionó a la primera, confirmando que las credenciales eran válidas para el acceso FTP.

### Explorando lo que había guardado ahí

Una vez adentro, listé el contenido disponible.

ls


Encontré tres imágenes:

- `Leave_me_alone.png`
- `Queen's_Gambit.png`
- `aa.jpg`

<img width="774" height="402" alt="image" src="https://github.com/user-attachments/assets/fa5d30d4-23f8-4ce8-baf7-2562fad95781" />

Las descargué todas para analizarlas con calma.

get Leave_me_alone.png
get Queen's_Gambit.png
get aa.jpg


### Un vistazo a los usuarios del sistema

Antes de meterme de lleno con las imágenes, aproveché que estaba adentro para revisar el directorio `/home` y entender qué usuarios había en el sistema.

cd /home
ls


Aparecieron dos:

- `slade`
- `vigilante`

Esto coincidía muy bien con el usuario que ya se había encontrado antes. Buena señal de que se iba por el camino correcto.

---

## Paso 8: analizando las imágenes descargadas

Se fueron revisando los archivos uno por uno. Uno de ellos llamó la atención de una vez.

`Leave_me_alone.png` no quería abrir. Ni el visor de imágenes ni las herramientas de metadatos hacían nada con él.

<img width="753" height="730" alt="image" src="https://github.com/user-attachments/assets/af6e431d-04d9-4014-ba41-723aba303fca" />

El error apuntaba a que el archivo estaba corrupto o manipulado a propósito. Para confirmarlo, se abrió en un editor hexadecimal.

<img width="1362" height="420" alt="image" src="https://github.com/user-attachments/assets/8edcf30b-d49e-4a43-871e-310cf8fcb42c" />

A simple vista no saltaba nada raro, pero al comparar la cabecera con la firma estándar de un PNG, el problema quedó claro: la cabecera estaba mal escrita.

<img width="824" height="512" alt="image" src="https://github.com/user-attachments/assets/d9f7fb6e-f828-4e06-a221-4905e20ecb56" />

Se corrigió a mano, byte por byte.

<img width="1367" height="342" alt="image" src="https://github.com/user-attachments/assets/cb72c22f-9e4c-494c-88e9-f92ee2ae9663" />

Y después de eso, la imagen abrió sin problema.

<img width="845" height="475" alt="Leave_me_alone" src="https://github.com/user-attachments/assets/a50b068c-0b6f-4b5a-a3e0-5742a1219a66" />

Adentro había una contraseña escondida:

password


---

## Paso 9: sacando información oculta con esteganografía

Se pasó a la otra imagen sospechosa. Para este archivo se usó `steghide`, una herramienta muy conocida para extraer datos escondidos dentro de imágenes.

steghide extract -sf aa.jpg


Con la contraseña que se había encontrado antes (`password`, del paso anterior), salió un archivo comprimido:

ss.zip


Al descomprimirlo aparecieron dos archivos.

cat passwd.txt
cat shado


**passwd.txt** decía esto:

This is your visa to Land on Lian_Yu # Just for Fun ***

a small Note about it

Having spent years on the island, Oliver learned how to be resourceful and
set booby traps all over the island in the common event he ran into dangerous
people. The island is also home to many animals, including pheasants,
wild pigs and wolves.


Y **shado** tenía esto:

M3tahuman


El segundo archivo tenía toda la pinta de ser una contraseña. Y por el nombre de usuario que ya se había visto en `/home`, era bastante claro a quién pertenecía.

---

## Paso 10: acceso de usuario por SSH

Con esta nueva credencial, se probó entrar por SSH.

ssh slade@10.49.161.195

Username: slade
Password: M3tahuman


Y adentro otra vez. Login exitoso, acceso de usuario confirmado en el sistema.

---

## Paso 11: la primera flag

Ya adentro, se listó el contenido del directorio home. Ahí estaba, esperando.

<img width="717" height="624" alt="image" src="https://github.com/user-attachments/assets/c367f4c8-afc5-47a5-9d7e-21e02ce6dd87" />

THM{P30P7E_K33P_53CRET5__C0MPUT3R5_D0N'T}


Con esto, el acceso de usuario quedaba resuelto. Ahora venía la parte más entretenida: subir a root.

---

## Paso 12: escalando privilegios

Con el acceso de usuario ya confirmado, tocaba ver qué margen de maniobra había. Lo primero que se revisa siempre en este punto es qué comandos se pueden correr con privilegios elevados.

```bash
sudo -l
```

<img width="1064" height="177" alt="image" src="https://github.com/user-attachments/assets/db28c1b3-b432-4641-9ad8-52c88913762b" />

Y ahí apareció: se podía ejecutar `/usr/bin/pkexec` con privilegios de root. Cada vez que aparece un binario permitido bajo sudo, el siguiente paso es siempre el mismo: revisar si ese binario tiene alguna forma conocida de ser aprovechado.

Para eso se recurrió a [GTFOBins](https://gtfobins.org), un recurso muy conocido en el mundo del pentesting que documenta cómo binarios comunes de Linux se pueden usar para saltarse restricciones y escalar privilegios en sistemas mal configurados. Vale la pena tenerlo siempre a la mano, salva en un montón de máquinas.

Se buscó `pkexec` ahí y apareció un método que funcionaba perfecto para levantar una shell como root.

<img width="959" height="518" alt="image" src="https://github.com/user-attachments/assets/424921d6-909c-4665-ac35-d5719a378723" />

La técnica era simple y directa:

```bash
sudo pkexec /bin/sh
```

¿Por qué funciona esto? Porque cuando `pkexec` se ejecuta a través de sudo, no suelta los privilegios elevados que ya tiene, y eso permite abrir una shell corriendo como root sin más vueltas. ([GTFOBins][2])

<img width="604" height="233" alt="image" src="https://github.com/user-attachments/assets/06f62b4d-7a25-4538-9729-184dedfa204c" />

Y listo, shell de root conseguida.

---

## Paso 13: la flag final

Con el acceso de root confirmado, se fue directo a buscar el último premio.

THM{MY_W0RD_I5_MY_B0ND_IF_I_ACC3PT_YOUR_CONTRACT_THEN_IT_WILL_BE_COMPL3TED_OR_I'LL_BE_D34D}


<img width="955" height="562" alt="image" src="https://github.com/user-attachments/assets/39784241-5f0e-42fd-9e55-5fce5aeb3042" />

---

## Algunas reflexiones finales

Esta sala es un buen ejemplo de por qué vale la pena mirar todo con calma y sin afán: un comentario en el código fuente, un archivo con extensión rara, una imagen que "no abre"... cada detalle terminó siendo una pieza del camino. Nada de exploits complicados ni técnicas raras, sino metodología ordenada: enumerar, observar, anotar, y probar credenciales en cada servicio que se va encontrando.

Si este recorrido te sirvió, vale la pena practicar Base58, esteganografía con `steghide` y explorar un rato GTFOBins en otras máquinas. Son herramientas que aparecen muchísimo en salas de nivel principiante e intermedio.

> Gracias por leer hasta acá.

<img width="860" height="384" alt="image" src="https://github.com/user-attachments/assets/9805ec51-6d15-4976-aec5-a3d6c490a3bb" />
