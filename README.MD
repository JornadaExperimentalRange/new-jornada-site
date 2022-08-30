# Jornada site update

Containerized setup for the Jornada site update using docker/docker-compose and bitnami's rootless image

## Features

- rootless containers prevent privilege escalation for any user that gains access to the inside of the container
- 512MB Max file size for uploads 
- docker-compose orchestrates website (bitnami wordpress container), and database (bitnami mysql container)


## Installation

Setup requires [docker] to run.

After installing docker, navigate to the directory where the docker compose .yml file is located, and deploy the containers using the following:

```sh
docker-compose up -d
```

Check if both containers are running successfully by:

```sh
docker container ls
```

or/and checking the logs (substitute ##### with container name or container id):

```sh
docker container log #####
```

If both containers are running successfully, altering the default max file size for bitnami images requires editing of the php.ini file within the container. First, docker cp will copy the default php.ini outside of the container. Replace ##### with docker container name or id in the following lines:

```sh
docker cp #####:/opt/bitnami/php/etc/php.ini ./php.ini
```

Modify the php.ini file using the text editor of your choice:
```sh
vim ./php.ini
```

Within the php.ini, the settings that control file size upload are the following (in this example, they will be set to 512MB):
```text
upload_max_filesize 512M
post_max_size 512M
```

Copy the modified php.ini file back into the running container, and restart container. (Replace ##### with docker container name or id): 

```sh
docker cp ./php.ini #####:/opt/bitnami/php/etc/php.ini 
docker container stop #####
docker-compose up -d
```

The final line should restart only the stopped container, and php.ini settings in wordpress will be updated once the site restarts. 

## Additional set up

This documentation assumes the domain host for the site is correctly set up. In order to deploy to a remote server, additional set up is required using

- NGINX
- Certbot


### NGINX configuration 

Within the sites-available directory within the default nginx install location (/etc/nginx), create a configuration file for the site and then modify with your text editor of choice (editing within NGINX directories may require use of SUDO privileges): 

```sh
cd /etc/nginx/sites-available
sudo touch newjornada.conf
sudo vim newjornada.conf
```

Add a server block to allow nginx to route traffic from the containers port to the web:
```
server{
    server_name test.jornada.nmsu.edu;

    access_log /var/log/nginx/newjornada-access.log;
    error_log /var/log/nginx/newjornada-error.log debug;

    location / {
        include  /etc/nginx/mime.types;
        # allowing upload of large files through nginx (in addition to wordpress settings)
        client_max_body_size 512M;
        
        # security headers
        proxy_pass http://localhost:8015;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        include common.conf; 
    }

    # for adminer/phpmyadmin visit /php route
    location ~ \/php {
        rewrite ^/php(/.*)$ $1 break;
        proxy_pass http://localhost:8104;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $remote_addr;

        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Host $host;
    }

    # certbot block for 443 handling

    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/new-jornada/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/new-jornada/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
}

server {
    if ($host = test.jornada.nmsu.edu) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


    server_name aim.landscapetoolbox.org;
    listen 80;
    return 404; # managed by Certbot
}

```

Additional settings within common.conf one level above the site configuration file (within /etc/nginx, not /etc/nginx/sites-available):
```
# HTTP STRICT TRANSPORT SECURITY
    add_header Strict-Transport-Security 'max-age=31536000; includeSubDomains; preload';
# X-XSS
    add_header X-XSS-Protection "1; mode=block";
# X-FRAME- OPTIONS
    add_header X-Frame-Options "SAMEORIGIN";
# X-CONTENT TYPE
    add_header X-Content-Type-Options nosniff;
# REFERRER POLICY
    add_header Referrer-Policy "strict-origin";
# PERMISSIONS POLICY
    add_header Permissions-Policy "geolocation=(),midi=(),sync-xhr=(),microphone=(),camera=(),magnetometer=(),gyroscope=(),fullscreen=(self),payment=()";

```

Enable the serving of the site by creating a symbolic link between the configuration file within the _sites-available_ directory and the _sites-enabled_ directory. Restart nginx after creating the link:

```sh
sudo ln -s /etc/nginx/sites-available/newjornada.conf /etc/nginx/sites-enabled
sudo systemctl restart nginx
```

### Certbot configuration 


## License

MIT


[//]: # (reference links)

   [docker]: <https://docs.docker.com/get-docker>
   
