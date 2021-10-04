# Deploying an application stack to the swarm
[![Build Status](https://travis-ci.org/joemccann/dillinger.svg?branch=master)]()
### (Docker-Compose + Docker-Swarm + Application Stack+ ALB)
## Description:
I going to demonstrate an application stack deployment into a docker swarm, where the application data have been mounted through an NFS server. 
Here, the docker swarm is configured with a manager node and 3 worker nodes.A compose file has been written for the PHP and Nginx services. The Nginx acts as webserver for the website and PHP request is served by the PHP-FPM services. Both the containers have the same bind mounting volumes where the volume is actually mounted with the NFS server for serving the data for the containers. 
The application(here a php website) will be updated in the NFS server, from where it will be fetched by the containers automatically. A docker swarm overlay network has been made use for the whole process since it acts as a cluster.

 
## Pre-requesites:

- Basic Knowledge in Docker, Docker swarm, AWS
- 4 AWS instances, where one act initialized as the swarm manager and 3 as workers. All these are connected to an ALB

## Features:

- Multi-host networking using overlay network.
- Easily scalable.
 - Integartion of an ALB for easy load balancing.

 How to Install:
 
 Click to know more about Installation:
 1) [Docker](https://docs.docker.com/engine/install/)
 2) [Docker-Compose](https://docs.docker.com/compose/install/)
 3) [NFS configuration](https://www.tecmint.com/how-to-setup-nfs-server-in-linux/)
 
 ## Architecture:
![
alt_txt
](https://github.com/anandg1/docker-swarm-app/blob/main/Architecture.jpg)
 ## Code:
 
 Docker-compose file
 ```sh
 version: '3.2'
services:

  php:
    image: php:fpm-alpine3.13
    restart: always
    container_name: php
    volumes:
      - /var/nfs:/var/www/html
    networks:
      - mynet
    deploy:
      replicas: 3
      placement:
        constraints:
          - "node.role == worker"
  nginx:
    image: nginx:alpine
    restart: always
    container_name: nginx
    volumes:
      - /var/nfs:/var/www/html/
      - /var/nfs/nginx.conf:/etc/nginx/conf.d/default.conf
    ports:
      - "80:80"
    networks:
      - mynet
    deploy:
      replicas: 3
      placement:
        constraints:
          - "node.role == worker"
networks:
  mynet:
    driver: overlay
 ```
 Here,
- the volume is bind-mounted as "/var/nfs:/var/www/html/", and the same path /var/nfs is mounted in the nfs server.
- replicas: 3 implies that 3 replicas will be created on stack deployment
- node.role == worker indicates that deployment will only take place on the worker node
 
 nginx configuration file for my application
 ```sh
 server {
  listen 80;
  listen [::]:80;
  access_log off;
  root /var/www/html;
  index index.php;
  server_name example.com;
  server_tokens off;
  location / {
    # First attempt to serve request as file, then
    # as directory, then fall back to displaying a 404.
    try_files $uri $uri/ /index.php?$args;
  }
  # pass the PHP scripts to FastCGI server listening on wordpress:9000
  location ~ \.php$ {
    fastcgi_split_path_info ^(.+\.php)(/.+)$;
    fastcgi_pass php:9000;          <====== php is the name of the container having php-fpm
    fastcgi_index index.php;
    include fastcgi_params;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    fastcgi_param SCRIPT_NAME $fastcgi_script_name;
  }
}
```
## NFS Configuration
- For installing NFS server: 
```sh
apt install nfs-kernel-server
```sh
- Update the permissions and ownership of the directory
- Grant NFS share access to Client systems by updating the exports file.
- ```sh
vim /etc/exports
/var/nfs IP(rw,sync,no_subtree_check)
exportfs -a
systemctl restart nfs-kernel-server
```
 NB : Need to mount /var/nfs in the manager as well as all the 3 workers.
 
# Deploying Stack to Swarm

## 1) Set up a Docker registry

Because a swarm consists of multiple Docker Engines, a registry is required to distribute images to all of them
```sh
 docker service create --name registry --publish published=5000,target=5000 registry:2
 ```
Check its status with 
 ```sh
 docker service ls
 ```
 ## 2) Push the generated image to the registry
 
 To distribute the web app’s image across the swarm, it needs to be pushed to the registry you set up earlier. With Compose, this is very simple:
```sh
 docker-compose push
 ```
 The stack is now ready to be deployed.
 
 ### 3) Deploy the stack to the swarm
 
 Create the stack with docker stack deploy
 ```sh
 docker stack deploy --compose-file docker-compose.yml mystack
 ```
 The last argument is a name for the stack. Each network, volume and service name is prefixed with the stack name.

Check that it’s running with 
```sh
docker stack services mystack
```
Once it’s running, you should see 3/3 under REPLICAS for both services. This might take some time if you have a multi-node swarm, as images need to be pulled.

If required increase the replica count(say 10) anytime in future by :
```sh
docker service scale mystack_nginx=10 mystack_php=10
```
## Conclusion:
Configured a web application and deployed it using the docker stack to swarm along with an ALB configured for the traffic flow. has been completed successfully.
