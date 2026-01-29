
---
Tags: #datos #burpsuite #tipos #php #bypass

---
## 📘 Definición

**Type Juggling** es una vulnerabilidad que aparece cuando un lenguaje de programación (como **PHP**) convierte implícitamente los tipos de datos durante comparaciones, permitiendo a un atacante **alterar la lógica** de validación.

> 📌 Muy común en PHP, debido a su sistema de tipado débil.

---

## 🧠 ¿Cómo funciona?

PHP, al comparar dos variables con `==` (comparación débil), **intenta igualar los tipos de datos automáticamente**, lo que puede llevar a resultados inesperados o peligrosos.

`var_dump("0e12345" == "0");   // true`
`var_dump("0e12345" == 0);     // true`
`var_dump("admin" == true);    // true`

---

## 🧪 Comparación débil (`==`) vs estricta (`===`)

|Comparación|Descripción|Resultado|
|---|---|---|
|`"123" == 123`|Compara número y string, convierte ambos a int|`true`|
|`"123" === 123`|Comparación estricta (tipo y valor)|`false`|
|`"0e12345" == 0`|Interpreta como `0 * 10^12345`|`true`|

---

## 🎯 Vectores comunes de ataque

### 1. 🔑 Comparación de contraseñas hash (0e bypass)
```php
if ($user_input == $hash_stored) {     // acceso concedido }
```

#### ❗ Exploitable si `$hash_stored` = `0e123456...`
```php
md5('240610708') = 0e462097431906509019562988736854
```

✅ Resultado:

`"0e462097431906509019562988736854" == "0e9999999"  // true`

---

### 2. 🔐 Validación de tokens

`if ($_GET['token'] == $secure_token) {     // token válido }`

> Si `$secure_token` comienza por `"0e"` seguido solo de números, cualquier cadena similar también será `== true`.

---

### 3. 💣 Comparaciones booleanas

`if ($_POST['role'] == true) {     // acceso como admin }`

✅ Con `$_POST['role'] = "admin"` → `"admin" == true` → `true`

---

## 🧪 Bypass con valores mágicos

Estos valores producen hashes en forma de `0e...` (interpretado como 0 * 10^n):

|Valor|Hash (`md5()`)|
|---|---|
|`240610708`|`0e462097431906509019562988736854`|
|`QNKCDZO`|`0e830400451993494058024219903391`|
|`aabg7XSs`|`0e087386482136013740957780965295`|

> [!tip] Son útiles cuando el backend compara con `==`.

---

## 🧰 Cómo detectar vulnerabilidad

|Técnica|Explicación|
|---|---|
|Revisión de código|Buscar uso de `==` para validar contraseñas, tokens o IDs|
|Fuzzing de entradas|Probar cadenas como `0e123456`, `"admin"` o `false`|
|Análisis del hash usado|Ver si el sistema usa `md5`, `sha1`, etc. sin verificación de tipo estricta|
|Comparaciones con arrays|Enviar arrays y ver si causa errores o bypass|

---

## ⚔️ Técnicas de explotación

### 1. Buscar un hash tipo `0e...` (con MD5 o SHA1)

`# Buscar input con hash "0e..." (fuerza bruta) python3 -c "import hashlib; print([x for x in range(100000000) if hashlib.md5(str(x).encode()).hexdigest().startswith('0e')])"`

---

### 2. Enviar input con mismo patrón

`$input = "0e123456";   // Válido para saltarse un hash con formato similar`

---

### 3. Manipulación de formularios

Ejemplo en HTML:
```html
<input type="text" name="role" value="admin">
```

Backend:

`if ($_POST['role'] == true) {     // Admin }`

---

## 📦 Herramientas útiles

|Herramienta|Uso|
|---|---|
|Burp Suite|Fuzzing con Intruder para cadenas `0e*`, `false`|
|Hashcat|Buscar colisiones o valores tipo `0e...`|
|wfuzz / ffuf|Automatizar pruebas de bypass en parámetros|
|PHP local|Probar valores con `md5()`, `sha1()` directamente|

---

## 🛡️ Medidas de mitigación

|Medida|Descripción|
|---|---|
|Usar `===` (comparación estricta)|Evita conversiones automáticas de tipo|
|Validar tipos de datos|Convertir y sanitizar antes de comparar|
|Verificar hash con hash_equals()|Función segura en PHP para comparar hashes|
|Evitar hashes con formato `0e...`|Usar algoritmos resistentes a colisiones|
|Aplicar tipado estricto|En lenguajes modernos o con modos como `declare(strict_types=1)`|

---

## 📌 Resumen

|Tipo de ataque|¿Qué se explota?|Ejemplo|
|---|---|---|
|`0e` bypass|Comparación débil con hash numérico|`"0e12345" == 0`|
|Booleans|Comparación con `true` o `false`|`"admin" == true`|
|Comparación con array|PHP lanza error si comparas string == []|`$_GET['id'] == []`|

---
# Referencias
- Type Juggling:[Enlace](https://www.php.net/manual/en/types.comparisons.php)