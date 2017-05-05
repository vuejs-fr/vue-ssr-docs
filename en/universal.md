# Écrire du code universel (En) <br><br> *Cette page est en cours de traduction française. Revenez une autre fois pour lire une traduction achevée ou [participez à la traduction française ici](https://github.com/vuejs-fr/vue-ssr-docs).*

Avant d'aller plus loin, prenons un moment pour discuter des contraintes lorsque l'on écrit du code "universel" - c'est à dire du code qui s'exécute à la fois côté serveur et côté client. En raison des différences d'API des deux plate-formes, le comportement de notre code ne sera pas exactement le même selon l'environnement. Nous allons examiner les points clés dont vous devez avoir connaissance.

## Réactivité des données côté serveur

Dans une application qui tourne exclusivement côté client, chaque utilisateur utilisera une nouvelle instance de l'application dans leur navigateur. Pour le rendu serveur nous voulons le même fonctionnement: chaque requête doit disposer d'une nouvelle instance d'application, et isolée. Ainsi il n'y aura pas de pollution liée à du partage d'état multi-requêtes.

Etant donné que le processus de rendu actuel doit être déterministe, nous allons aussi "pré-charger" les données sur le serveur - cela signifie que l'état de notre application sera déjà disponible quand nous commençons le rendu. Cela signifie que la réactivité des données est inutile côté serveur, et est donc désactivée par défaut. Désactiver la réactivité des données évite aussi le coût en performance de conversion des données en objets réactifs.

## Hooks de cycles de vie des composants

Vu qu'il n'y a pas de mises à jour dynamiques, de tous les connecteurs de cycles de vie, seuls `beforeCreate` et `created` seront appelés pendant le rendu côté serveur. Cela signifie que tout code présent dans d'autres connecteurs tels que `beforeMount` ou `mounted` sera exécuté uniquement côté client.

## Accès aux APIs spécifique à la plate-forme

Le code universel ne peut pas accéder aux APIs spécifiques à une plate-forme. Ainsi, si votre code utilise directement les variables globales exclusives au navigateur comme `window` ou `document`, elles lèveront des erreurs si elles sont exécutées sur Node.js, et vice-versa.

Pour les tâches partagées entre le serveur et le client, mais qui utilisent des APIs différentes selon la plate-forme, il est recommandé d'encapsuler le code spécifique à la plate-forme dans une API universelle, ou d'utiliser des bibliothèques qui le font pour vous. Par exemple, [axios](https://github.com/mzabriskie/axios) est un client HTTP qui présente la même API côté serveur et client.

Pour les APIs exclusives au navigateur, l'approche habituelle est de les utiliser dans les connecteurs de cycle de vie exclusifs au client.

Notez que si une bibliothèque tierce n'est pas écrite avec l'objectif d'être universelle, cela peut être délicat de l'intégrer dans une application rendue côté serveur. Vous *devriez* être capable de la faire fonctionner en substituant certaines variables globales, mais cela serait cavalier et pourrait interférer avec du code de détection de l'environnement d'autres bibliothèques.

## Directives personnalisées

La plupart des directives personnalisées manipulent directement le DOM, et vont ainsi provoquer des erreurs durant le rendu côté serveur. Il y a deux façons d'éviter cela :

1. Préférer l'utilisation de composants comme mécanisme d'abstraction et travailler au niveau du DOM Virtuel (par ex. en utilisant des fonctions de rendu)

2. Si vous avez une directive personnalisée qui ne peut être facilement remplacée par des composants, vous pouvez en fournir une "version serveur" qui utilise l'option [`directives`](./api.md#directives) lors de la création du rendu serveur.