# Introduction à l'empaquetage

## Problèmes du SSR de base

À ce point, nous supposons que le code empaqueter côté serveur sera directement utilisé via `require` :

``` js
const createApp = require('/path/to/built-server-bundle.js')
```

Cela est simple, mais à ce stade, à chaque fois que vous éditez votre code source, vous devez stopper et redémarrer votre serveur. Cela ralenti quelque peu la productivité pendant le développement. De plus, Node.js ne supporte pas le mapping de code source nativement.

## Le monde de l'empaquetage

`vue-server-renderer` fournit une API appelée `createBundleRenderer` pour résoudre ce problème. Avec un plugin webpack personnalisé, le paquet (« bundle ») serveur est généré comme un fichier JSON spécial qui peut être passé au moteur de dépaquetage (« bundle renderer »). Une fois que le moteur de dépaquetage est créé, l'usage est le même qu'un moteur de rendu, cependant le moteur de dépaquetage fournit les bénéfices suivants :

- Support du mapping de source inclus (avec `devtool: 'source-map'` dans la configuration de webpack)

- Rechargement à chaud pendant la phase de développement et même de déploiement (en relisant le paquet mis à jour et en re-créant l'instance du moteur)

- Injection CSS critique (en utilisant les fichiers `*.vue`) : insérer automatiqument dans le rendu le CSS nécéssaire pour les composants pendant le rendu. Voir la section [CSS](./css.md) pour plus de détails.

- Injection d'assets avec [clientManifest](./api.md#clientmanifest) : déduire automatiquement le pré-chargement et la récupération des directives, et les fragments scindés requis pour le rendu initial.

---

Nous allons discuter de la manière de configurer webpack pour générer les artefacts de build nécessaire au moteur de dépaquetage dans la prochaine section, mais pour le moment, imaginons que nous avons déjà ce dont nous avons besoin. Voici comment créer et utiliser un moteur de dépaquetage :

``` js
const { createBundleRenderer } = require('vue-server-renderer')

const renderer = createBundleRenderer(serverBundle, {
  runInNewContext: false, // recommandé
  template, // (optionnel) page de template
  clientManifest // (optionnel) manifeste de build client
})

// à l'intérieur du gestionnaire serveur...
server.get('*', (req, res) => {
  const context = { url: req.url }
  // Pas besoin de passer l'application ici car elle est automatiquement créée
  // à l'exécution du paquet. Maintenant notre serveur est découplé de notre application Vue !
  renderer.renderToString(context, (err, html) => {
    // gérér les erreurs...
    res.end(html)
  })
})
```

Quand `renderToString` est appelé sur le moteur de dépaquetage, il va automatiquement exécuté la fonction exportée par le paquet pour créer une instance de l'application (en passant `context` comme argument), et puis va en faire le rendu.

Notons qu'il est recommander de mettre l'option `runInNewContext` à `false` ou `'once'`. Plus de détails dans [la référence de l'API](./api.md#runinnewcontext).
