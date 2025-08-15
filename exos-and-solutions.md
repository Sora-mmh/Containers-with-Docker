# Containers with Docker — Exercises & Solutions

**Repository to use:**  
[GitLab – docker-exercises](https://gitlab.com/twn-devops-bootcamp/latest/07-docker/docker-exercises)

Your team member has improved your previous static java application and added mysql database connection, to let users edit information and save the edited data.

They ask you to configure and run the application with Mysql database on a server using docker-compose.

***

## Exercise 0: Clone Git repository and create your own
**Task:**  
You can check out the code changes and notice that we are using environment variables for the database and its credentials inside the application.

This is very important for 2 reasons:
- You don't want to expose the password to your database by hardcoding it into the app and checking it into the repository!
- These values may change based on the environment, so you want to be able to set them dynamically when deploying the application, instead of hardcoding them.

**Solution:**
```sh
# clone repository & change into project dir
git clone git@gitlab.com:twn-devops-bootcamp/latest/07-docker/docker-exercises.git
cd docker-exercises

# remove remote repo reference and create your own local repository
rm -rf .git
git init 
git add .
git commit -m "initial commit"

# create git repository on Gitlab and push your newly created local repository to it
git remote add origin git@gitlab.com:{gitlab-user}/{gitlab-repo}.git
git push -u origin master

# you can find the environment variables defined in src/main/java/com/example/DatabaseConfig.java file
```

***

## Exercise 1: Start Mysql container
**Task:**  
First you want to test the application locally with a mysql database. But you don't want to install Mysql, you want to get started fast, so you start it as a docker container:
- Start mysql container locally using the official Docker image. Set all needed environment variables.
- Export all needed environment variables for your application for connecting with the database (check variable names inside the code)
- Build a jar file and start the application. Test access from browser. Make some changes.

**Solution:**
```sh
# start mysql container using docker
docker run -p 3306:3306 \
--name mysql \
-e MYSQL_ROOT_PASSWORD=rootpass \
-e MYSQL_DATABASE=team-member-projects \
-e MYSQL_USER=admin \
-e MYSQL_PASSWORD=adminpass \
-d mysql

# create java jar file
gradle build

# set env vars in Terminal for the java application (these will read in DatabaseConfig.java)
export DB_USER=admin
export DB_PWD=adminpass
export DB_SERVER=localhost
export DB_NAME=team-member-projects

# start java application
java -jar build/libs/docker-exercises-project-1.0-SNAPSHOT.jar
```

***

## Exercise 2: Start Mysql GUI container
**Task:**  
Now you have a database, you want to be able to see the database data using a UI tool, so you decide to deploy phpmyadmin. Again, you don't want to install it locally, so you want to start it also as a docker container.
- Start phpmyadmin container using the official image.
- Access phpmyadmin from your browser and test logging in to your Mysql database

**Solution:**
```sh
# start phpmyadmin container using the official image
docker run -p 8083:80 \
--name phpmyadmin \
--link mysql:db \
-d phpmyadmin/phpmyadmin

# access it in the browser on
localhost:8083

# login to phpmyadmin UI with either of 2 mysql user credentials:
# * admin:adminpass
# * root:rootpass
```

***

## Exercise 3: Use docker-compose for Mysql and Phpmyadmin
**Task:**  
You have 2 containers your app needs and you don't want to start them separately all the time. So you configure a docker-compose file for both:
- Create a docker-compose file with both containers
- Configure a volume for your DB
- Test that everything works again

**Solution:**  
*Create **docker-compose.yaml**:*
```yaml
version: '3'
services:
  mysql:
    image: mysql
    ports:
      - 3306:3306
    environment:
      - MYSQL_ROOT_PASSWORD=rootpass
      - MYSQL_DATABASE=team-member-projects
      - MYSQL_USER=admin    
      - MYSQL_PASSWORD=adminpass
    volumes:
    - mysql-data:/var/lib/mysql
    container_name: mysql
  phpmyadmin:
    image: phpmyadmin
    environment:
      - PMA_HOST=mysql
    ports:
      - 8083:80
    container_name: phpmyadmin
volumes:
  mysql-data:
    driver: local
```

**Start containers with docker-compose:**
```sh
docker-compose -f docker-compose.yaml up    
```

***

## Exercise 4: Dockerize your Java Application
**Task:**  
Now you are done with testing the application locally with Mysql database and want to deploy it on the server to make it accessible for others in the team, so they can edit information.

And since your DB and DB UI are running as docker containers, you want to make your app also run as a docker container. So you can all start them using 1 docker-compose file on the server. So you do the following:
- Create a Dockerfile for your java application

**Solution:**  
*Create **Dockerfile**:*
```dockerfile
FROM openjdk:17.0.2-jdk
EXPOSE 8080
RUN mkdir /opt/app
COPY build/libs/docker-exercises-project-1.0-SNAPSHOT.jar /opt/app
WORKDIR /opt/app
CMD ["java", "-jar", "docker-exercises-project-1.0-SNAPSHOT.jar"]
```

***

## Exercise 5: Build and push Java Application Docker Image
**Task:**  
Now for you to be able to run your java app as a docker image on a remote server, it must be first hosted on a docker repository, so you can fetch it from there on the server. Therefore, you have to do the following:
- Create a docker hosted repository on Nexus
- Build the image locally and push to this repository

**Solution:**
```sh
# create jar file - docker-exercises-project-1.0-SNAPSHOT.jar
gradle build

# create docker image - {repo-name}/{image-name}:{image-tag}
docker build -t {repo-name}/java-app:1.0-SNAPSHOT .

# push docker to remote docker repo {repo-name}
docker push {repo-name}/java-app:1.0-SNAPSHOT
```

***

## Exercise 6: Add application to docker-compose
**Task:**  
- Add your application's docker image to docker-compose. Configure all needed env vars.
- **TIP:** Ensure you configure a health check on your mysql container by including the following in your docker-compose file:

```yaml
my-java-app:
  depends_on:
    mysql:
      condition: service_healthy
mysql:
  healthcheck:
    test: [ "CMD", "mysqladmin", "ping", "-h", "localhost" ]
    interval: 10s
    timeout: 5s
    retries: 5
```

Now your app and Mysql containers in your docker-compose are using environment variables.
- Make all these environment variable values configurable, by setting them on the server when deploying.

**INFO:** Again, since docker-compose is part of your application and checked in to the repo, it shouldn't contain any sensitive data. But also allow configuring these values from outside based on an environment

**Solution:**  
*Create **docker-compose-with-app.yaml**:*
```yaml
version: '3'
services:
  my-java-app:
    image: java-mysql-app:1.0 # specify the full image name with repository name
    environment:
      - DB_USER=${DB_USER}
      - DB_PWD=${DB_PWD}
      - DB_SERVER=${DB_SERVER}
      - DB_NAME=${DB_NAME}
    ports:
    - 8080:8080
    container_name: my-java-app
    depends_on:
      mysql:
        condition: service_healthy
  mysql:
    image: mysql
    ports:
      - 3306:3306
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_DATABASE=${DB_NAME}
      - MYSQL_USER=${DB_USER}
      - MYSQL_PASSWORD=${DB_PWD}
    volumes:
    - mysql-data:/var/lib/mysql
    container_name: mysql
    healthcheck:
      test: [ "CMD", "mysqladmin", "ping", "-h", "localhost" ]
      interval: 10s
      timeout: 5s
      retries: 5
  phpmyadmin:
    image: phpmyadmin
    ports:
      - 8083:80
    environment:
      - PMA_HOST=${PMA_HOST}
      - PMA_PORT=${PMA_PORT}
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
    container_name: phpmyadmin
    depends_on:
      - mysql
volumes:
  mysql-data:
    driver: local
```

**Set environment variables and start containers:**
```sh
# set all needed environment variables
export DB_USER=admin
export DB_PWD=adminpass
export DB_SERVER=mysql
export DB_NAME=team-member-projects

export MYSQL_ROOT_PASSWORD=rootpass

export PMA_HOST=mysql
export PMA_PORT=3306

# start all 3 containers 
docker-compose -f docker-compose-with-app.yaml up    
```

***

## Exercise 7: Run application on server with docker-compose
**Task:**  
Finally your docker-compose file is completed and you want to run your application on the server with docker-compose. For that you need to do the following:
- Set insecure docker repository on server, because Nexus uses http
- Run docker login on the server to be allowed to pull the image
- Your application index.html has a hardcoded localhost as a HOST to send requests to the backend. You need to fix that and set the server IP address instead, because the server is going to be the host when you deploy the application on a remote server. (Don't forget to rebuild and push the image and if needed adjust the docker-compose file)
- Copy docker-compose.yaml to the server
- Set the needed environment variables for all containers in docker-compose
- Run docker-compose to start all 3 containers

**Solution:**
```sh
# on Linux server - to add an insecure docker registry, add the file /etc/docker/daemon.json with the following content
{
  "insecure-registries" : [ "{repo-address}:{repo-port}" ]
}

# restart docker for the configuration to take affect
sudo service docker restart

# check the insecure repository was added - last section "Insecure Registries:"
docker info

# do docker login to repo
docker login {repo-address}:{repo-port}

# change hardcoded HOST env var in src/main/resources/static/index.html file, line 48
const HOST = "{server-ip-address}";

# rebuild the application and image and push to repo
gradle build
docker build -t {repo-name}/java-app:1.0-SNAPSHOT .
docker push {repo-name}/java-app:1.0-SNAPSHOT 

# copy docker-compose file to remote server
scp -i ~/.ssh/id_rsa docker-compose.yaml {server-user}:{server-ip}:/home/{server-user}

# ssh into the remote server
# set all env vars as you did in exercise 6
# run docker compose file
# open port 8080 on server to access java application
```

***

## Exercise 8: Open ports
**Task:**  
Congratulations! Your application is running on the server, but you still can't access the application from the browser. You know you need to configure firewall settings. So do the following:
- Open the necessary port on the server firewall and
- Test access from the browser

**Solution:**
```sh
# open port 8080 on server firewall
sudo ufw allow 8080

# test access from browser
# navigate to: http://{server-ip}:8080
```
