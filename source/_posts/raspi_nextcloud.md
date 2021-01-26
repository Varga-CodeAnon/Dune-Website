---
title: Raspberry Pi, Next Cloud & Traefik (4/4)
date: 2020-11-03 18:00:00
tags:
- raspberry pi
- linux
- docker
- nextcloud
- traefik
---

Pour ce dernier article, l'idée est de posséder un cloud autohébergé pour le stockage distant de nos données. 

Dans ce tutoriel, l'espace proposé sera égal à l'espace libre disponible sur votre clé USB (soit environ une quinzaine de gigas). 

Cependant, pour une plus grande capacité de stockage, vous pourrez connecter un disque dur ou une clé USB au Pi et fusionner les espaces de stockages ; mais ça ne sera pas abordé ici.

## Création d'une instance Nextcloud

1. Commencez par vous placer dans votre répertoire de travail, puis stoppez vos conteneurs
  ```
  cd projets_docker/ && docker-compose stop
  ```
2. Réalisez une sauvegarde de votre fichier de configuration docker compose
   ```
   cp docker-compose.yml docker-compose.yml.bak
   ```
3. Passez le fichier `docker-compose.yml` en version 2, et introduisez-y la partie concernant nextcloud en l'adaptant aux caractéristiques du Pi. 
   
   A la fin, votre fichier devrait ressembler à ceci :

```yml
version: '2'

volumes:
   nextcloud:
   db:

services:
   db:
      image: jsurf/rpi-mariadb:latest
      container_name: mariadb
      command: --transaction-isolation=READ-COMMITTED --binlog-format=ROW
      restart: always
      hostname: db
      volumes:
         - db:/var/lib/mysql
      environment:
         - PUID=1000
         - PGID=1000
         - MYSQL_ROOT_PASSWORD= # A définir
         - TZ=Europe/Paris
         - MYSQL_DATABASE=nextcloud
         - MYSQL_USER=nextcloud
         - MYSQL_PASSWORD= # A définir
      ports:
         - 3306:3306

   app:
      image: nextcloud:stable
      ports:
         - 8080:80
      links:
         - db
      volumes:
         - nextcloud:/var/www/html
      restart: always
      depends_on:
         - db

   lighttpd:
      image: sebp/lighttpd
      ports:
         - "80:80"
         - "443:443"
      volumes:
         - /home/pi/projets_docker/lighttpd/htdocs:/var/www/localhost/htdocs
         - /home/pi/projets_docker/lighttpd/config:/etc/lighttpd
         - /home/pi/projets_docker/lighttpd/html:/var/www/localhost/html
      tty: true
      restart: always
```
   > Pensez à attribuer des mots de passes dans les champs `# A définir`

4. Redémarrez vos conteneurs
   ```
   docker-compose up -d
   ```
   > Note : en cas de time-out, n'hésitez pas à augmenter les temps d'attente : `export DOCKER_CLIENT_TIMEOUT=180` et `export COMPOSE_HTTP_TIMEOUT=180` dans le terminal

5. Une fois que les conteneurs sont démarrés, rendez-vous à l'adresse http://192.168.1.20:8080/ pour continuer la suite de l'installation via une interface web
6. Remplissez ses champs comme il suit :
   
   | Champ                  |               Valeur |
   | :--------------------- | -------------------: |
   | Admin user             |        `<a choisir>` |
   | Admin password         |        `<a choisir>` |
   | Répertoire des données | `/var/www/html/data` |
   | Configurer la BDD      |      `MySQL/MariaDB` |
   |                        |          `nextcloud` |
   |                        |   `<MYSQL_PASSWORD>` |
   |                        |          `nextcloud` |
   |                        |                 `db` |

7. Cochez ou non *Installer les applications recommandées*, puis laissez le temps à Nextcloud le temps de tout configurer (prenez un bon café bien mérité)

## Problématique

Une fois l'installation terminée, vous pouvez bénéficier de votre cloud personnel, **mais uniquement à l'adresse** http://192.168.1.20:8080/ (contrainte de sécuité de nextcloud)

Seulement, vous voudrez peut-être accéder a votre cloud via un nom de domaine, et de manière sécurisée via HTTPS, et il en est de même de votre blog (pourquoi ne pas d'ailleurs créer un onglet *Cloud* sur votre blog pour y accéder ?)

## Déploiement du reverse proxy Traefik

Pour répondre à ces problèmes et rediriger les flux TLS vers nos web-apps, on configurera un reverse proxy open-source et français *Traefik*, là aussi via docker.

### Prérequis : ouverture sur le monde

Pour le moment, nos applications tournent sur un réseau local à l'adresse `192.168.X.X`. Pour y accéder depuis l'extérieur, il faut configurer notre box/routeur pour rediriger le flux entrant sur notre raspberry pi, et associer l'adresse IPv4 publique de notre routeur à un nom de domaine (nécessaire pour Nextcloud).

1. Sélectionnez un hébergeur de nom de domaine, et louez-y un nom. Certains opérateurs comme *Free* proposent des noms de domaines personnalisés gratuitement.
2. Une fois le nom de domaine loué, dans le tableau de bord de votre hébergeur section *Zone DNS*, créez un enregistrement de type A pour lier votre nom de domaine à votre adresse IP publique.

   Par exemple :
   ```
   dune.ovh IN A 82.243.55.66
   ```

3. Configurez ensuite 3 entrées dans la zone DNS en les adaptant à votre nom de domaine :
   ```
   traefik IN CNAME dune.ovh.
   www IN CNAME dune.ovh.
   cloud IN CNAME dune.ovh.
   ```

4. Rendez-vous dans l'interface web de votre box internet (les modalités changent selon les box), puis effectuez une redirection de port pour que les ports 22, 80 et 443 de votre box pointent vers ces mêmes ports sur votre Pi 

   Vous pouvez aussi lier des ports différents pour plus de sécurité, par exemple le port 7000 de votre routeur au port 22 de votre Pi.

## Configurer et lancer Traefik

1. Stoppez les conteneurs actifs
   ```
   docker-compose
   ```
2. Pour la suite, nous aurons besoin de générer un mot de passe avec le module `htpasswd` d'Apache. Pour se faire, taper la commande suivante
   ```
   docker run --rm --name apache httpd:alpine htpasswd -nb <username> <password>
   ```

   Un hash comme celui-ci est alors donné, notez-le :
   ```
   user:$apr1$1eZu7RXg$Ql9Z5AvZNc0Oe4In900mi0
   ```
3. Créez dans votre dossier `projets-docker` un fichier `traefik.toml`, et remplissez-le comme il suit :
```
nano traefik.toml
```
```toml
defaultEntryPoints = ["http", "https"]

[entryPoints]
# On configure l'acces au panel
  [entryPoints.dashboard]  # définition de la méthode de connection
    address = ":8080"  # port du dashboard
    [entryPoints.dashboard.auth]  # mise en place d'une authentification
      [entryPoints.dashboard.auth.basic]
        users = ["michael:$apr1$jABfdThR$vaO3dDwB4bvaFzYmHrS0Vk"]

# Points d'entrées concernant les application dockerisées (hors l'api)
  [entryPoints.http]
    address = ":80"
      [entryPoints.http.redirect]
        entryPoint = "https"
  [entryPoints.https]
    address = ":443"
      [entryPoints.https.tls]

# On active l'API
[api]
entrypoint="dashboard"

# On paramètre le protocole de communication avec Let's Encrypt
[acme]
email = "michael@gmail.com"
storage = "acme.json"
entryPoint = "https"
onHostRule = true
  [acme.httpChallenge]
  entryPoint = "http"

[docker]
domain = "dune.ovh"
watch = true
network = "web"
```
4. Créez un réseau commun à nos conteneurs :
   ```
   docker network create web
   ```
6. Créez un fichier vide `acme.json` qui permettra de stocker les différents certificats, et modifiez ses droits d'accès
   ```
   touch acme.json
   chmod 600 acme.json
   ```
7. Enfin, déployez le conteneur Traefik via docker, en **adaptant** la commande à votre nom de domaine :
   ```
   docker run -d \
      -v /var/run/docker.sock:/var/run/docker.sock \
      -v $PWD/traefik.toml:/traefik.toml \
      -v $PWD/acme.json:/acme.json \
      -p 80:80 \
      -p 443:443 \
      -l traefik.frontend.rule=Host:traefik.dune.ovh \
      -l traefik.port=8080 \
      --network web \
      --name traefik \
      traefik:1.7-alpine
   ```
   
   Vous pourrez ensuite accéder au panneau de contrôle Traefik à l'adresse https://traefik.dune.ovh (en l'adaptant à votre nom de domaine et sous-domaine bien sûr)

8. Passez le fichier `docker-compose.yml` en version 3, et introduisez-y la partie concernant Traefik en l'adaptant aux caractéristiques du Pi. 

    On modifiera aussi les services déjà configurés précédemment pour y rajouter la section `labels` nécessaire au reverse-proxy.
   
   A la fin, votre fichier devrait ressembler à ceci :
```yml
version: '3'

networks:
  web:
    external: true
  internal:
    external: false

volumes:
  nextcloud:
  db:
Administrat0r
services:
  db:
    image: jsurf/rpi-mariadb:latest
    container_name: mariadb
    command: --transaction-isolation=READ-COMMITTED --binlog-format=ROW
    restart: always
    hostname: db
    volumes:
        - db:/var/lib/mysql
    environment:
        - PUID=1000
        - PGID=1000
        - MYSQL_ROOT_PASSWORD= # A définir
        - TZ=Europe/Paris
        - MYSQL_DATABASE=nextcloud
        - MYSQL_USER=nextcloud
        - MYSQL_PASSWORD= # A définir
    # ports:
    #     - 3306:3306
    networks:
        - internal
    labels:
        - traefik.enable=false

  app:
    image: nextcloud:stable
    # ports:
    #     - 8080:80
    links:
        - db
    volumes:
        - nextcloud:/var/www/html
    restart: always
    depends_on:
        - db
    labels:
        - traefik.backend=app
        - traefik.frontend.rule=Host:cloud.dune.ovh
        - traefik.docker.network=web
        - traefik.port=80
    networks:
        - internal
        - web

  lighttpd:
    image: sebp/lighttpd
    # ports:
    #     - "80:80"
    #     - "443:443"
    volumes:
        - /home/pi/projets_docker/lighttpd/htdocs:/var/www/localhost/htdocs
        - /home/pi/projets_docker/lighttpd/config:/etc/lighttpd
        - /home/pi/projets_docker/lighttpd/html:/var/www/localhost/html
    labels:
        - traefik.backend=lighttpd
        - traefik.frontend.rule=Host:www.dune.ovh
        - traefik.docker.network=web
        - traefik.port=80
    networks:
        - internal
        - web
    tty: true
    restart: always
```
9. Redémarrez vos conteneurs :
   ```
   docker-compose up -d
   ```

Vous pouvez alors accéder à vos différents services en HTTPS depuis les différentes adresses configurées (ex: lighttpd via www.dune.ovh, nextcloud via cloud.dune.ovh)

## Sauvegarde

Maintenant que nos services sont déployés, il est important de procéder à une sauvegarde, comme dans notre premier article.

1. Assurez-vous tout d'abord que votre Pi soit éteint et débranché
2. Retirer la carte microSD, et connectez-là à votre PC
3. Créer une image système
   1. Soit avec l'interface graphique du paquet `gnome-disk-utility` exécutable via la commande `gnome-disks`
   2. Soit en ligne de commande (**utilisateur confirmé seulement**)
	  ```
	  dd bs=4M if=/dev/<sdX> of=<sauvegarde_pi.img> status=progress conv=fsync
      ```

Une fois l'image générée, vous pouvez rebrancher le Pi, allumer le système et l'utiliser comme bon vous semble.
