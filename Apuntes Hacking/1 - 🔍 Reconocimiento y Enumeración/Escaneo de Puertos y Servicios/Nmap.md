--------------------------
Tags: #herramientas #reconocimiento #scripting #puertos #servicios #enumeración #nmap #firewall

------------------
# Definición

[[Nmap]] (Network Mapper) es una herramienta de código abierto utilizada para escaneo de redes y detección de vulnerabilidades, empleada tanto por profesionales de la ciberseguridad como por hackers éticos para obtener información sobre dispositivos conectados a una red, servicios en ejecución y puertos abiertos.

------------------
# Parámetros y usos

|                                          |                                                                              |                                                                                                                |
| ---------------------------------------- | ---------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| Categoría                                | Parámetro                                                                    | Descripción                                                                                                    |
| Parámetros escaneo rápido general        | -p-                                                                          | escaneo a todos los puertos (EJ: -p22,23)                                                                      |
|                                          | -n                                                                           | No resolución DNS                                                                                              |
|                                          | -sS                                                                          | "Self Scan" Agiliza el reconocimiento de puertos, ya que no necesita establecer la conexión y deja menos logss |
|                                          | -sN                                                                          | Paquetes sin flags ("NO WINDOWS")                                                                              |
|                                          | -T5                                                                          | Establece la cantidad de peticiones que hace (del 1 al 5 siendo 5 el más agresivo)                             |
|                                          | -Pn                                                                          | Evita las validaciones de si esta activa la dirección ip y no esta apagada                                     |
|                                          | --min-rate 5000                                                              | Es el numero de paquetes que manda                                                                             |
| Parámetros escaneo profundo a puertos    | -sC                                                                          | Ejecuta una serie de scripts contra ese puerto                                                                 |
|                                          | -sV                                                                          | Detecta la versión del servicio que corre en ese puerto                                                        |
|                                          | -O                                                                           | Detecta el SO y su versión                                                                                     |
| Parámetros para evadir firewalls, IDS... | -f --mtu [Valor multiplo de 8]                                               | Fragmenta los paquetes.                                                                                        |
|                                          | --data-length [n]                                                            | Añade bytes extra.                                                                                             |
|                                          | -D [IP1,IP2,...]                                                             | Usa IPs señuelo (decoys) para ocultar IP real                                                                  |
|                                          | --spoof-mac [Marca EJ: Dell, VMware o una dirección MAC "00:11:22:33:44:55"] | Para cambiarte la dirección MAC en las peticiones                                                              |
| Búsqueda de Scripts                      | **locate .nse                                                                | grep [servicio o vulnerabilidad]**                                                                             |

#### Ejemplos Prácticos de NMAP

- Escaneo rápido general a todos los puertos:  
    ```bash
nmap -p- -n -sS -T5 -Pn 192.168.1.1
```  
      
    Escanea todos los puertos (-p-), sin resolución DNS (-n), con escaneo SYN rápido (-sS), velocidad agresiva (-T5) y sin ping previo (-Pn).
    
- Escaneo con mínimo de paquetes por segundo:  
    ```bash
    nmap -p- --min-rate 5000 192.168.1.1  
```
      
    Realiza un escaneo a todos los puertos (-p-) asegurando un mínimo de 5000 paquetes por segundo (--min-rate 5000).
    
- Escaneo profundo a puertos específicos con detección de servicio y scripts:  
    ```bash
    nmap -p 80,443 -sC -sV 192.168.1.1 
``` 
      
    Escanea los puertos 80 y 443 (-p 80,443), ejecuta scripts NSE básicos (-sC) y detecta la versión de los servicios (-sV).
    
- Fragmentación de paquetes para evadir firewalls:  
    ```bash
    nmap -f --mtu 24 192.168.1.1 
```
      
    Fragmenta los paquetes en unidades más pequeñas (-f --mtu 24) para intentar evadir sistemas de detección.
    
- Uso de **IPs señuelo** para ocultar la IP real:  
    ```bash
    nmap -D 192.168.1.100,192.168.1.101 192.168.1.1  
```
      
    Envía tráfico desde IPs señuelo (-D) además de la tuya para dificultar el rastreo.
    
- Cambio de dirección **MAC**:  
	```bash
	    nmap --spoof-mac Dell 192.168.1.1 
``` 
      
    Cambia la dirección MAC de origen a "Dell" (--spoof-mac Dell) para el escaneo.
    
- Búsqueda de **scripts** de Nmap para un servicio o vulnerabilidad:  
    ```bash
    locate .nse | grep ftp  
```
      
    Este comando busca todos los scripts de Nmap (.nse) en el sistema y filtra (grep) aquellos que contienen la palabra "ftp", ayudando a encontrar scripts relevantes para la enumeración de este servicio.
    

---

- ## NMAP con Proxychains (Modo rápido)

```bash
seq 1 65535 | xargs -P 500 -I {} proxychains nmap -sT -Pn -p{} -open -T5 -v -n 172.18.0.129 2>&1 | grep "tcp open"
```
