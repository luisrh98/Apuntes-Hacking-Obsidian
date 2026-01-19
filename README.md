# 🛡️ Hacking & Offensive Security – Master Notes (Obsidian)

Este repositorio es una **enciclopedia técnica avanzada de Pentesting y Red Teaming**, concebida como un **cerebro digital en Obsidian** y organizada siguiendo **el flujo real de una auditoría de seguridad profesional**.

Centraliza **metodologías, técnicas ofensivas, explotación práctica y documentación técnica**, con un enfoque **learning by doing**, orientado tanto a **laboratorios controlados** como a **entornos corporativos reales**.

---

## 🎯 Objetivo del repositorio

- Actuar como **base de conocimiento viva** en ciberseguridad ofensiva
    
- Documentar **vectores de ataque críticos** con ejemplos prácticos y reproducibles
    
- Reforzar la **metodología profesional de pentesting end‑to‑end**
    
- Facilitar la preparación de **certificaciones y escenarios reales**
    
- Servir como **portfolio técnico** y referencia personal en evolución continua
    

---

## 📑 Índice de Contenidos

### 1️⃣ 🔍 Reconocimiento y Enumeración

- **Escaneo de Puertos y Servicios**
    
    - Nmap
        
- **Fuzzing de Directorios y Subdominios**
    
    - Ffuf, Gobuster, Wfuzz
        
- **Fuerza Bruta**
    
    - Hydra
        
- **Enumeración de Sistemas**
    
    - Linux Smart Enumeration
        
- **OSINT**
    
    - Google Dorks
        
    - Bases de datos filtradas (DeHashed)
        
    - Correos electrónicos y RRSS (Hunter, Clearbit, Intelligence X, Phonebook)
        
    - Análisis de imágenes (PimEyes)
        
- **Identificación de Tecnologías Web**
    
    - Wappalyzer, WhatWeb
        

---

### 2️⃣ 💥 Explotación

#### 🧠 Explotación de Binarios – Buffer Overflow

- Metodología completa paso a paso:
    
    - Fuzzing y control de EIP
        
    - Gestión de memoria y espacio para shellcode
        
    - Detección de badchars
        
    - Búsqueda de OpCodes (JMP ESP)
        
    - Uso de NOPs y shellcoding
        
    - Modificación de shellcodes
        
- Entornos:
    
    - Linux (GDB y herramientas de Metasploit sin framework)
        
    - Windows (Immunity Debugger + Mona)
        
- Creación manual de shellcodes
    

---

#### 🌐 Explotación Web

- **Vulnerabilidades Web Generales**
    
    - IDOR
        
    - File Upload Bypass
        
    - CORS
        
    - JWT
        
    - Padding Oracle
        
    - Prototype Pollution
        
    - Type Juggling
        
    - Session Fixation / Puzzling / Variable Overloading
        
    - WebDAV
        
    - Squid Proxy
        

##### 💉 Inyecciones

- **SQL Injection**
    
    - Manual
        
    - Error‑Based
        
    - Blind (Time / Boolean / Conditional)
        
    - Out‑of‑Band
        
    - SQLi vía XXE
        
    - Sqlmap
        
- **Otras Inyecciones**
    
    - XSS (incluye DOM‑Based)
        
    - CSRF
        
    - CSTI
        
    - SSTI
        
    - SSRF
        
    - NoSQLi
        
    - XPath Injection
        
    - XXE
        
    - CSS Injection
        
    - Latex Injection
        
    - GraphQL (Introspection, Mutations, IDORs)
        

##### 📂 Inclusión de Ficheros

- LFI
    
- PHP Filter Chains
    

##### 🧠 Lógica de Negocio

- Clickjacking
    
- Race Conditions
    
- Open Redirect
    

##### 🧬 Deserialización

- Deserialización genérica
    
- Python Pickle
    
- Python YAML
    

##### 🧩 Gestores de Contenido (CMS)

- WordPress
    
- Drupal
    
- Joomla
    
- Magento
    

---

#### 🛰️ Servicios e Infraestructuras

- AXFR (DNS Zone Transfer)
    
- LDAP Injection
    
- ShellShock
    

---

### 3️⃣ 🚩 Post‑Explotación

#### 🔁 Pivoting

- Chisel
    
- Socat
    
- Netsh Portproxy
    

#### 🔑 Escalada de Privilegios

- **Linux**
    
    - SUID
        
    - Capabilities
        
    - Cron Jobs
        
    - Sudoers
        
    - PATH Hijacking
        
    - Python / Shared Library Hijacking
        
    - Docker Breakout
        
    - Abuso de permisos y grupos
        
- **Windows**
    
- **Active Directory**
    
    - Enumeración
        
    - Escalada con Certipy y BloodyAD
        

---

### 4️⃣ 📚 Reportes

- Redacción de informes técnicos profesionales
    
- Documentación en LaTeX
    

---

### 5️⃣ 🔧 Utilidades

#### 🧪 Entornos

- Docker
    

#### 📦 Payloads

- Msfvenom
    
- Payloads Staged y Non‑Staged
    
- Metasploit (uso de payloads)
    

#### 🖥️ Shells

- Reverse Shell
    
- Bind Shell
    
- Forward Shell
    
- Conversión a TTY interactiva
    

#### 🧰 Frameworks

- Metasploit Framework
    

---

_Organizado y mantenido como un **Cerebro Digital en Obsidian**._

---

## ☕ Apoya mi trabajo

Si estos apuntes te han sido útiles para certificaciones (eJPT, PNPT, OSCP) o laboratorios prácticos, puedes apoyarme para seguir manteniendo y ampliando esta base de conocimientos.

[<img src="https://cdn.ko-fi.com/cdn/kofi1.png?v=3" width="200">](https://ko-fi.com/luisrh98)

---

## 📜 Licencia / License

<p align="center"> <a href="http://creativecommons.org/licenses/by-nc-sa/4.0/"> <img src="https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey.svg?style=for-the-badge" alt="License: CC BY-NC-SA 4.0"> <img src="https://img.shields.io/badge/Sales-Prohibited-red?style=for-the-badge" alt="Sales Prohibited"> </a> </p>

**🇪🇸 Español**  
Este repositorio contiene apuntes personales de ciberseguridad protegidos bajo la licencia **CC BY‑NC‑SA 4.0**. Se permite su uso, edición y redistribución **siempre que no sea con fines comerciales**, se mantenga la misma licencia y se cite al autor original (**luisrh98**). **Queda terminantemente prohibida la venta de este contenido.**

**🇺🇸 English**  
This repository contains personal cybersecurity notes protected under the **CC BY‑NC‑SA 4.0** license. Usage, editing, and redistribution are allowed **provided they are not for commercial purposes**, the same license is maintained, and the original author (**luisrh98**) is credited. **Selling this content is strictly prohibited.**