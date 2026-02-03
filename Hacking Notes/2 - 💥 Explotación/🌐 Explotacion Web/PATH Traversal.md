---

---

> **Tags:** #WebSecurity #Pentesting #OWASP_Top_10 #LFI #PathTraversal
> 
> **CWE:** CWE-22: Improper Limitation of a Pathname to a Restricted Directory

---
# Índice:

- [[#1. Definición y Concepto]]
	- [[#¿Cómo funciona?]]
	
- [[#2. Análisis de Técnicas y Bypasses]]
	[[#A. El Caso Simple (Basic Traversal)]]
	[[#B. Bypass con Rutas Absolutas]]
	[[#C. Secuencias Eliminadas (Stripped Non-Recursively)]]
	[[#D. Bypass con Doble URL-Encoding]]
	[[#E. Validación de Inicio de Ruta (Path Start Validation)]]
	[[#F. Null Byte Injection (Versiones Legacy)]]
	
- [[#3. Cheatsheet de Payloads y Detección]]
	[[#Archivos de Interés (Targets)]]
	[[#Matriz de Payloads (Técnicas de Evasión)]]
	
- [[#4. Metodología de Detección]]
	
- [[#5. Prevención (Remediación)]]
---
## 1. Definición y Concepto

El **Path Traversal** (o Directory Traversal) es una vulnerabilidad web que permite a un atacante leer archivos arbitrarios en el servidor que ejecuta la aplicación. Esto ocurre cuando la aplicación utiliza input del usuario para construir una ruta de archivo sin la validación o saneamiento adecuados.

El objetivo principal es "escapar" del directorio raíz web (web root) y acceder a archivos críticos del sistema operativo o código fuente de la aplicación.

### ¿Cómo funciona?

Los sistemas de archivos utilizan secuencias especiales para la navegación relativa:

- `.` (punto): Directorio actual.
    
- `..` (punto-punto): Directorio padre (un nivel arriba).
    

Si la aplicación espera `filename=foto.jpg` y la busca en `/var/www/html/images/`, un atacante puede inyectar `../../../../etc/passwd`. El sistema operativo resolverá la ruta subiendo niveles hasta llegar a la raíz `/` y luego bajando a `/etc/passwd`.

---

## 2. Análisis de Técnicas y Bypasses

Aquí desgloso las técnicas que has practicado, explicando la lógica defensiva que falló en cada caso.

### A. El Caso Simple (Basic Traversal)

La aplicación concatena directamente tu input a la ruta base. No hay filtros.

- **Vector:** `../../../../../etc/passwd`
    
- **Lógica:** Subimos suficientes niveles para asegurarnos de llegar a la raíz del sistema (`/`). En Linux, subir más niveles de los existentes no genera error, simplemente te quedas en la raíz.
    

### B. Bypass con Rutas Absolutas

A veces, el desarrollador filtra los puntos `..`, pero olvida validar si el usuario introduce una ruta completa desde la raíz.

- **Vector:** `/etc/passwd`
    
- **Por qué funciona:** La aplicación puede estar haciendo algo como `open("/var/www/images/" + input)`. Sin embargo, en algunos lenguajes/librerías, si el input comienza con `/`, se trata como una ruta absoluta y se ignora la ruta base prefijada.
    

### C. Secuencias Eliminadas (Stripped Non-Recursively)

El filtro intenta limpiar el input eliminando la secuencia `../`.

- **Defensa (Fallida):** `input.replace("../", "")`
    
- **Ataque:** `....//`
    
- **Explicación:** El filtro encuentra el `../` central dentro de `....//` y lo elimina.
    
    - Original: `..` + `../` + `/`
        
    - Eliminación: `..` + `[ELIMINADO]` + `/`
        
    - Resultado final que procesa el OS: `../`
        

### D. Bypass con Doble URL-Encoding

Este es un clásico contra WAFs (Web Application Firewalls) o filtros que decodifican el input una sola vez.

- **Vector:** `..%252f..%252f..%252fetc/passwd`
    
- **Explicación detallada:**
    
    1. Queremos enviar una barra `/`.
        
    2. En URL Encode, `/` es `%2f`.
        
    3. Si el servidor bloquea `%2f`, codificamos el carácter `%`.
        
    4. En URL Encode, `%` es `%25`.
        
    5. Por tanto, `%2f` se convierte en `%252f`.
        
- **Flujo de ejecución:**
    
    1. El servidor web (Frontend/Load Balancer) recibe `%252f`. Decodifica `%25` a `%`. Resultado: `%2f`.
        
    2. Pasa el dato a la aplicación (Backend).
        
    3. La aplicación decodifica `%2f` a `/`.
        
    4. El filtro de seguridad ya pasó, y ahora tenemos un carácter malicioso activo.
        

### E. Validación de Inicio de Ruta (Path Start Validation)

El código verifica que la ruta empiece obligatoriamente por un directorio seguro.

- **Código vulnerable:** `if (path.startsWith("/var/www/images/")) { ... }`
    
- **Ataque:** `/var/www/images/../../../../etc/passwd`
    
- **Por qué funciona:** Cumplimos la condición del programador (la cadena empieza correctamente), pero usamos `..` inmediatamente después para salir de ahí. El sistema operativo resuelve la ruta lógica, anulando el directorio forzado.
    

### F. Null Byte Injection (Versiones Legacy)

Crucial en sistemas antiguos (PHP < 5.3.4, Java antiguos).

- **Defensa:** La app fuerza una extensión al final: `open(input + ".jpg")`
    
- **Ataque:** `../../etc/passwd%00.jpg`
    
- **Explicación Técnica:** En lenguajes de bajo nivel (C/C++), las cadenas de texto terminan con un byte nulo (`\0`).
    
    - La validación de la app ve que termina en `.jpg` (después del nulo) y dice "OK".
        
    - Cuando el sistema operativo (escrito en C) va a leer el archivo, lee hasta el `%00` y se detiene, ignorando el `.jpg`.
        
    - Archivo leído: `../../etc/passwd`.
        

---

## 3. Cheatsheet de Payloads y Detección

Esta tabla es fundamental para tu Obsidian. Úsala como referencia rápida durante un pentest.

### Archivos de Interés (Targets)

|**Sistema Operativo**|**Archivo**|**Descripción**|
|---|---|---|
|**Linux**|`/etc/passwd`|Usuarios del sistema.|
|**Linux**|`/etc/shadow`|Hashes de contraseñas (requiere root).|
|**Linux**|`/etc/hosts`|Mapeo de red interno.|
|**Linux**|`/proc/self/environ`|Variables de entorno (puede contener claves API).|
|**Windows**|`C:\Windows\win.ini`|Archivo de configuración (prueba de concepto clásica).|
|**Windows**|`C:\Windows\System32\drivers\etc\hosts`|Hosts de red.|

### Matriz de Payloads (Técnicas de Evasión)

|**Técnica**|**Payload Ejemplo (Linux)**|**Explicación**|
|---|---|---|
|**Básico**|`../../../../etc/passwd`|Movimiento simple hacia atrás.|
|**Windows (Slash)**|`..\..\..\..\windows\win.ini`|Windows usa backslash `\`.|
|**Windows (Mixto)**|`../../../../windows/win.ini`|Windows a veces acepta forward slash `/`.|
|**Url Encoded**|`%2e%2e%2f%2e%2e%2fetc/passwd`|Codificación estándar de `../`.|
|**Double URL**|`%252e%252e%252fetc/passwd`|Doble codificación para saltar filtros previos.|
|**UTF-8 Unicode**|`..%c0%af..%c0%afetc/passwd`|`%c0%af` es una representación "overlong" de `/`.|
|**Null Byte**|`../../../etc/passwd%00.jpg`|Trunca la extensión forzada.|
|**Filter Bypass**|`....//....//etc/passwd`|Para filtros que eliminan `../` una sola vez.|
|**Scheme Wrapper**|`file:///etc/passwd`|Si el input se pasa a una función como `curl` o `include`.|
|**PHP Wrapper**|`php://filter/convert.base64-encode/resource=index.php`|**Crítico:** Permite leer el código fuente (ej. `config.php`) en base64 sin ejecutarlo.|

---

## 4. Metodología de Detección

¿Cómo encuentro esto en una caja negra (Black Box)?

1. **Mapeo:** Identificar todos los parámetros que parecen hacer referencia a un archivo (`?file=`, `?doc=`, `?image=`, `?lang=es`).
    
2. **Prueba de Fuzzing:** Usar herramientas como Burp Suite Intruder o FFUF con wordlists específicas de Path Traversal (recomiendo las de _SecLists_).
    
3. **Observación de Respuestas:**
    
    - **HTTP 200:** Y contenido del archivo visible -> **VULNERABLE**.
        
    - **HTTP 500/403:** Puede indicar que el archivo existe pero no hay permisos o el formato falló. Intenta variantes.
        
    - **Mensajes de Error:** "File not found at /var/www/..." -> **Information Disclosure** (te revela la ruta absoluta).
        

---

## 5. Prevención (Remediación)

Como profesional, debes saber cómo arreglarlo:

1. **Evitar pasar rutas de usuario al filesystem:** Usar IDs indirectos (Ej: `?id=5` busca en base de datos el nombre real del archivo).
    
2. **Validación estricta (Allowlist):** Solo permitir caracteres alfanuméricos en el nombre del archivo.
    
3. **Normalización de Rutas:**
    
    - En el código, usa funciones como `realpath()` (PHP/Linux) o `Path.GetFullPath()` (C#).
        
    - Estas funciones resuelven los `../` _antes_ de la validación.
        
    - **Lógica segura:**
        
        Python
        
        ```
        base_dir = "/var/www/images/"
        input_path = user_input
        # 1. Resolver ruta completa
        real_path = os.path.realpath(os.path.join(base_dir, input_path))
        # 2. Verificar que la ruta resuelta empieza con el directorio base
        if not real_path.startswith(base_dir):
            raise SecurityException("Intento de Hacking detectado")
        ```