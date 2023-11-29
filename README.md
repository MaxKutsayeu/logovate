# Logovate Website

## Docker Application Architecture

The application consists of `docker` dicrectory.
In the docker directories, the main containers are divided into folders and the rules for
their assembly are described in the Dockerfile of each of them.
Consolidated containers using docker-compose based on the docker-compose.yml file.

### Containers management

All files of their configurations and all variables are reset for access to certain services,
installations or certain extensions, modules, software schemes, etc.- everything is found
in dot-env based config files and they don't fall under git tracking, i.e. you can change
them in environment configuration files, rebuild containers, and see changes in built
containers. Under git, only configuration file templates are found (files with the .example
or .{env}.example prefix, based on which configuration files have already been found and
all private data is automatically uploaded to the server regularly).

#### Containers:
+ ```workspace``` - the main app's container with the installed tools for development (bash, yarn, composer, artisan, npm, nano, etc.)
+ ```mysql``` - database container
+ ```nginx``` - web-server contained
+ ```php-fpm``` - PHP container
+ ```php-worker``` - container to work with the queues + scheduler + websockets server
+ ```redis``` - container with redis (driver to work with the queues)

##### Versions used:

- **Docker (Nginx, MySQL, Laravel Horizon, Redis, WebSockets, Scheduler)**
- **PHP 7.2
- **Laravel 6.0.***

All containers are managed within .env docker. To connect some libraries/extensions/dependencies you need:
- change/add a variable in the .end docker file
- forward it into the docker compose of the required service
- rebuild the app

#### Docker commands
+ raise all the containers (run a project)
>```docker-compose up -d```
+ raise all the containers and recreate them
>```docker-compose up -d --build --force-recreate```
+ rebuild all the containers (without images cache)
>```docker-compose build --no-cache```
+ stop a project
>```docker-compose stop```
+ connection to the main app container
>```docker-compose exec workspace bash```
+ connection to any container by its id
>```docker exec -it [container-id] bash```
+ list of all containers
>```docker ps```

#### Debugbar

Displayed in local development(**APP_ENV=local**). If you need to use it from the production, you need to enable
debug mode **APP_DEBUG=true** and insert your IP address to **APP_DEBUG_IP**

---
## Set up project for local development
1. Create app in you shopify partners account.
2. Download the repository with the skeleton:
   ```
   git clone git@github.com:[userName]/logovate.git
   ```
3. Install [docker](https://docs.docker.com/get-docker/).
4. Configure .env

## Server setup and application deployment (AWS)

### Preparing ES2 instance
1. create ES2 instance with latest ubuntu version
2. allocate Elastic IP to instance
3. create domain or subdomain and set A record
4. check volume size of instance (recommended 15 GIB or more)
5. [create swap] on 1 GIB ( a small-capacity servers (e.g. t3.small)):
   ```
   # checking if a swap exists
   5.1. free -h
   5.2. sudo fallocate -l 1G /swapfile
   5.3. sudo chmod 600 /swapfile
   5.4. sudo mkswap /swapfile
   5.5. sudo swapon /swapfile
   
   #and make swap permanent
   5.6. sudo cp /etc/fstab /etc/fstab.bak
   5.7. echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
   ```
[create swap]: https://www.digitalocean.com/community/tutorials/how-to-add-swap-space-on-ubuntu-20-04-ru

### Create user with a development role
1. Log into a server via password or via **ubuntu** user through a SHH-key.
2. Create a **developer** user and check the access to **sudo**
   ```
   2.1. ssh root@[remote-host]
   2.2. adduser developer
   2.3. usermod -aG sudo developer
   2.4. su - developer
   2.5. sudo ls -la /root
   ```
3. Setup of the SSH-keys for a developer user (should be done on a local machine)
- create a pair of keys in the following folder ~/.ssh/developer/ (developer and developer.pub)
  ```ssh-keygen -f ~/.ssh/developer/developer -m PEM -t rsa```
- check a private key begins with **-----BEGIN OPENSSH PRIVATE KEY-----** (PEM-key)
- transfer the keys on the servers manually or using the following command:
   ```
   #the command will install the public key into ~/.ssh/authorized_keys
   ssh-copy-id -f -i ~/.ssh/developer/developer.pub root@[remote-host]
   ```
4. Turning off a password verification for a developer user:
   ```
   4.1. ssh developer@[remote-host]
   4.2. sudo nano /etc/ssh/sshd_config
   4.3. PasswordAuthentication = no
   4.3. sudo systemctl restart ssh
   ```

### Downloading a project on a server

1. Create of a directory with projects
   ```
   1.1. ssh developer@[remote-host
   1.2. cd /home/developer/
   1.3. mkdir projects
   1.4. cd projects
   ```

2. Install GIT

   ```
   2.1. sudo apt update
   2.2. sudo apt install git
   2.3. git --version
   ```

#### Connecting to a git repository and downloading a project

##### Bitbucket
```
1. git init
2. Add your public key for a developer-user into [Bitbucket] account
3. Download your private key for a developer user into /home/developer/.ssh/developer file
4. eval `ssh-agent -s`
5. ssh-add ~/.ssh/developer
6. chmod 400 /home/developer/.ssh/developer
7. git clone git@bitbucket.org:[username/logovate.git]
```

##### Github
````
1. create project directory
2. create github access token
2. git remote add origin https://[user]:[token]@github.com/[logovate].git
3. git fetch --all
4. git checkout [branch]
5. git pull
````
### Docker

1. [install Docker]
   [install Docker]: https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-22-04

   ```
   1. sudo apt update
   2. sudo apt install apt-transport-https ca-certificates curl software-properties-common
   2. curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
   2. echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
   2. sudo apt update
   2. apt-cache policy docker-ce
   2. sudo apt install docker-ce
   2. Check docker status (should be Active):
   8.1. sudo systemctl status docker
   2. sudo usermod -aG docker ${USER}
   2. You need to logout - login for the changes from the previous step to take place 
   ```

2. install Docker compose
   ```
   10. sudo curl -L "https://github.com/docker/compose/releases/download/1.26.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
   11. sudo chmod +x /usr/local/bin/docker-compose
   12. docker-compose --version
   ```

#### Launch app's installation (the command is executed once, when deploying an app on a server)

1. Run setup script
   ```
   1. cd logovate/docker
   2. bash setup.sh prod
   ```
2. Configure www/.env

#### Setting permissions after docker was built:

Permissions for **Laravel**:

```
1. chown laradock:laradock www/bootstrap/ -R
2. chown laradock:laradock www/storage/ -R
```
#### Adding SSL certificate (Certbot + Lets Encrypt)
##### docker/docker-compose.yml
1. Uncomment certbot settings in docker/docker-compose.yml
2. Set correct email and domain in certbot environment
3. Make sure CN and EMAIL are without quotes
4. Uncomment nginx volumes:
   ```
   #- ./data/certbot/certs/:/var/certs
   #- ./certbot/letsencrypt/:/var/www/letsencrypt 
   ```
##### docker/nginx/sites/app.conf
1. Comment https settings in docker/nginx/sites/app.conf as in example:
   ```
   # For https
   # listen 443 ssl;
   # listen [::]:443 ssl ipv6only=on;
   # ssl_certificate /etc/nginx/ssl/default.crt;
   # ssl_certificate_key /etc/nginx/ssl/default.key;
   ```
2. Execute commands:
   ```
   1. docker-compose stop nginx
   2. docker-compose build --no-cache nginx
   3. docker-compose up -d nginx
   4. docker-compose up certbot
   5. docker-compose up -d certbot
   5.1. Check that SSL keys have appeared in the docker/data/certbot/certs folder
   6. chown -R developer:developer docker/data/
   7. move to docker/nginx/sites/app.conf and uncomment ssl settings
   8. set correct pathes to ssl cetificates:
   ssl_certificate /var/certs/fullchain1.pem;
   ssl_certificate_key /var/certs/[domain-name]-privkey1.pem;
   you can see certificate names in docker/data/cerbot/certs
   9. docker-compose stop nginx
   10. docker-compose build --no-cache nginx
   11. docker-compose up -d nginx
   ```

### Ports' opening (using AWS as an example)

https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/authorizing-access-to-an-instance.html (using SSH as an example)
- add 80 and 443 ports
- add 9010 port for Portainer - web-interface for managing Docker containers (will be available at https://domain.com:9010/)

### Setting up automatic deployment (GITHUB)
```
1. sudo nano /etc/ssh/sshd_config
2. Uncomment or add 
PubkeyAuthentication yes
PubkeyAcceptedKeyTypes=+ssh-rsa
3. sudo service ssh restart
4. eval `ssh-agent -s`

```

### Changing the default access to the database before the application goes into production

```
1. docker exec -it [mysql-container-id] bash
2. mysql -u root -p (default password: root)
3. CREATE USER 'dbuser'@'localhost' IDENTIFIED BY 'password';
4. FLUSH PRIVILEGES;
5. GRANT SELECT ON * . * TO 'dbuser'@'localhost';
6. GRANT ALL PRIVILEGES ON `default` . * TO 'dbuser'@'localhost';
7. FLUSH PRIVILEGES;
8. DROP USER 'default'@'localhost';
8. Change DB_USERNAME and DB_PASSWORD in the www/.env file of the app
```

### Possible issues while deploying

1. If it requires a password for git commands (after adding SSH keys everywhere):
    - find nano .git/config into the project's folder
    - replace https://.. url with git@bitbucket.org:.. in the [remote "origin"] section
