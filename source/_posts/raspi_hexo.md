---
title: Raspberry Pi et le générateur de blog Hexo (3/4)
date: 2020-11-02 19:00:00
tags:
- raspberry pi
- linux
- hexo
- blog
---


*Hexo* est un moteur de blog simple, rapide et puissant. Il suffit en effet d'écrire ses articles en *markdown* pour qu'Hexo génère en quelques secondes un site web statique avec des thèmes personnalisables.

## Installation

Il est au préalable nécessaire d'avoir installé *Node.js* version 10.13 ou supérieure (12.0 recommandée), ainsi que *Git*. Vous pouvez les installer respectivement avec les commandes suivantes :
```
curl -sL https://deb.nodesource.com/setup_15.x | sudo bash -
sudo apt-get install -y nodejs
```
```
sudo apt-get install git-core
```
Enfin, Hexo peut être installé avec la commande suivante :
```
sudo npm install -g hexo-cli
```

## Initialisation

Une fois Hexo installé, on doit l'initialiser dans notre dossier cible qui contiendra nos articles ainsi que les fichiers de configurations.

```
hexo init <folder>
cd <folder>
npm install
```

Par la suite, notre dossier projet `<folder>` devrait ressembler à ça :
```
.
\u251c\u2500\u2500 _config.yml
\u251c\u2500\u2500 package.json
\u251c\u2500\u2500 scaffolds/
\u251c\u2500\u2500 source/
|   \u251c\u2500\u2500 _drafts
|   \u2514\u2500\u2500 _posts
\u2514\u2500\u2500 themes/
```

## Configuration

Une fois initialisé, on peut configurer notre moteur de blog en éditant le fichier `_config.yml`. Pour déployer le blog sur le serveur lighttpd, modifiez la ligne `public_dir` de la manière suivante :
```
public_dir: /home/pi/projets_docker/lighttpd/html
```

> Les utilisateurs avancés peuvent aussi utiliser un fichier de [configuration alternatif](https://hexo.io/docs/configuration#Using-an-Alternate-Config)

> Pour consulter un lexique des paramètres et savoir ce qu'il font, vous pouvez regarder cette [fiche](https://hexo.io/docs/configuration) (EN) en annexe.


## Application d'un thème

1. Téléchargez le thème de votre choix depuis [cette adresse](https://hexo.io/themes/)
2. Glissez dans le dossier `themes` de votre répertoire hexo le nouveau thème, qui se trouvera donc à côté du dossier `landscape` qui est le thème par défaut
3. Éditez le fichier `_config.yml` présent dans le dossier de votre thème pour le configurer.
4. Concernant l'arborescence du dossier :
   - Dans le dossier `languages`, vous trouvez les différentes traduction des mots-clés générés par défaut par le thème.
   - Dans le dossier `layout`, vous pouvez éditer le rendu des pages, par exemple de la page d'accueil en éditant le fichier `index.ejs`.
   - Dans le dossier `script`, vous pouvez ajouter/supprimer des scripts 
   - Dans le dossier `source`, vous pouvez déposer des images et modifier les feuilles de styles `css`.
5. Enfin, éditez la ligne `theme: landscape` pour y inscrire le nom du dossier de votre thème et ainsi l'appliquer

## Ajout d'une barre de recherche

1. Dans le dossier Hexo, il suffit de lancer la commande suivante
   ```
   npm install hexo-generator-search --save
   ```
2. Ensuite, créez une nouvelle page pour afficher la barre de recherche
   ```
   hexo new page search
   ```
3. Inscrivez-y l'en-tête suivante
   ```
   ---
   title: Search
   type: search
   ---
   ```
4. Enfin, éditez le fichier `_config.yml` et ajoutez les lignes suivantes
   ```
   nav:
   search: /search/
   ```

## Génération du blog
Tout simplement :
```
hexo generate
```
Pour vérifier que tout fonctionne, ouvrez le navigateur de votre PC à l'adresse de votre Raspberry Pi, et admirez votre nouveau blog.

## Ecriture de son premier post

Pour rédiger un post, il suffit d'écrire un texte en markdown et déposer le fichier dans le sous-dossier `source/_posts` de votre répertoire Hexo. 

Une en-tête est cependant requise, vous pouvez vous inspirer de celle-ci :
```
---
title: Configuration d'un Raspberry Pi 3B+ sans fil
date: 2020-10-31 18:00:00
tags:
- raspberry pi
- linux
---
```

Penser ensuite à générer votre blog avec la commande `hexo generate`