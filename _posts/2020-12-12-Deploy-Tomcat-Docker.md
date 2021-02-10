---
layout: post
title: Deploying a Tomcat and MySQL Server in Docker
subtitle:
tags: [docker, java, jee, mysql]
comments: true
---


Recently, I was developing a Java Servlet web project, and I wanted a simple way deploy this project across all environments. There wasn't much information about deploying Java Servlet projects on Docker elsewhere, so I decided to make a short tutorial on deploying the JEE web application alongside MySQL.

# The Project
The project I was working on is a Java EE application (also known as a Dynamic Web Project in Eclipse), and a MySQL server was deployed seperately. I used Maven as the package manager for the project; all the project dependencies were loaded automatically on Maven. 

## Maven Configuration
When building the project, the war file have to be created to be deployed in Tomcat. You will have to tell Maven to generate a war file. 
In pom.xml, add the following line, if you haven't already, under the project tag:

```xml
<project>
    <packaging>war</packaging>
</project>
```

# The Dockerfile

Create a file called Dockerfile at the root directory of your project. 

{% highlight Docker linenos %}
FROM maven:3.6.0-jdk-11-slim AS build
COPY pom.xml /home/app/pom.xml
RUN mvn -f /home/app/pom.xml verify clean --fail-never

COPY src /home/app/src
RUN mvn -f /home/app/pom.xml package

# Install Tomcat    & openjdk 8 (openjdk has java and javac)
FROM tomcat:jdk8-openjdk
# Copy source files to tomcat folder structure
COPY --from=build /home/app/target/YOUR PROJECT.war /usr/local/tomcat/webapps/

EXPOSE 8080
{% endhighlight %}

There are two stages in the image building process: the first part builds the install the project's dependencies, and then compiles the source code. 