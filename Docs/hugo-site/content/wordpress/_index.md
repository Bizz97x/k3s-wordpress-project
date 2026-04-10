---
title: "WordPress"
---

# WordPress

Cette section regroupe la documentation applicative autour du namespace `wordpress`, de la stack WordPress / MariaDB, de la persistance et de l'exposition HTTP et HTTPS telle qu'elle est decrite dans les documents existants.

## Namespace et composants

La stack applicative est rassemblee dans le namespace `wordpress`.

Les ressources actuellement documentees sont les suivantes :

- Secret Kubernetes `wp-db-secret`
- PVC `mariadb-pvc`
- Deployment `mariadb`
- Service `mariadb`
- PVC `wordpress-pvc`
- Deployment `wordpress`
- Service `wordpress`
- Ingress Kubernetes `wordpress-ingress`
- IngressRoute Traefik `wordpress-http`
- IngressRoute Traefik `wordpress-https`

## Base de donnees MariaDB

MariaDB fait partie de la stack applicative de base.

- le Deployment `mariadb` porte la base de donnees
- le Service `mariadb` est expose en `ClusterIP`
- le port documente est `3306`
- les credentials utilises par la stack fonctionnelle sont portes par le Secret `wp-db-secret`

## Deploiement WordPress

Le deploiement WordPress repose sur :

- le Deployment `wordpress`
- le Service `wordpress` en `ClusterIP`
- le port `80` pour l'exposition interne
- le host actuel `web.etna.student`

La documentation indique que WordPress est accessible en HTTP via `web.etna.student`.

## Persistance

La persistance est documentee des deux cotes de la stack :

- `mariadb-pvc` pour les donnees MariaDB
- `wordpress-pvc` pour les fichiers WordPress

La roadmap indique explicitement que la persistance MariaDB et WordPress a ete testee et validee dans la phase HTTP.

## Exposition reseau

### Ingress HTTP

L'exposition HTTP standard est decrite via l'Ingress Kubernetes `wordpress-ingress` :

- host `web.etna.student`
- backend `wordpress:80`

### IngressRoute HTTP

Une route Traefik `wordpress-http` est aussi documentee :

- entryPoint `web`
- host `web.etna.student`
- backend `wordpress:80`

### IngressRoute HTTPS

Une route Traefik `wordpress-https` est egalement presente :

- entryPoint `websecure`
- host `web.etna.student`
- backend `wordpress:80`
- TLS avec `certResolver: letsencrypt`

## Etat actuel

La documentation permet de retenir les points suivants comme valides :

- le namespace `wordpress` est cree
- la stack WordPress / MariaDB est deployee
- WordPress est valide en HTTP
- la persistance MariaDB et WordPress est validee
- les Services `wordpress` et `mariadb` existent en `ClusterIP`
- les manifests HTTP et HTTPS sont presents

## Limites et points de vigilance

- `Ingress` et `IngressRoute` coexistent dans `k8s/wordpress`
- cette coexistence montre une transition vers `IngressRoute`, mais les fichiers seuls ne permettent pas d'affirmer quelle ressource est prioritaire sur le cluster
- le HTTPS est implemente dans les manifests, mais la validite du certificat n'est pas demontree en production
- le domaine `web.etna.student` est traite comme domaine local ou non public dans le contexte du projet
- dans ce contexte, le HTTPS ne peut pas etre considere comme valide en production sans DNS public joignable
