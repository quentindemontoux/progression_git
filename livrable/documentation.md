Note :

Le guide est fait en suivant notre configuration. Rien ne vous empêche de prendre des solutions qui vous plaisent plus que les nôtres. Cependant, ce sera à vous de vous adapter.

# Prérequis :

- Un hyperviseur (https://www.virtualbox.org/)
- 5 VM Centos 7 (https://www.centos.org/download/) avec un accès internet pour télécharger des packages

# Architecture :

|VM|IP privée|
|--------|--------|
|gitea|192.168.1.74|
|drone|192.168.1.99|
|bdd|192.168.1.98|
|prod|192.168.1.38|
|runner|192.168.1.35|

# Redirection de ports pour une utilisation à distance

1. Accéder a l'interface de votre box souvent en entrant 192.168.1.1 dans votre navigateur web.
2. Avoir les droits d'administrateur sur l'interface (voir selon l'operateur et la box).
3. aller dans la categorie NAT IPv4.
4. Puis configurer les differentes redirection comme indiquer si dessous:

- VM Gitea ports 22 et 3000,
- VM Drone ports 22 et 80,
- VM BDD ports 22 et 3306,
- VM Prod ports 22,
- VM Runner ports 22 et 3000.

5. Vous devriez avoir quelques choses comme ceci:

![](https://i.imgur.com/lY4wN9K.png)

# Installation :

## 1) Gitea

Suivez l'installation de gitea avec le binaire : https://docs.gitea.io/en-us/install-from-binary/  
Vous pouvez permettre à gitea de se lancer au démarrage : https://docs.gitea.io/en-us/linux-service/  

N'oubliez pas d'ouvrir le port de gitea (par défaut 3000) pour y accéder par le web.  
Sous centos 7 : sudo firewall-cmd --add-port=3000/tcp --permanent  
Il faut ensuite recharger le firewall avec sudo firewall-cmd --reload  

Si tout s'est bien passé, vous devriez pouvoir accéder à gitea avec votre.ip:3000  

Il vous faudra configurer gitea avec votre première connexion. Nous avons gardé les paramètres par défaut.  
N'oubliez pas de remplacer les adresses template par les vôtres (ex : ip BDD).  

## 2) Base de données

Si vous avez suivi la documentation de gitea, vous devriez avoir déjà installé la base de données.  
Nous avons utilisé mariaDB (sudo yum install -y mariadb-server).  

Le passage ci-dessous vous concerne si vous avez bloqué sur la configuration de gitea :  

Il est d'abord nécessaire d'indiquer l'IP de votre VM à mariaDB :
echo "bind-address = 192.168.1.98" >> /etc/my.cnf

Pour configurer la base de données, nous avons rentré cette suite de commande :  
mysql -u root -e "SET old_passwords=0;  
CREATE USER 'gitea'@'192.168.1.74' IDENTIFIED BY 'gitea';  
CREATE DATABASE giteadb CHARACTER SET 'utf8mb4' COLLATE 'utf8mb4_unicode_ci';  
GRANT ALL PRIVILEGES ON giteadb.* TO 'gitea'@'192.168.1.74';  
FLUSH PRIVILEGES;"  

Vous êtes censé être notifié par un message validant la création de votre base de données.  
N'oubliez pas d'ouvrir le port 3306.  
Vous pouvez maintenant retourner dans gitea et entrer les informations qui vous manquaient.  
Si tout s'est bien passé, gitea est maintenant utilisable, et vous avez créé un premier compte.  
Gardez ce compte en mémoire, il a des droits admin. Dans le pire des cas, vous pourrez en créer un autre.  

## 3) Drone

Nous avons utilisé docker pour faire fonctionner drone. Pour ce faire :  

$ sudo yum install -y yum-utils

$ sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo

$ sudo yum install docker-ce docker-ce-cli containerd.io

Il est maintenant possible de passer docker en service, pour le démarrer au lancement :  

$ sudo systemctl enable docker.service

$ sudo systemctl enable containerd.service

Vous pouvez maintenant ajouter l'utilisateur de votre VM au groupe docker.  
Cela vous évitera de devoir faire des sudo à chaque commande docker. Pour ce faire :  

$ sudo groupadd docker

$ sudo usermod -aG docker $USER

Déconnectez vous puis reconnectez vous. Vous devriez pouvoir faire des commandes docker sans problème.  

Maintenant, allez dans les paramètres de votre compte sur gitea, OAuth2 Applications.  
Créez une application nommée drone qui redirige vers http://(votre ip)/login  
Gardez de côté votre ID client et secret client, qui vous serons nécessaires pour la suite.  
Sauvegardez et retournez dans drone.  

Vous aurez besoin d'un secret partagé, pour ça :

$ openssl rand -hex 16

Gardez ce secret de côté.  
Il vous faut maintenant télécharger l'image de drone pour docker :  

$ docker pull drone/drone:1

Vous pouvez maitenant créer votre conteneur drone :  

$ docker run \
  --volume=/var/lib/drone:/data \
  --env=DRONE_GITEA_SERVER={{DRONE_GITEA_SERVER}} \
  --env=DRONE_GITEA_CLIENT_ID={{DRONE_GITEA_CLIENT_ID}} \
  --env=DRONE_GITEA_CLIENT_SECRET={{DRONE_GITEA_CLIENT_SECRET}} \
  --env=DRONE_RPC_SECRET={{DRONE_RPC_SECRET}} \
  --env=DRONE_SERVER_HOST={{DRONE_SERVER_HOST}} \
  --env=DRONE_SERVER_PROTO={{DRONE_SERVER_PROTO}} \
  --publish=80:80 \
  --publish=443:443 \
  --restart=always \
  --detach=true \
  --name=drone \
  drone/drone:1

Remplacez toutes les sections {{}} par vos informations.  
GITEA_SERVER est l'adresse de votre serveur gitea (ex : http://votre.ip:3000).  
DRONE_RPC_SECRET est votre secret partagé.  
DRONE_SERVER_HOST est l'adresse de votre serveur drone (ex : votre.ip:80)  
DRONE_SERVER_PROTO est votre protocole, http ou https. Choisissez celui qui vous correspond.  
Nous avons réalisé le projet en http, il est fortement recommandé de le faire en https.  

Si tout s'est bien passé, vous ne devriez pas avoir d'erreur avec la commande docker logs drone.  

## 4) Runner

Commencez par passer SElinux en permissif en éditant le fichier /etc/selinux/config pour changer la ligne SELINUX=enforced en SELINUX=permissive.  
Vous ne pourrez pas faire fonctionner le runner sans ça.  
Il vous faut ouvrir le port 3000.  

Il vous faut maintenant répéter exactement la même opération pour installer docker, sauf que vous allez installer une image de runner drone :  

$ docker pull drone/drone-runner-docker:1

Vous pouvez maintenant créer le conteneur runner drone :  

$ docker run -d \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -e DRONE_RPC_PROTO=http \
  -e DRONE_RPC_HOST=votre.ip:80 \
  -e DRONE_RPC_SECRET=secret partagé \
  -e DRONE_RUNNER_CAPACITY=2 \
  -e DRONE_RUNNER_NAME=${HOSTNAME} \
  -p 3000:3000 \
  DRONE_UI_USERNAME=root \
  DRONE_UI_PASSWORD=root \
  --restart always \
  --name runner \
  drone/drone-runner-docker:1

N'oubliez pas de remplacer les lignes de base par vos informations.  
Vous pouvez ensuite faire un docker logs runner. Vous devriez avoir quelque chose comme ça :  

$ docker logs runner

INFO[0000] starting the server
INFO[0000] successfully pinged the remote server

## 5) Tests unitaire

Ils existe 2 manières de faire des tests unitaires. 

Premièrement (la plus répandue): 
On crée 2 fichiers

- code.py (fichier comportent le code du programme)
- test_code.py (fichier comportent les tests unitaires)

Grace aux ````python from code.py  import ...````

Cela nous permet de lier le fichier de code à celui du test.

Généralement on fait passer une variable à faire rentrer à l'utilisateur depuis la console et c'est comme ça que nous testons la sensibilité à la casse de notre code (grace aux test_code.py).

Cette appel de variable se fait depuis le code de code.py mais pour la 2ème solution c'est different.

On crée 2 fichiers

- code.py (fichier comportent le code du programme)
- test_code.py (fichier comportent les tests unitaires)

Toujours grace aux  ````python from code.py  import ...````

Cela nous permet de lier le fichier de code à celui du test.

Différence il n'y aura pas d'appel de variable on **simulera** directement le résultat dans le fichier ````test_code.py````.

Bien entendu la création de classe , fonction , paramètre .... sera toujours dans le ````code.py````. Il n'y aura pas d'entrée dans la console, on simulera un scenario avec une réponse mise directement dans le code et si le retour "message d'erreur ou non" est correct alors le code est fonctionnel cela me permet d'aller plus **vite**  de tester **plus** de scenario possible sans passer par la console :D

Je vais dorénavant vous proposer de créer un test unitaire étape par étape.

### Création des deux fichiers.

```` code.py```` et ````test_code.py````

 
Nous allons tester le code d'une classe.

dans ```` code.py````...


```` python
    class Plane: # création de la classe avion
        def __init__(self):#on initialise la vitesse de l'avion à 0
            self._speed = 0
            self._start_plane = False

        def start_plane(self): # on implémente la classe plane
            if self._start_plane:
                raise Exception("l'avion est deja pret")
            self._start_plane = True

        def turn_off_plane(self): # on arreter l'avion
            self._start_plane = False

        def add_speed(self): # on ajoute 5 de vitesse à l'avion
            self._speed += 5

        def remove_speed(self):#pour que la vitesse ne soit pas négatif
            if self._speed <= 0:
                self._speed = 0
            else:
                self._speed -= 5

        def current_speed(self):#connaitre la vitesse actuel
            return self._speed

        def stop(self):#vitesse à 0
            self._speed = 0

        def plane_status(self):#connaitre le statut de l'avion ON ou OFF
            return self._start_plane
````

Les commentaires sont explicites pour la compréhension du code.

Nous allons jouer avec la vitesse de l'avion sur plusieurs scénarios.

Dans le fichier ````test_code.py ```` on implémente tout d'abord les librairies nécessaires.

````import unittest```` Le module unittest fournit un riche ensemble d'outils pour construire et lancer des tests.

````from code import Plane ```` Depuis le fichier code.py on import la classe Plane.

Dans le fichier de test unitaire utilisé pour l'évaluation j'ai fais 3 niveaux de test EASY , MEDIUM , HARD (qui décrit le niveau

de précision du programme qui influera sur la sensibilité de casse du code).

Pour cet exemple nous allons monter un programme de difficulté MEDIUM.


Dans le fichier ````test_code.py ````......

````python
#test difficulté MEDIUM
#on test si l'avion possede une vitesse aux dessus de 0 et que l'avion n'est pas une vitesse en dessous de 0
class MediumTestCase(unittest.TestCase):

    def setUp(self):
        self.plane =  Plane() #Création d'un objet nommé plane pour la classe Plane
        self.plane.start_plane() #On utilise l'objet plane crée pour start l'avion


    def test_medium_input(self):
        with self.assertRaises(Exception): #On soulève une exception si l'utilisateur veut utiliser l'avion alors qu'elle à déjà été initialiser. 
            self.plane.start_plane() #On utilise l'objet plane crée pour start l'avion


    def test_medium_input_two(self):
        self.plane.remove_speed
        self.plane.remove_speed
        self.plane.remove_speed # -5

        self.assertEqual(self.plane.current_speed(), 0) # 0 est le résultat attendu qui doit etre égal à current_speed == vitesse de l'avion


    def tearDown(self):
        self.plane.stop() # On stop l'avion
        self.plane.turn_off_plane() # turn off
        self.plane = None #On rentre la valeur avion en None
````

- **Précison** pour cette partie de code ...
```` python
    def test_medium_input(self):
        with self.assertRaises(Exception): #On soulève une exception si l'utilisateur veut utiliser l'avion alors qu'elle à déjà été initialiser. 
            self.plane.start_plane() #On utilise l'objet plane crée pour start l'avion
````

qui est directement lié aux code dans code.py.
````python
    def start_plane(self):
        if self._start_plane:
            raise Exception("l'avion est deja pret")
        self._start_plane = True
````
Dans le cas ou ces deux lignes ceux suivent.

````python self.plane.start_plane()````

````python self.plane.start_plane()````

Le code expliqué juste avant nous permet de répondre à cette situation.

Si on met en commentaire cette ligne ... 
````python
# with self.assertRaises(Exception):
````

````python
  File "c:\Perso\projet reseaux\first_project.py", line 99, in start_plane
    raise Exception("l'avion est deja pret")
Exception: l'avion est deja pret

----------------------------------------------------------------------
Ran 9 tests in 0.001s

FAILED (errors=1)
````

Voici le résultat de la console.

- **Précison** de cette partie de code.
````python
    def test_medium_input_two(self):
        self.plane.remove_speed
        self.plane.remove_speed
        self.plane.remove_speed # -5

        self.assertEqual(self.plane.current_speed(), 0)
````

On diminue la vitesse de l'avion par -5 3fois, l'avion est donc sensé avoir une vitesse de -15 mais dans le code.py nous avons fait

en sorte qu'il ne soit pas négatif.

Dans code.py

````python
    def remove_speed(self):#pour que la vitesse ne soit pas négatif
        if self._speed <= 0:
            self._speed = 0
        else:
            self._speed -= 5
````

Le code permet de ne pas avoir de nombre négatif.

Dans test_code.py 

````python
self.assertEqual(self.plane.current_speed(), 0)
````

self.assertEqual nous permet de chercher l'égalité entre deux valeur... a= le résultat du code, b= le résultat attendu.

self.assertEqual(a, b)

Derniere ligne de cette partie de code avant la virgule on met le résulat du programme = self.plane.current_speed() et apres la virgule on met la valeur attendu du programme qui est 0.
````python
self.assertEqual(self.plane.current_speed(), 0)
````

Si le programme retourne un message d'erreur c'est que  ````def remove_speed(self):```` comporte une erreur et que self.plane.current_speed() n'est pas égal aux résultat attendu soit 0 dans ceux cas la.

- **Précison** de cette partie de code.
````python
    def tearDown(self):
        self.plane.stop() # On stop l'avion
        self.plane.turn_off_plane() # turn off
        self.plane = None #On rentre la valeur avion en None
````

Cette partie de code n'est pas un test c'est comme si on éteignait notre console après avoir joué à un jeu.

## 6) Pipeline

Vous pouvez maintenant vérifier que tout fonctionne. Pour ça, connectez vous sur gitea et créez un repository. 
Ensuitez, tentez d'accéder à http://votre.ip/login. 
Validez le pop-up qui apparaît. Vous devriez être redirigé sur la dashboard de drone. 
Essayez de synchroniser. Après quelques secondes, votre repo apparaît. Faites enable. Votre repo est prêt. 

Il vous faut créer un .drone.yml à la racine de votre repo. Mettez y ces infos : 

```yaml
kind: pipeline
type: docker
name: default

steps:
- name: greeting
  image: alpine
  commands:
  - echo hello
  - echo world
```


Essayez de push. Si tout se passe bien, vous devriez voir votre push sur drone.  
Vous pouvez cliquer dessus pour voir ce qui s'y passe. Normalement, votre commit sera validé.  

Vous pouvez customiser votre .yml pour y mettre vos tests et votre déploiement automatisé.  
Avant ça, il faut configurer la VM de production.  

## 7) Production

Installez sur cette VM tous les packages nécessaires à votre projet.  
Pour nous, c'est du Python, alors nous avons installé python3.  

Il vous faudra obligatoirement git : sudo yum install -y git  

Si vous voulez suivre notre installation : sudo yum install -y python3  

Il vous faudra permettre l'accès SSH à votre VM, pour ça :  

$ sudo yum –y install openssh-server

$ sudo systemctl enable sshd

$ sudo systemctl start sshd

Il faut également ouvrir le port 22.  

## 8) Configurer .drone.yml

Vous pouvez maintenant mettre tout ce dont vous avez besoin dans votre .yml  
Pour bien comprendre son fonctionnement, vous êtes invité à aller ici : https://docs.drone.io/pipeline/overview/  

Si vous voulez suivre notre installation, alors voici notre .yml (modifiez selon votre config) :  

```yaml
kind: pipeline
type: docker
name: auto_deploy

steps:
- name: tests
  image: python
  commands:
  - python3 test_first_project.py

- name: deploy
  image: bash
  commands:
  - yum install -y sshpass
  - sshpass -p motdepasse ssh -oStrictHostKeyChecking=accept-new user@x.x.x.x -p 5500
  - if /repo/ == 1; git pull http://x.x.x.x:3000/user/repo; else git clone http://x.x.x.x:3000/user/repo; fi
  - exit
```


**ATTENTION** : notre configuration n'est pas du tout sécurisé, vous pouvez voir un mot de passe en clair.  
Il est fortement déconseillé de nous suivre au pied de la lettre ! Une alternative serait d'utiliser  
une paire de clés SSH (ssh-keygen, ssh-copy-id, ...) pour éviter d'avoir besoin du mot de passe.  
Quiquonque à accès à votre .yml peut voir un de vos mots de passe ! A éviter absolument !  

Vous devez remplacer les x.x.x.x par votre ip publique.  

Vous pouvez tenter de push tout ce que vous avez à mettre dans votre repo.  
Si tout se passe bien,vous aurez un déploiement automatisé (sauf si vous avez volontairement fait une erreur).  

Lancez votre VM de production et vérifiez que votre repo a bien été clone/pull.  
