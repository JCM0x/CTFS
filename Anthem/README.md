# Anthem — CTF Writeup

> **Plataforma:** TryHackMe  
> **Dificultad:** Fácil  
> **SO:** Windows  
> **Máquina:** Anthem

## Resumen

Anthem es una máquina de TryHackMe centrada principalmente en la enumeración. La ruta de compromiso comienza con un servidor web donde aparecen varias pistas repartidas entre `robots.txt`, el código fuente y el CMS Umbraco.

A partir de esa información fue posible identificar al usuario administrador y obtener acceso al panel de Umbraco. Después, las credenciales permitieron conectarse por RDP y acceder al sistema Windows.

La escalada de privilegios se consiguió mediante una mala configuración de permisos NTFS sobre un archivo de backup, lo que permitió modificar la ACL, leer `restore.txt` y recuperar las credenciales de Administrator.

La cadena completa fue:

```text
Reconocimiento
    ↓
HTTP + RDP
    ↓
robots.txt
    ↓
Umbraco + información expuesta
    ↓
Identificación del administrador
    ↓
Credenciales
    ↓
Flags en código fuente / metadatos
    ↓
RDP
    ↓
user.txt
    ↓
backup/restore.txt
    ↓
Enumeración de ACL
    ↓
Modificación de permisos
    ↓
Credenciales de Administrator
    ↓
root.txt
```

---

## Tabla de contenidos

1. [Reconocimiento](#1-reconocimiento)
2. [Enumeración del sitio web](#2-enumeración-del-sitio-web)
3. [Identificación de Umbraco](#3-identificación-de-umbraco)
4. [Identificación del administrador](#4-identificación-del-administrador)
5. [Obtención del correo](#5-obtención-del-correo)
6. [Acceso al panel de Umbraco](#6-acceso-al-panel-de-umbraco)
7. [Búsqueda de las flags](#7-búsqueda-de-las-flags)
8. [Acceso por RDP](#8-acceso-por-rdp)
9. [Obtención de user.txt](#9-obtención-de-usertxt)
10. [Descubrimiento de backup/restore.txt](#10-descubrimiento-de-backuprestoretxt)
11. [Análisis de permisos con icacls](#11-análisis-de-permisos-con-icacls)
12. [Obtención de las credenciales de Administrator](#12-obtención-de-las-credenciales-de-administrator)
13. [Acceso como Administrator](#13-acceso-como-administrator)
14. [Resumen de la cadena](#14-resumen-de-la-cadena)
15. [Lo que aprendí](#15-lo-que-aprendí)
16. [Herramientas utilizadas](#16-herramientas-utilizadas)

---

## 1. Reconocimiento

Como siempre, el primer paso fue identificar qué servicios estaban expuestos.

Utilicé Nmap con detección de servicios y scripts por defecto:

```bash
nmap -sC -sV <IP_OBJETIVO>
```

Entre los resultados aparecieron dos servicios especialmente importantes:

```text
80/tcp    open    http
3389/tcp  open    ms-wbt-server
```

El puerto **80** correspondía al servidor web y el **3389** al servicio **RDP (Remote Desktop Protocol)**.

En este punto todavía no tenía credenciales, así que el servidor web era el punto lógico para comenzar la enumeración.

---

## 2. Enumeración del sitio web

Al visitar el puerto 80 encontré el sitio web de Anthem.

La página principal no mostraba toda la información necesaria, así que empecé a revisar elementos que normalmente pueden revelar rutas o contenido no enlazado directamente.

Uno de los primeros archivos que revisé fue:

```text
/robots.txt
```

El archivo proporcionó información importante sobre el sitio y reveló una cadena que tenía apariencia de contraseña:

```text
UmbracoIsTheBest!
```

Además, esta pista ayudaba a identificar el CMS utilizado por la aplicación.

---

## 3. Identificación de Umbraco

La aplicación utilizaba **Umbraco**, un CMS basado en .NET.

Durante la enumeración encontré la ruta:

```text
/umbraco
```

Al acceder aparecía el panel de inicio de sesión.

Todavía faltaba descubrir qué usuario debía utilizar, así que continué revisando el contenido público del sitio.

---

## 4. Identificación del administrador

En una de las páginas encontré un poema que servía como pista.

La investigación del contenido permitió identificarlo como:

```text
Solomon Grundy
```

El nombre resultó ser importante porque correspondía al administrador de la aplicación.

La idea de esta parte de la máquina era conectar información aparentemente independiente: el contenido de la página terminaba proporcionando un dato útil para identificar al usuario.

---

## 5. Obtención del correo

Después de identificar el nombre del administrador, revisé el patrón utilizado para las direcciones de correo del sitio.

El formato utilizaba las iniciales del nombre y apellido.

Para:

```text
Solomon Grundy
```

las iniciales son:

```text
SG
```

Siguiendo el patrón encontrado, el correo era:

```text
SG@anthem.com
```

Con esto ya tenía dos piezas importantes para intentar acceder al panel de Umbraco:

- Usuario: `SG@anthem.com`
- Contraseña: `UmbracoIsTheBest!`

---

## 6. Acceso al panel de Umbraco

Con la información obtenida anteriormente pude probar las credenciales en:

```text
http://<IP_OBJETIVO>/umbraco
```

Las credenciales funcionaron y conseguí entrar al panel de administración de Umbraco.

Esta parte de Anthem deja bastante clara una idea importante: no siempre es necesario buscar una vulnerabilidad técnica complicada. A veces la información expuesta públicamente permite construir las credenciales necesarias.

---

## 7. Búsqueda de las flags

Una de las tareas indicaba que las flags estaban escondidas en el código fuente.

El formato esperado era:

```text
THM{...}
```

Para evitar revisar manualmente cada página, descargué el contenido del sitio de forma recursiva:

```bash
wget --recursive http://<IP_OBJETIVO>
```

Después busqué directamente las cadenas con formato de flag:

```bash
grep -RhoE 'THM\{[^}]+\}' .
```

### ¿Qué hace este comando?

- `-R` → busca recursivamente dentro de directorios.
- `-h` → no muestra el nombre del archivo.
- `-o` → muestra solamente la coincidencia encontrada.
- `-E` → permite utilizar expresiones regulares extendidas.
- `THM\{[^}]+\}` → busca cadenas que comiencen con `THM{`, continúen con uno o más caracteres distintos de `}` y terminen con `}`.
- `.` → indica que la búsqueda comienza en el directorio actual.

También revisé directamente el contenido desde Umbraco.

Una de las flags estaba escondida dentro de los metadatos de una página:

```text
Umbraco
   ↓
Content
   ↓
Página
   ↓
Meta Tags
```

Otra pista podía encontrarse dentro del HTML asociado al buscador.

La lección de esta parte fue sencilla: cuando una máquina indica que una información está en el código fuente, hay que revisar también comentarios, metadatos y elementos que no aparecen visualmente en la página.

---

## 8. Acceso por RDP

Después de terminar la enumeración web ya tenía las credenciales necesarias para utilizar el servicio RDP que había aparecido durante el reconocimiento.

También podía utilizar Remmina de forma gráfica, pero desde la terminal podía conectarme con:

```bash
rdesktop -u SG <IP_OBJETIVO>
```

El parámetro `-u SG` especifica el usuario de Windows.

Al conectarme mediante RDP obtuve acceso al escritorio del sistema.

---

## 9. Obtención de user.txt

Una vez dentro del sistema, revisé el escritorio del usuario.

Allí se encontraba:

```text
user.txt
```

Al abrir el archivo obtuve la flag correspondiente al usuario.

Con esto quedaba completada la fase de acceso inicial.

---

## 10. Descubrimiento de backup/restore.txt

El siguiente objetivo era encontrar la contraseña del administrador.

La pista indicaba que debía revisar archivos ocultos, así que habilité la visualización de elementos ocultos desde Windows:

```text
View → Hidden items
```

Después de revisar el disco `C:` encontré una carpeta llamada:

```text
backup
```

Dentro de ella estaba:

```text
restore.txt
```

El problema era que el usuario `SG` no tenía permiso para leer directamente el archivo.

Esto hizo que la siguiente fase consistiera en revisar los permisos del archivo.

---

## 11. Análisis de permisos con icacls

Desde CMD utilicé:

```cmd
icacls restore.txt
```

`icacls` permite consultar y modificar las listas de control de acceso (ACL) de archivos y carpetas en Windows.

El resultado era interesante: aunque `SG` no tenía permiso de lectura sobre el archivo, sí tenía la posibilidad de modificar la seguridad del mismo.

Por lo tanto, no era necesario buscar otra vulnerabilidad. La configuración de permisos existente proporcionaba el camino para recuperar el contenido.

---

## 12. Obtención de las credenciales de Administrator

Con la posibilidad de modificar la ACL, concedí al usuario `SG` permiso de lectura:

```cmd
icacls restore.txt /grant SG:R
```

Después comprobé nuevamente los permisos:

```cmd
icacls restore.txt
```

Ahora el usuario podía leer el archivo.

Finalmente utilicé:

```cmd
type restore.txt
```

El contenido de `restore.txt` reveló la contraseña de **Administrator**.

Esta fue la parte que más me llamó la atención de la máquina, porque la escalada no dependió de explotar un software vulnerable. El problema estaba en una mala configuración de permisos NTFS.

---

## 13. Acceso como Administrator

Con la contraseña recuperada pude acceder al perfil de Administrator.

La ruta relevante era:

```text
C:\Users\Administrator
```

Después fui al escritorio:

```text
C:\Users\Administrator\Desktop
```

Allí estaba:

```text
root.txt
```

Al abrirlo obtuve la última flag de la máquina.

---

## 14. Resumen de la cadena

La ruta completa de Anthem quedó así:

```text
Nmap
  ↓
80/tcp + 3389/tcp
  ↓
Enumeración web
  ↓
robots.txt
  ↓
Umbraco + información expuesta
  ↓
Solomon Grundy
  ↓
Patrón de correo
  ↓
SG@anthem.com
  ↓
Acceso a Umbraco
  ↓
Flags en HTML / Meta Tags
  ↓
RDP
  ↓
SG
  ↓
user.txt
  ↓
Elementos ocultos
  ↓
C:\backup\restore.txt
  ↓
Enumeración de ACL
  ↓
Modificar permisos
  ↓
Leer restore.txt
  ↓
Credenciales de Administrator
  ↓
Administrator
  ↓
root.txt
```

---

## 15. Lo que aprendí

Anthem me dejó principalmente una lección sobre la importancia de la **enumeración**.

No fue necesario comenzar intentando explotar una vulnerabilidad complicada. La información estaba repartida entre diferentes partes del sitio y del sistema:

- Reconocimiento con Nmap.
- Enumeración de aplicaciones web.
- Análisis de `robots.txt`.
- Identificación de un CMS.
- Revisión del código fuente.
- Búsqueda de información en metadatos.
- Uso de expresiones regulares con `grep`.
- Descarga recursiva con `wget`.
- Acceso remoto mediante RDP.
- Enumeración básica de Windows.
- Identificación de archivos ocultos.
- Análisis de permisos NTFS.
- Uso de `icacls`.
- Escalada de privilegios mediante una mala configuración de permisos.

La parte de `restore.txt` fue especialmente interesante. Al principio parecía que el archivo estaba fuera de nuestro alcance porque `SG` no tenía permisos de lectura. Sin embargo, al revisar las ACL apareció la posibilidad de modificar la seguridad del archivo.

En general, Anthem refuerza una metodología que quiero seguir aplicando en otras máquinas: **enumerar primero, conectar las pistas y explotar solamente cuando la evidencia indique por dónde continuar**.

---

## 16. Herramientas utilizadas

- Nmap
- wget
- grep
- RDP / rdesktop
- Remmina
- CMD
- icacls

> **Nota:** Esta documentación corresponde a un entorno de laboratorio autorizado de TryHackMe y tiene fines educativos.
