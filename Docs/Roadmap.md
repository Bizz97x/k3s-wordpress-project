# Roadmap - Projet K3s / Traefik / WordPress / Vault / Observabilite / Hugo

> Version Markdown generee depuis `Docs/Roadmap.html`.

## Progression actuelle

- Taches completees: `14 / 29`
- Phases completees: `2 / 6`
- Etat global: `Phases 1 et 2 terminees`, `Phases 3 a 6 en cours`

## Phase 1 - Socle (Cluster K3s OK) - Terminee

- Priorite: Bloquant
- Validation avant suite du projet

### Taches

- [x] Preparer les 3 VM
  - swap off, reseau OK, ports, hostname, roles (VM1 server / VM2-VM3 agents)
- [x] Installer K3s (server + agents)
  - objectif: les 3 noeuds en `Ready`
- [x] Verifier Traefik
  - Traefik actif, dashboard accessible (8080), ecoute 80/443

### Critere de validation

- `kubectl get nodes` -> 3 `Ready`
- Traefik operationnel

## Phase 2 - WordPress en HTTP (simple) - Terminee

- Objectif: Fonctionnel
- HTTP avant HTTPS

### Taches

- [x] Creer le namespace `wordpress`
  - manifeste present dans `k8s/wordpress/namespace.yaml`
- [x] Creer Secret Kubernetes `v1`
  - manifeste present dans `k8s/wordpress/secret-db.yaml`
- [x] Preparer les manifests MariaDB
  - PVC `mariadb-pvc`, Deployment `mariadb`, Service `mariadb`
- [x] Preparer les manifests WordPress HTTP
  - Deployment `wordpress`, Service `wordpress`, Ingress HTTP `web.etna.student`, IngressRoute HTTP
- [x] Ajouter la persistance WordPress
  - manifeste `wordpress-pvc` present et monte dans le Deployment `wordpress`
- [x] Appliquer et valider la stack WordPress en HTTP
  - acces HTTP valide sur `web.etna.student`, persistance MariaDB et WordPress testee et validee

### Avancement

- Namespace `wordpress` cree (`k8s/wordpress/namespace.yaml`)
- Secret Kubernetes `wp-db-secret` cree (`k8s/wordpress/secret-db.yaml`)
- Manifeste PVC MariaDB cree (`k8s/wordpress/mariadb-pvc.yaml`)
- Manifeste Deployment MariaDB cree (`k8s/wordpress/mariadb-deployment.yaml`)
- Manifeste Service MariaDB cree (`k8s/wordpress/mariadb-service.yaml`)
- Manifeste Deployment WordPress cree (`k8s/wordpress/wordpress-deployment.yaml`)
- Manifeste Service WordPress cree (`k8s/wordpress/wordpress-services.yaml`)
- Manifeste Ingress HTTP WordPress cree (`k8s/wordpress/wordpress-ingress.yaml`)
- Manifeste IngressRoute HTTP / HTTPS WordPress cree (`k8s/wordpress/wordpress-ingressroute.yaml`)
- Manifeste PVC WordPress cree (`k8s/wordpress/wordpress-pvc.yaml`)
- Host HTTP actuel: `web.etna.student`
- WordPress accessible en HTTP via `web.etna.student`
- Persistance MariaDB et WordPress testee et validee
- La stack HTTP WordPress est fonctionnelle
- Note: Ingress et IngressRoute coexistent actuellement dans le dossier
- Prochaine etape: poursuivre la mise en place du HTTPS et sa validation

### Critere de validation

- WordPress accessible en HTTP
- Donnees persistantes (MariaDB + WordPress)

## Phase 3 - HTTPS Let's Encrypt - En cours

- Objectif: Certificat valide
- Traefik + Let's Encrypt

### Taches

- [x] Activer Let's Encrypt dans Traefik
  - configuration documentee via HelmChartConfig avec resolver `letsencrypt`
- [x] Ajouter IngressRoute HTTPS WordPress
  - manifeste present dans `k8s/wordpress/wordpress-ingressroute.yaml` avec `entryPoint websecure` et `certResolver letsencrypt`
- [ ] Tester la validite du certificat
  - implementation visible dans les manifests, mais non validable proprement en production avec `web.etna.student`

### Avancement

- Manifeste IngressRoute HTTPS WordPress cree (`k8s/wordpress/wordpress-ingressroute.yaml`)
- Resolver `letsencrypt` reference dans l'IngressRoute HTTPS
- ACME / Let's Encrypt documente via HelmChartConfig applique sur le cluster
- Limite: le domaine `web.etna.student` est traite comme domaine local / non public dans le contexte du projet
- Note: le HTTPS est implemente cote manifests, mais ne peut pas etre valide en production sans DNS public joignable

### Critere de validation

- WordPress accessible en `https://...`
- Certificat valide en production

## Phase 4 - Vault (secrets ameliores) - En cours

- Objectif: Rotation + generation aleatoire
- Helm + Injector

### Taches

- [x] Installer Vault via Helm (repo officiel)
- [ ] Initialiser / unseal + acces admin
  - documenter les commandes et le stockage des cles (hors Git)
- [ ] Configurer auth Kubernetes + policies
  - autoriser les pods a lire les secrets
- [ ] Configurer le secrets engine DB
  - users DB temporaires (secrets dynamiques)
- [ ] Mettre en place rotation hebdomadaire + generation aleatoire
  - preuve via TTL / renouvellement / nouveau user-password
- [ ] Modifier les deploiements WordPress/DB
  - utilisation Vault (injector/agent) a la place du Secret K8s `v1`

### Avancement

- Namespace `vault` cree via l'installation Helm
- Installation Helm Vault realisee
- Commande utilisee : `helm install vault hashicorp/vault --namespace vault --create-namespace --set "server.dev.enabled=true"`
- Vault actuellement deploye en mode `dev`
- Le Secret Kubernetes `wp-db-secret` reste utilise par les deploiements WordPress / MariaDB existants
- Note: le demarrage de la phase Vault est valide, mais la configuration finale n'est pas encore en place
- Limite: auth Kubernetes, policies, roles, injection des secrets, secrets dynamiques et rotation hebdomadaire ne sont pas encore visibles comme configures
- Prochaines etapes: verifier l'etat du pod Vault, configurer l'auth Kubernetes, creer policies et roles, stocker les secrets WordPress dans Vault, modifier les deploiements pour consommer Vault, puis mettre en place la rotation

### Critere de validation

- Pods relies a Vault
- Rotation weekly demontree

## Phase 5 - Observabilite - En cours

- Objectif: Dashboards + logs
- Grafana Explore

### Taches

- [x] Deployer Prometheus + Grafana (Helm)
  - collecte metrics cluster + nodes/VM
- [x] Deployer Loki + Promtail
  - composants deployes, validation Grafana Explore encore instable
- [ ] Configurer les datasources Grafana
  - Prometheus + Loki
- [ ] Construire un dashboard Kubernetes + VM
  - CPU (limite + reel), RAM (%), connexions TCP, objets K8s + etats
- [ ] Construire un dashboard services (WordPress)
  - requetes HTTP + codes (200/401/404/500...)
- [ ] Maitriser le filtrage des logs
  - ex: logs d'acces WordPress via labels/namespace/pod

### Avancement

- Phase observabilite demarree
- kube-prometheus-stack installe dans `monitoring`
- Prometheus installe
- Grafana installe et accessible
- Loki et Promtail deployes
- Loki configure via des values custom (`k8s/monitoring/loki-values.yaml` et `k8s/monitoring/loki-values-clean.yaml`)
- Le pod Loki a pu devenir temporairement `Running` / `Ready` et certains endpoints ont parfois repondu
- Les endpoints Loki (`/ready` et certaines requetes API comme `vector(1)+vector(1)`) repondent de maniere intermittente
- Commande utile pour recuperer le mot de passe admin du secret dans `monitoring` :
  - `kubectl get secret --namespace monitoring -l app.kubernetes.io/component=admin-secret -o jsonpath="{.items[0].data.admin-password}" | base64 --decode ; echo`
- Diagnostic: readiness Loki instable, erreurs gRPC `9095`, erreurs `scheduler/querier/ingester`, erreurs `502` via `loki-gateway`, refus de connexion intermittents via le service `loki`, et erreurs de timeout lors du healthcheck Grafana
- Conclusion: Grafana et Prometheus sont exploitables, mais Loki reste partiellement fonctionnel et non stable pour une validation complete dans Grafana Explore
- Prochaine etape: stabiliser Loki et finaliser la datasource / Explore dans Grafana

### Critere de validation

- Dashboards conformes
- Logs exploitables en soutenance

## Phase 6 - Documentation Hugo (service) - En cours

- Objectif: HTTPS + Basic Auth
- Documentation evaluee

### Taches

- [ ] Creer le site Hugo (theme au choix)
- [ ] Deployer Hugo sur le cluster
  - namespace `docs`, service + ingress
- [ ] Activer HTTPS Let's Encrypt
  - URL type `https://docs.etna.student`
- [ ] Mettre en place une authentification basique
  - middleware Traefik BasicAuth (credentials hors Git)
- [ ] Rediger une documentation par service
  - WordPress/DB, Traefik+LE, Vault, Observabilite, Hugo + liens repo

### Critere de validation

- Site de documentation accessible en HTTPS
- Authentification obligatoire

## Checklist soutenance

- [ ] Montrer: 3 nodes Ready + Traefik OK
- [ ] Montrer: WordPress HTTP + persistance (redemarrage pod)
- [ ] Montrer: HTTPS + certificat valide
- [ ] Montrer: Vault (status/unseal) + generation/rotation weekly
- [ ] Montrer: Dashboards Grafana + Explore logs (acces WordPress)
- [ ] Montrer: Hugo docs HTTPS + basic auth

## Conseils de suivi

- Faire un commit a chaque phase validee
- Mettre a jour les cases directement dans ce fichier pour garder un historique clair dans Git
