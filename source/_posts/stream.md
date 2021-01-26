---
title: "Bonus : Raspberry Pi, Streaming vidéo et Termux (5/4)"
date: 2020-11-03 19:00:00
tags:
- raspberry pi
- linux
- jisti
- vlc
- android
- ssh
---

## A propos

L'idée de base était d'ajouter une fonction de surveillance vidéo au Raspberry Pi. J'avais déjà travaillé avec *Motion*, cependant malgré tous ces avantages, ce dernier n'était pas capable de réaliser de capture audio.

Or dans l'idéal, ce que je voulais, c'était être capable de rediriger le flux vidéo et audio d'une webcam sur internet pour qu'il puisse être consulté depuis n'importe quel appareil, le tout dirigé par un contrôle à distance simpliste réduit à deux commandes : `camera-on` pour diffuser le flux, et `camera-off` pour le couper.

Deux solutions ont été trouvé, une première via *VLC Media Player*, auto-hébergée, qui ne fonctionnait pas de manière optimale (latence, décalage, micro-coupures), et une seconde, via les serveurs *Jitsi*, qui fonctionne parfaitement, mais qui ne peut pas s'auto-héberger car Jitsi est incompatible avec une architecture *ARM*.

## Prérequis
1. Branchez la webcam au Raspberry Pi
2. Installez les paquets `pulseaudio` et `pavucontrol`
3. A l'aide de la commande suivante, identifiez la référence du micro
   ```
   pactl list short sources
   ```
   Pour la suite, l'identitifiant de notre appareil video sera `/dev/video0` (par défaut), et l'identifiant de notre micro sera `alsa_input.usb-046d_0825_515522D0-02.analog-mono` repéré grâce à la portion `usb` de l'identifiant

## Diffusion par VLC

Cette diffusion peut se faire de deux manière, une première graphiquement, mais qui semble ne pas parvenir à se connecter au micro (à cause d'un problème de partage de process entre alsa utilisé par défaut et pulseaudio), et une seconde en ligne de commande.
- Graphiquement, il faut :
  - ouvrir VLC, puis 
  - *Media* > *Stream...* > *Capture Device*,
  - sélectionner l'appareil vidéo et audio, 
  - *Stream*, 
  - sélectionner dans *New destination* > *HTTP*, 
  - sélectionner le port et le chemin, 
  - *Next*, 
  - *Activate Transcoding*, 
  - choisir son profil (ici au hasard *Video - DIV3 + MP3 (ASF)*), 
  - *Next*, 
  - et enfin *Stream*
- En ligne de commande, il suffit de saisir :
  ```
  vlc -vvv v4l2:///dev/video0 :input-slave=pulse://alsa_input.usb-046d_0825_515522D0-02.analog-mono :live-caching=300 :sout="#transcode{vcodec=DIV3,vb=800,acodec=mp3,ab=128,channels=2,samplerate=44100,scodec=none}:http{mux=asf,dst=:8080/}" :no-sout-all :sout-keep
  ```
  > Notez l'utilisation de `pulse://` au lieu d'Alsa

Pour lire le flux vidéo & audio diffusé, il suffit ensuite d'ouvrir VLC sur une autre machine, puis *Media* > *Ouvrir un flux réseau* > `http://<IPv4 du pi>:8080`

> Des performances meilleures doivent pouvoir s'obtenir en jouant sur le transcodage, mais après plusieurs essais différents je n'ai rien trouvé de concluant

## Diffusion par Jitsi

Jitsi est à la base un service libre de visioconférence, ne nécessitant ni logiciel client, ni logiciel serveur si on passe par le site https://meet.jit.si/.

1. Générez une suite aléatoire de 20 caractères alphanumériques
   ```
   head /dev/urandom | tr -dc A-Za-z0-9 | head -c 20 ; echo ''
   ```
2. Interposez un `/` au milieu de cette suite, qui sera alors coupée en deux parties. Par exemple `4rChtdv2w/Qw7BiojGHBA`
3. Connecter *graphiquement* votre Raspberry Pi depuis *Chromium* (malheureusement nécessaire) à l'adresse https://meet.jit.si/4rChtdv2w/Qw7BiojGHB, en adaptant l'adresse à la suite aléatoire générée
4. Sur la page d'accueil, activez la caméra et le micro, saisissez un nom, et **cochez** *"Ne plus afficher ceci*
   > Note : Depuis X2Go, le micro refusait de s'activer, car la configuration audio du raspberry pi était écrasée par la configuration audio de mon PC, et que je n'avais pas de micro sur mon PC. Je suis donc passé par VNC
5. Cliquez sur *Rejoindre la réunion*, assurez-vous que tout fonctionne en vous connectant à la conférence avec un second appareil (votre PC ou smartphone par exemple) en recopiant l'URL de la conférence
6. Si tout fonctionne comme vous le souhaitez, copiez l'URL de la conférence et quittez chromium.
7. La commande pour se connecter via un terminal est 
   ```
   DISPLAY=:0 chromium-browser --kiosk --disable-session-crashed-bubble --disable-infobars --disable-restore-session-state https://meet.jit.si/4rChtdv2w/Qw7BiojGHB &
   ```
   Vous pouvez la tester via SSH depuis votre PC
8. Créez 2 alias dans votre `.bashrc`
   ```
   nano ~/.bashrc
   ```
   ```
   alias camera-off='killall chromium-browse'
   alias camera-on='DISPLAY=:0 chromium-browser --kiosk --disable-session-crashed-bubble --disable-infobars --disable-restore-session-state https://meet.jit.si/4rChtdv2w/Qw7BiojGHB &'
   ```
9. Actualisez votre bash
   ```
   source .bashrc
   ```
Vous pouvez désormais activer/désactiver votre caméra depuis ssh via les commandes `camera-on`/`cameraoff`, et consulter le flux vidéo depuis depuis un autre appareil en se connectant à l'adresse https://meet.jit.si/4rChtdv2w/Qw7BiojGHB générée

## Activation/Désactivation par Smartphone : SSH via Termux
> Testé sous Android.

1. Téléchargez l'application *Termux*, puis lancez-là.
2. Configurez Termux en tapant les commandes suivantes :
   ```
   termux-setup-storage
   ```
   ```
   pkg openssh
   ```
3. Créez le dossier suivant :
   ```
   mkdir ~/.ssh
   ```
4. Pour la suite, vous pouvez soit générer un jeu de clé RSA comme vu dans l'article précédent, soit transférer une copie de la clé privée précédemment générée dans le dossier `Documents` de votre smartphone, au quel cas :
5. Déplacez la clé dans le dossier `.ssh` :
   ```
   mv /sdcard/Documents/pi ~/.ssh/
   ```
6. Modifiez les droits de la clé :
   ```
   chmod 700 .ssh/pi
   ```
7. Creez vos alias :
   ```
   nano ../usr/etc/bash.bashrc
   ```
   ```
   alias camera-off='killall chromium-browse'
   alias camera-on='DISPLAY=:0 chromium-browser --kiosk --disable-session-crashed-bubble --disable-infobars --disable-restore-session-state https://meet.jit.si/4rChtdv2w/Qw7BiojGHB &'
   ```
8. Actualisez votre bash
   ```
   source .bashrc
   ```

Vous pouvez ensuite vous connecter en utilisant votre clé par SSH, et même pourquoi pas créer un alias de cette commande pour se connecter plus rapidement.