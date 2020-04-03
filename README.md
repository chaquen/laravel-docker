## Ejemplo Laravel con Docker

Este proyecto tiene como finalidad servir de guia para crear un proyecto de laravel con Docker
fue tomado del [tutorial](https://www.youtube.com/watch?v=5N6gTVCG_rw) 

## Como crear proyecto

Cree los archivos ***docker-compose.yml***, y ***Dockerfile***, ***default.conf***

### docker-compose.yml
    version: "3"

    networks: 
        laravel:

    services: 
        nginx:
            image: nginx:stable-alpine
            container_name: nginx
            ports: 
                -   "8088:80"
            volumes: 
                - ./src:/var/www/html
                - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
            depends_on: 
                - php
                - mysql 
            networks: 
                -   laravel
        mysql:
            image: mysql:5.7.22
            container_name: mysql
            restart: unless-stopped
            tty: true
            ports: 
                - "4306:3306"
            volumes: 
                - ./mysql:/var/lib/mysql
            environment: 
                MYSQL_DATABASE: homestead
                MYSQL_USER: homestead
                MYSQL_PASSWORD: secret
                MYSQL_ROOT_PASSWORD: secret
                SERVICE_TAGS: dev
                SERVICE_NAME: mysql            
            networks: 
                -   laravel
        php:
             build:     
                context: .
                dockerfile: Dockerfile
             container_name: php
             volumes: 
                - ./src:/var/www/html
             ports: 
                - "9000:9000"
             networks: 
                -   laravel

### Dockerfile

    FROM php:7.2-fpm-alpine

    RUN docker-php-ext-install pdo pdo_mysql

### default.conf

    server {
        listen 80;
        index index.php index.html;
        server_name localhost;
        error_log /var/log/nginx/error.log;
        access_log /var/log/nginx/access.log;
        root /var/www/html/public;

        location / {
            try_files $uri $uri/ /index.php?$query_string; 
        }

        location ~ \.php$ {
            try_files $uri =404;
            fastcgi_split_path_info ^(.+\.php)(/.+)$;
            fastcgi_pass php:9000;
            fastcgi_index index.php;
            include fastcgi_params;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            fastcgi_param PATH_INFO $fastcgi_path_info;
        } 
    }
## Estructura de carpetas

    /mysql (esta carpeta se ignoro en el repositorio debe crearse)
    /nginx
        -default.conf
    /src
        -Estructura de laravel
    docker-compose.yml
    Dockerfile

### Comandos usados
Aqui se listan algunos comandos utiles para crear el proyecto, deben ejecutarse en la raiz del proyecto

-  Construir la imagen 

        docker-compose build --no-cache 

- Crear los contenedores

        docker-compose up -d

- Ver los logs de el proceso de creación  

        docker-compose logs

- Apagar los contenedores   

        docker-compose down

- Ejecutar comando en los contenedores

        docker-compose exec nombre_del_contenedor comando a ejectuar

- Para acceder a nginx 

        docker-compose exec nginx sh

### Instalar laravel
Debes acceder a la carpeta src, y ejectura el instalador de laravel en este caso se usara el comando para instalar la ultima versión de laravel para cuando se hizo este proyecto^ 7.0
    
    cd src
    composer create-project laravel/laravel .
### Crear migraciones
Para ello dede editar el archivo .env, agregando los datos de mysql o el motor de datos que desee usar, de acuerdo a la configuración de su archivo ***docker-compose.yml***

    DB_CONNECTION=mysql
    DB_HOST=mysql
    DB_PORT=3306
    DB_DATABASE=homestead
    DB_USERNAME=homestead
    DB_PASSWORD=secret

Luego de ello ejecute el comando, que le permitira ejecutar las migraciones

    docker-compose exec php php /var/www/html/artisan migrate


Ahora puede acceder a su servidor [local](http://localhost:8088) 

### Nota 
Si te aparece el error de permisos linux debes agregar los permiso

    chmod -R 777 storage
    cd storage  
    chmod -R logs

Para acceder a Mysql 

    docker exec -it mysql /bin/sh -c "[ -e /bin/bash ] && /bin/bash || /bin/sh"
    mysql -h localhost -u root -p


##  laravel [Passport](https://laravel.com/docs/7.x/passport)    
Instalar via composer

    composer require laravel/passport

Generar la migración y la clave para encriptar 

    #php artisan passport:install
    docker-compose exec php php /var/www/html/artisan migrate
    docker-compose exec php php /var/www/html/artisan passport:install
    

Agregar el paquete a la clase **App/User.php**

    use Laravel\Passport\HasApiTokens;
    
    class User extends Authenticatable
    {
        use HasApiTokens, Notifiable;
    }

Agregar **Passport::route** al **AuthServiceProvider**

    class AuthServiceProvider extends ServiceProvider
    {
        /**
         * The policy mappings for the application.
         *
         * @var array
         */
        protected $policies = [
            'App\Model' => 'App\Policies\ModelPolicy',
        ];

        /**
         * Register any authentication / authorization services.
         *
         * @return void
         */
        public function boot()
        {
            $this->registerPolicies();

            Passport::routes();
        }
    }

Finalmente, en el archivo **config/auth.php**, agregamos al **driver** de el **guard** **api** el valor de 
**passport**


    'guards' => [
        'web' => [
            'driver' => 'session',
            'provider' => 'users',
        ],

        'api' => [
            'driver' => 'passport',
            'provider' => 'users',
        ],
    ],


 Ver las rutas creadas
        
        docker-compose exec php php /var/www/html/artisan route:list   