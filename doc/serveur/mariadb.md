# MariaDB

MariaDB est un fork de MySQL qui reprend la plupart de ses éléments et qui peut donc le remplacer sans changer de connecteur ou de configuration.

## Installation

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
# (Optionnel) Configurer le SGBD pour tourner sur un port et non un socket 
sudo nano /etc/mysql/my.cnf
  # Décommenter la ligne port = 3306 si elle commentée (il est aussi possible de changer le port)
  # Commenter la ligne socket = /run/mysqld/mysqld.sock si elle ne l'est pas
sudo systemctl restart mariadb
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

## Rendre disponible depuis d'autre machine

Par défaut le SGBD n'est disponible que depuis la machine locale et il est impossible de se connecter depuis une machine distante. Voici la config à modifier pour le rendre disponible à l'extérieur :

```bash
# Ouverture du port (adapter le port à votre configuration)
sudo ufw allow 3306
sudo ufw reload
# Changer la directive bind-address, elle peut être dans l'un de ces deux fichiers
sudo nano /etc/mysql/my.cnf
  # Ou
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
  # Dans le bon fichier mettre la valeur à bind-address = 0.0.0.0
# Rédemmarage du SGBD
sudo systemctl restart mariadb
# Vérification (adapter le port à votre configuration)
netstat -ant | grep 3306
  # La ligne devrait être (avec votre port) : tcp        0      0 0.0.0.0:3306            0.0.0.0:*               LISTEN
```

## Liens

- https://webdock.io/en/docs/how-guides/database-guides/how-enable-remote-access-your-mariadbmysql-database
