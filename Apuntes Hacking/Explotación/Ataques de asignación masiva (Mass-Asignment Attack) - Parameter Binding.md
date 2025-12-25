
---
Tags: #web #burpsuite #parametros #json

---
## 📌 Definición

El **Mass Assignment Attack** (también llamado **Parameter Binding** o **Overposting**) ocurre cuando una aplicación **asigna automáticamente** los valores recibidos en una solicitud HTTP a las propiedades de un objeto o modelo **sin validar ni filtrar los campos permitidos**.

Esto permite a un atacante **modificar campos sensibles** que no deberían ser accesibles desde el cliente, como:

- Roles de usuario
    
- Estados de cuenta
    
- IDs de otros usuarios
    
- Configuraciones internas
    

---

## 🧠 Funcionamiento lógico

1. El backend recibe datos desde un **formulario o API**.
    
2. El framework usa un **mapeo automático** para asignar esos datos al modelo.
    
3. Si no hay una **lista blanca** (`whitelist`) de campos permitidos, **cualquier campo existente en el modelo se asigna**.
    
4. El atacante puede **enviar parámetros adicionales** para modificar datos críticos.
    

---

## 🔍 Detección

- Revisar si la API acepta **más campos** de los esperados en el formulario original.
    
- Probar enviando **campos ocultos** o **valores inesperados** en JSON, XML, form-data.
    
- Revisar código en frameworks que usan:
    
    - **Ruby on Rails** (`params.permit` / `attr_protected`)
        
    - **Laravel** (`$fillable` / `$guarded`)
        
    - **Spring Boot** (Java Bean Binding)
        
    - **Express.js + Mongoose** (MongoDB)
        
    - **Django Rest Framework** (serializers automáticos)
        

---

## 💥 Vectores de ataque comunes

|Tipo|Ejemplo de parámetro inyectado|Resultado|
|---|---|---|
|Escalar privilegios|`role=admin`|Usuario normal pasa a admin|
|Acceder a datos ajenos|`user_id=5`|Modifica datos de otro usuario|
|Cambiar estado de cuenta|`is_active=true`|Reactiva cuenta bloqueada|
|Saltar pagos|`subscription_paid=true`|Habilita cuenta premium gratis|

---

## 🚀 Ejemplos de explotación

### 1️⃣ Ruby on Rails

`POST /users HTTP/1.1 Content-Type: application/json  {   "username": "juan",   "password": "123456",   "admin": true }`

Si el modelo `User` no restringe `admin`, el usuario se crea con privilegios.

---

### 2️⃣ Laravel (PHP)

`User::create($request->all());`

Ataque:

`POST /register HTTP/1.1 Content-Type: application/json  {   "name": "test",   "email": "a@b.com",   "password": "1234",   "is_admin": 1 }`

---

### 3️⃣ Node.js (Express + Mongoose)

`User.create(req.body);`

Ataque:

`POST /api/users Content-Type: application/json  {   "username": "hack",   "role": "admin" }`

---

## 🛠 Herramientas útiles

- **Burp Suite / OWASP ZAP** → Añadir parámetros ocultos en requests.
    
- **Postman** → Enviar campos extra en JSON/XML.
    
- **Param Miner (Burp)** → Descubrir parámetros adicionales.
    
- **Autorize (Burp)** → Comprobar cambios de permisos.
    

---

## 🎯 Metodología de prueba

1. **Enumerar campos esperados** (HTML, API docs, respuestas JSON).
    
2. **Inyectar campos sospechosos** (`is_admin`, `role`, `user_id`, `balance`, `is_premium`).
    
3. Observar **respuestas y cambios** en la base de datos o en la UI.
    
4. Si es vulnerable → Explorar otras entidades del sistema (users, products, payments).
    

---

## 🔓 Bypass comunes

- Enviar parámetros con **camelCase / snake_case** alternativos (`isAdmin` / `is_admin`).
    
- Usar **JSON nested objects** (`profile[role]=admin`).
    
- Probar **arrays de objetos** para afectar múltiples registros a la vez.
    
- Enviar **null** o **valores booleanos** que cambien la lógica interna.
    

---

## 🛡 Mitigaciones

- Usar **listas blancas** (`whitelist`) para parámetros permitidos.
    
- Evitar asignaciones masivas (`$request->all()` en Laravel, `req.body` directo en Node).
    
- Usar **DTOs o serializers** que definan campos explícitos.
    
- Validar y filtrar siempre en el servidor, **no en el cliente**.
    

---

## ⚠️ Impacto

- Escalada de privilegios.
    
- Modificación de registros ajenos.
    
- Activación o desactivación de cuentas.
    
- Fraude y bypass de pagos.