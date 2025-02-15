## Docker

Installation et configuration de docker et docker-compose

```bash
# Jout de la clef GPG officiel de Docker
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
# Ajoute le repo aux sources d'APT
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
# Installation
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
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
docker compose down
# Récupère la dernière version de l'image
docker compose pull
# Lance le conteneur en foçant l'utilisation de la nouvelle image (ajouté le -d pour le faire en arrière plan)
docker compose up --force-recreate --build [-d]
# Supprime l'ancienne image
docker image prune -f
```

Anciennement, docker compose n'était pas un plugin de docker et pour l'utiliser il fallait écrire `docker-compose <args>`

### Liens

- https://docs.docker.com/engine/install/linux-postinstall/
- https://docs.docker.com/compose/releases/migrate/#how-do-i-switch-to-compose-v2
