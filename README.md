# Setup of Nextcloud with Docker behind Caddy reverse proxy

Note: This is the result of my own journey. It may not be optimized, it works for me and comes with no guarantees. 
I'm writing this up, because I found it surprisingly difficult to find descriptions on the web how to do a proper Nextcloud setup with Docker-compose. Maybe it's of help to anyone.

The basic setup is described first. 
An extension with Collabora and Elasticsearch is described below.
 



## Basic setup with MariaDB, Nginx, Redis

### Setting up Nextcloud
1) Create a directory on your server and give it a meaningful name, here I use "mynextcloud".
The directory will eventually hold the following
- nginx.conf, the configuration file for Nginx
- www.conf,  the PHP configuration file for Nextcloud (optional, but convenient if settings needed)
- a symbolic link to the folder where Docker creates it' volumes (optional, but convenient,)
- docker-compose.yml (of course)

2) The docker-compose.yml

### Run a Caddy reverse-proxy and make the Nextcloud externally accessible

## Extended with Collabora, Elasticsearch
