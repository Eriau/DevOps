# TP Part 03 - Ansible

## Intro

### 3-1 Document your inventory and base commands

Inventory file :

```yaml
    all:
        vars:
            ansible_user: centos # On spécifie l'user avec lequel on veut se connecter en ssh
            ansible_ssh_private_key_file: ../.ssh/id_rsa # Chemin vers la clé privée
        children:
            prod:
                hosts: mathieu.eriau.takima.cloud # Host sur lequel on veut se connecter en ssh
```

Commands :

```bash
    ansible all -i inventories/setup.yml -m ping # Simple ping
    ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*" # Permet de savoir la distribution de notre hôte
    ansible all -i inventories/setup.yml -m yum -a "name=httpd state=absent" --become # Remove apache de notre hôte
```

## Playbook

Playbook yaml :

```yaml
- hosts: all
  gather_facts: no # This is data gathered about target nodes, which don't interest us at the moment
  become: yes # On se met en root
  roles: # On indique les rôles à utiliser
    - docker
  tasks:
    - name: Test connection
      ping:
```

### 3-2 Document your playbook

First Playbook : Un playbook qui permet de ping tous les hôtes présents dans le fichier d'inventaire par défaut (ou bien celui fourni via l'option -i de la commande ansible-playbook).

```yaml
- hosts: all
  gather_facts: false
  become: yes
  tasks:
    - name: Test connection
      ping:

```

Advanced Playbook : Cette fois ci on rajoute toutes les étapes nécessaires à l'installation de Docker directement dans le fichier playbook.yaml.

```yaml
- hosts: all
  gather_facts: false
  become: yes
  tasks:
    - name: Test connection
      ping:
        
  # Install Docker
  tasks:
    - name: Clean packages
      command:
        cmd: dnf clean -y packages
    
    - name: Install device-mapper-persistent-data
      dnf:
        name: device-mapper-persistent-data
        state: latest
    
    - name: Install lvm2
      dnf:
        name: lvm2
        state: latest
  
    - name: add repo docker
      command:
        cmd: sudo dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
      
    - name: Install Docker
      dnf:
        name: docker-ce
        state: present
  
  - name: install python3
    dnf:
      name: python3
  
  - name: Pip install
    pip:
      name: docker
  
  - name: Make sure Docker is running
    service: name=do
```

Using role : On fait un rôle "docker" à l'aide de la commande 'ansible-galaxy init roles/docker', et on met tous ce qu'il faut pour installer docker dans ce rôle. Cela permet d'avoir un playbook comme ci-dessous, beaucoup plus clair.

```yaml
- hosts: all
  gather_facts: false
  become: yes
  tasks:
    - name: Test connection
      ping:
        
  # Install Docker
  roles:
    - docker
```

### 3-3 Document your docker_container tasks configuration

Le playbook : 

```yaml
- hosts: all
  gather_facts: no # This is data gathered about target nodes, which don't interest us at the moment
  become: yes # On se met en root
  roles: # On indique les rôles à utiliser
    - install-docker
    - create-network
    - launch-database
    - launch-app
    - launch-proxy
```

Le rôle create-network :

```yaml
- name: Create a network
  docker_network:
    name: network_tp03
```

Le rôle launch-app :

```yaml
- name: Remove container app if already exists
  docker_container:
    name: api
    state: absent

- name: Run App
  docker_container:
    name: api
    image: eriau/devops_tp01_api:1.0 # On va chercher l'image
    networks: # On spécifie le network
      - name: network_tp03
```

Le rôle launch-database :

```yaml
- name: Remove container db if already exists
  docker_container:
    name: postgresdb
    state: absent

- name: Run DB
  docker_container:
    name: postgresdb
    image: eriau/devops_tp01_db:1.0
    networks:
      - name: network_tp03
    volumes: # On fournit un chemin pour le volume
      - /data
```

Le rôle create-proxy :

```yaml
- name: Remove container http if already exists
  docker_container:
    name: http
    state: absent

- name: Run HTTPD
  docker_container:
    name: http
    image: eriau/devops_httpd:1.0
    networks:
      - name: network_tp03
    ports: # On map les ports
      - "80:80"
```

## Front

Afin d'utiliser le front dont le code est disponible à l'adresse : https://github.com/Mathilde-lorrain/devops-front il a fallu modifier plusieurs fichiers.

Tout d'abord **httpd.conf** :

```txt
  ServerName localhost
  <VirtualHost *:80>
      ProxyPreserveHost On
      ProxyPass /api http://api:8080/
      ProxyPassReverse /api http://api:8080/
      ProxyPass / http://front:80/
      ProxyPassReverse / http://front:80/
  </VirtualHost>
```

On rajoute les deux dernières lignes afin de rediriger les requêtes sur localhost vers le front sur le port 80, et lorsque l'on fait des requêtes sur localhost/api on renvoie sur l'api disponible sur le port 8080.

Puis on modifie le **docker-compose.yaml** afin de tester localement :

```yaml
  front:
    build:
      ./front
    networks:
      - network_tp01
    depends_on:
      - tp01_api
```

On rajoute la partie ci-dessus dans les services.

Il faut aussi modifier la variable d'environnement présente dans le fichier *.env.production* pour que ça cherche bien sur le bon url avec le bon port : **VUE_APP_API_URL=mathieu.eriau.takima.cloud:80/api**

Une fois cela fait, le front est disponible sur localhost:80.

## Continuous deployment

Maintenant il faut publier l'image du front, la récupérer via Ansible et la lancer sur notre serveur.

Pour publier l'image du front :

```yaml
name: Build & Push front
on:
  #to begin you want to launch this job in main and develop
  push:
    branches: ['main']
  pull_request:

jobs:
  build-and-push-front:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v2
      
      - name: Login to DockerHub
        run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{secrets.DOCKERHUB_PASSWORD }}
      
      - name: Build image and push database
        uses: docker/build-push-action@v2
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./TP2/TP1_ToPipeLine/front
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/devops_front:1.0
          push: ${{ github.ref == 'refs/heads/main' }}
 
```

Pour la récupérer via Ansible :

```yaml
---
# tasks file for roles/launch-front

- name: Remove container front if already exists
  docker_container:
    name: front
    state: absent

- name: Run front
  docker_container:
    name: front
    image: eriau/devops_front:latest
    networks:
      - name: network_tp03
```

Et finalement on test sur le serveur.