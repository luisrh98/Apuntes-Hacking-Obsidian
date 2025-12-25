--------------------------
Tags: #herramientas #fuerzabruta #scripting #login #servicios #contraseñas #usuarios #hydra

------------------
# Definición

[[Hydra]] es una herramienta de ciberseguridad utilizada en hacking ético para realizar ataques de fuerza bruta en cuentas de servicios, permitiendo probar la seguridad de sistemas mediante la simulación de ciberataques controlados.
En cuanto a las tecnologías que utiliza, Hydra puede trabajar con más de 40 protocolos distintos, como **HTTP, FTP, SSH, SMB, entre otros**, lo que le permite realizar ataques de fuerza bruta en diversos tipos de sistemas.

------------------
# Parámetros y usos

|   |   |
|---|---|
|Parámetro|Descripción|
|-l [usuario]|Define un único nombre de usuario para probar en el ataque de fuerza bruta.|
|-L [archivo_usuarios]|Proporciona un archivo de texto con una lista de nombres de usuario a probar, uno por línea.|
|-p [contraseña]|Define una única contraseña para probar con los usuarios especificados.|
|-P [archivo_contraseñas]|Proporciona un archivo de texto (diccionario) con una lista de contraseñas a probar, una por línea.|
|-t [hilos]|Establece el número de tareas o conexiones paralelas que Hydra ejecutará simultáneamente, afectando la velocidad del ataque.|
|-V|Activa el modo verboso, mostrando más detalles sobre el progreso del ataque y los intentos realizados.|
|-vV|Activa el modo muy verboso, proporcionando aún más información detallada, incluyendo las combinaciones de usuario/contraseña que se están probando.|
|-s [puerto]|Especifica el puerto de destino del servicio si es diferente del puerto estándar para el protocolo que se está atacando.|
|-f|Indica a Hydra que se detenga y salga inmediatamente al encontrar la primera combinación válida de usuario y contraseña.|
|-W [Segundos]|Espera el número de segundos especificado tras un intento fallido, útil para evitar bloqueos por parte del servidor.|

#### Ejemplos Prácticos de Hydra

- Estructura básica de Hydra:  
    ```bash
    hydra -l [usuario_o_L_archivo] -p [pass_o_P_archivo] [protocolo]://[IP_o_dominio] [opciones]  
```
      
    Esta es la sintaxis general. Debes especificar un usuario (-l) o un archivo de usuarios (-L), y una contraseña (-p) o un archivo de contraseñas (-P), seguido del protocolo y el objetivo.
    
- Ataque de fuerza bruta FTP con un usuario y un diccionario de contraseñas:  
    ```bash
    hydra -l admin -P /usr/share/wordlists/rockyou.txt ftp://192.168.1.100 -t 15 -V  
```
      
    Intenta iniciar sesión en el servidor FTP en 192.168.1.100 con el usuario admin (-l admin) y prueba cada contraseña del diccionario rockyou.txt (-P). Utiliza 15 hilos (-t 15) y muestra la salida detallada (-V).
    
- Ataque de fuerza bruta SSH con un diccionario de usuarios y contraseñas (saliendo al primer éxito):  
    ```bash
    hydra -L users.txt -P passwords.txt ssh://target.com -f  
```
      
    Intenta combinaciones de usuarios de users.txt (-L) y contraseñas de passwords.txt (-P) contra el servicio SSH en target.com, deteniéndose al encontrar la primera credencial válida (-f).
    
- Usar retardo entre intentos para reducir la agresividad del ataque:  
    ```bash
    hydra -l luisBOOM -P rockyou.txt -W 3 -t 2 -s 21 -V 127.0.0.1 ftp  
```
      
    Explicación: Este comando intenta adivinar la contraseña para el usuario luisBOOM en el servidor FTP 127.0.0.1 (puerto 21).
    
- -W 3: Espera 3 segundos tras cada intento fallido, lo cual es crucial para evitar bloqueos por parte de los sistemas de detección de intrusiones o cortafuegos.
- -t 2: Limita el ataque a solo 2 tareas en paralelo, reduciendo aún más la presión sobre el objetivo.
