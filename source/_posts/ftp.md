---
title: "Déploiement en local d'un serveur Pure-FTPd sécurisé"
date: 2021-01-19 19:00:00
tags:
- raspberry pi
- ftp
- pure-ftpd
- ftps
---

## Avant-propos

Pour éditer plus facilement les articles de nos différents blogs, il devenait nécessaire de déployer un serveur FTP. Un serveur *File Transfer Protocol* permet le transfert de fichier entre machines distantes.

Comme nous en avons uniquement besoin à des fins d'administration,  nous le déploierons uniquement sur le réseau local (par mesure de sécurité). 

Il existe beaucoup de serveurs FTP pour Debian. On choisira ici *Pure-FTPd* qui se veut être le plus épuré et le plus simple possible.

## Installation de Pure-FTPd

Comme nous sommes sous notre Raspberry Pi 4, nous sommes sous Debian Buster. L'installation est alors très simple : 

```
sudo apt update && sudo apt upgrade -y
sudo apt install pure-ftpd
```

## Configuration

Le fichier de configuration se trouve à l'adresse `/etc/pure-ftpd/pure-ftpd.conf`. On réalise alors une backup par précaution avant toute édition :

```
sudo cp /etc/pure-ftpd/pure-ftpd.conf /etc/pure-ftpd/pure-ftpd.conf.bak
sudo nano /etc/pure-ftpd/pure-ftpd.conf
```

> Vous pouvez retrouver le fichier de configuration commenté en français sur le [Fedora Wiki](https://doc.fedora-fr.org/wiki/PureFTPD_:_Installation_et_configuration#Proc.C3.A9dure_d.27installation)

On modifiera alors les paramètres suivant pour restreindre la connexion aux utilisateurs virtuels :
```conf
ChrootEveryone		yes
MaxClientsNumber	10
Daemonize		yes
MaxClientsPerIP	        3
NoAnonymous		yes
SyslogFacility		ftp
DontResolve		yes  # (on limite les résolutions DNS)
PureDB	                /etc/pure-ftpd/pureftpd.pdb # On décommente la ligne
PAMAuthentication	no
```

### Création de l'utilisateur virtuel

Pour sécuriser notre serveur, nous allons créer des utilisateurs connus seulement de pure-ftpd. Ces utilisateurs seront mappés sur de véritables utilisateurs unix.

On va donc créer notre utilisateur virtuel comme ceci :
```
sudo pure-pw useradd ftp-user -u pi -d /home/pi/projets_docker -m
```
> Note : le mot de passe pourra être changé avec la commande `pure-pw passwd <username>`

- `-u` : login de l'utilisateur dont on utilisera l'UID pour l'utilisateur virtuel
- `-d` : répertoire racine de l'utilisateur virtuel sans possibilité de remonter d'un niveau (*chroot*)
- `-m` : mise à jour automatique de la base de données des utilisateurs virtuels (/etc/pure-ftpd/pureftpd.pdb).

On pourra visualiser les informations de l'utilisateur via la commande suivante :
```
sudo pure-pw show ftp-user
```

Pour permettre à l'utilisateur de s'authentifier, on crée ensuite un lien symbolique vers la base de données des utilisateurs avec cette commande : 

```
sudo ln -s /etc/pure-ftpd/conf/PureDB /etc/pure-ftpd/auth/75puredb
```

Et on redémarre le serveur pour que les modifications prennent effet :
```
sudo /etc/init.d/pure-ftpd restart
```

### Ajout du support SSL/TLS

Pour éviter que nos informations circulent en clair, nous allons rajouter une couche de cryptage SSL/TLS. Comme on travaille sur un réseau local dans un but administratif, nous allons nous même générer le certificat sans passer par *Let's Encrypt*.

Commençons par rajouter l'option suivante dans le fichier de configuration :
```
sudo nano /etc/pure-ftpd/pure-ftpd.conf
```
```sh
TLS 2                                    # Refuse les connexions sans TLS
TLSCipherSuite HIGH                      # A décommenter
CertFile /etc/ssl/private/pure-ftpd.pem  # A décommenter
```
Puis on ajoute un fichier de configuration supplémentaire **en tant qu'utilisateur root** :
```
echo 2 > /etc/pure-ftpd/conf/TLS
```

On crée ensuite le certificat :
```
mkdir -p /etc/ssl/private/
sudo openssl req -x509 -nodes -days 7300 -newkey rsa:2048 -keyout /etc/ssl/private/pure-ftpd.pem -out /etc/ssl/private/pure-ftpd.pem
```
On sécurise son accès :
```
sudo chmod 600 /etc/ssl/private/pure-ftpd.pem
```
Et on redémarre le serveur pour que les modifications prennent effet :
```
sudo /etc/init.d/pure-ftpd restart
```
> Il se peut que simplement redémarrer le serveur ne suffise pas. Il faudra alors redémarrer la machine

### Connexion

Pour intéragir avec notre serveur, nous pouvons utiliser le client FileZilla (`sudo apt install filezilla`). Pour se faire, il suffit de cliquer sur le bouton en haut à gauche *Ouvrir le gestionnaire de Sites* > *Nouveau Site*, et le remplir comme ceci :
- *Protocole:* `FTP - Protocole de Tranfert de fichiers`
- *Hôte*: `192.168.1.20`		*Port*: `21`
- *Chiffrement*: `Connexion FTP explicite sur TLS`
- *Type d'authentification:* `Demander le mot de passe`
- *Idenfitiant:* `ftp-user`
- Mot de passe: `   `

Une fois validé, la connexion pourra s'effectuer depuis le menu déroulant du bouton *Ouvrir le gestionnaire de Sites.*

### Désactivation du protocole SFTP du serveur SSH

Pour forcer la connexion à notre serveur FTP limitant ainsi les droits d'accès et les permissions lors du transfert de fichier, nous allons désactiver SFTP dans le fichier de configuration du service SSH.
```
sudo nano /etc/ssh/sshd_config
```

Tout en bas du fichier se trouve la ligne suivante qu'il vous faudra commenter :
```
# Subsystem sftp /usr/lib/openssh/sftp-server
```

Vous pouvez alors redémarrer SSH, en prenant note que vous pourrez perdre la connexion si vous étiez connecté à votre machine distante via SSH
```
/etc/init.d/ssh restart
```

Vous pouvez maintenant profiter d'une solution de transfert de fichiers sécurisée !
