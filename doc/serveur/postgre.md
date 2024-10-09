# PostgreSQL

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

## Liens

- https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-20-04-fr
- https://devstory.net/12255/configurer-postgresql-pour-autoriser-les-connexions-a-distance