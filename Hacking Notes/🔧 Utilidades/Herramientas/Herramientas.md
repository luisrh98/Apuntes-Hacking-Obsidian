### Índice

1. [[#1. Reconocimiento y OSINT]] (Fase Inicial)
    [[#🔍 OSINT (Fuentes Abiertas)]]
    [[#📡 Reconocimiento de Red]]
	
2. [[#2. Enumeración y Fuzzing]] (Superficie de Ataque)
    [[#🌐 Enumeración Web (Fingerprinting)]]
    [[#📂 Fuzzing de Contenido (Directorios y Parámetros)]]
	
3. [[#3. Explotación por Técnicas]] (Vectores Web y Red)
    [[#💉 SQL Injection (SQLi)]]
    [[#🔄 Deserialización Insegura]]
    [[#💻 Inyección de Comandos y Web]]
    [[#🔑 Fuerza Bruta (Authentication)]]
	
4. [[#4. Infraestructura y Pivoting]] (Movimiento Lateral)
    [[#🔗 Túneles y Proxies]]
	
5. [[#5. Post-Explotación]] (PrivEsc y Persistencia)
    [[#📈 Escalada de Privilegios (PrivEsc)]]
    [[#🐚 Gestión de Shells y Payloads]]
	
6. [[#6. Especializadas]] (Buffer Overflow)
    [[#🧠 Buffer Overflow (Binary Exploitation)]]
	
[[#🤖 Otras]](Botnets...)

---

## 1. Reconocimiento y OSINT

_El primer paso: descubrir qué hay ahí fuera._

### 🔍 OSINT (Fuentes Abiertas)

- **Correos y Datos:** [[Hunter.io]], [[Intelligence X]], [[Phonebook.cz]], [[Clearbit Connect]], [[TheHarvester]].
    
- **Leaks y Brechas:** [[DeHashed]], [[HaveIBeenPwned]].
    
- **Imágenes:** [[PimEyes]].
    

### 📡 Reconocimiento de Red

- [[Nmap]] - Escaneo de puertos y servicios.
    
- [[RustScan]] - Escaneo de puertos ultra rápido (Rust).
    

---

## 2. Enumeración y Fuzzing

_Entendiendo las tecnologías y buscando rutas ocultas._

### 🌐 Enumeración Web (Fingerprinting)

- [[Wappalyzer]] / [[Whatweb]] / [[BuiltWith]] - Identificación de stacks tecnológicos.
    
- [[Joomscan]], [[Droopescan]], [[Magescan]], [[Wpscan]] - Escáneres específicos para CMS.
    

### 📂 Fuzzing de Contenido (Directorios y Parámetros)

|**Herramienta**|**Especialidad**|
|---|---|
|[[Gobuster]]|Rapidez en directorios y subdominios.|
|[[Ffuf]]|El más rápido y flexible para fuzzing masivo.|
|[[Wfuzz]]|Ideal para fuzzing de parámetros y payloads.|

---

## 3. Explotación por Técnicas

_Vectores específicos para comprometer el sistema._

### 💉 SQL Injection (SQLi)

- [[Sqlmap]] - Automatización total de inyecciones SQL.
    

### 🔄 Deserialización Insegura

- [[ysoserial]] - Generación de payloads Java.
    
- [[PHPGGC]] - Gadget chains para frameworks PHP.
    
- [[phar_jpg_polyglot]] - Ataques PHAR vía políglotos JPG.
    
- [[Universal Ruby Gadget]] - Marshalling en Ruby.
    

### 💻 Inyección de Comandos y Web

- [[Commix]] - Automatización de Command Injection.
    
- [[Burp Suite]] - Intercepción, Repeater e Intruder (Imprescindible).
    

### 🔑 Fuerza Bruta (Authentication)

- [[Hydra]] - Login cracker multi-protocolo.
    
- [[Hashcat]] / [[John the Ripper]] - Crackeo de hashes offline.
    

---

## 4. Infraestructura y Pivoting

_Cuando ya estás dentro y quieres moverte por la red interna._

### 🔗 Túneles y Proxies

- [[Pivoting – Chisel, Socat y Netsh Portproxy]] - Túneles sobre HTTP/TCP.
    
- [[Ligolo-ng]] - Túneles de red mediante interfaces TUN.
    
- [[Proxychains]] - Enrutado de herramientas a través de proxies.
    

---

## 5. Post-Explotación

_Asegurando el control y subiendo privilegios._

### 📈 Escalada de Privilegios (PrivEsc)

|**Sistema**|**Herramientas Sugeridas**|
|---|---|
|**Linux**|[[Linpeas]], [[Linux-smart-enumeration]]|
|**Windows**|[[Winpeas]], [[PowerUp]], [[Seatbelt]]|

### 🐚 Gestión de Shells y Payloads

- [[Msfvenom]] - Creación de reversas y binarios maliciosos.
    
- [[Metasploit]] - Framework de explotación y gestión de sesiones.
    

---

## 6. Especializadas

_Técnicas de bajo nivel y otros escenarios._

### 🧠 Buffer Overflow (Binary Exploitation)

- [[Buffer Overflow en Linux usando GDB y herramientas de Metasploit (sin framework)]].
    
- [[Buffer Overflow en Windows con Immunity Debugger + Mona]].
    
- [[mona.py]] - Automatización de búsqueda de offsets y JMP.
    

### 🤖 Otras

- [Creación de botnets (Ufonet)](https://github.com/epsylon/ufonet).