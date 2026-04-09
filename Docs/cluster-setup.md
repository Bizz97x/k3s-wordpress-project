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
- la datasource Prometheus est presente dans Grafana
- les requetes Prometheus fonctionnent dans Grafana Explore, notamment `up`
- le dashboard `Node Exporter Full` a ete importe avec succes
- l'affichage CPU / memoire / disque / reseau est valide pour `vm-server`, `vm2-agent` et `vm3-agent`
- Promtail est deploye
- Loki est deploye via Helm avec des values custom
- le cluster K3s utilise reste un cluster a 3 noeuds

Les fichiers `k8s/monitoring/loki-values.yaml` et `k8s/monitoring/loki-values-clean.yaml` montrent un travail de stabilisation de Loki en mode `SingleBinary` avec stockage `filesystem`, caches desactives et ajustements `memberlist`.

La collecte et la visualisation des metriques sont fonctionnelles via Prometheus, Grafana et node-exporter. Les dashboards Grafana ont ete valides avec succes, notamment le dashboard `Node Exporter Full`, qui permet de superviser les ressources des noeuds du cluster (CPU, memoire, disque, reseau). En revanche, la partie logs avec Loki/Promtail reste partiellement deployee mais instable, et n'est pas encore validee completement dans Grafana Explore.

### Dashboard Grafana valide

- Dashboard utilise : `Node Exporter Full`
- Source : Grafana.com
- ID : `1860`
- Usage actuel : base principale de demonstration de l'observabilite
- Interet : supervision exploitable des noeuds `vm-server`, `vm2-agent` et `vm3-agent`

Le dashboard Kubernetes plus generique teste auparavant ne remontait pas correctement les donnees attendues sur ce cluster. Le choix du dashboard `1860` est donc volontaire, car il est plus coherent avec les metriques effectivement collectees par `node-exporter`.

### Identification des noeuds (node-exporter)

Les pods `node-exporter` sont deployes sous forme de `DaemonSet`. Cela signifie qu'un pod `node-exporter` est lance sur chaque noeud du cluster.

Le nom des pods `node-exporter` (par exemple `monitoring-prometheus-node-exporter-xxxxx`) est genere dynamiquement par Kubernetes et ne correspond pas directement au nom des machines.

Pour faire la correspondance entre un pod `node-exporter` et une machine (`vm-server`, `vm2-agent`, `vm3-agent`), il faut utiliser une commande Kubernetes :

```bash
kubectl get pods -n monitoring -o wide | grep node-exporter
```

Dans le resultat :

- la colonne `NODE` permet d'identifier sur quel noeud tourne chaque pod `node-exporter`
- cela permet de faire le lien entre le pod Kubernetes (`monitoring-prometheus-node-exporter-xxxxx`) et la machine reelle (`vm-server`, `vm2-agent`, `vm3-agent`)

Note importante :

- les noms des pods `node-exporter` changent automatiquement lors des recreations
- il ne faut donc pas se baser sur ces noms dans Grafana ou dans l'analyse
- il vaut mieux s'appuyer sur le `nodename` ou sur l'`instance` (IP:9100)

### Recuperation du mot de passe Grafana

La commande suivante permet de recuperer le mot de passe admin Grafana apres deploiement :

```bash
kubectl get secret --namespace monitoring -l app.kubernetes.io/component=admin-secret -o jsonpath="{.items[0].data.admin-password}" | base64 --decode ; echo
```

### Acces utiles

#### Port-forward Grafana

```bash
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
```

#### Commandes pratiques

```bash
kubectl get pods -n monitoring -o wide
kubectl get svc -n monitoring
kubectl get endpoints -n monitoring
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

### Ports utiles du projet

| Port | Composant | Role | Mode d'acces / remarques |
| --- | --- | --- | --- |
| `6443` | K3s / Kubernetes API | API server du cluster | acces reseau vers le serveur K3s |
| `80` | Traefik / WordPress | HTTP | exposition web et service WordPress cote applicatif |
| `443` | Traefik | HTTPS | exposition web HTTPS via Traefik |
| `3000` | Grafana | acces local a l'UI | via `kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80` |
| `9090` | Prometheus | UI / API Prometheus | service interne monitoring, port-forward possible si necessaire |
| `9093` | Alertmanager | alerting | composant de kube-prometheus-stack, acces interne ou port-forward |
| `9100` | Node Exporter | metriques noeuds | collecte des metriques systeme des noeuds |
| `3100` | Loki | API HTTP | acces direct pod/service pour tests et debug |
| `9095` | Loki | gRPC interne | communication interne Loki ; erreurs observees pendant le debug |
| `9080` | Promtail | HTTP / metriques | port couramment utilise par Promtail pour son endpoint HTTP ; a verifier selon la release active |
| `8080` | Traefik dashboard / admin | verification Traefik | mentionne historiquement dans la phase 1 ; exposition effective a verifier sur le cluster |
| `3306` | MariaDB | base de donnees | service interne dans le namespace `wordpress` |
| `8200` | Vault | API / UI Vault | acces principal a Vault |
| `8201` | Vault | communication interne | port interne Vault, selon le mode de deploiement |

### Loki : acces direct, service et gateway

- acces direct pod Loki : utile pour tester rapidement `3100` et verifier `/ready`
- service `loki` : point d'acces interne attendu pour les requetes HTTP vers Loki
- `loki-gateway` : couche intermediaire / proxy observee pendant le debug, avec plusieurs erreurs `502`

Cette difference est importante : une reponse ponctuelle sur le pod Loki ne garantit pas forcement un comportement stable via le service `loki` ou via `loki-gateway`, ce qui explique une partie des difficultes de validation dans Grafana.

### Reperes utiles pour ne pas se perdre

- Grafana : port-forward local en `3000`
- Prometheus : service interne en `9090`
- Loki : API HTTP en `3100`
- MariaDB : `3306`
- K3s API : `6443`
- Traefik : `80/443`

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
