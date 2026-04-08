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

La phase Vault a ete demarree. Vault a ete installe avec Helm dans le namespace `vault` via la commande :

```bash
helm install vault hashicorp/vault --namespace vault --create-namespace --set "server.dev.enabled=true"
```

L'installation actuelle valide le demarrage de la phase Vault, mais elle repose sur un mode `dev` destine au bootstrap et aux tests. Cette installation ne correspond pas encore a une configuration finale d'exploitation.

Etat actuel documente :
- namespace `vault` cree
- installation Helm reussie
- Vault deploye en mode `server.dev.enabled=true`

Ce qui n'est pas encore documente comme realise :
- auth Kubernetes Vault
- policies et roles Vault
- injection des secrets WordPress via Vault
- remplacement du Secret Kubernetes `wp-db-secret`
- secrets dynamiques / rotation hebdomadaire

Note :
- un fichier `k8s/vault/values.yaml` est present dans le depot
- rien dans les fichiers analyses ne prouve qu'il est deja utilise par l'installation actuellement en place
- l'etat a documenter ici reste donc l'installation Helm en mode `dev`

## Observabilite

La phase observabilite a avance avec le namespace `monitoring` et l'installation de la stack kube-prometheus-stack. Dans l'etat actuel documente :

- Prometheus est installe
- Grafana est installe et accessible
- Promtail est deploye
- Loki est deploye via Helm avec des values custom

Les fichiers `k8s/monitoring/loki-values.yaml` et `k8s/monitoring/loki-values-clean.yaml` montrent une tentative de stabilisation de Loki en mode `SingleBinary` avec stockage `filesystem`, caches desactives et ajustements `memberlist`.

Etat reel a retenir :

- Prometheus et Grafana sont exploitables
- Loki et Promtail sont deployes
- le pod Loki a pu atteindre temporairement un etat `Running` / `Ready` et certains endpoints ont parfois repondu
- la validation complete de Loki dans Grafana Explore n'est pas stabilisee

Malgre un deploiement fonctionnel, Loki presente des instabilites (timeouts et erreurs de communication interne), empechant une validation complete dans Grafana Explore.

Le diagnostic a mis en evidence une instabilite persistante de Loki sur cet environnement :

- readiness Loki instable
- erreurs gRPC internes sur le port `9095`
- erreurs `scheduler` / `querier` / `ingester`
- erreurs `502` via `loki-gateway`
- refus de connexion intermittents via le service `loki`

## Limites

- La configuration Traefik / ACME via HelmChartConfig n'est pas visible dans `k8s/wordpress`
- Le domaine `web.etna.student` est traite comme domaine local / non public dans le contexte du projet
- Dans ce contexte, un certificat Let's Encrypt ne peut pas etre considere comme valide en production sans DNS public joignable
- Loki / Promtail sont deployes, mais la validation complete dans Grafana Explore reste instable sur l'environnement K3s utilise

## État d'avancement

- Phase 1 : socle K3s et Traefik documentes comme operationnels
- Phase 2 : stack WordPress / MariaDB deployee et validee en HTTP avec persistance
- Phase 3 : HTTPS implemente cote Traefik, mais non validable proprement en production avec `web.etna.student`
- Phase 4 : Vault installe avec Helm en mode `dev`, mais integration applicative non realisee
- Phase 5 : observabilite partiellement validee ; Prometheus et Grafana sont exploitables, Loki reste instable

## Prochaines étapes

- Finaliser l'observabilite (stabiliser Loki / Grafana Explore)
- Gestion des secrets (Vault)
- Documentation Hugo

## Arborescence principale

```text
group-1070802/
├── README.md
├── Docs/
│   ├── Roadmap.html
│   ├── Roadmap.md
│   └── cluster-setup.md
└── k8s/
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
