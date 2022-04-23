# mini-projet-docker
Déploiement d'un micro-service flask python

**Objectif du projet :** Valider les acquis sur le module **Docker**.

Dans ce mini-projet-docker, il est question pour nous d'embarquer une application dans un container docker. Cette application se trouve dans repository git du lien suivant [dirane](https://github.com/diranetafen/student-list)

# Résolution explicative du projet

## Partie 1: Build et test de l'application
Pour ce faire:
- Nous allons mettre en place un **Dockerfile** selon les instructions données dans l'énoncé.
- Cette application sera ensuite tester via un curl.

## Partie 2: Mise en place de l'Infra As Code (IAC)
Pour ce faire:
- Nous allons utiliser l'outil **docker-compose**. Cet outil va nous permettre d'automatiser le déploiement de l'application que nous allons conteneuriser.
- Nous allons aussi rajouter l'IHM avec un container provenant de l'image **php:apache**. Cette **IHM** va nous permettre de tester le fonctionnel de notre application via un navigateur.

En résumé, notre **docker-compose** devra nous déployer deux containers qui sont:
- Container 1: notre application (api développée en flask python)
- Container 2: L'**IHM** qui part taper sur le **container 1** afin de nous donner la possibilité de tester le fonctionnel de notre application.

## Partie 3: Docker Registry

Une fois que nous aurons notre application embarquée dans le container docker, on va créer l'image. Cette dernière est en locale.

Il sera donc question d'enterriner cette image dans un **registry**. Pas le **dockerhub** car il est public et nous avons juste droit droit à un seul compte privé et un gratuit.

Nous allons donc déployer un troisième container qui aura pour rôe d'être le **registry** où nous pourrons envoyer notre image nouvellement buildée dans ce **Registre** local.

Pour communiquer, nos quatre conainer doivent être dans un réseau de type **Bridge**. Ce dernier nous permet d'exposer nos application **IHM** et **IHM Registry** à l'extérieur. Il faudra mettre les règles d'exposition des ports.

Seul les containers **IHM** seront exposés. Par exemple le container **IHM** va écouter via le port 80 et **IHM Registry** via le port 8082

**Fin du mini-projet-docker**.

## Illustaration schématique de la solution


# Réalisation
## Partie 1: Build et test
### étape 1: Nous allons déployer la machine qui contient docker
Pour ce déploiement, je vais utiliser:
- L'hyperviseur **VirtualBox** installé
- Le script d'installation de **docker**
- Le provider **vagrant** qui va automatiser le déploiement des VM et l'installation du container docker.

Pour créer VM docker dans VirtualBox, je tape la commande suivante:
```
$ vagrant up --provision
```
Une fois ma VM mise en place, je me connecte à celle-ci via **ssh** (il va utiliser une clé privée qu'il a eu à générer) par la commande:
```
$ vagrant ssh
PS C:\Users\ib\Desktop\mini-projet-docker\vagrant\docker> vagrant ssh
```
```
Last login: Wed Apr  6 17:45:34 2022 from 10.0.2.2
[vagrant@docker ~]$
```
Je vais vérifier que ma VM est véritablement déployée
```
[vagrant@docker ~]$ uptime
 20:11:38 up  2:36,  1 user,  load average: 0.00, 0.01, 0.05
```
Je vérifie l'OS 
```
[vagrant@docker ~]$ cat /etc/redhat-release 
CentOS Linux release 7.9.2009 (Core)
```
Je vérifie si j'ai au pluss une image
```
[vagrant@docker ~]$ docker images
```
Je vérifie si j'ai au plus un container qui tourne
```
[vagrant@docker ~]$ docker ps -a
```
### étape 2: Phase du build
- Je vais télécharger le repository git du projet
  - J'installe d'abord **git** s'il n'est pas encore
```
[vagrant@docker ~]$ sudo yum install git -y
```
  - Je vais cloner
```
[vagrant@docker ~]$ git clone https://github.com/diranetafen/student-list.git
```
```
[vagrant@docker ~]$ git clone https://github.com/diranetafen/student-list.git
Cloning into 'student-list'...
remote: Enumerating objects: 27, done.
remote: Counting objects: 100% (8/8), done.
remote: Compressing objects: 100% (6/6), done.
remote: Total 27 (delta 4), reused 3 (delta 2), pack-reused 19
Unpacking objects: 100% (27/27), done.
```
Je vais liste tout ce qui se trouve dans le repertoire
```
[vagrant@docker ~]$ ls
get-docker.sh  student-list
```
```
[vagrant@docker ~]$ ll
total 20
-rw-r--r--. 1 root    root    18617 Apr  6 17:38 get-docker.sh
drwxrwxr-x. 5 vagrant vagrant    94 Apr  6 20:22 student-list 
```
Je me positionne dans le repertoire de travail (où se trouve l'application à builder)
```
[vagrant@docker ~]$ cd student-list/
[vagrant@docker student-list]$
```
Je liste le contenu du repertoire de travail
```
[vagrant@docker student-list]$ ll
total 8
-rw-rw-r--. 1 vagrant vagrant    0 Apr  6 20:22 docker-compose.yml
-rw-rw-r--. 1 vagrant vagrant 7133 Apr  6 20:22 README.md
drwxrwxr-x. 2 vagrant vagrant   70 Apr  6 20:22 simple_api        
drwxrwxr-x. 2 vagrant vagrant   23 Apr  6 20:22 website
```
**Pour builder l'image**, il me faut un **Dockerfile**

Je vais ouvrir le **Dockerfile** qui se trouve dans **simple_api**
```
[vagrant@docker student-list]$ cd simple_api/
[vagrant@docker simple_api]$ ll
total 8
-rw-rw-r--. 1 vagrant vagrant    0 Apr  6 20:22 Dockerfile      
-rw-rw-r--. 1 vagrant vagrant   39 Apr  6 20:22 student_age.json
-rwxrwxr-x. 1 vagrant vagrant 1667 Apr  6 20:22 student_age.py
```
J'ouvre le **Dockerfile**
```
[vagrant@docker simple_api]$ vi Dockerfile
```
Puis je vais renseigner ce **Dockerfile** des consignes de l'énoncé
```
FROM python:2.7-stretch
MAINTAINER Daniel MEDOU
ADD student_age.py /
RUN apt-get update -y && apt-get install python-dev python3-dev libsasl2-dev python-dev libldap2-dev libssl-dev -y
RUN pip install flask==1.1.2 flask_httpauth==4.1.0 flask_simpleldap python-dotenv==0.14.0
VOLUME [ "/data" ]
EXPOSE 5000
CMD [ "python", "./student_age.py" ]
```
Mon **Dockerfile** est prêt à être lancé.

Maintenant, je vais passer à la phase de **build**.

Pour ce faire, je suis déjà positionner dans le repètoire où se trouve mon **Dockerfile**, je tape la commande
```
[vagrant@docker simple_api]$ docker build -t api-pozos:V1 .
```
Le build s'est passé avec succès
```
Successfully built ddd9133d5014
Successfully tagged api-pozos:V1
```
Je vérifie que mon image est vraiment présente.
```
[vagrant@docker simple_api]$ docker images
REPOSITORY   TAG           IMAGE ID       CREATED         SIZE
api-pozos    V1            ddd9133d5014   4 minutes ago   1.13GB
python       2.7-stretch   e71fc5c0fcb1   23 months ago   928MB
```
Mon image est bien présente avec le tag V1. Elle est partie d'une image python 2.7 qui a éé téléchargée.

Mon image étant présente, il me reste juste à la tester.

### étape 3: Phase de test

Pour tester, je vais me créer un réseau. Sans option particulière donnée, c'est un réseau detype **Bridge**
```
[vagrant@docker simple_api]$ docker network create pozos_network
```
Je vérifie que mon réseau a été créé.
```
[vagrant@docker simple_api]$ docker network ls
```
```
[vagrant@docker simple_api]$ docker network ls
NETWORK ID     NAME            DRIVER    SCOPE
98b7ca2ba69f   bridge          bridge    local
dedef9dce2f6   host            host      local
dcfc2cb9e6e6   none            null      local
e29f6db30a97   pozos_network   bridge    local
```

Je vais lancer (déployer) mon premier container dans ce réseau **pozos_network**

```
[vagrant@docker simple_api]$ docker run -d --network pozos_network --name test_api_pozos -v ${PWD}/student_age.json:/data/student_age.json -p 4000:5000 api-pozos:V1

```

Je vérifie que mon container a bien été déployé
```
[vagrant@docker simple_api]$ docker ps -a
```
J'ai bien mon container 
```
CONTAINER ID   IMAGE          COMMAND                  CREATED          STATUS          PORTS                                       NAMES
42d6bbfbace1   api-pozos:V1   "python ./student_ag…"   22 seconds ago   Up 19 seconds   0.0.0.0:4000->5000/tcp, :::4000->5000/tcp   test_api_pozos
```

Dans l'énoncé, on nous a donné la commande qui va nous permettre de tester l'image

```
curl -u toto:python -X GET http://<host IP>:<API exposed port>/pozos/api/v1.0/get_student_ages
```

Etant en local, je vais tester avec l'ip 127.0.0.1 et le port d'écoute 4000
```
curl -u toto:python -X GET http://127.0.0.1:4000/pozos/api/v1.0/get_student_ages
```
C'est ok. J'ai bien mes données

```
{
  "student_ages": {
    "alice": "12",
    "bob": "13"
  }
}
```

**Fin de la phase de build et test**


## Partie 2: Infrastructure As Code (IAC)
L'objectif de cette partie est de mettre en place un **docker-compose** qui, nous permettra d'automatiser le déploiement de notre infrastructure.

Pour procéder à cette étape, nous allons déjà nous positionner dans le répertoire de travail où se trouve le **docker-compose.yml**
```
[vagrant@docker simple_api]$ cd ..
```
```
[vagrant@docker student-list]$ ll
total 8
-rw-rw-r--. 1 vagrant vagrant    0 Apr  6 20:22 docker-compose.yml
-rw-rw-r--. 1 vagrant vagrant 7133 Apr  6 20:22 README.md
drwxrwxr-x. 2 vagrant vagrant   70 Apr  6 20:40 simple_api
drwxrwxr-x. 2 vagrant vagrant   23 Apr  6 20:22 website
```
j'ouvre le fichier **docker-compose.yml** avec vi

```
[vagrant@docker student-list]$ vi docker-compose.yml
```
Puis dans ce fichier, je vais copier et coller le code ci-dessous
```
version: '2.3'
services:
    web-pozos1:
        image: 'php:apache'
        depends_on:
            - api
        volumes:
            - ./website:/var/www/html
        ports:
            - "8082:80"
        networks:
            - api_network_pozos
        environment:
            - USERNAME=toto
            - PASSWORD=python
        networks:
            - api_network_pozos
    api-pozos1:
        image: daniel-pozos:V1
        ports:
            - "4000:5000"
        volumes:
            - ./simple_api/student_age.json:/data/student_age.json
        networks:
            - api_network_pozos


networks:
  api_network_pozos:
```

Une fois mon **docker-compose.yml** mise en place, je vais supprimer le docker de test **test_api_pozos** ainsi que le réseau créé **pozos_network**
```
$ docker ps -a
$ docker rm -f [id_docker_test]
```
```
$ docker network rm pozos_network
```
```
[vagrant@docker student-list]$ docker ps -a
CONTAINER ID   IMAGE          COMMAND                  CREATED          STATUS          PORTS                                       NAMES
42d6bbfbace1   api-pozos:V1   "python ./student_ag…"   32 minutes ago   Up 32 minutes   0.0.0.0:4000->5000/tcp, :::4000->5000/tcp   test_api_pozos
[vagrant@docker student-list]$ docker rm -f 42d6bbfbace1
42d6bbfbace1
[vagrant@docker student-list]$ docker ps -a
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
[vagrant@docker student-list]$ docker network ls
NETWORK ID     NAME            DRIVER    SCOPE
98b7ca2ba69f   bridge          bridge    local
dedef9dce2f6   host            host      local
dcfc2cb9e6e6   none            null      local
e29f6db30a97   pozos_network   bridge    local
[vagrant@docker student-list]$ docker network rm pozos_network
pozos_network
[vagrant@docker student-list]$ docker network ls
NETWORK ID     NAME      DRIVER    SCOPE
98b7ca2ba69f   bridge    bridge    local
dedef9dce2f6   host      host      local
dcfc2cb9e6e6   none      null      local
```

**Je vais lancer le déploiement de mon docker-compose**

Je vais d'abord installer la commande **docker-compose**. les instruction se trouvent dans le site `https://docs.docker.com/compose/install/`
```
$  sudo curl -L "https://github.com/docker/compose/releases/download/v2.4.1/docker-compose-linux-x86_64" -o /usr/local/bin/docker-compose
```
```
sudo chmod +x /usr/local/bin/docker-compose
```
Je vérifie que doncker-composer est bien installé
```
[vagrant@docker student-list]$ docker-compose --version
Docker Compose version v2.4.1
```

**Déploiement proprement dite**
```
[vagrant@docker student-list]$ docker-compose up -d
```
Image téléchargée, reéseau créé et mes deux containers créés. Rendu ci-dessous
```
[+] Running 14/14
 ⠿ web-pozos1 Pulled                                                                                                                                 135.1s 
   ⠿ c229119241af Pull complete                                                                                                                       53.1s
   ⠿ 47e86af584f1 Pull complete                                                                                                                       53.5s 
   ⠿ e1bd55b3ae5f Pull complete                                                                                                                      110.9s 
   ⠿ 1f3a70af964a Pull complete                                                                                                                      111.3s 
   ⠿ 0f5086159710 Pull complete                                                                                                                      120.6s 
   ⠿ 7d9c764dc190 Pull complete                                                                                                                      121.0s 
   ⠿ ec2bb7a6eead Pull complete                                                                                                                      121.3s 
   ⠿ 8db7e76a80f5 Pull complete                                                                                                                      122.8s 
   ⠿ 6dd03167e257 Pull complete                                                                                                                      123.2s 
   ⠿ 227c07849d70 Pull complete                                                                                                                      129.9s 
   ⠿ c92237a494ba Pull complete                                                                                                                      130.3s 
   ⠿ 2f6a621ebcfb Pull complete                                                                                                                      130.6s 
   ⠿ 0f380edbfc61 Pull complete                                                                                                                      131.0s 
[+] Running 3/3
 ⠿ Network student-list_api_network_pozos  Created                                                                                                     0.4s 
 ⠿ Container student-list-api-pozos1-1     Started                                                                                                     4.3s
 ⠿ Container student-list-web-pozos1-1     Started 
```

Je vérifie que mes deux containers existent et qu'ils sont en mode **running**
```
[vagrant@docker student-list]$ docker-compose ps
```
```
NAME                        COMMAND                  SERVICE             STATUS              PORTS
student-list-api-pozos1-1   "python ./student_ag…"   api-pozos1          running             0.0.0.0:4000->5000/tcp, :::4000->5000/tcp
student-list-web-pozos1-1   "docker-php-entrypoi…"   web-pozos1          running             0.0.0.0:8082->80/tcp, :::8082->80/tcp
```
Je vois aussi que mon **IHM** est disponible au port **8082**

Je vérifie que le réseau est bien créé
```
[vagrant@docker student-list]$ docker network ls
```
```
NETWORK ID     NAME                             DRIVER    SCOPE
98b7ca2ba69f   bridge                           bridge    local
dedef9dce2f6   host                             host      local
dcfc2cb9e6e6   none                             null      local
bc83b306b3c2   student-list_api_network_pozos   bridge    local
```

### images de test phase IAC à mettre

**Fin de la phase de IAC, succès**

## Partie 3: Docker Registry

L'objectif de cette partie est de déployer un régistre privé afin d'enterriner notre image dans ce registre.

Pour ce faire, nous avons les indications dans l'énoncé du dépôt git du projet. Cliquer sur ce lien [registry](https://docs.docker.com/registry/) pour avoir la procédure à suivre.
Le lien suivant [interface](https://hub.docker.com/r/joxit/docker-registry-ui/) nous mène vers une image qui nous servira d'IHM pour notre **Registry**

Commençons par déployer le **Registry**
- Mon **Registry** doit être déployé dans le même réseau **student-list_api_network_pozos**
- Je demarrer mon registry
```
[vagrant@docker student-list]$ docker run -d -p 5000:5000 --name registry-pozos --network student-list_api_network_pozos registry:2
```
- Je vérifie que mon container **registry-pozos** est bien en place
```
[vagrant@docker student-list]$ docker ps -a
```
```
CONTAINER ID   IMAGE          COMMAND                  CREATED          STATUS          PORTS                                       NAMES
34d61a9bec77   registry:2     "/entrypoint.sh /etc…"   21 seconds ago   Up 18 seconds   0.0.0.0:5000->5000/tcp, :::5000->5000/tcp   registry-pozos
29b92de874b2   api-pozos:V1   "python ./student_ag…"   35 minutes ago   Up 35 minutes   0.0.0.0:4000->5000/tcp, :::4000->5000/tcp   student-list-api-pozos1-1
71b89d470909   php:apache     "docker-php-entrypoi…"   35 minutes ago   Up 35 minutes   0.0.0.0:8082->80/tcp, :::8082->80/tcp       student-list-web-pozos1-1
```
- Je vais renommer mon image pour qu'elle puisse aller taper le registry
```
[vagrant@docker student-list]$ docker images
```
```
REPOSITORY   TAG           IMAGE ID       CREATED         SIZE  
api-pozos    V1            ddd9133d5014   2 hours ago     1.13GB
registry     2             2e200967d166   36 hours ago    24.2MB
php          apache        c3d1b09e989b   8 days ago      458MB 
python       2.7-stretch   e71fc5c0fcb1   23 months ago   928MB 
```
Pour renommer mon image:
```
[vagrant@docker student-list]$ docker image tag api-pozos:V1 localhost:5000/api-pozos:V1
```
```
[vagrant@docker student-list]$ docker images
REPOSITORY                 TAG           IMAGE ID       CREATED         SIZE  
api-pozos                  V1            ddd9133d5014   2 hours ago     1.13GB
localhost:5000/api-pozos   V1            ddd9133d5014   2 hours ago     1.13GB
registry                   2             2e200967d166   36 hours ago    24.2MB
php                        apache        c3d1b09e989b   8 days ago      458MB 
python                     2.7-stretch   e71fc5c0fcb1   23 months ago   928MB
```
Voici donc le nouveau tag qui a le même id que l'image et le même size
```
localhost:5000/api-pozos   V1            ddd9133d5014   2 hours ago     1.13GB
```
C'est donc ce nouveau tag que je vais envoyer dans le **Registry** ci-dessous
```
registry                   2             2e200967d166   36 hours ago    24.2MB
```

Pour envoyer ce tag dans le container **Registry**, je vais le pusher
```
[vagrant@docker student-list]$ docker push localhost:5000/api-pozos:V1
```
Push effectué avec succès
```
The push refers to repository [localhost:5000/api-pozos]
b070f68f94e7: Pushed 
d0ce8915fc21: Pushed
0f1333464ea4: Pushed
811b6c5694d4: Pushed
1855932b077c: Pushed
fa28e7fcadc2: Pushed
4427a3d9a321: Pushed
4a03ae8d3bee: Pushed
a9286fedbd63: Pushed
d50e7be1e737: Pushed
6b114a2dd6de: Pushed
bb9315db9240: Pushed
V1: digest: sha256:bd06cbe98d9c2a1c7130195c578fe72f5926765c48eef4cbbd45cbbe1bf4c062 size: 2854
```

Pour lancer l'image **Registry**
```
[vagrant@docker student-list]$ docker run -d --name registry-pozos_UI --network student-list_api_network_pozos -p 4002:80 -e REGISTRY_TITLE="POZOS REGISTRY" -e REGISTRY_URL=http://registry-pozos:5000 -e DELETE_IMAGES=true joxit/docker-registry-ui:static
```

Je vérifie que mon container est présent
```
[vagrant@docker student-list]$ docker ps -a
CONTAINER ID   IMAGE                             COMMAND                  CREATED             STATUS             PORTS                                       NAMES
4fe42de81233   joxit/docker-registry-ui:static   "/docker-entrypoint.…"   36 seconds ago      Up 34 seconds      0.0.0.0:4002->80/tcp, :::4002->80/tcp       registry-pozos_UI
34d61a9bec77   registry:2                        "/entrypoint.sh /etc…"   38 minutes ago      Up 38 minutes      0.0.0.0:5000->5000/tcp, :::5000->5000/tcp   registry-pozos   
29b92de874b2   api-pozos:V1                      "python ./student_ag…"   About an hour ago   Up About an hour   0.0.0.0:4000->5000/tcp, :::4000->5000/tcp   student-list-api-pozos1-1
71b89d470909   php:apache                        "docker-php-entrypoi…"   About an hour ago   Up About an hour   0.0.0.0:8082->80/tcp, :::8082->80/tcp       student-list-web-pozos1-1
```
Mes containers sont bien en **up**

Je vais aller sur le navigateur voir si j'ai bien mon **Registry private**
```
http://192.168.56.30:4002
```


### images du registry private


