# Crowdsec

Crowdsec est un logiciel open-source, gratuit et collaboratif de détection des menaces permettant de bloquer les attaques sur le serveur (c'est un équivalent plus moderne et rapide à fail2ban). 

Le principe de fonctionnement est simple, sur la machine un moteur va permettre la détection des menaces grâce à un ensemble de collections adapté au rôle et logiciel du serveur. Ensuite, un ou des bouncer sont installés sur le serveur et permettent d'appliquer les décisions prises par le moteur (le moteur ne fait qu'analyser et prendre des décisions, il ne protège pas directement). Il y a différents types de bouncer qui gèrent les alertes différemment en fonction de vos besoins (par exemple en bloquant l'IP, en retournant une erreur HTTP 403 ou en affichant un captcha).

## Installation

```bash
# Ajouter le dépot
curl -s https://packagecloud.io/install/repositories/crowdsec/crowdsec/script.deb.sh | sudo bash 
# Installation
sudo apt-get install -y crowdsec
# Affiche les service à protéger detecté par Crowdsec
sudo cscli collections list
  # Crowdsec installe par défaut certainnes collections en fonction de ce qu'il trouve sur le serveur
```

Une liste des collections existante est disponible sur le [site de Crowdsec](https://app.crowdsec.net/hub/configurations). Ci dessous une liste de collections utile à installer pour un serveur web (si elles ne sont pas déjà installées automatiquement)

```bash
# Collection pour une machine Linux
sudo cscli collections install crowdsecurity/linux
# Collection pour le SSH
sudo cscli collections install crowdsecurity/sshd
# Collections pour une machine utilsée comme serveur web
sudo cscli collections install crowdsecurity/base-http-scenarios
sudo cscli collections install crowdsecurity/http-cve
sudo cscli collections install crowdsecurity/http-dos # Optionnel
# Collection pour l'application de serveur web utilisée sur la machine (il faut installer uniquement celle utilisée)
sudo cscli collections install crowdsecurity/apache2 # Apache2
sudo cscli collections install crowdsecurity/nginx # Nginx
sudo cscli collections install crowdsecurity/caddy # Caddy
```

### Mise à jour des collections

Si dans la liste des collections il y a des :warning:, il faut les mettre à jours

```bash
# Toutes les collections
sudo cscli collections upgrade -a
# Une collection
sudo cscli collections upgrade <collection>
  # Par exemple : sudo cscli collections upgrade crowdsecurity/sshd
sudo systemctl reload crowdsec
```

## Bouncer

Il existe plein de bouncer adaptés à plein de situations et de technologies qui peuvent être retrouvés dans la [section dédiée du site web](https://app.crowdsec.net/hub/remediation-components). Ici seulement deux bouncer vont être évoqués et un seul détaillé mais qui peut être utilisé dans toutes les circonstances.

### Bouncer Firewall

Le bouncer Firewall ajoute les IP bloquées directement dans le pare-feu pour bloquer leur connexion (ce qui provoquera un timeout)

```bash
# Installation
sudo apt-get install -y crowdsec-firewall-bouncer-iptables
```

Si le bouncer ne fonctionne plus suite à une mise à jour il suffit de le reinstaller

```bash
# Suppression du bouncer
cscli bouncers delete <bouncer-name>
# Désintallation
sudo apt-get purge crowdsec-firewall-bouncer-iptables
# Reinstallation
sudo apt-get install -y crowdsec-firewall-bouncer-iptables
```

### Bouncer PHP

Le bouncer PHP analyse les nouvelle IP et si un requête est bloqué une page Web avec l'erreur 403 ou un captcha est affichés. **Attention** ce bouncer ne permet de bloquer que les appels direct vers une page PHP, ainsi si une adresse est par exemple un reverse proxy vers une autre application alors le connexion ne seras pas bloquées par le bouncer (la connexion étant juste envoyée sur l'application derrière le proxy). Dans ce cas plutôt utilisé le bouncer firewall qui bloque les requêtes au moment de la connexion avec la machine qui héberge le serveur et non lors du chargement d'une page.

Pour plus d'informations sur comment installer et configurer le bouncer PHP, se référer à son [dépôt Github](https://github.com/crowdsecurity/cs-standalone-php-bouncer)

## Commandes utiles

```bash
# Afficher la liste des bouncers
sudo cscli bouncers list
# Affiche les service à protéger detecté par Crowdsec
sudo cscli collections list
# Affiche les décisions prise par Crowdsec
sudo cscli decisions list
# Affiche les alertes détecté par Crowdsec
sudo cscli alerts list

# Ban une IP
sudo cscli decisions add --ip X.X.X.X
# Deban une IP
sudo cscli decisions delete --ip X.X.X.X

# En cas de problème lors de l'installation, il est possible de la relancer
sudo /usr/share/crowdsec/wizard.sh -c
# Il est aussi possible de redétecter les fichiers de logs des servies à protèger
sudo /usr/share/crowdsec/wizard.sh -d
```

### Tester Crowdsec

Si aucun bouncer n'est présent lors du test, on doit pouvoir voir la décision de crowdsec mais aucun blocage n'aura lieu. Le test doit être fait depuis une autre machine, les test fait depuis la même machine n'activeront rien car le trafic local est whitelisté

````bash
# Nikto (analyse un serveur Web à la recherche de vulnérabilités ou de défaut de configuration)
sudo apt-get install -y nikto
curl -I <site.com>
  # Verification que le serveur est joignable
nikto -h <site.com>
  # Aller vérifier la liste de décision de Crowdsec
curl --connect-timeout 1 http://<site.com>
  # Si un bouncer est en place on peut vérifier que l'ip est bien bloquée
  
# Wapiti (Scanner de vulnérabilités d'application web)
sudo apt-get install -y wapiti
curl -I <site.com>
wapiti -u http://<site.com>
  # Aller vérifier la liste de décision de Crowdsec
  
# Avec des User-agent bloqué (permet de tester le captcha, les autres vont se faire ban quoi qu'il arrive)
curl -I <site.com> -H "User-Agent: Cocolyzebot"
curl -I <site.com> -H "User-Agent: Mozlila"
curl -I <site.com> -H "User-Agent: OpenVAS"
curl -I <site.com> -H "User-Agent: Nikto"
  # Un code d'erreur devrait finir par arriver
````

## Dashboard

Il existe deux dashboards, le 1er Metabase hébergé en local, le 2nd Crowdsec console qui permet de voir plusieurs instance de Crowdsec sur le même dashboard

### Metabase

```bash
# Installe le container docker
sudo cscli dashboard setup -l 0.0.0.0 -p <port>
  # Les identifiants généré lors de l'installation sont stocké dans /etc/crowdsec/metabase/metabase.yaml
# Ajout du rédemmarage auto
docker update --restart unless-stopped crowdsec-metabase  
# Le container est gérable soit par docker soit par cscli, voir l'aide
sudo cscli dashboard -h
# Supprime le dashboard
sudo cscli dashboard remove [-f]
# Stop le dashboard
sudo cscli dashboard stop
# Démmare le dashboard
sudo cscli dashboard start
```

### Crowdsec console

Disponible sur https://app.crowdsec.net

```bash
# Ajouter l'instance Crowdsec sur la console
sudo cscli console enroll <token>
  # Le token est disponible sur la console
sudo systemctl restart crowdsec
```

## Liens

- Installation

  - https://github.com/crowdsecurity/crowdsec
  - https://crowdsec.net/blog/tutorial-crowdsec-v1-1/
  - https://www.it-connect.fr/bloquer-les-attaques-sur-son-serveur-web-apache-php-avec-crowdsec/

- Dashboard

  - https://docs.crowdsec.net/docs/observability/dashboard/
  
  
  

 

  

  



