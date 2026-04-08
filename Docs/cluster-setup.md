# Cluster Setup

## Infrastructure

| VM | Role | Ressources |
| --- | --- | --- |
| VM-SERVER | `k3s-server` | 2 vCPU / 4 GB RAM |
| VM2-Agent | `k3s-agent-1` | 1 vCPU / 1 GB RAM |
| VM3-Agent | `k3s-agent-2` | 1 vCPU / 1 GB RAM |

## Installation de K3s

### 1. Installer le serveur (VM1)

```bash
curl -sfL https://get.k3s.io | sh -
```

### 2. Récupérer le token sur VM1

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

### 3. Installer les agents (VM2 et VM3)

```bash
curl -sfL https://get.k3s.io | \
K3S_URL=https://IP_VM1:6443 \
K3S_TOKEN=<TOKEN> \
sh -
```

## Vérification du cluster

```bash
kubectl get nodes
```

Résultat attendu: tous les nœuds sont en statut `Ready`.

Exemple:

```text
NAME           STATUS   ROLES                  AGE   VERSION
vm-server      Ready    control-plane,master   ...   ...
vm2-agent      Ready    <none>                 ...   ...
vm3-agent      Ready    <none>                 ...   ...
```

## Traefik

Traefik est installé automatiquement avec K3s par défaut.

### Vérification

```bash
kubectl get pods -A
kubectl get svc traefik -n kube-system
```

Résultat attendu: un pod Traefik actif dans le namespace `kube-system`

### Configuration Traefik (HelmChartConfig)

Dans ce projet, la configuration avancee de Traefik ne se limite pas aux manifests presents dans le repository. Le chart Traefik fourni par K3s est complete via un `HelmChartConfig` applique directement sur le cluster, dans `/var/lib/rancher/k3s/server/manifests`.

Cette configuration n'apparait donc pas dans `k8s/wordpress`, mais elle fait partie de l'infrastructure reelle utilisee pour l'exposition HTTPS.

Le `HelmChartConfig` sert ici a :

- activer ACME
- declarer le resolver `letsencrypt`
- configurer la persistance des donnees Traefik dans `/data`

Le lien avec le HTTPS est direct : l'IngressRoute `wordpress-https` reference `certResolver: letsencrypt`, ce qui suppose que ce resolver soit deja configure cote Traefik. Sans le `HelmChartConfig` applique sur le cluster, la partie HTTPS ne peut pas fonctionner comme prevu.

## Architecture actuelle

L'architecture actuelle du projet repose sur un namespace unique `wordpress` qui contient l'ensemble des ressources applicatives.

- Namespace
  - `wordpress`
- Base de donnees MariaDB
  - Secret `wp-db-secret` pour les variables de connexion
  - PVC `mariadb-pvc` pour la persistance des donnees MariaDB
  - Deployment `mariadb`
  - Service `mariadb` expose en interne sur le port `3306`
- Application WordPress
  - PVC `wordpress-pvc` pour la persistance des fichiers WordPress
  - Deployment `wordpress`
  - Service `wordpress` expose en interne sur le port `80`
  - Ingress `wordpress-ingress` gere par Traefik
  - IngressRoute `wordpress-http` pour l'entree `web`
  - IngressRoute `wordpress-https` pour l'entree `websecure`
- Exposition HTTP
  - host actuel: `web.etna.student`
  - routage HTTP via l'Ingress Kubernetes vers le Service `wordpress`

## Reseau

### Ressources exposees

- Service `mariadb`
  - type `ClusterIP`
  - port `3306`
- Service `wordpress`
  - type `ClusterIP`
  - port `80`
- Ingress Kubernetes `wordpress-ingress`
  - host `web.etna.student`
  - backend `wordpress:80`
- IngressRoute Traefik `wordpress-http`
  - entryPoint `web`
  - host `web.etna.student`
  - backend `wordpress:80`
- IngressRoute Traefik `wordpress-https`
  - entryPoint `websecure`
  - host `web.etna.student`
  - backend `wordpress:80`
  - TLS avec `certResolver: letsencrypt`

### Ingress vs IngressRoute

- `Ingress` est la ressource reseau standard Kubernetes
- `IngressRoute` est une CRD specifique a Traefik
- Les deux approches coexistent actuellement dans le dossier `k8s/wordpress`
- Cette coexistence montre une migration vers IngressRoute, mais les fichiers seuls ne permettent pas d'affirmer quelle ressource est effectivement utilisee en priorite sur le cluster

## Securite

- Secret Kubernetes `wp-db-secret` pour les credentials MariaDB / WordPress
- Services internes exposes en `ClusterIP`
- Route HTTPS declaree via `wordpress-https`
- Resolver `letsencrypt` reference dans l'IngressRoute HTTPS

## Limites

- La configuration Traefik / ACME via HelmChartConfig n'est pas visible dans `k8s/wordpress` ni dans `Docs/`
- Le domaine `web.etna.student` est traite comme domaine local / non public dans le contexte du projet
- Dans ce contexte, le HTTPS est implemente cote manifests mais ne peut pas etre considere comme valide en production sans DNS public joignable

## Vault (Phase 4)

### Installation actuelle

Vault a ete installe sur le cluster avec Helm dans le namespace `vault` via la commande suivante :

```bash
helm install vault hashicorp/vault --namespace vault --create-namespace --set "server.dev.enabled=true"
```

Cette commande valide :

- la creation du namespace `vault`
- l'installation Helm
- le deploiement initial de Vault
- un demarrage en mode `dev` pour les premiers tests

### Limites actuelles de cette installation

L'installation actuelle ne doit pas etre consideree comme finale. Le mode `server.dev.enabled=true` sert au bootstrap et aux tests, mais il ne valide pas encore une mise en place plus realiste pour la suite du projet.

Points non encore visibles comme configures dans les fichiers :

- auth Kubernetes Vault
- policies Vault
- roles Vault
- secrets WordPress stockes dans Vault
- modification des deploiements WordPress / MariaDB pour consommer Vault
- rotation hebdomadaire ou secrets dynamiques
- remplacement du Secret Kubernetes `wp-db-secret`

### Lien avec le depot

- un fichier `k8s/vault/values.yaml` est present dans le depot
- rien dans les fichiers analyses ne prouve qu'il est deja utilise par l'installation actuellement deployee
- la documentation doit donc refleter en priorite l'installation Helm en mode `dev`

### Prochaines etapes realistes

- verifier l'etat du pod Vault
- configurer l'auth Kubernetes
- creer les policies et roles Vault
- stocker les secrets WordPress dans Vault
- modifier les deploiements WordPress / MariaDB pour consommer Vault
- mettre en place la rotation hebdomadaire et les secrets dynamiques

## Observabilite (Phase 5)

### Etat actuel

La phase observabilite est en cours et deja partiellement validee.

- kube-prometheus-stack est installe dans le namespace `monitoring`
- Prometheus est installe
- Grafana est installe et accessible
- Promtail est deploye
- Loki est deploye via Helm avec des values custom
- le cluster K3s utilise reste un cluster a 3 noeuds

Les fichiers `k8s/monitoring/loki-values.yaml` et `k8s/monitoring/loki-values-clean.yaml` montrent un travail de stabilisation de Loki en mode `SingleBinary` avec stockage `filesystem`, caches desactives et ajustements `memberlist`.

### Recuperation du mot de passe Grafana

La commande suivante permet de recuperer le mot de passe admin Grafana apres deploiement :

```bash
kubectl get secret --namespace monitoring -l app.kubernetes.io/component=admin-secret -o jsonpath="{.items[0].data.admin-password}" | base64 --decode ; echo
```

### Etat Loki / Grafana Explore

Loki et Promtail ont ete deployes dans le namespace `monitoring`. L'ingestion de logs a ete partiellement observee, le pod Loki a pu devenir temporairement `Running` / `Ready` et certains endpoints ont parfois repondu, mais la validation complete de la datasource Loki dans Grafana reste instable sur l'environnement K3s utilise.

Les endpoints Loki (`/ready` et certaines requetes API comme `vector(1)+vector(1)`) repondent de maniere intermittente. Cependant, Grafana echoue regulierement lors du healthcheck avec des erreurs de timeout. Le service Loki est donc fonctionnel partiellement mais non stable pour une validation complete.

### Diagnostic Loki reel

Problemes rencontres :

- erreurs readiness probe (`503` / `timeout`)
- erreurs gRPC port `9095` (`DeadlineExceeded`)
- erreurs `scheduler` / `querier` / `ingester`
- timeouts lors des requetes depuis Grafana
- erreurs Grafana : `Unable to connect with Loki`
- erreurs Grafana : `Client.Timeout exceeded while awaiting headers`
- erreurs `502` via `loki-gateway`
- refus de connexion intermittents via le service `loki`

Les tests suivants ont fonctionne ponctuellement :

```bash
kubectl exec -n monitoring deploy/monitoring-grafana -c grafana -- \
wget -S -O- "http://<LOKI_IP>:3100/ready"

kubectl exec -n monitoring deploy/monitoring-grafana -c grafana -- \
wget -S -O- "http://<LOKI_IP>:3100/loki/api/v1/query?query=vector%281%29%2Bvector%281%29"
```

Ces tests ne sont toutefois pas fiables dans le temps et ne suffisent pas a considerer la datasource Loki comme validee de maniere stable dans Grafana.

Conclusion :

- Prometheus et Grafana sont exploitables
- Loki / Promtail sont deployes
- la validation complete dans Grafana Explore n'est pas finalisee

### Prochaines etapes

- verifier les pods et services de la stack de monitoring
- confirmer l'acces a l'interface associee au secret admin
- stabiliser la datasource Loki dans Grafana
- poursuivre l'integration de l'observabilite dans la phase 5

### Limites

- Grafana et Prometheus sont exploitables, mais Loki / Promtail ne sont pas encore valides completement
- la consultation des logs dans Grafana Explore reste instable et doit etre finalisee

## Deploiement WordPress (Phase 2)

### 1. Creer le namespace `wordpress`

```bash
kubectl apply -f k8s/wordpress/namespace.yaml
```

### 2. Creer le secret base de donnees

```bash
kubectl apply -f k8s/wordpress/secret-db.yaml
```

### 3. Creer le volume persistant MariaDB (PVC)

```bash
kubectl apply -f k8s/wordpress/mariadb-pvc.yaml
```

### 4. Deployer MariaDB

```bash
kubectl apply -f k8s/wordpress/mariadb-deployment.yaml
```

### 5. Creer le service MariaDB

```bash
kubectl apply -f k8s/wordpress/mariadb-service.yaml
```

### 6. Creer le volume persistant WordPress (PVC)

```bash
kubectl apply -f k8s/wordpress/wordpress-pvc.yaml
```

### 7. Deployer WordPress

```bash
kubectl apply -f k8s/wordpress/wordpress-deployment.yaml
```

### 8. Creer le service WordPress

```bash
kubectl apply -f k8s/wordpress/wordpress-services.yaml
```

### 9. Creer l'Ingress HTTP WordPress

```bash
kubectl apply -f k8s/wordpress/wordpress-ingress.yaml
```

### 10. Creer les IngressRoute Traefik HTTP et HTTPS

```bash
kubectl apply -f k8s/wordpress/wordpress-ingressroute.yaml
```

### 11. Verification

```bash
kubectl get all -n wordpress
kubectl get pvc -n wordpress
kubectl get secrets -n wordpress
kubectl get ingress -n wordpress
kubectl get ingressroute -n wordpress
```

## Avancement actuel (Phase 2 - WordPress)

- Namespace `wordpress` cree
  - manifeste: `k8s/wordpress/namespace.yaml`
- Secret Kubernetes `wp-db-secret` cree
  - manifeste: `k8s/wordpress/secret-db.yaml`
- Manifeste PVC MariaDB cree
  - manifeste: `k8s/wordpress/mariadb-pvc.yaml`
- Manifeste Deployment MariaDB cree
  - manifeste: `k8s/wordpress/mariadb-deployment.yaml`
- Manifeste Service MariaDB cree
  - manifeste: `k8s/wordpress/mariadb-service.yaml`
- Manifeste Deployment WordPress cree
  - manifeste: `k8s/wordpress/wordpress-deployment.yaml`
- Manifeste Service WordPress cree
  - manifeste: `k8s/wordpress/wordpress-services.yaml`
- Manifeste Ingress HTTP WordPress cree
  - manifeste: `k8s/wordpress/wordpress-ingress.yaml`
- Manifeste IngressRoute WordPress cree
  - manifeste: `k8s/wordpress/wordpress-ingressroute.yaml`
- Manifeste PVC WordPress cree
  - manifeste: `k8s/wordpress/wordpress-pvc.yaml`
- Host HTTP actuel: `web.etna.student`
- WordPress accessible en HTTP via `web.etna.student`
- Persistance MariaDB et WordPress testee et validee
- La stack HTTP WordPress est fonctionnelle
- Note HTTPS: l'IngressRoute HTTPS reference `letsencrypt`, mais la configuration ACME n'est pas visible ici et le domaine `web.etna.student` reste un domaine local / non public
- Prochaine etape: poursuivre la validation HTTPS

## Arborescence actuelle

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
