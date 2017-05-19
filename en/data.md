# Pré-chargement et état

## Gestionnaire d'état des données

Pendant le SSR, nous allons essentiellement faire le rendu d'un « instantané » de notre application, aussi si votre applications est liée à des données asynchrones, **c'est données vont devoir être pré-chargée et résolue avant de débuter la phase de rendu**.

Un autre point important côté client, les mêmes données doivent être disponibles avant que l'application ne soit montée, autrement, l'application côté client va faire le rendu d'un état différent et l'hydratation va échouée.

Pour résoudre cela, les données pré-chargées doivent vivre en dehors de la vue du composant, dans un gestionnaire de données, ou dans un « gestionnaire d'état ». Côté serveur, nous pouvons pré-chargé et remplir les données dans le gestionnaire de donnée avant le rendu. De plus, nous allons en sérialiser et en injecter l'état dans le HTML. Le gestionnaire de données côté client poura directement récupérer l'état depuis le HTML avant que l'application ne soit montée.

Nous allons utiliser le gestionnaire d'état officiel (« store ») de la bibliothèque [Vuex](https://github.com/vuejs/vuex/) pour cette partie. Créons un fichier `store.js`, avec divers jeu de logique pour pré-chargé un élément en nous basant sur un identifiant :

``` js
// store.js
import Vue from 'vue'
import Vuex from 'vuex'

Vue.use(Vuex)

// Supposons que nous ayons une API universelle retournant
// des Promesses (« Promise ») et ignorons les détails de l'implémentation
import { fetchItem } from './api'

export function createStore () {
  return new Vuex.Store({
    state: {
      items: {}
    },
    actions: {
      fetchItem ({ commit }, id) {
        // retournant la Promesse via `store.dispatch()` nous savons
        // quand les données ont été pré-chargées
        return fetchItem(id).then(item => {
          commit('setItem', { id, item })
        })
      }
    },
    mutations: {
      setItem (state, { id, item }) {
        Vue.set(state.items, id, item)
      }
    }
  })
}
```

et mettons à jour `app.js`:

``` js
// app.js
import Vue from 'vue'
import App from './App.vue'
import { createRouter } from './router'
import { createStore } from './store'
import { sync } from 'vuex-router-sync'

export function createApp () {
  // créer le routeur et l'instance du store
  const router = createRouter()
  const store = createStore()

  // synchroniser pour que l'état de la route soit disponible en tant que donnée du store
  sync(store, router)

  // créer l'instance de l'app, injecter le routeur et le store
  const app = new Vue({
    router,
    store,
    render: h => h(App)
  })

  // exposer l'application, le routeur et le store.
  return { app, router, store }
}
```

## Collocation logique avec les composants

Donc, ou devons nous appeler le code en charge de l'action de pré-chargement ?

Les données que nous avons besoin de pré-chargée sont déterminée par la route visitée, qui va aussi déterminé quel composant va être rendu. En fait, les données nécéssaire a une route donnée sont aussi les données nécéssaire au composant pour être rendu pour une route. Aussi il serait naturel de placer la logique de pré-chargement à l'intérieur des composants de route.

Nous allons exposer une fonction statique personnalisée `asyncData` sur nos composants de route. Notez que, puisque cette fonction va être appelée avant l'instanciation des composants, l'accès à `this` n'est pas possible. Le store et les informations de route ont donc besoin d'être passés en tant qu'arguments :

``` html
<!-- Item.vue -->
<template>
  <div>{{ item.title }}</div>
</template>

<script>
export default {
  asyncData ({ store, route }) {
    // retourne la Promesse depuis l'action
    return store.dispatch('fetchItem', route.params.id)
  },

  computed: {
    // affiche l'élément depuis l'état du store.
    item () {
      return this.$store.state.items[this.$route.params.id]
    }
  }
}
</script>
```

## Pré-chargement de données côté serveur

Dans `entry-server.js` nous pouvons obtenir des composants qu'ils concordent avec une route grâce à `router.getMatchedComponents()`, et appeler `asyncData` si le composant l'expose. Nous avons ensuite besoin d'attacher l'état résolue au contexte de rendu.

``` js
// entry-server.js
import { createApp } from './app'

export default context => {
  return new Promise((resolve, reject) => {
    const { app, router, store } = createApp()

    router.push(context.url)

    router.onReady(() => {
      const matchedComponents = router.getMatchedComponents()
      if (!matchedComponents.length) {
        reject({ code: 404 })
      }

      // appeler `asyncData()` sur toutes les routes concordant
      Promise.all(matchedComponents.map(Component => {
        if (Component.asyncData) {
          return Component.asyncData({
            store,
            route: router.currentRoute
          })
        }
      })).then(() => {
        // Après que chaque hook de pré-chargement soit résolue, notre store est maintenant
        // rempli avec l'état nécéssaire au rendu de l'application.
        // Quand nous attachons l'état au contexte, et que l'option `template`
        // est utilisée pour faire le rendu, l'état va automatiquement être
        // sérialisé et injetté dans le HTML pour alimenter window.__INITIAL_STATE__.
        context.state = store.state

        resolve(app)
      }).catch(reject)
    }, reject)
  })
}
```

En utilisant `template`, `context.state` va automatiquement être encapsulé dans le HTML final en tant que qu'état `window.__INITIAL_STATE__`. Côté client, le store voudra récupérer cet état avant de monter l'application :

``` js
// entry-client.js

const { app, router, store } = createApp()

if (window.__INITIAL_STATE__) {
  store.replaceState(window.__INITIAL_STATE__)
}
```

## Pré-chargement de données côté client

On the client, there are two different approaches for handling data fetching:

1. **Resolve data before route navigation:**

  With this strategy, the app will stay on the current view until the data needed by the incoming view has been resolved. The benefit is that the incoming view can directly render the full content when it's ready, but if the data fetching takes a long time, the user will feel "stuck" on the current view. It is therefore recommended to provide a data loading indicator if using this strategy.

  We can implement this strategy on the client by checking matched components and invoking their `asyncData` function inside a global route hook. Note we should register this hook after the initial route is ready so that we don't unnecessarily fetch the server-fetched data again.

  ``` js
  // entry-client.js

  // ...omitting unrelated code

  router.onReady(() => {
    // Add router hook for handling asyncData.
    // Doing it after initial route is resolved so that we don't double-fetch
    // the data that we already have. Using router.beforeResolve() so that all
    // async components are resolved.
    router.beforeResolve((to, from, next) => {
      const matched = router.getMatchedComponents(to)
      const prevMatched = router.getMatchedComponents(from)

      // we only care about none-previously-rendered components,
      // so we compare them until the two matched lists differ
      let diffed = false
      const activated = matched.filter((c, i) => {
        return diffed || (diffed = (prevMatched[i] !== c))
      })

      if (!activated.length) {
        return next()
      }

      // this is where we should trigger a loading indicator if there is one

      Promise.all(activated.map(c => {
        if (c.asyncData) {
          return c.asyncData({ store, route: to })
        }
      })).then(() => {

        // stop loading indicator

        next()
      }).catch(next)
    })

    app.$mount('#app')
  })
  ```

2. **Fetch data after the matched view is rendered:**

  This strategy places the client-side data-fetching logic in a view component's `beforeMount` function. This allows the views to switch instantly when a route navigation is triggered, so the app feels a bit more responsive. However, the incoming view will not have the full data available when it's rendered. It is therefore necessary to have a conditional loading state for each view component that uses this strategy.

  This can be achieved with a client-only global mixin:

  ``` js
  Vue.mixin({
    beforeMount () {
      const { asyncData } = this.$options
      if (asyncData) {
        // assign the fetch operation to a promise
        // so that in components we can do `this.dataPromise.then(...)` to
        // perform other tasks after data is ready
        this.dataPromise = asyncData({
          store: this.$store,
          route: this.$route
        })
      }
    }
  })
  ```

The two strategies are ultimately different UX decisions and should be picked based on the actual scenario of the app you are building. But regardless of which strategy you pick, the `asyncData` function should also be called when a route component is reused (same route, but params or query changed. e.g. from `user/1` to `user/2`). We can also handle this with a client-only global mixin:

``` js
Vue.mixin({
  beforeRouteUpdate (to, from, next) {
    const { asyncData } = this.$options
    if (asyncData) {
      asyncData({
        store: this.$store,
        route: to
      }).then(next).catch(next)
    } else {
      next()
    }
  }
})
```

## Store Code Splitting

In a large application, our vuex store will likely be split into multiple modules. Of course, it is also possible to code-split these modules into corresponding route component chunks. Suppose we have the following store module:

``` js
// store/modules/foo.js
export default {
  namespaced: true,
  // IMPORTANT: state must be a function so the module can be
  // instantiated multiple times
  state: () => ({
    count: 0
  }),
  actions: {
    inc: ({ commit }) => commit('inc')
  },
  mutations: {
    inc: state => state.count++
  }
}
```

We can use `store.registerModule` to lazy-register this module in a route component's `asyncData` hook:

``` html
// inside a route component
<template>
  <div>{{ fooCount }}</div>
</template>

<script>
// import the module here instead of in `store/index.js`
import fooStoreModule from '../store/modules/foo'

export default {
  asyncData ({ store }) {
    store.registerModule('foo', fooStoreModule)
    return store.dispatch('foo/inc')
  },

  // IMPORTANT: avoid duplicate module registration on the client
  // when the route is visited multiple times.
  destroyed () {
    this.$store.unregisterModule('foo')
  },

  computed: {
    fooCount () {
      return this.$store.state.foo.count
    }
  }
}
</script>
```

Because the module is now a dependency of the route component, it will be moved into the route component's async chunk by webpack.

---

Phew, that was a lot of code! This is because universal data-fetching is probably the most complex problem in a server-rendered app and we are laying the groundwork for easier further development. Once the boilerplate is set up, authoring individual components will be actually quite pleasant.
