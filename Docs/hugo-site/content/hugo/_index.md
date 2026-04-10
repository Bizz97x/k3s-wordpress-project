---
title: "Hugo"
---

# Documentation Hugo

## Presentation

Le service Hugo sert a produire la documentation statique du projet. Il centralise la documentation technique des differentes briques du cluster dans un site simple, lisible et compatible avec l'environnement retenu.

## Architecture

Le site est organise autour des repertoires suivants :

- `content/` pour les pages Markdown
- `layouts/` pour les templates HTML custom
- `static/` pour les ressources statiques locales
- `public/` pour les fichiers HTML generes par Hugo

Les sections actuellement publiees sur le site sont :

- `cluster`
- `wordpress`
- `vault`
- `observabilite`
- `hugo`

## Choix technique

Le projet n'utilise pas de theme Hugo externe.

Ce choix est volontaire pour plusieurs raisons :

- compatibilite avec Hugo `0.111.x`
- maitrise du rendu HTML et CSS
- reduction des risques d'incompatibilite avec des themes plus recents

Le site repose donc sur des layouts custom et un CSS local, avec un rendu sobre de documentation technique.

## Generation

La generation locale du site se fait avec :

```bash
hugo
```

La sortie statique est produite dans le dossier `public/`.

## Deploiement

Le site statique est servi sur le cluster par nginx.

- Deployment `hugo-docs`
- Service `hugo-docs`
- exposition via Traefik
- host : `docs.etna.student`

Le deploiement utilise un volume `emptyDir` monte dans le conteneur nginx pour servir le contenu genere.

## Acces

- URL HTTP : `http://docs.etna.student`
- URL HTTPS ciblee : `https://docs.etna.student`
- acces protege par Basic Auth

## Securite

La protection d'acces repose sur Traefik :

- middleware `docs-basicauth`
- secret Kubernetes `docs-basic-auth`

Les identifiants ne sont pas stockes dans le depot Git.

## HTTPS

Le service de documentation dispose d'une IngressRoute HTTPS :

- entryPoint `websecure`
- `certResolver: letsencrypt`

Le HTTPS est fonctionnel dans la configuration actuelle, mais la validite d'un certificat ne peut pas etre consideree comme garantie si le domaine reste dans un contexte DNS local ou non public.

## Mise a jour

Une mise a jour typique du site suit ce flux :

```bash
hugo
kubectl cp public/. docs/<pod>:/usr/share/nginx/html
```

Ce fonctionnement correspond a l'etat actuel du projet : il n'y a pas d'image Docker custom embarquant automatiquement le site.

## Debug

Un cas de debug concret a concerne l'integration d'une image Grafana dans la section Observabilite :

- l'image etait bien presente dans `static/images`
- elle etait bien generee dans `public/images`
- elle etait correctement copiee dans le pod nginx
- le probleme venait du rendu Hugo, qui supprimait une balise HTML `<img>` dans le Markdown

La correction retenue a ete d'utiliser une image Markdown standard plutot que d'activer le rendu HTML non securise.

## Limites

- volume `emptyDir` non persistant
- copie manuelle de `public/` apres recreation du pod
- pas de CI/CD pour automatiser la regeneration et la republication
- pas d'image custom dediee au service de documentation
