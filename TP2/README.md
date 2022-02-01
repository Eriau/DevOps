# TP Part 02 - CI/CD

## 2-1 : What are testcontainers ?

C'est une librairie qui permet de créer automatiquement des containers docker pour tester.

## 2-2 : Document your Github Actions configurations.

J'ai commencé par ajouter mes credentials pour Docker hub dans Github Action.
On les mets dans l'onglet secret afin de ne pas les écrire en dur dans le code.

La ligne 'needs: build-and-test-backend' permet de s'assurer que le backend fonctionne avant de pull et build les images, ce qui peut prendre assez longtemps.
Sans cette ligne s'il y a une erreur dans le backend on ne le saura pas avant la fin.

Finalement on push les dockers images dans notre répertoire docker car cela permet de garder une image en lien avec le nouveau code que l'on a pu écrire.
Si l'on ne push pas les images on va avoir quelque chose d'obsolète.

main.yaml ci-dessous :

```yaml
name: CI devops 2022 CPE
on:
  #to begin you want to launch this job in main and develop
  push:
    branches: ['main', 'develop']
  pull_request:
  
jobs:
  test-backend:
    runs-on: ubuntu-18.04
    steps:
      #checkout your github code using actions/checkout@v2.3.3
      - uses: actions/checkout@v2.3.3
    
      #do the same with another action (actions/setup-java@v2) that enable to setup jdk 11
      - name: Set up JDK 11
        uses: actions/setup-java@v2
        with:
          java-version: '11'
          distribution: 'adopt'
  
      #analyse with sonar
      - name: Analyse with sonar
        env: 
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: mvn -B verify sonar:sonar -Dsonar.projectKey=dc599d03-0619-4425-bd4c-8753a5a8c7c7 -Dsonar.organization=3cc2c7a5-d688-481a-9f31-945ae33a35cc -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }} --file ./TP2/TP1_ToPipeLine/simple-api/pom.xml
      
      #finally build your app with the latest command
      - name: Build and test with Maven
        run: mvn clean verify --file ./TP2/TP1_ToPipeLine/simple-api
  
  # define job to build and publish docker image
  build-and-push-docker-image:
    needs: test-backend
    # run only when code is compiling and tests are passing
    runs-on: ubuntu-latest

    # steps to perform in job
    steps:
      - name: Login to DockerHub
        run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{ secrets.DOCKERHUB_PASSWORD }}
      
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Build image and push backend
        uses: docker/build-push-action@v2
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./TP2/TP1_ToPipeLine/simple-api
          # Note: tags has to be all lower-case
          tags: ${{secrets.DOCKERHUB_USERNAME}}/devops_tp01_api:1.1
          push: ${{ github.ref == 'refs/heads/main' }}
      
      - name: Build image and push database
        uses: docker/build-push-action@v2
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./TP2/TP1_ToPipeLine/database
          # Note: tags has to be all lower-case
          tags: ${{secrets.DOCKERHUB_USERNAME}}/devops_tp01_db:1.1
          push: ${{ github.ref == 'refs/heads/main' }}

      
      - name: Build image and push httpd
        uses: docker/build-push-action@v2
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./TP2/TP1_ToPipeLine/http
          # Note: tags has to be all lower-case
          tags: ${{secrets.DOCKERHUB_USERNAME}}/devops_httpd:1.1
          push: ${{ github.ref == 'refs/heads/main' }}
```

## Next step

Première amélioration : Cache dependencies (on rajoute la ligne cache ci-dessous).

```yaml
- name: Set up JDK 11
  uses: actions/setup-java@v2
  with:
    java-version: '11'
    distribution: 'adopt'
    cache: 'maven'
```

Deuxième amélioration : Pipeline // (on run trois jobs au lieu de trois steps)

```yaml
name: CI devops 2022 CPE
on:
  #to begin you want to launch this job in main and develop
  push:
    branches: ['main', 'develop']
  pull_request:
  
jobs:
  test-backend:
    runs-on: ubuntu-18.04
    steps:
      #checkout your github code using actions/checkout@v2.3.3
      - uses: actions/checkout@v2.3.3
    
      #do the same with another action (actions/setup-java@v2) setups jdk 11
      - name: Set up JDK 11
        uses: actions/setup-java@v2
        with:
          java-version: '11'
          distribution: 'adopt'
          cache: 'maven'
  
      #analyse with sonar
      - name: Analyse with sonar
        env: 
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: mvn -B verify sonar:sonar -Dsonar.projectKey=dc599d03-0619-4425-bd4c-8753a5a8c7c7 -Dsonar.organization=3cc2c7a5-d688-481a-9f31-945ae33a35cc -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }} --file ./TP2/TP1_ToPipeLine/simple-api/pom.xml
      
      #finally build your app with the latest command
      - name: Build and test with Maven
        run: mvn clean verify --file ./TP2/TP1_ToPipeLine/simple-api
        

  build-and-push-databases:
    needs: test-backend
    runs-on: ubuntu-latest
    env:
      working-directory: ./TP2/TP1_ToPipeLine/database
    steps:
      - name: Checkout code
        uses: actions/checkout@v2
      
      - name: Login to DockerHub
        run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{secrets.DOCKERHUB_PASSWORD }}
      
      - name: Build image and push database
        uses: docker/build-push-action@v2
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./TP2/TP1_ToPipeLine/database
          # Note: tags has to be all lower-case
          tags: ${{secrets.DOCKERHUB_USERNAME}}/devops_tp01_db:1.1
          push: ${{ github.ref == 'refs/heads/main' }}

  build-and-push-simple-api:
    needs: test-backend
    runs-on: ubuntu-latest
    env:
      working-directory: ./TP2/TP1_ToPipeLine/simple-api
    steps:
      - name: Checkout code
        uses: actions/checkout@v2
      
      - name: Login to DockerHub
        run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{secrets.DOCKERHUB_PASSWORD }}
 
      - name: Build image and push backend
        uses: docker/build-push-action@v2
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./TP2/TP1_ToPipeLine/simple-api
          # Note: tags has to be all lower-case
          tags: ${{secrets.DOCKERHUB_USERNAME}}/devops_tp01_api:1.1
          push: ${{ github.ref == 'refs/heads/main' }}
 
  build-and-push-http:
    needs: test-backend
    runs-on: ubuntu-latest
    env:
      working-directory: ./TP2/TP1_ToPipeLine/http
    steps:
      - name: Checkout code
        uses: actions/checkout@v2
      
      - name: Login to DockerHub
        run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{secrets.DOCKERHUB_PASSWORD }}
      
      - name: Build image and push httpd
        uses: docker/build-push-action@v2
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./TP2/TP1_ToPipeLine/http
          # Note: tags has to be all lower-case
          tags: ${{secrets.DOCKERHUB_USERNAME}}/devops_httpd:1.1
          push: ${{ github.ref == 'refs/heads/main' }}

```

Troisième amélioration : Ne pas run test backend si aucun fichier dans le dosier /simple-api n'a changé.

TODO