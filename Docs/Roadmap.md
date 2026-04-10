# Roadmap - Projet K3s / Traefik / WordPress / Vault / Observabilite / Hugo

> Version Markdown generee depuis `Docs/Roadmap.html`.

## Progression actuelle

- Taches completees: `27 / 34`
- Phases completees: `3 / 6`
- Etat global: `Phases 1, 2 et 6 fonctionnelles`, `Phases 3 a 5 en cours`

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
- [x] Initialiser / unseal + verifier l'etat du pod Vault
  - `vault-0` en `Running`, `Initialized: true`, `Sealed: false`
- [x] Configurer auth Kubernetes + policies
  - role `wordpress-role` et acces WordPress au secret Vault
- [x] Valider un POC d'injection Vault sur WordPress
  - `wordpress-sa`, Vault Agent Injector et fichier `/vault/secrets/wp-env`
- [ ] Configurer le secrets engine DB
  - users DB temporaires (secrets dynamiques)
- [ ] Mettre en place rotation hebdomadaire + generation aleatoire
  - preuve via TTL / renouvellement / nouveau user-password
- [ ] Finaliser l'integration Vault de tous les deploiements
  - MariaDB reste provisoirement sur le Secret K8s `wp-db-secret`

### Avancement

- Namespace `vault` cree via l'installation Helm
- Installation Helm Vault realisee
- Commande utilisee : `helm install vault hashicorp/vault --namespace vault --create-namespace --set "server.dev.enabled=true"`
- Vault operationnel sur le cluster
- Pod `vault-0` en `Running`, `Initialized: true`, `Sealed: false`
- `vault-agent-injector` en `Running`
- Authentification Kubernetes Vault configuree
- Policy Vault et role `wordpress-role` crees pour WordPress
- Secret WordPress cree dans Vault a l'emplacement `secret/wordpress/config`
- ServiceAccount `wordpress-sa` cree dans le namespace `wordpress`
- Deployment WordPress modifie pour utiliser Vault Agent Injector
- POC valide : le pod WordPress genere `/vault/secrets/wp-env` avec `WORDPRESS_DB_HOST`, `WORDPRESS_DB_NAME`, `WORDPRESS_DB_USER` et `WORDPRESS_DB_PASSWORD`, puis demarre correctement
- MariaDB reste pour l'instant sur le Secret Kubernetes `wp-db-secret`
- Note: une tentative d'integration Vault a ete faite cote MariaDB avec `mariadb-sa`, policy, role et secret dedie, mais le pod est tombe en `CrashLoopBackOff` a cause d'un probleme de demarrage (`mysqld: command not found`)
- Limite: l'integration complete WordPress + MariaDB, les secrets dynamiques DB, la generation aleatoire et la rotation hebdomadaire ne sont pas encore finalises
- Prochaines etapes: stabiliser l'approche MariaDB avec Vault, finaliser le secrets engine DB, puis mettre en place la rotation hebdomadaire et la generation aleatoire

### Critere de validation

- Pods relies a Vault
- Rotation weekly demontree

## Phase 5 - Observabilite - En cours

- Objectif: Dashboards + logs
- Grafana Explore

### Taches

- [x] Deployer Prometheus + Grafana (Helm)
  - collecte metrics cluster + nodes/VM
- [x] Valider la datasource Prometheus dans Grafana
  - requetes Explore fonctionnelles, notamment `up`
- [x] Importer un dashboard de supervision metrique des noeuds
  - dashboard `Node Exporter Full` (Grafana.com ID `1860`)
- [x] Valider la visualisation CPU / memoire / disque / reseau
  - supervision exploitable pour `vm-server`, `vm2-agent` et `vm3-agent`
- [x] Deployer Loki + Promtail
  - composants deployes, validation Grafana Explore encore instable
- [ ] Stabiliser la datasource Loki dans Grafana Explore
  - healthcheck et requetes encore instables
- [ ] Construire un dashboard services (WordPress)
  - requetes HTTP + codes (200/401/404/500...)
- [ ] Maitriser le filtrage des logs
  - ex: logs d'acces WordPress via labels/namespace/pod

### Avancement

- Phase observabilite demarree
- kube-prometheus-stack installe dans `monitoring`
- Prometheus installe
- Grafana installe et accessible
- Datasource Prometheus presente dans Grafana
- Les requetes Prometheus fonctionnent dans Explore, notamment `up`
- Dashboard `Node Exporter Full` importe avec succes (Grafana.com ID `1860`)
- Visualisation CPU / memoire / disque / reseau validee pour `vm-server`, `vm2-agent` et `vm3-agent`
- Loki et Promtail deployes
- Loki configure via des values custom (`k8s/monitoring/loki-values.yaml` et `k8s/monitoring/loki-values-clean.yaml`)
- Le pod Loki a pu devenir temporairement `Running` / `Ready` et certains endpoints ont parfois repondu
- Les endpoints Loki (`/ready` et certaines requetes API comme `vector(1)+vector(1)`) repondent de maniere intermittente
- Commande utile pour recuperer le mot de passe admin du secret dans `monitoring` :
  - `kubectl get secret --namespace monitoring -l app.kubernetes.io/component=admin-secret -o jsonpath="{.items[0].data.admin-password}" | base64 --decode ; echo`
- La supervision metrique via Prometheus + Grafana est validee
- Note: le dashboard Kubernetes plus generique teste auparavant n'etait pas suffisamment compatible avec les metriques remontees sur ce cluster ; le choix de `Node Exporter Full` est volontaire pour la soutenance et le debug
- Diagnostic: readiness Loki instable, erreurs gRPC `9095`, erreurs `scheduler/querier/ingester`, erreurs `502` via `loki-gateway`, refus de connexion intermittents via le service `loki`, et erreurs de timeout lors du healthcheck Grafana
- Conclusion: Grafana et Prometheus sont exploitables, mais Loki reste partiellement fonctionnel et non stable pour une validation complete dans Grafana Explore
- Prochaine etape: stabiliser Loki et finaliser la datasource / Explore dans Grafana

### Critere de validation

- Dashboards conformes
- Logs exploitables en soutenance

## Phase 6 - Documentation Hugo (service) - Fonctionnelle avec limites

- Objectif: HTTPS + Basic Auth
- Documentation evaluee

### Taches

- [x] Creer le site Hugo
  - layouts custom utilises a la place d'un theme externe
- [x] Generer le site statique
  - generation locale via `hugo` vers `Docs/hugo-site/public/`
- [x] Deployer nginx sur le cluster
  - Deployment `hugo-docs` dans le namespace `docs`
- [x] Exposer le site via Service + IngressRoute HTTP
  - Service `ClusterIP` + host `docs.etna.student`
- [x] Mettre en place une authentification basique
  - middleware Traefik BasicAuth avec secret `docs-basic-auth`
- [x] Activer HTTPS Let's Encrypt
  - IngressRoute `websecure` avec `certResolver: letsencrypt`
- [x] Rediger une documentation par service
  - pages `cluster`, `wordpress`, `vault`, `observabilite` et `hugo`

### Avancement

- Site Hugo present dans `Docs/hugo-site`
- Aucun theme externe retenu
- Choix de layouts custom et d'un CSS local pour rester compatibles avec Hugo `0.111.x`
- Generation du site validee avec sortie dans `public/`
- Documentation statique servie par nginx via le Deployment `hugo-docs`
- Namespace `docs` cree
- Service `hugo-docs` expose en `ClusterIP`
- IngressRoute HTTP sur `docs.etna.student`
- Middleware Traefik `docs-basicauth` present
- Secret Kubernetes `docs-basic-auth` utilise pour l'authentification
- IngressRoute HTTPS implementee sur `websecure`
- `certResolver: letsencrypt` reference sur la route HTTPS
- Acces HTTP et HTTPS valides avec authentification
- Les pages de documentation principales sont redigees
- Une page Hugo dediee a la documentation du service est ajoutee au site lui-meme
- Un debug specifique a ete mene sur l'image Grafana : image statique correcte, mais balise HTML supprimee au rendu Hugo ; correction via image Markdown standard
- Le rendu visuel du site a ete ameliore avec une interface sobre de documentation technique

### Notes

- L'usage d'un theme a ete evite volontairement
- La contrainte principale est la compatibilite avec Hugo `0.111.x`
- Une solution simple a base de layouts HTML custom a ete privilegiee pour garantir la generation du site
- Ce choix technique est assume pour privilegier la compatibilite et la maitrise du rendu plutot qu'un theme tiers

### Limites

- le service nginx utilise un volume `emptyDir`
- le contenu de `public/` doit etre recopie apres recreation du pod
- aucune image custom ni pipeline de build n'embarque actuellement le site statique
- le HTTPS est implemente, mais la validite d'un certificat reste dependante du contexte DNS du projet
- il n'y a pas de CI/CD pour reconstruire et republier automatiquement la documentation

### Critere de validation

- Site de documentation accessible en HTTPS
- Authentification obligatoire

## Checklist soutenance

- [ ] Montrer: 3 nodes Ready + Traefik OK
- [ ] Montrer: WordPress HTTP + persistance (redemarrage pod)
- [ ] Montrer: HTTPS + certificat valide
- [ ] Montrer: Vault (status/unseal) + generation/rotation weekly
- [ ] Montrer: Dashboards Grafana + Explore logs (acces WordPress)
- [x] Montrer: Hugo docs HTTPS + basic auth

## Conseils de suivi

- Faire un commit a chaque phase validee
- Mettre a jour les cases directement dans ce fichier pour garder un historique clair dans Git
