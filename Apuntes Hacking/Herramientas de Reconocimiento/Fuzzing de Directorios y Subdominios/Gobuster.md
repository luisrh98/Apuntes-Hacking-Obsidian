-----------------------
Tags: #herramientas #reconocimiento #scripting #puertos #servicios #enumeración #gobuster #fuzzing #fuzz

------------------
# Definición

[[Gobuster]] es una herramienta utilizada para la enumeración de directorios y archivos en aplicaciones web, facilitando la identificación de recursos ocultos o no documentados.

------------------
# Parámetros y usos

|   |   |
|---|---|
|Parámetro|Descripción|
|vhost|Para fuzzear subdominios|
|dir|Para fuzzear directorios|
|-u|URL objetivo|
|-w|Wordlist (diccionario de palabras)|
|-t|Número de hilos (procesos concurrentes)|
|-b|Filtra para no mostrar códigos HTTP (por ejemplo 403, 404)|
|-x|Extensiones específicas a buscar|
|-s|Mostrar solo respuestas con código HTTP 200|

#### Ejemplos Prácticos de Gobuster

- Fuzzear subdominios:  
    ```bash
    gobuster vhost -u http://example.com -w wordlist.txt -t 20 -b 403,404  
```
      
    Busca subdominios (vhost) en http://example.com (-u), usando wordlist.txt (-w), con 20 hilos (-t 20) y bloqueando respuestas 403 y 404 (-b 403,404).
    
- Fuzzear directorios con extensiones específicas:  
    ```bash
    gobuster dir -u http://example.com -w wordlist.txt -t 50 -x php,html -s 200 
``` 
      
    Busca directorios (dir) en http://example.com (-u), usando wordlist.txt (-w), con 50 hilos (-t 50), buscando archivos .php y .html (-x php,html) y mostrando solo respuestas 200 (-s 200).

---
## Gobuster con proxies

```bash
gobuster dir -u http://172.18.0.129/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt --proxy socks5://127.0.0.1:9999 -t 20 -x php,txt,html 
```