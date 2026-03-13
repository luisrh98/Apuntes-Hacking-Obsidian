
---
Tags: #BusinessLogicVulnerabilities #ParameterTampering #IntegerOverflow #StateMachineBypass #EncryptionOracle #EmailParsingInconsistency #UTF7Exploitation #DataTruncation #WorkflowBypass #LogicExploitation #WebHacking #BurpSuiteMacros #BrokenLogic

---

Las vulnerabilidades de lógica de negocio son fallos en el diseño y la implementación de una aplicación que permiten a un atacante inducir un comportamiento inesperado. No se trata de un error de sintaxis en el código, sino de una **falla en el razonamiento de las reglas de negocio**.

### 📑 Índice

- [[#1. Confianza Excesiva en el Lado del Cliente]]
    
- [[#2. Vulnerabilidad Lógica de Alto Nivel (Cantidades Negativas)]]
    
- [[#3. Controles de Seguridad Inconsistentes]]
    
- [[#4. Reglas de Negocio Mal Aplicadas (Abuso de Cupones)]]
	
- [[#5. Fallo Lógico de Bajo Nivel (Integer Overflow)]]
    
- [[#6. Gestión Inconsistente de Entradas Atípicas (Truncado de Emails)]]
    
- [[#7. Aislamiento Débil en Endpoint de Doble Uso]]
    
- [[#8. Validación Insuficiente del Flujo de Trabajo]]
	
- [[#9. Fallo Lógico para Generar Dinero Infinito]]
    
- [[#10. Bypass por Máquina de Estados Rota]]
    
- [[#11. Bypass por Oráculo de Cifrado (AES Padding & Impersonation)]]
    [[#🔍 El Escenario (AES & Padding)]]
    [[#🛠️ Explotación paso a paso]]
    
- [[#12. Bypass por Parsing Inconsistente de Email (UTF-7 Deep Dive)]]
	[[#🔍 Concepto Técnico]]
	[[#🛠️ Anatomía del Payload (UTF-7)]]
	
- [[#Checklist Auditoría de Lógica de Negocio]]
	[[#1. Fase de Reconocimiento de Flujo]]
	[[#2. Manipulación de Parámetros de Confianza]]
	[[#3. Pruebas de Límites y Tipos de Datos]]
	[[#4. Integridad del Flujo de Trabajo (Workflows)]]
	[[#5. Identidad y Parsing Avanzado]]
	[[#📊 Resumen de Severidad por Técnica]]
---

## 1. Confianza Excesiva en el Lado del Cliente

Esta técnica se basa en la suposición errónea de que los datos enviados desde el navegador son inalterables o han sido validados previamente. El servidor "confía" en parámetros críticos que el usuario puede manipular fácilmente mediante un proxy (Burp Suite).

### 🔍 Descubrimiento y Explotación

- **Descubrimiento:** Interceptar peticiones `POST` o `PUT` al añadir productos al carrito o procesar pagos. Buscar parámetros como `price`, `discount` o `tax`.
    
- **Explotación:** Modificar el valor del precio antes de que la petición llegue al servidor.
    

### 🛠️ PoC (Proof of Concept)

En este ejemplo, el atacante modifica el parámetro `price` directamente en la solicitud de compra:

```HTTP
POST /cart HTTP/2
Host: target-app.net
Cookie: session=uYRxItdLCqICQ4GM757vLff1UTbNw5x1

productId=1&redir=PRODUCT&quantity=1&price=1
```

> [!IMPORTANT]
> 
> **Impacto:** Compra de productos de alto valor por una fracción de su precio real. El servidor no vuelve a consultar la base de datos para verificar el precio del `productId=1`.

---

## 2. Vulnerabilidad Lógica de Alto Nivel

A diferencia de la anterior, aquí el atacante no modifica el precio, sino que manipula la **lógica matemática** de la aplicación enviando valores inesperados (como números negativos) que el backend procesa sin validar rangos lógicos.

### 🔍 Descubrimiento y Explotación

- **Descubrimiento:** Probar valores fuera de los límites normales (0, números negativos, letras, caracteres especiales) en campos de cantidad o saldo.
    
- **Explotación:** Añadir un producto legítimo y luego añadir una cantidad negativa de otro producto para reducir el total de la factura.
    

### 🛠️ PoC

Si el carrito permite cantidades negativas, el total se calcula como: $Total = (P1 \times Q1) + (P2 \times -Q2)$.

```HTTP
POST /cart HTTP/2
Host: target-app.net

productId=1&redir=PRODUCT&quantity=-1
```

|**Producto**|**Precio**|**Cantidad**|**Subtotal**|
|---|---|---|---|
|MacBook Pro|$2000|1|$2000|
|Cable USB|$10|-199|-$1990|
|**TOTAL**|||**$10**|

---

## 3. Controles de Seguridad Inconsistentes

Ocurre cuando la aplicación aplica reglas de seguridad de forma parcial o asume que ciertos cambios de estado son inofensivos, permitiendo una **Escalada de Privilegios**.

### 🔍 Descubrimiento y Explotación

- **Descubrimiento:** Analizar funciones de actualización de perfil o registro. ¿Qué pasa si cambio mi correo a uno con el dominio interno de la empresa?
    
- **Explotación:** Cambiar el email personal por uno de tipo `admin@empresa.com`. Si la aplicación solo verifica el dominio para otorgar permisos de administrador, habremos comprometido el panel de control.
    

### 💡 Ejemplo de Flujo

1. El usuario se registra como `user@normal.com`.
    
2. Accede a "Editar Perfil".
    
3. Cambia el email a `victim@admin-domain.com`.
    
4. La lógica de la app detecta el dominio y habilita el botón de "Admin Panel" en la sesión actual.
    

---

## 4. Reglas de Negocio Mal Aplicadas

Este fallo reside en cómo el servidor gestiona el **estado** de una transacción o proceso (como la aplicación de descuentos). Si el sistema no lleva un seguimiento correcto de la secuencia, se pueden evadir restricciones.

### 🔍 Descubrimiento y Explotación

- **Descubrimiento:** Intentar aplicar múltiples cupones o el mismo cupón varias veces. Observar si el orden de los factores altera el producto.
    
- **Explotación:** Jugar con la secuencia de entrada. Si la app solo bloquea la repetición consecutiva de un código, alternar entre dos códigos para "limpiar" el estado de validación.
    

### 🛠️ Caso de Uso: Bypass de Cupones

Supongamos dos cupones: `NEWCUST5` y `NEWCUST30`.

1. Aplicar `NEWCUST5` -> ✅ Éxito.
    
2. Aplicar `NEWCUST5` de nuevo -> ❌ Error (Ya usado).
    
3. Aplicar `NEWCUST30` -> ✅ Éxito.
    
4. Aplicar `NEWCUST5` de nuevo -> ✅ Éxito (El sistema solo verificó que el _último_ no fuera igual).

---
## 5. Fallo Lógico de Bajo Nivel (Integer Overflow)

Esta vulnerabilidad ocurre cuando la aplicación realiza cálculos matemáticos sobre tipos de datos con límites fijos (como un entero de 32 bits) sin validar si el resultado excede dicho límite.

### 🔍 Descubrimiento y Explotación

- **Concepto:** En muchos lenguajes, un entero de 32 bits tiene un valor máximo de $2,147,483,647$. Si sumas 1 a ese valor, el sistema sufre un **"wrap around"** y el valor se convierte en $-2,147,483,648$.
    
- **Explotación:** Añadir cantidades masivas de productos al carrito hasta que el precio total "dé la vuelta" y se vuelva negativo. Luego, ajustar con otros productos para que el total sea un número positivo pequeño.
    

### 🛠️ Escenario de Explotación

Si el carrito tiene un límite de precio, el atacante busca el punto de quiebre:

1. Añadir 1,000,000 de unidades de un producto caro.
    
2. Repetir hasta que el `Total Price` en el backend supere los $2^{31}-1$.
    
3. El precio se vuelve negativo (ej. -$500.00).
    
4. Añadir productos normales hasta que el balance sea $0.01 o similar.
    

---

## 6. Gestión Inconsistente de Entradas Atípicas

Ocurre cuando diferentes componentes del sistema (frontend, backend, base de datos, servidor de correo) procesan la misma entrada de forma distinta. El caso más común es el **Truncado de Datos**.

### 🔍 Descubrimiento y Explotación

- **Descubrimiento:** Identificar campos con límites de caracteres (ej. 255 caracteres en la DB).
    
- **Técnica:** Si el sistema de registro acepta emails largos pero la base de datos solo guarda los primeros 255 caracteres, podemos registrar una cuenta que "parezca" ser de un dominio interno tras el truncado.
    

### 🛠️ PoC: Registro con Truncado (Admin Impersonation)

Queremos una cuenta en `@dontwannacry.com` (dominio admin). El sistema nos permite registrarnos pero envía un correo de confirmación. Usamos un payload que sea:

`[255 chars de padding] + @dontwannacry.com + .nuestro-dominio.com`.


```
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA@dontwannacry.com.exploit-0a1000f00497ba5e84510e96011000f3.exploit-server.net
```

1. **Servidor de Correo:** Ve el dominio completo y envía el enlace de activación a **nuestro** servidor de exploit.
    
2. **Base de Datos:** Al guardar el registro, corta a los 255 caracteres, guardando solo: `...AAAAA@dontwannacry.com`.
    
3. **Resultado:** Al activar la cuenta, el sistema nos reconoce como un usuario del dominio administrativo.
    

---

## 7. Aislamiento Débil en Endpoint de Doble Uso

Muchas aplicaciones reutilizan el mismo endpoint para funciones de usuario normal y funciones administrativas, diferenciándolas solo por los parámetros enviados.

### 🔍 Descubrimiento y Explotación

- **Punto Débil:** Un endpoint de "Cambio de Contraseña" que requiere la `current-password` para seguridad, pero que también permite pasar un `username`.
    
- **Explotación:** ¿Qué pasa si el desarrollador programó que, si no se envía `current-password`, el sistema asuma que es una acción administrativa y no la pida?
    

### 🛠️ PoC: Cambio de Password de Admin

Petición original (Usuario Wiener):

```HTTP
POST /my-account/change-password HTTP/2
...
csrf=MxUJ...&username=wiener&current-password=peter&new-password-1=123&new-password-2=123
```

Ataque (Eliminando validación y cambiando objetivo):

```HTTP
POST /my-account/change-password HTTP/2
...
csrf=MxUJ...&username=administrator&new-password-1=123&new-password-2=123
```

> [!WARNING]
> 
> Al omitir `current-password`, la lógica del servidor puede fallar y procesar el cambio de contraseña para _cualquier_ `username` proporcionado.

---

## 8. Validación Insuficiente del Flujo de Trabajo

Las aplicaciones web suelen guiar al usuario a través de una secuencia de pasos (Step 1 -> Step 2 -> Step 3). Esta vulnerabilidad ocurre cuando el servidor asume que, si llegas al "Paso 3", es porque completaste con éxito los anteriores.

### 🔍 Descubrimiento y Explotación

- **Técnica:** Forzar la navegación (**Forceful Browsing**) a la página final de un proceso sin pasar por la pasarela de pago o validación.
    
- **Explotación:** Identificar el endpoint de confirmación de orden y llamarlo directamente tras añadir productos al carrito.
    

### 🛠️ PoC: Compra Gratuita

En lugar de seguir el flujo de pago, llamamos directamente al trigger de éxito:

```HTTP
GET /cart/order-confirmation?order-confrimed=true HTTP/2
Host: target-app.net
Cookie: session=yBWoEdIf3omkuX6xfJDOwMmMQX3GYfzr
```

Si el backend no verifica en el lado del servidor que el pago asociado a esa `session` fue completado, la orden se procesará como "Pagada".

---
## 9. Fallo Lógico para Generar Dinero Infinito

Este fallo ocurre cuando la aplicación permite que un proceso de "recompensa" o "reembolso" sea más rentable que el costo de adquisición, y no limita la recursividad del proceso.

### 🔍 Descubrimiento y Explotación

- **Lógica:** Comprar un producto (Gift Card) con descuento, pero redimirlo por su valor nominal (sin descuento).
    
- **Explotación:** Automatizar el ciclo mediante **Burp Suite Macros**.
    
    1. Aplicar Cupón (30% off) -> Comprar Gift Card de 10€ por 7€.
        
    2. Recibir código de Gift Card.
        
    3. Redimir Gift Card -> Saldo aumenta 10€.
        
    4. **Profit:** +3€ netos por ciclo.
        

> [!TIP]
> 
> **Pro Tip:** Usa el "Project Options" > "Sessions" en Burp Suite para crear una regla que ejecute estas peticiones secuencialmente de forma automática.

---

## 10. Bypass por Máquina de Estados Rota

Una "Máquina de Estados" controla en qué fase de un proceso se encuentra el usuario (ej: _Logged In_ -> _Selecting Role_ -> _Dashboard_). Si el servidor asume que el usuario pasará por todos los pasos sin forzar la transición, el estado puede "romperse".

### 🔍 Descubrimiento y Explotación

- **Escenario:** Tras el login exitoso, la app te redirige a `/role-selector`.
    
- **Fallo:** Si el atacante **dropea** (intercepta y descarta) la petición al selector de roles y navega directamente a `/admin`, la aplicación podría asignar un rol por defecto (a veces `admin`) al no haber recibido una selección válida.
    

### 🛠️ PoC: Salto de Selección de Rol

1. `POST /login` -> Credenciales válidas.
    
2. Servidor responde con `302 Redirect` a `/role-selector`.
    
3. **Acción:** Interceptar la petición a `/role-selector` y eliminarla.
    
4. Navegar directamente a `/`.
    

---

## 11. Bypass por Oráculo de Cifrado

Un "Oráculo" es cualquier función del sistema que nos permite deducir información sobre datos cifrados o que cifra datos por nosotros de forma indirecta.

### 🔍 El Escenario (AES & Padding)

Si la cookie `stay-logged-in` está cifrada pero existe otra cookie (ej: `notification`) que el servidor **descifra y muestra en el HTML**, tenemos un Oráculo de Descifrado.

### 🛠️ Explotación paso a paso

1. **Identificar el Formato:** Desciframos nuestra propia cookie usando el campo de notificación. Vemos que el formato es `usuario:timestamp`.
    
2. **Construir el Payload:** Queremos suplantar al `administrator`. El payload deseado es `administrator:1771428631576`.
    
3. **Cifrado mediante Error:** Forzamos un error en un campo (ej. email) que se refleje en la cookie de notificación.
    
4. **Alineación de Bloques (AES):** AES cifra en bloques de 16 bytes. Si el servidor añade un prefijo (como "Invalid email: "), debemos añadir caracteres de padding (como 'x') para que nuestro payload empiece exactamente al inicio de un nuevo bloque.
    


```HTTP
POST /post/comment 
...
email=xxxxxxxxxadministrator:1771428631576
```

_Las 9 'x' alinean el texto para que `administrator...` sea un bloque limpio que podamos extraer de la cookie `notification` resultante y pegarlo en `stay-logged-in`._

---

## 12. Bypass por Parsing Inconsistente de Email

Esta es una técnica avanzada basada en la investigación ["Splitting the Email Atom"](https://portswigger.net/research/splitting-the-email-atom). Se aprovecha de cómo distintos parsers interpretan estándares como **MIME Encoded-Words**.

### 🔍 Concepto Técnico

El servidor web utiliza un valider (Regex o librería) que busca una cadena que termine en `@ginandjuice.shop`. Sin embargo, el **MTA (Mail Transfer Agent)** encargado de enviar el correo interpreta codificaciones como **UTF-7** de forma distinta.

### 🛠️ Anatomía del Payload (UTF-7)

El atacante usa la siguiente cadena en el registro:

```
=?utf-7?q?attacker&AEA=-exploit-0a0f0065035d35bb818d8a0001dc00f5.exploit-server.net&ACA=-?=@ginandjuice.shop
```

1. **La Trampa del Validador:** El servidor ve que el string termina en `@ginandjuice.shop` y permite el registro.
    
2. **La Trampa del Mailer:** El sistema de envío de correos ve la secuencia `=?utf-7?q?...?=`. Esto es un "MIME encoded-word". Al decodificarlo, el mailer ignora el final y envía el token de activación a: `attacker@exploit-server.net`.
    
3. **Inconsistencia:**
    
    - **Parser A (App):** "Es un usuario de @ginandjuice.shop".
        
    - **Parser B (Email):** "Enviar a @exploit-server.net".
        

### 📊 Tabla Comparativa de Interpretación

|**Componente**|**Interpretación de la Dirección**|**Resultado**|
|---|---|---|
|**Filtro de Registro**|Cadena literal que termina en `@ginandjuice.shop`|**PASS** ✅|
|**Servidor SMTP**|Decodifica UTF-7 y extrae dominio del atacante|**ENTREGA** 📧|
|**Base de Datos**|Almacena el alias decodificado o la cadena original|**PERMITIDO** 💾|

---

Con esto concluyes el bloque de **Lógica de Negocio**. Estas técnicas te permiten comprometer aplicaciones que, a nivel de código (SQLi, XSS), podrían estar perfectamente blindadas.

---

## Checklist: Auditoría de Lógica de Negocio

### 1. Fase de Reconocimiento de Flujo

- [ ] **Mapeo de Endpoints:** Identificar todas las funciones que impliquen una transacción (compras, cambios de perfil, registros).
    
- [ ] **Identificación de Roles:** ¿Existen diferentes niveles de acceso? (User, Mod, Admin).
    
- [ ] **Análisis de Parámetros:** Listar qué datos se envían al servidor y cuáles parecen ser "informativos" (precios, descripciones).
    

---

### 2. Manipulación de Parámetros de Confianza

- [ ] **Modificación de Precios/Descuentos:**
    
    - Intentar cambiar `price=1337` por `price=1` en el POST de compra.
        
- [ ] **Inyección de Atributos:**
    
    - Añadir parámetros no presentes en la interfaz (ej. añadir `&isAdmin=true` a un registro).
        
- [ ] **Bypass de Cupones:**
    
    - Probar aplicación múltiple: `Cupón A` -> `Cupón B` -> `Cupón A`.
        

---

### 3. Pruebas de Límites y Tipos de Datos

- [ ] **Cantidades Negativas:**
    
    - Introducir `-1` en campos de cantidad para restar valor al total.
        
- [ ] **Integer Overflow:**
    
    - Introducir el valor máximo de un entero de 32 bits ($2,147,483,647$) o múltiplos masivos para forzar el _wrap-around_.
        
- [ ] **Agotamiento de Recursos:**
    
    - ¿Puedo pedir 0 unidades? ¿Puedo pedir $10^{15}$ unidades?
        

---

### 4. Integridad del Flujo de Trabajo (Workflows)

- [ ] **Navegación Forzada (Forceful Browsing):**
    
    - Saltar directamente a `/checkout/success` o `/admin/config`.
        
- [ ] **Máquina de Estados:**
    
    - Interceptar y eliminar peticiones intermedias (ej. el selector de roles tras el login).
        
- [ ] **Endpoints de Doble Uso:**
    
    - En funciones de "Cambio de Password", eliminar el parámetro `current-password` y cambiar el `username`.
        

---

### 5. Identidad y Parsing Avanzado

- [ ] **Truncado de Datos:**
    
    - Registrarse con un email de más de 255 caracteres para ver si el dominio final se corta en la base de datos.
        
- [ ] **Oráculo de Cifrado:**
    
    - ¿Hay cookies cifradas cuyo contenido se refleje en el HTML (notificaciones, mensajes de error)?
        
    - Usar padding de bloques (AES 16 bytes) para alinear payloads.
        
- [ ] **Parsing Inconsistente (Email):**
    
    - Probar payloads UTF-7: `=?utf-7?q?...?=`.
        
    - Verificar si el validador de la web y el servidor SMTP interpretan el `@dominio` de la misma forma.
        

---

#### 📊 Resumen de Severidad por Técnica

| **Técnica**              | **Impacto Potencial**      | **Dificultad de Detección** |
| ------------------------ | -------------------------- | --------------------------- |
| **Confianza en Cliente** | Crítico (Fraude)           | Baja (Manual)               |
| **Integer Overflow**     | Alto (Bypass de Pago)      | Media                       |
| **Máquina de Estados**   | Crítico (Privesc)          | Media                       |
| **Parsing UTF-7**        | Crítico (Account Takeover) | Alta                        |

---

> [!NOTE]
> 
> **Consejo de Profesional:** Siempre que encuentres un error de lógica, documenta el "Estado Esperado" vs el "Estado Obtenido". Esto ayuda a los desarrolladores a entender que el fallo no es un bug de código, sino de diseño.