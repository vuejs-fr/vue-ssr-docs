# Réference de l'API

## `createRenderer([options])`

Créer une instance de [`Renderer`](#class-renderer) avec des [options](#renderer-options) optionelles.

``` js
const { createRenderer } = require('vue-server-renderer')
const renderer = createRenderer({ ... })
```

## `createBundleRenderer(bundle[, options])`

Créer une instance de [`BundleRenderer`](#class-bundlerenderer) avec un paquetage serveur et des [options](#renderer-options) optionelles.

``` js
const { createBundleRenderer } = require('vue-server-renderer')
const renderer = createBundleRenderer(serverBundle, { ... })
```

L'argument `serverBundle` peut être l'un des éléments suivants :

- Un chemin absolue pour générer un fichier de paquetage (`.js` ou `.json`). Doit commencer par `/` pour être considéré comme un chemin de fichier.

- Un objet de paquetage généré par webpack + `vue-server-renderer/server-plugin`.

- Une chaîne de caractères de code JavaScript (non recommandé).

Voyez l'[Introduction au moteur de dépaquetage](./bundle-renderer.md) et la [Configuration de pré-compilation](./build-config.md) pour plus de détails.

## `Class: Renderer`

- #### `renderer.renderToString(vm[, context], callback)`

  Faire le rendu d'une instance de Vue sous forme de chaîne de caractères. L'objet de contexte est optionnel. La fonction de rappel est une fonction de rappel typique de Node.js avec en premier argument l'erreur potentielle et en second argument la chaîne de caractères du rendu.

- #### `renderer.renderToStream(vm[, context])`

  Faire le rendu d'une instance de Vue sous forme de flux. L'objet de contexte est optionnel. Voir l'[Envoi par flux](./streaming.md) pour plus de détails.

## `Class: BundleRenderer`

- #### `bundleRenderer.renderToString([context, ]callback)`

  Faire le rendu d'un paquetage sous forme de chaîne de caractères. L'objet de contexte est optionnel. La fonction de rappel est une fonction de rappel typique de Node.js avec en premier argument l'erreur potentielle et en second argument la chaîne de caractères du rendu.

- #### `bundleRenderer.renderToStream([context])`

  Faire le rendu d'un paquetage sous forme de flux. L'objet de contexte est optionnel. Voir l'[Envoi par flux](./streaming.md) pour plus de détails.

## Options de `Renderer`

- #### `template`

  Fournit un modèle de page pour la page HTML complète. Le modèle de page devrait contenir en commentaire `<!--vue-ssr-outlet-->` qui permet de définir l'emplacement du contenu de chaque rendu de l'application.

  Le modèle de page supporte également l'interpolation basique en utilisant le contexte du rendu :

  - Utilisez les double moustaches pour de l'interpolation avec HTML échappé et
  - Utilisez les triple moustaches pour de l'interpolation avec HTML non échappé.

  Le modèle de page injecte automatiquement le contenu quand certaines données sont trouvées dans le contexte du rendu :

  - `context.head`: (string) n'importe quel balise d'entête qui devrait être injectée dans la balise `<head>` de la page.

  - `context.styles`: (string) n'importe quel CSS qui devrait être injecté dans la balide `<head>` de la page. Notez que cette propriété va automatiquement être injectée si vous utilisez `vue-loader` + `vue-style-loader` pour la CSS de vos composants.

  - `context.state`: (Object) L'état initial du store Vuex devrait être injecté dans la page sous la variable `window.__INITIAL_STATE__`. Le JSON en ligne est automatiquement désinfecté avec [serialize-javascript](https://github.com/yahoo/serialize-javascript) pour éviter les injections XSS.

  En plus, quand `clientManifest` est fourni, le modèle de page injecte automatiquement les éléments suivants :

  - JavaScript client et fichiers CSS nécessaire pour le rendu (avec les fragments asynchrones automatiquement déduit),
  - Utilisation optimal des indices de ressources `<link rel="preload/prefetch">` pour le rendu de la page.

  Vous pouvez désactiver toutes ces injections en passant `inject: false` au moteur de rendu.

  Voir également :

  - [Utiliser un modèle de page](./basic.md#utiliser-un-modele-de-page)
  - [Injection manuel des fichiers](./build-config.md#injection-manuel-des-fichiers)

- #### `clientManifest`

  - 2.3.0+

  Fournit un objet de build de manifeste généré par `vue-server-renderer/client-plugin`. Le manifeste client fournit le paquetage de moteur de rendu avec ses propres informations pour l'injection automatique de fichier dans le modèle de page HTML. Pour plus de détails, consultez [Générer le `clientManifest`](./build-config.md#generer-le-clientmanifest).

- #### `inject`

  - 2.3.0+

  Controls whether to perform automatic injections when using `template`. Defaults to `true`.

  See also: [Manual Asset Injection](./build-config.md#manual-asset-injection).

- #### `shouldPreload`

  - 2.3.0+

  A function to control what files should have `<link rel="preload">` resource hints generated.

  By default, only JavaScript and CSS files will be preloaded, as they are absolutely needed for your application to boot.

  For other types of assets such as images or fonts, preloading too much may waste bandwidth and even hurt performance, so what to preload will be scenario-dependent. You can control precisely what to preload using the `shouldPreload` option:

  ``` js
  const renderer = createBundleRenderer(bundle, {
    template,
    clientManifest,
    shouldPreload: (file, type) => {
      // type is inferred based on the file extension.
      // https://fetch.spec.whatwg.org/#concept-request-destination
      if (type === 'script' || type === 'style') {
        return true
      }
      if (type === 'font') {
        // only preload woff2 fonts
        return /\.woff2$/.test(file)
      }
      if (type === 'image') {
        // only preload important images
        return file === 'hero.jpg'
      }
    }
  })
  ```

- #### `runInNewContext`

  - 2.3.0+
  - only used in `createBundleRenderer`
  - Expects: `boolean | 'once'` (`'once'` only supported in 2.3.1+)

  By default, for each render the bundle renderer will create a fresh V8 context and re-execute the entire bundle. This has some benefits - for example, the app code is isolated from the server process and we don't need to worry about the [stateful singleton problem](./structure.md#avoid-stateful-singletons) mentioned in the docs. However, this mode comes at some considerable performance cost because re-executing the bundle is expensive especially when the app gets bigger.

  This option defaults to `true` for backwards compatibility, but it is recommended to use `runInNewContext: false` or `runInNewContext: 'once'` whenever you can.

  > In 2.3.0 this option has a bug where `runInNewContext: false` still executes the bundle using a separate global context. The following information assumes version 2.3.1+.

  With `runInNewContext: false`, the bundle code will run in the same `global` context with the server process, so be careful about code that modifies `global` in your application code.

  With `runInNewContext: 'once'` (2.3.1+), the bundle is evaluated in a separate `global` context, however only once at startup. This provides better app code isolation since it prevents the bundle from accidentally polluting the server process' `global` object. The caveats are that:

  1. Dependencies that modifies `global` (e.g. polyfills) cannot be externalized in this mode;
  2. Values returned from the bundle execution will be using different global constructors, e.g. an error caught inside the bundle will not be an instance of `Error` in the server process.

  See also: [Source Code Structure](./structure.md)

- #### `basedir`

  - 2.2.0+
  - only used in `createBundleRenderer`

  Explicitly declare the base directory for the server bundle to resolve `node_modules` dependencies from. This is only needed if your generated bundle file is placed in a different location from where the externalized NPM dependencies are installed, or your `vue-server-renderer` is npm-linked into your current project.

- #### `cache`

  Provide a [component cache](./caching.md#component-level-caching) implementation. The cache object must implement the following interface (using Flow notations):

  ``` js
  type RenderCache = {
    get: (key: string, cb?: Function) => string | void;
    set: (key: string, val: string) => void;
    has?: (key: string, cb?: Function) => boolean | void;
  };
  ```

  A typical usage is passing in an [lru-cache](https://github.com/isaacs/node-lru-cache):

  ``` js
  const LRU = require('lru-cache')

  const renderer = createRenderer({
    cache: LRU({
      max: 10000
    })
  })
  ```

  Note that the cache object should at least implement `get` and `set`. In addition, `get` and `has` can be optionally async if they accept a second argument as callback. This allows the cache to make use of async APIs, e.g. a redis client:

  ``` js
  const renderer = createRenderer({
    cache: {
      get: (key, cb) => {
        redisClient.get(key, (err, res) => {
          // handle error if any
          cb(res)
        })
      },
      set: (key, val) => {
        redisClient.set(key, val)
      }
    }
  })
  ```

- #### `directives`

  Allows you to provide server-side implementations for your custom directives:

  ``` js
  const renderer = createRenderer({
    directives: {
      example (vnode, directiveMeta) {
        // transform vnode based on directive binding metadata
      }
    }
  })
  ```

  As an example, check out [`v-show`'s server-side implementation](https://github.com/vuejs/vue/blob/dev/src/platforms/web/server/directives/show.js).

## webpack Plugins

The webpack plugins are provided as standalone files and should be required directly:

``` js
const VueSSRServerPlugin = require('vue-server-renderer/server-plugin')
const VueSSRClientPlugin = require('vue-server-renderer/client-plugin')
```

The default files generated are:

- `vue-ssr-server-bundle.json` for the server plugin;
- `vue-ssr-client-manifest.json` for the client plugin.

The filenames can be customized when creating the plugin instances:

``` js
const plugin = new VueSSRServerPlugin({
  filename: 'my-server-bundle.json'
})
```

See [Build Configuration](./build-config.md) for more information.
