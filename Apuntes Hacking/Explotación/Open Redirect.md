
---
Tags: #web #redirect #botnet 

---
## 📌 Definición

Un **Open Redirect** ocurre cuando una aplicación redirige al usuario hacia una URL proporcionada por el cliente **sin validarla correctamente**.  
Esto permite a un atacante **manipular el destino** de la redirección para enviar a la víctima a un sitio malicioso.

---

## 🧠 Funcionamiento lógico

1. El usuario accede a una URL legítima con un parámetro de destino:
    
    `https://site.com/redirect?url=https://example.com`
    
2. El servidor **no valida** si la URL es segura o pertenece al dominio permitido.
    
3. El atacante modifica la URL para apuntar a un dominio malicioso:
    
    `https://site.com/redirect?url=https://evil.com`
    
4. La víctima confía en el dominio legítimo pero termina en el sitio malicioso.
    

---

## 🔍 Detección

- Buscar parámetros relacionados con redirecciones:
    
    - `url=`, `next=`, `redirect=`, `goto=`, `dest=`, `target=`, `r=`, `continue=`
        
- Probar valores externos:
    
    `?redirect=https://evil.com`
    
- Observar si el servidor:
    
    - Redirige sin validar.
        
    - Permite esquemas `http://`, `https://`, `//`, `\evil.com`.
        

---

## 💥 Ejemplos de explotación

### 1️⃣ Redirección simple

`GET /redirect?url=https://evil.com`

---

### 2️⃣ Uso de protocolo especial

`GET /redirect?url=javascript:alert(1)`

_(en navegadores modernos, la mayoría bloquea `javascript:` pero puede combinarse con otras técnicas)_

---

### 3️⃣ Bypass con doble slash

`GET /redirect?url=//evil.com`

_(algunos frameworks lo interpretan como externo aunque parezca relativo)_

---

### 4️⃣ Bypass con codificación

`GET /redirect?url=https:%2f%2fevil.com`

o usando Unicode:

`GET /redirect?url=https://%65%76%69%6C.com`

---

## 🔗 Combinaciones con otras vulnerabilidades

| Vulnerabilidad combinada | Descripción                                                                        | Ejemplo                                                |
| ------------------------ | ---------------------------------------------------------------------------------- | ------------------------------------------------------ |
| **Phishing**             | El atacante envía un link legítimo que luego redirige a un sitio falso.            | `https://bank.com/login?redirect=https://fakebank.com` |
| **XSS**                  | Redirige hacia un `javascript:` o hacia una página vulnerable a XSS.               | `?url=https://victim.com/vuln?xss=<script>`            |
| **SSRF**                 | Un Open Redirect interno puede usarse para acceder a recursos internos.            | `?url=http://127.0.0.1/admin`                          |
| **OAuth token theft**    | En flujos OAuth, puede robarse el `code`/`token` si la redirección es manipulable. | `?redirect_uri=https://attacker.com/capture`           |
| **CSRF**                 | Redirigir tras acción maliciosa para despistar a la víctima.                       | `?next=https://evil.com`                               |
| **Malware hosting**      | Redirigir a descarga automática de malware.                                        | `?goto=https://evil.com/malware.exe`                   |

---

## 🛠 Herramientas útiles

- **Burp Suite / ZAP** → Reemplazar parámetros `url`, `redirect`, `next` y observar redirecciones.
    
- **Param Miner** → Descubrir parámetros ocultos.
    
- **Open Redirect Scanner** de OWASP.
    

---

## 🎯 Metodología de prueba

1. Localizar parámetros de redirección.
    
2. Probar con dominios externos.
    
3. Intentar bypass con:
    
    - `//evil.com`
        
    - `%2f%2fevil.com`
        
    - `https:evil.com`
        
    - `\evil.com`
        
4. Comprobar si la respuesta es `302 Found`, `301 Moved Permanently`, o redirección HTML/meta.
    
5. Explorar impacto combinando con otras vulnerabilidades.
    

---

## 🔓 Bypass comunes

- Usar **URLs relativas con prefijos** (`//`, `/\/evil.com`)
    
- Codificación **URL-encoded**, **double encoding**, **UTF-8**
    
- Inyección de **CRLF** en Location header para manipular respuesta
    
- Usar **subdominios engañosos**:
    
    `https://site.com.evil.com`
    

---

## 🛡 Mitigaciones

- Validar que el dominio de destino pertenezca a una lista blanca.
    
- Usar **IDs internos** para destinos en vez de URLs completas.
    
- No confiar en parámetros del cliente para decidir el destino.
    
- Escapar y filtrar valores antes de usarlos en redirecciones.
    

---

## ⚠️ Impacto

- Phishing avanzado sin alertas del navegador.
    
- Robo de credenciales y tokens.
    
- SSRF interno.
    
- Descarga y ejecución de malware.
    
- Escalada de otras vulnerabilidades.