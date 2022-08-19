# Setup of Nextcloud with Docker behind Caddy reverse proxy

Note: This is the result of my own journey. The setup as described works for me. It comes with no guarantees whatsoever. There may be better ways to do it, too.
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
- a symbolic link to the folder where Docker creates its volumes (optional, but convenient to access everything from one location). The location of the docker volumes may be installation specific. In my case (Ubuntu server) its  ln -s /var/lib/docker/volumes/ ./link_to_docker_volumes 

2) You will need to create a docker bridge network first with ```docker network create -d bridge mybridge```. "mybridge" is an arbritrary name of the network over which all of the services can communicate.

3) The docker-compose.yml
This is largely inspired by the official documentation of the docker nextcloud image with a bit of cleaning up (no links, no ports): https://github.com/nextcloud/docker

```yaml
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
- you should replace the MY_SQL passwords etc. with your own.
- there is no exposed port, that is the services are only visible on the "mybridge" docker network. You may add a port, but it will only be accessible via http://, so not safe for external use.
- we'll access the service via our Caddy proxy in the next step
- you can define some of the config variables either in the docker-compose or in the nextcloud config.php file. I like to have the important ones in the compose file.
- the TRUSTED_PROXIES is the Docker network address of your Caddy proxy. You may define a narrower scope, if you like, instead of the /8 block used here.

#### Notes on the nginx.conf
this is basically taken from the nextcloud docker site again. You will have to replace the php handler with your docker service. In the naming scheme above it will be
```
    upstream php-handler {
        server my_nextcloud_nc:9000;
    }
```



#### Notes on the www.conf
this is the general PHP config file. You may want to optimize this based e.g. on the nextcloud documentation: https://docs.nextcloud.com/server/latest/admin_manual/installation/server_tuning.html#tune-php-fpm

#### Notes on nextcloud config.php
after the first run, your nextcloud config.php will be created and persisted on the /var/lib/docker/volumes/my_nextcloud_nc volume. You may edit it there.

4) start the stack 
as usual, from your project folder containing the docker-compose.yml with ``` docker-compose up -d```


### Run a Caddy reverse-proxy and make the Nextcloud externally accessible
We will now start a reverse-proxy. When there is a request for Nextcloud service the reverse-proxy will fetch that information and pass it to the requester. The traffic between Caddy and Nextcloud is via simple HTTP on the docker network, which is just fine, if both are on the same server. To the outside the traffic is HTTPS encrypted. 
The nice thing about Caddy is, that it requires very little configuration and it retrieves and renews certificates for your domain automagically!

#### Prerequisites:
- you need a domain name that you own, say "mydomain.tld" 
- you need to point the A and AAAA records of your domain to the IP of your server

#### Setup the Caddy container.

I'm using an independent folder and docker-compose.yml for Caddy as Caddy may serve other services as well.

1) Create a directory on your server and give it a meaningful name, here I simply use "caddy".
The directory will eventually hold the following
- the "Caddyfile", the configuration for Caddy
- the "docker-compose.yml"
- a directory "srv", (optional, would hold a website if you like to use Caddy as a webserver too)

2) The docker-compose.yml for Caddy
There is a good description at https://hub.docker.com/_/caddy
I started from that yml and simplified a bit.

```yaml
version: "3.7"

networks:
  default:
    external: true
    name: mybridge

volumes:
  data:
  config:
  
services:
  caddy:
    image: caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
   
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - ./srv:/srv
      - data:/data
      - config:/config
```
Note that the Caddy container is on the "mybridge" network. Two volumes are created to persist data and the Caddyfile and the srv directory are bind-mounted.
Caddy exposes ports 80 and 443 on your server.

3) Caddyfile

This could be extremely simple (that's Caddy: a reverse proxy in three lines!)
```
mydomain.tld {
    reverse_proxy my_nextcloud_web
}
```
and Nextcloud will run with it. However Nextcloud will give you errors in the administrator section. To avoid those we extend the Caddyfile as follows:

```
mydomain.tld {
	header Strict-Transport-Security max-age=31536000
	rewrite /.well-known/carddav /remote.php/dav
	rewrite /.well-known/caldav /remote.php/dav
    rewrite /.well-known/webfinger /index.php/.well-known/webfinger
    rewrite /.well-known/nodeinfo /index.php/.well-known/nodeinfo
    reverse_proxy my_nextcloud_web
}
```

Startup with docker-compose up -d. 
Enjoy a safe Nextcloud installation!


## Extend the setup with Collabora, Elasticsearch
 
Description to follow   later  on  
another line  