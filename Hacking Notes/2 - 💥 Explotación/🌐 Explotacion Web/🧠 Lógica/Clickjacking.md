
**Tags:** #pentesting #web-security #clickjacking #frontend #vulnerability

---
## 📑 Índice

1. [[#Conceptos Fundamentales]]
    
2. [[#Técnica Básica con Iframe (Overlay)]]
    
3. [[#Bypass de Scripts Anti-Frame (Frame-Busting)]]
    
    - [[#Bypass con Atributo Sandbox]]
        
    - [[#Otros Métodos de Bypass]]
        
4. [[#Defensas Modernas (Mitigación)]]
    
---
##  Conceptos Fundamentales

El **Clickjacking**, también conocido como **UI Redressing**, es un ataque que engaña a un usuario para que haga clic en un elemento de una página web invisible o disfrazada. El objetivo es que el usuario realice acciones involuntarias (cambiar emails, borrar cuentas, transferir fondos) en una aplicación donde ya está autenticado.

- **Z-Index:** Propiedad CSS que controla la profundidad. El atacante pone la web vulnerable arriba (z-index mayor) pero invisible.
    
- **Opacity:** Permite que el iframe sea invisible para el usuario pero funcional para el navegador.
    
- **Same-Origin Policy (SOP):** Impide que el atacante lea el contenido del iframe, pero **no** impide que el usuario haga clic en él.
    

---

##  Técnica Básica con Iframe (Overlay)

Esta técnica consiste en crear una "página de señuelo" y superponer el sitio vulnerable de forma invisible exactamente encima de un botón falso.

### Ejemplo de Código (PoC)

```HTML
<style>
    iframe {
        width: 1000px;
        height: 1000px;
        /* El secreto: Invisible para el usuario pero clickable */
        opacity: 0.0001; 
        position: absolute;
        top: 0;
        left: 0;
        z-index: 2; /* Siempre encima del contenido falso */
    }
    .wrapper {
        position: relative;
        z-index: 1; /* Debajo del iframe invisible */
    }
    .fake-button {
        position: absolute;
        /* Ajuste preciso de coordenadas */
        top: 480px;
        left: 50px;
        background-color: blue;
        color: white;
        padding: 10px;
        /* Evita que el clic sea capturado por el texto del atacante */
        pointer-events: none; 
    }
</style>

<div class="wrapper">
    <h1 class="fake-button">¡HAS GANADO UN PREMIO! CLICK AQUÍ</h1>
</div>

<iframe src="https://vulnerable-site.com/my-account?email=attacker@evil.com"></iframe>
```

### Explicación Detallada:

1. **`opacity: 0.5` (Pruebas) -> `0.0001` (Ataque):** Se usa 0.5 para alinear el botón azul con el botón real. En el ataque real, se hace casi invisible.
    
2. **`z-index: 2`:** Obliga al navegador a poner el iframe al frente. El usuario cree que clica en el botón azul, pero el clic lo recibe el iframe invisible.
    
3. **`pointer-events: none`:** Crucial. Si el usuario clica exactamente sobre las letras de "Click me", el clic podría ser capturado por el `<h1>` en lugar del iframe. Esta propiedad hace que el clic "atraviese" el texto y llegue al iframe.
    
4. **Pre-filling:** Usamos parámetros URL (`?email=...`) para que el formulario ya tenga los datos que queremos cambiar al hacer clic.
    

---
##  Bypass de Scripts Anti-Frame (Frame-Busting)

Muchos sitios antiguos usan scripts de JavaScript para detectar si están siendo cargados en un iframe y, de ser así, redirigir a la página principal. Ejemplo: `if (top != self) { top.location = self.location; }`

### 1. Bypass con Atributo Sandbox

El estándar HTML5 introdujo el atributo `sandbox` para los iframes. Podemos usarlo para restringir los privilegios del sitio vulnerable y **desactivar su script de detección**.

```HTML
<iframe 
    sandbox="allow-forms" 
    src="https://vulnerable-site.com/my-account?email=a@a.com">
</iframe>
```

**¿Por qué funciona?**

- Al poner **solo** `allow-forms`, permitimos que el usuario envíe el formulario.
    
- Al **no incluir** `allow-scripts`, el JavaScript del sitio vulnerable (el que detecta el iframe) no se ejecuta. El sitio no puede "escapar" del marco.
    

### 2. Otros Métodos de Bypass

#### A. Doble Frame (Double Framing)

En algunos navegadores antiguos, anidar el sitio dentro de dos iframes de distintos dominios confundía la lógica de `top.location`.

#### B. Evento onBeforeUnload

El atacante puede intentar bloquear la redirección del sitio vulnerable mostrando un mensaje de confirmación falso que detenga la carga de la nueva página.

```JavaScript
window.onbeforeunload = function() {
    return "¡Espera! Hay una oferta para ti.";
};
```

#### C. Uso de bibliotecas de "Legacy"

Algunos sitios usan `X-Frame-Options: ALLOW-FROM`, que ya no es soportado por navegadores modernos como Chrome, dejando la puerta abierta si no se configuró una CSP moderna.

---

##  Defensas Modernas (Mitigación)

Para evitar que tu sitio sea usado en estos ataques, se deben implementar estas cabeceras en el servidor:

1. **Content-Security-Policy (CSP):** `Content-Security-Policy: frame-ancestors 'none';` (Nadie puede poner mi sitio en un iframe). `Content-Security-Policy: frame-ancestors 'self';` (Solo mi propio dominio puede).
    
2. **X-Frame-Options (XFO):** `X-Frame-Options: DENY` o `SAMEORIGIN`. Es la defensa clásica y más compatible.
    
3. **Cookies SameSite:** Configurar las cookies de sesión como `SameSite=Lax` o `Strict` evita que el navegador envíe la cookie de sesión automáticamente en peticiones cross-site, neutralizando el impacto del clic.