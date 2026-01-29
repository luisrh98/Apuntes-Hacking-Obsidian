
---
Tags: #web #pentesting #sqli #sql-truncation #owasp #injection #authentication #longitud #bbdd #login

---
## 📖 Definición

El **SQL Truncation Attack** es una vulnerabilidad que ocurre cuando una aplicación **no valida correctamente la longitud de los datos** que inserta en una base de datos.  
Un atacante puede aprovechar el **truncamiento automático** que hacen los motores SQL cuando los datos son más largos que el campo definido (ejemplo: `VARCHAR(20)`), causando **colisiones de valores** y logrando suplantar identidades o crear usuarios duplicados.

👉 Muy común en **campos de usuario, emails, tokens o IDs**.

---

## ⚙️ Funcionamiento lógico

1. El desarrollador define una columna con un tamaño fijo, ej. `username VARCHAR(20)`.
    
2. La aplicación **no valida la longitud** antes de insertar.
    
3. Si un atacante envía un `username` con más de 20 caracteres, el motor SQL lo **trunca automáticamente**.
    
4. Esto puede provocar que:
    
    - Se cree un usuario con nombre **idéntico** al de otro ya existente.
        
    - Se **bypasseen restricciones de unicidad** (`UNIQUE KEY`).
        
    - Se **haga login como otra persona** si el sistema confía solo en el nombre truncado.
        

---

## 🔍 Métodos de Detección

|Método|Descripción|
|---|---|
|**Fuzzing de longitud**|Enviar cadenas largas (ejemplo: 100 caracteres) en parámetros como _username_, _email_, _ID_.|
|**Errores de BD**|Observar si la aplicación devuelve errores relacionados con longitud (`Data too long`, `String or binary data would be truncated`).|
|**Bypass de unicidad**|Intentar registrar un usuario con un valor largo que empiece con el mismo _username_ existente.|
|**Pruebas manuales**|Revisar campos `VARCHAR`, `CHAR`, `TEXT` en la estructura de la base de datos si se filtra información (ej. errores SQL).|

---

## 💣 Métodos de Explotación

### 🧪 Ejemplo básico

Supongamos que la tabla `users` tiene:

`username VARCHAR(20) UNIQUE, password VARCHAR(50)`

Ya existe un usuario:

`username = admin password = hash_admin`

📌 Ataque:

1. Registrar un nuevo usuario con nombre:
    
    `adminAAAAAAAAAAAAAAAAAAAA`
    
    (25 caracteres, se trunca a `adminAAAAAAAAAAAAAAAA` → **20 chars**).
    
2. En la BD, si no hay validación, el valor se recorta a:
    
    `adminAAAAAAAAAAAAAAAA`
    
    que puede colisionar con `admin` o permitir bypass en autenticación.
    

---

### 🎯 Escenarios de explotación

|Escenario|Ejemplo|
|---|---|
|**Login takeover**|Crear un usuario con nombre truncado igual al de otro usuario privilegiado.|
|**Bypass de validaciones**|El sistema cree que insertó un nombre distinto, pero en BD quedó truncado al mismo valor.|
|**Colisiones en tokens**|Si los tokens se guardan en un campo con límite de longitud, se puede provocar que dos tokens diferentes se guarden como uno.|
|**Subida de archivos**|Si nombres de archivos se almacenan truncados en BD, se puede sobrescribir contenido.|

---

## 🛠️ Herramientas útiles

- **Burp Suite** → Para fuzzing de longitudes en parámetros.
    
- **SQLMap** → Aunque está más orientado a inyección, puede detectar errores de truncamiento en ciertos escenarios.
    
- **PayloadsAllTheThings** → Tiene ejemplos prácticos de truncation.
    

---

## 🛡️ Medidas de Mitigación

|Medida|Descripción|
|---|---|
|**Validar longitudes en el lado servidor**|Rechazar entradas más largas que el campo definido en BD.|
|**Normalizar datos**|Evitar que datos equivalentes se interpreten distinto (ej. trimming, case-folding).|
|**Revisar unicidad**|Asegurarse de que los índices `UNIQUE` se verifiquen tras truncar.|
|**Errores explícitos**|Configurar la BD para lanzar error en vez de truncar automáticamente (ej. `STRICT_TRANS_TABLES` en MySQL).|

---
# 🧪 Ejemplo Práctico – SQL Truncation Attack

## 📂 Escenario de prueba

La aplicación web tiene un sistema de **registro y login** con la siguiente tabla SQL:

`CREATE TABLE users (     id INT AUTO_INCREMENT PRIMARY KEY,     username VARCHAR(20) UNIQUE,     password VARCHAR(100) );`

Ya existe un usuario administrador:

`username = admin password = hash_admin`

---

## 📝 Paso 1 – Registro del atacante

El atacante intenta registrarse con el siguiente `username`:

`adminAAAAAAAAAAAAAAAAAAAA`

👉 Este nombre tiene **25 caracteres**, pero el campo en la BD es `VARCHAR(20)`.

Cuando se inserta, **MySQL trunca automáticamente** la cadena a:

`adminAAAAAAAAAAAAAAAA`

---

## 📝 Paso 2 – Colisión en la base de datos

- La aplicación cree que creó un usuario distinto (`adminAAAAAAAAAAAAAAAAAAAA`).
    
- Pero en la base de datos realmente se almacenó como:
    

`username = adminAAAAAAAAAAAAAAAA password = hash_attacker`

⚠️ Si no hay validación estricta, esto puede entrar en **conflicto con el usuario `admin`**, o permitir **colisiones lógicas** en la autenticación.

---

## 📝 Paso 3 – Intento de login

Si el sistema valida el login con algo como:

`SELECT * FROM users WHERE username = 'adminAAAAAAAAAAAAAAAAAAAA' AND password = 'hash_attacker';`

En la BD se truncará el valor buscado a **20 caracteres**:

`'adminAAAAAAAAAAAAAAAA'`

👉 Resultado: El atacante puede entrar como el usuario truncado (y si coincide con un usuario privilegiado, como `admin`).