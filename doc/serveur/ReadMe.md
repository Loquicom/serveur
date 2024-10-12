# Installation d'un serveur web et d'un SGBD

Dans ce dossier se trouvent tous les documents pour installer un serveur web et le sécuriser, ainsi que les documents décrivant l'installation d'un SGBD pouvant être utilisé par les applications hébergées.

Plusieurs serveurs web sont documentés, il faut en installer qu'un seul. Pour les SGBD, il est possible d'en installer plusieurs sur la machine. Les documentations évoquent l'installation directement sur le serveur des SGBD et non en passant par Docker. Si votre application utilise un SGBD par Docker (ou n'en utilise pas), il n'est pas nécessaire d'en installer un pour le bon fonctionnement du serveur web.

## Sommaire

- Serveur web
  - [Apache](./apache.md)
  - [Caddy](./caddy.md)
- Sécurité
  - [Crowdsec](./crowdsec.md)
  - [Certbot](./certbot.md)
- SGBD
  - [MySQL](./mysql.md)
  - [MariaDB](./mariadb.md)
  - [PostgreSQL](./postgre.md)