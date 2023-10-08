# Guía de Configuración de Servidor Web

Esta guía fue realizada por **Sorac**.

## Redes Sociales

- [Twitch](https://www.twitch.tv/elsorac_)
- [YouTube](https://www.youtube.com/c/sorac)
- [Discord](https://discord.gg/V4gq2p6MfR)

## Instalación de NGINX

Para instalar NGINX, sigue estos pasos:

1. Ejecuta el siguiente comando:

    ```
    apt install nginx
    ```

2. Habilita el servicio de NGINX:

    ```
    systemctl enable nginx && systemctl start nginx
    ```

## Instalación de MariaDB

1. Instala MariaDB con los siguientes comandos:

    ```
    sudo apt install mariadb-server mariadb-client
    sudo systemctl enable mariadb && sudo systemctl start mariadb
    ```

2. Configura MariaDB ejecutando:

    ```
    sudo mysql_secure_installation
    ```

3. Abre el archivo de configuración de MariaDB:

    ```
    nano /etc/mysql/my.cnf
    ```

4. Agrega la siguiente línea en el archivo:

    ```
    [mysqld]
    bind-address=0.0.0.0
    ```

5. Reinicia MariaDB:

    ```
    systemctl restart mariadb
    ```

6. Accede a la consola de MySQL:

    ```
    mysql -u root -p
    ```

7. Crea un usuario y otorga permisos:

    ```
    CREATE USER 'USUARIO_MYSQL'@'%' IDENTIFIED BY 'CLAVE_USUARIO_MYSQL';
    GRANT ALL PRIVILEGES ON *.* TO 'soracdev'@'%' WITH GRANT OPTION;
    FLUSH PRIVILEGES;
    ```

## Instalación de PHP 8.1

Para instalar PHP 8.1, sigue estos pasos:

1. Actualiza el sistema y agrega el repositorio PHP:

    ```
    sudo apt update
    sudo apt install lsb-release ca-certificates apt-transport-https software-properties-common -y
    sudo add-apt-repository ppa:ondrej/php
    ```

2. Instala PHP 8.1 y sus módulos:

    ```
    sudo apt install php8.1 php8.1-{cli,common,gd,mysql,mbstring,bcmath,xml,fpm,curl,zip}
    ```

## Instalación de PHPMyAdmin

Para instalar PHPMyAdmin, sigue estos pasos:

1. Descarga los archivos de PHPMyAdmin:

    ```
    wget https://files.phpmyadmin.net/phpMyAdmin/5.2.1/phpMyAdmin-5.2.1-all-languages.zip
    ```

2. Instala `unzip` en tu servidor:

    ```
    apt install unzip -y
    ```

3. Descomprime el archivo de PHPMyAdmin:

    ```
    unzip phpMyAdmin-5.*-all-languages.zip
    ```

4. Elimina el archivo antiguo de PHPMyAdmin:

    ```
    rm phpMyAdmin-5.*-all-languages.zip
    ```

5. Crea un nuevo directorio para PHPMyAdmin:

    ```
    mkdir /var/www/phpmyadmin/
    ```

6. Mueve todos los archivos al nuevo directorio:

    ```
    mv phpMyAdmin-5.*-all-languages/* /var/www/phpmyadmin/
    ```

7. Elimina el archivo antiguo de PHPMyAdmin:

    ```
    rm -rf phpMyAdmin-5.*-all-languages
    ```

8. Accede al directorio de PHPMyAdmin:

    ```
    cd /var/www/phpmyadmin/
    ```

9. Cambia el nombre del archivo de configuración de ejemplo:

    ```
    mv config.sample.inc.php config.inc.php
    ```

10. Edita la configuración de PHPMyAdmin:

    ```
    nano config.inc.php
    ```

11. Cambia el secreto de Blowfish con la nueva clave y agrega la siguiente línea:

    ```php
    $cfg['TempDir'] = '/tmp/';
    ```

12. Instala Certbot para obtener el certificado:

    ```
    apt install certbot -y
    ```

13. Detén todas las sesiones de NGINX:

    ```
    systemctl stop nginx
    ```

14. Obtén el certificado para tu dominio:

    ```
    certbot certonly -d tudominio.com
    ```

15. Configura la configuración de PHPMyAdmin en NGINX:

    ```
    nano /etc/nginx/sites-available/phpmyadmin.conf
    ```

16. Pega la siguiente configuración:
    
## Recuerda cambiar "tudominio.com" por el que configuraste con cerbot

```nginx
server {
    listen 80;
    server_name tudominio.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name tudominio.com;

    root /var/www/phpmyadmin;
    index index.php;

    access_log /var/log/nginx/tudominio.app-access.log;
    error_log  /var/log/nginx/tudominio.app-error.log error;

    # allow larger file uploads and longer script runtimes
    client_max_body_size 100m;
    client_body_timeout 120s;

    sendfile off;

    # SSL Configuration
    ssl_certificate /etc/letsencrypt/live/tudominio.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/tudominio.com/privkey.pem;
    ssl_session_cache shared:SSL:10m;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384";
    ssl_prefer_server_ciphers on;

    # See https://hstspreload.org/ before uncommenting the line below.
    # add_header Strict-Transport-Security "max-age=15768000; preload;";
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    add_header X-Robots-Tag none;
    add_header Content-Security-Policy "frame-ancestors 'self'";
    add_header X-Frame-Options DENY;
    add_header Referrer-Policy same-origin;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:/run/php/php8.1-fpm.sock;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param PHP_VALUE "upload_max_filesize = 500M \n post_max_size=500M";
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param HTTP_PROXY "";
        fastcgi_intercept_errors off;
        fastcgi_buffer_size 16k;
        fastcgi_buffers 4 16k;
        fastcgi_connect_timeout 300;
        fastcgi_send_timeout 300;
        fastcgi_read_timeout 300;
        include /etc/nginx/fastcgi_params;
    }

    location ~ /\.ht {
        deny all;
    }
}
```
17. Habilita la configuración para NGINX:

    ```
    ln -s /etc/nginx/sites-available/phpmyadmin.conf /etc/nginx/sites-enabled/phpmyadmin.conf
    ```

18. Reinicia NGINX:

    ```
    systemctl restart nginx
    ```

Esperamos que esta guía te ayude a configurar tu servidor web correctamente.
