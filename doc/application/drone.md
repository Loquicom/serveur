## Drone (CI/CD)

Installation d'une application de CI/CD pour Github et pour Gogs (serveur git auto-hébergé)

### Installation

```bash
# Création du dossier
sudo mkdir /opt/drone
sudo chown <user>:<group> /opt/drone
# Création du dossier pour les données
mkdir /opt/drone/data/github
mkdir /opt/drone/data/gogs
# Création du fichier de configuration docker
nano /opt/drone/docker-compose.yaml
# Génération d'un token secret pour la communication entre server et runner (lancer 2 fois, une fois par installation)
openssl rand -hex 16
```

Remplir le fichier avec les données suivantes (bien remplacer les éléments entre <>). 

Dans le fichier il y a deux runner pour chaque serveur, un de type Docker (qui peut lancer des docker) et un de type SSH (qui permet exécuter des commandes sur le serveur). Pour désactiver l'un des runner il suffit de supprimer le service correspondant dans le fichier `docker-compose.yaml`. Il est possible d'ajouter d'autre type de runner, plus d'informations sur : https://docs.drone.io/runner/overview/

```yaml
version: '3.3'

services:
  drone-server-github:
    container_name: "drone-github"
    image: drone/drone:latest
    ports:
      - "3123:80"
    volumes:
      - /opt/drone/data/github:/data
      - /var/run/docker.sock:/var/run/docker.sock
    restart: unless-stopped
    environment:
      - DRONE_GITHUB_SERVER=https://github.com
      - DRONE_GITHUB_CLIENT_ID=<github-client-id>
      - DRONE_GITHUB_CLIENT_SECRET=<github-client-secret>
      - DRONE_RPC_SECRET=<drone-github-rpc-secret>
      - DRONE_SERVER_HOST=<github-host>
      - DRONE_SERVER_PROTO=https
      - DRONE_USER_FILTER=<github-username>
      - DRONE_USER_CREATE=username:<github-username>,admin:true
      - DRONE_LOGS_TRACE=false
      - DRONE_LOGS_PRETTY=true
      - DRONE_LOGS_COLOR=true

  drone-runner-docker-github:
    container_name: "drone-github-docker"
    image: drone/drone-runner-docker:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    restart: unless-stopped
    environment:
      - DRONE_RPC_PROTO=http
      - DRONE_RPC_HOST=drone-github
      - DRONE_RPC_SECRET=<drone-github-rpc-secret>
    depends_on:
      - "drone-server-github"

  drone-runner-ssh-github:
    container_name: "drone-github-ssh"
    image: drone/drone-runner-ssh:latest
    restart: unless-stopped
    environment:
      - DRONE_RPC_PROTO=http
      - DRONE_RPC_HOST=drone-github
      - DRONE_RPC_SECRET=<drone-github-rpc-secret>
    depends_on:
      - "drone-server-github"

  drone-server-gogs:
    container_name: "drone-gogs"
    image: drone/drone:latest
    ports:
      - "3321:80"
    volumes:
      - /opt/drone/data/gogs:/data
      - /var/run/docker.sock:/var/run/docker.sock
    restart: unless-stopped
    environment:
      - DRONE_GOGS_SERVER=https://git.loquico.me
      - DRONE_RPC_SECRET=<drone-gogs-rpc-secret>
      - DRONE_SERVER_HOST=<gogs-host>
      - DRONE_SERVER_PROTO=https
      - DRONE_USER_FILTER=<gogs-username>
      - DRONE_USER_CREATE=username:<gogs-username>,admin:true
      - DRONE_LOGS_TRACE=false
      - DRONE_LOGS_PRETTY=true
      - DRONE_LOGS_COLOR=true

  drone-runner-docker-gogs:
    container_name: "drone-gogs-docker"
    image: drone/drone-runner-docker:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    restart: unless-stopped
    environment:
      - DRONE_RPC_PROTO=http
      - DRONE_RPC_HOST=drone-gogs
      - DRONE_RPC_SECRET=<drone-gogs-rpc-secret>
    depends_on:
      - "drone-server-gogs"

  drone-runner-ssh-gogs:
    container_name: "drone-gogs-ssh"
    image: drone/drone-runner-ssh:latest
    restart: unless-stopped
    environment:
      - DRONE_RPC_PROTO=http
      - DRONE_RPC_HOST=drone-gogs
      - DRONE_RPC_SECRET=<drone-gogs-rpc-secret>
    depends_on:
      - "drone-server-gogs"
```

Terminer l'édition du fichier

```bash
# Lancement
docker-compose up
  # Si les serveurs sont bien lancé et que les runner arrive à les ping tout est ok
  # Ctrl+C
# Lancement en fond
docker-compose up
```

### Migration des données

Copier le contenue du dossier `/opt/drone/data` dans le nouveau dossier data

### Mise en ligne

Le fichier `docker-compose.yaml` ci-dessus a été pensé pour être utilisé dérriere un serveur web en reverse proxy. Pour utiliser directement le serveur de drone sans intermédiaire modifier le mapping des port du fichier et référez vous à la documentation officiel pour activer la connexion https.

Pour configurer le reverse proxy sur un serveur Apache voir [la section dédiée du fichier LAMP](./lamp.md#reverse-proxy). En passant par un reverse proxy la connexion https passe par le serveur web, c'est donc lui qu'il faut configurer pour fournir une connexion sécurisée (en effet c'est la connexion entre le serveur web et le client qui passe par l'extérieur et qui doit donc être sécurisée, la connexion entre le serveur web et le serveur Drone est une connexion interne à la machine il n'y a donc aucun risque à utiliser une connexion http).

### Architecture distribuée

Il est possible de mettre les runners et le serveur sur des ordinateurs différents. Pour cela il suffit d'extraire les services des runner du `docker-compose.yaml` pour les mettre dans un autre fichier présent sur l'ordinateur qui va les exécuter. Dans la définition du runner il faut simplement modifier l'host et le proto par ceux d'accès au serveur (normalement les même que ceux pour accéder à l'interface dans un navigateur) et retirer le `depends_on`

### Liens

- https://docs.drone.io/server/provider/github/
- https://docs.drone.io/server/provider/gogs/