# 🛡️ Hacking & Offensive Security - Master Notes (Obsidian).

Este repositorio es una enciclopedia técnica de Pentesting y Red Teaming, estructurada siguiendo las fases de una auditoría real y documentando vectores de ataque críticos.

---

## 📑 Índice de Contenidos Detallado

### 1. 🔍 Reconocimiento y Enumeración
* **OSINT:** Metodologías de recolección de información.
* **Enumeración de Sistemas:** * Estrategias de reconocimiento en **Linux** (`linux-smart-enumeration`).
  * Enumeración de infraestructura en redes locales.
* **Herramientas de Reconocimiento:** Automatización y descubrimiento de activos.

### 2. 🧠 Explotación de Binarios (Buffer Overflow)
Metodología completa para explotación de memoria en el Stack:
1. **Fuzzing:** Control del registro EIP.
2. **Asignación de Memoria:** Espacio para el Shellcode.
3. **Badchars:** Generación de Bytearrays y detección de caracteres prohibidos.
4. **OpCodes:** Búsqueda de saltos al ESP (`JMP ESP`).
5. **NOPs & Shellcoding:** Desplazamientos en pila y creación manual de Shellcodes.
6. **Entornos:** Metodologías para **Linux (GDB)** y **Windows (Immunity Debugger + Mona.py)**.

### 3. 🌍 Explotación Web Avanzada (Full Stack)
Clasificación detallada según el vector de ataque:

* **Inyecciones de Código:**
  * **SQLi:** Manual, automatizada (Sqlmap) y **Ataque de Truncado SQL**.
  * **NoSQLI, XPath, LDAP, Latex Injection** y **CSSI (CSS Injection)**.
* **Vulnerabilidades de Lado del Servidor (Server-Side):**
  * **LFI (Local File Inclusion):** Uso de `Php_filter_chain_generator.py` y wrappers.
  * **SSRF, SSTI** y **XXE (XML External Entity)**.
  * **Deserialización:** Explotación en **Python (Pickle y Yaml)**.
  * **Ejecución de Comandos:** ShellShock y ataques de transferencia de zona (**AXFR**).
  * **Otros:** **Squid Proxy**, **WebDav** y **CORS**.
* **Vulnerabilidades de Lado del Cliente y Lógica:**
  * **Sesión:** CSRF, **CSTI**, **Session Puzzling**, Fixation y Variable Overloading.
  * **Identidad/API:** **JWT** (Enumeración), **GraphQL** (Introspección e IDORs).
  * **Lógica de Negocio:** **Mass-Assignment**, **Race Conditions**, **Type Juggling** y **Padding Oracle Attack**.
  * **Ficheros:** **File Upload Bypass** y **Open Redirect**.

### 4. 🔑 Escalada de Privilegios (PrivEsc)
* **Linux Privilege Escalation:**
  * **Hijacking:** Python Library Hijacking y Shared Library Hijacking.
  * **Sistemas:** Abuso de **Sudoers**, binarios **SUID**, **Capabilities** y tareas **Cron**.
  * **Persistencia:** Abuso de grupos especiales y permisos de archivos incorrectos.
* **Active Directory (AD):**
  * Enumeración y explotación de certificados con **Certipy** y **BloodyAD**.

### 5. 🚩 Post-Explotación y Movimiento Lateral
* **Pivoting:** Salto entre redes usando **Chisel**, **Socat** y **Netsh Portproxy**.
* **Gestión de Shells:** Bind Shells, Forward Shells y **Reverse Shells**.
* **Payloads:** Generación con **Msfvenom** y Metasploit (Staged vs Non-Staged).

### 6. 📦 Gestores de Contenido (CMS) y Herramientas
* **CMS Auditing:** WordPress (Wpscan/xmlrpc.php), Drupal (Droopescan), Joomla y Magento.
* **Utilidades:** Seguridad en **Docker**, **Herramientas de Fuerza Bruta** y gestión de imágenes.

---
*Organizado y mantenido como un Cerebro Digital en **Obsidian**.*

---

## ☕ Apoya mi trabajo

Si estos apuntes te han sido de utilidad para tus certificaciones (eJPT, PNPT, OSCP) o laboratorios, considera apoyarme para seguir manteniendo y actualizando esta base de conocimientos.

[<img src="https://cdn.ko-fi.com/cdn/kofi1.png?v=3" width="200">](https://ko-fi.com/luisrh98)

---

## 📜 Licencia / License

<p align="center">
  <a href="http://creativecommons.org/licenses/by-nc-sa/4.0/">
    <img src="https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey.svg?style=for-the-badge" alt="License: CC BY-NC-SA 4.0">
    <img src="https://img.shields.io/badge/Sales-Prohibited-red?style=for-the-badge" alt="Sales Prohibited">
  </a>
</p>

**🇪🇸 Español:**
Este repositorio contiene apuntes personales de ciberseguridad protegidos bajo la licencia **CC BY-NC-SA 4.0**. Se permite su uso, edición y redistribución **siempre que no sea con fines comerciales**, se mantenga la misma licencia y se cite al autor original (**luisrh98**). **Queda terminantemente prohibida la venta de este contenido.**

**🇺🇸 English:**
This repository contains personal cybersecurity notes protected under the **CC BY-NC-SA 4.0** license. Usage, editing, and redistribution are allowed **provided they are not for commercial purposes**, the same license is maintained, and the original author (**luisrh98**) is credited. **Selling this content is strictly prohibited.**

---
