# Docker-Nginx-PHP-MySQL-PHPMyAdmin-Gitlab
Docker running PHP-FPM, MySQL, using Nginx as a reverse proxy for Gitlab and PHPMyAdmin

### Description
The goal of set up will be to get Nginx up and Gitlab running on a reverse proxy through Nginx. You can alter the ports that are published by each service so they're all public facing but in general I found it easier to route all traffic through Nginx as a reverse proxy and using Docker's network feature to avoid having to deal with changing any ports at all. You don't have to self-host Gitlab, in fact **you probably shouldn't**, but I'm doing it for fun. Make sure your system meets [Gitlab's minimum requirements](https://docs.gitlab.com/ee/install/requirements.html) or else Gitlab installation will fail.

### Installing Docker and Docker Compose
Find your OS, download [Docker](https://docs.docker.com/install/) and [Docker Compose](https://docs.docker.com/compose/install/) for it

### Docker Images to pull
- [nginx](https://hub.docker.com/_/nginx/)
- [mysql](https://hub.docker.com/_/mysql/)
- [phpmyadmin/phpmyadmin](https://hub.docker.com/r/phpmyadmin/phpmyadmin/)
- [gitlab/gitlab-ce](https://hub.docker.com/r/gitlab/gitlab-ce/)
- [php](https://hub.docker.com/_/php)

### My Set up
I'm doing this all on a fresh Ubuntu 18.04 install. It comes by default with curl, git, etc.
I'm setting this up for my own personal domain (trap.fashion) and have already preconfigured the DNS records for the domain and subdomains I'm using. 

Final Project Tree
```
.
└── Website
  ├── docker-compose.yml
  ├── nginx
  │   ├── blablabla bunch of normal nginx stuff  
  │   ├── nginx.conf
  │   └── conf.d
  │       └── default.conf
  │       └── gitlab.trap.fashion.conf
  │       └── phpmyadmin.trap.fashion.conf
  ├── gitlab
  │   └── blablabla persistent gitlab storage, will retain logins and branches and stuff
  ├── mysql
  │   └── blablabla persistent mysql storage
  └── www
      └── index.html
```
### Installation Steps
**If you aren't installing Gitlab, feel free to omit it and skip to part 7**
1. Install Docker, Docker Compose, and pull the images above. I used all the default images at the time. The only directory I've created is /home/plykiya/website and /home/plykiya/website/www. Used ``sudo chown plykiya:plykiya /home/plykiya/website`` to get rid of permission errors. The Nginx, Gitlab, PHPMyAdmin, and MySQL folders are made automatically through docker volumes copying the contents of the container onto your host machine. Any changes to the files on your host machine will be reflected in the container upon starting the container.

2. Run Nginx, settings to create persistent volume storage, and this will be the only container we expose to port 80:80 because we want it to process http requests to the server IP. We're only starting this Nginx container so that we can begin editing the config and setting up Gitlab. Add an index.html file to the host directory to test if Nginx works.
```
docker run -v /home/plykiya/website/www:/usr/share/nginx/html:ro -v /home/plykiya/website/nginx:/etc/nginx:ro -d -p 80:80 nginx
cd /home/plykiya/website/www
nano index.html
```
Check your conf.d directory to ensure that you have a default.conf file. If you don't, you can copy one from the currently running container. If you have the default.conf file, stop and remove your Nginx container as we won't be using this instance anymore.
```
docker cp containername:/etc/nginx/conf.d/default.conf /home/plykiya/website/nginx/conf.d/
```

3. Navigate to the conf.d folder and add in a config file for Gitlab.
```
cd /home/plykiya/website/nginx/conf.d
sudo vi gitlab.trap.fashion.conf
```
The .conf file will make Nginx listen to standard http requests for the server_name you give it. We're going to be using Docker's network feature so the IP we're passing along will be the local IP of the container on the Docker network. Setting client_max_body_size is necessary here instead of on the normal nginx.conf since we're using this as a reverse proxy. It needs to be that large to handle the size of the initial Gitlab installation's push/pulls.
```
server {
listen 80;
listen [::]:80;

server_name gitlab.trap.fashion;

location / {
proxy_pass http://172.18.0.2:80/;
proxy_set_header Host $host;
proxy_buffering off;
proxy_set_header X-Real-IP $remote_addr;
client_max_body_size 100m;
}
}
```

4. Create a Docker network so that your containers can communicate with each other.
```
docker network create trap-network
```

5. Start up Gitlab with settings to create persistent storage and accessible at the domain name we give it in the nginx reverse proxy config and the config here. Before you start up the Nginx container, we can confirm the IP address of the Gitlab container by running ``docker inspect containername``. If the IP is different than what we've written down in the Nginx reverse proxy config, feel free to change it before starting Nginx. Once you've confirmed it's correct, start up Nginx as well. 
```
docker run -d --hostname gitlab.trap.fashion --network trap-network --restart unless-stopped --volume /home/plykiya/website/gitlab/config:/etc/gitlab --volume /home/plykiya/website/gitlab/logs:/var/log/gitlab --volume /home/plykiya/website/gitlab/data:/var/opt/gitlab gitlab/gitlab-ce
docker run --network trap-network -v /home/plykiya/website/www:/usr/share/nginx/html:ro -v /home/plykiya/website/nginx:/etc/nginx -d -p 80:80 nginx
```
Gitlab should now be accessible at the domain name you've given it. It may take a few minutes for Gitlab to start up. 

6. Now we set up our directory as the git origin and begin working mainly through gitlab to make any adjustments to our files for sanity's sake.
```
git config --global user.email "youremail@here"
git config --global user.name "username"
cd /home/plykiya/website
git init
git remote add origin http://gitlab.trap.fashion/plykiya/trap-fashion.git
sudo git add .
sudo git commit -m "Initial commit"
sudo git push -u origin master
```

7. In the main directory, we can now create our docker-compose.yml file. You can adjust the host directories to wherever you would like to store the persistent data. 
```
version: '3.7'
networks:
    trap-network:
        driver: bridge
services:
    php:
        image: php
        container_name: trap-php
        working_dir: /home/plykiya/website/www
        networks:
            - trap-network
        environment:
            TIMEZONE: EST
    nginx:
        image: nginx
        container_name: trap-nginx
        networks:
            - trap-network
        depends_on:
            - php
            - gitlab
            - phpmyadmin
        restart: always
        volumes:
            - '/home/plykiya/website/www:/usr/share/nginx/html:ro'
            - '/home/plykiya/website/nginx:/etc/nginx:ro'
        ports:
            - '80:80'
    db:
        image: mysql
        container_name: trap-mysql
        networks:
            - trap-network
        command: --default-authentication-plugin=mysql_native_password
        environment:
            MYSQL_ROOT_PASSWORD: somesupersecretpasswordyoumade
        restart: always
        volumes:
            - '/home/plykiya/website/mysql:/var/lib/mysql'
    phpmyadmin:
        image: phpmyadmin/phpmyadmin
        container_name: trap-phpmyadmin
        networks:
            - trap-network
        depends_on:
            - db
        restart: always
    gitlab:
        image: gitlab/gitlab-ce
        container_name: trap-gitlab
        networks:
            - trap-network
        volumes:
            - '/home/plykiya/website/gitlab/config:/etc/gitlab'
            - '/home/plykiya/website/gitlab/logs:/var/log/gitlab'
            - '/home/plykiya/website/gitlab/data:/var/opt/gitlab'
        restart: always
```

8. Now we go back into our Nginx conf.d folder to add in another config file for PHPMyAdmin. We also need to update the Gitlab config as well. As you can see, instead of using the local IP we use the name of the service in our docker-compose file. All our containers will now be using Docker's network feature, so http://phpmyadmin/ will be interpreted as http://ipofthecontainer/. Change the Gitlab config file to also use http://gitlab/.
```
server {
listen 80;
listen [::]:80;

server_name phpmyadmin.trap.fashion;

location / {
proxy_pass http://phpmyadmin/;
proxy_buffering off;
proxy_set_header X-Real-IP $remote_addr;
}
};
```

9. Once you've pulled the new config files and the docker-compose.yml using git, shut down and remove the Nginx/Gitlab docker containers and the Docker network we were working with in parts 4-6. Navigate to the directory with your Docker Compose file and start it up.
```
docker-compose up -d
```
If you get 502 errors on your Gitlab, update the permissions and restart the gitlab and nginx containers.
```
sudo docker exec trap-gitlab update-permissions
sudo docker restart trap-gitlab
sudo docker restart trap-nginx
```
