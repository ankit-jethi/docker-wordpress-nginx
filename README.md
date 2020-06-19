# WordPress on Docker with Nginx and ELK Stack

Prerequisites: Install docker & docker-compose.

## Scenarios
Different files have been created for different scenarios.

### Scenario 1: Multiple containers on a single host without ELK Stack - [docker-compose.yml](../master/docker-compose.yml):

>Minimum compose file version required is 3.1 if you want to use secrets.  

There are 3 services: **Web (Nginx)**, **WordPress** & **DB (MariaDB)**.  
Dependency has been created between the services so that MariaDB starts first, then WordPress and lastly Nginx.  
A network has been setup for the services to communicate with each other.

The following lines create **secrets** from files - One for the MySQL root password & the other one for the WordPress database password. 
```
# Top-level secrets configuration of the compose file

secrets:
  mysql_root_password:
    file: mysql_root_password.txt
  wp_db_password:
    file: wp_db_password.txt
```
Create these files an put the password in them and remember to restrict their access.

>Secrets in Docker Compose cannot be created externally. You also cannot specify the permissions for the secrets mounted inside the container. You can do both of these things in Docker Swarm though.

Now, run in this directory:
```
docker-compose up -d
```
You can access the website on ports 80 and 443 (With redirects to HTTPS).

#### Web (Nginx):
I have built my own image for Nginx based on **nginx:1.18.0-alpine** (Please refer to the 
[Nginx config section](#nginx-config) for details).  
Ports have mapped for 80 & 443 (With redirects to HTTPS).  
Named volumes have beeen setup for storing cache & sharing wordpress files.

#### WordPress:
Based on the official image - **wordpress:5.4.1-php7.4-fpm-alpine**.  
Named volume has been setup to store wordpress files.  
The following lines contain the environment variables for connecting to the database service (Change these values accordingly):
```
    environment:
      WORDPRESS_DB_HOST: db:3306    # Here, db is the service name of MariaDB.
      WORDPRESS_DB_USER: wp-db-user
      WORDPRESS_DB_PASSWORD_FILE: /run/secrets/wp_db_password
      WORDPRESS_DB_NAME: wp-db
```
The following lines grants the Wordpress service access to a secret called wp_db_password.
```
# Under the WordPress service

    secrets:
      - wp_db_password
```
**/run/secrets/** - is the default path where secrets are mounted inside the container. (A different path can be specified)  
So, WordPress will look for a file called **wp_db_password** in the **/run/secrets/** directory for the database password.

#### DB (MariaDB):
Based on the official image - **mariadb:10.3.23-bionic**.  
Named volume has been setup to store database files.  
The following lines contain the environment variables for setting up the database (Change these values accordingly):
```
    environment:
      MYSQL_ROOT_PASSWORD_FILE: /run/secrets/mysql_root_password
      MYSQL_DATABASE: wp-db
      MYSQL_USER: wp-db-user
      MYSQL_PASSWORD_FILE: /run/secrets/wp_db_password
```
The following lines grants the MariaDB service access to multiple secrets.
```
# Under the MariaDB service

    secrets:
      - mysql_root_password
      - wp_db_password
```      

### Scenario 2: Multiple containers on multiple hosts without ELK Stack - [docker-stack.yml](../master/docker-stack.yml):

Building on the docker-compose.yml, here are the differences:
>Minimum compose file version required is 3.4 for the following reasons:  
Deploy placement preference requires version 3.2 and order for update configurations requires version 3.4.

WordPress and the Nginx service are sharing a volume. While in docker compose this is no problem because it is all running on a single host, but for docker swarm to share volumes between containers, the approach has to be changed because we need data across multiple hosts. We cannot use the default storage driver for this.  
So, for the shared storage I have used **AWS EFS** service using the **rexray/efs** storage driver.

The **rexray/efs** storage driver is available as a plugin and it requires that **nfs utilities** be installed. And you should be able to mount an nfs export to the host.  
You can install the plugin using the following command:
```
docker plugin install --grant-all-permissions \
  rexray/efs \
  EFS_ACCESSKEY=abc \
  EFS_SECRETKEY=123 \
  EFS_SECURITYGROUPS="sg-123 sg-456" \
  EFS_REGION=us-east-1 \
  REXRAY_PREEMPT=true \
  LINUX_VOLUME_FILEMODE=0755
```
The plugin can also use the instance profile IAM permissions, so it is not necessary to provide the EFS_ACCESSKEY & EFS_SECRETKEY. Also, open port **2049** on the EFS_SECURITYGROUPS. Do make sure to set the **LINUX_VOLUME_FILEMODE=0755** or else nginx won't be able to access the wordpress files.

Also, make sure that the AWS credentials (user or role) has the following AWS permissions on the server instance that will be making calls to the AWS API:
```
elasticfilesystem:CreateFileSystem
elasticfilesystem:CreateMountTarget
ec2:DescribeSubnets
ec2:DescribeNetworkInterfaces
ec2:CreateNetworkInterface
elasticfilesystem:CreateTags
elasticfilesystem:DeleteFileSystem
elasticfilesystem:DeleteMountTarget
ec2:DeleteNetworkInterface
elasticfilesystem:DescribeFileSystems
elasticfilesystem:DescribeMountTargets
```
Then, in the docker-stack.yml file, you can use the rexray/efs driver like this:
```
# Top-level volumes configuration of the compose file

volumes:
  nginx_cache:
  wp_data:
    driver: rexray/efs
  db_data:
```
Secrets for Docker Swarm can be created externally. There are 2 ways to create a secret:
1.  `printf <secret> | docker secret create wp_db_password -`  
wp_db_password is the name you provide to the secret.
2.  `docker secret create wp_db_password wp_db_password.txt`  
wp_db_password.txt is the file which contains the password.

Then, in the docker-stack.yml file, mention the secrets like this:
```
# Top-level secrets configuration of the compose file

secrets:
  mysql_root_password:
    external: true
  wp_db_password:
    external: true
```

Now, to run this in a Docker Swarm, first make sure you have a swarm. If you don't, run:
```
docker swarm init
```
Once you have your swarm, then in this directory run:
```
docker stack deploy --compose-file docker-stack.yml wp-docker
```
You can access the website on ports 80 and 443 (With redirects to HTTPS).

#### Web (Nginx):
The following lines configure the deployment for the Nginx service (Change these values accordingly):
```
    deploy:                     
      replicas: 4               # Specifies the number of containers to run.
      update_config:            # Configures how the service should be updated. Useful for configuring rolling updates.   
        order: start-first      # The order of updates is set to start-first (new task is started first and then the old one is stopped)
        parallelism: 2          # The number of containers to update at a time.
        delay: 10s              # The time to wait between updating a group of containers.
      restart_policy:           # Configures if and how to restart containers when they exit.
        condition: on-failure   # Containers will be restarted if they fail.
        delay: 20s              # The duration to wait between restart attempts.
```

#### WordPress:
The following lines grants the Wordpress service access to a secret called wp_db_password:
```
    secrets:
      - source: wp_db_password
        mode: 0400
```
The difference as compared to docker compose is that you can specify the permissions for the file to be mounted in **/run/secrets/** in the container.

#### DB (MariaDB):
The following lines configure the deployment for the MariaDB service (Change these values accordingly):
```
    deploy:
      placement:
        constraints: [node.role == manager] # This placement constraint limits the task to be only deployed on Manager node.
```

### Scenario 3: Multiple containers on a single host with ELK Stack - [docker-compose-elk.yml](../master/docker-compose-elk.yml):

There are 4 services: **Web (Nginx)**, **WordPress**, **DB (MariaDB)** & **ELK**.  

Prerequisites for setting up ELK:
1. A minimum of 4GB RAM assigned to Docker.
2. A limit on mmap counts equal to 262,144 or more.  
You can view the nmap count by running - `sysctl vm.max_map_count`    
You can change it by running - `sysctl -w vm.max_map_count=262144`  
To set this value permanently, update the **vm.max_map_count** setting in **/etc/sysctl.conf**. And to verify after rebooting, run `sysctl vm.max_map_count` again.
3. The host's limits on open files must be increased: You can view the soft limit by running `ulimit -Sn` and hard limit by running `ulimit -Hn`. To set this value permanently, set **nofile to 65536 in /etc/security/limits.conf**.  
And to adjust Docker's ulimit settings globally, create or edit **/etc/docker/daemon.json** and add the following lines:
```
{
"default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 65536,
      "Soft": 1024
    }
  }
}
```
Then, restart the docker service.

Now, run in this directory:
```
docker-compose -f docker-compose-elk.yml up -d
```
You can access the website on ports 80 and 443 (With redirects to HTTPS).

Building on the docker-compose.yml, here are the differences:

#### Web (Nginx):
The following lines configure the Nginx service to send logs to Logstash.
```
    logging:
      driver: gelf
      options:
        gelf-address: udp://localhost:12201 # Send logs to ELK stack service listening on 12201/udp.
        tag: "nginx"                        # Creates a tag to differentiate between logs of different services.
```
The logging driver has been changed to **gelf**. You can use GELF to send logs to Graylog or Logstash. Other logging drivers are also available like **syslog** & **awslogs** for Amazon CloudWatch.

#### ELK:
Based on - **sebp/elk:771** image.  
The following lines configure the port mappings for the ELK service.
```
    ports:
      - "5601:5601"       # Kibana web interface.
      - "9200:9200"       # Elasticsearch JSON interface.
      - "5044:5044"       # Logstash Beats interface, receives logs from Beats such as Filebeat.
      - "12201:12201/udp" # Logstash port for the docker logging driver.
```
The following lines associate a named volume & a bind mount for the ELK service.
```
    volumes:
      - elk_data:/var/lib/elasticsearch                                   # For persisting the log data.
      - ./logstash_docker.conf:/etc/logstash/conf.d/logstash_docker.conf  # For the Logstash config, refer below.     
```
The following lines adjusts the containers ulimit settings.  
Also, by default Elasticsearch has 30 seconds to start before other services are started, which may not be enough and cause the container to stop. So changing that to 60 seconds.
```
# Under the ELK service

    ulimits:
      nofile:
        soft: 1024
        hard: 65536
    environment:
      ES_CONNECT_RETRY: 60
```

### Scenario 4: Multiple containers on multiple hosts with ELK Stack - [docker-stack-elk.yml](../master/docker-stack-elk.yml):

Building on the previous files, here are the differences:

>You cannot adjust the individual containers ulimit settings in docker swarm. For that you have to change the settings globally by editing **/etc/docker/daemon.json**.

Docker Swarm does support use of **configs** unlike docker-compose. So, using that instead of bind mount for the logstash configuration.  
The following lines create a config called logstash_docker from a file called [logstash_docker.conf](../master/logstash_docker.conf) for use with the ELK service:
```
# Top-level configs configuration of the compose file

configs:
  logstash_docker:              # Config name.
    file: logstash_docker.conf  # The file to create the config from.
```
Now, to run this in a Docker Swarm, first make sure you have a swarm. If you don't, run:
```
docker swarm init
```
Once you have your swarm, then in this directory run:
```
docker stack deploy --compose-file docker-stack-elk.yml wp-docker-elk
```
You can access the website on ports 80 and 443 (With redirects to HTTPS).

#### ELK:
The following lines associate the config - logstash_docker with the ELK service:
```
# Under the ELK service

    configs:
      - source: logstash_docker                           # Config name.
        target: /etc/logstash/conf.d/logstash_docker.conf # The path and name of the file to be mounted inside the container.
        mode: 0444                                        # Permissions for the mounted config file.
```

## Nginx config:

1. [Dockerfile](../master/nginx/Dockerfile)   

\- Based on the official image - **nginx:1.18.0-alpine**.  
\- Just added my own global nginx & virtual host configuration. Also, self-signed certificates & dhparam.

2. Global nginx configuration - [nginx.conf](../master/nginx/nginx.conf):    

\- Adjust the worker_connections as per demand.  
\- Configure the worker_rlimit_nofile directive to adjust the nofile settings.  
\- Configure settings for gzip & gunzip.  
\- Also, adjust the keep_alive timeout.

3. Virtual host configuration - [blog.example.com.conf](../master/nginx/blog.example.com.conf):  

\- Setup caching.  
\- Setup redirects to HTTPS.  
\- Setup HTTP2.  
\- Setup the self-signed ssl certificates.  
\- Setup some more ssl configuration - timeout, cache, dhparam, protocols, ciphers.  
\- Setup HSTS.  
\- Setup OCSP stapling (Cannot be used with self-signed certs).  
\- Setup Reverse proxy to WordPress.  
\- Setup header to check cache status.

4. SSL directory:  

\- Includes self-signed certificates & dhparam.

## Logstash config - [logstash_docker.conf](../master/logstash_docker.conf):

1. Input:  

\- This is where the logstash service sets up the incoming port for the docker gelf logging driver.

2. Filter:  

\- Every service (Nginx, WordPress & MariaDB) was tagged in the compose files. You can use those tags and apply grok pattern to each service & map them with fields for use with Elasticsearch.  
\- As an example, the nginx access logs & error logs have been mapped.

3. Output:  

\- The output is sent to Elasticsearch with the specified index prefix.


### Sources:
1. [Manage sensitive data with Docker secrets](https://docs.docker.com/engine/swarm/secrets/)
2. [Store configuration data using Docker Configs](https://docs.docker.com/engine/swarm/configs/)
3. [Compose file version 3 reference](https://docs.docker.com/compose/compose-file/)
4. [Example Docker Compose app](https://github.com/dockersamples/example-voting-app)
5. [Tips for Deploying Nginx with Docker](https://www.docker.com/blog/tips-for-deploying-nginx-official-image-with-docker/)
6. [Mozilla Security/Server Side TLS](https://wiki.mozilla.org/Security/Server_Side_TLS)
7. [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/)
8. [Ulimits in compose file](https://docs.docker.com/compose/compose-file/#ulimits)
9. [Docker daemon configuration file](https://docs.docker.com/engine/reference/commandline/dockerd/#daemon-configuration-file)
10. [REX-Ray - AWS EFS volume plugin](https://rexray.readthedocs.io/en/stable/user-guide/schedulers/docker/plug-ins/aws/#aws-efs)
11. [Make Stateful Applications Highly Available w/ Docker Swarm Mode (and Docker plugins!)](https://www.youtube.com/watch?v=U8Dsi5V-XG0)
12. [Docker Storage: Designing a Platform for Persistent Data](https://www.youtube.com/watch?v=y5wMbA_T0tA)
13. [Docker Lab - Swarm Mode, Setup shared storage between VMs & ELK stack](https://github.com/cloud-coder/docker-lab-2016/tree/master/part-3)
14. [ELK Docker image documentation](https://elk-docker.readthedocs.io/)
15. [Docker - Log Driver - ELK](https://www.youtube.com/watch?v=eM8H2c7L_Iw)
16. [Send docker logs to ELK through gelf log driver](https://gist.github.com/eunomie/e7a183602b8734c47058d277700fdc2d)
17. [Graylog Extended Format logging driver](https://docs.docker.com/config/containers/logging/gelf/)
