# Comment fonctionne (vraiment) le lien registre → yml → site public

> **Note de mise à jour** : ce document a été révisé après que `packages/ofceweb` a été rebasculé sur
> sa branche `main` (v1.0.4.9000, commit `dd6d77c`). La première version de cette analyse avait été
> écrite en lisant une autre branche (`dev_pb`) du package, dont une partie du design a depuis été
> changée sur `main`. Les sections ci-dessous reflètent l'état sur `main` ; les endroits où mon
> analyse précédente était fausse sont signalés explicitement.

Résumé d'abord, détails ensuite : **il n'existe, dans aucun code que j'ai pu trouver localement,
d'automatisation qui mette à jour `working_papers.yml` ou `policy.yml` quand une PR est fusionnée
dans `wp-registry`.** La documentation officielle du package (`ofceweb`, vignettes sur `main`)
affirme pourtant, au présent, qu'un tel mécanisme existe côté `webhome` — c'est le point le plus
important de cette mise à jour, détaillé en §3.

---

## 1. Les deux circuits sont conceptuellement séparés

Il y a **deux systèmes distincts**, et le lien entre eux (normalement le second lirait le premier)
est la partie dont le statut reste incertain :

### Circuit A — le registre (`wp-registry`), consulté par chaque dépôt WP/PB individuellement

- Un·e auteur·e lance `ofceweb::registry_request()` (fonction générique sur `main`, qui détecte le
  type de dépôt — `wp` ou `pb` — et appelle en interne `wp_registry_request()`/`pb_registry_request()`)
  depuis son propre dépôt. Elle ouvre une PR sur **`OFCE/wp-registry`** (ce dépôt) qui ajoute une
  entrée à `wp/{année}.json` (WP, sharded par année) ou à `pb/pb.json` (PB, fichier plat, numérotation
  séquentielle **indépendante de l'année** — voir §2).
- Un·e admin (`xtimbeau`, seul admin actuel pour `/wp/` — voir `.github/CODEOWNERS`) relit et fusionne.
- **Ce que la fusion déclenche dans ce dépôt** : rien, directement. Le registre est en lecture publique
  (`raw.githubusercontent.com`), sans authentification. C'est le dépôt WP/PB **lui-même** qui, à son
  prochain `setup_wp()`/`setup_pb()` ou `publish()` (lancé manuellement ou en CI, **dans son propre
  dépôt**, pas dans `wp-registry`), interroge le registre via `sync_wp_registry_state()`/
  `sync_pb_registry_state()` pour savoir s'il a une entrée confirmée, puis se déploie en conséquence
  (`www.ofce.fr/wp/{année}/{N}/` ou `www.ofce.fr/pb/{N}/`, via FTP, dans le workflow du dépôt WP/PB —
  pas dans `wp-registry`).
- Le registre est donc une **autorisation** (« tel dépôt a le droit de publier tel numéro »), pas un
  système de notification. Rien dans le code de `wp-registry` ni dans `ofceweb` ne pousse quoi que ce
  soit vers `webhome` à ce stade.

### Circuit B — les pages publiques (`webhome/publications/*.qmd` + `*.yml`)

- `working_papers.qmd` et `policy.qmd` (dans `web/webhome/publications/`) sont des pages Quarto
  statiques qui lisent respectivement `working_papers.yml` et `policy.yml`.
- D'après `webhome/README.md` (§ « Modifier ou ajouter du contenu ») : la procédure **documentée et
  actuelle** pour ajouter une entrée à ces listes est **manuelle** (ajouter en haut du YAML). Le seul
  fichier de `publications/` qui soit *généré* au rendu Quarto est `revue.yml` (via
  `scripts/generate_revue_pages.R`) — rien d'équivalent n'existe pour `working_papers.yml`/`policy.yml`.
- `scripts/generate_working_papers_yaml.R` et `generate_pbrief_yaml.R` sont des **scripts de
  migration one-shot** (commentaire explicite en tête de fichier) : ils ont exporté l'ancienne base
  MySQL vers ces YAML une fois, au moment de la migration, et ne tournent ni en CI ni ailleurs.
- L'historique git de `working_papers.yml`/`policy.yml` dans `webhome` (commits : « update », « zou
  194 », « 2026-9 », « corrections typos », « pb154 »…) est cohérent avec des **modifications
  manuelles**, pas avec des commits automatiques d'un bot.
- Le seul workflow GitHub Actions de `webhome` (`.github/workflows/ftp_deploy.yml`) déploie par FTP
  le contenu déjà rendu de la branche `site-deploy`, déclenché manuellement (`workflow_dispatch`) —
  aucun rapport avec une collecte depuis `wp-registry` ou les dépôts WP/PB.

---

## 2. Le registre PB : design désormais tranché (correction par rapport à ma première analyse)

Sur la branche `dev_pb` que j'avais lue initialement, le code attendait une structure `pb/{année}.json`
+ `pb/index.json` identique à celle des WP, avec un champ `annee` — ce qui ne correspondait pas au
`pb/pb.json` plat réellement présent dans ce dépôt, et j'avais décrit ça comme une migration de
données non terminée.

**Sur `main`, ce n'est plus le cas : le code a changé pour correspondre exactement aux données
réelles.** `R/pb_registry_sync.R` (fonction interne `fetch_pb_registry()`) lit directement
`pb/pb.json` — pas `pb/index.json`/`pb/{année}.json` — et le commentaire en tête de fichier le pose
comme un choix de design définitif, pas provisoire :

> Contrairement au registre WP (sharded par année, `wp/{annee}.json`), le registre PB est un
> **fichier plat unique** : les numéros PB sont attribués de manière strictement séquentielle depuis
> l'origine, indépendamment de l'année de publication — l'année n'a donc aucune valeur d'indexation
> ici.

Confirmé aussi par la vignette `policy-briefs.Rmd` (tableau de comparaison WP/PB) : `annee` n'est
« pas utilisé » pour les PB, et le registre PB est bien « partagé avec les WP » dans
`ofceweb/wp-registry` sous `pb/`. Mon analyse précédente était donc fausse sur ce point précis — ce
n'est pas un gap à corriger, c'est le design voulu.

**Ce qui reste un vrai gap, en revanche** (inchangé par le changement de branche, car ce sont des
fichiers de *ce* dépôt, indépendants de la version d'`ofceweb`) :
- `.github/CODEOWNERS` ne liste que `/wp/` — pas de protection `/pb/` dédiée ;
- `validate-registry.yml` (`on.pull_request.paths`) ne couvre que `wp/**` — une PR touchant
  `pb/pb.json` n'est pas un *required status check* et peut être fusionnée sans aucune validation
  JSON/schéma automatique.

---

## 3. Le vrai sujet : webhome collecte-t-il déjà le registre/les manifestes ? (point nouveau)

C'est ici que le changement de branche change le plus de choses. Les deux vignettes de `main`
(`vignettes/working-papers.Rmd` et `vignettes/policy-briefs.Rmd`), au présent de l'indicatif,
affirment :

> Ce manifeste est collecté par `webhome` via la GitHub API pour construire l'index des WPs de
> l'OFCE. *(working-papers.Rmd, section « Manifeste JSON »)*

> `pb_manifest()` ... génère `manifest.json`, **collecté par `webhome`** pour construire l'index des
> PB de l'OFCE. *(policy-briefs.Rmd, section « Manifeste JSON »)*

C'est exactement le mécanisme que tu cherchais dans ta première question — sauf que **je ne trouve
aucun code qui l'implémente** dans ma copie locale de `webhome` :
- `scripts/` ne contient que les scripts one-shot MySQL (§1) et `generate_revue_pages.R`/
  `generate_blog_latest.R` (pour d'autres sections du site, sans rapport avec WP/PB) ;
- `.github/workflows/` ne contient que `ftp_deploy.yml`, un déploiement FTP manuel du contenu déjà
  rendu — aucune requête GitHub API, aucun cron, aucune génération de YAML ;
- aucune fonction de `ofceweb` (vérifié sur `main`) n'appelle l'API GitHub pour *lister* les dépôts
  WP/PB de l'organisation ou leurs `manifest.json` — `ofceweb` sait écrire `manifest.json` dans le
  dépôt WP/PB lui-même, jamais le lire depuis un autre dépôt ;
- `packages/ofceweb/dev_pb.md` (notes de conception, non modifiées depuis) dit explicitement que
  cette collecte automatique est un **chantier séparé, non commencé**, hors périmètre du travail PB :
  *« L'apparition dans `publications/policy.qmd` reste, pour l'instant, alimentée par le `policy.yml`
  statique existant (comme les WP via `working_papers.yml`). »*

**Donc deux lectures possibles, que je ne peux pas départager avec ce qui est visible localement :**
1. La vignette décrit une **intention/un objectif de design**, au présent par anticipation, pour un
   mécanisme qui n'a en réalité pas encore été construit (cohérent avec `dev_pb.md`, qui lui est
   explicite sur le "pas encore").
2. Le mécanisme existe réellement mais **vit ailleurs** que dans les dépôts que j'ai en local — par
   exemple un script ou une tâche planifiée côté serveur OFCE, un outil interne non versionné dans
   `webhome`, ou une étape manuelle semi-régulière (quelqu'un relit les `manifest.json` publiés et met
   à jour le YAML à la main, aidé par un script qui ne vit pas dans ces dépôts).

**Pour trancher, il faudrait vérifier** (je ne peux pas le faire depuis l'environnement local) :
- si un `OFCE/webhome` (ou `ofceweb/webhome`) distant a des commits automatiques sur
  `working_papers.yml`/`policy.yml` signés par un compte bot/Action plutôt que par une personne ;
- s'il existe un outil/service hors des dépôts Git (serveur OFCE, Google Apps Script, Zapier, etc.)
  qui fait cette collecte ;
- auprès de qui maintient `ofceweb`/`webhome` (les vignettes citées plus haut), si la phrase décrit
  un état réel ou une cible.

---

## 4. Comment construire ce workflow s'il n'existe pas (pistes, rien d'implémenté)

Si la lecture (1) ci-dessus est la bonne — rien n'existe encore — voici ce qu'il faudrait :

### 4.1 Gouvernance PB (préalable indépendant, voir §2)
- ajouter `/pb/` à `.github/CODEOWNERS` ;
- étendre `validate-registry.yml` à `pb/**` (contrôles JSON/schéma adaptés au fichier plat, pas de
  contrôle année↔fichier puisque `pb/pb.json` n'a pas d'`annee`), et le déclarer *required status
  check*.

### 4.2 Le collecteur webhome lui-même
Deux approches, pas exclusives :
- **Déclenchement sur merge** : un workflow dans `wp-registry`, sur `push` vers `main` touchant
  `wp/**`/`pb/**`, qui déclenche (`repository_dispatch` ou `workflow_dispatch` via PAT) un workflow
  côté `OFCE/webhome` qui régénère les YAML.
- **Reconstruction périodique (cron)** côté `webhome` : plus simple côté permissions inter-dépôts,
  moins réactif.

Dans les deux cas :
- **Entrées `pdf-only`** (pas de dépôt Quarto) : conversion directe depuis `pdf-path`/`type`, sans
  `source-repo`.
- **Entrées `repo`** : le registre ne stocke que `{numéro, source-repo, contact}` — titre, auteurs,
  résumé, date doivent être récupérés depuis le `manifest.json` **déployé** du dépôt source (ce que
  les vignettes citées en §3 décrivent), via `https://raw.githubusercontent.com/{source-repo}/.../manifest.json`
  ou l'API GitHub — pas une simple conversion JSON→YAML.
- Garder un passage humain (PR plutôt que commit direct sur `main` de `webhome`) pour éviter qu'une
  entrée mal formée ne casse silencieusement la page publique.

---

## 5. Fichiers consultés pour cette analyse

- `wp-registry/.github/workflows/validate-registry.yml`, `wp-registry/.github/CODEOWNERS`
- `wp-registry/wp/*.json`, `wp-registry/pb/*.json` (structure réelle actuelle)
- `web/webhome/README.md`, `web/webhome/publications/{working_papers,policy}.qmd`,
  `web/webhome/.github/workflows/ftp_deploy.yml`, `web/webhome/scripts/*.R`
- `packages/ofceweb` sur `main` (v1.0.4.9000, commit `dd6d77c`) : `R/registry_request.R`,
  `R/pb_registry_sync.R`, `R/wp_registry_sync.R`, `NAMESPACE`, `NEWS.md`
- `packages/ofceweb/vignettes/working-papers.Rmd`, `vignettes/policy-briefs.Rmd` (section « Manifeste
  JSON » — source de la nouvelle question ouverte en §3)
- `packages/ofceweb/dev_pb.md` (notes de conception du pipeline PB, non retouchées depuis la lecture
  précédente — confirme toujours le périmètre hors-sujet pour l'intégration webhome)
