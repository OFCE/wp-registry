# wp-registry
## Registre des documents de travail de l'OFCE

Pour être publié, un document de travail doit être enregistré ici. Pour faire un enregistrement, il faut faire une PR avec le nouveau document de travail et cette PR doit être approuvée par un administrateur. La fonction `ofceweb::wp_registry_request()` automatise la PR.

Un document de travail doit être un dépôt de l'organisation OFCE et doit être configuré par la fonction `ofceweb::setup_wp()`. Il est nécessaire qu'il se compile sans erreur (`ofceweb::render()`)

## Registre des policy briefs (`pb/`)

Les policy briefs sont répartis en **séries**, une par institut d'origine. Chaque série a sa propre numérotation, qui commence à 1, dans son propre fichier `pb/{série}.json`. La série est donnée par la clé `pb_series` du `_quarto.yml` du dépôt PB ; elle est indépendante de la clé `institut`, qui ne choisit que le logo.

- `pb/index.json` déclare les séries et leurs paramètres : `open` (la série accepte-t-elle de nouveaux numéros ?), préfixe d'URL sous `www.ofce.fr/pb/`, préfixes des PDF publié et brouillon, `container-title` de la citation. Fermer une série ou en ajouter une se fait ici, sans nouvelle version d'`ofceweb`.
- `pb/{série}.json` contient les entrées de la série (schéma : `pb/registry.schema.json`). Chaque entrée porte `pb_series`, égal au nom du fichier.
- `pb/pb.json` est l'ancien registre plat (une seule numérotation). Il est **figé** : la CI refuse toute PR qui le modifie.

La fonction `ofceweb::pb_registry_request()` automatise la demande de numéro. La CI (`validate-registry.yml`) refuse notamment un numéro en double dans une série, un dépôt enregistré deux fois, et tout nouveau numéro dans une série fermée.
