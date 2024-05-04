# Serveur

Ce dépôt a pour but de contenir toutes la documentation nécessaire pour l'installation d'un serveur Linux Debian, en fonction des composants souhaités.

## Guides d'installation

- [Installation des composants de base](./doc/base.md) 
- [Installation d'un serveur LAMP / LAPP](./doc/lamp.md)
- [Installation d'un serveur Git](./doc/git.md)
- [Installation d'un WebDav](./doc/webdav.md)
- [Installation d'un serveur d'envoi d'email](./doc/email.md)

## Applications

Le dossier `app` contient les fichiers docker-compose et de configuration pour différentes applications :

- Drone (Outil de CI/CD)
- Ferdium
- Gogs (Serveur Git)
- Umami (Analytics)

## Template

Le dossier `template` contient des fichiers d'exemple réutilisables pour différents services :

- Vhost Apache
- .htaccess
- Scripts utilitaires
- HTML/PHP

## Todo one day

- Serveur bitwarden
- [Service systemd](https://abhinand05.medium.com/run-any-executable-as-systemd-service-in-linux-21298674f66f)
- VPN Wireguard ou OpenVPN avec PiVPN
