---------------------------------
- Tags: #web #magento  #reconocimiento #herramientas
-------------------
# Definición

**Magescan** es una herramienta de escaneo de seguridad diseñada específicamente para identificar vulnerabilidades en instalaciones de [[Magento]]. Permite detectar la versión de Magento, módulos y extensiones instaladas, y configuraciones inseguras.

--------------------
# Parámetros y usos

| Parámetro             | Descripción                                                                                                |
| --------------------- | ---------------------------------------------------------------------------------------------------------- |
| scan                  | Subcomando principal para iniciar un escaneo.                                                              |
| --url [URL_objetivo]  | Especifica la URL base de la instalación de Magento a escanear.                                            |
| --full                | Realiza un escaneo completo y exhaustivo, incluyendo la detección de archivos sensibles y configuraciones. |
| --proxy [host:puerto] | Configura un servidor proxy para el escaneo.                                                               |
| --verbose             | Muestra una salida más detallada durante el proceso de escaneo.                                            |
| --user-agent [agente] | Permite especificar un User-Agent personalizado para las peticiones.                                       |

#### Ejemplos Prácticos de Magescan

- Escaneo básico de vulnerabilidades de Magento:  
    ```bash
    magescan scan --url http://127.0.0.1:8080  
```
      
    Este comando realiza un escaneo de la instalación de Magento ubicada en http://127.0.0.1:8080. Magescan intentará identificar la versión de Magento y buscar vulnerabilidades comunes.
    
- Escaneo completo y detallado de Magento:  
    ```bash
    magescan scan --url http://target-magento.com --full --verbose
```  
      
    Realiza un escaneo exhaustivo de la tienda Magento en http://target-magento.com, incluyendo la detección de archivos sensibles y mostrando una salida detallada del proceso.
    

- Escaneo de Magento a través de un proxy:  
```bash
magescan scan --url http://target-magento.com --proxy http://127.0.0.1:8080 
``` 
  
Este comando escanea la instalación de Magento dirigiendo todo el tráfico a través de un proxy local, lo cual es útil para interceptar y analizar las peticiones.**