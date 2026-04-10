---
title: "Vault"
---

# Vault

Cette section decrit l'etat actuel de Vault sur le cluster, son mode de deploiement, son integration avec WordPress et les limites encore ouvertes pour la gestion des secrets.

## Installation

Vault a ete installe sur le cluster avec Helm dans le namespace `vault` via la commande suivante :

```bash
helm install vault hashicorp/vault --namespace vault --create-namespace --set "server.dev.enabled=true"
```

Cette commande correspond a l'installation initiale documentee dans le projet.

## Mode de fonctionnement actuel

La documentation precise que Vault est actuellement deploie en mode `dev`. Ce mode a servi a lancer les premiers tests d'integration applicative, mais ne vaut pas finalisation complete de la phase Vault.

## Etat actuel

Les elements suivants sont documentes comme observes sur le cluster :

- le namespace `vault` est cree
- l'installation Helm est reussie
- le pod `vault-0` est en `Running`
- `Initialized: true`
- `Sealed: false`
- `vault-agent-injector` est en `Running`

Un fichier `k8s/vault/values.yaml` est aussi mentionne dans le depot, avec notamment `injector.enabled: true`.

## Lien avec WordPress et les secrets

Le lien actuellement valide entre Vault et la stack WordPress est un POC d'injection de secrets :

1. un `ServiceAccount` `wordpress-sa` est present dans le namespace `wordpress`
2. l'authentification Kubernetes de Vault est configuree
3. une policy Vault et un role `wordpress-role` ont ete crees
4. un secret WordPress est stocke dans Vault a l'emplacement `secret/wordpress/config`
5. le Deployment WordPress utilise Vault Agent Injector
6. Vault genere le fichier `/vault/secrets/wp-env`
7. WordPress charge `WORDPRESS_DB_HOST`, `WORDPRESS_DB_NAME`, `WORDPRESS_DB_USER` et `WORDPRESS_DB_PASSWORD` depuis ce fichier

La documentation considere ce POC comme valide cote WordPress : le pod demarre correctement avec les variables injectees depuis Vault.

Note technique : dans le manifeste WordPress, le chemin lu par Vault Agent Injector est `secret/data/wordpress/config`, ce qui correspond a la lecture KV v2 du secret stocke sous `secret/wordpress/config`.

## Ce qui n'est pas encore configure ou finalise

La phase Vault n'est pas consideree comme terminee. Les points suivants restent ouverts dans la documentation :

- integration complete de tous les deploiements applicatifs
- configuration du secrets engine database
- secrets dynamiques pour la base de donnees
- generation aleatoire des credentials
- rotation hebdomadaire des secrets

## Limites actuelles

MariaDB ne doit pas etre presente comme completement migree vers Vault.

La documentation mentionne :

- une tentative d'integration avec `mariadb-sa`, une policy, un role et un secret dedie
- une injection Vault amorcee cote MariaDB
- un echec du pod MariaDB en `CrashLoopBackOff`
- une erreur observee au demarrage : `mysqld: command not found`

Pour conserver une stack fonctionnelle, MariaDB reste donc pour l'instant sur le Secret Kubernetes natif `wp-db-secret`.

## Prochaines etapes realistes

Les prochaines etapes deja documentees sont les suivantes :

- stabiliser l'approche MariaDB avec Vault sans casser le demarrage du conteneur
- finaliser l'integration Vault sur l'ensemble de la stack applicative
- configurer le secrets engine database
- mettre en place la rotation hebdomadaire et les secrets dynamiques
