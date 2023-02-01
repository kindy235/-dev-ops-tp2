BAH Aboubacar & ACET Emre

# Compte rendu

## Contruction d'images docker
----------------
### BDD - Postgres

 - Dockerfile

```Dockerfile
FROM postgres:14.1-alpine AS postgres-bdd

ENV POSTGRES_DB=test_db \
   POSTGRES_USER=kindy \
   POSTGRES_PASSWORD=kindy
```

 - Géneration de l'image

```
docker build -t postgre-bdd .
```

 - Vérification

```
❯ docker image ls
REPOSITORY               TAG       IMAGE ID       CREATED         SIZE
postgre-bdd              latest    d129cb2bb16a   13 months ago   209MB
```

### Serveur Web - HTTPD

 - Dockerfile

```Dockerfile
FROM httpd:2.4 AS web-server
COPY ./index.html /usr/local/apache2/htdocs/
```

 - Géneration de l'image

```
docker build -t web-server .
```

 - Vérification

```
❯ docker image ls
REPOSITORY               TAG       IMAGE ID       CREATED         SIZE
web-server               latest    43f4509bb86d   2 hours ago     145MB
```

## Lancement des services
--------
### Création du network

```
docker network create app-network
```

### Base de données - Postgres 
 
 - lancement d'adminer

```
docker run -p "8090:8080" --net=app-network --name=adminer -d adminer
```

- lancement de Postgre

```
❯ docker run -itd --network app-network -p 5432:5432 -v /data:/var/lib/postgresql/data --name postgres-bdd -d postgre-bdd
```
### Serveur Web - httpd

```
❯ docker run -itd --network app-network -p 5000:80 --name web-server -d web-server
```

```
❯ docker ps
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS          PORTS                    NAMES
5f6a46e626b2   postgre-bdd   "docker-entrypoint.s…"   16 minutes ago   Up 16 minutes   0.0.0.0:5432->5432/tcp   postgres-bdd
67971f6d6fce   web-server    "httpd-foreground"       39 minutes ago   Up 39 minutes   0.0.0.0:5000->80/tcp     gallant_merkle
e3f1a4bdcbeb   adminer       "entrypoint.sh php -…"   3 hours ago      Up 45 minutes   0.0.0.0:8090->8080/tcp   adminer
```

## Backend API

### Construction d'image pour l'exécution d'un programme

- Porgramme Java Main.java

```java
public class Main {

   public static void main(String[] args) {
       System.out.println("Hello World!");
   }
}
```
1. Compilation

```
javac Main.java
```

2. Dockerfile pour l'éxecution

```Dockerfile
FROM openjdk:11 as java-jdk
COPY Main.class Main.class
CMD [ "java", "Main"]
```

3. Construction de l'image

```
docker build -t java-jdk .
```

4. Exécution de l'image

```
❯ docker run java-jdk
Hello World!
```

### Construction d'image Multistage

- Porgramme Java `Main.java`

```java
public class Main {

   public static void main(String[] args) {
       System.out.println("Hello World!");
   }
}
```

1. Dockerfile Compilation et l'éxecution

```Dockerfile
FROM openjdk:11
COPY Main.java Main.java
RUN javac Main.java

FROM openjdk:11-jre
COPY --from=0 ./Main.class .
CMD [ "java", "Main"]
```

2. Construction de l'image

```
docker build -t java-jdk .
```

3. Exécution de l'image

```
❯ docker run java-jdk
Hello World!
```

Why do we need a multistage build? And explain each step of this dockerfile ?

### Backend simple api

1. Dockerfile

```Dockerfile
# Build
FROM maven:3.8.6-amazoncorretto-17 AS myapp-build
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY greeting greeting
RUN mvn package -DskipTests

# Run
FROM amazoncorretto:17
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar

ENTRYPOINT java -jar myapp.jar
```

2. Construction de l'image Multistage

```
docker build -t greeting .
```

3. Vérification 

```
❯ docker image list
REPOSITORY               TAG       IMAGE ID       CREATED         SIZE
greeting                 latest    d264b30b3f76   8 minutes ago   480MB
```

4. Lancement de l'API

```
❯ docker run -p 8000:8080 --name simpleapi -d greeting
1eb31a85662d782f84c78da3849d02753369efcfd5cb4489bae2f28260ab683c
```

```
❯ docker ps
CONTAINER ID   IMAGE         COMMAND                  CREATED         STATUS         PORTS                    NAMES
1eb31a85662d   greeting      "/bin/sh -c 'java -j…"   4 seconds ago   Up 3 seconds   0.0.0.0:8000->8080/tcp   simpleapi
4abf567feab3   web-server    "httpd-foreground"       19 hours ago    Up 3 hours     0.0.0.0:5000->80/tcp     web-server
5f6a46e626b2   postgre-bdd   "docker-entrypoint.s…"   20 hours ago    Up 3 hours     0.0.0.0:5432->5432/tcp   postgres-bdd
```

1-2 Why do we need a multistage build? And explain each step of this dockerfile.


### API avec BDD

```Dockerfile
# Build
FROM maven:3.8.6-amazoncorretto-17 AS myapp-build
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY greeting greeting
RUN mvn package -DskipTests

# Run
FROM amazoncorretto:17
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar

ENTRYPOINT java -jar myapp.jar
```

```
❯ docker build -t kindy/api .
```

```
❯ docker image list
REPOSITORY               TAG       IMAGE ID       CREATED             SIZE
kindy/api                latest    7b8a84d11092   5 minutes ago       502MB
```

```
❯ docker run -p "10000:8080" --net=app-network --name api -d kindy/api
cb6da56adefdbb7282ac88e56567f3f8252da13b3bac464f5684f65b75831394
```

```
❯ docker ps
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS          PORTS                     NAMES
cb6da56adefd   kindy/api     "/bin/sh -c 'java -j…"   4 seconds ago    Up 3 seconds    0.0.0.0:10000->8080/tcp   api
5218f1ebb17e   adminer       "entrypoint.sh php -…"   51 minutes ago   Up 51 minutes   0.0.0.0:8090->8080/tcp    adminer
4abf567feab3   web-server    "httpd-foreground"       21 hours ago     Up 4 hours      0.0.0.0:5000->80/tcp      web-server
5f6a46e626b2   postgre-bdd   "docker-entrypoint.s…"   21 hours ago     Up 4 hours      0.0.0.0:5432->5432/tcp    postgres-bdd
```

### Reverse Proxy



1-3 Document docker-compose most important commands. 1-4 Document your docker-compose file.

1-5 Document your publication commands and published images in dockerhub