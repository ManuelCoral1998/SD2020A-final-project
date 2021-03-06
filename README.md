# SD2020A-final-project
Repository to develop final project

## Equipo de trabajo

Manuel Alejandro Coral Lozano (@ManuelCoral1998)

Juan Sebastián Quintero Yoshioka (@squintero14)

Alejandra Ramos Vélez (@alejandraRamos)

Andrés David Varela López (@dvlopez9811)

## Documentación

### Prerrequisitos

- Tener instalado Docker-compose. Se puede seguir estas [instrucciones de instalación.](https://www.digitalocean.com/community/tutorials/como-instalar-docker-compose-en-ubuntu-18-04-es) </br>


### Desarrollo
Docker Compose es una herramienta para orquestar aplicaciones de múltiples contenedores. Utiliza un archivo YAML para definir e iniciar diferentes contenedores y cualquier relación entre ellos. </br>

Se puede usar el Dockerfile como un bloque de construcción para Docker Compose.</br>

Como se puede evidenciar el en repositorio de Github, se tienen 5 carpetas:</br>

- app-ui: será el front-end de nuestra aplicación.
- proxy: contienen la configuración del proxy en nginx.
- rest-api: será el back-end que se usará en nuestra aplicación.
- test: contiene los test del back/front end.

Cada carpeta contiene un Dockerfile el cual especifica el aprovisionamiento de cada contenedor.

### docker-compose.yml

Primero, se especifica la versión, se utiliza la versión 2.x para disponer de la opción scale, el cual permitirá configurar cuántos contenedores réplicas se tienen del front-end.

Después, se crea una red propia en la cual estarán los microservicios:
```
networks:
  app-network:
    driver: bridge
```
Después, se especifica servicio por servicio.

- mongo:

MongoDB es un programa de base de datos orientado a documentos multiplataforma gratuito y de código abierto. Clasificado como un programa de base de datos NoSQL, MongoDB utiliza documentos similares a JSON con esquemas. MongoDB es desarrollado por MongoDB Inc., y se publica bajo una combinación de la Licencia pública del lado del servidor y la Licencia de Apache.

Para iniciar una instancia de servidor mongo, se utiliza una imagen mongo, donde se le especifica como etiqueta la versión de MongoDB que se va a utilizar: `image: mongo:4.2.6`

Después, se mapea el puerto 27017 del host al puerto 27017 del contenedor: `ports: -27017:27017`.
```
mongo:
  container_name: mongo
  image: mongo:4.2.6
  ports:
    - 27017:27017
  networks: 
    - app-network
```

- rest-api:

[Documentación del proceso de creación y configuración del API-REST y documentación con Swagger](https://github.com/dvlopez9811/SD2020A-final-project/tree/development/rest-api) 

Como el entorno del API-REST se creó en un Dockerfile, se especifica en la carpeta donde se encuentra éste: `build: rest-api`

Se mapea el puerto 3000 del host al puerto 3000 del contenedor: `ports: -3000:3000`.

Se crea una dependencia con el servicio de mongo para que no se cree el api-rest antes que la base de datos: `depends_on: - mongo`
```
rest-api:
    build: rest-api
    ports: 
      - 3000:3000
    depends_on:
      - mongo
    networks:
      - app-network
```
- app-ui:

[Documentación del proceso de creación y configuración del front-end](https://github.com/dvlopez9811/SD2020A-final-project/blob/development/app-ui/readme.MD)

Aquí la configuración es diferente, no se epecifica a qué puerto se mapea del host puesto que se van a tener varios contenedores escuchando por el mismo puerto 3030 sin mapearse a un puerto específico del host, puesto que se accede a ellos mediante el proxy. En este caso, se crean dos contenedores, respondiendo al requerimiento de tener al menos dos réplicas del front-end: `scale: 2`

Se agrega una dependencia, para que no se cree el front-end sin haber creado antes el API-REST: `depends_on: rest-api`
```
app-ui:
    build: app-ui
    scale: 2
    networks: 
      - app-network
    depends_on: 
      - rest-api
```
- proxy:

[Documentación del proceso de creación y configuración del proxy](https://github.com/dvlopez9811/SD2020A-final-project/blob/development/proxy/readme.MD)

Por último, se crea el proxy, no hay configuración adicional que antes no se haya explicado.
```
proxy:
    build: proxy
    ports:
      - 8080:8080
    depends_on: 
      - app-ui
    networks: 
      - app-network
```

- Healthcheck 

### Travis CI

Travis-CI es un sistema de Integración Continua, gratuita para proyectos Open Source y de pago para proyectos privados. Se integra sin problemas con GitHub y automáticamente ejecuta el pipeline definido en cada push o pull requests. Testea y buildea aplicaciones escritas en Ruby, Node, Objective-C, Go, Java, C# y F#, entre otras (que corran en Linux).

1. Para configurar travis en el repositorio, se debe dar permisos a Travis para conectarse con el mismo. Para esto, se va las configuraciones del repositorio, hacer click sobre “Servicios” y seleccionar Travis-CI. Una vez allí seleccionar “Add to GitHub” y otorgar todos los permisos.

2. Una vez en Travis-CI, seleccionar el “+”, buscar el repositorio sobre el cual queremos trabajar y hacer click sobre el switch.

3. En el repositorio se debe crear el archivo .travis.yml. Este archivo debe estar alojado en la raíz del proyecto. Le indicará a Travis-CI lo que debe realizar cada vez que ejecuta un Build.

4. El archivo tiene las siguientes líneas:

- `language: node_js`: Indica el lenguaje de programación.
- `node_js: "8.10.0`: Versión del mismo.

Lo siguiente que se realiza es configurar Docker-compose para poder utilizarlo en los builds, puesto que toda la infraestructura está orquestada por éste:

- Como variable de entorno se guarda la versión deseada para realizar el docker-compose: `DOCKER_COMPOSE_VERSION=1.25.`
- En `before_install` se colocan los comandos para realizar la instalación de docker-compose.
- Una vez realizado esto, se instalan las dependencias necesarias para ejecutar las pruebas: `npm install`
- Por último, se especifica el script a ejecutar, el cual es `docker-compose up` con la opción -d para ejecutarlo en background y `npm test` para ejecutar las pruebas.

Al final, el archvio de configuración quedará así:
`.travis.yml`

```
language: node_js
node_js:
  - "8.10.0"
env:
  - DOCKER_COMPOSE_VERSION=1.25.5
before_install:
  - sudo rm /usr/local/bin/docker-compose
  - curl -L https://github.com/docker/compose/releases/download/${DOCKER_COMPOSE_VERSION}/docker-compose-`uname -s`-`uname -m` > docker-compose
  - chmod +x docker-compose
  - sudo mv docker-compose /usr/local/bin
install:
  - npm install
script:
  - docker-compose up -d
  - npm test
```

### Healthcheck

Por último, en cada uno de los Dockerfile del Proxy, Front-end y REST-API se agrega la instrucción `HEALTCHECK`.

La instrucción HEALTHCHECK le dice a Docker cómo probar un contenedor para verificar que todavía esté funcionando. Esto puede detectar casos como un servidor web que está atascado en un bucle infinito y no puede manejar nuevas conexiones, aunque el proceso del servidor aún se esté ejecutando.

Cuando un contenedor tiene una verificación de salud especificada, tiene un estado de `heal` además de su estado normal. Este estado está en `starting` inicialmente. Cada vez que se aprueba un control de salud, se vuelve `healthy` (en cualquier estado en el que se encontraba anteriormente). Después de un cierto número de fallas consecutivas, se vuelve `unhealthy`.

Admite las siguientes opciones:

--interval = <interval>: el intervalo de tiempo entre dos comprobaciones de estado. El valor predeterminado es 30 segundos.

--timeout = <interval>: el tiempo de espera para ejecutar el comando de comprobación de estado. La comprobación de estado falla si se excede el tiempo de espera. El valor predeterminado es 30 segundos.

--retries = <número de veces>: el estado del contenedor se considera no saludable si la comprobación de estado falla continuamente durante un número específico de veces. El valor predeterminado es 3.

--start-period = <intervalo>: el tiempo de inicialización del inicio de la aplicación. La comprobación de estado fallida durante el inicio no se cuenta. El valor predeterminado es 0 segundos.
  
El comando final queda así:

```
HEALTHCHECK --interval=30s --timeout=30s --start-period=1s --retries=5 \     
  CMD curl -f http://localhost:3000/ || exit 1
```
Para revisar el estado, se ejecuta el siguiente comando: </br>
`docker inspect --format '{{json .State.Health.Status}}' ContainerID`

Además, como la base de datos no tiene un archivo Dockerfile, se configura el healtcheck en el docker-compose.

```
healthcheck:
      test: echo 'db.runCommand("ping").ok' | mongo localhost:27017/ourdb --quiet 
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 40s
```

### Evidencias del funcionamiento

![Docker-compose --version](https://github.com/dvlopez9811/SD2020A-final-project/blob/master/images/00_docker-compose-version.png)

![Docker-compose --build](https://github.com/dvlopez9811/SD2020A-final-project/blob/master/images/01_docker-compose-build.png)

![Docker-compose --ps](https://github.com/dvlopez9811/SD2020A-final-project/blob/master/images/02_docker-compose-ps.png)

![Docker-compose --up](https://github.com/dvlopez9811/SD2020A-final-project/blob/master/images/03_docker-compose-up.png)

![Front-end up](https://github.com/dvlopez9811/SD2020A-final-project/blob/master/images/04_front-end-up.png)

![Proxy funcionando](https://github.com/dvlopez9811/SD2020A-final-project/blob/master/images/05_proxy.png)

![API-REST](https://github.com/dvlopez9811/SD2020A-final-project/blob/master/images/06_rest-api.png)

![Insertando valores](https://github.com/dvlopez9811/SD2020A-final-project/blob/master/images/07_insertando-valores.png)

![Healthcheck](https://github.com/dvlopez9811/SD2020A-final-project/blob/master/images/08_healthcheck.png)

![Travis](https://github.com/dvlopez9811/SD2020A-final-project/blob/master/images/09_travis.png)

![Build complete](https://github.com/dvlopez9811/SD2020A-final-project/blob/master/images/10_build-complete.png)

### Problemas encontrados y acciones efectuadas para su solución

### Información construida con base en:
- https://github.com/ofstudio/docker-compose-scale-example/blob/master/docker-compose.yaml
- https://cloudbuilder.in/blogs/2017/11/26/docker-compose-nginx-nodejs/
- https://www.youtube.com/watch?v=BSKox1DEsQo
- https://hub.docker.com/_/mongo
- https://www.federico-toledo.com/travis-ci-para-integracion-continua/
- https://docs.docker.com/engine/reference/builder/
- https://www.alibabacloud.com/help/doc-detail/58588.htm
