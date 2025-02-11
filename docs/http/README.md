# HTTP Requests

## Architecture

Dans une *Single Page Application (SPA)*, la communication avec le serveur se fait via des requêtes HTTP asynchrones (*AJAX*) ou des protocoles plus spécialisés comme les WebSocket. Nous allons voir comment faire ces requêtes réseau depuis une application Vue.

Vue.js est un framework qui se focalise sur l'interface utilisateur et ne propose rien de particulier en ce qui concerne les échanges avec le serveur, préférant laisser ce choix d'implémentation au développeur. Le moyen le plus simple de lancer une requête HTTP asynchrone en JavaScript est d'utiliser [la méthode `fetch`](https://developer.mozilla.org/fr/docs/Web/API/Fetch_API/Using_Fetch). Elle n'est pas supportée sur Internet Explorer mais il existe des polyfills pour compléter le support. Comme alternative, vous pouvez passer par d'autres bibliothèques tierces plus complètes telles que [Axios](https://github.com/axios/axios), qui est recommandée par l'équipe de Vue.

::: tip
On évitera de faire les appels réseau directement depuis le code d'un composant. Cela empêche de réutiliser le code facilement si un autre composant doit faire le même appel. Optez plutôt pour systématiquement séparer la logique applicative de la logique de vos vues.

Par convention, on code la logique applicative dans des fichiers JS appelés *services*, répartis selon les grands pans fonctionnels de votre application et placés dans un dossier `src/services`
:::

## TP: Echanger avec un back-end

Nous allons nous servir d'une API fournie par un serveur (le *back-end*) pour authentifier les utilisateurs et rechercher des films. Ce back-end a déjà été développé et déployé sur Heroku.

::: tip
Le contrat d'interface du back-end est disponible ici: [api-docs](https://vue-js-backend.herokuapp.com/api-docs)
:::

1. Créer un service générique `services/api.js` permettant d'appeler le backend, avec ce contenu:

```js
import store from '@/store.js'

export const BASE_URL = 'https://vue-js-backend.herokuapp.com'

export async function api (url, params = {}) {
    params = Object.assign({
        mode: 'cors',
        cache: 'no-cache',
    }, params)

    params.headers = Object.assign({
        Authorization: `Bearer ${store.state.token}`,
        'Content-Type': 'application/json'
    }, params.headers)

    let response = await fetch(BASE_URL + url, params)
    let json = await response.json() || {}
    if (!response.ok){
        let errorMessage = json.error ? json.error.error || json.error : response.status;        
        throw new Error(errorMessage);
    }
    return json
}
```

Il n'y a pas de code spécifique à Vue ici, il s'agit d'une fonction utilitaire autour de la méthode `fetch` branchée sur notre back-end. Le header `Authorization` est utilisé pour envoyer le token d'authentification au back-end. Les autres options servent à configurer le cache HTTP et les autorisations cross-origin à appliquer.

2. Créer un service `services/UserService.js` qui exploitera les endpoints de l'API concernant l'inscription et le login des utilisateurs:

```js
import { api } from '@/services/api.js'

export default {
  register (credentials) {
    return api('/user/register', {
      method: 'POST',
      body: JSON.stringify(credentials)
    })
  },
  login (credentials) {
    return api('/user/login', {
      method: 'POST',
      body: JSON.stringify(credentials)
    })
  },
  user () {
    return api('/user')
  }
}
```

3. Dans le composant `LoginForm`, ajoutez un second bouton pour s'inscrire à côté de celui pour se login, puis modifiez les méthodes de `LoginForm` pour appeler les méthodes du `UserService`:

```js
import UserService from '@/services/UserService.js'

export default {
  methods: {
    async register () {
      try {
        const response = await UserService.register({
          email: this.email,
          password: this.password,
          firstname: 'John',
          lastname: 'Smith'
        })
        this.$store.dispatch('login', { user: response.user, token: response.token })
        this.$router.push('/search')
      } catch (error) {
        this.error = error.toString()
      }
    },
    async login () {
      try {
        const response = await UserService.login({ email: this.email, password: this.password })
        this.$store.dispatch('login', { user: response.user, token: response.token })
        this.$router.push('/search')
      } catch (error) {
        this.error = error.toString()
      }
    }
  }
}
```

4. Notez qu'en cas d'erreur, on stocke le message d'erreur retourné dans une variable `error`. Déclarez cette variable dans les `data` du composant et utilisez-là dans le template pour afficher le message d'erreur en cas d'échec d'authentification.

4. Notez également que la réponse du back-end au login contient un token permettant d'authentifier l'utilisateur, qui est transmis au store dans les paramètres de l'action `login`. Modifiez `store.js` pour stocker ce token dans le store, en ajoutant une mutation `setToken` appelée par l'action `login`.

5. Le service `api` est déjà configuré pour ajouter ce token en header `Authorization` des requêtes. Vérifiez que le token est bien envoyé en header HTTP via les outils développeur de votre navigateur.

6. **Bonus**: Modifier l'action `logout` du store pour supprimer le token et les infos utilisateur du store à la déconnexion, et rediriger vers le formulaire de login.

7. Créez un service `FilmService` avec une méthode pour rechercher les films, en suivant la documentation de l'API (`GET /movies/search`).

8. Modifier la page de recherche de films pour appeler le back-end et permettre à l'utilisateur d'entrer le nom d'un film, et d'avoir tout le détail de ce film en sortie.

::: tip

Si vous avez découvert qu'il existait un film appelé *Undefined*, c'est que vous vous êtes trompés quelque-part :)

![Undefined, the movie](../assets/undefined.jpg)
:::
