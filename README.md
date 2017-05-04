# vue-ssr-docs

### Traduction

Cette branche de travail `working` est volontairement mise en avant et doit uniquement être mise à jour dans le sens :

`vuejs/vue-ssr-docs:master` --> `vuejs-fr/vue-ssr-docs:working`.

Nous traduisons les fichiers directement dans le dossier `en` sans les renommer. Cela permet lors de la mise à jour de la documentation via l'utilisation des commandes :

```
git fetch upstream
git merge upstream/master
```

d'obtenir des conflits **sur les pages déjà traduites** et ainsi maintenir la documentation à jour en fonction des modifications à travers **les documents déjà traduits**.

### Reverssement

Quand un fichier traduit est validé par pull request, on le met à jour dans le dossier `fr` de `vuejs-fr/vue-ssr-docs:master` puis on propose une pull request au site principal :

`vuejs-fr/vue-ssr-docs:master` --> `vuejs/vue-ssr-docs:master`

ainsi le dossier officiel hébergeant la documentation possède bien le dossier `fr` en français et le dosser `en` en anglais.

Note : il peut être intéressant de faire une pull request par ficher validé et donc de créer une branche dérivée de `vuejs-fr/vue-ssr-docs:master` pour faire la pull request (`vuejs-fr/vue-ssr-docs:master` --> `vuejs-fr/vue-ssr-docs:only_one_changed_file_from_master` --> `vuejs/vue-ssr-docs:master`)