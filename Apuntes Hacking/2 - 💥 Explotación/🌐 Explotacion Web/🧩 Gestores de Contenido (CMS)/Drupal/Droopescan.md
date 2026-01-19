--------------------------
- Tags: #web #drupal #wordpress #joomla #moodle #reconocimiento #herramientas
---------------------------
# Definición

**[[Droopescan]]** es una herramienta de escaneo de seguridad que se especializa en la detección de versiones y vulnerabilidades en CMS como Drupal, WordPress, Joomla, SilverStripe y Moodle. Es eficaz para identificar la versión del CMS, los plugins/módulos instalados y las vulnerabilidades asociadas.

--------------------------
# Parámetros y usos

| Parámetro               | Descripción                                                                          |
| ----------------------- | ------------------------------------------------------------------------------------ |
| scan drupal             | Subcomando para especificar que se va a escanear una instalación de Drupal.          |
| --url [URL_objetivo]    | Especifica la URL base de la instalación de Drupal a escanear.                       |
| --force-scan            | Fuerza el escaneo incluso si Droopescan detecta que no es una instalación de Drupal. |
| --verbose               | Muestra una salida más detallada durante el proceso de escaneo.                      |
| --detection-mode [modo] | Ajusta el modo de detección: aggressive (más ruidoso) o passive (menos ruidoso).     |

#### Ejemplos Prácticos de Droopescan

- Escaneo básico de vulnerabilidades de Drupal:  
    ```bash
    droopescan scan drupal --url http://127.0.0.1:8080
```  
      
    Este comando realiza un escaneo de la instalación de Drupal ubicada en http://127.0.0.1:8080. Droopescan intentará identificar la versión de Drupal, los módulos y temas instalados, y cualquier vulnerabilidad conocida asociada a ellos.
    
- Escaneo verboso de Drupal:  
    ```bash
    droopescan scan drupal --url http://example-drupal.com --verbose 
``` 
      
    Realiza un escaneo de Drupal en http://example-drupal.com y muestra información detallada sobre el proceso de escaneo, incluyendo las comprobaciones que se están realizando y los resultados intermedios.
    