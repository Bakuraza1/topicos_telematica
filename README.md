# jgiraldop-st0263
## info de la materia: st0263 Topicos Especiales en Telematica
#
## Estudiante(s): Julian Giraldo Perez, jgiraldop@eafit.edu.co
#
## Profesor: Edwin Nelson Montoya Munera, emontoya@eafit.edu.co
#

# Reto 3, Wordpress escalable
### Link del proyecto
https://www.devjgp.tech/


# 1. Actividad
En este reto se desarrollo el montaje de una arquitectura con wordpress, la cual cuenta con 5 elementos

- Un servidor Nginx: Servidor encargado de realizar el balanceo de carga sobre los dos servidores de wordpress, ademas este servidor implementa los certificados SSL. Cabe destacar que este servidor es quien esta en los registros del DNS del dominio

- Dos servidores Wordpress: Estos servidores son quienes estan corriendo wordpress, ambos estan conectados tanto al servidor de base de datos como al servidor NFS, desde el cual obtienen los archivos para correr Wordpress.

- Un servidor de base de datos: Este servidor contiene la base de datos Mysql.

- Un servidor NFS: Este servidor es quien se encarga de compartir ciertos archivos a los servidores de wordpress.

## A tener en cuenta
Para el desarrollo de esta actividad se utilizo GCP para instanciar todas las maquinas virtuales, y se hizo uso de docker tanto para los servidores de wordpress como para el servidor de base de datos.

Para los servidores NFS y Nginx se realizo la configuracion directamente en las maquinas virtuales, las instrucciones para la instalacion se encontraran mas adelante.

Ademas para obtener el dominio se utilizo la pagina hostinger, desde la cual tambien se hace el manejo del DNS, en este caso para la configuracion del DNS se hizo un registro A que apunta a la ip del servidor Nginx.

# 1.1 Que aspectos cumplió o desarrolló de la actividad propuesta por el profesor (requerimientos funcionales y no funcionales)
En esta actividad se logro:
- la implementacion en gcp de los 5 elementos anteriormente expuestos, a traves de la dockerizacion y/o configuracion de la vm.

- Implementar la arquitectura planteada en el documento, y las comunicaciones entre maquinas.

- La obtencion del dominio y su configuracion junto a la creacion del certificado SSL.

- En cuanto a los requisitos no funcionales, entre otros, se logro que al correr las maquinas virtuales ya todos los servicios queden instanciados. 


# 1.2 Que aspectos no cumplió o no desarrolló de la actividad propuesta por el profesor (requerimientos funcionales y no funcionales)
De forma general se lograron todos los items propuestos en la actividad.

# 2. Informacion general de diseño de alto nivel y arquitectura

![image](https://user-images.githubusercontent.com/110442546/227290977-4fcd469d-f334-4391-913c-d5ed41dcee1d.png)


La arquitectura del proyecto se basa en 5 nodos tal y como se ve en la imagen (Cajas dentro del cuadrado), cada nodo corresponde a una maquina virtual en gcp.
El cliente ingresara el dominio en su browser, y el browser buscara la IP correspondiente a este dominio, la cual es la ip publica de nuestro nginx server, esto se logra gracias a Hostinger (Plataforma usada para la gestion del DNS). Luego Nginx es quien hara un balanceo de carga sobre los dos servidores de wordpress, los cuales usan un NFS server, y un Db server ubicados en otras instancias para funcionar.

# 3. Descripcion del ambiente se desarrollo y tecnico 
Para el desarrollo de la actividad se utilizo:

- Maquinas virtuales micro de gcp con 10gb de almacenamiento y ubuntu 20.04 

- Docker para correr wordpress y Mysql

- Nginx para realizar la configuracion del balanceo de carga.

- Certbot para la generacion de los certificados Certbot

- Para crear el sistema de archivos compartidos se utilizo NFS, tanto para cliente como para el servidor.

## Compilacion y ejecucion

### Docker

Para instalar docker debe correr en su maquina

        sudo apt update
        sudo apt install docker.io -y
        sudo apt install docker-compose -y
        sudo apt install git -y

        sudo systemctl enable docker
        sudo systemctl start docker
        sudo usermod -a -G docker <username>
        
Para correr un archivo docker-compose correr

        docker-compose -f <nombre de archivo> up

Para la ejecucion de este proyecto se deben crear 5 instancias en gcp, para mayor facilidad se recomienda que se llamen, wordpress-1, wordpress-2, db-server, nfs-server y nginx-server. 

En cada carpeta de este repositorio se encuentran las instrucciones pertintentes de cada tipo de instancia, aun asi se listara a continuacion las configuraciones de cada una.

### Nginx
Configuracion



        1. Instalar nginx en la maquina
                sudo apt update
                sudo apt install nginx

        2. Para configurar nginx copiar y pegar el contenido de nginxconf dentro de etc/nginx/nginx.conf

        3. To create the SSL certificate install certbot
            sudo apt install certbot python3-certbot-nginx

        4. revisar el archivo nginx.conf para asegurarse que el server_name sea el mismo dominio para generar SSL

        5. Para generar SSL correr el comando
            sudo certbot --nginx -d example.com -d www.example.com
    

Nginx.config



    http{
            include mime.types;

            upstream backendserver{ #Specify our servers
                    server 10.128.0.2;
                    server 10.128.0.7;
            }

    server{

                    root /var/www/wordpress;

                    server_name devjgp.tech www.devjgp.tech;

                    index index.php;

                    location / {
                            proxy_pass http://backendserver/; #round robin by default
                    }


        listen 443 ssl; # managed by Certbot
        ssl_certificate /etc/letsencrypt/live/devjgp.tech/fullchain.pem; # managed by Certbot
        ssl_certificate_key /etc/letsencrypt/live/devjgp.tech/privkey.pem; # managed by Certbot
        include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
        ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot


    }

    server{
        if ($host = www.devjgp.tech) {
            return 301 https://$host$request_uri;
        } # managed by Certbot


        if ($host = devjgp.tech) {
            return 301 https://$host$request_uri;
        } # managed by Certbot



                    server_name devjgp.tech www.devjgp.tech;
        listen 80;
        return 404; # managed by Certbot




    }}

    events{}


### Wordpress

Docker-Compose

    version: '3.1'
    services:
    wordpress:
        container_name: wordpress
        image: wordpress
        ports:
        - 80:80
        restart: always
        environment:
        WORDPRESS_DB_HOST: 10.128.0.3 #Replace with ip of server with db 
        WORDPRESS_DB_USER: exampleuser
        WORDPRESS_DB_PASSWORD: examplepass
        WORDPRESS_DB_NAME: exampledb
        volumes:
        - wordpress:/var/www/html
    volumes:
    wordpress:


Config nfs client

    Para configurar las instancias como NFS clients:

    1. Instalar nfs client corriendo 
        sudo apt install nfs-common

    2. Una vez creado el directorio donde se quieren guardar los archivos correr:
        sudo mount <server ip>:<path on the server> <path on client>

    NOTA: Para esto ya debio configurar el NFS server 

### NFS-Server

Para configurar el NFS server:

    1. Instalar el NFS server:
        sudo apt install nfs-kernel-server

    2. Ir al archivo /etc/exports y modificarlo para configurar los archivos a compartir:
        <PATH to share> <client ip>(rw,sync,no_root_squash,no_subtree_check)

    3. Reiniciar el NFS server:
        sudo systemctl restart nfs-kernel-server

### db

    version: '3.1'
    services:
    db:
        image: mysql:5.7
        restart: always
        ports:
        - 3306:3306
        environment:
        MYSQL_DATABASE: exampledb
        MYSQL_USER: exampleuser
        MYSQL_PASSWORD: examplepass
        MYSQL_RANDOM_ROOT_PASSWORD: '1'
        volumes:
        - db:/var/lib/mysql
    volumes:
    db:


### DNS 

    1. Comprar un dominio en hostinger
    2. Crear un A register apuntando a la ip publica del nginx server
    3. Crear certificado con certbot (pasos en nginx)

## Detalles del desarrollo
Para el desarrollo de este reto se comenzo por estudiar un poco sobre gcp para entender como crear instancias y hacer diferentes configuraciones. Posteriormente se comenzo por instanciar solo una maquina virtual, donde estaba todo Wordpress de forma monolitica.

Luego de esto se comenzo a "separar" esta aplicacion monolitica, primero creando dos instancias, una de db y otra de wordpress. A partir de esto fue facil la creacion de una tercera maquina virtual que tambien solo tuviera Wordpress y se conectara de igual forma a la db.

Posterior a instanciar estas 3 maquinas virtuales se creo el servidor NFS, se realizo la configuracion pertitente y posteriromente se configuraron las 2 maquinas de Wordpress como clientes NFS, que consumen el directorio compartido del servidor.

Finalmente en cuanto a las instancias se crea una 5 instancia para el manejo de nginx, en esta se hace toda la configuracion para realizar un round robin entre Wordpress-1 y wordpress-2.

Cuando ya se tuvieron las 5 instancias corriendo y funcionando se consiguio un dominio en hostinger, y se realizo un registro A apuntando a la ip publica del servidor de nginx, de esta forma se completo la configuracion del Dominio, en el momento en que el dominio esta configurado se procede a generar los certificados SSL en el servidor de nginx.

## Detalles tecnicos

Se recomienda crear cada una de las instancais utilizar:
- SO ubuntu 20.04
- Maquina tipo micro 
- Para evitar posibles fallos habilitar comunicacion http y https


## Configuracion y los parametros del proyecto

Todas las configuraciones que se deben aplicar posterior a la creacion de las instancias estan definidas en las secciones anteriores y tambien estan en las carpetas corresponientes de cada instancia dentro de este repositorio.


## ESTRUCTURA DE DIRECTORIOS Y ARCHIVOS IMPORTANTE DEL PROYECTO
## Estructura de directorios y archivos importantes del proyecto
La estructura del directorio cambiara en cada vm debido a las diferentes librerias que se necesitan, pero de forma general cada vm debera clonar este repo, y aplicar las configuraciones que necesite, de esta forma tenemos:

![image](https://user-images.githubusercontent.com/110442546/227306161-ba8f1096-0167-413d-92c4-318190adad6f.png)

# Descripción del ambiente de EJECUCIÓN (en producción)


## IP o nombres de dominio en nube o en la máquina servidor
Para facilitar el acceder a la pagina, se utilizo un dominio el cual esta configurado para que apunte a la ip publica del Nginx-server. 

        devjgp.tech
        www.devjgp.tech

## Como se lanza el servidor.

Despues de lanzar la configuracion inicial simplemente al iniciar las instancias de gcp el usuario ya podra ingresar a la pagina a traves del dominio.

## Guia de como el usuario utilizaria el software o la aplicacion

Para utilizar la aplicacion el usuario simplemente debe ingresar a la url: jgpdev.tech

## Resultados o Pantallazos
![image](https://user-images.githubusercontent.com/110442546/227307679-f4571722-8703-4393-bdc0-2b95beb464e5.png)
![image](https://user-images.githubusercontent.com/110442546/227308776-34055ef8-97a2-4f05-8ff7-54cb6fbbb8af.png)

