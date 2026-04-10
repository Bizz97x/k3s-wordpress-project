---
title: "Observabilite"
---

# Observabilite

Cette section regroupe l'etat actuel de la supervision et de la collecte de logs du projet, en distinguant clairement ce qui est valide cote metriques et ce qui reste instable cote logs.

## Perimetre

La phase observabilite est documentee dans le namespace `monitoring` avec :

- `kube-prometheus-stack`
- Prometheus
- Grafana
- Loki
- Promtail

Le cluster observe reste le cluster K3s a 3 noeuds compose de `vm-server`, `vm2-agent` et `vm3-agent`.

## Prometheus

Prometheus est documente comme installe et operationnel.

Les points valides dans la documentation sont les suivants :

- Prometheus est installe
- la collecte des metriques est fonctionnelle
- les requetes Prometheus fonctionnent dans Grafana Explore
- la requete `up` est citee comme exemple de verification

## Grafana

Grafana est documente comme installe et accessible.

### Datasource

La datasource Prometheus est presente dans Grafana et fonctionne. C'est la partie actuellement validee de la chaine d'observabilite.

### Dashboards

Le dashboard explicitement valide est :

- `Node Exporter Full`
- source Grafana.com
- ID `1860`

Ce dashboard est retenu comme base actuelle de demonstration car il remonte correctement les indicateurs CPU, memoire, disque et reseau des noeuds.

La documentation precise aussi qu'un dashboard Kubernetes plus generique avait ete teste auparavant, mais qu'il ne remontait pas correctement les donnees attendues sur ce cluster.

## Loki et Promtail

Promtail est deployee et Loki est deployee via Helm avec des values personnalisees.

Les fichiers `k8s/monitoring/loki-values.yaml` et `k8s/monitoring/loki-values-clean.yaml` sont cites comme traces d'un travail de stabilisation en mode `SingleBinary`, avec stockage `filesystem`, caches desactives et ajustements `memberlist`.

## Etat actuel

La documentation permet de distinguer deux niveaux de validation.

### Valide

- Prometheus est operationnel
- Grafana est operationnel
- la datasource Prometheus est validee
- les requetes Explore sur Prometheus fonctionnent
- le dashboard `Node Exporter Full` est importe avec succes
- la supervision CPU / memoire / disque / reseau des noeuds est validee

## Dashboard Grafana

Voici un apercu du dashboard Grafana utilise pour le monitoring du cluster Kubernetes.

![Dashboard Grafana](/images/grafana.png)

*Figure : Dashboard Grafana affichant les metriques du cluster*

### Partiellement valide ou instable

- Loki et Promtail sont deployes
- le pod Loki a pu atteindre temporairement un etat `Running` / `Ready`
- certains endpoints Loki ont parfois repondu
- la validation complete des logs dans Grafana Explore n'est pas finalisee

## Problemes connus avec Loki

Le diagnostic documente fait apparaitre plusieurs instabilites persistantes :

- readiness Loki instable
- erreurs gRPC internes sur le port `9095`
- erreurs `scheduler`, `querier` et `ingester`
- erreurs `502` via `loki-gateway`
- refus de connexion intermittents via le service `loki`
- timeouts lors du healthcheck Grafana

Dans cet etat, Grafana et Prometheus sont exploitables pour les metriques, mais Loki ne peut pas encore etre presente comme completement stable pour l'exploration des logs.

## Limites actuelles

- la partie metriques est validee, mais la partie logs ne l'est pas completement
- la datasource Loki dans Grafana Explore reste instable
- le filtrage de logs pour les services applicatifs n'est pas encore demontre comme maitrise
- un dashboard services centre sur WordPress reste a construire d'apres la roadmap

## Suite documentee

Les prochaines etapes deja identifiees dans la documentation sont :

- stabiliser la datasource Loki dans Grafana Explore
- construire un dashboard services pour WordPress
- maitriser le filtrage des logs via labels, namespace ou pod
