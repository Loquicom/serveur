## Installation de logiciels et de langages de programmation

Ce document résume les étapes d'installations de certains logiciels utiles à avoir, ainsi que l'installation de plusieurs langages de programmation pour pouvoir compiler et exécuter du code sur le serveur. Il est possible de n'installer que certains d'entre eux.

### Logiciel

Divers logiciel utilitaire pratique sur un serveur. Il est possible d'en installer plusieurs d'un coup en une seul commande (par exemple `sudo apt install -y git unzip`).

#### Git

Gestionnaire de version le plus couramment utilisé. Pour plus d'explication sur comment l'utiliser aller voir un cour dédié (par exemple https://learngitbranching.js.org/?locale=fr_FR)

```bash
# Installation
sudo apt install -y git
```

#### Unzip

Unzip est un utilitaire qui permet de dézipper des fichiers compressés

```bash
# Installation
sudo apt install -y unzip
# Utilisation (sans le -d dézip dans un dossier avec le même nom que le zip)
unzip file.zip [-d destination_folder]
```

#### Screen

Screen permet d'ouvrir plusieurs terminaux dans une seule console, permet notamment de lancer un serveur de le détacher et d'y retourner simplement plus tard pour saisir des commandes ou voir les logs : https://doc.ubuntu-fr.org/screen

```bash
# Installation
sudo apt install -y screen
# TODO faire exemple d'utilisation
```

### Langages de programmation

Voici les procèdures pour installer et configurer certains des principaux langages de programmation, pour pouvoir les utiliser sur le serveur

### PHP

Installation de PHP avec plusieurs des extension les plus utilisées ainsi que de de Composer

```bash
# Installation
sudo apt install -y php
# Vérification
php -v
  # Si tous est ok, affiche la version de PHP on peut installer les modules complémentaires
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
```

#### Node

Installation et configuration de NodeJs, de NPM son gestionnaire de paquet et de n pour gérer les versions

```bash
# Installation
sudo apt install -y php nodejs npm
# Configuration
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
```

https://docs.npmjs.com/resolving-eacces-permissions-errors-when-installing-packages-globally

#### Java

Il est possible d'installer Java de deux façon, une automatique via le gestionnaire de paquet ou en l'installant manuellement la JDK de son choix.

***Automatique***

```bash
# Installer soit l'Open JDK, soit Adoptium mais pas les deux
# Open JDK
sudo apt install -y openjdk-<version>-(jre|jdk)[-headless]
  # exemple : sudo apt install -y openjdk-17-jre-headless
  # exemple : sudo apt install -y openjdk-11-jdk
# Adoptium (Temurin JDK par Eclipse)
sudo apt install -y wget apt-transport-https gpg
wget -qO - https://packages.adoptium.net/artifactory/api/gpg/key/public | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/adoptium.gpg > /dev/null
echo "deb https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | sudo tee /etc/apt/sources.list.d/adoptium.list
sudo apt update
sudo apt install -y temurin-<version>-jdk
  # exemple : sudo apt install -y temurin-17-jdk
# Vérification
java -version
  # Si tous est ok affiche la version de Java
```

https://adoptium.net/fr/installation/

***Manuelle***

```bash

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

#### Go

Il est possible d'installer Go de deux façon, une automatique via le gestionnaire de paquet ou en l'installant manuellement. L'installation par gestionnaire de paquet peut avoir des limitation sur la version installée, la méthode manuelle est conseillée. Une fois installer il est aussi possible d'installer d'autre version à partir de go lui même.

***Automatique***

```bash
# Installation
sudo apt install -y golang-go
# Configuration
echo "" >> "$HOME/.profile"
echo "# Go env" >> "$HOME/.profile"
echo "export PATH=\"\$HOME/go/bin:\$PATH\"" >> $HOME/.profile
source $HOME/.profile
# Vérification
go version
```

***Manuelle***

Dans cette version il faut récupérer le bon lien de téléchargement sur le site de Go (https://go.dev/dl/) 

```bash
# Installation
sudo apt install -y wget
wget <lien> -O go.tar.gz
  # exemple : wget https://go.dev/dl/go1.21.4.linux-amd64.tar.gz -O go.tar.gz
sudo tar -xzvf go.tar.gz -C /opt
# Configuration
echo "" >> "$HOME/.profile"
echo "# Go env" >> "$HOME/.profile"
echo "export PATH=\"\$HOME/go/bin:/opt/go/bin:\$PATH\"" >> $HOME/.profile
source $HOME/.profile
# Vérification
go version
```

https://www.cherryservers.com/blog/install-go-ubuntu

***Installer d'autre versions***

```bash
# Choix de la version
go install golang.org/dl/go<version>@latest 
  # exemple : go install golang.org/dl/go1.20@latest
# Installation de la version
go<version>
  #exemple : go1.20 download
# Verification
export PATH=/home/ubuntu/sdk/go<version>/bin:$PATH go version 
  # exemple : export PATH=/home/ubuntu/sdk/go1.20/bin:$PATH go version 
```

https://www.ovhcloud.com/en/community/tutorials/how-to-install-go-ubuntu/

#### Python

TODO un jour