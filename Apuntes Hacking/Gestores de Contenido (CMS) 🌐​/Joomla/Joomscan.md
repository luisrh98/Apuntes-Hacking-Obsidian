--------------------------
- Tags: #web #magento  #reconocimiento #herramientas

---------------------------
# Definición

JoomScan es un escáner de vulnerabilidades diseñado específicamente para [[Joomla]]. Permite identificar la versión de Joomla, enumerar componentes, detectar archivos sensibles y buscar vulnerabilidades conocidas.

--------------------------
# Parámetros y usos

| Parámetro                                             | Descripción                                                                                                                                    |
| ----------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| -u [URL_objetivo]                                     | Especifica la URL base de la instalación de Joomla a escanear.                                                                                 |
| --enumerate [opción]                                  | Permite enumerar diferentes elementos: <br> c (componentes), <br> m (módulos), <br> p (plugins), <br> t (plantillas/temas), <br> u (usuarios). |
| --scan-plugins                                        | Escanea y detecta plugins instalados en la instancia de Joomla.                                                                                |
| --scan-themes                                         | Escanea y detecta temas/plantillas instaladas en la instancia de Joomla.                                                                       |
| --bruteforce [archivo_usuarios] [archivo_contraseñas] | Realiza un ataque de fuerza bruta contra el panel de administración de Joomla, utilizando listas de usuarios y contraseñas.                    |
| --cookie [valor_cookie]                               | Permite especificar una cookie para autenticación, útil si el escaneo requiere una sesión.                                                     |
| --proxy [host:puerto]                                 | Configura un servidor proxy para el escaneo.                                                                                                   |
| --random-agent                                        | Utiliza un agente de usuario aleatorio para cada petición.                                                                                     |

#### Ejemplos Prácticos de JoomScan

- Escaneo básico de vulnerabilidades de Joomla:  
    ```bash
    joomscan -u http://target-joomla.com 
``` 
      
    Este comando realiza un escaneo general de la instalación de Joomla en la URL especificada, intentando identificar la versión y detectar vulnerabilidades básicas.
    
- Enumerar componentes de Joomla:  
    ```bash
    joomscan -u http://target-joomla.com --enumerate c 
``` 
      
    Este comando escanea la instalación de Joomla para enumerar los componentes instalados, que a menudo son una fuente de vulnerabilidades.
    
- Escaneo de plugins y temas de Joomla:  
    ```bash
    joomscan -u http://target-joomla.com --scan-plugins --scan-themes 
``` 
      
    Este comando detecta los plugins y temas utilizados en el sitio Joomla, lo que puede ayudar a identificar extensiones con vulnerabilidades conocidas.
    
- Ataque de fuerza bruta contra el panel de administración de Joomla:  
    ```bash
    joomscan -u http://target-joomla.com --bruteforce users.txt passwords.txt
```  
      
    Realiza un ataque de fuerza bruta contra el panel de administración de Joomla, utilizando las listas de usuarios (users.txt) y contraseñas (passwords.txt) proporcionadas.
    