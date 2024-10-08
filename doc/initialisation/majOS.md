## Mise à jour de l'OS (Ubuntu)

```bash
# Affiche la version actuel de l'OS
lsb_release -a

# Pour le mettre à jour, il faut dans un premier temps mettre à jour tous les paquets
sudo apt update
sudo apt upgrade
sudo apt dist-upgrade

# Ouvrir le port 1022 pour accèder au SSH de secours
sudo ufw allow 1022
sudo ufw reload

# Ensuite mettre à jour l'OS
sudo do-release-upgrade
  # Attention si des fichiers de config ont été modifié refuser le redemmarage automatique et bien penser à les remodifier
  # Une fois les fichiers modifiés exécuter la commande pour redemmarer
  sudo shutdown -r now

# Désactiver le port 1022 ouvert pour le SSH de secours
sudo ufw delete allow 1022
sudo ufw reload
```

