
---
Tags: #Pentesting #WebSecurity #OWASP_Top_10 #AccessControl #IDOR #PrivilegeEscalation #Referrer

---
# Índice

- [[#1. Introducción y Concepto]]
- [[#2. Escalada de Privilegios Vertical (De Usuario a Admin)]]
	- [[#2.1 Funcionalidad administrativa sin protección (Security by Obscurity)]]
	- [[#2.2 Funcionalidad administrativa con URL impredecible (Obfuscation)]]
	- [[#2.3 Manipulación de Roles (Parameter Tampering)]]
	
- [[#3. Escalada Horizontal e IDOR (Insecure Direct Object Reference)]]
	- [[#3.1 ID de usuario predecible]]
	- [[#3.2 ID de usuario impredecible (GUID/UUID) con filtración]]
	- [[#3.3 IDOR en Archivos Estáticos]]
	
- [[#4. Fugas de Información por Fallos de Lógica]]
	- [[#4.1 Filtración en Redirección (302 vs 200)]]
	- [[#4.2 Password Disclosure (Revelación de Contraseña)]]
	
- [[#5. Bypasses de Mecanismos de Control (Platform Misconfiguration)]]
	- [[#5.1 Bypass por Encabezados HTTP (URL Rewriting)]]
	- [[#5.2 Verb Tampering (Bypass por Método HTTP)]]
	- [[#5.3 Referer-based Access Control]]
	
- [[#6. Lógica de Procesos Multipasos]]


---
## 1. Introducción y Concepto

El **Control de Acceso** es la política que aplica restricciones sobre lo que un usuario autenticado (o no autenticado) puede hacer. Las vulnerabilidades ocurren cuando la aplicación no valida correctamente si el usuario que realiza la petición tiene los permisos necesarios para acceder al recurso o realizar la acción.

Existen dos tipos principales de escalada:

- **Escalada Vertical:** Un usuario normal accede a funciones de administrador.
    
- **Escalada Horizontal:** Un usuario accede a los datos de otro usuario con su mismo nivel de privilegios (a menudo relacionado con IDOR).
    

---
## 2. Escalada de Privilegios Vertical (De Usuario a Admin)

### 2.1 Funcionalidad administrativa sin protección (Security by Obscurity)

Ocurre cuando el panel de administración no valida la sesión, sino que confía en que el atacante "no conoce" la URL.

- **Técnica de Descubrimiento:** Fuzzing y Enumeración de archivos/directorios.
    
- **Fuentes de información:** `robots.txt`, `sitemap.xml`, comentarios en HTML.
    

**🛠️ Comando de Fuzzing (FFUF/Gobuster):** Buscar directorios comunes y extensiones sensibles.

Bash

```
ffuf -u https://TARGET.net/FUZZ -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt -e .php,.txt,.html,.bak -t 50
```

**📌 Ejemplo (Robots.txt):** Al revisar `/robots.txt`, encontramos una ruta "prohibida" para los bots, que a menudo es la ruta sensible.

```HTTP
User-agent: *
Disallow: /administrator-panel
```

_Impacto:_ Acceso directo al panel sin autenticación previa.

---
### 2.2 Funcionalidad administrativa con URL impredecible (Obfuscation)

El desarrollador intenta ocultar el panel con nombres aleatorios, pero el código fuente del cliente (JavaScript) revela la ubicación.

**📌 Análisis de Código (JavaScript):** Revisando el código fuente (CTRL+U o Inspector) o archivos `.js` vinculados.

```JavaScript
var isAdmin = false;
if (isAdmin) {
   // ... código ...
   var adminPanelTag = document.createElement('a');
   adminPanelTag.setAttribute('href', '/admin-qcdbfk'); // <--- URL LEAK
   adminPanelTag.innerText = 'Admin panel';
   // ...
}
```

_Explotación:_ Aunque `isAdmin` sea false en nuestro navegador, la URL `/admin-qcdbfk` existe en el servidor. Navegar directamente a ella permite el acceso.

---
### 2.3 Manipulación de Roles (Parameter Tampering)

La aplicación confía en datos controlados por el usuario (Cookies, parámetros GET/POST) para definir el nivel de acceso.

**📌 Caso 1: Cookie de Rol** Interceptar la petición y modificar la cookie que define el privilegio.

```HTTP
Cookie: session=xyz; Admin="false"
```

_Modificación:_ `Admin="true"`

**📌 Caso 2: Mass Assignment (Asignación Masiva) en JSON** Muchos frameworks vinculan automáticamente el JSON de entrada a los objetos internos del código. Si el objeto `User` tiene una propiedad `roleid`, podemos intentar sobrescribirla aunque no esté en el formulario original.

_Petición legítima (Cambio de email):_

```JSON
{
  "email": "wiener@normal.com"
}
```

_Respuesta del servidor (Leak de estructura):_

```JSON
{
  "username": "wiener",
  "email": "wiener@normal.com",
  "roleid": 2
}
```

_Exploit (Inyección de parámetro):_ Añadimos `roleid` en nuestra petición `POST`.

```HTTP
POST /my-account/change-email HTTP/2
...
{
  "email":"hacker@evil.com",
  "roleid": 1  // O "admin", 0, etc.
}
```

---
## 3. Escalada Horizontal e IDOR (Insecure Direct Object Reference)

### 3.1 ID de usuario predecible

Cuando la aplicación utiliza identificadores secuenciales (1, 2, 3...) o nombres de usuario para recuperar datos.

**📌 PoC:**

```http
GET /my-account?id=wiener HTTP/2
```

Cambiar a:

```HTTP
GET /my-account?id=carlos
```

_Nota:_ No siempre requiere estar logueado. Si no hay control de sesión, es un acceso público a datos privados.

### 3.2 ID de usuario impredecible (GUID/UUID) con filtración

Los GUIDs (`bdc29121-ddc8...`) son imposibles de adivinar por fuerza bruta, pero a menudo la aplicación los filtra en otros lugares públicos.

- **Paso 1:** Identificar que el perfil usa GUID: `/my-account?id=PROPIO_GUID`.
    
- **Paso 2:** Buscar fugas de GUID de otros usuarios.
    
    - _Ejemplo:_ En la sección de comentarios o blog, la URL del perfil del autor revela su GUID.
        
    - `GET /blogs?userId=7a63e9d7-dab6-41ec-a82c-cd7b9f887990`
        
- **Paso 3:** Usar ese GUID capturado para suplantar al usuario en el endpoint vulnerable.
    

### 3.3 IDOR en Archivos Estáticos

Acceso directo a archivos generados por el servidor sin validación de propiedad.

**📌 PoC:** El usuario descarga su chat `1.txt`.

```HTTP
GET /download-transcript/1.txt
```

_Exploit:_ Cambiar a `2.txt`, `3.txt`. Fuzzear el número para descargar chats de todos los usuarios.

---
## 4. Fugas de Información por Fallos de Lógica

### 4.1 Filtración en Redirección (302 vs 200)

Un error clásico donde el servidor envía los datos sensibles **antes** de redirigir al usuario al login.

**Escenario:**

1. Atacante pide `/my-account?id=admin`.
    
2. Servidor detecta que no es admin -> Envía código `302 Found` (Redirección a `/login`).
    
3. **ERROR:** El cuerpo (Body) de la respuesta `302` contiene el HTML del panel de administración. El navegador obedece al 302 y no te lo muestra, pero Burp Suite sí lo ve.
    

**🔧 Técnica (Burp Suite Match and Replace):**

- Proxy -> Options -> Match and Replace.
    
- Rule: Match `HTTP/1.1 302 Found` Replace `HTTP/1.1 200 OK`.
    
- Esto obliga al navegador a renderizar el cuerpo de la respuesta en lugar de redirigir.
    

### 4.2 Password Disclosure (Revelación de Contraseña)

Si un endpoint recupera la contraseña actual y la pone en un campo del formulario (incluso si es `type="password"`), es una vulnerabilidad crítica.

- **Técnica:** Inspeccionar elemento -> Cambiar `<input type="password">` a `<input type="text">`.
    
- _Nota:_ Esto indica que la contraseña no está hasheada correctamente en el tránsito o almacenamiento reversible.
    

---

## 5. Bypasses de Mecanismos de Control (Platform Misconfiguration)

### 5.1 Bypass por Encabezados HTTP (URL Rewriting)

Algunos frameworks (como Symfony, ASP.NET o reglas de Load Balancers) permiten sobrescribir la URL destino usando cabeceras no estándar, engañando al control de acceso frontal.

**📌 PoC:** El backend bloquea `/admin/delete`, pero permite `/`.

```HTTP
POST / HTTP/2
X-Original-Url: /admin/delete
...
username=carlos
```

_Variantes:_ `X-Rewrite-Url`, `X-Forwarded-Prefix`.

### 5.2 Verb Tampering (Bypass por Método HTTP)

Las reglas de acceso (ACL) a menudo se configuran solo para métodos específicos (ej. bloquear `POST` para cambios), olvidando otros métodos que el backend podría procesar.

**📌 PoC:**

```HTTP
POST /admin-roles (401 Unauthorized)
```

Prueba cambiando el verbo (Method):

```HTTP
GET /admin-roles?username=wiener&action=upgrade (200 OK)
```

_Variantes:_ Probar `HEAD` (a veces ejecuta la acción sin devolver cuerpo) o métodos extraños como `PUT`.

### 5.3 Control de Acceso basado en Referer (Insecure Referer Validation)
 
Esta vulnerabilidad ocurre cuando la aplicación confía ciegamente en la cabecera HTTP `Referer` para tomar decisiones de autorización. Los desarrolladores asumen erróneamente que si la petición proviene (es "referida") de una página administrativa (ej. `/admin`), entonces el usuario debe ser un administrador autorizado.

**⚠️ El problema conceptual:** El encabezado `Referer` es un dato controlado completamente por el cliente (navegador). No es una prueba de autorización del lado del servidor.

**🕵️‍♂️ Cómo descubrirlo:**

1. Intercepta una petición que deniega el acceso (401/403) o una petición administrativa legítima.
    
2. Envía la petición al **Repeater** de Burp Suite.
    
3. Modifica o añade la cabecera `Referer` apuntando a la URL del panel de administración o la página donde se supone que debería originarse la acción.
    

**📌 PoC (Tu ejemplo):** Tenemos una petición para actualizar el rol de usuario que normalmente requiere ser admin. La aplicación verifica si vienes de la ruta `/admin`.

_Petición de ataque forzada:_

```HTTP
GET /admin-roles?username=wiener&action=upgrade HTTP/2
Host: 0aac005e04acc67881247ad6005a00be.web-security-academy.net
Cookie: session=GTEjsxRYV6p8hzx3Zo2gBIyGbbkA9LpD
Referer: https://0aac005e04acc67881247ad6005a00be.web-security-academy.net/admin
```

_Resultado:_ El servidor ve `Referer: .../admin`, asume confianza y ejecuta la acción `upgrade`.

**🛠️ Tip Avanzado (Automatización):** Si detectas que una aplicación valida el Referer en muchas partes, no necesitas cambiarlo manualmente en cada petición.

1. Ve a **Burp Proxy** > **Match and Replace**.
    
2. Añade una regla para el _Request Header_.
    
3. **Match:** `^Referer:.*$` (Regex para capturar cualquier referer).
    
4. **Replace:** `Referer: https://vulnerable-site.com/admin`.
    
5. Esto automatizará el bypass mientras navegas normalmente.```

---
## 6. Lógica de Procesos Multipasos

Los desarrolladores a veces validan permisos en el paso 1, pero asumen que si llegas al paso 2, ya fuiste validado.

**Escenario:**

1. `POST /admin-roles` (Valida permisos -> Si eres Admin, muestra confirmación).
    
2. `POST /admin-roles/confirm` (Ejecuta la acción).
    

**📌 Exploit:** Saltar directamente al paso 2 (Petición de confirmación) ignorando el paso 1.

```HTTP
POST /admin-roles HTTP/2
Content-Type: application/x-www-form-urlencoded

action=upgrade&confirmed=true&username=wiener
```

---

> [!TIP] **Consejo de Experto: AuthMatrix** Para auditar controles de acceso de forma profesional en Burp Suite, utiliza la extensión **AuthMatrix**.
> 
> 1. Creas usuarios de diferentes roles (Admin, User, Anon).
>     
> 2. Capturas las cookies de cada uno.
>     
> 3. Defines las peticiones objetivo.
>     
> 4. AuthMatrix ejecutará automáticamente cada petición con las cookies de cada usuario y te mostrará con un "Semáforo" (Verde/Rojo) si hay vulnerabilidades de acceso cruzado.
>