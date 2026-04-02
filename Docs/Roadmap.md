# Roadmap - Projet K3s / Traefik / WordPress / Vault / Observabilite / Hugo

> Version Markdown generee depuis `Docs/Roadmap.html`.

## Progression actuelle

- Taches completees: `8 / 29`
- Phases completees: `1 / 6`
- Etat global: `Phase 1 terminee`, `Phases 2 a 6 en cours`

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

## Phase 2 - WordPress en HTTP (simple) - En cours

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
  - Deployment `wordpress`, Service `wordpress`, Ingress HTTP `web.etna.student`
- [x] Ajouter la persistance WordPress
  - manifeste `wordpress-pvc` present et monte dans le Deployment `wordpress`
- [ ] Appliquer et valider la stack WordPress en HTTP
  - aucune preuve visible dans les fichiers d'un deploiement teste et valide en HTTP

### Avancement

- Namespace `wordpress` cree (`k8s/wordpress/namespace.yaml`)
- Secret Kubernetes `wp-db-secret` cree (`k8s/wordpress/secret-db.yaml`)
- Manifeste PVC MariaDB cree (`k8s/wordpress/mariadb-pvc.yaml`)
- Manifeste Deployment MariaDB cree (`k8s/wordpress/mariadb-deployment.yaml`)
- Manifeste Service MariaDB cree (`k8s/wordpress/mariadb-service.yaml`)
- Manifeste Deployment WordPress cree (`k8s/wordpress/wordpress-deployment.yaml`)
- Manifeste Service WordPress cree (`k8s/wordpress/wordpress-services.yaml`)
- Manifeste Ingress HTTP WordPress cree (`k8s/wordpress/wordpress-ingress.yaml`)
- Manifeste PVC WordPress cree (`k8s/wordpress/wordpress-pvc.yaml`)
- Host HTTP actuel: `web.etna.student`
- Note: les manifests existent, mais aucune validation fonctionnelle HTTP n'est visible dans les fichiers
- Prochaine etape: appliquer les manifests puis tester l'acces HTTP

### Critere de validation

- WordPress accessible en HTTP
- Donnees persistantes (MariaDB + WordPress)

## Phase 3 - HTTPS Let's Encrypt - En cours

- Objectif: Certificat valide
- Traefik + Let's Encrypt

### Taches

- [ ] Activer Let's Encrypt dans Traefik
  - ConfigMap + email ETNA + copie dans `/var/lib/rancher/k3s/server/manifests/...`
- [ ] Ajouter IngressRoute HTTPS WordPress
  - host + resolver LE, port 443
- [ ] Tester la validite du certificat
  - cadenas navigateur OK, certificat reconnu

### Critere de validation

- WordPress accessible en `https://...`
- Certificat valide

## Phase 4 - Vault (secrets ameliores) - En cours

- Objectif: Rotation + generation aleatoire
- Helm + Injector

### Taches

- [ ] Installer Vault via Helm (repo officiel)
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

### Critere de validation

- Pods relies a Vault
- Rotation weekly demontree

## Phase 5 - Observabilite - En cours

- Objectif: Dashboards + logs
- Grafana Explore

### Taches

- [ ] Deployer Prometheus + Grafana (Helm)
  - collecte metrics cluster + nodes/VM
- [ ] Deployer Loki + Promtail
  - logs visibles dans Grafana / Explore
- [ ] Configurer les datasources Grafana
  - Prometheus + Loki
- [ ] Construire un dashboard Kubernetes + VM
  - CPU (limite + reel), RAM (%), connexions TCP, objets K8s + etats
- [ ] Construire un dashboard services (WordPress)
  - requetes HTTP + codes (200/401/404/500...)
- [ ] Maitriser le filtrage des logs
  - ex: logs d'acces WordPress via labels/namespace/pod

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
