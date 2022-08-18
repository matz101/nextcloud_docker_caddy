# Setup of Nextcloud with Docker behind Caddy reverse proxy

Note: This is the result of my own journey. The setup as described works for me and comes with no guarantees. There may be better ways to do it.
I'm writing this up, because I found it surprisingly difficult to find descriptions on the web how to do a proper Nextcloud setup with Docker-compose. Maybe it's of help to anyone.

I started from the official nextcloud docker description: https://github.com/docker-library/docs/blob/master/nextcloud/README.md

And the official nextcloud documentation: https://docs.nextcloud.com/server/latest/admin_manual/installation/server_tuning.html

The basic setup is described first. 

An extension with Collabora and Elasticsearch is described below.
 
## Basic setup with MariaDB, Nginx, Redis

### Setting up Nextcloud
1) Create a directory on your server and give it a meaningful name, here I use "my_nextcloud".
The directory will eventually hold the following
- nginx.conf, the configuration file for Nginx
- www.conf,  the PHP configuration file for Nextcloud (optional, but convenient if settings needed)
- docker-compose.yml (of course)
- a symbolic link to the folder where Docker creates it's volumes (optional, but convenient, not included in the sources). The location of the docker volumes may be installation specific. In my case (Ubuntu server) they are located in /var/lib/docker/volumes/ 

2) You will need to create a docker bridge network first with ```docker network create -d bridge mybridge```. "mybridge" is an arbritrary name of the network over which all of the services can communicate.

3) The docker-compose.yml
This is largely inspired by the official documentation of the docker nextcloud image with a bit of cleaning up (no links, no ports): https://github.com/nextcloud/docker

```
version: '3.6'

networks:
  default:
    external: true
    name: mybridge
    
volumes:
  db:
  nextcloud:
  redis:

services:
  redis:
    image: redis
    restart: unless-stopped
    volumes:
      - redis:/data

  db:
    image: mariadb
    restart: unless-stopped
    volumes:
      - db:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=MY_SQL_PASSWORD
      - MYSQL_PASSWORD=MY_SQL_PASSWORD
      - MYSQL_DATABASE=MY_SQL_DATABASE_NAME
      - MYSQL_USER=MY_SQL_USER
      
  nc:
    image: nextcloud:fpm
    volumes:
      - nextcloud:/var/www/html
      - ./www.conf:/usr/local/etc/php-fpm.d/www.conf
    restart: unless-stopped
    environment:
      - MYSQL_PASSWORD=MY_SQL_PASSWORD
      - MYSQL_DATABASE=MY_SQL_DATABASE_NAME
      - MYSQL_USER=MY_SQL_USER
      - MYSQL_HOST=db
      - REDIS_HOST=redis
      - PHP_MEMORY_LIMIT=1G
      - PHP_UPLOAD_LIMIT=8G
      - TRUSTED_PROXIES=172.0.0.0/8
  web:
    image: nginx
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - nextcloud:/var/www/html
    restart: unless-stopped  
```
#### Notes on the docker-compose.yml
- there is no exposed port, that is the services are only visible on the "mybridge" docker network. You may add one, but it will only be accessible via http://, so not safe for external use.
- we'll access the service via our Caddy proxy in the next step
- you can define some variable either in the docker-compose or in the nextcloud config.php file. I like to have the important ones in the compose file.
- you should replace the MY_SQL passwords etc. with your own.
- the TRUSTED_PROXIES is the Docker network address of your Caddy proxy. You may define a narrower scope, if you like, instead of the /8 block used here.

#### notes on the nginx.conf
this is basically taken from the nextcloud docker site again. You will have to replace the php handler with your docker service. In the naming scheme above it will be
```
    upstream php-handler {
        server my_nextcloud_web:9000;
    }
```

#### notes on the www.conf
this is the PHP config. You may want to optimize this based e.g. on the nextcloud documentation: https://docs.nextcloud.com/server/latest/admin_manual/installation/server_tuning.html#tune-php-fpm

#### notes on nextcloud config.php
after the first run, your nextcloud config.php will be created and persisted on the /var/lib/docker/volumes/my_nextcloud_nc volume. You may edit it there.

### Run a Caddy reverse-proxy and make the Nextcloud externally accessible

## Extend the setup with Collabora, Elasticsearch
 
Description to follow   later  on  
another line  