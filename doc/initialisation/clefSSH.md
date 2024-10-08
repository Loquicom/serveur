## Connexion par clef SSH uniquement

Mise en place d'une connexion par clef SSH sur le serveur, de la génération de la clef jusqu'à son installation sur le serveur

### Génération clef SSH

Dans un premier temps il faut générer une paire de clef privée/publique soit via PuttyGen (Windows), soit via ssh-keygen (Linux)

**PuttyGen**

- Ouvrir PuttyGen
- Si votre serveur est compatible généré une clef avec algorithme EdDSA (Depuis Debian 8)
  - Key > SSH 2 EdDSA Key ou dans Parameters > Type of Key > EdDSA 
    - Attention souvent le 448 bits n'est pas supporté (ed448)
    - Utiliser plutôt le 255 bits (ed25519)
  - Sinon utilisé RSA (algorithme par défaut) avec un clef de 4096 bits
- Modifier les paramètres de génération en fonction de la sécurité souhaitée (dans Key et Parameters)
- Cliquer sur Generate
- Modifier le nom si besoins
- Ajouter une mot de passe pour plus de sécurité
- Sauvegarder la clef publique (.ppuk) et la clef privée (.ppk) au format Putty
- Sauvegarder la clef privée au format OpenSSH
  - Conversions > Export OpenSSH Key
  - Nommer le fichier `<nom-clef>.<algo>` (Exemple : site-com.eddsa ou site-com.rsa)
- Sauvegarder la clef publique au format OpenSSH
  - Copier le contenue de Key > Public key for pasting into OpenSSH authorized_key file
  - Coller dans un fichier nommé `<nom-clef>.pub`

**ssh-keygen (OpenSSH)**

Utiliser une des commandes suivantes puis suivre les instructions. Par défaut un fichier nommé `<clef>.pub` et `<clef>` vont être crée dans le dossier `.ssh` de l'utilisateur. Ces fichiers correspondent respectivement à la clef publique et la clef privée.

```bash
# Générer une paire de clef avec les paramétres par défaut (en géneral RSA 2048 bits)
ssh-keygen
# Générer une paire de clef RSA de 4096 bits (recommandé si l'on doit utiliser RSA)
ssh-keygen -t rsa -b 4096
# Générer une paire de clef EdDSA 255 bits (recommandé si le serveur est compatible)
ssh-keygen -t ed25519
```

Pour obtenir les versions Putty des clefs publique et privée il faut importer le fichier de clef privée OpenSSH dans PuttyGen et utiliser les boutons pour sauvegarder au format Putty.

### OVH

Aouter la clef publique dans `Mon Compte` > `Mes services` > `Clef SSH`, puis réinstaller le serveur avec authentification par clef ssh. Attention ceci n'est possible que pour installer une première clef (et ne pas générer de mot de passe pour le compte par défaut). Pour ajouter une autre clef voir la section **Manuel**

### Manuel

Si le PC (et non le serveur) est un linux et qu'il est possible de se connecter par mot de passe en SSH (Attention il faut dans le dossier un fichier nommé `<clef>.pub` pour la clef publique et un fichier nommé `<clef>` pour la clef privé, sinon il y a juste la clef publique utilisé l'option `-f` pour envoyer son contenue sans vérification)

```bash
# Copie automatiquement la clef publique du PC
ssh-copy-id <username>@<remote_host>
# Copie une clef spécifique sur le serveur
ssh-copy-id -i <path-to-key.pub> <username>@<remote_host>
# Copie une clef spécifique sur le serveur via un port différent de celui par défaut (22)
ssh-copy-id -i <path-to-key> -p <port> <username>@<remote_host>
```

Sinon par SSH (la clef doit être dans le format pub de ssh-keygen)

```bash
cat <path_to_key> | ssh <username>@<remote_host> "mkdir -p ~/.ssh && touch ~/.ssh/authorized_keys && chmod -R go= ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

### Liens

- https://docs.ovh.com/fr/public-cloud/premiers-pas-instance-public-cloud/#etape-1-creer-des-cles-ssh
- https://docs.ovh.com/fr/public-cloud/configurer-des-cles-ssh-supplementaires/
- https://docs.ovh.com/fr/public-cloud/changer-sa-cle-ssh-en-cas-de-perte/
- https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys-on-ubuntu-20-04-fr
- https://tutox.fr/2020/04/16/generer-des-cles-ssh-qui-tiennent-la-route/

