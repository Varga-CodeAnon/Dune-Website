---
title: Configuration initiale d'un Raspberry Pi 3B+ sans fil (1/4)
date: 2020-10-31 18:00:00
tags:
- raspberry pi
- linux
- ssh
---

## Avant-propos

Cette série de 4 articles a pour but de vous aider à configurer un Raspberry Pi 3B+ complètement vierge en un serveur web déployant un blog et un cloud personnel. 
Le Pi sera *Dockerisé* pour une facilité de déploiement et de maintenance. Il possèdera 4 conteneurs :
- un serveur web *Lighttpd*
- une instance *Nextcloud*
- Une base de données *MariaDB*
- Un reverse proxy *Traefik*
Il hébergera en plus le moteur de blog *Hexo* pour simplifier la construction de son site web statique. L'ensemble de ces technologies sont libres et open-sources.

Les outils utilisé ont été choisi pour que la configuration du Pi soit à la fois la plus **rapide**, la plus **simple**, la plus **facile** à maintenir et la plus **performante**.

La seule compétence requise pour aborder ces articles est d'avoir déjà utilisé une console linux une fois dans sa vie.

## Prérequis
- Un PC sous GNU/Linux
- Une carrte Raspberry Pi, son alimentation et sa carte microSD vierge ou préalablement formatée
- Une image de l'OS [Raspberry Pi OS](https://www.raspberrypi.org/downloads/raspberry-pi-os/)
- L'application [BalenaEtcher](https://www.balena.io/etcher/) pour flasher l'image

## Installation du système d'exploitation
1. Connecter la carte microSD au PC
2. Lancer *BalenaEtcher* en double-cliquant sur le fichier `balenaEtcher*.AppImage`
3. Dans l'application :
   1. Sélectionner l'image `*raspios*.img` préalablement téléchargée
   2. Sélectionner la carte microSD comme périphérique
   3. Flasher !

Votre carte SD contient désormais 2 partitions, `boot` et `rootfs`, signe que l'installation s'est déroulée correctement.

## Accès SSH & WiFi
   1. Ouvrir le gestionnaire de fichier, et aller dans la partition *boot* de la carte microSD
   2. Ici, créer un simple fichier vide nommé `ssh`. La présence de ce fichier informera l'OS que vous souhaitez activer SSH lors de son démarrage.
   3. Pour le WiFi, créer un second fichier nommé `wpa_supplicant.conf`
   4. A l'interieur, remplir le fichier comme il suit :
	```
	ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
	update_config=1
	country=FR
	network={
		ssid="votre-ssid"
		psk="votre-mot-de-passe"
	}
       ```
      Par exemple :
	```
	ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
	update_config=1
	country=FR
	network={
		ssid="Livebox-25DA"
		psk="s3cr3tpassw0rd!"
   	   }
       ```
	   > Si vous ne souhaitez pas inscrire le mot de passe en clair, vous pouvez indiquer directement la Pre-Shared Key (PSK), elle ne sera alors plus indiquée entre guillemets
   5. Une fois ces manipulations faites, vous pouvez retirer la carte microSD du PC, la connecter au Pi, puis le mettre sous alimentation pour l'allumer.
   6. Une fois allumé, vous pouvez vous connecter à votre Pi en saisissant dans un terminal la commande suivante, le mot de passe par défaut étant `raspberry` :
      ```
	  ssh pi@raspberrypi.local
	  ```
	  > Si cette commande ne fonctionne pas, connectez-vous à l'interface web de votre box internet (les modalités sont différentes selon les FAI). Rendez-vous ensuite dans la partie *Périphériques connectés* pour voir les appareils actuellement sur votre réseau.
	  > - Si vous ne voyez pas votre Pi, c'est qu'il y a un problème au niveau de la configuration du WiFi
	  > - Si vous voyez votre Pi, relevez son adresse IPv4 locale (par exemple `192.168.1.10`), puis saisissez la commande suivante :
	  > `ssh pi@192.168.1.10`

Vous voilà connecté à votre Raspberry Pi.

## Configuration initiale
Commener par mettre à jour votre système
```
sudo apt update && sudo apt full-upgrade
```

> On pourra créer un alias de commande pour des mises à jour plus simples. Pour cela, tapez `nano .bashrc`, puis ajouter en dessous des alias existant, la ligne suivante : `alias maj='sudo apt update && sudo apt full-upgrade'`. Pour la suite, il suffira de taper `maj` pour que le système se mette à jour.

Redémarrer ensuite le Pi.
```
sudo shutdown -r now
```

Ouvrir dans la console SSH l'outil de configuration
```
sudo raspi-config
```
On obtient alors les options suivantes :
1. Change User Password
1. Network Options
2. Boot Options
3. Localisation Options
4. Interfacing Options
5. Overclock
6. Advanced Options
7. Update
8. About raspi-config

Pour naviguer dans le menu de rapsi-config, vous devez utiliser les flèches directionnelles de votre clavier « haut » et « bas ».

N'hésitez pas à parcourir les options, et notamment changer le mot de passe utilisateur ainsi que l'heure du système

## Bureau à distance : configuration d'un serveur *X2Go*

*X2Go* est un logiciel client-serveur qui permet de se connecter à un ordinateur serveur linux distant, ce qui est le cas de notre Pi. Il s'appuie sur le protocole libre *freeNX*, très performant (beaucoup plus que VNC présent par défaut) et la navigation est fluide même avec une connexion à faible débit.

Il permet d'avoir accès au bureau en utilisant la carte vidéo et audio du client. La connexion est sécurisée par le protocole ssh.

Pour cela, il suffit d'installer sur le Pi le paquet `x2goserver`, et sur le PC `x2goclient` :
- Sur le Pi:
	```
	sudo apt install x2goserver
	```
- Sur le PC:
	```
	sudo apt install x2goclient
	```

Côté Pi, il n'y a plus rien a faire. Côté PC client, lancer X2GoClient pour le configurer. Nommez la session, laissez le chemin `/`, spécifiez l'adresse de votre Pi dans le champ *Hôte*, entrez le nom de l'utilisateur, laissez le port SSH par défaut, puis (si vous ne la connaissez pas, tapez la commande `ip a` sur votre Pi).
Dans type de session, sélectionnez `LXDE`. Vous pouvez ensuite cliquer sur *Ok*, sur votre session nouvellement créée apparaissant dans le bandeau droit de votre fenêtre X2Go, entrer les identifiants, et le tour est joué.

## Sécuriser SSH

Comme nous avons créé une porte d'entrée dans le système de notre Raspberry Pi, nous devons la sécuriser un minimum. En effet, le protocole SSH (aussi utilisé par X2Go) permet en l'état un accès total au système pour quiconque connait l'adresse IP du Pi, l'identifiant utilisateur et son mot de passe.

Pour renforcer la sécurité de notre Pi, nous allons utiliser une authentification par clé asymétrique combiné à une passphrase.

1. Éditez le fichier de configuration SSH en ayant effectué une sauvegarde au préalable
   ```
   sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
   sudo nano /etc/ssh/sshd_config
   ```
2. Décommentez la ligne `#PubKeyAuthentication yes` en effaçant le `#`, ainsi que la ligne `#PermitRootLogin prohibit-password`
3. Sur le PC, si ce n'est pas déjà fait, créez un répertoire .ssh avec des permissions restreintes
   ```
   mkdir -p ~/.ssh/ && chmod 0700 ~/.ssh/
   ```
4. Toujours sur le PC, générez la paire de clés. Vous serez amené à saisir 2 fois une passphrase.
   ```
   ssh-keygen -t rsa -b 8196 -C "Raspberry Pi" -f ~/.ssh/pi
   ```
   > L'option `-t` indique le mode de chiffrement, `-b` le nombre de bits, `-C` permet d'intégrer un commentaire, `-f` de nommer sa clé et de la placer dans le répertoire.

   Deux clés ont alors été créé, `pi` et `pi.pub`, respectivement clé privée et clé publique.
5. Déployez la clé publique dans le fichier `~/.ssh/authorized_keys` de votre Pi
   - Soit en tapant la commande suivante depuis votre PC
     ```
	 ssh-copy-id -i .ssh/pi.pub pi@raspberrypi.local
     ```
   - Soit avec X2Go, en créant le fichier `~/.ssh/authorized_keys` et en y copiant-collant le contenu de la clé publique presente sur le PC tel quel
6. Vérifiez le bon fonctionnement de la connexion par clé
   ```
   ssh -i ~/.ssh/pi pi@raspberrypi.local
   ```
7. Une fois que vous êtes certain que vous pouvez vous connecter via votre clé et votre passphrase, désactivez la connection SSH par mot de passe en changeant la ligne `#PasswordAuthentication yes` en `PasswordAuthentication no`
8. Redémarrez votre Raspberry Pi
   ```
   sudo shutdown -r now
   ```
9. Actualisez votre client X2Go. Pour se faire, allez dans les *Réglages de la session* > *Utiliser une clé RSA/DSA pour la connexion SSH* > *~/.ssh/pi*
10. Vérifiez que la connection fonctionne avec X2Go

Un premier niveau de sécurité vient d'être mis en place. D'autres seront abordés dans les prochains chapitres.

## Première sauvegarde

Maintenant que le Raspberry Pi est installé, configuré, accessible et sécurisé, procédez à une sauvegarde pour qu'en cas de problème ou d'acquisition d'une nouvelle carte Pi vous n'ayez pas à refaire les manipulations précédentes.

1. Assurez-vous tout d'abord que votre Pi soit éteint et débranché
2. Retirer la carte microSD, et connectez-là à votre PC
3. Créer une image système
   1. Soit avec l'interface graphique du paquet `gnome-disk-utility` exécutable via la commande `gnome-disks`
   2. Soit en ligne de commande (**utilisateur confirmé seulement**)
	  ```
	  dd bs=4M if=/dev/<sdX> of=<sauvegarde_pi.img> status=progress conv=fsync
      ```

Une fois l'image générée, vous pouvez rebrancher le Pi, allumer le système et l'utiliser comme bon vous semble.
