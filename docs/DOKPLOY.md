# Déployer Djinbar avec Dokploy

Configuration vérifiée pour ce dépôt le 24 septembre 2026.

## Architecture recommandée

Créer un **nouveau Project `Djinbar`** dans l'instance Dokploy existante, puis une
**Application `djinbar-site`** dans son environnement `production`.

Ce découpage est préférable à une deuxième application dans le projet Djinlist : il garde les
variables, domaines, logs et déploiements clairement séparés. Il ne faut pas installer une seconde
instance Dokploy et il ne faut pas cloner le dépôt du logiciel Dokploy.

## 1. Créer le dépôt GitHub

Le dépôt Git local existe déjà sur la branche `main`, sans remote. GitHub CLI n'étant pas installé
sur la machine, créer d'abord dans l'interface GitHub un dépôt **vide** nommé `djinbar-site` sous le
compte `Bueno92` : ne pas ajouter de README, de `.gitignore` ou de licence.

Avant de le créer, vérifier dans GitHub que `Bueno92/djinbar-site` n'existe pas déjà, y compris parmi
les dépôts privés. Puis exécuter :

```sh
cd "/Users/bueno/Documents/Codex/2026-09-24/contexte-produit-djinbar-djinbar-est-une/outputs/djinbar-site"
git remote add origin https://github.com/Bueno92/djinbar-site.git
git remote -v
git push -u origin main
```

Le remote doit afficher uniquement `Bueno92/djinbar-site`. Il ne doit jamais contenir
`Bueno92/captur-website`.

## 2. Donner accès au dépôt à Dokploy

Si l'intégration GitHub de Dokploy est déjà installée pour tous les dépôts, aucune action n'est
nécessaire. Si elle n'a accès qu'à une sélection :

1. Dans GitHub, ouvrir les paramètres de l'installation de la GitHub App Dokploy.
2. Ajouter uniquement `djinbar-site` à la sélection autorisée.
3. Revenir dans Dokploy et rafraîchir la liste des dépôts.

Ne pas modifier l'application Dokploy Djinlist existante.

## 3. Créer le projet et l'application

1. Dans Dokploy, créer un **Project** nommé `Djinbar`.
2. Garder ou créer l'environnement **production**.
3. Dans cet environnement, créer une **Application** nommée `djinbar-site`.
4. Dans **General**, choisir la source **GitHub**.
5. Sélectionner les valeurs suivantes :

| Champ Dokploy | Valeur |
| --- | --- |
| Compte / Owner | `Bueno92` |
| Repository | `djinbar-site` |
| Branch | `main` |
| Build Path / Root Directory | `/` |
| Trigger Type | `push` |
| Auto Deploy | activé |

6. Enregistrer la source.

## 4. Configurer le build

Dans **General → Build Type**, choisir **Dockerfile** et saisir exactement :

| Champ Dokploy | Valeur |
| --- | --- |
| Build Type | `Dockerfile` |
| Dockerfile Path | `Dockerfile` |
| Docker Context Path | `.` |
| Docker Build Stage | laisser vide |
| Install Command | laisser vide / non applicable |
| Build Command | laisser vide / non applicable |
| Start Command | laisser vide / non applicable |
| Publish Directory | laisser vide / non applicable |

Le `Dockerfile` exécute déjà `npm ci`, puis `npm run build`. Il copie ensuite `dist/` dans Nginx,
qui démarre automatiquement et écoute sur le port `80`.

Dans **Environment**, ne rien ajouter :

- variables obligatoires : aucune ;
- variables optionnelles : aucune actuellement ;
- build arguments : aucun ;
- build-time secrets : aucun ;
- variables Djinlist : inutiles et à ne surtout pas recopier.

Dans **Advanced → Ports**, ne publier aucun port hôte. Le domaine utilisera le port interne du
conteneur via Traefik.

## 5. Faire le premier déploiement

1. Cliquer sur **Deploy**.
2. Ouvrir **Deployments** pour suivre la construction.
3. Vérifier dans les logs que les étapes suivantes réussissent :
   - image `node:22-alpine` récupérée ;
   - `npm ci` terminé ;
   - `npm run build` terminé ;
   - Astro indique `1 page(s) built` et crée `dist/` ;
   - image finale `nginx:1.27-alpine` créée ;
   - déploiement au statut `Done` / application `Running`.
4. Dans **Logs**, Nginx ne doit pas redémarrer en boucle et ne doit afficher aucune erreur de
   configuration.

Avant le DNS final, un domaine temporaire généré par Dokploy peut servir au test. Son port de
conteneur doit être `80`.

## 6. Configurer le DNS chez OVH

Trouver d'abord l'IPv4 publique de l'unique serveur qui héberge l'instance Dokploy. Ne pas inventer
la valeur. Dans la zone DNS de `djinbar.com`, créer :

```text
A       @       IP_DU_SERVEUR_DOKPLOY
CNAME   www     djinbar.com.
```

Remplacer `IP_DU_SERVEUR_DOKPLOY` par la vraie IPv4. Le point final du CNAME est accepté par OVH ;
si l'interface le retire, ce n'est pas un problème. Supprimer ou corriger uniquement les anciens
enregistrements `@` ou `www` qui entreraient en conflit après avoir vérifié leur usage.

Si le serveur possède une IPv6 publique correctement routée, un enregistrement `AAAA @` peut être
ajouté avec cette vraie IPv6 ; sinon, ne rien créer.

## 7. Ajouter les domaines et HTTPS

Dans l'onglet **Domains** de l'application Djinbar :

1. Ajouter `djinbar.com`.
2. Path : `/`.
3. Container Port : `80`.
4. Activer HTTPS / Let's Encrypt.
5. Enregistrer.
6. Ajouter `www.djinbar.com` avec Path `/`, Container Port `80` et HTTPS / Let's Encrypt.
7. Dans **Advanced → Redirects**, choisir le preset **www to non-www** ; à défaut, créer une
   redirection permanente vers `https://djinbar.com`.

L'URL canonique construite dans le HTML est déjà `https://djinbar.com/`. Attendre la propagation
DNS avant de conclure à un échec Let's Encrypt, puis tester :

```sh
curl -I https://djinbar.com
curl -I https://www.djinbar.com
```

La première URL doit répondre en HTTPS. La seconde doit répondre par une redirection permanente
vers `https://djinbar.com`.

## 8. Déploiements automatiques et isolation

Avec l'intégration GitHub et **Auto Deploy** activé, un push sur la branche `main` de
`Bueno92/djinbar-site` redéploie uniquement l'application Djinbar. Le dépôt, l'application Dokploy,
le projet Dokploy et le domaine sont distincts de Djinlist ; un push Djinbar ne doit donc jamais
déclencher Djinlist.

Après le premier push, vérifier une fois dans **Deployments** que l'événement affiche bien :

- repository : `djinbar-site` ;
- branch : `main` ;
- application : `djinbar-site`.

## Erreurs courantes pour ce projet

- **Le dépôt n'apparaît pas** : la GitHub App Dokploy n'a pas accès au nouveau dépôt.
- **`package-lock.json` mismatch** : ne pas remplacer le lockfile ; le build utilise `npm ci`.
- **Mauvais port** : utiliser `80`, jamais `4321` (dev Astro) ni `3000`.
- **Dockerfile introuvable** : Build Path `/`, Dockerfile Path `Dockerfile`, Context `.`.
- **502 Bad Gateway** : le domaine cible probablement le mauvais container port, ou le conteneur
  Nginx ne tourne pas ; vérifier **Logs** puis le port `80`.
- **Certificat en attente** : confirmer que `A @` et `CNAME www` pointent vers ce serveur et que les
  ports publics 80/443 atteignent Traefik.
- **404 sur un asset** : vérifier que le build Astro a bien généré `dist/` avant la copie Nginx.
- **Aucun auto-deploy** : vérifier Auto Deploy, la branche `main` et l'accès de la GitHub App.

## Références officielles

- [Dokploy — Build Type](https://docs.dokploy.com/docs/core/applications/build-type)
- [Dokploy — GitHub](https://docs.dokploy.com/docs/core/github)
- [Dokploy — Auto Deploy](https://docs.dokploy.com/docs/core/auto-deploy)
- [Dokploy — Domains](https://docs.dokploy.com/docs/core/domains)
