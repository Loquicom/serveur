## Mise à jour de l'OS (Ubuntu)

```bash
# Affiche la version actuel de l'OS
lsb_release -a

# Pour le mettre à jour, il faut dans un premier temps mettre à jour tous les paquets puis la distribution
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

## Mettre à jour le noyau sans mise à jour de l'OS

```bash
# Affiche les versions actuels
hostnamectl
# Ou
uname -r

# Pour le mettre à jour, il faut dans un premier temps mettre à jour tous les paquets puis le noyau
sudo apt update
sudo apt upgrade
sudo apt upgrade linux-generic

#Redemmarer le serveur
sudo shutdown -r now
```

## Notes diverses

Il peut être intéressant de faire un autoremove avant et après l'installation afin de nettoyer les paquets (`sudo apt autoremove --purge`)

Ne pas hésiter à faire une snapshot du VPS si c'est possible avant ce genre d'installation afin de pouvoir revenir en arrière en cas de problème
