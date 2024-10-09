# MariaDB

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
