-----------
Tags: #docker #contenedores #imagenes #volumenes

---
# Definición

[[Docker]] es una tecnología de contenerización que permite a los usuarios agrupar una aplicación y sus dependencias en una sola imagen de contenedor. Los contenedores Docker comparten el mismo kernel del sistema operativo que el sistema host, lo que reduce el uso de recursos y brinda más flexibilidad con respecto a los requisitos de hardware.

Docker es una plataforma de software que permite crear, probar e implementar aplicaciones rápidamente. Docker empaqueta software en unidades estandarizadas llamadas contenedores que incluyen todo lo necesario para que el software se ejecute, incluidas bibliotecas, herramientas de sistema, código y versión ejecutable.

---
# Parámetros y Usos

### 1. Docker Compose

>Docker Compose es una herramienta de Docker que permite definir y ejecutar aplicaciones Docker de varios contenedores de forma fácil y rápida. Esta herramienta se utiliza para la orquestación local de contenedores, lo que significa que permite definir y ejecutar aplicaciones multicontenedor utilizando archivos YAML. El archivo YAML, comúnmente llamado docker-compose.yml, se utiliza para configurar los diferentes servicios pertenecientes a la aplicación.

|   |   |
|---|---|
|Comando|Descripción|
|docker-compose up -d|Levanta los contenedores en segundo plano (modo “detached”) según el archivo docker-compose.yml.|
|docker-compose down|Detiene y elimina los contenedores definidos por Docker Compose.|

### 2. Gestión de Contenedores

|   |   |
|---|---|
|Comando|Descripción|
|docker ps -a|Lista todos los contenedores, incluidos los detenidos.|
|docker rm [ID] --force|Elimina un contenedor de forma forzada, aunque esté en ejecución.|
|docker exec -it [ID] bash|Ejecuta un shell interactivo (bash) dentro de un contenedor en ejecución.|
|docker port [ID]|Muestra las reglas de redirección de puertos del contenedor.|
|docker run -it [imagen]|Inicia un contenedor en modo interactivo con terminal.|

### 3. Gestión de Imágenes

|   |   |
|---|---|
|Comando|Descripción|
|docker images -q|Muestra solo los IDs de las imágenes disponibles en el sistema.|
|docker rmi [imagen]|Elimina una imagen del sistema local.|
|docker pull [imagen]|Descarga una imagen desde Docker Hub (o repositorio definido).|
|docker build -t [nombre] .|Construye una imagen a partir del Dockerfile en el directorio actual y le asigna un nombre.|

### 4. Comando docker run con Argumentos

El comando `docker run` es utilizado para crear y ejecutar un contenedor a partir de una imagen especificada. Este comando es fundamental en Docker, ya que permite la creación e inicio de contenedores con configuraciones personalizadas.

|   |   |
|---|---|
|Argumento|Descripción|
|-d|Modo detached (para ejecutarlo en segundo plano).|
|-i|Permite la entrada estándar (stdin) al contenedor.|
|-t|Asignación de nombre (tag) a la imagen.|
|-p [host:cont]|Redirecciona puertos (host → contenedor).|
|-v [ruta_host:ruta_cont]|Monta un volumen del host al contenedor.|
|--name|Asigna un nombre específico al contenedor.|

### 5. Otros Argumentos y Comandos Importantes

|   |   |
|---|---|
|Comando / Argumento|Descripción|
|docker logs [ID]|Muestra los logs de un contenedor.|
|--rm|Elimina el contenedor automáticamente al detenerse.|
|--network host|Ejecuta el contenedor utilizando la red del host directamente.|
|-e [VAR=VALOR]|Define variables de entorno.|
|--privileged|Otorga privilegios extendidos al contenedor (por ejemplo, para acceso completo al host).|

#### Ejemplos Prácticos de Docker

- Levantar contenedores en segundo plano con Docker Compose:  
    ```bash
    docker-compose up -d  
```
      
    Levanta todos los contenedores definidos en docker-compose.yml y los ejecuta en segundo plano.
    
- Ejecutar un contenedor con puertos y volúmenes montados:  
    ```bash
    docker run -dit -p 8080:80 -v /home/user:/data --name miweb nginx 
``` 
      
    Ejecuta un contenedor interactivo en segundo plano (-dit) con la imagen NGINX, mapea el puerto 8080 del host al 80 del contenedor (-p 8080:80) y monta la carpeta /home/user del host en /data dentro del contenedor (-v /home/user:/data). Le asigna el nombre miweb (--name miweb).
    
- Ver todos los contenedores (detenidos y activos):  
    ```bash
    docker ps -a  
```
      
    Muestra una lista completa de todos los contenedores, incluyendo aquellos que no están en ejecución.
    
- Eliminar un contenedor forzadamente:  
    ```bash
    docker rm contenedor123 --force
```  
      
    Elimina el contenedor con el ID o nombre contenedor123 de forma forzada, incluso si está en ejecución.
    
- Eliminar una imagen del sistema local:  
    ```bash
    docker rmi nginx
```  
      
    Elimina la imagen llamada nginx de tu sistema local.
    
- Entrar en un contenedor en ejecución para interactuar con él:  
    ```bash
    docker exec -it contenedor123 bash  
```
      
    Ejecuta un shell bash interactivo (-it) dentro del contenedor con el ID o nombre contenedor123.
    
