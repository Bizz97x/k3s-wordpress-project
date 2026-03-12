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
kubectl apply -f k8s/wordpress/mariadb-pcv.yaml
```

### 4. Deployer MariaDB

```bash
kubectl apply -f k8s/wordpress/mariadb-deployment.yaml
```

### 5. Verification

```bash
kubectl get all -n wordpress
kubectl get pvc -n wordpress
kubectl get secrets -n wordpress
```

## Avancement actuel (Phase 2 - WordPress)

- Namespace `wordpress` cree
  - manifeste: `k8s/wordpress/namespace.yaml`
- Secret Kubernetes `wp-db-secret` cree
  - manifeste: `k8s/wordpress/secret-db.yaml`
- Manifeste PVC MariaDB cree
  - manifeste: `k8s/wordpress/mariadb-pcv.yaml`
- Manifeste Deployment MariaDB cree
  - manifeste: `k8s/wordpress/mariadb-deployment.yaml`
- Prochaine etape: creer le Service MariaDB, puis deployer WordPress + Service

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
        ├── mariadb-pcv.yaml
        ├── namespace.yaml
        ├── secret-db.exemple.yaml
        └── secret-db.yaml
```
