
---
Tags:- #idor #insecure-objects #authorization #broken-access-control`

---
## 📖 Definición

**IDOR** (Insecure Direct Object Reference) es una vulnerabilidad que ocurre cuando una aplicación expone de forma insegura referencias a objetos internos (archivos, registros de base de datos, identificadores de usuarios, etc.) y no valida adecuadamente si el usuario tiene permisos para acceder a esos objetos.

👉 Básicamente: el atacante modifica un parámetro (ID, número de cuenta, nombre de archivo…) para acceder a recursos que **no debería poder ver o modificar**.

Ejemplo sencillo:

`https://example.com/profile?user_id=123   <-- usuario legítimo https://example.com/profile?user_id=124   <-- atacante cambia el ID y accede a otro perfil`

---

## 🔎 Métodos de Detección

|Método|Descripción|Herramientas útiles|
|---|---|---|
|**Manipulación de parámetros**|Cambiar IDs numéricos, UUIDs, nombres de archivo en parámetros GET/POST.|Burp Suite Repeater, curl|
|**Predictibilidad de identificadores**|Si los IDs son secuenciales, aumenta la probabilidad de IDOR.|SecLists (wordlists), Burp Intruder|
|**Pruebas con usuarios diferentes**|Usar varias cuentas con distintos roles (user vs admin).|Burp Suite con múltiples sesiones|
|**Análisis de respuestas**|Observar diferencias en respuestas (403, 404, 200 con datos ajenos).|Proxy HTTP, diff de respuestas|
|**Forzar navegación directa**|Acceder a endpoints descubiertos sin usar la interfaz web.|Dirbuster, Feroxbuster|

---

## 💥 Métodos de Explotación

|Técnica|Ejemplo|Objetivo|
|---|---|---|
|**Cambio de ID numérico**|`/invoice?id=101` → `/invoice?id=102`|Ver facturas de otros usuarios|
|**Cambio de UUID**|`/order?uuid=550e8400-e29b-41d4-a716-446655440000`|Acceder a pedidos de otros usuarios si no hay validación|
|**Acceso a ficheros**|`/download?file=report_2023.pdf` → `/download?file=report_2022.pdf`|Exfiltrar documentos confidenciales|
|**Modificación de datos**|`POST /update?user=123&email=hacker@evil.com`|Cambiar datos de otras cuentas|
|**Escalada horizontal**|Usuario accede a recursos de otros usuarios|Robo de información|
|**Escalada vertical**|Usuario accede a recursos de administrador|Control de sistema|

---

## 🛠 Ejemplos prácticos

### 1. Explotación básica en GET

`GET /account?id=1001 HTTP/1.1 Host: vulnerable.htb Cookie: session=abc123`

➡ Cambiamos `id=1001` por `id=1002` y accedemos a otra cuenta.

---

### 2. Explotación en POST

`POST /updateUser HTTP/1.1 Host: vulnerable.htb Content-Type: application/x-www-form-urlencoded  id=1002&role=admin`

➡ Si no hay validación, podemos modificar usuarios ajenos o escalar privilegios.

---

### 3. Explotación en APIs REST

`GET /api/v1/users/123 Authorization: Bearer <token_usuario_normal>`

➡ Cambiando `123` por `124` accedemos a otro perfil.

---

### 4. File Download IDOR

`GET /download?file=contract_123.pdf`

➡ Cambiando el nombre del archivo obtenemos documentos sensibles.

---

## 📌 Consecuencias comunes

- Robo de información sensible (datos personales, financieros, médicos).
    
- Modificación o borrado de datos de otros usuarios.
    
- Escalada de privilegios (si se puede manipular roles, permisos, etc.).
    
- Exposición de ficheros confidenciales en el servidor.
    

---

## 🧪 Técnicas de apoyo

- **Forzar navegación directa:** probar rutas descubiertas en `/api/`, `/admin/`, `/files/`.
    
- **Combinación con fuzzing:** usar listas como `SecLists/Discovery/` para adivinar objetos accesibles.
    
- **Uso de dos cuentas distintas:** ayuda a detectar diferencias de permisos.
    
- **Automatización:** Burp Intruder o `ffuf` para iterar sobre IDs numéricos.
    

Ejemplo con `ffuf`:

`ffuf -u https://target.htb/profile?id=FUZZ -w /usr/share/seclists/Discovery/IDs.txt -b "session=abc123"`

---

## 🛡 Mitigación

- Implementar **controles de autorización a nivel de objeto** (ABAC / RBAC).
    
- Usar identificadores **no predecibles** (UUID, hash aleatorio).
    
- Validar siempre que el usuario tenga permisos sobre el objeto solicitado.
    
- Revisar **APIs REST** y endpoints ocultos.
    
- Tests automáticos de seguridad (DAST/SAST).
    

---

## 🎯 Resumen

- **Qué es:** IDOR = acceso a objetos internos sin autorización adecuada.
    
- **Cómo detectarlo:** manipulando parámetros, forzando IDs, probando con distintas cuentas.
    
- **Cómo explotarlo:** acceder, modificar o eliminar recursos de otros usuarios.
    
- **Impacto:** fuga de información, corrupción de datos, escalada de privilegios.
    
- **Mitigación:** validación de autorización en backend + IDs impredecibles.