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
- a symbolic link to the folder where Docker creates it's volumes (optional, but convenient, not included in the sources)

2) The docker-compose.yml
```
version: '3.6'
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
      - MYSQL_ROOT_PASSWORD=Langeoog1#
      - MYSQL_PASSWORD=Langeoog1#
      - MYSQL_DATABASE=dbKLAUDnc
      - MYSQL_USER=ncadmin
      
  nc:
    image: nextcloud:fpm
    volumes:
      - nextcloud:/var/www/html
      - ./www.conf:/usr/local/etc/php-fpm.d/www.conf
    restart: unless-stopped
    environment:
      - MYSQL_PASSWORD=Langeoog1#
      - MYSQL_DATABASE=dbKLAUDnc
      - MYSQL_USER=ncadmin
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
    
networks:
  default:
    external: true
    name: mybridge
    
volumes:
  db:
  nextcloud:
  redis:
```
### Run a Caddy reverse-proxy and make the Nextcloud externally accessible

## Extend the setup with Collabora, Elasticsearch
 
Description to follow   later  on  
another line  