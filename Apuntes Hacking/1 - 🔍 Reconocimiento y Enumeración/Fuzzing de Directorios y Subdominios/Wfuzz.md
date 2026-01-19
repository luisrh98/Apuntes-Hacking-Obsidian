-----------------------
Tags: #herramientas #reconocimiento #scripting #puertos #servicios #enumeración #wfuzz #fuzzing #fuzz

------------------
# Definición

[[Wfuzz]] es una herramienta de pentesting de código abierto que permite identificar posibles vulnerabilidades mediante ataques de fuerza bruta y fuzzing a través de diccionarios predefinidos en Kali, con el objetivo de descubrir directorios ocultos, archivos, subdominios, entre otros.

------------------
# Parámetros y usos

|           |                                                                               |                                               |
| --------- | ----------------------------------------------------------------------------- | --------------------------------------------- |
| Parámetro | Descripción                                                                   | Ejemplo de Uso                                |
| -t        | Número de hilos (concurrencia)                                                | -t 20                                         |
| -u        | URL objetivo con el lugar a fuzzear indicado con FUZZ                         | -u http://example.com/FUZZ                    |
| -w        | Wordlist para fuzzing                                                         | -w wordlist.txt                               |
| --hc      | Oculta respuestas con códigos HTTP indicados (hide codes).                    | --hc 404,403                                  |
| --sc      | Muestra solo respuestas con códigos HTTP indicados (show codes).              | --sc 200,301                                  |
| --hw      | Oculta respuestas con número de palabras en contenido indicadas (hide words). | --hw 100 (oculta respuestas con 100 palabras) |
| --sw      | Muestra solo respuestas con número de palabras indicadas (show words).        | --sw 50 (muestra respuestas con 50 palabras)  |
| --hh      | Oculta respuestas con número de caracteres indicados (hide length).           | --hh 500                                      |
| --sh      | Muestra solo respuestas con número de caracteres indicados (show length).     | --sh 1024                                     |
| --sl      | Muestra solo respuestas con tamaño específico en bytes (show length exacto).  | --sl 300                                      |

#### Ejemplos Prácticos de Wfuzz

- Fuzzing básico con ocultación de código 404:  
    ```bash
    wfuzz -t 20 -u http://example.com/FUZZ -w wordlist.txt --hc 404 
``` 
      
    Realiza un fuzzing (-u con FUZZ) usando wordlist.txt (-w), con 20 hilos (-t 20), y ocultando las respuestas con código 404 (--hc 404).
    
- Mostrar solo respuestas exitosas (código 200):  
    ```bash
    wfuzz -t 10 -u http://example.com/FUZZ -w common.txt --sc 200
```  
      
    Fuzzea con 10 hilos (-t 10), usando common.txt (-w), y mostrando solo las respuestas con código HTTP 200 (--sc 200).
    
- Ocultar respuestas por número de palabras:  
    ```bash
    wfuzz -u http://example.com/FUZZ -w directory-list-2.3-medium.txt --hw 120  
```
      
    Fuzzea directorios ocultando las respuestas que tienen exactamente 120 palabras (--hw 120), útil para filtrar páginas de error estándar.
    

**