# s8_devops

# Database

Il faut utiliser l'option `-e` pour préciser les variables d'environnements comme le mot de passe car ce sont des données sensibles. C'est donc plus sécurité que de les mettre en clair dans le Dockerfile.

Il est nécessaire d'attacher un volume à notre container afin de faire persister les données. Sinon, la base de données se réinitialiserait à chaque fois que le container redémarre.

Dockerfile utilisé: 

```Dockerfile
FROM postgres:11.6-alpine

ENV POSTGRES_DB=db \
    POSTGRES_USER=usr \
    POSTGRES_PASSWORD=pwd

COPY init_db/*.sql /docker-entrypoint-initdb.d/
```

Commandes utilisées depuis ```s8_devops/database```:

```docker
docker build -t meriau/postgres_db .

docker run -p 5432:5432 -v /mnt/c/Users/V3D/Desktop/COURS/DevOps/database/data:/var/lib/postgresql/data -d meriau/postgres_db
```

# Backend API

Dockerfile app java première étape : 

```Dockerfile
FROM openjdk:11-jre
COPY . /usr/src/myapp
WORKDIR /usr/src/myapp
# Run java code with the JRE
CMD ["java", "Main"]
```

App java deuxième étape:
```Dockerfile
FROM openjdk:11
COPY . /usr/src/myapp
WORKDIR /usr/src/myapp
RUN javac Main.java
FROM openjdk:11-jre
COPY --from=0 /usr/src/myapp/Main.class .
# Run java code with the JRE
CMD ["java", "Main"]
```

Simple API:

On utilise un multistage build afin de créer des images de plus petite taille. 

```Dockerfile
# Build
FROM maven:3.6.3-jdk-11 AS myapp-build
# ENV est une variable d'environnement
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
# On copie le fichier de dépendences du projet + les sources
COPY pom.xml .
COPY src ./src
# Compile le projet
RUN mvn package -DskipTests

# Run
FROM openjdk:11-jre
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
# On copie le fichier .jar créé à l'étape précedente et on l'exécute
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar
ENTRYPOINT java -jar myapp.jar
```

Simple Api avec db :

```Dockerfile
# Build
FROM maven:3.6.3-jdk-11 AS myapp-build
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY pom.xml .
COPY src ./src
RUN mvn dependency:go-offline
RUN mvn package -DskipTests


# Run
FROM openjdk:11-jre
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar
ENTRYPOINT java -jar myapp.jar
```

Appliction.yaml ->

```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          lob:
            non_contextual_creation: true
    generate-ddl: false
    open-in-view: true
  datasource:
    url: jdbc:postgresql://postgresdb:5432/db
    username: usr
    password: pwd
    driver-class-name: org.postgresql.Driver
management:
 server:
   add-application-context-header: false
 endpoints:
   web:
     exposure:
       include: health,info,env,metrics,beans,configprops

```

Commandes pour run les conatainers -> 

docker build -t meriau/simple-api-main .
docker run -p 8080:8080 --net network_tp01 --name tp01_api meriau/simple-api-main
docker run -p 5432:5432 -v postgresvolume:/var/lib/postgresql/data --net network_tp01 --name tp01_db -d meriau/postgres_db

# Http Server

docker build -t meriau/httpd
docker run -p 80:80 --net network_tp01 --name tp01_http -d meriau/http

Pourquoi un reverse-proxy ? -> Cela sert d'intermédiaire entre un client et les ressources d'un server. Cela permet de gêrer plus simplement les requêtes (Autoriser, bloquer, rediriger).

# Docker-compose

Docker-compose permet de permet de documenter / configurer les parties d'une application simplement.

Pour les commandes :

  - docker-compose up
  - docker-compose up -d
  - docker-compose down
  - docker-compose stop

```Dockerfile
version: '3.7'
services:
  tp01_api: # Configure le backend
    build:
      ./simple-api-main/simple-api # Chemin vers le Dockerfile
    networks: # Network qui lie les applications
      - network_tp01
    depends_on: # Ce service dépend de la bdd
      - tp01_db
    container_name: api # Nom du container qui sera crée

  tp01_db: # Configure la bdd
    build:
      ./database
    volumes: # Volume qui permet d avoir des données qui persiste
      - postgresvolume:/var/lib/postgresql/data
    networks:
      - network_tp01
    container_name: postgresdb

  httpd:
    build:
      ./http
    ports: # Permet de map le port sur la machine locale
      - "80:80"
    networks:
      - network_tp01
    depends_on:
      - tp01_api

networks:
  network_tp01:

volumes:
  postgresvolume:
    external: true
```

# Publish

docker tag devops_tp01_db eriau/devops_tp01_db:1.0
docker push eriau/devops_tp01_db:1.0
docker tag devops_httpd eriau/devops_httpd:1.0
docker push eriau/devops_httpd:1.0
docker tag devops_tp01_api eriau/devops_tp01_api:1.0
docker push eriau/devops_tp01_api:1.0

Mettre les images sur un répertoire en ligne permet de travailler sur un autre poste, de partager avec d'autres gens... bref d'être plus flexible.

