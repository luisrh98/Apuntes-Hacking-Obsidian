---

---
---
Tags: #web #cookie #burpsuite #bit #cifrado #padding #oracle #PaddingOracle #padbuster

---
## 🔎 ¿Qué es un ataque Padding Oracle?

Un **[[Padding Oracle Attack]]** es un tipo de ataque criptográfico que se aprovecha de **respuestas diferenciadas** al procesar datos cifrados con **criptografía simétrica por bloques (CBC)** y con **relleno (padding)** incorrecto.

> 📌 **Padding**: técnica para rellenar el último bloque cuando el tamaño del texto no es múltiplo del tamaño del bloque.

---

## 🧬 ¿Cómo funciona el ataque?

1. Se intercepta un **mensaje cifrado (token, cookie...)**.
    
2. El atacante modifica el mensaje y lo reenvía al servidor.
    
3. El servidor **intenta descifrar** el mensaje modificado:
    
    - Si el **padding es válido**, responde con éxito o error distinto.
        
    - Si el **padding es inválido**, devuelve un error de padding específico.
        
4. El atacante usa la respuesta como **oráculo (oracle)** para deducir el contenido original o incluso **descifrar y cifrar mensajes arbitrarios**.
    

---

## 📖 Ejemplo simplificado

Supón que el servidor descifra una cookie cifrada y:

- Devuelve `403 Invalid padding` si el padding no cuadra.
    
- Devuelve `200 OK` si sí cuadra.
    

🔁 El atacante modifica la cookie y prueba millones de variantes hasta obtener la respuesta esperada.

---

## 🛠️ Detección de la vulnerabilidad

|Método|Descripción|
|---|---|
|Cambiar 1 byte del cifrado|Si el padding está mal, debe cambiar la respuesta (ej: 403 vs 500).|
|Respuesta con código específico|403 o mensaje como "invalid padding", "MAC mismatch", etc.|
|Tiempo de respuesta diferente|Si el servidor tarda más cuando el padding es válido.|
|Burp Suite Plugin|Burp detecta diferencias en respuestas automáticamente.|

---

## 🧪 Vectores comunes

- **JWT cifrados (JWE)**.
    
- **Cookies cifradas**.
    
- **Tokens de restablecimiento de contraseña**.
    
- **URLs firmadas**.
    
- Aplicaciones que usan **AES-CBC** con padding PKCS#7 o similares.
    

---

## 📋 Herramientas más usadas

### 🔹 PadBuster

Permite automatizar el ataque completamente.
```bash
padbuster <URL> <EncryptedValue> <BlockSize> [Options]
```

|Parámetro|Descripción|
|---|---|
|`URL`|URL vulnerable con marcador `§` para insertar el payload|
|`EncryptedValue`|Token cifrado (base64/hex)|
|`BlockSize`|Tamaño del bloque (16 para AES)|
|`--encoding`|Forzar codificación (`base64`, `hex`, `raw`)|
|`--auth`|Autenticación si es necesario (usuario:pass o cookie)|

#### Ejemplo:
```bash
padbuster http://target.com/page.php?data=abcd1234 abcd1234 16
```

---

### 🔹 Burp Suite

- Usa **Burp Active Scanner** para detectar padding oracle.
    
- Puedes usar extensiones como **Padding Oracle Hunter**.
    
- Detecta automáticamente respuestas distintas según el padding.
    

---

## 🔐 Funcionamiento técnico (CBC + padding)

Imagina dos bloques cifrados: C1 y C2. El servidor:

1. Descifra `C2` usando la clave secreta ➡ `P2 XOR C1`.
    
2. Aplica el padding ➡ si no es válido, error.
    

El atacante **controla C1** y puede ajustar sus bytes para conseguir que el resultado tenga padding válido (por ejemplo: `0x01`, `0x02 0x02`, etc).

---

## 🧠 Fases del ataque

1. Se manipula el bloque anterior al que se quiere descifrar.
    
2. Se prueba cada posible valor hasta encontrar uno que **dé padding válido**.
    
3. Se obtiene **byte por byte** el texto plano.
    
4. (Opcional) Se puede generar un nuevo bloque válido (re-encriptar).
    

---

## 🎯 Objetivos del atacante

|Objetivo|Descripción|
|---|---|
|Descifrar datos|Extraer contenido de cookies, JWT, tokens...|
|Cifrar contenido controlado|Crear nuevos tokens con datos falsificados|
|Forjar autenticación|Cambiar el ID de usuario o roles|
|RCE (en casos muy raros)|Si el contenido descifrado se evalúa como código|

---

## 🧪 Ejemplo visual de prueba manual


`# Original: GET /app.php?data=abcd1234`
`# Pruebo con 1 byte cambiado: GET /app.php?data=abcf1234`
`# Cambia el código de estado o el mensaje: posible Padding Oracle`

---

## 📘 Consejos para pruebas

- Cambia el último byte y observa cambios en respuesta.
    
- Usa `ff`, `00`, `01` como valores típicos de prueba.
    
- Guarda múltiples respuestas y haz **diffs**.
    
- Usa proxy para automatizar la prueba (Burp, Zap...).
    

---

## 🧱 Medidas de mitigación

|Técnica|Descripción|
|---|---|
|Cifrado autenticado|Usa **AES-GCM** o **AES-CCM**, no CBC con padding|
|No revelar errores detallados|Muestra mensajes genéricos ("Error de autenticación")|
|Verificación de MAC antes del descifrado|Primero comprobar HMAC antes de descifrar con CBC|
|Zero padding + longitud fija|Relleno fijo para evitar detección|
|Uso de tokens firmados|JWT firmados en vez de cifrados, cuando no es necesario ocultar|

---
# Referencias

- PortSwigger: [Enlace](https://portswigger.net/bappstore/0efabfee59404068a8c4071fa18a2e00)