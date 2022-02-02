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
