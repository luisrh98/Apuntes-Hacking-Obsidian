
---
Tags: #web #pentesting #session #session-fixation #session-hijacking #session-puzzling #session-overloading #owasp #insecure-session

---
## 📖 Definición General

Son vulnerabilidades relacionadas con la **gestión insegura de sesiones** en aplicaciones web.  
Un atacante puede manipular variables de sesión o fijar un `Session ID` para:

- **Secuestrar la sesión** de un usuario.
    
- **Confundir la aplicación** con variables de sesión conflictivas.
    
- **Forzar privilegios** o comportamiento no esperado.
    

---

## 🧩 1. Session Puzzling

### 🔹 Definición

Se produce cuando **la misma variable de sesión es reutilizada en diferentes contextos lógicos**, provocando que los datos de una sesión se apliquen a otra parte de la aplicación.

### 🔹 Ejemplo

1. Variable de sesión `role=guest` en un formulario de compras.
    
2. En otra parte de la aplicación, esa misma variable `role` se interpreta como permisos de usuario.
    
3. Un atacante manipula el flujo y consigue que `role=admin` quede fijado en sesión.
    

### 🔹 Impacto

- Escalada de privilegios.
    
- Acceso a áreas restringidas.
    
- Inconsistencia en la lógica de negocio.
    

---

## 🔒 2. Session Fixation

### 🔹 Definición

El atacante **fija un Session ID conocido** antes de que la víctima se autentique.  
Cuando la víctima inicia sesión, ese `Session ID` ya está bajo control del atacante.

### 🔹 Flujo de ataque

1. Atacante genera un `Session ID` válido:
    
    `http://vulnerable.htb/login.php?PHPSESSID=1234abcd`
    
2. Envía este enlace a la víctima.
    
3. La víctima inicia sesión, la aplicación **no renueva el ID de sesión** tras loguearse.
    
4. El atacante reutiliza `PHPSESSID=1234abcd` para secuestrar la sesión autenticada.
    

### 🔹 Impacto

- Hijacking completo de la sesión.
    
- Suplantación de identidad.
    

---

## 🌀 3. Session Variable Overloading

### 🔹 Definición

Ocurre cuando **la aplicación usa la misma clave de sesión para almacenar diferentes datos**, dependiendo del flujo.  
Un atacante puede **inyectar datos maliciosos** en esa variable para alterar la lógica.

### 🔹 Ejemplo

// En checkout.php
`$_SESSION["user"] = "attacker";   // se espera un string  

// En admin.php
`if ($_SESSION["user"]["is_admin"] === true) {     // acceso admin }`

⚠️ La variable fue sobrecargada (`string` → `array`), lo que puede permitir al atacante redefinir la sesión.

### 🔹 Impacto

- Confusión de tipos y estructuras.
    
- Escalada a permisos administrativos.
    
- Ejecución de lógica insegura.
    

---

## 🔍 Métodos de Detección

- Revisar si el **Session ID cambia** tras login/logout.
    
- Probar accesos con **Session ID fijados manualmente**.
    
- Verificar **colisiones de variables de sesión** (`role`, `user`, `auth`).
    
- Revisar si la aplicación usa **la misma variable en distintos contextos**.
    

---

## 🎯 Métodos de Explotación

- **Session Fixation** → enviar enlaces con Session ID predefinido.
    
- **Session Puzzling** → probar variaciones de la misma variable (`role`, `auth`, `is_admin`).
    
- **Variable Overloading** → sobrecargar sesión con valores distintos (`string` → `array`, `int` → `string`).
    

---

## 🛡️ Medidas de Mitigación

- Regenerar `Session ID` tras login/logout (`session_regenerate_id()` en PHP).
    
- Usar **cookies seguras** (`HttpOnly`, `Secure`, `SameSite`).
    
- Separar claramente variables de sesión por contexto (no reutilizar `user`, `role`, etc.).
    
- Validar tipos y estructuras de las variables de sesión.
    
- Invalidar sesión en el servidor al cerrar sesión.
    

---

## 🚨 Impacto Global

- **Session Hijacking**
    
- **Escalada de privilegios**
    
- **Acceso a datos sensibles**
    
- **Inconsistencias en la lógica de negocio**
    

---
