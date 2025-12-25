------------
# Definción
>[[Google Dorks ]]es una técnica que utiliza comandos o atributos específicos en el motor de búsqueda de Google para encontrar información más específica y detallada en las búsquedas. Esta técnica es legal y se emplea para realizar búsquedas avanzadas, permitiendo a los usuarios acceder a información que no se encuentra fácilmente en la web.

---------------------
# Parámetros y Usos
Google Dorking utiliza operadores avanzados para encontrar información sensible oculta en los resultados de búsqueda de Google.

|   |   |   |   |
|---|---|---|---|
|Nº|Google Dork|Ejemplo|Descripción|
|1|intitle:"index of" config|intitle:"index of" config|Buscar archivos de configuración expuestos.|
|2|filetype:log password|filetype:log password|Buscar archivos log con contraseñas.|
|3|inurl:login|inurl:login|Buscar páginas con formularios de login.|
|4|filetype:sql intext:password|filetype:sql intext:password|Buscar bases de datos SQL expuestas con contraseñas.|
|5|filetype:xls confidential|filetype:xls confidential|Buscar documentos Excel confidenciales.|
|6|inurl:"view/view.shtml"|inurl:"view/view.shtml"|Buscar cámaras IP expuestas.|
|7|filetype:bak|filetype:bak|Buscar archivos de backup.|
|8|intitle:"index of" /admin|intitle:"index of" /admin|Buscar directorios de administración expuestos.|
|9|inurl:phpmyadmin|inurl:phpmyadmin|Buscar paneles phpMyAdmin.|
|10|filetype:pdf "confidencial"|filetype:pdf "confidencial"|Buscar PDFs con texto confidencial.|
|11|filetype:env|filetype:env|Buscar archivos de entorno con credenciales en texto plano.|
|12|filetype:doc username|filetype:doc username|Buscar documentos con nombres de usuario.|
|13|apikey filetype:txt|apikey filetype:txt|Buscar claves API en archivos .txt.|
|14|inurl:index.php?id=|inurl:index.php?id=|Buscar URLs con posible vulnerabilidad de SQL Injection.|
|15|intext:"Warning: mysql_connect()"|intext:"Warning: mysql_connect()"|Buscar errores de servidor que revelan información.|
