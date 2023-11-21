##### Daniel Vicente Ramos - 2023


### Tabla de contenidos

1. [Introducción](#introduction)
2. [Requisitos previos](#prerequisites)
3. [Docker Compose](#dockercompose)
	1. [MySQL](#mysql)
		1. [Configuración básica del servicio](#mysqlbasicconfig)
		2. [Configuración específica MySQL](#mysqlspecificconfig)
		3. [Configuración del entorno del servicio](#mysqlenvironmentconfig)
		4. [Configuración de red](#mysqlnetworkconfig)
		5. [Configuración de política de reinicio](#mysqlrestartbehaviour)
		6. [Configuración de persistencia](#mysqldatapersistence)
	2. [WordPress](#wordpress)
		1. [Configuración básica del servicio](#wordpressbasicconfig)
		3. [Configuración del entorno del servicio](#wordpressenvironmentconfig)
		4. [Configuración de red](#wordpressnetworkconfig)
		5. [Configuración de política de reinicio](#wordpressrestartbehaviour)
		6. [Configuración de persistencia](#wordpressdatapersistence)
	3.  [Prueba de despliegue en local](#examplelocaldeploy)
4. [Despliegue en Azure](#azuredeploy)
	1. [Iniciar sesión en Azure](#azurelogin)
	2. [Crear un grupo de recursos](#createresourcegroup)
	3. [Azure Container Registry (ACR)](#acr)
	4. [Modificar Docker Compose](#modifydockercompose)
	5. [Crear un plan de servicio de aplicaciones](#createappserviceplan)
	6. [Crear y configurar Web App](#createwebapp)
	7. [Configuración final y resultados](#finalconfig)
5. [Problemas encontrados y soluciones](#problemsandsolutions)
	1. [Problemas en el despliegue en local](#localdeployproblems)
	2. [Problemas en el despliegue en Azure](#azuredeployproblems)
		1. [Problemas con Azure Container Registry (ACR)](#acrproblems)
		2. [Problemas en el plan de servicio de aplicaciones](#appserviceplanproblems)
		3. [Problemas al crear y configurar Web App](#webappproblems)
6. [Conclusiones](#conclusions)
7. [Referencias](#references)

<div id="introduction"/>

### Introducción

Este documento muestra el proceso completo para hacer un despliegue de un WordPress con MySQL en Azure con persistencia, mediante el uso de *Docker Compose*. Las imágenes de WordPress y MySQL estarán alojadas en la nube de Azure y serán públicas. El servicio de WordPress estará expuesto al exterior por el puerto 80 y la comunicación entre los servicios se hará a través de la red interna de Docker.

Se ha separado la configuración inicial del *Docker Compose* y el despliegue en Azure, ya que, el despliegue en local y en Azure tiene ligeras variaciones.
<div id="prerequisites"/>

### Requisitos previos

* Docker Desktop instalado en el equipo y ejecutándose en segundo plano (En referencias se encuentra la [guía de instalación](#dockerinstalation)).
* Cuenta en Azure con saldo (Ej: cuenta de estudiante, dan 100€ para gastar)
* Tener instalada la **[consola de Azure](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)** en el equipo e instalar **[Helm](https://helm.sh/docs/intro/quickstart/#install-helm)** para despliegue de contenedores.
* Conexión a internet
* Tiempo
<div id="dockercompose"/>

### Docker Compose 

Antes de empezar seleccionaremos dos imágenes de [DockerHub](https://hub.docker.com/) para WordPress y MySQL, en concreto para esta práctica se ha optado por [MySQL 8.2.0](https://hub.docker.com/layers/library/mysql/8.2.0/images/sha256-d43bab9d9bd18d3770f6156bdb7c5364cac797c6a906e67cf548b0a439ff1d2d?context=explore) y [WordPress php8.2-apache](https://hub.docker.com/layers/library/wordpress/php8.2-apache/images/sha256-471e18f46fbfda911c90ec5af17425b298689ae318498ced4a96530b49551c09?context=explore). De estas páginas se guardará la etiqueta de la imagen que necesitaremos más adelante, en el caso de MySQL (mysql:8.2.0)([Imagen 1.1](#img-1.1)) y en el de WordPress (wordpress:php8.2-apache)([Imagen 1.2](#img-12)).  
<div id="img-11"/>

![](/imgs/img-11.png)

Imagen 1.1 - Etiqueta de la imagen de MySQL
<div id="img-12"/>

![](/imgs/img-12.png)

Imagen 1.2 - Etiqueta de la imagen de WordPress

Se procede a crear un fichero *docker-compose.yml* con la siguiente configuración:

```
version: '3.7'

services:

    # MySQL DB
    db:
    
        image: mysql:8.2.0
        
        container_name: mysqldb
        
        # Force MySQL use authentication based on the password hashing method
        # Hide pid in another folder for security reasons
        command: ["--default-authentication-plugin=caching_sha2_password", "--pid-file=/hidden/services/mysqld.pid"]
        
        environment:
            MYSQL_ROOT_PASSWORD: root
            MYSQL_DATABASE: wordpress
            MYSQL_USER: user
            MYSQL_PASSWORD: root
            MYSQL_INITDB_SKIP_TZINFO: 1
            
        ports:
            - "8081:3306"
            
        restart: always
        
        volumes:
            - mysqldata:/var/lib/mysql
            
        networks:
            - wordpressnet


    # WordPress
    wordpress:
    
        image: wordpress:php8.2-apache
        
        container_name: wordpress
        
        depends_on:
            - db
            
        environment:
            WORDPRESS_DB_USER: user
            WORDPRESS_DB_PASSWORD: root
            WORDPRESS_DB_NAME: wordpress
            WORDPRESS_DB_HOST: db
            
        ports:
            - "8080:80"
            
        restart: unless-stopped
        
        volumes:
            - wordpressdata:/var/www/html
            
        networks:
            - wordpressnet

# Networks
networks:
    wordpressnet:

# Persistency
volumes:
    mysqldata:
    wordpressdata:
```

En el código anterior creamos dos servicios, uno para MySQL ("db") y otro para WordPress ("wordpress"). A continuación se verá más en detalle las configuraciones para los servicios.
<div id="mysql"/>

#### MySQL
<div id="mysqlbasicconfig"/>

##### Configuración básica del servicio

```
    image: mysql:8.2.0
    container_name: mysqldb
```

Se pega la etiqueta que se había copiado antes en la imagen ("image") de este servicio. A mayores, con la opción "container_name" le ponemos un nombre al contenedor donde se va a ejecutar (en este caso "mysqldb" sino por defecto será el nombre del servicio o una variación por ej. "db-1"). 
<div id="mysqlspecificconfig"/>

##### Configuración específica MySQL

```
    command: ["--default-authentication-plugin=caching_sha2_password", "--pid-file=/hidden/services/mysqld.pid"]
```

El comando ("command") que se ejecuta al desplegar el servicio, esta opción permite enviar comandos al servicio en el despliegue. El primer comando(\*) permite conectarnos a MySQL usando contraseña que hemos puesto en "*MYSQL_PASSWORD*" como autenticación (Mirar el apartado de [Problemas en el despliegue en local](#localdeployproblems) para más información). El segundo comando sirve para cambiar de ruta el *pid* de MySQL para evitar problemas de seguridad, ya que la ruta por defecto es accesible por cualquier usuario dentro del contenedor.

(\*) **Actualización:** *En algunos casos parece que ya no es necesario usar este comando y por defecto usa esta configuración pero para evitar problemas se debería utilizar.*
###### Ejemplo de conexión con la base de datos de MySQL para esta configuración

En el caso de que se necesite conectar con la base de datos de MySQL se puede hacer de la siguiente forma.

Ejecutar en cualquier terminal: 

```shell
mysql -h localhost -p 8081 -u user -p 
``` 

o alternativamente acceder a la terminal del contenedor y luego conectar a MySQL:

```bash 
docker exec -it mysqldb /bin/bash
mysql -u user -p
```
<div id="mysqlenvironmentconfig"/>

##### Configuración del entorno del servicio

```
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: wordpress
      MYSQL_USER: user
      MYSQL_PASSWORD: root
      MYSQL_INITDB_SKIP_TZINFO: 1
```

Para este servicio también se configurará un entorno con una configuración básica de MySQL:

* Contraseña de administrador de base de datos (*MYSQL_ROOT_PASSWORD*): "root" (\*)
* Nombre de la base de datos (*MYSQL_DATABASE*): "wordpress" (o cualquier otro)
* Nombre de usuario (*MYSQL_USER*): "user" (\*)
* Contraseña del usuario (*MYSQL_PASSWORD*): "root" (\*)
* Para arrancar la base de datos más rápido, con la configuración (*MYSQL_INITDB_SKIP_TZINFO*) en cualquier valor no vacío deshabilita esta opción. Al deshabilitarla se evitan logs innecesarios y tiempos de carga a cambio de no poder cargar la información de nuestra zona horaria necesaria en la función "CONVERT_TZ()". En este ejemplo no es necesaria esta función, por ello se ha deshabilitado.
<sub>(*) Evitar usar estos nombres y contraseñas, esto es simplemente un ejemplo de configuración.</sub>
<div id="mysqlnetworkconfig"/>

##### Configuración de red

```
    ports:
      - "8081:3306"
```

En este ejemplo, los puertos (*ports*) del servicio de MySQL se han configurado de forma que el puerto de host (máquina local) "8081" esté vinculado con el del contenedor "80". Esto se ha hecho así  para evitar problemas con otros puertos, aunque se podría configurar el puerto de host a cualquier otro (siempre que no colisione con otros servicios). 

```
    expone:
      - "3306"
```

Alternativamente a la opción de "ports" se puede utilizar "expone" si no se quiere exponer el puerto de la máquina host, esto es, que no se pueda acceder desde fuera del entorno de Docker a la base de datos. Esto no permitiría acceder a la base de datos utilizando el comando "mysql" desde el terminal.

**Conexión entre servicios**

```
    networks:
      - wordpressnet
```

Para que el servicio de MySQL se pueda acceder desde otros servicios, a mayores de configurar los puertos, es necesario establecer una red ("network"). Esta opción la deben tener todos los servicios en los que se requiera que haya un intercambio de datos (conexión).  Se podría utilizar cualquier nombre para la red, para este ejemplo es "wordpressnet". 
<div id="mysqlrestartbehaviour"/>

##### Configuración de política de reinicio

```
    restart: always
```

Se ha establecido una configuración de reinicio del servicio para que se reinicie siempre en caso de fallo o se detenga, permitiendo así la máxima disponibilidad de la base de datos.
<div id="mysqldatapersistence"/>

##### Configuración de persistencia

```
    volumes:
      - mysqldata:/var/lib/mysql
```

En lugar de que Docker cree un volumen temporal (cuando se detiene se eliminan los datos) para los datos de la base de datos parece lógico que haya que crear un volumen de almacenamiento para persistir los datos. Para ello, se utiliza la opción "volumes" del fichero *docker-compose.yml*, en este caso, se ha llamado al volumen "mysqldata" (se podría utilizar cualquier otro nombre y ubicación).
<div id="wordpress"/>

#### WordPress
<div id="wordpressbasicconfig"/>

##### Configuración básica del servicio

```
    image: wordpress:php8.2-apache
    container_name: wordpress
    depends_on:
      - mysqldb
```

Se pega la etiqueta que se había copiado antes en la imagen ("image") de este servicio. A mayores, con la opción "container_name" le ponemos un nombre al contenedor donde se va a ejecutar (en este caso "wordpress" sino por defecto será el nombre del servicio o una variación por ej. "wordpress-1"). Se ha añadido la opción "depends_on" para que inicie primero el contenedor de "mysqldb" antes de ejecutar el contenedor de "wordpress".
<div id="wordpressenvironmentconfig"/>

##### Configuración del entorno del servicio

```
    environment:
      WORDPRESS_DB_USER: user
      WORDPRESS_DB_PASSWORD: root
      WORDPRESS_DB_NAME: wordpress
      WORDPRESS_DB_HOST: mysqldb
```

En el entorno del WordPress tenemos que configurarlo de forma que se pueda conectar con la base de datos de MySQL. Hay que asegurarse que los parámetros coincidan con los del entorno del servicio de MySQL. Por último, con la opción "WORDPRESS_DB_HOST" para especificar la dirección del servidor, en este caso "mysqldb" que es donde está ubicado el servicio de MySQL.
<div id="wordpressnetworkconfig"/>

##### Configuración de red

```
    ports:
      - "8080:80"
```

En este ejemplo, los puertos (*ports*) del servicio de WordPress se han configurado de forma que el puerto de host (máquina local) "8080" esté vinculado con el del contenedor "80". Esto se ha hecho así  para evitar problemas con otros puertos, aunque se podría configurar el puerto de host a cualquier otro (siempre que no colisione con otros servicios). 

**Conexión entre servicios**

```
    networks:
      - wordpressnet
```

Como se ha mencionado antes, para que los servicios puedan reconocerse entre si tienen que estar en la misma red. El nombre de la red "wordpressnet", debe coincidir con el del servicio de la base de datos.
<div id="wordpressrestartbehaviour"/>

##### Configuración de política de reinicio

```
    restart: unless-stopped
```

Se ha establecido una configuración de reinicio del servicio para que se reinicie siempre en caso de fallo excepto si se detiene de forma explícita, permitiendo así mayor control sobre el contenedor.
<div id="wordpressdatapersistence"/>

##### Configuración de persistencia

```
    volumes:
      - wordpressdata:/var/www/html
```

Para mantener la persistencia de los datos del WordPress, se ha de crear otro volumen. Para ello, se utiliza la opción "volumes" del fichero *docker-compose.yml*, en este caso, se ha llamado al volumen "wordpressdata" (se podría utilizar cualquier otro nombre y ubicación).
<div id="examplelocaldeploy"/>

#### Prueba de despliegue en local

Si se han seguido los pasos correctamente y accedemos a la ubicación del fichero *docker-compose.yml* desde el terminal, podremos ejecutar el siguiente comando:

```
docker compose up -d
```

Con este comando se iniciaran los servicios y opcionalmente, utilizando el parámetro "-d" ("detach" o desconectado), nos permitirá seguir utilizando la terminal como se puede apreciar en la siguiente imagen ([Imagen 1.3](#img-13)).
<div id="img-13"/>

![](/imgs/img-13.png)

Imagen 1.3 - Despliegue local

Después de realizar el despliegue en local, al acceder a la dirección web "http://localhost:8080" debería redireccionar a la siguiente página ([Imagen 1.4](#img-14)), sin mostrar ningún error. En caso de que salga un error estableciendo conexión con la base de datos, tendremos que esperar a que cargue (Normalmente 1~2min, recargar cada 30s). En caso de que siga sin funcionar, revisar el apartado de problemas y soluciones referente a este apartado.
<div id="img-14"/>

![](/imgs/img-14.png)

Imagen 1.4 - Instalación de WordPress

En el caso de que el error persista, revisar el apartado [Problemas en despliegue local](#localdeployproblems) del apartado de [Problemas encontrados y soluciones](#problemsfoundandsolutions).
<div id="azuredeploy"/>

### Despliegue con Azure
<div id="azurelogin"/>

##### Iniciar sesión en Azure

Accedemos a la terminal del sistema operativo y comprobamos que podemos ejecutar el siguiente comando para hacer el inicio de sesión en Azure mediante la Azure CLI.

```
az login
```

Una vez iniciada la sesión en Azure debería mostrar algo como la siguiente imagen.
<div id="img-17"/>

![](/imgs/img-17.png)

Imagen 1.7 - Iniciar sesión en Azure con Azure CLI
<div id="createresourcegroup"/>

##### Crear un grupo de recursos

Para crear cualquier recurso en Azure es necesario tener un grupo de recursos en la plataforma y debe estar asignado a una ubicación. A continuación se muestra el comando para poder crear un grupo de recursos.

```
az group create --name "resourceGroup" --location "West Europe"
```
<div id="img-18"/>

![](/imgs/img-18.png)

Imagen 1.8 - Crear grupo de recursos con Azure CLI
<div id="acr"/>

##### Azure Container Registry (ACR) 

Para poder tener un repositorio con las imágenes de MySQL y WordPress en Azure, se ha de crear un ACR (Azure Container Registry). Este ACR debe estar asociado a un grupo de recursos y utilizar una ubicación compatible con el grupo de recursos. Para crear el ACR es necesario ejecutar el siguiente comando:

```
az acr create --name "danivwordpressimages" --resource-group "resourceGroup" --location "West Europe" --sku Standard
```
<div id="img-19" />

![](/imgs/img-19.png)

Imagen 1.9 - Crear grupo de recursos con Azure CLI

Para poder acceder al contenido del ACR se debe iniciar sesión en él, con esto se quedan las credenciales guardadas en local. No es obligatorio ejecutar este paso y se puede saltar, se puede mostrar un error en la consola cuando se intente iniciar sesión desde Docker (en ambos casos va a seguir pidiendo iniciar sesión igualmente desde Docker) pero no influye en el resto de pasos.

```
az acr login --name "danivwordpressimages"
```
<div id="img-110"/>

![](/imgs/img-110.png)

Imagen 1.10 - Iniciar sesión en ACR con Azure CLI

Se debe hacer un *pull* de las imágenes de DockerHub si no se tienen ya del despliegue en local. Estas son las imágenes que se subirán al ACR para hacer el despliegue en Azure. Para obtener las imágenes se ejecutan los siguientes comandos:

```
docker pull wordpress:php8.2-apache
```
<div id="img-111"/>

![](/imgs/img-111.png)

Imagen 1.11 - Obteniendo la imagen de WordPress

```
docker pull mysql:8.2.0
```
<div id="img-112"/>

![](/imgs/img-112.png)

Imagen 1.12 - Obteniendo la imagen de MySQL

Las imágenes no se deben subir sin etiquetar al ACR, pueden estar recuperando las imágenes de DockerHub en lugar de las de Azure, también es una mala práctica etiquetarlas como "latest", ya que "la última versión de hoy puede no coincidir con la de mañana". Por ello, la etiquetaremos de la siguiente forma "<\nombre del acr\>.azurecr.io/<\servicio\>:<\version\>". Para hacer esto, se utilizan los siguientes comandos:

```
docker tag wordpress:php8.2-apache danivwordpressimages.azurecr.io/wordpress:8.2
```
<div id="img-113"/>

![](/imgs/img-113.png)

Imagen 1.13 - Etiquetando la imagen de WordPress

```
docker tag mysql:8.2.0 danivwordpressimages.azurecr.io/mysql:8.2.0
```
<div id="img-114"/>

![](/imgs/img-114.png)

Imagen 1.14 - Etiquetando la imagen de MySQL

Para poder subir las imágenes al repositorio del ACR, se necesitan las claves de acceso. Por ello, se tiene que acceder al portal de Azure ("https://portal.azure.com/") > Todos los recursos ("All resources") > "danivwordpressimages" (el nombre de tu ACR) > "Access keys" (Claves de acceso). Aquí tendremos que hacer clic en la casilla de "Admin user", copiaremos el "Username" y la "password" en un bloc de notas u otro editor de texto.
<div id="img-115"/>

![](/imgs/img-115.png)

Imagen 1.15 - Portal de Azure
<div id="img-116"/>

![](/imgs/img-116.png)

Imagen 1.16 - Vista todos los recursos en la plataforma de Azure
<div id="img-117"/>

![](/imgs/img-117.png)

Imagen 1.17 - Vista de administración de un ACR en la plataforma de Azure
<div id="img-118"/>

![](/imgs/img-118.png)

Imagen 1.18 - Vista de las claves de acceso de un ACR en la plataforma de Azure

Ahora procedemos a iniciar sesión con Docker al repositorio del ACR, utilizaremos el usuario y contraseña que acabamos de copiar (Mirar apartado de [Problemas encontrados y soluciones](#acrproblems)). Para poder iniciar sesión, se utiliza el comando: 

```
docker login danivwordpressimages.azurecr.io
```
<div id="img-119"/>

![](/imgs/img-119.png)

Imagen 1.19 - Inicio de sesión con Docker en el ACR

Una vez hemos iniciado sesión sólo queda subir las imágenes al ACR. Con los siguientes comandos podremos subir las imágenes:

```
docker push danivwordpressimages.azurecr.io/wordpress:8.2

docker push danivwordpressimages.azurecr.io/mysql:8.2.0
```
<div id="img-121"/>

![](/imgs/img-121.png)

Imagen 1.21 - Subiendo la imagen de WordPress al ACR
<div id="img-122"/>

![](/imgs/img-122.png)

Imagen 1.22 - Subiendo la imagen de MySQL al ACR

Después de subir las imágenes al repositorio del ACR, este se puede poner público utilizando los siguientes comandos:

```
az acr update --name "danivwordpressimages" --public-network-enabled true
```
<div id="img-123"/>

![](/imgs/img-123.png)

Imagen 1.23 - Configurando ACR para habilitar el acceso público

```
az acr update --name "danivwordpressimages" --anonymous-pull-enabled
```
<div id="img-124"/>

![](/imgs/img-124.png)

Imagen 1.24 - Configurando ACR para no necesitar inicio de sesión para obtener las imágenes

No es obligatorio hacerlo público pero para el objetivo de este ejemplo se hará así.
<div id="modifydockercompose"/>

##### Modificar Docker Compose

En este punto, ya podemos modificar el fichero *docker-compose.yml* para poder hacer el despliegue utilizando las imágenes alojadas en Azure.

```
version: '3.7'

services:

    # MySQL DB
    db:
    
        image: danivwordpressimages.azurecr.io/mysql:8.2.0
        
        container_name: mysqldb
        
        # Force MySQL use authentication based on the password hashing method
        # Hide pid in another folder for security reasons
        command: ["--default-authentication-plugin=caching_sha2_password", "--pid-file=/hidden/services/mysqld.pid"]
        
        environment:
            MYSQL_ROOT_PASSWORD: root
            MYSQL_DATABASE: wordpress
            MYSQL_USER: user
            MYSQL_PASSWORD: root
            MYSQL_INITDB_SKIP_TZINFO: 1
            
        expose:
            - "3306"
            
        restart: always
        
        volumes:
            - mysqldata:/var/lib/mysql
            
        networks:
            - wordpressnet


    # WordPress
    wordpress:
    
        image: danivwordpressimages.azurecr.io/wordpress:8.2
        
        container_name: wordpress
        
        depends_on:
            - db
            
        environment:
            WORDPRESS_DB_USER: user
            WORDPRESS_DB_PASSWORD: root
            WORDPRESS_DB_NAME: wordpress
            WORDPRESS_DB_HOST: db
            
        ports:
            - "80:80"
            
        restart: unless-stopped
        
        volumes:
            - wordpressdata:/var/www/html
            
        networks:
            - wordpressnet

# Networks
networks:
    wordpressnet:

# Persistency
volumes:
    mysqldata:
    wordpressdata:
```

En concreto se ha modificado las imágenes por las que se han subido a Azure y los puertos ("ports") de la base de datos de MySQL a la opción de "expose", para evitar poder acceder directamente desde el exterior. También se ha cambiado el puerto de WordPress al puerto 80, ya que al crear el Web App por defecto el puerto estará libre.
<div id="createappserviceplan"/>

##### Crear un plan de servicio de aplicaciones

Un plan de servicio de aplicaciones de Azure permite configurar un entorno para ser compatible con un sistema operativo, el cual se puede utilizar para hospedar y escalar aplicaciones web en Azure. Este plan está asignado a un grupo de recursos y se utilizará para posteriormente crear un *Web App* en él.

```
az appservice plan create --name "danivwordpressplan" --resource-group "resourceGroup" --sku S1 --is-linux 
```
<div id="img-125"/>

![](/imgs/img-125.png)

Imagen 1.25 - Crear un plan de servicio de aplicaciones

Mirar los [Problemas encontrados y soluciones referentes a este apartado](#planproblems).
<div id="createwebapp"/>

##### Crear y configurar Web App

Un "Web App" es una aplicación basada en web en la que los usuarios pueden acceder a través del navegador web. El siguiente comando sirve para crear un *Web App* a partir de un plan de servicio, un grupo de recursos y un fichero "docker-compose.yml". 

```
az webapp create --resource-group "resourceGroup" --plan "danivwordpressplan" --name "danivwordpressweb" --multicontainer-config-type "compose" --multicontainer-config-file "docker-compose.yml"
```
<div id="img-126"/>

![](/imgs/img-126.png)

Imagen 1.26 - Crear una aplicación web desde Azure CLI [1/4]
<div id="img-127"/>

![](/imgs/img-127.png)

Imagen 1.27 - Crear una aplicación web desde Azure CLI [2/4]
<div id="img-128"/>

![](/imgs/img-128.png)

Imagen 1.28 - Crear una aplicación web desde Azure CLI [3/4]
<div id="img-129"/>

![](/imgs/img-129.png)

Imagen 1.29 - Crear una aplicación web desde Azure CLI [4/4]

Una vez creada la *Web App*, se tiene que configurar para que utilice el ACR que hemos creado sino igual no va a poder encontrar las imágenes del Docker Compose (por defecto utiliza un "Private Registry"), mirar el apartado "Problemas encontrados y soluciones" correspondiente para este apartado. Para poder configurar correctamente la *Web App*, se ha de detener primeramente mediante este comando:

```
az webapp stop --name "danivwordpressweb" --resource-group "resourceGroup"
```
<div id="img-130"/>

![](/imgs/img-130.png)

Imagen 1.30 - Detener la aplicación web

Ahora si hemos hecho público el ACR, como en este ejemplo, utilizaremos este comando para añadir el ACR:

```
az webapp config container set --name "danivwordpressweb" --resource-group "resourceGroup" --docker-registry-server-url "https://danivwordpressimages.azurecr.io"
```
<div id="img-131"/>

![](/imgs/img-131.png)

Imagen 1.31 - Configurando la aplicación web para que utilice el ACR público que se ha creado previamente, en lugar de usar el privado que viene por defecto

En caso de que tengamos un ACR privado, se va a tener que utilizar los datos de inicio de sesión del ACR que teníamos copiados en un bloc de notas. El parámetro "--docker-registry-server-url" será la url del ACR (normalmente es del estilo "https://<\nombre acr\>.azurecr.io"), la opción "--docker-registry-server-user " que corresponde con el usuario del ACR y la opción "--docker-registry-server-password" con la contraseña del mismo. El comando es el siguiente:

```
az webapp config container set --name "danivwordpressweb" --resource-group "resourceGroup" --docker-registry-server-url "https://danivwordpressimages.azurecr.io" --docker-registry-server-user "danivwordpressimages" --docker-registry-server-password "<password>"
```
<div id="img-132"/>

![](/imgs/img-132.png)

Imagen 1.32 - Configurando la aplicación web para que utilice el ACR privado que se ha creado previamente, en lugar de usar el privado que viene por defecto

**Importante:** Revisar que los campos son correctos, recordad cambiar "password" por la contraseña del ACR que previamente teníamos en un bloc de notas.

Ahora podemos volver a iniciar la *Web App* sin problemas mediante el comando:

```
az webapp start --name "danivwordpressweb" --resource-group "resourceGroup"
```
<div id="img-133"/>

![](/imgs/img-133.png)

Imagen 1.33 - Iniciando la aplicación web

Si todo ha ido bien debería salir la siguiente imagen al acceder a la dirección web ("https://<\nombre-web-app\>.azurewebsites.net"). En este ejemplo:  ("https://danivwordpressweb.azurewebsites.net/"). Puede aparecer temporalmente que no se puede establecer conexión con la base de datos (Alrededor de 1~2 min, actualizar cada 30s). Si el error persiste, revisar en problemas encontrados y soluciones, lo referente a este apartado.
<div id="img-134"/>

![](/imgs/img-134.png)

Imagen 1.34 - WordPress desplegado en Azure con éxito

Mirar el apartado de [Problemas encontrados y soluciones](#webappproblems).
<div id="finalconfig"/>

##### Configuración final y resultados

Seleccionamos el idioma de nuestra preferencia (en este ejemplo "Español") y completamos los campos del formulario de la instalación de WordPress. En este caso, el título del sitio va a ser "Sitio para ICS de Daniel Vicente Ramos". Una vez cubiertos los campos, se hace clic en "Instalar WordPress".

**IMPORTANTE:** Evitar utilizar la contraseña "root" utilizada en este ejemplo y dejar que WordPress genere una contraseña única (guardar esta contraseña en un sitio seguro para evitar perderla).
<div id="img-137"/>

![](/imgs/img-137.png)

Imagen 1.37 - Configuración inicial de WordPress [1/2]
<div id="img-138"/>

![](/imgs/img-138.png)

Imagen 1.38 - Configuración inicial de WordPress [2/2]

Se accede con el usuario y la contraseña elegidas y se configura el WordPress para que no haya moderación en los comentarios. Esto se hace para mostrar la persistencia de los datos y no bloquee el comentario del usuario. En el caso de que se desconecte y quiera volver a iniciar sesión, acceder a la siguiente dirección web: https://danivwordpressweb.azurewebsites.net/wp-admin/ (o igual en su caso "https://<\nombre del web app\>.azurewebsites.net/web-admin/").
<div id="img-139"/>

![](/imgs/img-139.png)

Imagen 1.39 - Inicio de sesión en WordPress

Accedemos a "Ajustes", luego a "Comentarios" y por último, desmarcamos la opción "El autor del comentario debe tener un comentario previamente aprobado", en el apartado "Para que un comentario aparezca". Una vez hecho esto, en la misma página abajo de todo hay un botón que dice "Guardar cambios" y hacemos clic.
<div id="img-140"/>

![](/imgs/img-140.png)

Imagen 1.40 - Quitando moderación de los comentarios en WordPress

Para añadir una entrada, accedemos al apartado "Entradas" y hacemos clic en "Añadir una nueva entrada". En este caso se añadirá una entrada con título "Mi primera entrada de blog!" y añadiré un código hecho en CodePen como contenido.
<div id="img-141"/>

![](/imgs/img-141.png)

Imagen 1.41 - Pasos para añadir una nueva entrada al WordPress

Después de publicar la entrada, para este ejemplo, cambiamos la apariencia > Editor > Borramos toda la apariencia y dejamos sólo las entradas, quedando como en la imagen.
<div id="img-142"/>

![](/imgs/img-142.png)

Imagen 1.42 - Vista de entradas de WordPress con el diseño simplificado para este ejemplo

Salimos del editor, cerramos sesión en WordPress y volvemos a "https://danivwordpressweb.azurewebsites.net/", hacemos clic en "Mi primera entrada de blog!", se debería ver algo como en la imagen.
<div id="img-143"/>

![](/imgs/img-143.png)

Imagen 1.43 - Vista de la entrada "Mi primera entrada de blog!"

Si bajamos hacia la sección de comentarios, podremos añadir un comentario, en este caso será "Hola mundo!" y hacemos clic en "Publicar el comentario".
<div id="img-144"/>

![](/imgs/img-144.png)

Imagen 1.44 - Añadiendo un comentario a la entrada de WordPress

Como se puede apreciar en la imagen, el comentario quedaría registrado.
<div id="img-145"/>

![](/imgs/img-145.png)

Imagen 1.45 - Vista de la entrada "Mi primera entrada de blog!" con el comentario añadido
<div id="problemsandsolutions"/>

### Problemas encontrados y soluciones

En este apartado se pueden ver los problemas que han surgido a la hora de realizar el despliegue, tanto en local como en Azure y cómo solucionarlos.
<div id="localdeployproblems"/>

#### Problemas en el despliegue en local

Durante el despliegue en local, se han encontrado los siguientes errores:

1. **MySQL no conecta**
	
	Una de las causas que se pueden dar es que la base de datos no conecte debido a que no utiliza el plugin de autenticación por defecto, de usuario y contraseña, para ello se utiliza el comando "--default-authentication-plugin=caching_sha2_password" (MySQL 8 en adelante) o "--default-authentication-plugin=mysql_native_password" (en caso de una versión menor a MySQL 8)
	
2. **WordPress no conecta con la base de datos** (> MySQL 8)
	
	En el caso de que se vea alguna de las siguientes imágenes ([Imagen 1.5](#img-15) y [Imagen 1.6](#img-16)), se debería revisar si el contenedor de la base de datos está ejecutándose, revisar la configuración de los entornos ("environments") del *docker-compose.yml* y añadir el parámetro "--default-authentication-plugin=caching_sha2_password" (MySQL 8 en adelante) o "--default-authentication-plugin=mysql_native_password" (en caso de una versión menor a MySQL 8), en la opción de comando ("command") del mismo fichero, si no estuviera añadido previamente.
	<div id="img-15"/>
	
	![](/imgs/img-15.png)
	
	Imagen 1.5 - Error de MySQL en WordPress
	<div id="img-16"/>
	
	![](/imgs/img-16.png)
	
	Imagen 1.6 - Error de MySQL en la instalación de WordPress
<div id="azuredeployproblems"/>

#### Problemas en el despliegue en Azure 
<div id="acrproblems"/>

##### Problemas con Azure Container Registry (ACR)

1. **Error credenciales no existentes**
	
	Si no se hace el inicio de sesión previo en el ACR como se ha mencionado en el apartado correspondiente, puede aparecer el siguiente error (Este error se puede omitir, ya que se pedirá iniciar sesión de ambas formas).
	<div id="img-120"/>
	![](/imgs/img-120.png)
	
	Imagen 1.20 - Error credenciales no existentes
	
2. **Error no se pueden poner públicas las imágenes del repositorio**
	
	Si al ejecutar el comando "az acr update --name "danivwordpressimages" --anonymous-pull-enabled", sale un error, revisar el tipo de SKU del ACR esté en "Standard" y no en "Basic". Si no es el caso, volver a configurar el ACR para este tipo o volver a crear el ACR cambiando el tipo.

3. **Error nombre del ACR**
	
	Las etiquetas en Azure son únicas, esto es, por ejemplo al crear un *ACR* (Azure Container Registry) ponemos de nombre "example" y este ya existe, no va a dejar crearlo. La solución a este error es simple, hay que crear una etiqueta/nombre distinguible para evitar duplicados, por ejemplo "danivrexample".
<div id="appserviceplanproblems"/>

##### Problemas en el plan de servicio de aplicaciones

1. **Error en el nombre del plan**
	
	El plan que se crea con Azure tiene que ser en minúsculas, por ejemplo: "miPlanServicio" **no** sirve pero "miplanservicio" **si**. Sin embargo, para el nombre del grupo de recursos si se pueden utilizar tanto mayúsculas como minúsculas.
<div id="webappproblems"/>

##### Problemas al crear y configurar Web App

1. **Error nombre de WebApp
	
	Mismo error que al crear la ACR, la solución a este error es simple, hay que crear una etiqueta/nombre distinguible para evitar duplicados, por ejemplo "danivrexample".

2. **Error estableciendo una conexión con la base de datos**
	
	Revisar configuración *docker-compose.yml*, comprobar el entorno (environment) de WordPress y MySQL. 
	<div id="img-135"/>
	![](/imgs/img-135.png)
	
	Imagen 1.35 - Error WordPress no se puede conectar con la base de datos

3. **Error de aplicación**
	
	Se produce cuando en el Docker Compose no se pueden obtener las imágenes. La solución es sencilla, revisar correctamente la contraseña que se introduce para vincular el ACR al "Web App".
	<div id="img-136"/>
	![](/imgs/img-136.png)
	
	Imagen 1.36 - Error las imágenes no se pueden obtener
<div id="conclusions"/>

### 4. Conclusiones

Al realizar esta práctica he profundizado bastante sobre estas tecnologías, gracias a ello mis próximos proyectos estarán orientados a utilizar servicios en la nube por sus numerosas ventajas. Por otro lado, el único problema grave que he visto del servicio de Azure son los registros, estos no muestran toda la traza del error y sólo partes que a veces ni tienen que ver con el error. Confío que estos problemas los arreglarán en un futuro, por lo de pronto se debe revisar la configuración antes de realizar ningún despliegue así como sus conexiones y realizar una ejecución en local para prevenir estos errores.
<div id="references"/>

### 5. Referencias

1. [Informática como servicio - Alejandro Puente Castro](https://themvs.github.io/es_fic_muei_ics/ics.html) - Información y documentación
2. [Overview of Docker Desktop | Docker Docs](https://docs.docker.com/desktop/) - Información, documentación y guía de instalación de Docker Desktop<div id="dockerinstalation"/>
3. [DockerHub](https://hub.docker.com/) - Repositorio de imágenes de contenedor público de Docker
4. [MySQL :: MySQL 8.2 Reference](https://dev.mysql.com/doc/refman/8.2/en/) - Información y documentación de MySQL
5. [Documentation - WordPress.org](https://wordpress.org/documentation/) - Información y documentación de WordPress
6. [Quickstart: Create a multi-container app - Azure App Service | Microsoft Learn](https://learn.microsoft.com/en-us/azure/app-service/quickstart-multi-container) Información sobre como desplegar una aplicación en Azure con múltiples contenedores con Docker Compose.
7. [Manage Resource Groups - Azure CLI - Azure Resource Manager | Microsoft Learn](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/manage-resource-groups-cli) Información sobre grupos de recursos en Azure y comandos básicos.
8. [Resource Skus - List - REST API (Azure Compute) | Microsoft Learn](https://learn.microsoft.com/en-us/rest/api/compute/resource-skus/list?view=rest-compute-2021-07-01&tabs=HTTP) Información sobre los tipos de SKUs (Stock Keeping Units) disponibles.
9. [Habilitación del acceso de extracción anónimo - Azure Container Registry | Microsoft Learn](https://learn.microsoft.com/es-es/azure/container-registry/anonymous-pull-access) Información sobre cómo hacer público un ACR como repositorio de imágenes.
10. [Sparkle Effect by Daniel Vicente - CodePen](https://codepen.io/darlich/pen/XWOZWaO) CodePen utilizado para esta práctica.