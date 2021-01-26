---
title: Raspberry Pi, Docker et conteneur Lighttpd (2/4)
date: 2020-11-02 18:00:00
tags:
- raspberry pi
- linux
- docker
- lighttpd
---

## Utilisation de conteneurs : *Docker*

Pour la suite du développement de notre Raspberry Pi, nous utiliserons la solution de conteneurisation *Docker* pour pouvoir lancer des applications tout en les rendant portables et isolées les unes des autres.

### Docker

1. Installez les prérequis suivants
   ```
   sudo apt-get install apt-transport-https ca-certificates software-properties-common -y
   ```
2. Téléchargez le script officiel d'installation
   ```
   curl -fsSL https://get.docker.com -o get-docker.sh
   ```
3. Exécutez le script
   ```
   sh get-docker.sh
   ```
4. Ajoutez l'utilisateur `pi` au groupe *docker* pour qu'il puisse le lancer
   ```
   sudo usermod -aG docker pi
   ```
5. Mettez à jour votre système
   ```
   sudo apt-get update && sudo apt-get upgrade
   ```
6. Démarrez Docker
   ```
   sudo systemctl start docker.service
   ```
7. Vérifiez que docker soit correctement installé
   ```
   docker run hello-world
   ```
   La sortie suivante s'affiche alors :
   ```
   Unable to find image \u2018hello-world:latest\u2019 locally
   latest: Pulling from library/hello-world
   4ee5c797bcd7: Pull complete
   Digest: sha256:f9dfddf63636d84ef479d645ab5885156ae030f611a56f3a7ac7f2fdd86d7e4e
   Status: Downloaded newer image for hello-world:latest

   Hello from Docker!
   This message shows that your installation appears to be working correctly.

   To generate this message, Docker took the following steps:
   1. The Docker client contacted the Docker daemon.
   2. The Docker daemon pulled the \u201chello-world\u201d image from the Docker Hub.
   (arm32v7)
   3. The Docker daemon created a new container from that image which runs the
   executable that produces the output you are currently reading.
   4. The Docker daemon streamed that output to the Docker client, which sent it
   to your terminal.

   To try something more ambitious, you can run an Ubuntu container with:
   $ docker run -it ubuntu bash

   Share images, automate workflows, and more with a free Docker ID:
   https://hub.docker.com/

   For more examples and ideas, visit:
   https://docs.docker.com/get-started
   ```

### Docker Compose

Docker Compose est un outil qui permet de décrire (dans un fichier YAML) et gérer (en ligne de commande) plusieurs conteneurs comme un ensemble de services inter-connectés.

> Par exemple, un ensemble composé de 2 conteneurs : un conteneur *Nextcloud*, un conteneur *Jitsy* et un conteneur *Lighttpd*.

1. Installez les prérequis suivants
   ```
   sudo apt-get install python3-distutils python3-dev libffi-dev libssl-dev
   ```
2. On installe le manager de packets Python *PIP*
   ```
   curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
   python3 get-pip.py
   export PATH="/home/pi/.local/bin:$PATH"
   ```
3. On peut ensuite installer Compose
   ```
   pip3 install docker-compose
   ```

   > Le Raspberry PI est basé sur une architecture ARM. Les images Docker sont, pour beaucoup, uniquement disponible pour les architectures plus standard comme x86_64.
   > 
   > Vous pouvez mettre un [filtre sur Docker](https://registry.hub.docker.com/search?q=&type=image&architecture=arm%2Carm64) Hub de façon a lister les images compatibles Raspberry.
   >
   > Dans certains cas, builder le Dockerfile d'une image depuis le Raspberry plutôt que la récupérer depuis Docker Hub permet de la faire fonctionner.

## Serveur web : *Lighttpd* pour un serveur web léger

Dans le top 5 des serveurs les plus utilisés dans le monde, *Lighttpd* est un serveur web (HTTP) qui se veut rapide et léger. Il supporte un grand nombre de fonctionnalités comparables à celles d'*Apache* (comme les *rewrite*, *fast-cgi*, *proxy*... etc) pour des performances aussi bonnes sinon meilleures dans les tests faits par Lighttpd.

> Par rapport à Apache, il ne supporte pas les fichiers *htaccess* ou encore *htpasswd*. Ces 2 problèmes sont contournables si vous avez accès à la configuration de votre serveur. 

1. On télécharge l'image docker
   ```
   docker pull sebp/lighttpd
   ```
2. On crée l'arborescence de fichiers suivante :
   ```
   ~/projets_docker/
      \u251c\u2500\u2500  lighttpd/
      |   \u251c\u2500\u2500  htdocs/
      |   \u251c\u2500\u2500  html/
      |   |    \u2514\u2500\u2500  index.html
      |   \u2514\u2500\u2500 config/
      |       \u2514\u2500\u2500 lighttpd.conf
      \u2514\u2500\u2500 docker-compose.yml
   ```
   - A l'interieur de notre fichier `index.html`, on écrit un simple `Hello`
   - Dans le fichier `lighttpd.conf`, on recopie la configuration minimaliste suivante
     ```
     server.document-root = "/var/www/localhost/html/" 

     server.port = 80

     mimetype.assign = (
         ".html" => "text/html", 
         ".txt" => "text/plain",
         ".jpg" => "image/jpeg",
         ".png" => "image/png" 
     )

     index-file.names = ( "index.html" )
     ```
     > On pourra allonger la liste en s'inspirant de ce [fichier](https://redmine.lighttpd.net/projects/1/wiki/mimetype_assigndetails) en cas de problèmes d'association de fichiers
   - Et enfin dans le fichier `docker-compose.yml`, la configuration suivante
     ```
     lighttpd:
        restart: always 
        image: sebp/lighttpd
        volumes:
           - /home/pi/projets_docker/lighttpd/htdocs:/var/www/localhost/htdocs
           - /home/pi/projets_docker/lighttpd/config:/etc/lighttpd
           - /home/pi/projets_docker/lighttpd/html:/var/www/localhost/html
        ports:
           - "80:80"
        tty: true
     ```
     > Attention : ce fichier est très sensible à l'indentation !

3. On peut ensuite lancer notre conteneur lighttpd en exécutant la commande suivante dans le dossier `projets_docker`
   ```
   docker-compose up lighttpd
   ```
4. Pour tester si tout fonctionne, il suffit d'ouvrir son navigateur web sur son PC et de se rendre à l'adresse http://raspberrypi.local/. 
   
   Si l'on voit s'afficher *Hello*, le tour est joué.

### Point de sortie : Passer Lighttpd en HTTPS avec Let's Encrypt
> Comme abordé dans le précédent article, idéalement notre serveur Lighttpd sera placé derrière un reverse proxy. C'est alors lui qui sera configuré pour offrir à nos web-apps la sécurité HTTPS.

> Si vous souhaitez installer sur votre Pi d'autres services qui utiliseront le port 80, ou si vous souhaitez aller au bout de ce tutoriel, **vous pouvez passer à l'article suivant** ; SSL sera abordé à la fin.

> Si vous souhaitez cependant en rester là dans la configuration, ou que vous ne voulez simplement qu'un serveur web, mais sécurisé, vous pouvez alors directement passer Lighttpd en HTTPS en suivant la démarche ci-dessous.

1. Installez le paquet pré-requis `snapd`
   ```
   sudo apt install snapd
   ```
2. Redémarrez le système
3. Assurez-vous de disposer de la dernière version de *Snap*
   ```
   sudo snap install core; sudo snap refresh core
   ```
4. Installez Certbot
   ```
   sudo snap install --classic certbot
   ```
5. Préparez l'exécution de la command *Certbot*
   ```
   sudo ln -s /snap/bin/certbot /usr/bin/certbot
   ```
6. Stoppez votre serveur web
   ```
   docker-compose stop lighttpd
   ```
7. Démarrez la certification et suivez les instructions
   ```
   sudo certbot certonly --standalone
   ```
   Vos certificats devraient alors se trouver à l'adresse `/etc/letsencrypt/live/<votre_domaine>/`
8. Combinez vos certificats en un seul fichier :
   > Commande a éxécuter avec les privilèges root
   ```
   cat /etc/letsencrypt/live/<votre_domaine>/privkey.pem /etc/letsencrypt/live/<votre_domaine>/cert.pem > /etc/letsencrypt/live/<votre_domaine>/merged.pem
   ```
9. Copiez les fichiers `merged.pem` et `chain.pem` dans le volume docker `/home/pi/projets_docker/lighttpd/config`
10. Modifiez la configuration du serveur en rajoutant dans le fichier `lighttpd.conf` les lignes suivantes adaptées à votre configuration
    ```
    server.modules += ("mod_openssl")
    $SERVER["socket"] == ":443" {
        ssl.engine = "enable"
        ssl.pemfile = "/etc/lighttpd/merged.pem"
        ssl.ca-file = "/etc/lighttpd/chain.pem"
        server.name = "<votre_domaine>"
    }
    ```
11. Rajoutez dans la section *ports* du fichier `docker-compose.yml` la liaison `- "443:443"`
12. Forcez la redirection HTTP -> HTTPS en ajoutant le block suivant dans le fichier `lighttpd.conf`
    ```
    server.modules += ("mod_redirect")
    $HTTP["scheme"] == "http" {
            url.redirect = ("" => "https://${url.authority}${url.path}${qsa}")
    }
    ```

### Persnonaliser l'affichage des pages d'erreurs

Pour afficher des pages d'erreurs personnalisées, il vous faut :

1. Créer un dossier `errors` dans le volume `/home/pi/projets_docker/lighttpd/htdocs`
2. Rédiger vos pages d'erreurs au format HTML à l'interieur de ce dossier, et les nommer avec le préfixe `status-` (exemple : `status-404.html`)
3. Editer le fichier `lighttpd.conf` et y ajouter la ligne suivante
   ```
   server.errorfile-prefix = "/var/www/localhost/htdocs/errors/status-"
   ```
