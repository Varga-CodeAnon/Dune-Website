---
title: "Création d'un second blog : Traefik & VHosts Lighttpd"
date: 2021-01-13 19:00:00
tags:
- raspberry pi
- linux
- traefik
- lighttpd
- vhost
- conteneur
---

## A propos

Dans un [précédent post](https://www.dune.ovh/2020/11/03/raspi_nextcloud/), nous avions vu le déploiement du reverse proxy *Traefik* pour rediriger les flux vers nos différents conteneurs. Nous [avions aussi](https://www.dune.ovh/2020/11/02/raspi_docker_lighttpd/) déployé un conteneur *Lighttpd* pour héberger notre premier blog.

L'idée ici est d'ajouter un second blog, non pas sous le nom de domaine `dune.ovh`, mais `shian.ovh`, le tout sur le même conteneur *Lighttpd* grâce aux **virtual hosts**.

## Création des virtual hosts

> Il existe globalement deux méthodes pour procéder. La première serait de séparer distinctement les deux vhosts, avec des dossiers, des fichiers de configuration, et des fichiers de journalisation différents, et la seconde serait de simplement travailler dans le fichier `/etc/lighttpd/lighttpd.conf`
>
> Comme nous travaillons avec un conteneur et pour nous faciliter la vie, nous opterons pour la seconde méthode.

### Cloisonner l'espace

1. On commence par créer deux répertoires dans notre dossier `lighttpd/html/`, ils contiendront les fichiers sources de nos deux blogs
   ```
   mkdir dune && mkdir shian
   ```

> Si vous utilisiez *hexo* sur votre premier blog, pensez à modifier le chemin de votre répertoire `public` dans le fichier `_config.yml` du générateur de blog !

2. Dans le dossier `dune`, nous glissons les fichiers sources déjà existant. A ce moment là, le site devient inaccessible depuis internet, renvoyant en toute logique une erreur 403.
3. A l'intérieur du dossier `shian`, nous créons une simple page html qui nous permettra de voir si notre vhost est bien accessible par le navigateur.
   ```
   nano ~/projets_docker/lighttpd/html/shian/index.html
   ```
   ```
   <html>
   <body>
   Bienvenue sur Shian, blog en construction.
   </body>
   </html>
   ```

### Configurer Lighttpd

1. Pour configurer notre serveur web, rien de plus simple, il suffit de rajouter les blocs de code suivant à la fin du fichier `projets_docker/lighttpd/config/lighttpd.conf`
   ```
   $HTTP["host"] =~ "(^|.)www.dune.ovh$" {
    server.document-root = "/var/www/localhost/html/dune/"
    server.errorfile-prefix = "/var/www/localhost/htdocs/errors/status-"
   }

   $HTTP["host"] =~ "(^|.)www.shian.ovh$" {
      server.document-root = "/var/www/localhost/html/shian/"
      server.errorfile-prefix = "/var/www/localhost/htdocs/errors/status-"
   }
   ```
2. On redémarrera enfin notre conteneur
   ```
   docker container restart sebp/lighttpd 
   ```

## Configuration de Traefik

1. On se rend à l'adresse de notre reverse proxy, ici en l'occurrence `traefik.dune.ovh`
2. Dans notre panneau de configuration, section `BACKENDS`, on peut observer une adresse IP affectée à notre `backend-lighttpd`. On la note alors (dans notre exemple ce sera http://172.19.0.5).

3. Lors du déploiement de notre conteneur, nous avions configuré notre reverse proxy avec un backend de type *docker* pour qu'il détecte automatiquement les conteneurs à déservir, et ce via la balise `[docker]`

Pour les vhosts, nous allons rajouter à la fin du fichier de configuration `traefik.toml` un nouveau type de backend via la balise `[file]` comme il suit, en y ajoutant l'adresse IP du backend-lighttpd relevée précédemment :

```toml
# Parametrage des VHOSTS
[file]

[backends]

  [backends.webserver]

    [backends.webserver.servers]
      [backends.webserver.servers.server0]
        url = "http://172.19.0.5" # Visible depuis le dashboard traefik

# Frontends
[frontends]
  [frontends.frontend1]
      backend = "webserver"
      passHostHeader = true
      [frontends.frontend1.routes.route0]
      rule = "Host:www.dune.ovh" # Premier nom de domaine de notre premier blog

  [frontends.frontend2]
      backend = "webserver"
      passHostHeader = true
      [frontends.frontend2.routes.route0]
      rule = "Host:www.shian.ovh" # Second nom de domaine de notre second blog
```

4. On redémarrera enfin notre conteneur
   ```
   docker container start traefik
   ```

5. En se reconnectant au dashboard Traefik, on remarquera qu'un nouvel onglet est apparu, l'onglet *file* à côté de l'onglet *docker*, dans lequel apparaissent nos deux vhosts.

   Il suffit alors de rentrer leur adresse url dans notre navigateur pour y accéder, et le tour est joué !

Et voilà, nos vhosts sont désormais accessibles depuis l'extérieur, et notre serveur web héberge nos deux blogs sous deux noms de domaines différents.

> Note : Si vous ne disposez pas d'un nom de domaine, vous pouvez en simuler un localement. Par exemple, nous ne possédons pas le nom de domaine *shian.ovh*. On pourra alors rajouter dans notre fichier `/etc/hosts/` la ligne suivante `192.168.1.20    www.shian.ovh` en renseignant bien évidemment l'adresse IP de notre serveur web.
>
> Une fois cela fait, il suffira de taper *www.shian.ovh* dans notre navigateur pour qu'il accède automatiquement à notre vhost.
