
---
Tags: #DNS #network #redes 

---
## 📌 Definición

Una **transferencia de zona DNS** es el proceso por el cual un servidor DNS primario replica sus registros a un servidor DNS secundario.  
Este proceso se realiza usando el protocolo **AXFR** (Asynchronous Full Zone Transfer).  
Si está **mal configurado** y sin restricciones, un atacante puede solicitar una copia **completa** de todos los registros DNS del dominio, exponiendo información crítica como:

- Subdominios internos
    
- Servidores ocultos
    
- IPs internas
    
- Información sensible sobre la infraestructura
    

---

## 🧠 Funcionamiento lógico

1. El **DNS primario** recibe solicitudes AXFR de servidores secundarios autorizados.
    
2. Si la configuración es laxa, **cualquier cliente** puede solicitar la transferencia.
    
3. El servidor devuelve **todos los registros** del dominio (A, AAAA, MX, TXT, CNAME, NS, SRV, etc.).
    
4. El atacante obtiene **un mapa completo** de la infraestructura.
    

---

## 🔍 Cómo detectar si es vulnerable

- Usar herramientas como `dig`, `host`, `nslookup` contra **cada servidor autoritativo** del dominio.
    
- Si devuelve los registros completos, es vulnerable.
    
- La vulnerabilidad se encuentra **por servidor DNS**, no por dominio.
    

---

## 💥 Vectores de ataque comunes

| Comando                                 | Descripción                                   | Ejemplo                                                     |
| --------------------------------------- | --------------------------------------------- | ----------------------------------------------------------- |
| `dig axfr dominio.com @ns1.dominio.com` | Solicita transferencia de zona                | `dig axfr ejemplo.com @ns1.ejemplo.com`                     |
| `host -l dominio.com ns1.dominio.com`   | Transferencia usando `host`                   | `host -l ejemplo.com ns1.ejemplo.com`                       |
| `nslookup` + `ls`                       | Transferencia interactiva                     | `nslookup` → `server ns1.ejemplo.com` → `ls -d ejemplo.com` |
| `dnsrecon`                              | Enumera subdominios y prueba AXFR             | `dnsrecon -d ejemplo.com -t axfr`                           |
| `fierce`                                | Fuerza transferencia y escaneo de subdominios | `fierce --domain ejemplo.com --dns-servers ns1.ejemplo.com` |

---

## 🛠 Herramientas y uso

|Herramienta|Uso|Ejemplo|
|---|---|---|
|**dig**|Cliente DNS flexible|`dig axfr ejemplo.com @ns1.ejemplo.com`|
|**host**|Cliente DNS simple|`host -l ejemplo.com ns1.ejemplo.com`|
|**nslookup**|Interactivo|`nslookup` → `server ns1.ejemplo.com` → `ls -d ejemplo.com`|
|**dnsrecon**|Escaneo avanzado|`dnsrecon -d ejemplo.com -t axfr`|
|**fierce**|Fuerza y enumera|`fierce --domain ejemplo.com`|
|**dnsenum**|Enumera y prueba AXFR|`dnsenum ejemplo.com`|

---

## 🚀 Ejemplo práctico

- Paso 1: Obtener servidores autoritativos

```bash
dig ns ejemplo.com +short
```

  # Salida: 
```bash
ns1.ejemplo.com. ns2.ejemplo.com
```

- Paso 2: Probar transferencia en cada uno:
```bash
dig axfr ejemplo.com @ns1.ejemplo.com
```

**Salida típica vulnerable:**

```python
; <<>> DiG 9.16.1-Ubuntu <<>> axfr ejemplo.com @ns1.ejemplo.com ... mail.ejemplo.com.      IN A     192.168.1.5 dev.ejemplo.com.       IN A     10.0.0.5 internal-db.ejemplo.com IN A    172.16.5.3 ...
```

---

## 🔐 Mitigaciones

- Restringir AXFR solo a **servidores secundarios autorizados**.
    
- Usar **TSIG (Transaction Signature)** para autenticar transferencias.
    
- Deshabilitar AXFR si no es necesario.
    

---

## ⚠️ Impacto de un AXFR exitoso

- **Reconocimiento total** de subdominios internos y externos.
    
- Exposición de **IPs internas** y posibles **puertas traseras**.
    
- Facilita ataques posteriores como:
    
    - **Spear phishing**
        
    - **SSRF**
        
    - **Acceso a entornos internos**
        
    - **Descubrimiento de paneles de administración**
    