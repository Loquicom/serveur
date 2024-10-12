# Certbot

Certbot permet d'installer et de renouveler les certificats SSL pour permettre une connexion en HTTPS des différents domaines installés sur le serveur de façon automatique, et de modifier la configuration du serveur pour forcer la redirection dessus. Les certificats sont émis par l'autorité "Let's Encrypt" gratuitement.

Dans les commandes ci-dessous, l'exemple d'un serveur Apache va être utilisé, mais il est aussi compatible avec Nginx (il n'est pas nécessaire pour Caddy car il gère lui même les certificats).

## Installation

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

## Sécuriser un domaine

Lors de la 1er installation Cerbot demanderas certains éléments pour sa configuration dont un email pour envoyer les avis de renouvellement, il suffit simplement  de suivre l'installateur.

```bash
sudo certbot --apache -d <example.com>
# Il est possible de sécuriser plusieurs domaines avec un seul certificat
  # Dans ce cas le 1er domaine est le domaine de base du certificat
  # Il est donc conseiller de mettre le domaine le plus "top-level"
sudo certbot --apache -d <example.com> -d <www.example.com>
# Pour tester que les domaines sont bien en place et renouvelé automatiquement
sudo certbot renew --dry-run
```

### Forcer le renouvellement d'un certificat

```bash
sudo certbot --force-renewal -d <domain-name-1-here>[,<domain-name-2-here>]
```

### Supprimer un certificat

```bash
certbot delete --cert-name <example.com>
```

### Annuler le renouvèlement automatique d'un certificat

```bash
# Renommer le fichier avec disabled (il suffit de retirer le disabled pour le faire refonctionner)
sudo mv /etc/letsencrypt/renewal/<exemple.com>.conf /etc/letsencrypt/renewal/<exemple.com>.disabled
# Dans les versions récentes de Certbot il est possible d'ajouter en 1er ligne du fichier de renouvellement
sudo nano /etc/letsencrypt/renewal/<exemple.com>.conf
  # Ajouter au tout debut du fichier sur une nouvelle ligne : autorenew = False
```

## Transférer un certificat

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

## Liens

- https://certbot.eff.org/instructions?ws=apache&os=snap
- https://www.digitalocean.com/community/tutorials/how-to-secure-apache-with-let-s-encrypt-on-ubuntu-18-04
- https://www.cyberciti.biz/faq/how-to-forcefully-renew-lets-encrypt-certificate/
- https://stackoverflow.com/questions/40030537/how-to-stop-renewing-a-letsencrypt-certbot-certificate
- https://serverfault.com/questions/1057412/how-to-automate-certbot-certificate-renewal-on-ubuntu-20-04