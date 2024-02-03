# Installation composant de base

Ce document décrit la procédure d'installation de tous les composants de base nécessaire sur chaque serveur. Toutes les autres procédures partiront du principe que les composants ci-dessous sont bien installées et configurée.

## Connexion par clef SSH uniquement

https://docs.ovh.com/fr/public-cloud/premiers-pas-instance-public-cloud/#etape-1-creer-des-cles-ssh

https://docs.ovh.com/fr/public-cloud/configurer-des-cles-ssh-supplementaires/

https://docs.ovh.com/fr/public-cloud/changer-sa-cle-ssh-en-cas-de-perte/

https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys-on-ubuntu-20-04-fr

https://tutox.fr/2020/04/16/generer-des-cles-ssh-qui-tiennent-la-route/

### Génération clef SSH

Dans un premier temps il faut générer une paire de clef privée/publique soit via PuttyGen (Windows), soit via ssh-keygen (Linux)

**PuttyGen**

- Ouvrir PuttyGen
- Si votre serveur est compatible généré une clef avec algorithme EdDSA (Depuis Debian 8)
  - Key > SSH 2 EdDSA Key ou dans Parameters > Type of Key > EdDSA 
    - Attention souvent le 448 bits n'est pas supporté (ed448)
    - Utiliser plutôt le 255 bits (ed25519)
  - Sinon utilisé RSA (algorithme par défaut) avec un clef de 4096 bits
- Modifier les paramètres de génération en fonction de la sécurité souhaitée (dans Key et Parameters)
- Cliquer sur Generate
- Modifier le nom si besoins
- Ajouter une mot de passe pour plus de sécurité
- Sauvegarder la clef publique (.ppuk) et la clef privée (.ppk) au format Putty
- Sauvegarder la clef privée au format OpenSSH
  - Conversions > Export OpenSSH Key
  - Nommer le fichier `<nom-clef>.<algo>` (Exemple : site-com.eddsa ou site-com.rsa)
- Sauvegarder la clef publique au format OpenSSH
  - Copier le contenue de Key > Public key for pasting into OpenSSH authorized_key file
  - Coller dans un fichier nommé `<nom-clef>.pub`

**ssh-keygen (OpenSSH)**

Utiliser une des commandes suivantes puis suivre les instructions. Par défaut un fichier nommé `<clef>.pub` et `<clef>` vont être crée dans le dossier `.ssh` de l'utilisateur. Ces fichiers correspondent respectivement à la clef publique et la clef privée.

```bash
# Générer une paire de clef avec les paramétres par défaut (en géneral RSA 2048 bits)
ssh-keygen
# Générer une paire de clef RSA de 4096 bits (recommandé si l'on doit utiliser RSA)
ssh-keygen -t rsa -b 4096
# Générer une paire de clef EdDSA 255 bits (recommandé si le serveur est compatible)
ssh-keygen -t ed25519
```

Pour obtenir les versions Putty des clefs publique et privée il faut importer le fichier de clef privée OpenSSH dans PuttyGen et utiliser les boutons pour sauvegarder au format Putty.

### OVH

Aouter la clef publique dans `Mon Compte` > `Mes services` > `Clef SSH`, puis réinstaller le serveur avec authentification par clef ssh. Attention ceci n'est possible que pour installer une première clef (et ne pas générer de mot de passe pour le compte par défaut). Pour ajouter une autre clef voir la section **Manuel**

### Manuel

Si le PC (et non le serveur) est un linux et qu'il est possible de se connecter par mot de passe en SSH (Attention il faut dans le dossier un fichier nommé `<clef>.pub` pour la clef publique et un fichier nommé `<clef>` pour la clef privé, sinon il y a juste la clef publique utilisé l'option `-f` pour envoyer son contenue sans vérification)

```bash
# Copie automatiquement la clef publique du PC
ssh-copy-id <username>@<remote_host>
# Copie une clef spécifique sur le serveur
ssh-copy-id -i <path-to-key.pub> <username>@<remote_host>
# Copie une clef spécifique sur le serveur via un port différent de celui par défaut (22)
ssh-copy-id -i <path-to-key> -p <port> <username>@<remote_host>
```

Sinon par SSH (la clef doit être dans le format pub de ssh-keygen)

```bash
cat <path_to_key> | ssh <username>@<remote_host> "mkdir -p ~/.ssh && touch ~/.ssh/authorized_keys && chmod -R go= ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

## Mise en place

```bash
# Renommer le serveur
sudo nano /etc/hostname
  # Modifier par le nouveau nom du serveur
sudo nano /etc/hosts
  # Modfier la ligne 127.0.1.1 par le nouveau nom du serveur
# Mise à jour
sudo apt update
sudo apt upgrade
# Rédemmarer le serveur
sudo shutdown -r now
```

### Maj automatique

https://www.cyberciti.biz/faq/how-to-set-up-automatic-updates-for-ubuntu-linux-18-04/

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
  # Ajouter les lignes suivantes
  #   Unattended-Upgrade::Automatic-Reboot "true";
```

### Sécurisation

https://docs.ovh.com/fr/vps/conseils-securisation-vps/

Nouvel utilisateur et connexion SSH

```bash
# Changement port ssh
sudo nano /etc/ssh/sshd_config
  # Modifier le numéro sur la ligne : Port 22 (décommenter si besoins)
  # Mettre PermitRootLogin à la valeur no (décommenter si besoins)
  # Mettre PubkeyAuthentication yes
  # Mettre PasswordAuthentication à la valeur no
sudo systemctl restart sshd
# Relancer la connexion SSH
# Création d'un nouvel utilisateur (et d'un groupe avec le même nom)
sudo adduser <user>
sudo usermod -aG sudo <user>
sudo mkdir /home/<user>/.ssh
sudo cp $HOME/.ssh/authorized_keys /home/<user>/.ssh
# Aller sur le nouveau compte (il possible de basculer avec la commande : su - <user>)
# Les commandes suivantes sont à faire sur le nouveau compte
sudo visudo
  # Ajouter la ligne suivante avant la derniere ligne du fichier
  #  <user> ALL=(ALL) NOPASSWD: ALL
sudo chown -R <user>:<group> $HOME/.ssh
sudo systemctl restart sshd
# Deconnexion reconnexion (normalement sans mot de passe)
sudo echo 1
  # Pour tester le sudo sans mot de passe
# Si tout est bon à partir d'ici on utilise plus le compte par défaut
sudo usermod --expiredate 1 <login-par-defaut>
  # Tester la connexion au compte par défaut pour être sur qu'elle est desactivée
# Activer les couleur dans le bash
nano $HOME/.bashrc
  # Décommenter la ligne force_color_prompt=yes
source $HOME/.bashrc
```

Firewall

https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-ubuntu-20-04-fr

https://doc.ubuntu-fr.org/ufw

```bash
sudo apt install -y ufw
sudo nano /etc/default/ufw
  # Verifier que IPV6=yes
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow <port-ssh>
  # Possible d'ajouter d'autre port ou restriction, voir les guides
sudo ufw enable 
sudo ufw status verbose
  # Permet de voir les regles en place
```

### Logiciel

https://docs.docker.com/engine/install/linux-postinstall/

https://docs.npmjs.com/resolving-eacces-permissions-errors-when-installing-packages-globally

https://adoptium.net/installation.html?variant=openjdk11

```bash
sudo apt install -y git php nodejs npm docker docker-compose unzip
# PHP
php -v
  # Si tous est ok affiche la version de PHP on peut installer les modules complémentaires
sudo apt install -y php-pdo php-pgsql php-mysql php-sqlite3 php-zip php-gd php-mbstring php-curl php-xml php-pear php-bcmath
# Composer (Gestion dépendance PHP)
cd $HOME
curl -sS https://getcomposer.org/installer -o composer-setup.php
  # Verification du HASH
HASH=`curl -sS https://composer.github.io/installer.sig`
php -r "if (hash_file('SHA384', 'composer-setup.php') === '$HASH') { echo 'Installer verified'; } else { echo 'Installer corrupt'; unlink('composer-setup.php'); } echo PHP_EOL;"
  # Installation
sudo php composer-setup.php --install-dir=/usr/local/bin --filename=composer
rm $HOME/composer-setup.php
composer
  # La commande devrait fonctionner
# Docker
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker
docker run hello-world
  # Si tous est ok le docker ce lance
# Node
mkdir $HOME/.npm-global
mkdir $HOME/.node-n
npm config set prefix '~/.npm-global'
  # Ajoute les lignes à la fin du fichier .profile
echo "" >> "$HOME/.profile"
echo "# Node env" >> "$HOME/.profile"
echo "export N_PREFIX=\"\$HOME/.node-n\"" >> "$HOME/.profile"
echo "export PATH=\"\$HOME/.npm-global/bin:\$PATH\"" >> "$HOME/.profile"
echo "export PATH=\"\$HOME/.node-n/bin:\$PATH\"" >> "$HOME/.profile"
source $HOME/.profile
npm i -g n
  # Si l'installation ce passe bien tous est ok
n lts
sudo apt purge nodejs npm
  # Supprime la version de base de node et npm utilisé pour ne garder que celle de n
  # Il est possible qu'une déco/reco ne soit nécessaire pour que node et npm fonctionne après la désintallation
# Java (automatique)
sudo apt install -y openjdk-<version>-(jre|jdk)[-headless]
  # exemple : sudo apt install -y openjdk-17-jre-headless
  # exemple : sudo apt install -y openjdk-11-jdk
java -version
  # Si tous est ok affiche la version de Java
# Java (manuelle)
cd /opt
sudo wget <URL java> 
  # exemple : wget https://github.com/adoptium/temurin11-binaries/releases/download/jdk-11.0.13%2B8/OpenJDK11U-jdk_x64_linux_hotspot_11.0.13_8.tar.gz
sudo tar xzf <fichier-telecharger>
  # exemple : OpenJDK11U-jdk_x64_linux_hotspot_11.0.13_8.tar.gz
sudo rm <fichier-telecharger>
sudo ln -s $PWD/<dossier-extrait>/bin/* /usr/local/bin/.
  # exemple : ln -s $PWD/jdk-11.0.13+8/bin $HOME/bin/java
java -version
  # Si tous est ok affiche la version de Java

```

#### Plus d'options sur Docker

Gestion des redémarrage : https://docs.docker.com/config/containers/start-containers-automatically/

Mise à jour automatique : https://containrrr.dev/watchtower/

#### Gestion multi terminaux

Screen permet d'ouvrir plusieurs terminaux dans une seule console, permet notamment de lancer un serveur de le détacher et d'y retourner simplement plus tard pour saisir des commandes ou voir les logs : https://doc.ubuntu-fr.org/screen

## Scripts personalisés & Alias

Ajout de scripts personnalisés pouvant être utilisé comme des commandes n'importes où sur le serveur

### Mise en place

```bash
# Création d'un dossier script
mkdir $HOME/script
# Ajoute les lignes à la fin du fichier .profile
echo "" >> "$HOME/.profile"
echo "# Custom env" >> "$HOME/.profile"
echo "export PATH=\"\$HOME/script:\$PATH\"" >> "$HOME/.profile"
# Création d'un fichier d'alias
touch $HOME/.bash_aliases
# Remplissage du fichier 
echo "# Alias" >> $HOME/.bash_aliases
echo "alias ll='ls -laF'" >> $HOME/.bash_aliases
echo "alias la='ls -lA'" >> $HOME/.bash_aliases
echo "alias l='ls -CF'" >> $HOME/.bash_aliases
# Recharge les fichiers
source $HOME/.profile
source $HOME/.bashrc
```

### Ajout d'un nouveau script

Il suffit d'ajouter un fichier exécutable dans le dossier `$HOME/script`. La 1er ligne doit être indiquant avec quel programme exécuter le script avec le format suivant `#!/path/to/exec`. Voici quelques exemples (:warning: les chemins ne sont pas forcément les mêmes)

- Fichier bash (.sh) : `#!/bin/bash`
- Fichier PHP (.php) : `!#/usr/bin/php`
- Fichier Node (.js) : `!#/usr/bin/node`

Il suffit ensuite de taper le nom du fichier dans le CLI pour exécuter de n'importe avec le bonne interpréteur (l'extension n'est pas obligatoire à la fin du fichier). Plusieurs script sont déjà prêt à l'emploi dans le dossier [script](./script).

### Ajout de nouveaux alias

Il suffit d'ouvrir le fichier `$HOME/.bash_aliases` et d'ajouter de nouveaux alias de la façon suivante

```bash
alias <nom-alias>='<cmd-a-executer>'
```

Une fois ajouté pour en bénéficier tout de suite exécuter 

```bash
source $HOME/.bash_aliases
```

## Conseil de sécurité

- [Digital Ocean](https://www.digitalocean.com/community/tutorials/recommended-security-measures-to-protect-your-servers?utm_source=Customerio&utm_medium=Email_Internal&utm_campaign=Email_UbuntuDistroNginxWelcome&mkt_tok=eyJpIjoiTXpVeVlXSTNOemcyTldaayIsInQiOiJuWitnaUFuTkttY1ZGRXlSZGRiNG5ONDJFV21DSXVPQ3RiMGYzR2s1UXZHc1Eza1k5dTBBWmtRT3dxNDVBVU9RbFVkb1o4aVF4TzdSWHpCQ01ybWhpS3E5VFdUWXlnNm90bGYwVmxDTHRzYWhSUTBENlU2VGEyakw0WUphVmZDbCJ9)

## Mise à jour de l'OS (Ubuntu)

```bash
# Affiche la version actuel de l'OS
lsb_release -a

# Pour le mettre à jour, il faut dans un premier temps mettre à jour tous les paquets
sudo apt update
sudo apt upgrade
sudo apt dist-upgrade

# Ouvrir le port 1022 pour accèder au SSH de secours
sudo ufw allow 1022
sudo ufw reload

# Ensuite mettre à jour l'OS
sudo do-release-upgrade
  # Attention si des fichiers de config ont été modifié refuser le redemmarage automatique et bien penser à les remodifier
  # Une fois les fichiers modifiés exécuter la commande pour redemmarer
  sudo shutdown -r now

# Désactiver le port 1022 ouvert pour le SSH de secours
sudo ufw delete allow 1022
sudo ufw reload
```

