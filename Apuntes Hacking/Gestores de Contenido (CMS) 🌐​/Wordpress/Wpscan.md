------------------------
- Tags: #web #wordpress #reconocimiento #herramientas
-------------------------
>**[[Wpscan]]** es un escáner de caja negra de WordPress, diseñado para encontrar vulnerabilidades de seguridad en instalaciones de [[Wordpress]].
---------------------------
# Parámetros y usos

| Parámetro                         | Descripción                                                                                                                                                                   |
| --------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| --url [URL_objetivo]              | Especifica la URL de la instalación de WordPress a escanear.                                                                                                                  |
| --enumerate [opción]              | Permite enumerar diferentes elementos: <br> vp (plugins vulnerables), <br> vt (temas vulnerables), <br> u (usuarios), <br> d (directorios de plugins/temas por defecto), etc. |
| --api-token [token]               | Utiliza un token de API de WPScan para acceder a la base de datos de vulnerabilidades de WPScan, lo que permite resultados más completos y actualizados.                      |
| --passwords [archivo_contraseñas] | Realiza un ataque de fuerza bruta contra usuarios de WordPress utilizando un diccionario de contraseñas.                                                                      |
| --usernames [archivo_usuarios]    | Especifica un archivo con una lista de nombres de usuario para ataques de fuerza bruta o enumeración.                                                                         |
| --random-agent                    | Utiliza un agente de usuario aleatorio para cada petición, ayudando a evadir algunas detecciones básicas.                                                                     |
| --detection-mode [modo]           | Ajusta el modo de detección: aggressive (más ruidoso, pero más exhaustivo) o passive (menos ruidoso).                                                                         |

#### Ejemplos Prácticos de Wpscan

- Escaneo básico de vulnerabilidades de Wordpress:  

```bash
    wpscan --url http://target-wordpress.com --api-token YOUR_API_TOKEN  
```
>Este comando realiza un escaneo completo de la instalación de WordPress en la URL especificada, utilizando el token de API para acceder a la base de datos de vulnerabilidades y obtener resultados actualizados sobre el core de WordPress, plugins y temas.
    
- Enumerar usuarios de WordPress:  

    ```bash
    wpscan --url http://target-wordpress.com --enumerate u --api-token YOUR_API_TOKEN 
```
     
    Este comando intenta enumerar los nombres de usuario válidos en la instalación de WordPress, lo cual es útil para preparar ataques de fuerza bruta.
    
- Ataque de fuerza bruta contra usuarios de WordPress:  

    ```bash
    wpscan --url http://target-wordpress.com --usernames users.txt --passwords passwords.txt --api-token YOUR_API_TOKEN  
	```      
    
    Realiza un ataque de fuerza bruta utilizando una lista de usuarios (users.txt) y un diccionario de contraseñas (passwords.txt) contra el panel de administración de WordPress.
    
- Detección de vulnerabilidades en xmlrpc.php: 

    WPScan detecta automáticamente si xmlrpc.php está habilitado y si es vulnerable a ataques de fuerza bruta o pingback. No se necesita un parámetro específico para esto, ya que forma parte del escaneo general.


- **WPScan API Token**:  

    El WPScan API Token es una clave personal que se obtiene registrándose en el sitio web de WPScan. Permite que la herramienta acceda a la base de datos de vulnerabilidades de WordPress mantenida por WPScan. Sin este token, las comprobaciones de vulnerabilidades de plugins y temas serán limitadas o incompletas. Su uso es altamente recomendado para obtener resultados de escaneo precisos y actualizados.
    
- **¿Qué es xmlrpc.php y a qué es vulnerable?**  

    [[xmlrpc.php]] es un archivo en WordPress que permite la comunicación entre WordPress y otros sistemas utilizando el protocolo XML-RPC. Históricamente, se usaba para funciones como publicar posts desde aplicaciones de escritorio o realizar pings. Aunque su uso ha disminuido con la API REST de WordPress, sigue presente en muchas instalaciones.  
    Vulnerabilidades: xmlrpc.php es conocido por ser vulnerable a:
    

- Ataques de fuerza bruta: Permite un número ilimitado de intentos de inicio de sesión, lo que facilita que los atacantes prueben muchas combinaciones de usuario y contraseña.
    
- Ataques de denegación de servicio (DoS) / DDoS: Puede ser explotado para realizar ataques de "pingback" o "trackback" amplificados, donde un atacante envía una petición a xmlrpc.php que a su vez provoca que el servidor WordPress envíe múltiples peticiones a un objetivo, saturándolo.
-------------------
# Referencias

Enlace de la herramienta a Github: [Enlace](https://github.com/wpscanteam/wpscan)
