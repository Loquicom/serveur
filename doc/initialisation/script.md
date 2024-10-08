## Scripts personnalisés & Alias

Ajout de scripts personnalisés pouvant être utilisé comme des commandes n'importes où sur le serveur

### Mise en place

```bash
# Création d'un dossier script
mkdir $HOME/script
# Ajoute les lignes à la fin du fichier .profile
echo "" >> "$HOME/.profile"
echo "# Custom env" >> "$HOME/.profile"
echo "export PATH=\"\$HOME/script:\$PATH\"" >> "$HOME/.profile"
# Création d'un fichier d'alias
touch $HOME/.bash_aliases
# Remplissage du fichier 
echo "# Alias" >> $HOME/.bash_aliases
echo "alias ll='ls -laF'" >> $HOME/.bash_aliases
echo "alias la='ls -lA'" >> $HOME/.bash_aliases
echo "alias l='ls -CF'" >> $HOME/.bash_aliases
# Recharge les fichiers
source $HOME/.profile
source $HOME/.bashrc
```

### Ajout d'un nouveau script

Il suffit d'ajouter un fichier exécutable dans le dossier `$HOME/script`. La 1er ligne doit être indiquant avec quel programme exécuter le script avec le format suivant `#!/path/to/exec`. Voici quelques exemples (:warning: les chemins ne sont pas forcément les mêmes)

- Fichier bash (.sh) : `#!/bin/bash`
- Fichier PHP (.php) : `!#/usr/bin/php`
- Fichier Node (.js) : `!#/usr/bin/node`

Il suffit ensuite de taper le nom du fichier dans le CLI pour exécuter de n'importe avec le bonne interpréteur (l'extension n'est pas obligatoire à la fin du fichier). Plusieurs script sont déjà prêt à l'emploi dans le dossier [script](./script).

### Ajout de nouveaux alias

Il suffit d'ouvrir le fichier `$HOME/.bash_aliases` et d'ajouter de nouveaux alias de la façon suivante

```bash
alias <nom-alias>='<cmd-a-executer>'
```

Une fois ajouté pour en bénéficier tout de suite exécuter 

```bash
source $HOME/.bash_aliases
```

