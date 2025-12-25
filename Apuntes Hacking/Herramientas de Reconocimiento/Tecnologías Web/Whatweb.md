-----------------------
Tags: #herramientas #reconocimiento #web #enumeración #whatweb

------------------
# Definición

[[WhatWeb]] es una herramienta de código abierto utilizada para recopilar información sobre una aplicación web, identificando las tecnologías que se utilizan en su desarrollo. Esta herramienta es un escáner de nueva generación que detecta tecnologías como sistemas de gestión de contenidos, plataformas de blog, paquetes de estadísticas/analítica, librerías de JavaScript, servidores web, dispositivos integrados, versiones de software, entre otros.

------------------
# Parámetros y usos

```bash
whatweb example.com
```

- Ejemplo con parámetros avanzados:

```
whatweb -v -a 3 example.com
```

Explicación:

- `-v`: Muestra información detallada (modo _verbose_) durante el escaneo.    
- `-a 3`: Establece el nivel de agresividad del escaneo. Los niveles van de 0 a 3:
    - `0`: Solo encabezados (rápido).
    - `1`: Escaneo normal (por defecto).
    - `2`: Escaneo completo.
    - `3`: Fuerza bruta, más intrusivo pero más exhaustivo.