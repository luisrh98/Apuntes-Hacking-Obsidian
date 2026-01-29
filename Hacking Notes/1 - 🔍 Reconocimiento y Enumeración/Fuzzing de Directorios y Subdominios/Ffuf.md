-----------------------
Tags: #herramientas #reconocimiento #scripting #puertos #servicios #enumeración #ffuf #fuzzing #fuzz

------------------
# Definición

[[Ffuf]] es un web fuzzer que se utiliza para identificar rutas activas en un sitio web mediante la prueba de diferentes combinaciones de direcciones URL de un dominio. Este tipo de herramientas es fundamental en el campo de la ciberseguridad para encontrar posibles vulnerabilidades en aplicaciones web. Ffuf utiliza listas de palabras que el usuario puede modificar, diseñar o conseguir por su propia cuenta, con el fin de identificar qué rutas están activas y cuáles no.

En cuanto a las tecnologías que utiliza, Ffuf está desarrollado en el lenguaje de programación Go, lo que le permite funcionar de manera más rápida en comparación con otras herramientas como Dirb, que fue escrito en Python

------------------
# Parámetros y usos

|   |   |
|---|---|
|Parámetro|Descripción|
|-u|URL objetivo con el indicador FUZZ|
|-w|Wordlist (diccionario de palabras)|
|-t|Número de hilos (concurrencia)|
|-H|Añadir cabecera HTTP personalizada|
|-X|Método HTTP a usar (GET, POST, PUT, etc.)|
|-mc|Códigos de estado HTTP a mostrar (match codes)|
|-fc|Códigos de estado HTTP a filtrar (filter codes)|
|-ac|Auto-calibración de filtros (detecta respuestas comunes de error)|
|-v|Modo verboso (muestra más detalles)|

#### Ejemplos Prácticos de Ffuf

- Fuzzing básico con ocultación de código 404:  
    ```bash
    ffuf -t 20 -u http://example.com/FUZZ -w wordlist.txt --fc 404  
```
      
    Realiza un fuzzing (-u con FUZZ) usando wordlist.txt (-w), con 20 hilos (-t 20), y filtrando las respuestas con código 404 (--fc 404).
    
- Fuzzing de cabeceras HTTP:  
    ```bash
    ffuf -u http://example.com -H "X-Forwarded-For: FUZZ" -w ips.txt -mc 200
```  
      
    Fuzzea la cabecera X-Forwarded-For (-H) con IPs de ips.txt (-w), mostrando solo respuestas 200 (-mc 200).
    

- Fuzzing de parámetros POST con auto-calibración:  
```bash
ffuf -X POST -u http://example.com/login -d "username=FUZZ&password=password" -w users.txt -ac
```  
  
Realiza un fuzzing POST (-X POST) al formulario de login (-u), probando nombres de usuario (-d "username=FUZZ...") desde users.txt (-w), y usando auto-calibración (-ac) para ignorar respuestas de error comunes.**

[!]**Importante**
	Fuzz de subdominios:
	
```bash
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt \
-u http://soulmate.htb/ \
-H "Host: FUZZ.soulmate.htb" \
-mc 200,403,301,302 \
-fs 7643
```
