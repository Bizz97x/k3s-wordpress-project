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

### 6. Deployer WordPress

```bash
kubectl apply -f k8s/wordpress/wordpress-deployment.yaml
```

### 7. Creer le service WordPress

```bash
kubectl apply -f k8s/wordpress/wordpress-services.yaml
```

### 8. Creer l'Ingress HTTP WordPress

```bash
kubectl apply -f k8s/wordpress/wordpress-ingest.yaml
```

### 9. Verification

```bash
kubectl get all -n wordpress
kubectl get pvc -n wordpress
kubectl get secrets -n wordpress
kubectl get ingress -n wordpress
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
  - manifeste: `k8s/wordpress/wordpress-ingest.yaml`
- Host HTTP actuel: `web.etna.student`
- Prochaine etape: ajouter la persistance WordPress puis tester l'acces HTTP

## Arborescence actuelle

```text
group-1070802/
├── README.md
├── Docs/
│   ├── Roadmap.html
│   ├── Roadmap.md
│   └── cluster-setup.md
└── k8s/
    └── wordpress/
        ├── mariadb-deployment.yaml
        ├── mariadb-pvc.yaml
        ├── mariadb-service.yaml
        ├── namespace.yaml
        ├── secret-db.exemple.yaml
        ├── secret-db.yaml
        ├── wordpress-deployment.yaml
        ├── wordpress-ingest.yaml
        └── wordpress-services.yaml
```
