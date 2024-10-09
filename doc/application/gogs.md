## Gogs (serveur Git)

Gogs est un serveur git auto-hébergeable léger avec une interface web pour le configurer et parcourir les dépôts.

### Mise en place du serveur

```bash
# Création du dossier
sudo mkdir /opt/gogs
sudo chown <user>:<group> /opt/gogs
# Création du dossier pour les données
mkdir /opt/gogs/data
# Création du fichier de configuration docker
nano /opt/gogs/docker-compose.yaml
```

Remplir le fichier avec les données suivantes (bien remplacer les éléments entre <>)

```yaml
version: '3.3'

services:
  gogs-db:
    image: mysql:latest
    container_name: "gogs-db"
    restart: always
    environment:
      - MYSQL_DATABASE=gogs
      - MYSQL_ROOT_PASSWORD=<root-password>
      - MYSQL_USER=gogs
      - MYSQL_PASSWORD=<password>
  gogs:
    container_name: gogs
    image: gogs/gogs:latest
    ports:
      - "<port-web>:3000"
      - "<port-ssh>:22"
    volumes:
      - /opt/gogs/data:/data
    restart: always
    environment:
      - RUN_CROND=true
      - BACKUP_INTERVAL=1d
      - BACKUP_RETENTION=3d
    depends_on:
      - "gogs-db"
```

Terminer l'édition du fichier

```bash
# Lancer le docker
docker-compose up -d
# Ouvrir les ports si besoins
sudo ufw allow <port-web>
sudo ufw allow <port-ssh>
```

### BDD

Pour la BDD il soit possible d'utiliser docker pour l'ajouter ou d'en utiliser une directement présente sur le serveur. Sinon il reste la possibilité d'utiliser SQLite ce qui ne demande aucune configuration mais qui est moins efficace.

#### Utilisation d'un MySQL/MariaDB via docker

Dans notre cas le `docker-compose.yaml` possède déjà la définition d'un conteneur MySQL. Il suffit juste lors de la configuration de Gogs via l'interface web de sélectionner MySQL et de mettre `gogs-db:3306` dans le champ hôte de la base de données, ainsi que le mot de passe définit dans le fichier `docker-compose.yaml`

#### Utilisation d'un MySQL/MariaDB déjà présent sur le serveur

Pour utiliser le SGBD déjà installer localement sur la machine il faut d'abord retirer du `docker-compose.yaml` la création du docker SGBD puis ouvrir une session MySQL avec un compte root

```mysql
# Création d'une nouvelle BDD
CREATE DATABASE gogs CHARACTER SET utf8 COLLATE utf8_general_ci;
# Création d'un utilisateur
CREATE USER 'gogs'@'%' IDENTIFIED BY '<password>';
# Droit sur la BDD
GRANT ALL ON gogs.* TO 'gogs'@'%';
```

Il suffit ensuite de mette l'ip du serveur et de rendre la connexion au serveur MySQL depuis l'exterieur (notamment débloquer le port dans le parefeu). 

Il est possible d'utiliser le localhost pour accéder (puisque les deux sont sur la même machine) mais c'est déconseillé puisque que cela brise l'isolation du container. Pour ce faire il faut ajouter la ligne suivante dans le `docker-compose.yaml`

```yaml
services:
  gogs:
    network_mode: host
```

### Configuration

Lors de la 1er connexion au site la configurateur ce lance, il suffit de renseigner les différents champs :

- Nom de l'application
- Domaine (ex: git.site.com)
- Port SSH (par le port-ssh dans le `docker-compose.yaml`)
- Port HTTP (par le port-web dans le `docker-compose.yaml`)
- l'URL
- Les infos pour l'envoie d'email si besoins
- Les infos pour la gestion des comptes

Tous les paramètres sont modifiable après coup en éditant le fichier `/opt/gogs/data/gogs/custom/conf/app.ini` (bien penser à redémarrer Gogs après les modifications).

### Mise en ligne

Il est maintenant possible accéder au serveur git par le port configurer dans le docker-compose. Cependant si il y a déjà un serveur web et que l'on souhaite accéder à l'interface du serveur git par celui-ci il faut mettre en place en reverse proxy. 

Pour un serveur Apache voir [la section Reverse proxy de la documentation dédiée](../serveur/apache.md#reverse-proxy)

En passant par un reverse proxy la connexion https passe par le serveur web, c'est donc lui qu'il faut configurer pour fournir une connexion sécurisée (en effet c'est la connexion entre le serveur web et le client qui passe par l'extérieur et qui doit donc être sécurisée, la connexion entre le serveur web et le serveur git est une connexion interne à la machine il n'y a donc aucun risque à utiliser une connexion http). Cependant si le serveur web intégré à Gogs est directement exposé il faut bien veillé à le configurer avec les certificats pour qu'il puisse fournir une connexion sécurisée

### Migration / Restauration

Description du processus de migration / restauration dans la cas de l'utilisation du docker-compose présent dans le repo, ainsi qu'une liste de toutes les commandes utiles dans ce cas là.

#### Depuis une installation avec le docker-compose du repo

Copie de la configuration et des différents dépôts

```shell
# Reprise de la configuration
/opt/gogs/data/ # Tous le dossier data
/opt/gogs/data/gogs/conf/app.ini # Ou juste le fichier de conf principale
# Reprise des anciens dépots git
/opt/gogs/data/git/gogs-repositories/
```

Ensuite il faut aller dans le container docker de l'ancienne installation pour récupérer la base de données

```shell
# Affiche les containers
docker ps
# Utilise l'id du container Gogs pour ouvrir un session bash dedans
docker exec -it <container_id> bash

# Dans le container
cd ..
USER=git ./gogs backup --database-only
exit

# Récupèration du fichier de backup dans le dossier courant
docker cp <container_id>:/app/gogs/<backup_name> .
```

Remettre tous les fichier copier sur le nouveau serveur (par exemple dans `/opt/gogs/`), suivant la même arborescence qu'avant puis démarrer les containers. Une fois les containers démarrés, depuis le dossier où le backup de la BDD est présent lancé les commandes suivantes

```shell
# Affiche les containers
docker ps
# Copie le fichier de backup dans le container
docker cp <backup_name> <container_id>:/data/
# Utilise l'id du container Gogs pour ouvrir un session bash dedans
docker exec -it <container_id> bash

# Dans le container
mkdir /data/tmp
cd /data
USER=git /app/gogs/gogs restore --database-only --from="<backup_name>" -t /data/tmp -c data/gogs/conf/app.ini
rm -R /data/tmp
rm <backup_name>
exit
```

Lors d'une migration le mot de passe des comptes peut ne pas être exporté, il suffit d'aller voir la table dans le docker mysql de gogs et si besoins update avec le mot de passe présent sur le site de départ de la migration, ou alors de créer un nouveau compte et de copier son `passwd` et son `salt` sur le compte d'origine avant de le supprimer.

```shell
# Commande de connexion à la BDD une fois dans le docker mysql
mysql -u gogs -p gogs
```

```sql
/* La table des utilisateurs */
SELECT name,passwd,salt FROM user;
Update user set passwd = '<password>' where name = '<username>';
commit;
```

#### Commandes

Les commandes sont exécutées depuis le dossier ou il y a l'exécutable. Dans le cas d'un utilisation avec docker c'est donc depuis le container et il faut ensuite copié avec `docker cp` les fichier entre le container et la machine hôte.

```shell
# Backup l'ensemble des données (gogs-repositories, data, custom et database)
./gogs backup
# Backup tous sauf gogs-repositories (par exemple s'il est trop lourd)
./gogs backup --exclude-repos
# Backup uniquement la base de données, le format est en JSON et permet lleur utilisation dans la cadre d'un changement de SGBD
./gogs backup --database-only
# Si le fichier de config n'est pas à l'endroit standard
./gogs backup --config=my/custom/conf/app.ini
# Restore tous
./gogs restore --from="gogs-backup-xxx.zip"
# Restore uniquement la base de données (l'option config est obligatoire si elle n'est pas présent dans le zip)
./gogs restore --database-only --from="gogs-backup-xxx.zip" --config=data/gogs/conf/app.ini
```

Et un ensemble de commande utile dans le cas ou Gogs est conteneurisé avec Docker.

**Faire le backup :**

```shell
# Dans le container Gogs
cd ..
USER=git ./gogs backup
exit
# Sur la machine hôte
docker cp <container_id>:/app/gogs/<backup_name> .
```

Ou alors il est possible de le faire en une seule commande

```shell
docker exec gogs bash -c 'ls /app/gogs | rm -f gogs-backup-*.zip && cd /app/gogs && USER=git ./gogs backup' && docker cp gogs:$(docker exec gogs bash -c 'readlink -f $(ls /app/gogs | grep gogs-backup-*.zip -)') . && md5sum $(docker exec gogs bash -c 'ls /app/gogs | grep gogs-backup-*.zip -') > $(docker exec gogs bash -c 'ls /app/gogs | grep gogs-backup-*.zip -').md5 && docker exec gogs bash -c 'ls /app/gogs | rm gogs-backup-*.zip'
```

**Restaurer les données :**

```shell
# Sur la machine hôte
docker cp <backup_name> <container_id>:/data/
# Dans le container Gogs
mkdir /data/tmp
cd /data
USER=git /app/gogs/gogs restore --from="<backup_name>" -t /data/tmp
```

Note: cross-device-link is caused by move between 2 mount so the solution is to use tmp folder in the same mount as data where gogs reside which is /data by default for docker gogs.

### Liens

- https://hub.docker.com/r/gogs/gogs
- https://www.cloudsavvyit.com/14114/how-to-connect-to-localhost-within-a-docker-container/

