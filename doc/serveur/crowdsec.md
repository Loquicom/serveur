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

