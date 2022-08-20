# Setup of Nextcloud with Docker behind Caddy reverse proxy

Note: This is the result of my own journey. The setup as described works for me. It comes with no guarantees whatsoever. There may be better ways to do it, too.
I'm writing this up, because I found it surprisingly difficult to find descriptions on the web how to do a proper Nextcloud setup with Docker-compose. Maybe it's of help to anyone.

I started from the official nextcloud docker description: https://github.com/docker-library/docs/blob/master/nextcloud/README.md

And the official nextcloud documentation: https://docs.nextcloud.com/server/latest/admin_manual/installation/server_tuning.html

The basic setup is described first. 

An extension with Collabora and Elasticsearch is described below.
 
## Basic setup with MariaDB, Nginx, Redis

### Setting up Nextcloud
#### 1) Create a docker bridge network 
On your server use ```docker network create -d bridge mybridge```. 

"mybridge" is an arbritrary name of the network over which all of the services can communicate.

#### 2) Create a project directory on your server 
and give it a meaningful name, here I use "next".

The directory will eventually hold the following
- nginx.conf, the configuration file for Nginx
- www.conf,  the PHP configuration file for Nextcloud (optional, but convenient if settings needed)
- docker-compose.yml (of course)
- a symbolic link to the folder where Docker creates its volumes (optional, but convenient to access everything from one location). The location of the docker volumes may be installation specific. In my case (Ubuntu server) its  ` ln -s /var/lib/docker/volumes/ ./link_to_docker_volumes`


#### 3) The docker-compose.yml
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
    restart: unless-stopped
    volumes:
      - nextcloud:/var/www/html
      - ./www.conf:/usr/local/etc/php-fpm.d/www.conf
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
    restart: unless-stopped  
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - nextcloud:/var/www/html
```
*Notes on the docker-compose.yml*

- Ports: there is no exposed port, that means the services are only visible on the "mybridge" docker network. You may add a port, but it will only be accessible via http://, so not safe for external use. We'll access the service via our Caddy proxy in the next step.
- Container names. The automatic naming scheme for containers is inconsistent between the old 'docker-compose' and the new 'docker compose'.
the new Docker Compose will name the container [projectfolder]-[service]-[number] (used here), whereas the older Docker-compose will name them with an underscore `_` instead of the hyphen `-`. You may want to assign explicit names with `container_name: mycontainername` 
- Volume names. The Docker volumes are created in a installation specific directory (in my case /var/lib/docker/volumes/) and are auto-named <projectfolder>_<volumename>
- Environment variables. 
you may want to replace the MY_SQL passwords etc. with your own.
You can define some of the Nextcloud config variables either in the docker-compose or in the nextcloud config.php file. I like to have the important ones in the compose file.
- the TRUSTED_PROXIES is the Docker network address of your Caddy proxy. You may define a narrower scope, if you like, instead of the /8 block used here.

#### 4) nginx.conf
This is basically taken from the Nextcloud Docker site. You will have to replace the php handler with your respective Nextcloud Docker service. In the naming scheme above it will be
```
    upstream php-handler {
        server next-nc-1:9000;
    }
```

#### 5) www.conf
This is the general PHP config file. You may want to optimize this based e.g. on the nextcloud documentation: https://docs.nextcloud.com/server/latest/admin_manual/installation/server_tuning.html#tune-php-fpm


#### 6) Start the stack 
From your project folder containing the docker-compose.yml with ` docker compose up -d` or, depending on your installation, ` docker-compose up -d`

#### 7) Nextcloud config.php
After the first run, your nextcloud config.php will be created, pre-populated and persisted on the /var/lib/docker/volumes/next_nextcloud volume. You may edit it there.

#### 8) Add cron 
You might prefer a pure Docker solution and startup a dedicated container just for the cronjob. As cron is already running on the server I decided to go the easy route and add the following line with `crontab -e`
```crontab
*/5 * * * * docker exec -u 33 next-nc-1 php cron.php
``` 
Short vi editor instruction: type `i` to go to insert-mode. Type or copy/paste above line. `ESC` to leave insert mode. `:wq` to write and quit.

### Run a Caddy reverse-proxy and make the Nextcloud externally accessible
We will now start a reverse-proxy. When there is a request for Nextcloud service the reverse-proxy will fetch that information and pass it to the requester. The traffic between Caddy and Nextcloud is via simple HTTP on the docker network, which is just fine, if both are on the same server. To the outside the traffic is HTTPS encrypted. 
The nice thing about Caddy is, that it requires very little configuration and it retrieves and renews certificates for your domain automagically!

#### 1) Prerequisites:
- you need a domain name that you own, say "mydomain.tld" 
- you need to point the A records (and if you want to use IPv6,  AAAA records) of your domain to the IP of your server. Note that this will take some hours to spread to all DNS-servers.

#### 2) Create a project directory.
I'm using an independent folder and docker-compose.yml for Caddy as Caddy may serve other services as well.

Create a directory on your server and give it a meaningful name, here I simply use "caddy".

The directory will eventually hold the following
- the "Caddyfile", the configuration for Caddy
- the "docker-compose.yml"
- a directory "srv", (optional, would hold a website if you like to use Caddy as a webserver too)

#### 3) The docker-compose.yml for Caddy
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
Note that the Caddy container is on the "mybridge" network, just as the Nextcloud containers. Two volumes are created to persist data and the Caddyfile and the srv directory are bind-mounted.
Caddy exposes ports 80 and 443 on your server.

#### 4) Caddyfile

The Caddy configuration could be as simple as
```
mydomain.tld {
    reverse_proxy next-web-1
}
```
and Nextcloud would run with it. 

Remember to change mydomain.tld to your own and check the naming of the Nextcloud web server container (here: next-web-1). 

With this Caddyfile however Nextcloud will give you warnings in the administrator section. To avoid those we extend the Caddyfile as follows:

```
mydomain.tld {
	header Strict-Transport-Security max-age=31536000
	rewrite /.well-known/carddav /remote.php/dav
	rewrite /.well-known/caldav /remote.php/dav
  rewrite /.well-known/webfinger /index.php/.well-known/webfinger
  rewrite /.well-known/nodeinfo /index.php/.well-known/nodeinfo
  reverse_proxy next-web-1
}
```

Startup with docker compose up -d. 

Point your browser to your domain and start installing Nextcloud!

## Extend the setup with Collabora, Elasticsearch
 
### Collabora CODE
I will not use the built-in CODE-server, rather set up a separate container, that can be used by several Nextcloud instances.
Documentation can be found here: https://sdk.collaboraonline.com/docs/installation/CODE_Docker_image.html

#### 0) Prerequisite
Collabora will be externally available (I have not found another way to do it).
Define a subdomain that points to your server, say collabora.mydomain.tld.
When set-up there is an admin GUI to your Collabora install: collabora.mydomain.tld/browser/dist/admin/admin.html. Use your password from the docker-compose.yml.

#### 1) Setup a project directory
I simply call it `collabora` and it will contain the docker-compose.yml

#### 2) The docker-compose.yml

```yaml
version: '3.6'

networks:    
  default:
    external: true
    name: mybridge

services:
  collabora:
    image: collabora/code
    restart: unless-stopped
    container_name: collabora
    cap_add:
      - MKNOD
    expose:
      - 9980
    environment:
      - aliasgroup1=https://mydomain1.tld
      - aliasgroup2=https://mydomain2.tld
      - username=YourUserName    
      - password=YourStrongPassword!   
      - "extra_params=--o:ssl.enable=false --o:ssl.termination=true"
```

I used an explicit container_name for brevity and clarity. 
Port 9980 needs to be exposed internally on the Docker network but not on the host.
MKNOD is needed and extra_params force the container to talk http to our Caddy reverse proxy.
aliasgroup1 and aliasgroup2 define domains that can access the service, in this case your Nextcloud installation.

#### 3) Startup
docker compose up -d

#### 4) Adapt Caddyfile
add the following lines to your Caddyfile
```
collabora.mydomain.tld {
  encode gzip
  reverse_proxy http://collabora:9980
}
```
Restart Caddy from your caddy directory `docker compose down`, `docker compose up -d`

#### 5) Make Nextcloud Docker talk to your Collabora Docker
Install app "Nextcloud Office"
In the adminstration panel find "Office" and add under "use your own server" the following : `https://collabora.yourdomain.tld`
If Nextcloud can connect to your collabora server you should get the green checkmark and are good to try it out with some office document.

### Full text search with Elasticsearch
Again, I'm using this for more than one Nextcloud instance so it is in a separate project folder.
Documentation on Full Text Search can be found here: https://github.com/nextcloud/fulltextsearch/wiki/Commands


#### 1) Set up project folder
Call it e.g. "elastic"

It will contain
- docker-compose.yml
- Dockerfile

#### 2) Docker-compose.yml
```yaml
version: '3.6'

networks:
  default:
    external: true
    name: mybridge
    
volumes:
  data:

services:
  elastic:
    build:
      context: .
    container_name: elastic
    restart: unless-stopped
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false 
    volumes:
      - data:/usr/share/elasticsearch/data
```
This one builds the image from the official Elasticsearch image and extends it by the required "ingest-attachment". This is done by the build: directive in the docker-compose.yml which will execute the Dockerfile and build the extended image.
```Dockerfile
FROM elasticsearch:7.12.0
RUN bin/elasticsearch-plugin install  --batch ingest-attachment
```
I used version 7.12.0 because the newest version did not work for me (or Nextcloud)

Start with docker compose up -d

#### 3) Connect with Nextcloud
Install the following Apps 
- "Full text search" and 
- "Full text search - Files" and  
- "Full text search - ElasticSearch Platform"

In the administration panel look for full text search and write as "address of servlet" `http://elastic:9200`. Choose some name for your index. 

#### 4) Test install and build the initial index
Test:
docker exec -u 33 -it next-nc-1 php occ fulltextsearch:test

Build first index:
docker exec -u 33 -it next-nc-1 php occ fulltextsearch:index

Update with each file change:
docker exec -u 33 -it next-nc-1 php occ fulltextsearch:live

