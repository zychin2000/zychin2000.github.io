---
layout: post
title: Deploying a Tomcat and MySQL Server in Docker
subtitle:
tags: [docker, java, jee, mysql]
comments: true
---


Recently, I was developing a Java Servlet web project, and I wanted a simple way to deploy this project across all environments. There wasn't much information about deploying Java Servlet projects on Docker elsewhere, so I decided to make a short tutorial on deploying the JEE web application alongside MySQL.

# The Project
The project I was working on is a Java EE application (also known as a Dynamic Web Project in Eclipse), and a MySQL server was deployed separately. I used Maven as the package manager for the project; all the project dependencies were loaded automatically on Maven. 

## Maven Configuration
When building the project, the war file has to be created to be deployed in Tomcat. You will have to tell Maven to generate a war file. 
In pom.xml, add the following line, if you haven't already, under the project tag:

```xml
<project>
    <packaging>war</packaging>
</project>
```

## The Dockerfile

Create a file called Dockerfile at the root directory of your project. 

{% highlight Docker linenos %}
FROM maven:3.6.0-jdk-11-slim AS build

# Copy pom.xml to the image
COPY pom.xml /home/app/pom.xml
RUN mvn -f /home/app/pom.xml verify clean --fail-never

# Copy the source code 
COPY src /home/app/src
RUN mvn -f /home/app/pom.xml package

# Install Tomcat    & openjdk 8 (openjdk has java and javac)
FROM tomcat:jdk8-openjdk
# Copy source files to tomcat folder structure
COPY --from=build /home/app/target/YOUR PROJECT.war /usr/local/tomcat/webapps/

EXPOSE 8080
{% endhighlight %}

There are two stages in the image building process: the first part builds the project's dependencies and then compiles the source code. After doing these two steps, a separate image is created, where the output of the previous compilation gets copied to the Tomcat image.

## Docker Compose 

We have just finished building the image and deploying the container for our Java EE application, but what about the SQL server? With Docker Compose, we can create a multi-container application, where this project has two containers: the Java application and the MySQL server. 

Create a separate file called docker-compose.yml at the root of the project.

{% highlight yml linenos %} 
  version: "3.9" #version of docker-compose
  services:
    mysql:
      container_name: mysql
      image: mysql
      ports:
      - "3307:3306"
      environment:
        - MYSQL_ROOT_PASSWORD=admin1                    #default password of the MySQL container
      volumes:
        - ./data:/var/lib/mysql                         #database data volume
        - ./src/main/db:/docker-entrypoint-initdb.d/:ro #database files called when the container is built and started for the first time
    web:
      container_name: tomcat
      build: .
      ports:
        - "8080:8080"
{% endhighlight %}

This file describes the whole stack of the application, where includes both the web server and MySQL. For our Tomcat application, it builds the Docker image based on the Dockerfile at the same directory. 

A separate image of MySQL is also built. The official MySQL image is downloaded from the Docker repository, and we will have to configure our image to suit our needs. The root password of MySQL can be changed, and you will have to set the volumes for the container. For example, the ./data folder on your host machine will be completely mirrored with the /var/lib/mysql on the Docker container. The left side of the volume entries has to be set to the appropriate folder of the project. 

The first volume that is described in the Dockerfile is the data volume, where all the MySQL data will be stored. The second volume corresponds to the database initialization files, where every .sh, .sql, and .sql.gz files are executed when the container initially starts. This means you can create the load your database schema by mounting a SQL dump in that folder. 

### Inter-container Connection

To communicate with another container, you can connect with it by using its hostname as indicated by its container name. For example, to connect to the MySQL server from the 'tomcat' container, set the database address as mysql:3306.

## Using Docker Compose

To deploy the app, you can call ```docker-compose up```, and Docker Compose will automatically build the two images according to docker-compose.yml and run these two images on two containers concurrently.

If it's deployed properly on your local machine, you can see your webapp at localhost:8080!