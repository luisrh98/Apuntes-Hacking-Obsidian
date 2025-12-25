
---
Tags: #csrf #web #burpsuite #get #post #formularios #form #redireccion

---
# Definición

>[[CSRF - Cross-Site Request Forgery]] es un ataque que fuerza a un usuario autenticado a ejecutar acciones no deseadas en una aplicación web donde está autenticado.

**CSRF** ocurre cuando un atacante induce a un usuario autenticado a enviar una petición maliciosa a un servidor en el que ya tiene una sesión válida, sin su conocimiento.

📌 **Ejemplo básico**: Si estás logueado en un sitio como `banco.com`, y visitas una página maliciosa que ejecuta una petición sin tu consentimiento, el servidor la aceptará **porque tus cookies de sesión son válidas**.

---

### 🧪 ¿Cómo detectar CSRF?

|Método|Descripción|
|---|---|
|🔍 Revisión de formularios|Ver si los formularios tienen un **token anti-CSRF** (por ejemplo, un campo oculto aleatorio).|
|🧪 Uso de Burp Suite|Repetir una petición desde otro sitio, o eliminar el token y verificar si se acepta.|
|🔁 Fuzzing|Automatizar peticiones POST/GET sin el token y observar si la acción ocurre igual.|
|🔓 Inspección del código JavaScript|Revisar si hay validación basada en cookies o referrer pero **sin token CSRF**.|

---

### 🧬 Condiciones para que funcione CSRF

- ✅ El usuario debe estar autenticado.
    
- ✅ El atacante debe saber o adivinar el endpoint y sus parámetros.
    
- ❌ No debe haber validación de origen (`Referer`, `Origin`) o **token anti-CSRF**.
    

---

### 🛠️ Técnicas comunes de explotación

#### 1️⃣ Formulario HTML oculto
```html
<form action="https://victima.com/cambiar_email" method="POST">   <input type="hidden" name="email" value="atacante@maligno.com">   <input type="submit" value="Enviar"> </form>  <script>document.forms[0].submit();</script>
```

💥 El navegador enviará la cookie de sesión del usuario a `victima.com`.

---

#### 2️⃣ Imagen (GET request)
```html
<img src="https://victima.com/delete_user?id=1">
```

🎯 Útil para ataques en endpoints GET sin validación.

---

#### 3️⃣ Fetch / JS moderno
```js
fetch("https://victima.com/cambiar_pass", {   method: "POST",   body: "password=123456",   credentials: "include" // 🔑 necesario para que se envíen cookies });
```

> [!note] Algunos navegadores modernos bloquean esto por CORS si no se permiten orígenes cruzados.

---

### 🎯 Vectores de ataque

|Vector|Descripción|
|---|---|
|📝 Formularios|Se usan en HTML para enviar solicitudes POST.|
|📸 Etiquetas `<img>` o `<script>`|Envían solicitudes GET automáticamente.|
|🔗 `<a href>`|Forzar clics o redirecciones maliciosas.|
|🎯 Redirecciones HTTP o Meta Refresh|Usadas para ataques CSRF basados en navegación forzada.|
|📥 JSON, XML o APIs|Si la API no valida tokens ni `Origin`, puede ser vulnerable.|

---

### 🧱 Parámetros comunes en endpoints vulnerables

|Parámetro|Acción|
|---|---|
|`email`, `new_email`|Cambiar email del usuario|
|`password`, `new_password`|Cambiar contraseña|
|`id`, `delete`, `action=delete`|Eliminar recursos|
|`admin=true`, `role=admin`|Escalar privilegios|

---

### 🧰 Herramientas útiles

|Herramienta|Uso principal|
|---|---|
|**Burp Suite**|Detectar, interceptar y automatizar pruebas CSRF|
|**OWASP ZAP**|Detección automática de CSRF|
|**NoCSRF**|Generar PoC de ataques CSRF|
|**Postman**|Repetición de peticiones sin tokens|

---

### 🛡️ Prevención

|Método|Descripción|
|---|---|
|✅ **Tokens anti-CSRF**|Campos aleatorios que deben enviarse con cada petición.|
|✅ Validar `Origin` o `Referer`|Asegurarse que la petición proviene del dominio propio.|
|✅ Cabeceras `SameSite=Strict` o `Lax` en cookies|Evitan el envío de cookies desde sitios externos.|
|❌ Evitar GETs para operaciones sensibles|Las acciones críticas deben hacerse por POST.|

---

### 🧪 Cómo probarlo en entornos reales

1. Localiza un formulario sensible (cambio de email, contraseña, etc.).
    
2. Captura la petición con Burp.
    
3. Elimina el token CSRF si hay.
    
4. Repite la petición sin token o desde otro origen.
    
5. Observa si se realiza la acción igual.

---
# Referencias

- Enlace a Portswigger: [Enlace](https://portswigger.net/web-security/csrf)
