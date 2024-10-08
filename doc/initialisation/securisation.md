## Mise à jour et sécurisation

Procédure pour mettre à jour et sécuriser un serveur linux Debian vierge

### Mise à jour

Renomme le serveur et met à jour les logiciels présents dessus

```bash
# Renommer le serveur
sudo nano /etc/hostname
  # Modifier par le nouveau nom du serveur
sudo nano /etc/hosts
  # Modfier la ligne 127.0.1.1 par le nouveau nom du serveur
# Mise à jour
sudo apt update
sudo apt upgrade
# Rédemmarer le serveur
sudo shutdown -r now
```

### Maj automatique

Paramètre les mise à jour automatique

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
  # Ajouter les lignes suivantes
  #   Unattended-Upgrade::Automatic-Reboot "true";
```

### Sécurisation

Création d'un nouvel utilisateur et modification de la configuration SSH

```bash
# Changement port ssh
sudo nano /etc/ssh/sshd_config
  # Modifier le numéro sur la ligne : Port 22 (décommenter si besoins)
  # Mettre PermitRootLogin à la valeur no (décommenter si besoins)
  # Mettre PubkeyAuthentication yes
  # Mettre PasswordAuthentication à la valeur no
sudo systemctl restart sshd
# Relancer la connexion SSH
# Création d'un nouvel utilisateur (et d'un groupe avec le même nom)
sudo adduser <user>
sudo usermod -aG sudo <user>
sudo mkdir /home/<user>/.ssh
sudo cp $HOME/.ssh/authorized_keys /home/<user>/.ssh
# Aller sur le nouveau compte (il possible de basculer avec la commande : su - <user>)
# Les commandes suivantes sont à faire sur le nouveau compte
sudo visudo
  # Ajouter la ligne suivante avant la derniere ligne du fichier
  #  <user> ALL=(ALL) NOPASSWD: ALL
sudo chown -R <user>:<group> $HOME/.ssh
sudo systemctl restart sshd
# Deconnexion reconnexion (normalement sans mot de passe)
sudo echo 1
  # Pour tester le sudo sans mot de passe
# Si tout est bon à partir d'ici on utilise plus le compte par défaut
sudo usermod --expiredate 1 <login-par-defaut>
  # Tester la connexion au compte par défaut pour être sur qu'elle est desactivée
# Activer les couleur dans le bash
nano $HOME/.bashrc
  # Décommenter la ligne force_color_prompt=yes
source $HOME/.bashrc
```

### Firewall

Installation est paramétrage d'un firewall

```bash
sudo apt install -y ufw
sudo nano /etc/default/ufw
  # Verifier que IPV6=yes
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow <port-ssh>
  # Possible d'ajouter d'autre port ou restriction, voir les guides
sudo ufw enable 
sudo ufw status verbose
  # Permet de voir les regles en place
```

### Liens

- Mise à jour automatique
  - https://www.cyberciti.biz/faq/how-to-set-up-automatic-updates-for-ubuntu-linux-18-04/
- Sécurisation
  - https://docs.ovh.com/fr/vps/conseils-securisation-vps/
  - https://www.digitalocean.com/community/tutorials/recommended-security-measures-to-protect-your-servers
- Firewall
  - https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-ubuntu-20-04-fr
  - https://doc.ubuntu-fr.org/ufw