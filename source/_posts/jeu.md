---
title: "Installation manuelle d'un jeu Windows sous Linux : The Witcher 3 - GOTY"
date: 2020-11-07 19:00:00
tags:
- jeu
- wine
- gog
- witcher
- lutris
---

## Avant-propos

Cet article a pour but d'aider dans l'installation manuelle d'un jeu Windows sous un système Linux, lorsque les méthodes conventionnelles recommandées sur les plateformes *[Lutris](https://lutris.net/)* ou sur *[Wine](https://www.winehq.org/)* ne fonctionnent pas.

Pour l'exemple, on prendra le cas du jeu *The Witcher 3 : Wild Hunt*, édition *Game Of The Year* qui comprends le jeu de base avec des extensions. Il a été acheté sur la plateforme [GOG](https://www.gog.com/game/the_witcher_3_wild_hunt_game_of_the_year_edition). 

Lutris possède un script d'installation pour ce jeu, cependant il ne fonctionne pas, même après plusieurs essais. Le problème ne semble pas être isolé, nous allons donc procéder à une installation manuelle.

## Pré-requis
1. Assurez-vous qu'il est possible de jouer à votre jeu sur Linux. Cette méthode n'est pas miracle, si aucun *runner* tel que Wine ou Proton ne peut faire tourner le jeu, l'installation manuelle ne fonctionnera pas.

   Dans le cas de *The Witcher*, le jeu est sensé fonctionner sur la plateforme Lutris qui utilise Wine, c'est donc qu'il est possible de le faire tourner via le runner *Wine*

   > N'hésitez pas à regarder du côté de forums ou plateformes communautaires telles que *reddit* pour voir si des joueurs ont déjà réussi à faire tourner votre jeu sous Linux

2. Installez Lutris :
   
   Lutris est une plateforme de gaming open-source pour linux. Elle permet de jouer à des jeux *Steam*, *GOG*, *Battle.net*, *Origin*, *Uplay* et bien d'autres.
   - Ajoutez le dépôt :
	 ```
	 echo "deb http://download.opensuse.org/repositories/home:/strycore/Debian_10/ ./" | sudo tee /etc/apt/sources.list.d/lutris.list
	 ```
   - Ajoutez la clé du dépôt :
     ```
	 wget -q https://download.opensuse.org/repositories/home:/strycore/Debian_10/Release.key -O- | sudo apt-key add -
	 ```
   - Mettez à jour `apt` et installez Lutris :
     ```
	 sudo apt update && sudo apt install lutris
	 ```

3. Vérifiez que vous disposez d'assez d'espace disque pour procéder à l'installation.

4. Rendez-vous sur la plateforme de téléchargement de votre jeu, puis téléchargez l'installer (ici, l'installer *GOG-Galaxy*)

Vous pouvez désormais procéder à l'installation.

## Initialisation du préfixe

Le préfixe est une simulation d'une arborescence Windows de type `C:\Program Files\...` dans laquelle va s'installer l'application. Pour le créer, procédez comme il suit :

1. Ouvrez Lutris
2. Appuyez sur le logo dans le coin superieur gauche, puis sur *Manage Runners*
3. Descendez sur le runner *Wine*, puis cliquez sur *Manage Versions*
4. Vérifiez que la version `lutris-5.7*` est sélectionnée
5. Fermez cette sous-fenêtre pour revenir à l'affichage principal, puis cliquez sur le "+" > *Add Game...*
6. Dans le premier onglet *Game info*, donnez un nom au jeu et choisissez *Wine* comme runner dans le menu déroulant
7. Dans le second onglet *Game options*, cliquez sur le bouton *Browse* à côté d'*Executable* puis allez chercher l'installer GOG-Galaxy
   > Cela sera modifié plus tard
8. Ignorez les champs *Arguments* et *Working directory*
9. Dans *Wine prefix*, sélectionnez le dossier dans lequel vous souhaitez créer votre préfixe (dans notre cas `~/Games/witcher/`)
10. Sélectionnez comme architecture *64-bit* si votre jeu est en 64-bit
11. Dans l'onglet *Runner options*, sélectionnez comme version de Wine la `lutris-5.7*` (en principe celle par défaut), puis activez l'option *DXVK*
    > Cela peut être changé à tout moment 
12. Ignorez l'onglet *System options*, puis sauvegardez et fermez la sous-fenêtre

Vous devriez désormais voir votre jeu dans la liste de la page principale Lutris. Cependant s'il n'y est pas, il s'agit d'un bug mineur ; redémarrez simplement l'application

## Installation 

Pour procéder à l'installation, vous pouvez simplement lancer votre jeu depuis Lutris. L'installeur devrait s'exécuter, et télécharger les fichiers. 

A partir de là, procédez à l'installation comme vous le feriez sur Windows, les fichiers seront automatiquement déployé dans le préfixe configuré précédemment.

## Configuration finale

Une fois que le jeu est installé et qu'il fonctionne, quittez le pour revenir sur la fenêtre d'accueil Lutris.

Faites un clic droit sur la bannière de jeu, sélectionnez *Configure* > *Game Options*, et changez l'*Executable* pour qu'il pointe vers l'actuel `.exe` du jeu et non plus l'installer.

Dans notre cas, il se trouve à l'adresse suivante :
```
~/Games/witcher/drive_c/Program Files (x86)/GOG Galaxy/Games/The Witcher 3 Wild Hunt GOTY/bin/x64/witcher3.exe
```

Vous pouvez sauvegarder et jouer à votre jeu sur Linux comme si vous étiez sur Windows.

> Remerciements à *toidi* du forum de lutris pour son [tutoriel](https://forums.lutris.net/t/very-brief-tutorial-on-manually-installing-a-game-in-lutris/2028) dont cet article s'inspire plus que largement
