
---
Tags: #infodisclosure #reconnaissance #fuzzing #vulnerability-research #bugbounty #information-leakage #git-exploitation #http-methods #bypass-techniques  #data-exposure

---

La divulgación de información ocurre cuando una aplicación revela involuntariamente datos sensibles a los usuarios. Esto no es una vulnerabilidad de explotación directa por sí misma, sino el **combustible** necesario para diseñar ataques complejos (RCE, SQLi, Auth Bypass).

### 📑 Índice

- [[#1. Divulgación en Mensajes de Error]]
    [[#🔍 Metodología de Descubrimiento]]
    [[#🛠️ PoC]]
    
- [[#2. Exposición en Páginas de Depuración y Diagnóstico]]
    [[#🛠️ Herramientas de Enumeración]]
    [[#📂 Caso de estudio `phpinfo.php`]]
	
- [[#3. Fuzzing y Directorios Sensibles por Lenguaje]]
    
- [[#4. Análisis de Ficheros de Backup y Robots.txt]]
    [[#🛠️ PoC (Análisis de Ruta)]]
	
- [[#5. Bypass de Autenticación vía HTTP TRACE]]
    [[#🔍 Descubrimiento y Explotación]]
    [[#🛠️ PoC Descubrimiento de Cabeceras Ocultas]]
    
- [[#6. Cabeceras de Control de Acceso (Custom Headers)]]
    [[#🛠️ PoC Acceso Administrativo (Bypass)]]
    
- [[#7. Exfiltración de Secretos en Historial de Git]]
    [[#🔍 Metodología de Explotación]]
    [[#🛠️ Comandos Críticos de Análisis]]
    
- [[#8. Tabla de Extensiones de Archivos Críticos]]
	

---

## ## 1. Divulgación en Mensajes de Error

Esta técnica consiste en **forzar estados no controlados** en la aplicación para que el framework o el servidor web devuelvan "Verbose Errors" (errores detallados).

### 🔍 Metodología de Descubrimiento

Se busca romper la lógica de los parámetros. Si el servidor no tiene una gestión de excepciones adecuada, revelará rutas internas, versiones de librerías o estructuras de consultas.

**Tácticas comunes:**

- **Input Inválido:** Cambiar tipos de datos (de número a texto).
    
- **Inyección de Caracteres Especiales:** Introducir `'`, `"`, `;`, `[]`, o `%00` (null byte).
    
- **Modificación de Parámetros:** Eliminar parámetros obligatorios de la URL.
    

### 🛠️ PoC

Si tenemos un endpoint que espera un ID de producto:

`GET /product?productId=1`

Al alterar el valor a un tipo de dato no esperado o inexistente:

`GET /product?productId=noexisto`

**Resultado del Error (Ejemplo):**

> `Internal Server Error: Unexpected value 'noexisto' in /var/www/html/app/models/ProductModel.php on line 42. Using Apache/2.4.41 (Ubuntu).`

**Información obtenida:**

1. **Ruta absoluta:** `/var/www/html/app/models/` (Útil para ataques de LFI).
    
2. **Tecnología:** Apache 2.4.41 (Permite buscar CVEs específicos).
    
3. **Lógica interna:** Sabemos que usa un modelo llamado `ProductModel.php`.
    

---

## ## 2. Exposición en Páginas de Depuración y Diagnóstico

Muchos desarrolladores olvidan desactivar interfaces de diagnóstico en producción. Estas páginas están diseñadas para dar una visión total del entorno del servidor.

### 🛠️ Herramientas de Enumeración

Para encontrar estas rutas, utilizamos herramientas de **Fuzzing/Brute Force**:

- **Burp Suite (Target Tab):** Excelente para mapeo pasivo mientras navegas.
    
- **FFUF / Gobuster:** Para ataques activos de diccionario.
    

### 📂 Caso de estudio: `phpinfo.php`

En tu auditoría, identificaste: `/cgi-bin/phpinfo.php`.

Esta página es una mina de oro que expone:

- **Variables de Entorno (`ENV`):** Pueden contener `AWS_SECRET_KEY`, `DB_PASSWORD`, etc.
    
- **Configuración de PHP:** Si `allow_url_include` está en `On`, facilita un RCE.
    
- **Módulos cargados:** Versiones de ImageMagick o librerías vulnerables.
    

---

## ## 3. Fuzzing y Directorios Sensibles por Lenguaje

| **Lenguaje / Framework** | **Archivo / Directorio Común**                        | **Información Expuesta**                                |
| ------------------------ | ----------------------------------------------------- | ------------------------------------------------------- |
| **PHP**                  | `/phpinfo.php`, `config.php.bak`, `/.composer/`       | Credenciales DB, Versiones, Keys.                       |
| **Java / Spring**        | `/actuator`, `/heapdump`, `/logview`                  | Tokens de sesión, variables de entorno.                 |
| **Python / Django**      | `/.env`, `/__pycache__/`, `settings.py.bak`           | Secret Keys, Debug Mode logs.                           |
| **Node.js / JS**         | `/package.json`, `/.npmrc`, `/bundle.js.map`          | Dependencias, scripts internos, código fuente original. |
| **ASP.NET**              | `/web.config`, `Trace.axd`, `/.vs/`                   | Connection strings, MachineKeys.                        |
| **General**              | `/.git/`, `/.env`, `/backup.sql`, `/.aws/credentials` | **CRÍTICO:** Repositorios, claves de nube.              |

---

## ## 4. Análisis de Ficheros de Backup y Robots.txt

El archivo `robots.txt` es un estándar para buscadores, pero los atacantes lo usamos como una "guía de áreas prohibidas".

### 🛠️ PoC (Análisis de Ruta)

1. Acceso a `https://target.com/robots.txt`.
    
2. Identificación de entrada: `Disallow: /backup`.
    
3. Al navegar a `/backup`, se lista un directorio con `db_export_2023.sql.gz` o `config.old`.
    

**Nota de Seguridad:** El uso de extensiones de backup comunes como `.bak`, `.old`, `.swp` (de vim) o `~` (de gedit) sobre archivos sensibles permite descargar el código fuente en texto plano, ya que el servidor web no los interpreta como scripts ejecutables.

---

## ## 5. Bypass de Autenticación vía HTTP TRACE

El método **TRACE** está diseñado para fines de diagnóstico (Echo service). El servidor devuelve al cliente exactamente lo que recibió, incluyendo cabeceras añadidas por proxies o balanceadores de carga que el usuario normalmente no ve.

### 🔍 Descubrimiento y Explotación

Si un atacante puede realizar una petición `TRACE`, puede descubrir cabeceras internas que el servidor utiliza para identificar si una petición proviene de una IP de confianza (como la red interna o el localhost).

### 🛠️ PoC: Descubrimiento de Cabeceras Ocultas

En tu ejemplo, enviamos una petición `TRACE` para ver qué añade el servidor por debajo:

```HTTP
TRACE /my-account?id=wiener HTTP/2
Host: [TARGET-ID].web-security-academy.net
Cookie: session=[TU_SESSION]
```

**Respuesta del Servidor (Revelación):**

El servidor responde con un `200 OK` y el cuerpo del mensaje contiene la petición que el servidor procesó _después_ de pasar por su infraestructura interna:

```HTTP
...
X-Custom-IP-Authorization: 86.127.227.131
```

> **Dato Clave:** Acabamos de descubrir que el sistema utiliza `X-Custom-IP-Authorization` para validar la IP del cliente.

---

## ## 6. Cabeceras de Control de Acceso (Custom Headers)

Como profesional, debes saber que muchas aplicaciones confían ciegamente en ciertas cabeceras para "autenticar" que una petición es local. Si forzamos estas cabeceras (Header Injection / Smuggling), podemos saltarnos paneles de login.

### 🛠️ PoC: Acceso Administrativo (Bypass)

Usando la cabecera descubierta anteriormente, suplantamos nuestra identidad como si estuviéramos en la propia máquina del servidor (`127.0.0.1`):

```HTTP
GET /admin/delete?username=carlos HTTP/2
Host: [TARGET-ID].web-security-academy.net
X-Custom-Ip-Authorization: 127.0.0.1
```

**Resultado:** El servidor interpreta que la orden viene desde la consola local y ejecuta el borrado del usuario sin pedir login de administrador.

---

## ## 7. Exfiltración de Secretos en Historial de Git

El directorio `/.git` contiene todo el historial de cambios de un proyecto. Si es accesible, un atacante puede reconstruir el código fuente completo, incluso si los archivos actuales están protegidos.

### 🔍 Metodología de Explotación

1. **Detección:** Fuzzing para encontrar `/.git`.
    
2. **Descarga:** Usamos `wget -r` para descargar el repositorio de forma recursiva.
    
3. **Análisis:** Navegamos por el historial de commits.
    

### 🛠️ Comandos Críticos de Análisis

Una vez descargado el directorio, usamos la herramienta nativa `git`:

- `git log`: Muestra el historial de cambios. Buscamos mensajes como "removed config", "fixed hardcoded pass" o "security update".
    
- `git show [COMMIT_ID]`: Nos permite ver qué se borró exactamente en ese commit.
    
- `git diff [COMMIT_A] [COMMIT_B]`: Compara versiones para encontrar credenciales que fueron eliminadas en parches posteriores pero que siguen siendo válidas.
    

**Ejemplo de hallazgo:**

> `commit a1b2c3d4... "Remove admin API key from source"`
> 
> Al hacer `git show a1b2c3d4`, veríamos:
> 
> `- define('ADMIN_KEY', '5f39c1...[CLAVE_BORRADA]');`

---

## ## 8. Tabla de Extensiones de Archivos Críticos

Para tu Obsidian, aquí tienes una tabla para identificar rápidamente archivos que exponen datos según su extensión:

| **Extensión**    | **Tipo de Información**                               | **Riesgo** |
| ---------------- | ----------------------------------------------------- | ---------- |
| `.env` / `.conf` | Credenciales de DB, APIs, Tokens.                     | Crítico    |
| `.log`           | Rutas internas, errores, IDs de sesión.               | Medio/Alto |
| `.sql` / `.dump` | Estructura de tablas y datos de usuarios.             | Crítico    |
| `.git` / `.svn`  | Todo el código fuente e historial.                    | Crítico    |
| `.old` / `.bak`  | Código fuente sin interpretar (ej. `config.php.old`). | Alto       |
| `.swp`           | Archivos temporales de edición (Vim).                 | Medio      |
