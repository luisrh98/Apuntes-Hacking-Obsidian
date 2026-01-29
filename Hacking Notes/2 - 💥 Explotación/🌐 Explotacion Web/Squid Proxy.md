
---
Tags: #proxy #http #enumeración 

---
## 📌 Definición

**Squid** es un **proxy caching** para redes, usado para:

- Acelerar el acceso a la web mediante **caché**.
    
- Filtrar y controlar el acceso a Internet.
    
- Registrar tráfico HTTP/HTTPS para auditoría o seguridad.
    

Un servidor Squid mal configurado puede permitir a un atacante:

- Acceder a **Internet de manera anónima** a través del proxy.
    
- Realizar **escaneos internos** de la red interna.
    
- Exploits de **Open Proxy** o bypass de filtros de contenido.
    

---

## 🔍 Detección de Squid

1. **Escaneo de puertos comunes**:
    
    - Squid suele escuchar en **3128, 8080, 8000, 80**.
        
    
    `nmap -p 3128,8080,8000,80 target.com -sV`
    
2. **Prueba de proxy abierto**:
    
    `curl -x http://target.com:3128 http://example.com`
    
    - Si devuelve el contenido → proxy abierto (vulnerable).
        
3. **Identificación mediante cabeceras HTTP**:
    
    `curl -I -x http://target.com:3128 http://example.com`
    
    Busca cabeceras como:
    
    `Via: 1.1 squid Server: squid/4.13`
    

---

## 💥 Vulnerabilidades y ataques comunes

|Ataque|Descripción|Ejemplo|
|---|---|---|
|**Open Proxy / Uso no autorizado**|Cualquier atacante puede usar Squid para navegar por la web y ocultar su IP.|`curl -x http://openproxy:3128 http://target.com`|
|**Cache Poisoning**|Manipular caché para inyectar contenido malicioso o falso.|Inyección de respuesta HTTP en caché|
|**Bypass de filtrado**|Usar proxy para saltarse filtros de contenido o firewalls.|Acceder a sitios bloqueados internamente|
|**Escaneo interno**|Redirigir peticiones hacia la red interna del servidor.|`curl -x http://proxy:3128 http://192.168.0.1`|
|**Exposición de logs**|Acceso a logs si el servidor no restringe `/var/log/squid/`|Descarga de historial de navegación|

---

## 🛠 Herramientas para pruebas

### 🔹 1. **curl**

- Test de proxy abierto:
    

`curl -x http://target:3128 http://example.com`

- Probar bypass de filtros internos:
    

`curl -x http://target:3128 http://intranet.local`

### 🔹 2. **proxychains**

- Forzar tráfico de aplicaciones a través de Squid:
    

`proxychains firefox`

### 🔹 3. **nmap NSE scripts**

- Escaneo específico Squid:
    

`nmap -p 3128 --script http-open-proxy target.com`

---

## 🧨 Ejemplo práctico

1. Detectamos Squid en puerto 3128:
    

`nmap -p 3128 target.com -sV`

Resultado:

`3128/tcp open  http-proxy Squid http proxy 4.13`

2. Probamos si es un **proxy abierto**:
    

`curl -x http://target.com:3128 http://example.com`

- Si devuelve contenido → **Open Proxy confirmado**.
    

3. Usamos el proxy para escanear la red interna:
    

`curl -x http://target.com:3128 http://192.168.1.100`

---

## ⚠️ Bypass y técnicas avanzadas

- **Rotación de headers**: Algunos filtros bloquean User-Agent o Referer.
    
- **Túnel HTTPS**: Para saltarse inspección de tráfico interno.
    
- **Uso de SOCKS → HTTP**: Algunos proxies permiten convertir tráfico SOCKS a HTTP para evadir restricciones.
    

---

## 🛡 Mitigaciones

- Restringir acceso solo a IPs internas o autenticadas.
    
- Deshabilitar **proxy abierto** (`http_access allow localnet`).
    
- Limitar métodos HTTP y bloquear PUT/DELETE si no son necesarios.
    
- Monitorear logs y detectar patrones anómalos.

---
# 📝 Squid Proxy Cheat Sheet – Ataques y Pruebas

| Vector / Ataque                | Descripción                                                   | Ejemplo / Payload                                                            | Observaciones                                         |
| ------------------------------ | ------------------------------------------------------------- | ---------------------------------------------------------------------------- | ----------------------------------------------------- |
| **Open Proxy / Proxy Abierto** | Uso no autorizado del proxy para navegar o anonimizar tráfico | `curl -x http://target:3128 http://example.com`                              | Si devuelve contenido → proxy abierto                 |
| **Escaneo de red interna**     | Acceso a hosts internos a través del proxy                    | `curl -x http://target:3128 http://192.168.1.100`                            | Útil para descubrir servicios internos                |
| **Cache Poisoning**            | Inyección de contenido malicioso en caché                     | Modificar headers HTTP o inyectar payload en respuesta GET                   | Afecta a otros usuarios que usen el proxy             |
| **Bypass de filtrado**         | Saltarse filtros de contenidos internos o firewalls           | `curl -x http://target:3128 http://sitio-bloqueado.local`                    | Funciona con filtros basados en IP o URL              |
| **Rotación de Headers**        | Evadir filtros que bloquean ciertos User-Agent o Referer      | `curl -x http://target:3128 -H "User-Agent: Mozilla/5.0" http://example.com` | Útil combinando con proxy abierto                     |
| **Túnel HTTPS**                | Saltar inspección de tráfico interno mediante CONNECT         | `curl -x http://target:3128 -L https://intranet.local`                       | Solo funciona si CONNECT está habilitado              |
| **Logs Exposure**              | Acceso no autorizado a archivos de logs del proxy             | `curl -x http://target:3128 http://target.com/var/log/squid/access.log`      | Solo si el servidor tiene ruta accesible públicamente |
| **HTTP Method Bypass**         | Probar métodos PUT, DELETE para subir o borrar archivos       | `curl -X PUT -x http://target:3128 http://target.com/test.txt -d "payload"`  | Solo si proxy permite métodos avanzados               |
| **SOCKS → HTTP Tunneling**     | Convertir tráfico SOCKS a HTTP para evadir filtros            | `proxychains nmap -sT target.com`                                            | Útil para escaneos internos y anonimización           |
| **Open Proxy Detection Nmap**  | Script NSE para comprobar proxies abiertos                    | `nmap -p 3128 --script http-open-proxy target.com`                           | Devuelve resultado True/False                         |