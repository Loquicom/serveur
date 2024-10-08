## Docker

Installation et configuration de docker et docker-compose

```bash
# Installation
sudo apt install -y docker docker-compose
# Configuration
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker
docker run hello-world
  # Si tous est ok le docker ce lance
```

### Redémarrage et mise à jour d'un container

Gestion des redémarrage : https://docs.docker.com/config/containers/start-containers-automatically/

Mise à jour automatique : https://containrrr.dev/watchtower/

Mise à jour manuelle d'un conteneur avec docker-compose :

```bash
# Coupe le conteneur actuel et le supprime (attention les données non stockées dans un volume sont perdues)
docker-compose down
# Récupère la dernière version de l'image
docker-compose pull
# Lance le conteneur en foçant l'utilisation de la nouvelle image (ajouté le -d pour le faire en arrière plan)
docker-compose up --force-recreate --build [-d]
# Supprime l'ancienne image
docker image prune -f
```

### Liens

- https://docs.docker.com/engine/install/linux-postinstall/