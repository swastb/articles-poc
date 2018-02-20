# Guide to MicroService Deployment using Docker

Docker needs no introduction. If you feel you still need a guide, feel free to take a look here https://docs.docker.com/get-started/

Going forward I assume that you have docker CE installed in your machine. The concepts that we will be using here are as following

1. Dockerfile : It is a text document which contains all the instruction which are needed to build a docker image. Using instruction set of Docker file we can write steps which will copy files, do installation etc.
For more reference please visit 
https://docs.docker.com/engine/reference/builder/
2. Docker compose :- It is a tool using which we can create and spawn multiple containers. It helps to build the required environemnt with a single command. 

As shown in the micro service architecture diagram we will be creating individual container for each services. So below will be list of containers for our example. 
1. Config Server
2. EmployeeService
3. Employee Board Service
4. Employee Dashboard Service
5. Gateway Service

Docker configuration for Config Server

we will make sure of the following for the Config Server container. 

a) The container should container our config server jar file. Here we will pick the jar file from local machine. In real life scenario we should be pusing the jar file to an Artifact Repository Manager system such as Nexus or Artifactory and the container should download the file from the repo Manager. 
b) The config server should be available on the port 8888.
c) As mentioned above we will have config server to read the configuration from a file location, so we will make sure that those properties file can be retrieved even if the container goes down. 

1. Create a folder called config-repo which will contain the required properties file. 
 # mkdir config-repo
 # cd config-repo
 # echo "user.role=Developer" > user-dev.properties
 # echo "user.role=Production" > user-prod.properties

Come back to the parent folder with create a Docker file called Dockerfile. This docker file will create out base image which contain java.

 # cd ../
 # vi Dockerfile

put the below content

FROM alpine:edge
MAINTAINER javaonfly
RUN apk add --no-cache openjdk8

FROM: This Keyword, tells Docker to use a given image with its tag as build-base. 
MAINTAINER: A MAINTAINER is the author of an image
RUN: This command will installed openjdk8 in the system.

Execute the below command to create the base Docker image
#docker build --tag=alpine-jdk:base --rm=true .

After the base image is built successfully, it is time to create the docker image for Config Server. 

2. Create a folder called files and place the config server jar file in the directory 

3. Create a file called Dockerfile-configserver with the below content

FROM alpine-jdk:base
MAINTAINER javaonfly
COPY files/MicroserviceConfigServer.jar /opt/lib/
RUN mkdir /var/lib/config-repo
COPY config-repo /var/lib/config-repo
ENTRYPOINT ["/usr/bin/java"]
CMD ["-jar", "/opt/lib/MicroserviceConfigServer.jar"]
VOLUME /var/lib/config-repo
EXPOSE 9090

Here we have mentioned to built the image from previously create alpine-jdk image. We will copy the jar file named employeeconfigserver.jar in the /opt/lib location and also copy the config-repo to /root directory.  When the container starts up we want the config server to start running, hence the ENTRYPOINT and CMD is set to run the java command. We need to mount a volume to share the configuration files from outside the container, VOLUME command helps us to achieve that. The config server should be accessible to the outside world with the port 9090, thats why we have EXPOSE 9090. 

Now let us build the docker image and tag it as config-server

# docker build --file=Dockerfile-configserver --tag=config-server:latest --rm=true .

Now let us create a docker volume 
# docker volume create --name=config-repo

# docker run --name=config-server --publish=9090:9090 --volume=config-repo:/var/lib/config-repo config-server:latest

Phew !! Once we run the above command we should be able to see a docker container up and running. If we go to browser and hit the url http://localhost:9090/config/default/, we should be able access the properties as well. 

Similarly we need to create docker file for EurekaServer which will be running on port 9091. The Dockerfile for Eureka Server should be as below

FROM alpine-jdk:base
MAINTAINER javaonfly
COPY files/MicroserviceEurekaServer.jar /opt/lib/
ENTRYPOINT ["/usr/bin/java"]
CMD ["-jar", "/opt/lib/MicroserviceEurekaServer.jar"]
EXPOSE 9091

To build the image use the command as
docker build --file=Dockerfile-EurekaServer --tag=eureka-server:latest --rm=true .
docker run --name=eureka-server --publish=9091:9091 eureka-server:latest

Now its time deploy our actualy MicroServices, The steps should be similar but only thing we need to remember is our microservices are depedendent on ConfigServer and EurekaServer. So we always need to make sure that before we start our microservices the above 2 are up and running. So there are depdencies involved among containers. So it is time to expore Docker Compose. Its a beautiful way to make sure that containers are getting spawned maintaining a certain order. 

To do that we should write up Dockerfile for the rest of the containers. So below is the Dockerfile for  

Dockerfile-EmployeeSearch. 

FROM alpine-jdk:base
MAINTAINER javaonfly
RUN apk --no-cache add netcat-openbsd
COPY files/EmployeeSearchService.jar /opt/lib/
COPY EmployeeSearch-entrypoint.sh /opt/bin/EmployeeSearch-entrypoint.sh
RUN chmod 755 /opt/bin/EmployeeSearch-entrypoint.sh
EXPOSE 8080

Dockerfile-EmployeeDashboard

FROM alpine-jdk:base
MAINTAINER javaonfly
RUN apk --no-cache add netcat-openbsd
COPY files/EmployeeDashBoardService.jar /opt/lib/
COPY EmployeeDashBoard-entrypoint.sh /opt/bin/EmployeeDashBoard-entrypoint.sh
RUN chmod 755 /opt/bin/EmployeeDashBoard-entrypoint.sh
EXPOSE 8080

Dockerfile-ZuulServer

FROM alpine-jdk:base
MAINTAINER javaonfly
COPY files/EmployeeZuulService.jar /opt/lib/
ENTRYPOINT ["/usr/bin/java"]
CMD ["-jar", "/opt/lib/EmployeeZuulService.jar"]
EXPOSE 8084

Now let us create a  file called docker-compose.yml which will use all these dockerfiles to spawn our required environments. It will also make sure that the required containers are getting spawned maintaining correct order and they are interlinked. 
version: '2.2'
services:
    config-server:
        container_name: config-server
        build:
            context: .
            dockerfile: Dockerfile-configserver
        image: config-server:latest
        expose:
            - 9090
        ports:
            - 9090:9090
        networks:
            - emp-network
        volumes:
            - config-repo:/var/lib/config-repo

    eureka-server:
        container_name: eureka-server
        build:
            context: .
            dockerfile: Dockerfile-EurekaServer
        image: eureka-server:latest
        expose:
            - 9091
        ports:
            - 9091:9091
        networks:
            - emp-network
        
    EmployeeSearchService:
        container_name: EmployeeSearch
        build:
            context: .
            dockerfile: Dockerfile-EmployeeSearch
        image: employeesearch:latest
        environment:
            SPRING_APPLICATION_JSON: '{"spring": {"cloud": {"config": {"uri": "http://config-server:9090"}}}}'

        entrypoint: /opt/bin/EmployeeSearch-entrypoint.sh
        expose:
            - 8080
        ports:
            - 8080:8080
        networks:
            - emp-network
        links:
            - config-server:config-server
            - eureka-server:eureka-server
        depends_on:
            - config-server
            - eureka-server
        logging:
            driver: json-file
    EmployeeDashboardService:
        container_name: EmployeeDashboard
        build:
            context: .
            dockerfile: Dockerfile-EmployeeDashboard
        image: employeedashboard:latest
        environment:
            SPRING_APPLICATION_JSON: '{"spring": {"cloud": {"config": {"uri": "http://config-server:9090"}}}}'

        entrypoint: /opt/bin/EmployeeDashBoard-entrypoint.sh
        expose:
            - 8081
        ports:
            - 8081:8081
        networks:
            - emp-network
        links:
            - config-server:config-server
            - eureka-server:eureka-server
        depends_on:
            - config-server
            - eureka-server
        logging:
            driver: json-file
    ZuulServer:
        container_name: ZuulServer
        build:
            context: .
            dockerfile: Dockerfile-ZuulServer
        image: zuulserver:latest
        expose:
            - 8084
        ports:
            - 8084:8084
        networks:
            - emp-network
        links:
            - eureka-server:eureka-server
        depends_on:
            - eureka-server
        logging:
            driver: json-file
networks:
    emp-network:
        driver: bridge
volumes:
    config-repo:
        external: true

In the docker compose file below are few important entries. 

1. version :- it is a mandatory field, where we need to maintain of the version of the docker compose format.
2. services	:- Each entry defines the container we need to spawn. 
	a. build :- if mentioned then docker compose should build an image from the given dockerfile. 
	b. image:- name of the image which will be created
	c. networks:- Name of the network to be used. This name should be present at the networks section. 
	d. links:- This will create an internal link between the service and the mentioned service. Here the EmployeeSearch service needs to access the config server and eureka server, hence when we add those services in the links section, Docker will create link between these services and make the config server and eureka server accessible to EmployeeSearch container. 
	e. depends:- This is needed to maintain the order. The EmployeeSearch container depends on the Eureka and Config Server.Hence docker makes sure that the Eureka and Config Server containers are spawn before the EmployeeSearch Container is spawned. 
	

After creating the file let us build our images, create the required containers and also start with a single command:
docker-compose up --build

To Stop the complete environment we can use the command as 
docker-compose down		
		
The complete documentation for Docker compose is present in 
https://docs.docker.com/compose/

To summarize writing the Dockerfile and Docker Compose file is an one time activity. But it allows you the spawn a complete environment on demand at any time. 
