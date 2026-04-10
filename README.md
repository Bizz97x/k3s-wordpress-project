# Projet DevOps K3s WordPress

[![Kubernetes](https://img.shields.io/badge/Kubernetes-K3s-blue)]()
[![Traefik](https://img.shields.io/badge/Ingress-Traefik-blue)]()
[![Monitoring](https://img.shields.io/badge/Monitoring-Prometheus%20%7C%20Grafana-orange)]()
[![Logging](https://img.shields.io/badge/Logging-Loki-lightgrey)]()
[![Secrets](https://img.shields.io/badge/Secrets-Vault-green)]()
[![Statut](https://img.shields.io/badge/Statut-En%20cours-yellow)]()

---

## Présentation

Ce projet illustre le déploiement d'une stack applicative complète sur un cluster Kubernetes K3s. Il couvre la mise en place de l'infrastructure, le déploiement de l'application, l'exposition sécurisée, la gestion des secrets et l'observabilité.

L'objectif est de construire et d'opérer un environnement proche de la production en appliquant des pratiques DevOps modernes.

---

## Architecture

L'application est déployée sur un cluster K3s à 3 nœuds avec Traefik comme contrôleur d'ingress.

```
Client
  │
  ▼
Traefik (Contrôleur d'Ingress)
  │
  ├── Ingress / IngressRoute
  ▼
Service WordPress
  │
  ▼
Déploiement WordPress
  │
  ▼
Volume Persistant (PVC)
  │
  ▼
Service MariaDB
  │
  ▼
Déploiement MariaDB
  │
  ▼
Volume Persistant (PVC)
```

---

## Stack Technique

| Composant | Rôle |
|-----------|------|
| Kubernetes (K3s) | Orchestration de conteneurs |
| Traefik | Contrôleur d'ingress |
| WordPress | Application frontend |
| MariaDB | Base de données |
| Vault | Gestion des secrets (partiel) |
| Prometheus | Collecte de métriques |
| Grafana | Visualisation |
| Loki & Promtail | Agrégation de logs (partiel) |

---

## Fonctionnalités

- Déploiement d'un cluster Kubernetes avec K3s
- Déploiement WordPress + MariaDB avec stockage persistant (PVC)
- Exposition interne des services via ClusterIP
- Accès externe via Traefik (Ingress / IngressRoute)
- Configuration HTTPS avec Let's Encrypt *(limitation d'environnement)*
- Déploiement de Vault via Helm (mode dev)
- Monitoring avec Prometheus et Grafana
- Stack de logs avec Loki et Promtail *(partiellement opérationnel)*

---

## Structure du Projet

```
group-1070802/
├── README.md
├── Docs/
│   ├── Roadmap.md
│   ├── Roadmap.html
│   └── cluster-setup.md
└── k8s/
    ├── monitoring/
    ├── vault/
    └── wordpress/
```

---

## Déploiement

### Prérequis

- Cluster Kubernetes (K3s)
- `kubectl` configuré
- Helm installé

### Déployer la stack WordPress

```bash
kubectl apply -f k8s/wordpress/
```

### Vérifier les ressources

```bash
kubectl get all -n wordpress
kubectl get pvc -n wordpress
kubectl get ingress -n wordpress
```

---

## Observabilité

- **Prometheus** collecte les métriques du cluster et des nœuds
- **Grafana** fournit les dashboards et la visualisation
  - Dashboard Node Exporter (ID `1860`) utilisé pour le monitoring infra
- **Loki + Promtail** déployés pour l'agrégation de logs *(actuellement instable)*

---

## Sécurité

- Identifiants de la base de données stockés dans des Kubernetes Secrets
- HTTPS configuré via Traefik IngressRoute avec Let's Encrypt
- Vault déployé pour la gestion avancée des secrets *(mode dev)*

---

## Limitations

- Le HTTPS ne peut pas être entièrement validé en raison de l'utilisation d'un domaine local (`web.etna.student`)
- Vault est déployé en mode développement et n'est pas encore intégré aux applications
- Loki est déployé mais pas encore stable pour un usage en production
- Certaines configurations d'infrastructure (Traefik ACME) ne sont pas versionnées dans le dépôt

---

## Compétences Démontrées

- Déploiement et gestion d'un cluster Kubernetes (K3s)
- Déploiement d'applications via des manifests Kubernetes
- Gestion de l'ingress avec Traefik
- Configuration du stockage persistant (PVC)
- Gestion des secrets (Kubernetes Secrets + Vault)
- Mise en place du monitoring (Prometheus, Grafana)
- Concepts d'agrégation de logs (Loki, Promtail)
- Débogage et résolution de problèmes d'infrastructure

---

## Améliorations Futures

- Intégrer Vault aux workloads Kubernetes (injection de secrets)
- Mettre en place des secrets dynamiques et des politiques de rotation
- Stabiliser Loki et valider les logs dans Grafana Explore
- Utiliser un domaine public pour valider les certificats HTTPS
- Ajouter un pipeline CI/CD pour le déploiement automatisé
- Améliorer la documentation avec un service Hugo dédié

---

## Auteur

**Randy Bizet**  
Étudiant en Master Architecture des Systèmes d'Informations
