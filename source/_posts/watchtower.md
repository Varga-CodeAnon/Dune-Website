---
title: "Maintenir ses images Docker à jour avec Watchtower"
date: 2021-01-20 19:00:00
tags:
- docker
- docker-compose
- watchtower
---

## Avant-propos

L'un des premiers vecteurs d'attaque des systèmes informatique est le retard ou l'absence de mises à jour de ces systèmes. En effet, lorsqu'une vulnérabilité sur un système est identifiée, un papier sort, ainsi qu'un patch de sécurité quelques temps après. Si ce patch n'est pas appliqué via une mise à jour, la vulnérabilité est alors présente sur votre système, mais elle est aussi documentée sur le net et pourrait inspirer des personnes malveillantes.

Par mesure de sécurité, nous allons donc déployer une solution qui va maintenir nos conteneurs à jour : il s'agit de *[Watchtower](https://github.com/containrrr/watchtower)*

## Configuration

Watchtower est un conteneur Docker qui va comparer les images de nos conteneurs avec la base [Docker Hub](https://hub.docker.com/), télécharger les images dont la version est supérieure, éteindre nos conteneurs et les redémarrer à jour avec les mêmes options de déploiement.

Comme watchtower est un conteneur, sont déploiement et sa configuration sont très simples. On ajoutera Watchtower à notre docker-compose pour l'ajouter à notre groupe de conteneurs définit dans les articles précédents :

```yml
  watchtower:
    command: --label-enable --cleanup --schedule '0 0 1 ? * MON'  # Ici sont inscrits les paramètres de lancement du conteneur
    image: containrrr/watchtower:latest
    labels:
      - "com.centurylinklabs.watchtower.enable=true"  # Seuls les conteneurs avec ce label seront tenus à jour
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock  # on connecte watchtower à notre instance docker
    environment:
      - TZ=Europe/Paris  # défini la timezone pour la planification des mises à jours
    restart: always
```

Les options de la ligne `command:` sont :
- `--label-enable` : seuls les conteneurs ayant le label d'actif seront tenus à jour
- `--cleanup` : les anciennes images seront supprimées
- `--schedule` : prend une expression crontab pour planifier les mise à jour, ici tous les lundi à 01h00. Vous pouvez générer une expression crontab grâce à l'aide de cette [documentation](https://pkg.go.dev/github.com/robfig/cron@v1.2.0#hdr-CRON_Expression_Format).

### Phase de tests

Pour vérifier que Watchtower fonctionne bien, nous allons remplacer la ligne `command:` comme ceci :
```yml
command: --label-enable --cleanup --interval 300
```

Cela va nous permettre de vérifier les mises à jours toutes les 5 minutes (300 secondes).
Passé 5 minutes, nous pourrons alors vérifier les logs pour voir si tout a bien fonctionné :
```
docker-compose logs watchtower
```
```
Attaching to projets_docker_watchtower_1
watchtower_1  | time="2021-01-20T23:28:58+01:00" level=info msg="Starting Watchtower and scheduling first run: 2021-01-20 23:33:58 +0100 CET m=+300.965656577"

```

### `docker-compose.yml`

En intégrant watchtower à notre `docker-compose.yml` et en rajoutant le label `com.centurylinklabs.watchtower.enable=true` que l'on souhaite mettre à jour, on obtient le fichier de configuration suivant :

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

services:
  watchtower:
    command: --label-enable --cleanup --schedule '0 0 1 ? * MON'
    image: containrrr/watchtower:latest
    labels:
      - "com.centurylinklabs.watchtower.enable=true"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - TZ=Europe/Paris
    restart: always

  traefik:
    image: traefik:1.7-alpine
    container_name: traefik
    ports:
        - 80:80
        - 443:443
    volumes:
        - /var/run/docker.sock:/var/run/docker.sock
        - /home/pi/projets_docker/traefik/traefik.toml:/traefik.toml
        - /home/pi/projets_docker/traefik/acme.json:/acme.json
    labels:
        - traefik.frontend.rule=Host:traefik.mondomaine.fr
        - traefik.port=8080
        - "com.centurylinklabs.watchtower.enable=true"
    networks:
        - internal
        - web
    restart: always

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
        - MYSQL_ROOT_PASSWORD=  # A définir
        - TZ=Europe/Paris
        - MYSQL_DATABASE=nextcloud
        - MYSQL_USER=  # A définir
        - MYSQL_PASSWORD=  # A définir
    networks:
        - internal
    labels:
        - traefik.enable=false
        - "com.centurylinklabs.watchtower.enable=true"

  app:
    image: nextcloud:stable
    links:
        - db
    volumes:
        - nextcloud:/var/www/html
    restart: always
    depends_on:
        - db
    labels:
        - traefik.backend=app
        - traefik.frontend.rule=Host:cloud.mondomaine.fr
        - traefik.docker.network=web
        - traefik.port=80
        - "com.centurylinklabs.watchtower.enable=true"
    networks:
        - internal
        - web

  lighttpd:
    image: sebp/lighttpd
    volumes:
        - /home/pi/projets_docker/lighttpd/htdocs:/var/www/localhost/htdocs
        - /home/pi/projets_docker/lighttpd/config:/etc/lighttpd
        - /home/pi/projets_docker/lighttpd/html:/var/www/localhost/html
    labels:
        - traefik.backend=lighttpd
        - traefik.frontend.rule=Host:www.mondomaine.fr
        - traefik.docker.network=web
        - traefik.port=80
        - "com.centurylinklabs.watchtower.enable=true"
    networks:
        - internal
        - web
    tty: true
    restart: always
```

On relance alors nos conteneurs :
```
docker-compose stop
```
```
docker-compose up -d
```

Et voilà, on dispose désormais d'une solution logicielle permettant de maintenir nos conteneurs à jour !