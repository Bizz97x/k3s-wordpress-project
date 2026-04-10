# Projet d'hébergement K3s

Ce projet met en place un cluster Kubernetes avec K3s, incluant :

- Déploiement WordPress
- HTTPS avec Let's Encrypt
- Gestion des secrets avec Vault
- Stack d'observabilité
- Service de documentation Hugo

## Architecture

Le projet s'appuie sur un cluster K3s a 3 noeuds avec Traefik en frontal. Les manifests actuellement presents dans `k8s/wordpress` decrivent une stack WordPress complete avec persistance et exposition HTTP/HTTPS via Traefik.

```text
Client
  |
  v
Traefik
  |- Ingress Kubernetes -> Service wordpress -> Deployment wordpress -> PVC wordpress-pvc
  |- IngressRoute HTTP  -> Service wordpress
  |- IngressRoute HTTPS -> Service wordpress
  |
  v
Service mariadb -> Deployment mariadb -> PVC mariadb-pvc
                     ^
                     |
                Secret wp-db-secret
```

## Déploiement

### Infrastructure

- Cluster K3s fonctionnel
- Cluster exploite sur 3 noeuds
- Traefik documente comme operationnel dans le projet
- Namespace applicatif `wordpress`

### Ressources WordPress / MariaDB

- Secret Kubernetes `wp-db-secret`
- PVC `mariadb-pvc`
- Deployment `mariadb`
- Service `mariadb` en `ClusterIP`
- PVC `wordpress-pvc`
- Deployment `wordpress`
- Service `wordpress` en `ClusterIP`

## Réseau

- Ingress Kubernetes `wordpress-ingress`
- IngressRoute Traefik HTTP `wordpress-http`
- IngressRoute Traefik HTTPS `wordpress-https`
- Host actuel : `web.etna.student`

L'Ingress standard Kubernetes et l'IngressRoute Traefik coexistent actuellement dans le dossier. Cela montre une transition vers IngressRoute, mais les fichiers seuls ne prouvent pas quelle ressource est effectivement utilisee en priorite sur le cluster.

## Configuration Traefik (HelmChartConfig)

Traefik est configure sur le cluster via un `HelmChartConfig` applique directement dans le repertoire systeme de K3s : `/var/lib/rancher/k3s/server/manifests`.

Cette configuration n'est pas versionnee dans ce repository, mais elle fait partie de l'infrastructure reelle du projet. Elle permet de completer le chart Traefik fourni par K3s avec les parametres necessaires au HTTPS.

Le `HelmChartConfig` est utilise pour :
- activer la configuration ACME
- declarer le resolver `letsencrypt`
- utiliser la persistance des donnees Traefik dans `/data`

Cette partie est importante car les manifests applicatifs, notamment l'IngressRoute HTTPS, s'appuient sur ce resolver pour demander un certificat. Sans cette configuration appliquee directement sur le cluster, la route HTTPS ne pourrait pas exploiter ACME.

## Sécurité

- Les credentials MariaDB sont portes par le Secret `wp-db-secret`
- L'IngressRoute HTTPS reference `certResolver: letsencrypt`
- La chaine HTTPS est implementee dans les manifests, mais pas validee en production dans la documentation actuelle

## Vault

La phase Vault a avance. Vault est installe et operationnel sur le cluster dans le namespace `vault`.

L'installation initiale a ete lancee avec Helm via la commande :

```bash
helm install vault hashicorp/vault --namespace vault --create-namespace --set "server.dev.enabled=true"
```

Etat actuel retenu :

- namespace `vault` cree
- installation Helm reussie
- pod `vault-0` en `Running`
- `Initialized: true`
- `Sealed: false`
- `vault-agent-injector` en `Running`

Un POC d'integration Vault est maintenant valide cote WordPress :

- un `ServiceAccount` `wordpress-sa` est present dans le namespace `wordpress`
- l'authentification Kubernetes de Vault est configuree
- une policy Vault et un role `wordpress-role` ont ete crees pour WordPress
- un secret WordPress est stocke dans Vault a l'emplacement `secret/wordpress/config`
- le Deployment WordPress utilise Vault Agent Injector pour generer le fichier `/vault/secrets/wp-env`
- le pod WordPress charge bien les variables `WORDPRESS_DB_HOST`, `WORDPRESS_DB_NAME`, `WORDPRESS_DB_USER` et `WORDPRESS_DB_PASSWORD` depuis Vault
- WordPress demarre correctement avec les variables injectees depuis Vault

Ce POC valide donc l'integration applicative Vault sur WordPress.

MariaDB ne doit en revanche pas etre presente comme migree completement vers Vault :

- une tentative d'integration a ete faite avec `mariadb-sa`, une policy, un role et un secret Vault dedie
- l'injection Vault cote MariaDB semble avoir ete amorcee, mais le pod MariaDB est tombe en `CrashLoopBackOff`
- l'echec observe est lie au demarrage du conteneur MariaDB (`mysqld: command not found`)
- pour conserver une stack fonctionnelle, MariaDB reste pour l'instant sur le Secret Kubernetes natif `wp-db-secret`

Note :
- un fichier `k8s/vault/values.yaml` est present dans le depot
- le fichier montre notamment `injector.enabled: true`
- cela reste coherent avec le POC WordPress valide

Ce qui reste a finaliser pour satisfaire completement la consigne Vault :

- integration complete de tous les deploiements applicatifs
- secrets dynamiques pour la base de donnees
- generation aleatoire des credentials
- rotation hebdomadaire des secrets

## Observabilite

La phase observabilite a avance avec le namespace `monitoring` et l'installation de la stack kube-prometheus-stack. Dans l'etat actuel documente :

- Prometheus est installe
- Grafana est installe et accessible
- La datasource Prometheus est presente dans Grafana
- Les requetes Prometheus fonctionnent dans Explore, notamment `up`
- Le dashboard `Node Exporter Full` (Grafana.com ID `1860`) a ete importe avec succes
- La supervision CPU / memoire / disque / reseau des noeuds est validee
- Promtail est deploye
- Loki est deploye via Helm avec des values custom

Les fichiers `k8s/monitoring/loki-values.yaml` et `k8s/monitoring/loki-values-clean.yaml` montrent une tentative de stabilisation de Loki en mode `SingleBinary` avec stockage `filesystem`, caches desactives et ajustements `memberlist`.

Etat reel a retenir :

- Prometheus est operationnel
- Grafana est operationnel
- la supervision metrique via Prometheus + Grafana est validee
- le dashboard `Node Exporter Full` constitue la base actuelle de demonstration de l'observabilite
- Loki et Promtail sont deployes
- le pod Loki a pu atteindre temporairement un etat `Running` / `Ready` et certains endpoints ont parfois repondu
- la validation complete des logs dans Grafana Explore n'est pas finalisee

Le dashboard Kubernetes plus generique teste precedemment ne remontait pas correctement les donnees attendues sur ce cluster. Le choix du dashboard `Node Exporter Full` est volontaire, car il est compatible avec les metriques effectivement collectees par `node-exporter`.

Malgre un deploiement fonctionnel, Loki presente des instabilites (timeouts et erreurs de communication interne), empechant une validation complete dans Grafana Explore.

Le diagnostic a mis en evidence une instabilite persistante de Loki sur cet environnement :

- readiness Loki instable
- erreurs gRPC internes sur le port `9095`
- erreurs `scheduler` / `querier` / `ingester`
- erreurs `502` via `loki-gateway`
- refus de connexion intermittents via le service `loki`

## Documentation Hugo

La documentation du projet est generee avec Hugo sous forme de site statique.

Etat actuel retenu :

- le site Hugo est genere localement dans `Docs/hugo-site`
- aucun theme externe n'est utilise
- des layouts HTML et un CSS custom ont ete ecrits pour rester compatibles avec Hugo `0.111.x`
- l'interface a ete amelioree pour obtenir un rendu de documentation technique plus lisible
- les sections actuellement documentees sont `cluster`, `wordpress`, `vault`, `observabilite` et `hugo`

La generation du site se fait via :

```bash
hugo
```

Le dossier de sortie utilise est `public/`.

### Deploiement

Le site statique est servi sur le cluster par nginx.

- namespace `docs`
- Deployment `hugo-docs`
- Service `hugo-docs` en `ClusterIP`
- IngressRoute Traefik HTTP et HTTPS sur `docs.etna.student`

Le contenu de `Docs/hugo-site/public/` est copie dans le pod nginx. Il n'y a pas d'image Docker custom pour ce service.

### Securite

- Middleware Traefik Basic Auth `docs-basicauth`
- Secret Kubernetes `docs-basic-auth`
- authentification obligatoire pour acceder au site

### HTTPS

- IngressRoute HTTPS dediee sur l'entryPoint `websecure`
- `certResolver: letsencrypt`
- HTTPS implemente cote Traefik

Dans le contexte du projet, le certificat ne peut toutefois pas etre considere comme forcement valide en environnement local ou sans DNS public joignable.

### Debug et rendu

Un point de debug a ete documente pendant la mise en place du site :

- une image Grafana etait bien presente dans `static/images`, bien copiee dans le pod et correctement servie par nginx
- le probleme venait du rendu Hugo : une balise HTML `<img>` ecrite dans le Markdown etait supprimee au build
- la correction retenue a ete d'utiliser une image Markdown standard plutot que d'activer le rendu HTML non securise
- ce choix conserve une configuration Hugo simple et compatible avec l'environnement actuel

## Limites

- La configuration Traefik / ACME via HelmChartConfig n'est pas visible dans `k8s/wordpress`
- Le domaine `web.etna.student` est traite comme domaine local / non public dans le contexte du projet
- Dans ce contexte, un certificat Let's Encrypt ne peut pas etre considere comme valide en production sans DNS public joignable
- Loki / Promtail sont deployes, mais la validation complete dans Grafana Explore reste instable sur l'environnement K3s utilise
- le service Hugo utilise un volume `emptyDir`
- le contenu de `public/` doit donc etre recopie dans le pod apres recreation ou redemarrage
- il n'y a pas de pipeline de build d'image pour embarquer automatiquement le site statique
- il n'y a pas de CI/CD pour regenerer et republier automatiquement la documentation

## État d'avancement

- Phase 1 : socle K3s et Traefik documentes comme operationnels
- Phase 2 : stack WordPress / MariaDB deployee et validee en HTTP avec persistance
- Phase 3 : HTTPS implemente cote Traefik, mais non validable proprement en production avec `web.etna.student`
- Phase 4 : Vault operationnel, POC WordPress valide, mais integration complete et rotation non finalisees
- Phase 5 : observabilite partiellement validee ; la supervision metrique Prometheus / Grafana est validee, mais Loki reste instable
- Phase 6 : documentation Hugo fonctionnelle ; site statique genere, servi par nginx, protege par Basic Auth et expose en HTTP / HTTPS, avec des limites liees a `emptyDir` et au contexte DNS local

## Prochaines étapes

- Finaliser l'observabilite (stabiliser Loki / Grafana Explore)
- Gestion des secrets (Vault)
- Industrialiser le deploiement de la documentation Hugo si une persistance ou une image dediee devient necessaire

## Arborescence principale

```text
group-1070802/
├── README.md
├── Docs/
│   ├── Roadmap.html
│   ├── Roadmap.md
│   └── cluster-setup.md
└── k8s/
    ├── docs/
    │   ├── docs-deployment.yaml
    │   ├── docs-ingressroute-https.yaml
    │   ├── docs-ingressroute.yaml
    │   ├── docs-middleware-basicauth.yaml
    │   ├── docs-service.yaml
    │   └── namespace.yaml
    ├── monitoring/
    │   ├── loki-values-clean.yaml
    │   └── loki-values.yaml
    ├── vault/
    │   └── values.yaml
    └── wordpress/
        ├── mariadb-deployment.yaml
        ├── mariadb-pvc.yaml
        ├── mariadb-service.yaml
        ├── namespace.yaml
        ├── secret-db.exemple.yaml
        ├── secret-db.yaml
        ├── wordpress-deployment.yaml
        ├── wordpress-ingress.yaml
        ├── wordpress-ingressroute.yaml
        ├── wordpress-pvc.yaml
        └── wordpress-services.yaml
```

## Documentation

- La roadmap du projet est dans `Docs/Roadmap.md`
- La documentation d'installation et de déploiement est dans `Docs/cluster-setup.md`
