
---
Tags: #bytearrays #badcharts #buffer #bufferoverflow #mona #inmunityDebuger

---
# Objetivo 

Generar una secuencia de bytes para identificar qué bytes son corruptos/filtrados al ser escritos en memoria y luego excluirlos del shellcode.

---

# Flujo mínimo (4 pasos)

1. Generar bytearray con todos los valores excepto `\x00`.
    
2. Insertarlo en payload en la posición justo después del control de EIP/RIP.
    
3. En Immunity Debugger, provocar crash y localizar la dirección donde quedó la secuencia.
    
4. Ejecutar `!mona compare` para ver qué bytes faltan (badchars). Generar nuevo bytearray excluyendo esos badchars.
    

---

# Mona: comandos clave

- Generar bytearray desde mona (opcional):  
    `!mona bytearray -cpb "\x00"`  
    crea `bytearray.bin` con todos los bytes excepto `\x00`.
    
- Tras crash, identificar dirección donde se escribió el buffer (por ejemplo ESP, EIP-4, etc.).
    
- Comparar memoria con fichero:  
    `!mona compare -f C:\mona\bytearray.bin -a <direccion>`  
    Resultado indica los badchars detectados.
Comandos útiles (Immunity Debugger, dentro del depurador):

- Generar bytearray excluyendo `\x00`:
    

---
# Flujo

`!mona bytearray -cpb "\x00"`

Esto crea `bytearray.bin` (ubicación mostrada por mona).

- Generar excluyendo varios bytes:
    

`!mona bytearray -cpb "\x00\x0a\x0d"`

- Comparar el archivo con la memoria tras el crash (suponiendo que apuntas a la dirección donde quedó la secuencia):
    

`!mona compare -f C:\mona\bytearray.bin -a 0x0012FFB0`

Salida: lista de bytes que no coinciden = **badchars**.