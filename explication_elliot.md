# Comment fonctionne (vraiment) le lien registre → yml → site public

Résumé d'abord, détails ensuite : **il n'existe aujourd'hui aucune automatisation qui mette à jour
`working_papers.yml` ou `policy.yml` quand une PR est fusionnée dans `wp-registry`.** Ce n'est pas
une particularité de `policy.yml` qui serait « en retard » sur `working_papers.yml` — les deux
fonctionnent de la même façon (mal) : ils sont alimentés à la main, ou par un script de migration
one-shot qui a déjà tourné une fois et qu'il ne faut plus relancer.

Ce document explique ce qui existe réellement, pourquoi ta lecture (« la fusion d'une PR devrait
déclencher une action ») est une hypothèse raisonnable mais fausse à ce jour, et ce qu'il faudrait
construire pour qu'elle devienne vraie.

---

## 1. Les deux circuits sont complètement séparés

Il y a en réalité **deux systèmes indépendants** qui n'ont aucun lien de code entre eux :

### Circuit A — le registre (`wp-registry`), consulté par chaque dépôt WP individuellement

- Un·e auteur·e lance `ofceweb::wp_registry_request()` depuis son dépôt WP (ex. `OFCE/wp-pam-pmq`).
  Cette fonction ouvre une PR sur **`OFCE/wp-registry`** (ce dépôt) qui ajoute une entrée à
  `wp/{année}.json` (et met à jour `wp/index.json` si c'est la première entrée de l'année).
- Un·e admin (`xtimbeau`, seul admin actuel — voir `.github/CODEOWNERS`) relit et fusionne.
- **Ce que la fusion déclenche réellement** : rien, directement. Le registre est en lecture publique
  (`raw.githubusercontent.com`), sans authentification. C'est le dépôt WP **lui-même** qui, à son
  *prochain* `render_wp()` (lancé manuellement ou en CI dans **son propre dépôt**, pas dans
  `wp-registry`), va interroger le registre via `sync_wp_registry_state()`
  (`ofceweb/R/wp_registry_sync.R`) pour savoir s'il a une entrée confirmée. Si oui, il bascule
  `draft: false`, récupère son `wp`/`annee` définitifs, et son prochain `deploy_wp()` publie sur
  `www.ofce.fr/wp/{année}/{numéro}/` via FTP (`ftp_deploy.yml`, dans le dépôt WP, pas ici).
- Le registre est donc une **autorisation** (« tel dépôt a le droit de publier tel numéro »), pas un
  système de notification. Rien dans `wp-registry` ne sait que `webhome` existe.

### Circuit B — les pages publiques (`webhome/publications/*.qmd` + `*.yml`)

- `working_papers.qmd` et `policy.qmd` (dans `web/webhome/publications/`) sont des pages Quarto
  statiques qui lisent respectivement `working_papers.yml` et `policy.yml` — deux fichiers YAML
  **indépendants du registre**, au même format que `rapports.yml`.
- D'après `webhome/README.md` (§ « Modifier ou ajouter du contenu ») : la procédure documentée et
  actuelle pour ajouter une publication à ces listes est **manuelle** — ajouter une entrée en haut du
  YAML. Le seul des fichiers de `publications/` qui est *généré* automatiquement au rendu est
  `revue.yml` (via `scripts/generate_revue_pages.R`, exécuté en pré-rendu Quarto). Rien d'équivalent
  n'existe pour `working_papers.yml` ou `policy.yml`.
- Les deux scripts qui portent un nom qui laisse penser à une génération —
  `scripts/generate_working_papers_yaml.R` et `scripts/generate_pbrief_yaml.R` — sont en fait des
  **scripts de migration one-shot** (le commentaire en tête de fichier le dit explicitement) : ils
  lisaient l'ancienne base MySQL (`ndtravail2`, `npbrief`) pour produire les YAML une fois pour
  toutes, au moment de la migration hors de la DB. Ils ne tournent pas en CI, ne sont déclenchés par
  rien, et le commentaire précise qu'ils seront obsolètes une fois la DB supprimée.

**Conclusion du point 1** : la fusion d'une PR dans `wp-registry` ne touche jamais `webhome`. Le
`yml` qui nourrit la page publique est un fichier à jour manuellement (ou figé depuis la migration
MySQL), pas un reflet du registre.

---

## 2. Un équivalent pour les PB existe-t-il ?

Côté **registre**, oui, partiellement — et c'est là que ça devient intéressant pour comprendre ton
observation sur `policy.yml` :

- Le dossier `pb/` de ce dépôt (`pb/pb.json`, `pb/registry.schema.json`) a été ajouté a posteriori
  (entrées `"registered-by": "ofceweb-assistant"`, 2026-09-05), apparemment pour « rattraper »
  l'historique des PB déjà publiés en PDF (`type: "pdf-only"`, `pdf-path: pdf/pbrief/...`).
- Mais ce `pb/` est **incomplet par rapport à `wp/`**, et surtout **déconnecté du code R prévu pour
  le consommer** :
  - Pas de split par année (`pb/{année}.json` + `pb/index.json`) comme pour `wp/` — tout est dans un
    seul `pb.json`, et le schéma (`pb/registry.schema.json`) ne contient même pas de champ `annee`.
  - Pas de `CODEOWNERS` sur `/pb/` (seul `/wp/` est listé dans `.github/CODEOWNERS`).
  - `validate-registry.yml` ne valide que `wp/**` (`on.pull_request.paths` ne liste que `wp/**`) —
    une PR qui modifierait `pb/pb.json` ne serait **pas couverte par le required status check**.
  - Le code R existant côté `ofceweb` (`R/pb_registry_request.R`, `R/pb_registry_sync.R`) ne pointe
    **pas** vers ce `pb/` de `OFCE/wp-registry` : son paramètre par défaut est
    `registry_repo = "ofce/pb-registry"` — un **autre dépôt**, distinct, qui n'existe pas dans ton
    espace de travail local (`web/pb-registry` n'existe pas, contrairement à `web/wp-registry`). Il
    attend en plus la structure `pb/index.json` + `pb/{année}.json`, pas le `pb.json` plat présent
    ici.
  - `packages/ofceweb/dev_pb.md` (notes de conception, trouvées en local) confirme que ce dépôt
    séparé `OFCE/pb-registry` a été créé *localement* à un moment du projet, « rempli sur le modèle
    de `wp-registry` », mais **jamais commité** — et le `pb/` flat qu'on voit ici semble être une
    tentative ultérieure et différente (schéma sans année, pas de CODEOWNERS/CI), pas une suite de
    ce travail.

  → Autrement dit : il y a deux tentatives de registre PB non réconciliées, et aucune des deux n'est
  branchée sur le code R qui est censé les lire.

Côté **page publique (`policy.yml`)**, la réponse est plus simple : **non**, il n'y a pas
d'équivalent qui fonctionnerait mieux que pour les WP, pour la bonne raison que **rien ne génère
`working_papers.yml` depuis le registre non plus** (point 1). Ce n'est donc pas que `policy.yml` ait
« raté » une automatisation que `working_papers.yml` aurait : aucun des deux ne l'a. C'est documenté
explicitement dans `packages/ofceweb/dev_pb.md`, section « 4.2 — Intégration webhome » :

> La collecte automatique (API GitHub sur registre/manifestes) est traitée comme un **chantier
> séparé, commun WP+PB**, hors du périmètre de ce plan. Le pipeline PB s'arrête donc au déploiement
> FTP + manifeste ; l'apparition dans `publications/policy.qmd` reste, pour l'instant, alimentée par
> le `policy.yml` statique existant (comme les WP via `working_papers.yml`).

Donc ta question initiale contenait une hypothèse à corriger : ce n'est pas que le workflow WP
marche et que le PB n'a pas encore rattrapé — le « workflow registre → yml » **n'a jamais été
construit**, ni pour l'un ni pour l'autre. Ce qui existe et fonctionne (registre → déploiement FTP
numéroté du document *lui-même*) est une chose différente de ce que tu cherches (registre → liste
affichée sur la page `policy.qmd`/`working_papers.qmd`).

---

## 3. Comment créer ce workflow (pistes, rien d'implémenté)

Deux couches à traiter séparément, puisqu'elles sont indépendantes aujourd'hui :

### 3.1 Remettre le registre PB en état (préalable indispensable)

Avant même de parler d'automatiser `policy.yml`, le registre PB lui-même a besoin d'être choisi et
cohérent — actuellement il y a deux designs concurrents :
- **Option A** : aligner `pb/` dans `OFCE/wp-registry` (ce dépôt) sur le modèle `wp/` — ajouter le
  champ `annee`, passer à `pb/{année}.json` + `pb/index.json`, ajouter `/pb/` à `CODEOWNERS`, étendre
  `validate-registry.yml` à `pb/**` (en dupliquant ses contrôles, adaptés) — puis changer le défaut
  `registry_repo` de `pb_registry_request()`/`sync_pb_registry_state()` dans `ofceweb` pour viser ce
  dépôt au lieu de `ofce/pb-registry`.
- **Option B** : créer réellement `OFCE/pb-registry` comme dépôt séparé (ce que le code R suppose
  déjà), avec la même gouvernance que `wp-registry` (CODEOWNERS, branch protection, CI de
  validation) — et migrer/supprimer le `pb/` actuel d'ici.

Sans trancher ça, toute automatisation construite par-dessus lira la mauvaise source ou une source
incomplète.

### 3.2 Construire le pont registre → yml (le vrai sujet de ta question)

Une fois le registre PB stabilisé, le pont vers `working_papers.yml`/`policy.yml` reste à construire
pour les deux types de documents. Deux approches possibles, pas exclusives :

- **Déclenchement sur merge (push)** : un workflow GitHub Actions dans `wp-registry` (ou dans
  `pb-registry` si Option B), sur `push` vers `main` touchant `wp/**`/`pb/**`, qui envoie un
  `repository_dispatch` (ou déclenche directement via un PAT un `workflow_dispatch`) vers
  `OFCE/webhome`. Un workflow côté `webhome` reconstruit alors `working_papers.yml`/`policy.yml`
  depuis `wp/index.json`+`wp/{année}.json` (et l'équivalent PB) via `raw.githubusercontent.com`, et
  committe le résultat (ou ouvre une PR, pour garder un œil humain avant publication sur le site).
- **Reconstruction périodique (cron) côté `webhome`** : plus simple à mettre en place côté
  gouvernance (pas besoin de permission inter-dépôts), un peu moins réactif. Un `schedule:` dans un
  nouveau workflow `webhome/.github/workflows/sync-registries.yml` qui tourne par ex. une fois par
  jour, lit les deux registres publics et régénère les deux YAML.

Dans les deux cas, il faut décider :
- **Que faire des entrées `pdf-only`** (pas de dépôt Quarto, juste un PDF déjà en ligne) : elles
  existent déjà en masse dans `pb/pb.json` et dans l'historique WP (cf. `dev_pb.md`, « 35 entrées
  ajoutées... à partir de `working_papers.yml` ») — le script de génération doit produire une entrée
  `yml` correcte à partir des seuls champs `pdf-path`/`type`, sans `source-repo`.
- **Que faire des entrées `repo`** dont le document Quarto n'a peut-être pas encore de métadonnées
  exploitables pour `working_papers.yml` (titre, auteurs, description, date) — ces infos vivent dans
  le dépôt WP/PB lui-même (`_quarto.yml`, `manifest.json`), pas dans le registre, qui ne stocke que
  `{année, numéro, source-repo, contact}`. Le script de synchronisation devra donc, pour les entrées
  `repo`, aller chercher ces métadonnées dans le dépôt source (API GitHub ou `manifest.json` déployé)
  — un vrai morceau de travail, pas une simple conversion JSON → YAML.
- **Garder un pas humain avant publication** (PR plutôt que commit direct sur `main` de `webhome`),
  pour éviter qu'une entrée mal formée ne casse silencieusement la page publique.

Ce chantier (« webhome : collecte automatique ») est explicitement noté comme **hors périmètre** et
**non commencé** dans `packages/ofceweb/dev_pb.md` (§4.2, §Phase 4) — ce document confirme qu'il n'y
a, à ce jour, aucun code ni aucun plan détaillé déjà écrit pour cette partie précise, au-delà de la
phrase citée plus haut.

---

## 4. Fichiers consultés pour cette analyse

- `wp-registry/.github/workflows/validate-registry.yml`, `wp-registry/.github/CODEOWNERS`
- `wp-registry/wp/*.json`, `wp-registry/pb/*.json` (structure réelle actuelle)
- `web/webhome/README.md`, `web/webhome/publications/{working_papers,policy}.qmd`
- `web/webhome/scripts/generate_working_papers_yaml.R`, `generate_pbrief_yaml.R` (scripts one-shot)
- `packages/ofceweb/R/{wp,pb}_registry_{request,sync}.R`, `deploy_{wp,pb}.R`
- `packages/ofceweb/plans/note-equipe-publication-wp.md` (note d'équipe, processus WP)
- `packages/ofceweb/dev_pb.md` (notes de conception du pipeline PB, confirme le périmètre hors-sujet
  pour l'intégration webhome)
