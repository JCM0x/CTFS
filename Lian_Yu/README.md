# <div align="center">[Lian_Yu - Writeup de TryHackMe](https://tryhackme.com/room/lianyu)</div>
<div align="center">Una sala de nivel principiante, con toda la onda de Arrow</div>
<div align="center">
  <img src="https://github.com/user-attachments/assets/af788293-2527-4656-8dc2-2db94d4e01b7" height="200" width="200"></img>
</div>

## Antes de empezar

Che, vamos a resolver juntos esta máquina. Se llama Lian_Yu y está inspirada en el universo de Arrow (Arrowverse), así que si sos fan vas a reconocer varias referencias en el camino: nombres de archivos, usuarios, contraseñas... todo tiene su guiño.

Es una sala pensada para quien recién arranca en CTFs, pero no por eso es aburrida. Vamos a tocar un poco de todo: enumeración web, FTP, esteganografía, edición de archivos a mano con un editor hexadecimal, y al final una escalada de privilegios bastante clásica. Perfecta para practicar metodología de principio a fin.

### Resumen rápido de lo que vamos a encontrar

| Pregunta | Respuesta |
|---|---|
| Directorio web oculto | `2100` |
| Archivo con el ticket | `green_arrow.ticket` |
| Contraseña de FTP | `!#th3h00d` |
| Archivo con la contraseña SSH | `shado` |
| user.txt | `THM{P30P7E_K33P_53CRET5__C0MPUT3R5_D0N'T}` |
| root.txt | `THM{MY_W0RD_I5_MY_B0ND_IF_I_ACC3PT_YOUR_CONTRACT_THEN_IT_WILL_BE_COMPL3TED_OR_I'LL_BE_D34D}` |

Ahora sí, vamos paso a paso.

---

## Paso 1: reconocimiento inicial

Como siempre, lo primero es entender contra qué estamos jugando. Nada de tirarse de cabeza a probar cosas al azar — primero hay que mapear la superficie de ataque.

Para eso corrí un escaneo con detección de versiones y scripts por defecto, así saco el máximo de información posible en una sola pasada, en lugar de hacer un escaneo básico y después tener que volver:

```bash
nmap -sV -sC 10.49.161.195
```

El resultado fue corto pero jugoso. Cuatro servicios expuestos:

- **FTP** en el puerto 21
- **SSH** en el puerto 22
- Un **servidor web** en el puerto 80
- **RPC** en el puerto 111

Apenas vi FTP conviviendo con un servicio web, se me prendió la lamparita. Esa combinación suele ser terreno fértil para credenciales mal guardadas o mal protegidas. Lo anoté mentalmente y seguí para adelante.

---

## Paso 2: metiendo mano en el sitio web

Con el puerto 80 abierto, lo lógico era abrir el navegador y ver qué había.

<img width="1891" height="957" alt="image" src="https://github.com/user-attachments/assets/a483707b-ac54-47f3-a7f9-8b6c578a6ead" />

Y, como pasa casi siempre en estas salas, a simple vista no había nada. Ni formularios, ni links raros, ni un comentario suelto. Una página bastante muda. Cuando pasa esto, ya sabés lo que toca: hay que cavar un poco más abajo de la superficie.

---

## Paso 3: fuerza bruta de directorios

Tiré `dirsearch` con una wordlist conocida, de esas que nunca fallan para un primer barrido:

dirsearch -u http://10.49.161.195 -w SecLists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt

Target: http://10.49.161.195/

[20:13:50] Starting:
[20:14:30] 301 - 236B - /island -> http://10.49.161.195/island/
[20:18:22] 403 - 199B - /server-status

Task Completed


Y ahí apareció algo interesante:

/island


Un nuevo camino para explorar. El `/server-status` con 403 lo descarté rápido, esas suelen estar bien bloqueadas y no vale la pena perder tiempo ahí en una sala como esta.

---

## Paso 4: pistas escondidas en el código fuente

Entré a `/island` y, en vez de quedarme solo con lo que se veía renderizado, fui directo a mirar el código fuente de la página. Es un hábito que vale la pena tener siempre: muchas veces el dato jugoso está en un comentario HTML que nadie borró.

<img width="1920" height="406" alt="image" src="https://github.com/user-attachments/assets/ea6d9390-00e4-4cda-838c-a195bfe9d486" />

Y bingo, ahí estaba escondida esta palabra:

vigilante


Mi primera sospecha fue que se trataba de un nombre de usuario. Podía servir para FTP, para SSH, quién sabe. Pero sin contraseña todavía no era útil por sí solo. Aun así, lo guardé como una pieza clave del rompecabezas y seguí enumerando.

---

## Paso 5: cavando un poco más hondo

Repetí la fuerza bruta, esta vez apuntando directamente a `/island`:

dirsearch -u http://10.49.161.195/island/ -w SecLists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt

Target: http://10.49.161.195/

[20:25:42] Starting: island/
[20:26:01] 301 - 241B - /island/2100 -> http://10.49.161.195/island/2100/

Task Completed


Nueva ruta descubierta:

/island/2100


La abrí en el navegador y a primera vista tampoco había nada llamativo.

<img width="1640" height="565" alt="image" src="https://github.com/user-attachments/assets/dde7c989-0fd3-4bdf-9baa-5bcdaeedd7b1" />

Pero otra vez, mirando el código fuente, encontré una referencia a un archivo con extensión `.ticket`. Eso ya me hizo sospechar que había algo escondido a propósito. Así que repetí la fuerza bruta, pero esta vez indicándole a la herramienta que probara específicamente esa extensión:

dirsearch -u http://10.49.161.195/island/2100 -w SecLists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -e .ticket

Target: http://10.49.161.195/

[20:25:42] Starting: island/2100
[20:26:01] 301 - 241B - /green_arrow.ticket -> http://10.49.161.195/island/2100/green_arrow.ticket

Task Completed


Y apareció el archivo que buscaba:

/island/2100/green_arrow.ticket


---

## Paso 6: sacando las primeras credenciales

Al abrir el archivo me encontré con una cadena codificada.

<img width="455" height="124" alt="image" src="https://github.com/user-attachments/assets/30091d5a-ec70-4d66-a0f1-d4f16ff20d3d" />

RTy8yhBQdscX


No parecía texto al azar, así que me puse a mirar el patrón con más cuidado. Terminé identificando que estaba en codificación Base58 (la misma familia que usa Bitcoin para sus direcciones, dato de color). La decodifiqué y salió a la luz el valor original:

<img width="1920" height="935" alt="image" src="https://github.com/user-attachments/assets/ac068391-589a-4e90-b50b-5517b49adad7" />

!#th3h00d


Con esto ya tenía en la mano un posible usuario (`vigilante`) y una posible contraseña. Lo que seguía era simple: probar dónde encajaban esas credenciales.

---

## Paso 7: entrando por FTP

Con usuario y contraseña en mano, fui directo a probar el servicio FTP.

ftp 10.49.161.195

Name: vigilante
Password: !#th3h00d


Y adentro. El login funcionó a la primera, confirmando que las credenciales eran válidas para acceso FTP.

### Explorando lo que había guardado ahí

Una vez dentro, listé el contenido disponible:

ls


Encontré tres imágenes:

- `Leave_me_alone.png`
- `Queen's_Gambit.png`
- `aa.jpg`

<img width="774" height="402" alt="image" src="https://github.com/user-attachments/assets/fa5d30d4-23f8-4ce8-baf7-2562fad95781" />

Las bajé todas para analizarlas con calma en mi máquina:

get Leave_me_alone.png
get Queen's_Gambit.png
get aa.jpg


### Un vistazo a los usuarios del sistema

Antes de meterme de lleno con las imágenes, aproveché que estaba adentro para chusmear el directorio `/home` y entender qué usuarios existían en el sistema:

cd /home
ls


Aparecieron dos:

- `slade`
- `vigilante`

Esto encajaba perfecto con el usuario que ya había encontrado antes. Buena señal de que iba por el camino correcto.

---

## Paso 8: analizando las imágenes descargadas

Me puse a revisar los archivos uno por uno. Uno de ellos llamó la atención de inmediato.

`Leave_me_alone.png` se negaba a abrir. Ni el visor de imágenes ni las herramientas de metadatos podían hacer nada con él.

<img width="753" height="730" alt="image" src="https://github.com/user-attachments/assets/af6e431d-04d9-4014-ba41-723aba303fca" />

El error apuntaba a que el formato estaba corrupto o manipulado a propósito. Para confirmarlo, abrí el archivo en un editor hexadecimal.

<img width="1362" height="420" alt="image" src="https://github.com/user-attachments/assets/8edcf30b-d49e-4a43-871e-310cf8fcb42c" />

A simple vista no saltaba nada raro, pero cuando comparé la cabecera con la firma estándar de un PNG, quedó claro el problema: la cabecera estaba mal escrita.

<img width="824" height="512" alt="image" src="https://github.com/user-attachments/assets/d9f7fb6e-f828-4e06-a221-4905e20ecb56" />

La corregí a mano, byte por byte.

<img width="1367" height="342" alt="image" src="https://github.com/user-attachments/assets/cb72c22f-9e4c-494c-88e9-f92ee2ae9663" />

Y después de eso, la imagen abrió sin problemas.

<img width="845" height="475" alt="Leave_me_alone" src="https://github.com/user-attachments/assets/a50b068c-0b6f-4b5a-a3e0-5742a1219a66" />

Dentro había una contraseña escondida:

password


---

## Paso 9: sacando información oculta con esteganografía

Me fui a la otra imagen sospechosa. Para este tipo de archivo usé `steghide`, una herramienta clásica para extraer datos escondidos dentro de imágenes:

steghide extract -sf aa.jpg


Le di la contraseña que había encontrado antes (`password`, del paso anterior) y extrajo un archivo comprimido:

ss.zip


Lo descomprimí y aparecieron dos archivos:

cat passwd.txt
cat shado


**passwd.txt** decía esto:

This is your visa to Land on Lian_Yu # Just for Fun ***

a small Note about it

Having spent years on the island, Oliver learned how to be resourceful and
set booby traps all over the island in the common event he ran into dangerous
people. The island is also home to many animals, including pheasants,
wild pigs and wolves.


Y **shado** contenía esto:

M3tahuman


El segundo archivo tenía toda la pinta de ser una contraseña. Y por el nombre de usuario que ya había visto antes en `/home`, era bastante evidente a quién pertenecía.

---

## Paso 10: acceso de usuario por SSH

Con esta nueva credencial, probé entrar por SSH:

ssh slade@10.49.161.195

Username: slade
Password: M3tahuman


Y adentro otra vez. Login exitoso, acceso de usuario confirmado en el sistema.

---

## Paso 11: la primera flag

Ya adentro, listé el contenido del directorio home. Ahí estaba, esperando:

<img width="717" height="624" alt="image" src="https://github.com/user-attachments/assets/c367f4c8-afc5-47a5-9d7e-21e02ce6dd87" />

THM{P30P7E_K33P_53CRET5__C0MPUT3R5_D0N'T}


Con esto, el acceso de usuario quedaba cerrado. Ahora venía la parte más entretenida: subir a root.

---

## Paso 12: escalando privilegios

Con acceso de usuario ya confirmado, tocaba ver qué margen de maniobra tenía. Lo primero que reviso siempre en este punto es qué comandos puedo correr con privilegios elevados:

```bash
sudo -l
```

<img width="1064" height="177" alt="image" src="https://github.com/user-attachments/assets/db28c1b3-b432-4641-9ad8-52c88913762b" />

Y ahí apareció: podía ejecutar `/usr/bin/pkexec` con privilegios de root. Cada vez que aparece un binario permitido bajo sudo, mi reflejo es siempre el mismo: ir a chequear si ese binario tiene alguna forma conocida de ser abusado.

Para eso recurrí a [GTFOBins](https://gtfobins.org), un recurso muy conocido en el mundo del pentesting que documenta cómo binarios comunes de Linux pueden usarse para saltarse restricciones y escalar privilegios en sistemas mal configurados. Si todavía no lo tenés en tus favoritos, hacelo ya, te va a salvar en un montón de máquinas.

Busqué `pkexec` ahí y encontré un método que funcionaba de diez para levantar una shell como root.

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

Con acceso de root confirmado, fui directo a buscar el último premio.

THM{MY_W0RD_I5_MY_B0ND_IF_I_ACC3PT_YOUR_CONTRACT_THEN_IT_WILL_BE_COMPL3TED_OR_I'LL_BE_D34D}


<img width="955" height="562" alt="image" src="https://github.com/user-attachments/assets/39784241-5f0e-42fd-9e55-5fce5aeb3042" />

---

## Algunas reflexiones finales

Esta sala es un lindo ejemplo de por qué conviene mirar todo con lupa: un comentario en el código fuente, un archivo con extensión rara, una imagen que "no abre"... cada detalle terminó siendo una pieza del camino. Nada de exploits complicados ni técnicas exóticas, sino metodología prolija: enumerar, observar, anotar, y probar credenciales en cada servicio que vas encontrando.

Si te gustó este recorrido, te recomiendo practicar Base58, esteganografía con `steghide` y jugar un rato con GTFOBins en otras máquinas. Son herramientas que se repiten un montón en salas de nivel principiante e intermedio.

> Gracias por leer hasta acá.

<img width="860" height="384" alt="image" src="https://github.com/user-attachments/assets/9805ec51-6d15-4976-aec5-a3d6c490a3bb" />
