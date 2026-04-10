---
title: "Cluster"
---

# Cluster

Cette section presente le socle d'infrastructure du projet : le cluster K3s, son organisation sur 3 VM, le role de Traefik et l'architecture generale qui porte les services documentes dans le depot.

## Vue d'ensemble

Le projet repose sur un cluster K3s a 3 noeuds avec Traefik en frontal. La documentation existante decrit une architecture dans laquelle Traefik expose la stack WordPress, tandis que MariaDB reste joignable uniquement en interne.

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

## Infrastructure

L'infrastructure documentee s'appuie sur 3 machines virtuelles :

| VM | Role | Ressources |
| --- | --- | --- |
| VM-SERVER | `k3s-server` | 2 vCPU / 4 GB RAM |
| VM2-Agent | `k3s-agent-1` | 1 vCPU / 1 GB RAM |
| VM3-Agent | `k3s-agent-2` | 1 vCPU / 1 GB RAM |

La feuille de route indique que la preparation des 3 VM et l'installation du cluster sont terminees. Le critere de validation retenu est un `kubectl get nodes` montrant 3 noeuds en statut `Ready`.

## Installation de K3s

L'installation documentee est la suivante.

### Serveur

```bash
curl -sfL https://get.k3s.io | sh -
```

### Recuperation du token

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

### Agents

```bash
curl -sfL https://get.k3s.io | \
K3S_URL=https://IP_VM1:6443 \
K3S_TOKEN=<TOKEN> \
sh -
```

### Verification

```bash
kubectl get nodes
```

Le resultat attendu dans la documentation est un cluster compose de `vm-server`, `vm2-agent` et `vm3-agent`, tous en `Ready`.

## Traefik

Traefik est installe automatiquement avec K3s par defaut. La roadmap le presente comme verifie et operationnel dans la phase de socle du projet.

### Verifications documentees

```bash
kubectl get pods -A
kubectl get svc traefik -n kube-system
```

La documentation associe Traefik a l'ecoute sur `80/443`, avec un dashboard accessible sur `8080`.

### Configuration HTTPS

Le projet s'appuie aussi sur une configuration complementaire appliquee directement sur le cluster via un `HelmChartConfig` place dans :

```text
/var/lib/rancher/k3s/server/manifests
```

Cette configuration n'est pas versionnee dans le depot, mais elle est decrite comme necessaire pour :

- activer ACME
- declarer le resolver `letsencrypt`
- persister les donnees Traefik dans `/data`

Ce point est important car l'IngressRoute HTTPS de WordPress reference `certResolver: letsencrypt`.

## Architecture du projet

La documentation actuelle fait apparaitre plusieurs espaces de travail logiques sur le cluster :

- `wordpress` pour la stack applicative WordPress / MariaDB
- `vault` pour la gestion des secrets
- `monitoring` pour l'observabilite

Le namespace `wordpress` concentre aujourd'hui les ressources applicatives principales :

- Secret `wp-db-secret`
- PVC `mariadb-pvc`
- Deployment `mariadb`
- Service `mariadb`
- PVC `wordpress-pvc`
- Deployment `wordpress`
- Service `wordpress`
- Ingress `wordpress-ingress`
- IngressRoute `wordpress-http`
- IngressRoute `wordpress-https`

## Reseau et exposition

Les elements reseau utiles a la comprehension de l'architecture sont les suivants :

- le service `mariadb` est expose en `ClusterIP` sur le port `3306`
- le service `wordpress` est expose en `ClusterIP` sur le port `80`
- l'host documente pour WordPress est `web.etna.student`
- l'Ingress Kubernetes `wordpress-ingress` route vers `wordpress:80`
- l'IngressRoute `wordpress-http` utilise l'entryPoint `web`
- l'IngressRoute `wordpress-https` utilise l'entryPoint `websecure`

La documentation souligne que `Ingress` et `IngressRoute` coexistent actuellement dans le dossier `k8s/wordpress`. Cela montre une transition vers les CRD Traefik, sans permettre d'affirmer a partir des seuls fichiers quelle ressource est prioritaire sur le cluster.

## Etat actuel

Au niveau infrastructure, les points suivants sont documentes comme acquis :

- le cluster K3s fonctionne sur 3 noeuds
- Traefik est documente comme operationnel
- le socle K3s / Traefik correspond a une phase terminee dans la roadmap

## Limites connues

- la configuration Traefik / ACME via `HelmChartConfig` fait partie de l'infrastructure reelle, mais n'est pas versionnee dans ce depot
- le domaine `web.etna.student` est traite comme un domaine local ou non public dans le contexte du projet
- par consequent, la presence des manifests HTTPS ne suffit pas a considerer le HTTPS comme valide en production
