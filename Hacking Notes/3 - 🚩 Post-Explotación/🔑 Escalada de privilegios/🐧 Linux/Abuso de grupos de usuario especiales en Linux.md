
---
Tags: #privilege-escalation #grupos #docker

---
## 1️⃣ Concepto

Algunos grupos de usuario en Linux dan **privilegios especiales** sobre el sistema.  
Si un atacante consigue pertenecer a estos grupos, puede **escalar privilegios o acceder a recursos críticos**.

**Idea general:** Identificar a qué grupos pertenece tu usuario y si alguno te da acceso que normalmente requiere root.

---

## 2️⃣ Grupos críticos y su potencial de abuso

|Grupo|Privilegios / Abuso típico|Ejemplo práctico|
|---|---|---|
|**sudo**|Permite ejecutar cualquier comando como root usando `sudo`.|`sudo bash` → shell root.|
|**docker**|Puede ejecutar contenedores y montar directorios de host; acceso root sobre la máquina a través de contenedores.|`docker run -v /:/mnt/root -it ubuntu bash` → accedes a todo el host.|
|**wheel**|Similar a `sudo`, en algunas distribuciones controla permisos de administración.|`sudo -i` para obtener root.|
|**adm**|Permite leer logs del sistema (`/var/log`).|`cat /var/log/auth.log` → ver actividad de usuarios.|
|**lpadmin**|Administración de impresoras; en algunas máquinas puede permitir ejecución de scripts como root.|Escenario raro, útil en auditorías de seguridad física.|
|**shadow**|Acceso a `/etc/shadow`. Permite lectura de hashes de contraseñas de usuarios.|`sudo cat /etc/shadow` si tienes permiso.|
|**kvm, libvirt**|Controla máquinas virtuales; puede permitir acceso a memoria del host o snapshots.|`virsh dumpxml VM` → posibles vulnerabilidades.|
|**wireshark / tcpdump**|Acceso a interfaces de red en modo promiscuo.|Captura tráfico sensible.|

---

## 3️⃣ Ejemplo crítico: abuso del grupo **docker**

### 🔹 Contexto

- Usuario `kali` pertenece al grupo `docker`.
    
- Docker corre como root.
    
- Montando todo el sistema host dentro del contenedor puedes tener **acceso total al host sin root directo**.
    

### 🔹 Comando

`docker run -v /:/mnt/root -it ubuntu bash`

- `-v /:/mnt/root` → monta la raíz del host dentro del contenedor.
    
- `-it ubuntu bash` → lanza una shell interactiva en Ubuntu.
    
- Dentro del contenedor:
    

`ls /mnt/root cat /mnt/root/etc/shadow`

- Resultado: acceso **root indirecto** sobre todo el host.
    

---

## 4️⃣ Cómo identificar grupos de interés

`id                  # Muestra grupos de tu usuario groups               # Otra forma rápida getent group         # Lista todos los grupos del sistema`

- Busca grupos con privilegios especiales (`sudo`, `docker`, `adm`, `shadow`, `kvm`, `wireshark`).
    
- Si tu usuario está en alguno, evalúa qué acceso permite y si se puede abusar para escalada de privilegios.
    

---

## 5️⃣ Consejos prácticos

- Cambiar a un grupo con `newgrp` sin cerrar sesión:
    

`newgrp docker`

- Revisar permisos de sockets o archivos especiales:
    

`ls -l /var/run/docker.sock   # socket Docker ls -l /etc/shadow            # archivo sensible`

- Siempre documenta los pasos y no dejes rastros en `/tmp` ni en logs del sistema cuando pruebas escaladas de privilegios en entornos controlados.