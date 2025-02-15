# Serveur

Ce dépôt a pour but de contenir toutes la documentation nécessaire pour l'installation d'un serveur Linux Debian, en fonction des composants souhaités.

## Guides d'installation

La documentation est découpée en plusieurs catégorie, chaque catégorie possède son index

- [Configuration d'un linux pour en faire un serveur](./doc/initialisation)
- [Installation et configuration d'un serveur web et d'une base de données](./doc/serveur)
- [Installation et configuration de différentes applications](./doc/application)

## Applications

Le dossier `app` contient les fichiers docker-compose et de configuration pour différentes applications. Certaines applications possèdent une documentation détaillé pour leur mise en place, pour ceci ce référer aux guides d'installation

- Drone (Outil de CI/CD)
- Ferdium
- Gogs (Serveur Git)
- Umami (Analytics)

## Template

Le dossier `template` contient des fichiers d'exemple réutilisables pour différents services

- Vhost Apache
- .htaccess
- Scripts utilitaires
- HTML/PHP

## Todo one day

- Serveur bitwarden
- [Service systemd](https://abhinand05.medium.com/run-any-executable-as-systemd-service-in-linux-21298674f66f)
- VPN Wireguard ou OpenVPN avec PiVPN
- Ajouter documentation Webdav Apache
- Adapter documentation Crowdsec au nouveau format
- Faire documentation Caddy
- [Mount FSTab](https://doc.ubuntu-fr.org/mount_fstab)
