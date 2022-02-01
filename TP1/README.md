# TP Part 01 - Docker

## Database

### Why should we run the container with a flag -e to give the environment variables ?

On utilise l'option '-e' pour préciser les variables d'env comme le mot de passe car c'est des données sensibles. De cette manière c'est plus sécurisé.

### Why do we need a volume to be attached to our postgres container ?

On attache un volume à notre container afin de faire persister les données. Sans cela, la bdd se reset à chaque fois que le container s'éteint.

### 1.1 | Dockerfile

Le dockerfile :

```Dockerfile
FROM postgres:11.6-alpine # On pull l'image avec la version que l'on veut

ENV POSTGRES_DB=db \ # On setup les variables d'env
    POSTGRES_USER=usr \
    POSTGRES_PASSWORD=pwd

COPY init_db/*.sql /docker-entrypoint-initdb.d/ # On copie les scripts d'init de la bdd dans le répertoire adéquat dans Docker
```
 
Les commandes utilisées

```bash
  docker build -t meriau/postgres_db .
  docker run -p 5432:5432 -v postgresvolume:/var/lib/postgresql/data --name tp01_database -d meriau/postgres_db
```

## Backend API

### Run a Java program

On commence par créer un fichier Main.java avec un code qui output 'Hello World!' dans notre répertoire, et on fait ensuite le Dockerfile suivant :

```Dockerfile
  FROM openjdk:11-jre # On prend seulement un environnement qui nous permet de run du code car on ne compile pas
  COPY . /usr/src/myapp # On place le fichier .class obtenu via la commande javac Main.java
  WORKDIR /usr/src/myapp
  # Run java code with the JRE
  CMD ["java", "Main"] # On exécute le fichier .class
```

### Build and run a Java program (Multistage)

On fait comme l'étape précédente mais cette fois-ci on va commencer par build Main.java avant de le run.

```Dockerfile
  FROM openjdk:11 # On prend un environnement qui nous permet de compiler du code
  COPY . /usr/src/myapp
  WORKDIR /usr/src/myapp
  RUN javac Main.java # On compile pour avoir notre fichier .class

  FROM openjdk:11-jre
  COPY --from=0 /usr/src/myapp/Main.class .
  # Run java code with the JRE
  CMD ["java", "Main"]
```

### 1.2 | Why do we need a multistage build ? And explain each steps of this dockerfile

Chaque artefact va faire son propre build, et on choisit quel artefact sera potentiellement réutilisé plus tard. Ainsi, lorsque l'on fait un multistage build, on va avoir une image finale qui utilise uniquement ce dont elle a besoin, au lieu d'avoir plusieurs parties inutiles.

### Springboot application

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

### Springboot app avec bdd

Le Dockerfile :

```Dockerfile
  # Build
  FROM maven:3.6.3-jdk-11 AS myapp-build
  ENV MYAPP_HOME /opt/myapp
  WORKDIR $MYAPP_HOME
  COPY pom.xml .
  COPY src ./src
  RUN mvn dependency:go-offline # Permet d'éviter d'avoir à télécharger les dépendences à chaque build
  RUN mvn package -DskipTests


  # Run
  FROM openjdk:11-jre
  ENV MYAPP_HOME /opt/myapp
  WORKDIR $MYAPP_HOME
  COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar
  ENTRYPOINT java -jar myapp.jar
```

Le fichier Appliction.yaml :

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

Puis les commandes pour lancer les containers :

```bash
  docker build -t meriau/simple-api-main .
  docker run -p 8080:8080 --net network_tp01 --name tp01_api meriau/simple-api-main
  docker run -p 5432:5432 -v postgresvolume:/var/lib/postgresql/data --net network_tp01 --name tp01_db -d meriau/postgres_db
```


## Http Server

Ci-dessous le Dockerfile :

```Dockerfile
  FROM httpd:2.4
  COPY ./index.html /usr/local/apache2/htdocs/ # Copie la page qui sera affichée
  COPY ./httpd.conf /usr/local/apache2/conf/ # Copie la conf qui sera appliquée
```

Dans le fichier de conf on ajoute les lignes suivantes :

```txt
  ServerName localhost
  <VirtualHost *:80>
      ProxyPreserveHost On
      ProxyPass / http://api:8080/
      ProxyPassReverse / http://api:8080/
  </VirtualHost>
```

Et on décommente les lignes suivantes :

```txt
  LoadModule proxy_module modules/mod_proxy.so
  LoadModule proxy_http_module modules/mod_proxy_http.so
```

Les commandes pour run le container :

```bash
  docker build -t meriau/httpd
  docker run -p 80:80 --net network_tp01 --name tp01_http -d meriau/http
```

### Why do we need a reverse proxy ?

Cela sert d'intermédiaire entre un client et les ressources d'un serveur. Cela permet de gêrer plus simplement les requêtes (Autoriser, bloquer, rediriger).

## Docker-compose

### 1.3 | Document docker-compose most important commands

Docker-compose permet de permet de documenter / configurer les parties d'une application simplement.

Pour les commandes :

  - docker-compose up
  - docker-compose up -d
  - docker-compose down
  - docker-compose stop

### 1.4 | Document your docker-compose file

```Dockerfile
version: '3.7'
services: # On liste les services à installés
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

## Publish

### 1.5 | Document your publication commands and published images in dockerhub

```bash
  docker tag devops_tp01_db eriau/devops_tp01_db:1.0 # On va ajouter le tag 1.0 à notre image de la bdd
  docker push eriau/devops_tp01_db:1.0 # On push notre image avec le tag définit ci-dessus
  docker tag devops_httpd eriau/devops_httpd:1.0
  docker push eriau/devops_httpd:1.0
  docker tag devops_tp01_api eriau/devops_tp01_api:1.0
  docker push eriau/devops_tp01_api:1.0
```

### Why do we put our images into an online repository ?

Mettre les images sur un répertoire en ligne permet de travailler sur un autre poste, de partager avec d'autres gens... bref d'être plus flexible.

