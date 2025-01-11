# Caddy

Caddy est un serveur web moderne et simple à configurer avec la gestion des certificats SSL pour l'HTTPS intégrés et par défaut en HTTP/2. Il à pour avantage de pouvoir être configurer par JSON ou par des fichiers de configurations (nommé `Caddyfile`) et de pouvoir passer par une API pour automatiser le processus au lieu des lignes de commandes. Cette documentation ne s'attarderas pas sur l'utilisation de l'API et n'utiliseras que les fichiers de configuration. Il est possible de trouver un comparatif entre les différents mode de configuration à [cette adresse](https://caddyserver.com/docs/getting-started#json-vs-caddyfile).

## Installation

Il y a deux possibilité pour installer Caddy, passer par le gestionnaire de paquet ou l'installer manuellement. Passer par le gestionnaire est plus simple et rapide mais permet uniquement d'installer Caddy de base sans extension. Si une extension est nécessaire il faut passer par l'installation manuelle car elle doivent être ajoutée au moment de la compilation

### Par gestionnaire de paquet (Automatique)

```bash
# Installe les paquets nécessaire pour ajouter le dépot de Caddy à la liste des dépot de l'OS
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
# Ajout du dépot
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
# Installation de Caddy
sudo apt update
sudo apt install caddy
```

Une fois installer Caddy seras disponible en tant que service (via systemd) et les fichiers de config seront dans `/etc/caddy`. N'hésiter pas à regarder la configuration utiliser dans cette [documentation via la partie dédié dans l'installation manuelle](#création-des-fichiers-de-configuration)

### En compilant (Manuelle)

Pour compiler Caddy il nous faut l'outil développer pour faciliter le processus `xcaddy`, ainsi que Go installé en version 1.21 ou supérieur (pour installer Go vous référer à la [documentation dédié](../initialisation/logiciel.md#go)). Il existe deux façon de récupérer xcaddy soit depuis le gestionnaire de paquet soit en le compilant depuis ses sources. 

```bash
# Gestionnaire de paquet
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/xcaddy/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-xcaddy-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/xcaddy/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-xcaddy.list
sudo apt update
sudo apt install xcaddy
# Compilation
go install github.com/caddyserver/xcaddy/cmd/xcaddy@latest
  # Dans ce cas l'executable est présent dans le dossier $HOME/go/bin
  # Si la configuration de Go a été faite en suivant ce document alors l'appli est déjà disponible dans le PATH
```

Une fois xcaddy installé il suffit de lancer une commande pour créer l'exécutable de Caddy

```bash
# Création et déplacement dans le dossier d'installation
sudo mkdir /opt/caddy
sudo chown $USER:$USER /opt/caddy
cd /opt/caddy
# Compilation de caddy
xcaddy build
# Ajout de Caddy dans le PATH
echo "" >> "$HOME/.profile"
echo "# Caddy env" >> "$HOME/.profile"
echo "export PATH=\"/opt/caddy:\$PATH\"" >> "$HOME/.profile"
source $HOME/.profile
# Test de caddy
cd $HOME
caddy version
```

Dans l'exemple ci-dessus on compile un Caddy standard sans extension, pour ajouter il suffit d'utiliser le format suivant

```bash
xcaddy build \
    --with github.com/caddyserver/nginx-adapter
	--with github.com/caddyserver/ntlm-transport@v0.1.1
```

Une fois l'exécutable de Caddy créer il suffit de lancer pour avoir un serveur web. Cependant pour des raisons pratique (et pour être iso avec l'installation automatique par gestionnaire de paquet) il reste encore quelques étapes pour le rendre vraiment fonctionnel en tant que serveur web.

#### Création d'un utilisateur dédiée

Dans un premier temps nous allons créer un utilisateur et un groupe pout tout ce qui touche à Caddy

```bash
# Création du groupe
sudo groupadd --system caddy
# Création de l'utilisateur
sudo useradd --system \
    --gid caddy \
    --create-home \
    --home-dir /var/lib/caddy \
    --shell /usr/sbin/nologin \
    --comment "Caddy web server" \
    caddy
# Ajout de l'utilisateur courrant dans le groupe
sudo usermod -aG caddy $USER
```

#### Création des fichiers de configuration

Ensuite nous allons créer les fichiers de configuration de Caddy

```bash
# Création des dossier
sudo mkdir /etc/caddy
sudo mkdir /etc/caddy/config
sudo mkdir /etc/caddy/sites
# Création des fichiers
sudo touch /etc/caddy/Caddyfile
sudo touch /etc/caddy/config/security.conf
sudo touch /etc/caddy/sites/exemple.com.conf
# Gestion des logs
sudo mkdir /var/log/caddy
sudo chown caddy:caddy /var/log/caddy
```

Ouvrir chaque fichier de configuration et copier/coller le contenue dedans

```properties
# /etc/caddy/Caddyfile
{  
    # Décommenter la ligne si besoins de logger plus d'informations
    # debug
    
    log default {  
       	format console  
       	output file /var/log/caddy/system.log  
       	exclude http.log.access  
    }
    
    # Configuration certificat SSL
    # email admin@email.com
    
    # Activer ces lignes pour un serveur en local qui n'a pas accès à internet
    # local_certs
    # auto_https disable_redirects
}

import config/*  
import sites/*

:443, :80 {
    # Catcher final qui refuse l'accès si aucun site ne correspond
	respond "Access denied" 403 {
		close
	}
}

# /etc/caddy/config/security.conf
(security-header) {
	header {
    	Strict-Transport-Security "max-age=63072000; includeSubdomains; preload"
    	X-Frame-Options "SAMEORIGIN"
    	X-Content-Type-Options "nosniff"
    	Referrer-Policy "strict-origin-when-cross-origin"
    	# Changer My-Server par la valeur de votre choix
    	Server My-Server
	}
}

(disabled-dot-files) {
	@dotFiles {  
      	path */.*  
      	not path /.well-known/*  
    }
    
    respond @dotFiles 403 {
    	close
    }
}

# /etc/caddy/sites/exemple.com.conf
localhost {
	import security-header
	import disabled-dot-files
	
	# Définit la racine du site si besoins
	# root * /var/www/example.com
	
	# Set log file
	log {
        output file /var/log/caddy/example.access.log
        format console
    }

    # Encode la réponse si le navigateur le permet
    encode zstd gzip
	
	# Envoie une réponse
	respond "Hello, world!"
	# Si la racine commenter la ligne du dessus et décommenter celle du dessous pour servir les fichiers présents
	# file_server
}
```

Dans le fichier d'exemple il est possible de remplacer `localhost {` par `localhost, <ip-machine> {` (en remplacent \<ip-machine> par l'ip de votre machine) si vous appeler Caddy depuis un autre ordinateur. Il est aussi possible de commenter la ligne `import security-header` si la machine n'est pas exposé sur le web

#### Création d'un service

L'objectif est de créer un service pour permettre la gestion de Caddy et notamment son redémarrage après un reboot par exemple. L'autre avantage est de pouvoir lancer Caddy sans ce soucier des arguments à chaque fois ce qui est utile notamment car nous allons mettre les fichiers de config dans `/etc/caddy`.

Nous allons dans un premier temps nous allons créer le fichier de service :

```bash
sudo touch /etc/systemd/system/caddy.service
sudo nano /etc/systemd/system/caddy.service
```

Puis il faut copier le contenue suivant dans le fichier (le fichier d'origine est disponible sur le [lien suivant](https://github.com/caddyserver/dist/blob/master/init/caddy.service)). Tous les fichiers (services et config) sont aussi trouvable dans le dossier `app` de ce dépôt.

```properties
# caddy.service
#
# For using Caddy with a config file.
#
# WARNING: This service does not use the --resume flag, so if you
# use the API to make changes, they will be overwritten by the
# Caddyfile next time the service is restarted. If you intend to
# use Caddy's API to configure it, add the --resume flag to the
# `caddy run` command or use the caddy-api.service file instead.

[Unit]
Description=Caddy
Documentation=https://caddyserver.com/docs/
After=network.target network-online.target
Requires=network-online.target

[Service]
Type=notify
User=caddy
Group=caddy
ExecStart=/opt/caddy/caddy run --environ --config /etc/caddy/Caddyfile
ExecReload=/opt/caddy/caddy reload --config /etc/caddy/Caddyfile --force
TimeoutStopSec=5s
LimitNOFILE=1048576
PrivateTmp=true
ProtectSystem=full
AmbientCapabilities=CAP_NET_ADMIN CAP_NET_BIND_SERVICE

[Install]
WantedBy=multi-user.target
```

Maintenant que tout est bon on peut activer le service

```bash
# Lancer le service
sudo systemctl daemon-reload
sudo systemctl enable --now caddy
# Vérifier l'état de Caddy
systemctl status caddy
```

Voila le serveur tourne sur la machine et un Hello World est disponible sur le localhost

#### Mise à jour

Pour mettre à jour il existe deux solutions, recompiler ou passer par l'exécutable (expérimental)

```bash
# Stopper le serveur
sudo systemctl stop caddy

# Recompiler
cd /opt/caddy
xcaddy build
  # Adapter la commannde de build à vos besoins comme indiqué plus haut
# Via l'exécutable (expérimental)
caddy upgrade

# Redémmarer le serveur
sudo systemctl start caddy
```

### Post installation

#### Ouverture du pare-feu

Si vous avez besoins d'appeler le serveur depuis une autre machine, il faut penser à ouvrir le pare-feu

```bash
sudo ufw allow 80
sudo ufw allow 443
sudo ufw reload
# Vérifier l'état du pare-feu
sudo ufw status verbose
```

#### Définir Caddy comme propriétaire du dossier contenant les sites

Pour éviter les problèmes de lecture et d'écriture il est conseillé de donner les droits sur le dossier contenant les sites à Caddy

```bash
sudo chown caddy:caddy -R /var/www
```

#### Ajout du certificat local dans le truststore

Lors du premier lancement Caddy vous demanderas le mot passe de votre compte pour installer son certificat dans le truststore de la machine qui le fait tourner. Si vous souhaitez installer le certificat sur une autre machine ou un autre navigateur il est disponible à l'endroit suivant : `/var/lib/caddy/.local/share/caddy/pki/authorities/local/root.crt`.

De plus si votre navigateur bloque l'accès au site pour cause d'erreur de certificat SSL il est possible de quand même continuer en tapant `thisisunsafe` sur la page.

#### Commandes du service

```bash
# Démarrer le serveur
sudo systemctl start caddy
# Afficher le status du serveur
sudo systemctl status caddy
# Recharger la configuration du serveur (après l'avoir modifiée)
sudo systemctl reload caddy
# Stopper le serveur
sudo systemctl stop caddy
# Voir les logs
journalctl -u caddy --no-pager | less +G
```

## Configurer la gestion des certificats SSL

La gestion des certificats SSL fonctionne par défaut sans rien paramétrer, mais il est tout de même recommandé d'ajouter son email dans le Caddyfile. Pour plus d'information sur tous les paramètrege possible ce référer à la [documentation officielle](https://caddyserver.com/docs/caddyfile/options#tls-options)

## Ajouter un site

Pour ajouter un nouveau site rien de plus simple il suffit de copier/coller le fichier d'exemple dans `/etc/caddy/sites/` et de le modifier pour en faire ce que l'on souhaite (voir par exemple les configuration ci-dessous).

Si le site à besoins d'un répertoire il faut aussi le créer, par exemple :

```bash
sudo mkdir /var/www/exemple-com
sudo chown cady:cady /var/www/exemple-com
```

Une fois fait il faut recharger Caddy avec la commande suivante :

```bash
sudo systemctl reload caddy
```

## Configurer un site

Tous les exemples de configuration des différent site prend pour URL `exemple.com` et pour racine `/var/wwww/exemple.com`, il faut évidemment adapter ces valeurs à vos besoins.

### Site statique

Pour un site statique il suffit d'utiliser la directive file_server et si on ajoute browse après un navigateur de fichier est disponible s'il n'y a aucun fichier index.html

```properties
example.com {
	# Import des éléments de sécurité
	import security-header
	import disabled-dot-files

    root * /var/www/example.com

    log {
        output file /var/log/caddy/example.access.log
        format console
    }

    # Compression du résultat
    encode zstd gzip
    
    # Envoie des fichiers statiques (HTML, ...)
    file_server browse
}
```

### Site PHP

Pour faire avoir un site qui fait tourner du PHP avec Caddy il faut avoir PHP-FPM (PHP basique n'est pas compatible), voir la [section dédiée](#installation-php-fpm) pour son installation.

Caddy dispose d'une directive spécifique pour PHP qui est équivalent à un ensemble de directive standard pour alléger le fichier de config. Si la configuration par défaut de la directive PHP ne vous convient pas il es possible de passer par la version détaillé pour avoir un plus grand contrôle. La description complété de la directive et sa version détaillé est disponible sur [ce lien](https://caddyserver.com/docs/caddyfile/directives/php_fastcgi#php-fastcgi).

```properties
example.com {
	# Import des éléments de sécurité
	import security-header
	import disabled-dot-files

    root * /var/www/example.com

    log {
        output file /var/log/caddy/example.access.log
        format console
    }

    # Compression du résultat
    encode zstd gzip

    # Utilisation de PHP-FPM (à modifier en fonction de votre configuration)
    php_fastcgi unix//run/php/php-fpm.sock {
    	# Permet d'appeler le fichier sans l'extension
    	try_files {path} {path}.php {path}/index.php index.php
    }
    
    # Envoie des fichiers statiques (HTML, ...)
    file_server
}
```

#### Installation PHP-FPM

Si PHP-FPM n'est pas présent l'installer est simple avec la commande suivante 

```bash
# Remplacer X.Y par la version de PHP souhaitée
sudo apt-get install -y phpX.Y-fpm
# Il est même possible de désintaller la version de base de PHP
sudo apt-get purge phpX.Y
# Ensuite il faut vérifier le fichier de config de PHP-FPM
sudo nano /etc/php/X.Y/fpm/pool.d/www.conf
  # Modifier la valeur : user = caddy
  # Modifier la valeur : group = caddy
  # Modifier la valeur : listen.owner = caddy
  # Modifier la valeur : listen.group = caddy
  # Copier la valeur de listen pour l'utiliser dans les fichiers de conf des sites PHP
# Rédammarer le service PHP-FPM
sudo systemctl restart phpX.Y-fpm.service
```

Il est aussi possible de créer un config pour PHP dans caddy et limiter encore plus le nombre de lignes à écrire

```bash
sudo touch /etc/caddy/config/php.conf
sudo nano /etc/caddy/config/php.conf
```

Et de mettre le contenue suivant en faisant attention à bien remplacer le chemin de la socket par celui dans le fichier de conf de PHP

```properties
(php) {
	php_fastcgi unix//run/php/php-fpm.sock {
    	try_files {path} {path}.php {path}/index.php index.php
    }
}
```

Une fois fait il suffit de remplacer tout le bloc `php_fastcgi` par `import php`.

### Reverse proxy

La configuration d'un reverse proxy est simple et ce fait en une ligne, il suffit de prendre l'exemple ci-dessous et de changer l'URL d'arriver

```properties
example.com {
	# Import des éléments de sécurité
	import security-header

    log {
        output file /var/log/caddy/example.access.log
        format console
    }

    # Compression du résultat
    encode zstd gzip

    # Reverse proxy
    reverse_proxy localhost:9005
}
```

Pour faire du load-balancing il suffit de mettre plusieurs URL à la suite, par exemple `reverse_proxy node1:80 node2:80 node3:80`

### Webdav

TODO

- https://github.com/mholt/caddy-webdav
- https://marko.euptera.com/posts/caddy-webdav.html

## Authentification basique

Pour ajouter une authentification basique par HTTP il faut dans un premier temps hasher les mots de passe des utilisateur, pour faire cela caddy dispose d'un outil

```bash
caddy hash-password
  # Entrer le mot de passe
  # Re-entrer le mot de passe
# Affichage du hash
```

Copier le hash de chaque utilisateur dans un fichier le temps de l'ajouter dans le fichier de configuration du site. Dans le fichier de configuration il faut ajouter le block suivant avant les directives qui servent du contenue :

```properties
example.com {
	# ...

    # Authentification basique
    basic_auth {
		# Username "Bob", password "hiccup"
		Bob $2a$14$Zkx19XLiW6VYouLHR5NmfOFU0z2GTNmpkT/5qqR7hx4IjWJPDhjvG
		# ... Autre utilisateurs
	}
    
    # ...
}
```

Il est aussi possible de mette l'authentification uniquement sur un sous dossier avec par exemple ici un dossier secret : `basic_auth /secret/* {`.

## Liens

- https://caddyserver.com/docs/caddyfile/directives/basic_auth
- https://caddyserver.com/docs/running#linux-service
- https://php.watch/articles/caddy-php
- https://caddyserver.com/docs/running
- https://caddyserver.com/docs/caddyfile-tutorial
- https://caddyserver.com/docs/quick-starts/reverse-proxy#reverse-proxy-quick-start



