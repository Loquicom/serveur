# Installation d'un serveur LAMP / LAPP

Ce document décrit la procédure d'installation et de sécurisation d'un serveur Apache & PHP sur Linux avec MySQL/MariaDB (LAMP) ou PostgreSQL (LAPP). Il part du principe que tous les composant de base sont déjà installé sur le serveur.

## Apache

https://www.it-connect.fr/installer-un-serveur-lamp-linux-apache-mariadb-php-sous-debian-11/

https://www.digitalocean.com/community/tutorials/how-to-install-linux-apache-mysql-php-lamp-stack-on-ubuntu-20-04-fr

https://www.digitalocean.com/community/tutorials/how-to-set-up-apache-virtual-hosts-on-ubuntu-16-04

### Installation

```bash
# Installation d'Apache
sudo apt-get install -y apache2 apache2-utils libapache2-mod-security2
# Démarrage automatique
sudo systemctl enable apache2
# Paramétrage du parefeu
sudo ufw allow in "Apache Full"
sudo ufw reload
# Activation modules (a2dismod pour désactiver)
sudo a2enmod rewrite
sudo a2enmod deflate
sudo a2enmod headers
sudo a2enmod ssl
sudo a2enmod proxy
sudo a2enmod proxy_http
sudo a2enmod security2
# Permet la lecture/modification du réportoire www à l'utilisateur apache et du serveur
sudo chown -R www-data:www-data /var/www
sudo chmod -R 775 /var/www
sudo usermod -a -G www-data <user>
# Redémmarer le serveur
sudo systemctl restart apache2
```

### Utilisation de HTTP/2

Modification de la configuration pour activer HTTP/2, paramétrage de PHP-FPM et l'utilisation avec un reverse proxy

#### Mise en place

https://www.howtoforge.com/how-to-enable-http-2-in-apache/

https://medium.com/tech-learnings/how-to-configure-apache-reverse-proxy-with-http-2-ebc87c1c7bcb

```bash
# Installation de PHP FPM (PHP basique n'est pas compatible)
sudo apt-get install phpX.Y-fpm
# Si une version de base de PHP est déjà installer il faut la désactiver
sudo a2dismod phpX.Y
  # il est même possible de la désintaller
  sudo apt-get purge phpX.Y
# Activation de PHP FPM avec Apache 2
sudo a2enconf phpX.Y-fpm
sudo a2enmod proxy_fcgi
# Désactivation du Prefork MPM (qui n'est pas compatible) et activation du Event MPM
sudo a2dismod mpm_prefork
sudo a2enmod mpm_event
# Activation des modules nécessaires
sudo a2enmod ssl
sudo a2enmod http2
# Redémmarage
sudo systemctl restart apache2
```

A ce stade il est possible que tous les sites soit en HTTP/2 sinon il faut ajouter cette ligne dans les fichiers vhost pour indiquer d'utiliser HTTP/2

```htaccess
<VirtualHost *:443>
  ...
  Protocols h2 h2c http/1.1
</VirtualHost>
```

Dans le cas d'un reverse proxy il faut modifier ces lignes dans le fichier du vhost

```htaccess
<VirtualHost *:443>
	...
 	ProxyPass / h2://ip:port/
 	ProxyPassReverse / http://ip:port/
 	...
 	Protocols h2 h2c http/1.1
</VirtualHost>
```

#### PHP-FPM

**Utilisation php_value**

https://maxchadwick.xyz/blog/setting-a-php-value-in-php-fpm

Si dans les .htacces il y a (par exemple)

```htaccess
php_value include_path ".:/php_lib"
php_value auto_prepend_file "autoload.php"
```

Cette forme ne marche plus il y a trois possibilité :

- Créer un fichier `.user.ini` et mettre dedans sous forme clef valeur (exemple : `include_path = .:/php_lib`)
- Modifier le fichier de conf principal `www.conf` et ajouter dans le tableau php_value la valeur (exemple : `php_value[include_path] = .:/php_lib`)
- Modifier le fichier .htaccess avec SetEnv (exemple : SetEnv PHP_VALUE "include_path = .:/php_lib"). Attention avec cette méthode seul une valeur peut être attibuée

**Fichier PHP non interprété**

https://serverfault.com/questions/785782/apache-with-php-fpm-php-doesnt-execute

https://cwiki.apache.org/confluence/display/httpd/PHP-FPM

Si Apache exécute pas les fichiers PHP il est possible d'ajouter une config dans le vhost

```htaccess
<VirtualHost *:80>
    DocumentRoot "/var/www/html/example.com/"
    ServerName example.com

    # Setup php-fpm to process php files
    ProxyPassMatch ^/(.*\.php(/.*)?)$ fcgi://127.0.0.1:9000/var/www/html/example.com/$1
    DirectoryIndex /index.php index.php
</VirtualHost>
```

Attention vérifié que php-fpm est bien démarré et sur le port 9000 (si non modifié)

```bash
# Vérifier le statut de php-fpm
sudo systemctl status phpX.Y-fpm
# Pour voir si le port 9000 est utilisé
sudo netstat -tulpn | grep :8999
```

### Paramétrage de sécurité

```bash
# Ajoute des paramètres de sécurité
sudo nano /etc/apache2/conf-enabled/security.conf
  # Changer le nom et la version du serveur
    # Mettre la valeur de ServerTokens à Os
    # Mettre la valeur de ServerSignature à On
    # Mettre la valeur de TraceEnable à Off
    # Ajouter SecServerSignature avec la valeur qui seras affichées pour le nom du serveur
  # Ajoute des Header pour la sécurité
    # Ajouter (ou décommenter) Header always set Strict-Transport-Security "max-age=63072000; includeSubdomains; preload"
    # Ajouter (ou décommenter) Header always set X-Frame-Options "SAMEORIGIN"
    # Ajouter (ou décommenter) Header always set X-Content-Type-Options "nosniff"
    # Ajouter (ou décommenter) Header always set Referrer-Policy "strict-origin-when-cross-origin"
  # Bloque l'accès au .git
    # Ajouter (ou décommenter) <DirectoryMatch "/\.git">Require all denied</DirectoryMatch>
# Modification des paramètres pour le dossier contenant les sites
sudo nano /etc/apache2/apache2.conf
  # Sous la ligne <Directory /var/www/> les options doivent être :
    # Options -Indexes -FollowSymLinks +SymLinksIfOwnerMatch
    # AllowOverride All
    # Require all granted
# Bloquer les connexions via l'IP
sudo nano /etc/apache2/sites-available/000-default.conf
  # Commenter la ligner DocumentRoot /var/www/html (ajouter un  "devant")
  # Ajouter les lignes suivantes en dessous de la ligne commentées
    # <Location />
    #     Require all denied
    # </Location>
# Désactiver les sites/vhost non utilisés
sudo a2dissite default-ssl.conf
# Redémmarer le serveur
sudo systemctl restart apache2
```

### Ajouter un Virtual Host

```bash
# Création d'un dossier et attribution des droits
sudo mkdir -p /var/www/<example.com>
sudo chown -R www-data:www-data /var/www/<example.com>
sudo chmod -R 775 /var/www/<example.com>
# Copie et édition du fichier de conf par défaut
sudo cp /etc/apache2/sites-available/000-default.conf /etc/apache2/sites-available/<example.com>.conf
sudo nano /etc/apache2/sites-available/<example.com>.conf
  # Modifier ServerAdmin avec un email valide
  # Modifier DocumentRoot par /var/www/<example.com>
  # Ajouter ServerName <example.com>
  # Possibilité d'ajouter des alias avec ServerAlias www.<example.com>
# Activer le vhost
sudo a2ensite <example.com>.conf
# Redémmarer apache
sudo systemctl restart apache2
```

### Configuration Virtual Host

Ensemble de configuration possible (non exhaustive) de fichier Virtual Host. Dans tous les cas il faut ajouter les ligne dans la balise `<VirtualHost>`

#### Reverse proxy

https://www.vincentliefooghe.net/content/apache-reverse-proxy-https

https://www.digitalocean.com/community/tutorials/how-to-use-apache-as-a-reverse-proxy-with-mod_proxy-on-ubuntu-16-04

https://medium.com/@ppranav164/how-proxypass-proxypassreverse-works-in-the-apache-server-11177fb68b4c

```htaccess
ProxyRequests Off
# Permet de garder le nom DNS du reverse proxy, sinon retourne celui du serveur distant
ProxyPreserveHost On
ProxyPass / http://localhost:<port>/
ProxyPassReverse / https://localhost:<port>/
 
# === Configuration optionnelles === #

# Si le serveur que l'on appel (derrère le proxy) est en HTTPS
SSLProxyEngine On 
# Pour ne pas vérifier le certificat, le nom et le serveur distant
SSLProxyCheckPeerCN Off
SSLProxyCheckPeerName Off
SSLProxyVerify none
```

#### Reverse proxy & Load balancing

```htaccess
<Proxy balancer://mycluster>
    BalancerMember http://127.0.0.1:8080
    BalancerMember http://127.0.0.1:8081
</Proxy>

ProxyRequests Off
ProxyPreserveHost On

ProxyPass / balancer://mycluster/
ProxyPassReverse / balancer://mycluster/
```

#### Sous site

```htaccess
# Le 1er paramétre correspond au chemin dans l'url
  # Exemple /dev sur site.com => site.com/dev
# Le 2nd paramétre est le dossier du site
Alias /<URL> /var/www/public
```

#### Sous site & Reverse proxy

```htaccess
# Le 1er paramétre correspond au chemin dans l'url
  # Exemple /dev sur site.com => site.com/dev
# Le 2nd paramétre est l'adresse du site
ProxyRequests Off
ProxyPreserveHost On
ProxyPass /<URL>/ http://localhost:<port>/
ProxyPassReverse /<URL>/ http://localhost:<port>/

```

#### Bloquer l'accès quand connexion via une URL précise

```htaccess
<VirtualHost *:80>
    ServerName <dev.site.com>
    <Location />
        Require all denied
    </Location>
</VirtualHost>
```

## SGBD

Procédure d'installation de différents SGBD. Les sources de cette partie pour MySQL et MariaDB sont les même que celles pour la section précédente apache.

### MySQL

```bash
# Installation du SGBD
sudo apt-get install -y mysql-server
# Paramétrage du serveur
sudo mysql_secure_installation
  # Ne pas activer le plugin de validation de mot de passe (n)
  # Ne pas activer l'authentification par unix_socket (n) -> Si activer seul les comptes unix avec privilèges root présent sur le serveur peuvent se connecter
  # Définir le mot de passe de root
  # Supprimer l'utilisateur anonyme (y)
  # Désactiver la connexion à distance avec le compte root (y)
  # Supprimer la base de test (y)
  # Recharger les privilèges (y)
# Connexion au SGBD
sudo mysql -u root -p
  # Si l'authentification par unix_socket est active pas besoins du login et du mot de passe
```

Création d'un utilisateur (à exécuter dans la console SQL du SGBD)

```mysql
# Création d'un utilisateur
CREATE USER '<example_user>'@'%' IDENTIFIED WITH mysql_native_password BY '<password>';
  # Ou (peut poser problème d'authentification avec certain connecteur)
CREATE USER '<example_user>'@'%' IDENTIFIED WITH caching_sha2_authentification BY '<password>';
  # Le % peut être remplacer par une IP/localhost pour limiter la connexion

# Droit sur une BDD
GRANT ALL ON <example_database>.* TO '<example_user>'@'%';
# Ou droit sur toutes les BDD
GRANT ALL PRIVILEGES ON * . * TO '<example_user>'@'%';
FLUSH PRIVILEGES

```

### MariaDB

```bash
# Installation du SGBD
sudo apt-get install -y mariadb-server
# Paramétrage du serveur
sudo mariadb-secure-installation
  # Ne pas activer l'authentification par unix_socket (n) -> Si activer seul les comptes unix avec privilèges root présent sur le serveur peuvent se connecter
  # Définir le mot de passe de root
  # Supprimer l'utilisateur anonyme (y)
  # Désactiver la connexion à distance avec le compte root (y)
  # Supprimer la base de test (y)
  # Recharger les privilèges (y)
# Connexion au SGBD
mariadb -u root -p
  # Si l'authentification par unix_socket est active pas besoins du login et du mot de passe
```

Création d'un utilisateur (à exécuter dans la console SQL du SGBD)

```mariadb
# Création d'un utilisateur
CREATE USER '<example_user>'@'%' IDENTIFIED BY '<password>';
  # Le % peut être remplacer par une IP/localhost pour limiter la connexion
  
# Droit sur une BDD
GRANT ALL ON <example_database>.* TO '<example_user>'@'%';
# Ou droit sur toutes les BDD
GRANT ALL PRIVILEGES ON * . * TO '<example_user>'@'%';
FLUSH PRIVILEGES
```

### PostgreSQL

https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-20-04-fr

https://devstory.net/12255/configurer-postgresql-pour-autoriser-les-connexions-a-distance

```bash
# Installation du SGBD
sudo apt-get install -y postgresql postgresql-contrib
# Création d'un nouvel utilisateur (role)
sudo -u postgres createuser --interactive
  # Suivre les instructions

# Activation de la connexion à distance
sudo nano /etc/postgresql/<ver>/main/postgresql.conf
  # Décommenter et modifier la ligne listen_addresses = "localhost" par listen_addresses = "*"
  # Il est aussi possible de mettre une liste d'IP séparer par des virgules
sudo nano /etc/postgresql/<ver>/main/pg_hba.conf
  # Chercher la ligne sous # IPv4 local connections:
  # Remplacer l'IP par 0.0.0.0/0
  # Si on souhaite autoriser seulement certaine IP copier/coller la ligne et modifier l'IP
# Redémmarer Postgres
sudo systemctl restart postgresql

# Pour la connexion il faut passer par un compte unix avec le même nom que le role sur lequel on veut se connecter
# Au besoins il suffit de créer un compte
sudo adduser <nom-compte>
# Si on est pas sur le bon compte (mais que le compte existe)
sudo -u <nom-compte> psql
# Sinon si on est déjà sur le bon compte
psql

```

Création d'utilisateur, définition de mot de passe et attribution de droit

```sql
# Pour changer le mot de passe d'un compte, exécuter la commande suivante avec un compte superuser
\password <nom-compte>
# Ou en SQL
ALTER USER <user_name> WITH PASSWORD '<new_password>';

# Création d'un utilisateur (sans passer par l'assistant)
CREATE USER <user_name> WITH PASSWORD '<password>'

# Donner les droits sur une base à un utilisateur
GRANT ALL PRIVILEGES ON DATABASE "<database>" to <user_name>;
```

## Sécurisation

Configuration de logiciel et de fonctionnalités d'Apache permettant de protéger l'accès au serveur et des données (présentent sur le serveur ou envoyer par un client).

### Crowdsec

https://www.it-connect.fr/bloquer-les-attaques-sur-son-serveur-web-apache-php-avec-crowdsec/

https://crowdsec.net/blog/tutorial-crowdsec-v1-1/

```bash
# Installation
sudo apt-get install -y crowdsec
# Si le paquet n'est pas dispo faire cette commande avant de refaire celle au dessus
curl -s https://packagecloud.io/install/repositories/crowdsec/crowdsec/script.deb.sh | sudo bash 
# Affiche les service à protéger detecté par Crowdsec
sudo cscli collections list
  # Normalement base-http-scenarios est présent si Apache est installé
```

Crowdsec ne fait que prendre la décision, l'application est faite par des Bouncer

#### MaJ collections

Si dans la liste des collections il y a des :warning:, il faut les mettre à jours

```bash
# Toutes les collections
sudo cscli collections upgrade -a
# Une collection
sudo cscli collections upgrade <collection>
  # sudo cscli collections upgrade crowdsecurity/apache2
  # sudo cscli collections upgrade crowdsecurity/sshd
sudo systemctl reload crowdsec
```

#### Bouncer PHP

Le bouncer PHP analyse les nouvelle IP et si un requête est bloqué une page Web avec l'erreur 403 ou un captcha est affichés. **Attention** ce bouncer ne permet de bloquer que les appels direct vers une page PHP, ainsi si une adresse est par exemple un reverse proxy vers une autre application alors le connexion ne seras pas bloquées par le bouncer (la connexion étant juste envoyée sur l'application derrière le proxy). Dans ce cas plutôt utilisé le bouncer firewall qui bloque les requêtes au moment de la connexion avec la machine qui héberge le serveur et non lors du chargement d'une page.

```bash
cd /opt
sudo git clone https://github.com/crowdsecurity/cs-php-bouncer.git
sudo chown -R <user>:<group> cs-php-bouncer/
cd cs-php-bouncer/
# Installation
./install.sh --apache
  # Ne doit pas être lancé avec le compte root ou sudo
  # Suivre les instructions de l'installateur
# Mettre en proprietaire apache (par défaut l'utilisateur www-data) sur le dossier /usr/local/php/crowdsec
sudo chown www-data /usr/local/php/crowdsec/
# Recharger Apache
sudo systemctl reload apache2
# Afficher la liste des bouncers
sudo cscli bouncers list
  # Le bouncer doit aparaitre
  
# Afficher un captcha à la place du blocage
sudo nano /etc/crowdsec/profiles.yaml
  # C'est le fichier qui gère les décisions en fonction des alertes, la 1er regles valide s'appliqueet stop la lecture dans le fichier
  # Ajouter au début du fichier les lignes suivantes
    # # Bad User agents + Crawlers - Captcha 4H
    # name: crawler_captcha_remediation
    # filters:
    # - Alert.Remediation == true && Alert.GetScenario() in ["crowdsecurity/http-crawl-non_statics", "crowdsecurity/http-bad-user-agent"]
    # decisions:
    # - type: captcha
    # duration: 4h
    # on_success: break
    # ---
# Redémmarer Crowdsec
sudo systemctl restart crowdsec
```

**Attention Configuration à modifier en utilisant HTTP/2**

Le bouncer utilise un fichier de config apache `/etc/apache2/conf-available/crowdsec_apache.conf` qui utilise `Php_value` qui n'existe pas avec l'utilisation des modules HTTP/2.

TODO voir si il est possible de modifier le fichier de configuration pour le rendre de nouveau fonctionnel, sinon utiliser fichier .user.ini sur chaque projet php pour reset le auto_prepend

#### Bouncer Firewall

Le bouncer Firewall ajoute les ip bloqué directement dans le pare-feu pour bloquer leur connexion (ce qui provoquera un timeout)

```bash
# Installation
sudo apt-get install -y crowdsec-firewall-bouncer-iptables
```

Si le bouncer ne fonctionne plus suite à une mise à jour il suffit de le reinstaller

```bash
# Suppression du bouncer
cscli bouncers delete <bouncer-name>
# Désintallation
sudo apt-get purge crowdsec-firewall-bouncer-iptables
# Reinstallation
sudo apt-get install -y crowdsec-firewall-bouncer-iptables
```

#### Commandes utiles Crowdsec

```bash
# Afficher la liste des bouncers
sudo cscli bouncers list
# Affiche les service à protéger detecté par Crowdsec
sudo cscli collections list
# Affiche les décisions prise par Crowdsec
sudo cscli decisions list
# Affiche les alertes détecté par Crowdsec
sudo cscli alerts list

# Ban une IP
sudo cscli decisions add --ip X.X.X.X
# Deban une IP
sudo cscli decisions delete --ip X.X.X.X

# En cas de problème lors de l'installation, il est possible de la relancer
sudo /usr/share/crowdsec/wizard.sh -c
# Il est aussi possible de redétecter les fichiers de logs des servies à protèger
sudo /usr/share/crowdsec/wizard.sh -d
```

#### Tester Crowdsec

Si aucun bouncer n'est présent lors du test, on doit pouvoir voir la décision de crowdsec mais aucun blocage n'aura lieu. Le test doit être fait depuis une autre machine, les test fait depuis la même machine n'activeront rien car le trafic local est whitelisté

````bash
# Nikto (analyse un serveur Web à la recherche de vulnérabilités ou de défaut de configuration)
sudo apt-get install -y nikto
curl -I <site.com>
  # Verification que le serveur est joignable
nikto -h <site.com>
  # Aller vérifier la liste de décision de Crowdsec
curl --connect-timeout 1 http://<site.com>
  # Si un bouncer est en place on peut vérifier que l'ip est bien bloquée
  
# Wapiti (Scanner de vulnérabilités d'application web)
sudo apt-get install -y wapiti
curl -I <site.com>
wapiti -u http://<site.com>
  # Aller vérifier la liste de décision de Crowdsec
  
# Avec des User-agent bloqué (permet de tester le captcha, les autres vont se faire ban quoi qu'il arrive)
curl -I <site.com> -H "User-Agent: Cocolyzebot"
curl -I <site.com> -H "User-Agent: Mozlila"
curl -I <site.com> -H "User-Agent: OpenVAS"
curl -I <site.com> -H "User-Agent: Nikto"
  # Un code d'erreur devrait finir par arriver
````

#### Dashboard

Il existe deux dashboards, le 1er Metabase hébergé en local, le 2nd Crowdsec console qui permet de voir plusieurs instance de Crowdsec sur le même dashboard

**Metabase**

https://docs.crowdsec.net/docs/observability/dashboard/

```bash
# Installe le container docker
sudo cscli dashboard setup -l 0.0.0.0 -p <port>
  # Les identifiants généré lors de l'installation sont stocké dans /etc/crowdsec/metabase/metabase.yaml
# Ajout du rédemmarage auto
docker update --restart unless-stopped crowdsec-metabase  
# Le container est gérable soit par docker soit par cscli, voir l'aide
sudo cscli dashboard -h
# Supprime le dashboard
sudo cscli dashboard remove [-f]
# Stop le dashboard
sudo cscli dashboard stop
# Démmare le dashboard
sudo cscli dashboard start
```

**Crowdsec console**

Actuellement en Beta sur https://app.crowdsec.net

```bash
# Ajouter l'instance Crowdsec sur la console
sudo cscli console enroll <token>
  # Le token est disponible sur la console
sudo systemctl restart crowdsec
```



### HTTPS / SSL

https://certbot.eff.org/instructions?ws=apache&os=ubuntufocal

https://www.digitalocean.com/community/tutorials/how-to-secure-apache-with-let-s-encrypt-on-ubuntu-18-04

https://www.cyberciti.biz/faq/how-to-forcefully-renew-lets-encrypt-certificate/

https://stackoverflow.com/questions/40030537/how-to-stop-renewing-a-letsencrypt-certbot-certificate

https://serverfault.com/questions/1057412/how-to-automate-certbot-certificate-renewal-on-ubuntu-20-04

#### Installation de cerbot

```bash
# Installation via snap (Mise à jour avant installation)
sudo apt-get install snapd
sudo snap install core; sudo snap refresh core
# Installation
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
# Test
certbot --version
# Verifier que le timer de renouvellement des certificats est bien en place
systemctl list-timers
sudo systemctl status snap.certbot.renew.timer
```

#### Sécuriser un domaine

Lors de la 1er installation Cerbot demanderas un email pour envoyer les avis de renouvellement plus d'autre chose pour la configuration, il suffit de suivre simplement l'installateur

```bash
sudo certbot --apache -d <example.com>
# Il est possible de sécuriser plusieurs domaines avec un seul certificat
  # Dans ce cas le 1er domaine est le domaine de base du certificat
  # Il est donc conseiller de mettre le domaine le plus "top-level"
sudo certbot --apache -d <example.com> -d <www.example.com>
# Pour tester que les domaines sont bien en place et renouvelé automatiquement
sudo certbot renew --dry-run
```

#### Forcer le renouvellement d'un certificat

```bash
sudo certbot --force-renewal -d <domain-name-1-here>[,<domain-name-2-here>]
```

#### Supprimer un certificat

```bash
certbot delete --cert-name <example.com>
```

#### Annuler le renouvèlement automatique d'un certificat

```bash
# Renommer le fichier avec disabled (il suffit de retirer le disabled pour le faire refonctionner)
sudo mv /etc/letsencrypt/renewal/<exemple.com>.conf /etc/letsencrypt/renewal/<exemple.com>.disabled
# Dans les versions récentes de Certbot il est possible d'ajouter en 1er ligne du fichier de renouvellement
sudo nano /etc/letsencrypt/renewal/<exemple.com>.conf
  # Ajouter au tout debut du fichier sur une nouvelle ligne : autorenew = False
```

#### Transférer un certificat

**Étape 1 : récupération des fichiers**

```bash
# Création d'un dossier key pour faire une copie récupérable des clefs
mkdir $HOME/key
# Copie des clefs/conf
sudo cat /etc/letsencrypt/live/<exemple.com>/cert.pem > $HOME/key/cert.pem
sudo cat /etc/letsencrypt/live/<exemple.com>/chain.pem > $HOME/key/chain.pem
sudo cat /etc/letsencrypt/live/<exemple.com>/fullchain.pem > $HOME/key/fullchain.pem
sudo cat /etc/letsencrypt/live/<exemple.com>/privkey.pem > $HOME/key/privkey.pem
sudo cp /etc/letsencrypt/renewal/<exemple.com>.conf $HOME/key/<exemple.com>.conf
# Copier les clef sur la machine intermédiaire
# Copier le fichier /etc/letsencrypt/renewal/<exemple.com>.conf sur la machine intermédiaire
# Suppression des copies
rm -r $HOME/key
```

**Étape 2 : Modification des fichiers**

Récupérer un fichier de conf renewal généré sur ce serveur (création d'un virtualhost juste pour avoir le fichier si besoins). Modifier le fichier `<exemple.com>.conf` (Penser à faire un backup avant)

```bash
# Remplacer version par la version du nouveau fichier
version = <x.yy.z>
# Remplacer account par le jeton du nouveau fichier
account = <jeton>
```

**Étape 3 : Import des fichiers sur le nouveau serveur**

```bash
# Création d'un dossier pour faire transiter les fichiers
mkdir $HOME/key
  # Déposer les fichiers dans le dossier key
# Copie du fichier de renouvellement et gestion des droits
sudo cp $HOME/key/<exmple.com>.conf /etc/letsencrypt/renewal/<exemple.com>.conf
sudo chown root:root /etc/letsencrypt/renewal/<exemple.com>.conf
sudo chmod 644 /etc/letsencrypt/renewal/<exemple.com>.conf
# Copie des fichiers dans le dossier letsencrypt
sudo mkdir /etc/letsencrypt/archive/<exemple.com>
sudo cp $HOME/key/cert.pem /etc/letsencrypt/archive/<exemple.com>/cert1.pem
sudo cp $HOME/key/chain.pem /etc/letsencrypt/archive/<exemple.com>/chain1.pem
sudo cp $HOME/key/fullchain.pem /etc/letsencrypt/archive/<exemple.com>/fullchain1.pem
sudo cp $HOME/key/privkey.pem /etc/letsencrypt/archive/<exemple.com>/privkey1.pem
# Modification droit et proprietaire
sudo chown -R root:root /etc/letsencrypt/archive/<exemple.com>
sudo chmod -R 644 /etc/letsencrypt/archive/<exemple.com>
sudo chmod 600 /etc/letsencrypt/archive/<exemple.com>/privkey1.pem
# Création des lien symbolique
sudo mkdir /etc/letsencrypt/live/<exemple.com>
sudo ln -s /etc/letsencrypt/archive/<exemple.com>/cert1.pem /etc/letsencrypt/live/<exemple.com>/cert.pem
sudo ln -s /etc/letsencrypt/archive/<exemple.com>/chain1.pem /etc/letsencrypt/live/<exemple.com>/chain.pem
sudo ln -s /etc/letsencrypt/archive/<exemple.com>/fullchain1.pem /etc/letsencrypt/live/<exemple.com>/fullchain.pem
sudo ln -s /etc/letsencrypt/archive/<exemple.com>/privkey1.pem /etc/letsencrypt/live/<exemple.com>/privkey.pem
# Supprime le dossier de transition
rm $HOME/key
```

**Étape 4 : Tester l'application du certificat et son renouvellement**

```bash
# Attention le site doit être actif sur le nouveau serveur et désactivé sur l'ancien
sudo certbot renew --dry-run
  # Si aucune erreur tout est bon
# Ouvrir le site dans le navigateur pour vérifier si il fonctionne
```

**Étape 5 : Annuler le renouvellement automatique sur l'ancien serveur** 

Si le serveur n'est pas supprimé bien pensé à empêcher le renouvellement automatique des certificats pour éviter les erreurs, pour ça voir la partie [dédié](#Annuler le renouvèlement automatique d'un certificat)

### HTPassword

https://www.digitalocean.com/community/tutorials/how-to-set-up-password-authentication-with-apache-on-ubuntu-14-04

```bash
# Création d'un fichier htpasswd avec les utilisateur autorisé et leur mot de passe respectif
sudo htpasswd -c /<path>/<to>/.htpasswd <user>
  # Suivre les instruction pour le mot de passe
  # Exemple de chemin pour un fichier utilisable partout /etc/apache2/.htpasswd
# Ajout d'un utilisateur avec son mot de passe
sudo htpasswd /<path>/<to>/.htpasswd <user>
```

Le controle de l'authentification peut être fait sois dans le fichier du Virtual Host, soit dans un fichier .htaccess

**Virtual Host**

Modifier le fichier du virtual host correspondant et ajouter à la fin de la balise `<VirtualHost>` (avant la balise fermante)

```htaccess
# Pour un sous dossier (exemple <Directory "/var/www/html">)
<Directory "{dossier_a_restraindre}">
        AuthType Basic
        AuthName "Restricted Content"
        AuthUserFile /{path}/{to}/.htpasswd
        Require valid-user
</Directory>
# Ou pour un reverse proxy (exemple <Location />)
<Location "{url}">
        AuthType Basic
        AuthName "Restricted Content"
        AuthUserFile /{path}/{to}/.htpasswd
        Require valid-user
</Location>
```

Pour restreindre tout le host il suffit de mettre la même valeur que dans `DocumentRoot`

**.htaccess**

Ajouter ces lignes dans le fichier

```htaccess
AuthType Basic
AuthName "Restricted Content"
AuthUserFile /etc/apache2/.htpasswd
Require valid-user
```

Une fois la protection mise en place,  il faut redémarrer le serveur

```bash
sudo systemctl restart apache2
```

