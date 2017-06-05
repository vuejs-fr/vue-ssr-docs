# Gestion des CSS

La gestion recommandée pour l'utilisation des CSS est de simplement utiliser une balise `<style>` à l'intérieur d'un fichier de composant monopage `*.vue`. Cela permet :

- Une sortie CSS limitée au composant
- La possibilité d'utiliser des pré-processeur ou PostCSS
- Le rechargement à chaud pendant le développement

Et plus important, `vue-style-loader`, le loader utilisé en interne par `vue-loader`, a plusieurs fonctionnalités pour le rendu côté serveur :

- Expérience de création universelle côté client et côté serveur.

- Création automatique de CSS critique lors de l'utilisation de `bundleRenderer`.

  S'il est utilisé pendant le rendu côté serveur, un composant CSS peut être récupérer et injecté dans la source HTML (en étant automatiquement gérer avec l'option `template`). Côté client, quand le composant est utilisé pour la première fois, `vue-style-loader` va vérifier s'il n'y a pas déjà une sortie CSS dans la source HTML pour ce composant ; si non, le CSS va être automatiquement injecté via une balise `<stlye>`.

- Extraction CSS commune.

  Cette utilisation supporte [`extract-text-webpack-plugin`](https://github.com/webpack-contrib/extract-text-webpack-plugin) pour extraire le CSS du fragment principal en an fichier CSS séparé (automatiquement injecté avec l'option `template`), ce qui permet au fichier d'être mis en cache individuellement. Cela est recommandé quand il y a beaucoup de CSS partagés.

  Les CSS a l'intérieur des composants asynchrones vont être compilé en tant que chaîne de caractère JavaScript et pris en charge par `vue-style-loader`.

## Activer l'extraction CSS

Pour extraire le CSS des fichiers `*.vue`, utilisez l'option `extractCSS` du `vue-loader` (requiert `vue-loader>=12.0.0`) :

``` js
// webpack.config.js
const ExtractTextPlugin = require('extract-text-webpack-plugin')

// L'extraction CSS devrait uniquement être activé en production
// ainsi vous pourriez utiliser le rechargement à chaud pendant le development.
const isProduction = process.env.NODE_ENV === 'production'

module.exports = {
  // ...
  module: {
    rules: [
      {
        test: /\.vue$/,
        loader: 'vue-loader',
        options: {
          // activer l'extraction CSS
          extractCSS: isProduction
        }
      },
      // ...
    ]
  },
  plugins: isProduction
    // assurez-vous d'avoir ajouté le plugin !
    ? [new ExtractTextPlugin({ filename: 'common.[chunkhash].css' })]
    : []
}
```

Notez que la configuration ci-dessus est uniquement appliquée pour les styles en provenance des fichiers `*.vue`, mais vous pouvez toujours utiliser `<style src="./foo.css">` pour importer des CSS externes dans des composants Vue.

Si vous souhaitez importer des CSS depuis le JavaScript, par ex. avec `import 'foo.css'`, vous aurez besoin de configurer les loaders appropriés :

``` js
module.exports = {
  // ...
  module: {
    rules: [
      {
        test: /\.css$/,
        // important : utilisez `vue-style-loader` à la place de `style-loader`
        use: isProduction
          ? ExtractTextPlugin.extract({
              use: 'css-loader',
              fallback: 'vue-style-loader'
            })
          : ['vue-style-loader', 'css-loader']
      }
    ]
  },
  // ...
}
```

## Importer des styles depuis les dépendances

Quelque choses que vous devez prendre en note quand vous importez des CSS depuis des dépendances npm :

1. Il ne devrait pas être externalisés dans un build serveur.

2. Si vous utilisez de l'extraction CSS + de l'extraction de CSS tierces avec `CommonsChunkPlugin`, `extract-text-webpack-plugin` va mal fonctionner si le CSS extrait est à l'intérieur d'un extrait de fragment tierces. Pour résoudre cela, évitez d'inclure des fichiers CSS dans un fragment tier. Voici un exemple de configuration côté client avec webpack :

  ``` js
  module.exports = {
    // ...
    plugins: [
      // il est normal d'extraire un fragment tier pour une meilleure mise en cache.
      new webpack.optimize.CommonsChunkPlugin({
        name: 'vendor',
        minChunks: function (module) {
          // un module est extrait dans un fragment tier puis...
          return (
            // s'il est a l'intérieur d'un node_modules
            /node_modules/.test(module.context) &&
            // ne pas l'externaliser si la requête est un fichier CSS
            !/\.css$/.test(module.request)
          )
        }
      }),
      // extraction de l'exécuteur et du manifeste webpack
      new webpack.optimize.CommonsChunkPlugin({
        name: 'manifest'
      }),
      // ...
    ]
  }
  ```
