
---
Tags: #burpsuite #plugins #turbointruder #smuggling #http

---
# Índice

[[#1. La Interfaz y el Concepto Básico]]
	[[#El "Motor" (RequestEngine)]]
	
[[#2. Desglosando tu script (Pause-based)]]
	[[#Puntos críticos de depuración]]
	
[[#3. Cómo depurar cuando "no funciona"]]
	[[#A. Revisa la tabla de resultados (Abajo)]]
	[[#B. El problema de los Saltos de Línea (` r n`)]]
	[[#C. Calcula bien el Content-Length]]
	
[[#4. Receta para el Éxito con tus Apuntes]]
	[[#Un Truco Pro `x=1` (El "Canary")]]
	
---
### 1. La Interfaz y el Concepto Básico

Al enviar una petición a Turbo Intruder (Click derecho -> Extensions -> Turbo Intruder -> Send to...), verás una ventana dividida:

- **Arriba:** El código Python que controla el ataque.
    
- **Abajo:** La petición HTTP "base" (plantilla).
    

#### El "Motor" (RequestEngine)

La configuración del motor es el 90% del éxito en Smuggling.

```Python
engine = RequestEngine(endpoint=target.endpoint,
                       concurrentConnections=1,
                       requestsPerConnection=100,
                       pipeline=False
                       )
```

- **`concurrentConnections=1`**: **CRUCIAL para Smuggling**.
    
    - En la mayoría de ataques de Smuggling (especialmente CL.TE/TE.CL), quieres envenenar un socket y luego enviar una petición de seguimiento _por ese mismo socket_ para ver el error.
        
    - Si pones `concurrentConnections=50`, Turbo Intruder abrirá 50 "tubos" distintos. Podrías envenenar el tubo 1 y enviar la petición de verificación por el tubo 2, donde no hay veneno. El ataque fallará. **Mantenlo siempre en 1 para empezar.**
        
- **`requestsPerConnection=100`**:
    
    - Define cuántas peticiones se envían antes de cerrar el socket y abrir uno nuevo. En Smuggling queremos conexiones persistentes (Keep-Alive), así que un número alto (100 o 1000) está bien.
        
- **`pipeline=False` vs `True`**:
    
    - **False:** Envía Petición A -> Espera Respuesta A -> Envía Petición B. (Más estable para depurar).
        
    - **True:** Envía Petición A -> Envía Petición B -> Lee respuestas. (Necesario para algunos ataques de temporización o para inundar el servidor rápidamente).
        

---
### 2. Desglosando tu script (Pause-based)

Analicemos el script que tenías en tus apuntes, porque tiene detalles que si no ajustas bien, fallarán.

```Python
def queueRequests(target, wordlists):
    engine = RequestEngine(...)

    # 1. Definición del cuerpo del ataque
    # %s se sustituirá por una cadena, %d por un entero.
    atackerRequest = """POST / HTTP/1.1
Host: 0a4400a7...net
Content-Length: %s

%s"""

    # 2. La petición que queremos colar (Smuggled)
    requestSmuggled = """POST /admin/delete/ HTTP/1.1
Host: localhost
Content-Length: 53
...
"""

    # 3. La petición normal (para "empujar" o verificar)
    normalRequest = """GET / HTTP/1.1
Host: ...
"""

    # 4. LA COLA (Donde ocurre la magia)
    engine.queue(atackerRequest, 
                 [len(requestSmuggled), requestSmuggled], 
                 pauseMarker=['\r\n\r\nPOST'], 
                 pauseTime=61000)
    
    engine.queue(normalRequest)
```

#### Puntos críticos de depuración:

1. **El Formateo de Strings (`%s` vs `%d`):**
    
    - En `atackerRequest`, usas `%s` (string) para el Content-Length.
        
    - En `engine.queue`, pasas `len(requestSmuggled)`, que es un entero (`int`).
        
    - **Error común:** Python lanzará error si intentas meter un objeto complejo donde va un string. Asegúrate de que los tipos coincidan. Turbo Intruder es permisivo, pero es mejor usar `%s` para todo y dejar que Python lo convierta a texto.
        
2. **`pauseMarker` y `pauseTime`:**
    
    - `pauseMarker=['\r\n\r\nPOST']`: Esto le dice a Turbo Intruder: "Empieza a enviar `atackerRequest`. Cuando encuentres la secuencia `\r\n\r\nPOST`, **DETENTE**".
        
    - `pauseTime=61000`: "Espera 61 segundos con el socket abierto pero sin enviar nada más".
        
    - **Depuración:** Si el ataque falla, verifica que la cadena del `pauseMarker` existe **exactamente** en tu petición. Si hay un espacio extra o falta un salto de línea en tu definición de `atackerRequest`, el marcador no se encontrará, la pausa no ocurrirá y el ataque fallará.
        

---
### 3. Cómo depurar cuando "no funciona"

Estás lanzando el ataque y no ves el 404 o el 200 OK del admin. ¿Qué haces?

#### A. Revisa la tabla de resultados (Abajo)

No mires solo el código de estado (200, 403, 500). Mira la columna **"Length"** (Longitud).

- Si envías 10 peticiones y una de ellas tiene una longitud diferente (aunque sea por 5 bytes), **ahí está la desincronización**.
    

#### B. El problema de los Saltos de Línea (`\r\n`)

Este es el fallo número 1.

- En la ventana de código de Python, lo que ves como un "enter" a veces es solo `\n` (Linux style), pero HTTP exige `\r\n` (Windows style).
    
- **Solución:** Usa siempre cadenas explícitas si tienes dudas, o asegúrate de que al copiar/pegar desde tus apuntes no se han perdido los retornos de carro.
    
- Turbo Intruder suele normalizar esto, pero en ataques de Smuggling H2 o CRLF Injection, **tú controlas los bytes**. Si el exploit requiere `\r\n` y envías `\n`, fallará.
    

#### C. Calcula bien el Content-Length

En tu script usas `len(requestSmuggled)`. Esto es perfecto porque Python cuenta los bytes exactos.

- **Error:** Calcularlo a ojo o usar el valor que te dio Repeater. Repeater a veces añade headers ocultos.
    
- **Consejo:** Si el ataque falla, intenta sumar o restar 1 o 2 bytes al Content-Length en tu script (`len(requestSmuggled) + 1`). A veces falta contar el último par de `\r\n` que cierra la petición.
    

---
### 4. Receta para el Éxito con tus Apuntes

Para usar tus apuntes de forma efectiva, sigue este flujo cada vez que abras un laboratorio:

1. **Captura una petición válida** en Burp Proxy y envíala a Turbo Intruder.
    
2. **Borra todo el código Python** que viene por defecto y pega tu plantilla (la que tienes en los apuntes).
    
3. **Ajusta el `Target`:**
    
    - Actualiza el `Host:` en las variables de texto (`atackerRequest`, `smuggledRequest`, etc.) con el host del laboratorio actual.
        
4. **Ajusta el Payload:**
    
    - Si es un ataque **CL.TE**, asegúrate de que el Content-Length del "wrapper" abarca el cuerpo + la petición smuggled.
        
    - Si es **TE.CL**, asegúrate de que el chunk size (`5e` en tus ejemplos) cubre la petición smuggled.
        
5. **Lanza y Filtra:**
    
    - Dale al botón "Attack".
        
    - En la tabla de resultados, usa el filtro de abajo. Escribe `200` para ver si lograste entrar al admin, o `404` para ver si rompiste la petición siguiente.
        

### Un Truco Pro: `x=1` (El "Canary")

Cuando intentes confirmar la vulnerabilidad, añade siempre un parámetro "canario" en tu petición smuggled que se refleje si tienes éxito. Por ejemplo, si intentas provocar un 404, intenta pedir: `GET /noexisto-tucodigoaleatorio HTTP/1.1`

Luego, en la barra de búsqueda de filtros de Turbo Intruder, escribe `tucodigoaleatorio`. Si aparece en alguna respuesta (aunque sea en el cuerpo de un error), ¡has conseguido desincronizar y que el servidor te devuelva tu propia petición inyectada!