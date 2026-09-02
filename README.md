# wp-registry
## Registre des documents de travail de l'OFCE

Pour être publié, un document de travail doit être enregistré ici. Pour faire un enregistrement, il faut faire une PR avec le nouveau document de travail et cette PR doit être approuvée par un administrateur. La fonction `ofceweb::wp_registry_request()` automatise la PR.

Un document de travail doit être un dépôt de l'organisation OFCE et doit être configuré par la fonction `ofceweb::setup_wp()`. Il est nécessaire qu'il se compile sans erreur (`ofceweb::render()`)

Essai
