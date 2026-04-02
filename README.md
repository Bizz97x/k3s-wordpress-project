# Projet d'hébergement K3s

Ce projet met en place un cluster Kubernetes avec K3s, incluant :

- Déploiement WordPress
- HTTPS avec Let's Encrypt
- Gestion des secrets avec Vault
- Stack d'observabilité
- Service de documentation Hugo

## État actuel

Phase 1 terminée :
- 3 nœuds en état `Ready`
- Traefik opérationnel

Phase 2 en cours :
- Namespace `wordpress` créé
- Secret Kubernetes MariaDB créé
- PVC MariaDB créé
- Deployment MariaDB créé
- Service MariaDB créé
- Deployment WordPress créé
- Service WordPress créé
- Ingress HTTP WordPress créé

Prochaine étape :
- Ajouter la persistance WordPress
- Tester l'accès HTTP à WordPress

## Arborescence principale

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

## Documentation

- La roadmap du projet est dans `Docs/Roadmap.md`
- La documentation d'installation et de déploiement est dans `Docs/cluster-setup.md`
