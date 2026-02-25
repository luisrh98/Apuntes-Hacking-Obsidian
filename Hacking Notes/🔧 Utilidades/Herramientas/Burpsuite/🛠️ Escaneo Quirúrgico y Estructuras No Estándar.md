
---
Tags:  #burp-scanner #burp-collaborator #targeted-scanning

---
### Índice

- [[#1. Detección Rápida con Escaneo Dirigido (Targeted Scanning)]]
    
- [[#2. Escaneo de Estructuras de Datos No Estándar (Custom Insertion Points)]]
    

---

## 1. Detección Rápida con Escaneo Dirigido (Targeted Scanning)

En auditorías de tiempo limitado (como CTFs o Pentests en entornos productivos masivos), escanear todo el árbol de directorios es un error estratégico. El **Targeted Scanning** se basa en la **heurística del auditor**: buscar inputs que interactúan con el sistema de archivos o la lógica de negocio.

### Concepto Clave: Intuición sobre el Endpoint

Un ataque de **Arbitrary File Read (AFR)** suele esconderse tras parámetros que gestionan recursos externos:

- `?file=`
    
- `?page=`
    
- `?image=`
    
- `?download=`
    

### 🛠️ Flujo de Trabajo (PoC: Lectura de `/etc/passwd`)

1. **Interceptor**: Capturamos la petición en el Proxy.
    
2. **Selección**: Enviamos la petición al **Scanner** (Click derecho -> `Scan`).
    
3. **Filtro**: En lugar de "Crawl and Audit", seleccionamos **Audit selected insertion points**.
    
4. **Validación**: Burp detecta el Path Traversal. Explotamos manualmente:
    
    ```HTTP
    GET /image?filename=../../../etc/passwd HTTP/1.1
    Host: vulnerable-site.com
    ```
    

> [!TIP]
> 
> **Pro-Tip**: Si el escáner falla, prueba con **Bypass de filtros** (URL encoding, double encoding, o Null Byte terminators `%00`).

---

## 2. Escaneo de Estructuras de Datos No Estándar (Custom Insertion Points)

Muchas aplicaciones modernas utilizan formatos de serialización o delimitadores propios para almacenar múltiples datos en un solo parámetro (ej. Cookies, tokens JWT personalizados). Los escáneres automáticos por defecto suelen fallar aquí porque no entienden el delimitador.

### 📝 Análisis del Caso de Uso (Estructura con `:` )

Si tenemos una cookie como esta: `user%3asession_token`, el escáner estándar podría intentar inyectar en toda la cadena. Necesitamos indicarle que solo nos interesa el **segundo campo**.

|**Parámetro**|**Valor Original**|**Punto de Inserción Objetivo**|
|---|---|---|
|Cookie|`wiener%3aQDzSLWym6ieM...`|`QDzSLWym6ieM0Yr8Xl4kGP...`|

### 🛠️ Configuración de Escaneo Quirúrgico

Para este ataque en Burp Suite:

1. Envía la petición a **Intruder**.
    
2. Limpia los marcadores automáticos (`Clear §`).
    
3. Selecciona manualmente la parte de la cookie después del `%3a` (codificación de `:`).
    
4. Haz click derecho sobre esa selección y elige **"Scan selected insertion point"**.