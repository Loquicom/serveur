# Apache

Apache est un des serveur web les plus anciens et des plus utilisé. Il à l'avantage de pouvoir lui ajouter des extensions à la volé et d'avoir des fichiers de configurations bien séparé. De plus son ancienneté et son utilisation massive fait qu'il est très bien documenté?

## Installation

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
# Permet la lecture/modification du réportoire www à l'utilisateur apache
sudo chown -R www-data:www-data /var/www
sudo chmod -R 775 /var/www
sudo usermod -a -G www-data $USER
# Redémmarer le serveur
sudo systemctl restart apache2
```

## Utilisation de HTTP/2

Modification de la configuration pour activer HTTP/2, paramétrage de PHP-FPM et l'utilisation avec un reverse proxy

### Mise en place

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

### PHP-FPM

**Utilisation php_value**

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

## Paramétrage de sécurité

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

## Ajouter un Virtual Host

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

## Configuration Virtual Host

Ensemble de configuration possible (non exhaustive) de fichier Virtual Host. Dans tous les cas il faut ajouter les ligne dans la balise `<VirtualHost>`

### Reverse proxy

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

### Reverse proxy & Load balancing

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

### Sous site

```htaccess
# Le 1er paramétre correspond au chemin dans l'url
  # Exemple /dev sur site.com => site.com/dev
# Le 2nd paramétre est le dossier du site
Alias /<URL> /var/www/public
```

### Sous site & Reverse proxy

```htaccess
# Le 1er paramétre correspond au chemin dans l'url
  # Exemple /dev sur site.com => site.com/dev
# Le 2nd paramétre est l'adresse du site
ProxyRequests Off
ProxyPreserveHost On
ProxyPass /<URL>/ http://localhost:<port>/
ProxyPassReverse /<URL>/ http://localhost:<port>/

```

### Bloquer l'accès quand connexion via une URL précise

```htaccess
<VirtualHost *:80>
    ServerName <dev.site.com>
    <Location />
        Require all denied
    </Location>
</VirtualHost>
```

## HTPassword

```bash
# Création d'un fichier htpasswd avec les utilisateur autorisé et leur mot de passe respectif
sudo htpasswd -c /<path>/<to>/.htpasswd <user>
  # Suivre les instruction pour le mot de passe
  # Exemple de chemin pour un fichier utilisable partout /etc/apache2/.htpasswd
# Ajout d'un utilisateur avec son mot de passe
sudo htpasswd /<path>/<to>/.htpasswd <user>
```

Le controle de l'authentification peut être fait sois dans le fichier du Virtual Host, soit dans un fichier .htaccess

### Virtual Host

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

### .htaccess

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

## Webdav

TODO

1. sudo a2enmod dav
2. sudo a2enmod dav_fs


## Liens

- Généraux
  - https://www.it-connect.fr/installer-un-serveur-lamp-linux-apache-mariadb-php-sous-debian-11/
  - https://www.digitalocean.com/community/tutorials/how-to-install-linux-apache-mysql-php-lamp-stack-on-ubuntu-20-04-fr
  - https://www.digitalocean.com/community/tutorials/how-to-set-up-apache-virtual-hosts-on-ubuntu-16-04
- HTTP/2
  - https://www.howtoforge.com/how-to-enable-http-2-in-apache/
  - https://medium.com/tech-learnings/how-to-configure-apache-reverse-proxy-with-http-2-ebc87c1c7bcb
- PHP FPM
  - https://maxchadwick.xyz/blog/setting-a-php-value-in-php-fpm
  - https://serverfault.com/questions/785782/apache-with-php-fpm-php-doesnt-execute
  - https://cwiki.apache.org/confluence/display/httpd/PHP-FPM
- Reverse Proxy
  - https://www.vincentliefooghe.net/content/apache-reverse-proxy-https
  - https://www.digitalocean.com/community/tutorials/how-to-use-apache-as-a-reverse-proxy-with-mod_proxy-on-ubuntu-16-04
  - https://medium.com/@ppranav164/how-proxypass-proxypassreverse-works-in-the-apache-server-11177fb68b4c
- HTPassword
  - https://www.digitalocean.com/community/tutorials/how-to-set-up-password-authentication-with-apache-on-ubuntu-14-04
- Webdav
  - https://www.digitalocean.com/community/tutorials/how-to-configure-webdav-access-with-apache-on-ubuntu-18-04