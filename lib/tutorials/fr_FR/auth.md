## Authentification

_Ce tutoriel est compatible avec Hapi v11.x.x._

L'authentification avec Hapi est basée sur le concept des `schémas` et
des `stratégies`.

Pensez à un schéma comme à un type général d'authentification, comme "basic" ou
"digest". Une stratégie elle, est une instance du schéma pré-configurée et
nommée.

D'abord regardons comment utiliser
[hapi-auth-basic](https://github.com/hapijs/hapi-auth-basic) :

```javascript
'use strict';

const Bcrypt = require('bcrypt');
const Hapi = require('hapi');
const Basic = require('hapi-auth-basic');

const server = new Hapi.Server();
server.connection({ port: 3000 });

const users = {
    john: {
        username: 'john',
        password: '$2a$10$iqJSHD.BGr0E2IxQwYgJmeP3NvhPrXAeLSaGCj6IR/XU5QtjVu5Tm',   // 'secret'
        name: 'John Doe',
        id: '2133d32a'
    }
};

const validate = function (request, username, password, callback) {
    const user = users[username];
    if (!user) {
        return callback(null, false);
    }

    Bcrypt.compare(password, user.password, (err, isValid) => {
        callback(err, isValid, { id: user.id, name: user.name });
    });
};

server.register(Basic, (err) => {

    if (err) {
        throw err;
    }

    server.auth.strategy('simple', 'basic', { validateFunc: validate });
    server.route({
        method: 'GET',
        path: '/',
        config: {
            auth: 'simple',
            handler: function (request, reply) {
                reply('hello, ' + request.auth.credentials.name);
            }
        }
    });

    server.start((err) => {

        if (err) {
            throw err;
        }

        console.log('server running at: ' + server.info.uri);
    });
});
```

D'abord, nous définissons notre base de données d'utilisateurs, qui est un
simple objet dans cet exemple. Puis nous définissions une fonction de
validation, qui est une fonctionnalité spécifique à
[hapi-auth-basic](https://github.com/hapijs/hapi-auth-basic) et qui nous permet
de vérifier que l'utilisateur a fourni les credentials valides.

Ensuite, nous enregistrons le plugin, qui crée un schéma avec le nom `basic`.
Ceci est réalisé dans le plugin via
[server.auth.scheme()](/api#serverauthschemename-scheme).

Une fois le plugin enregistré, nous utilisons
[server.auth.strategy()](/api#serverauthstrategyname-scheme-mode-options)
Pour créer une stratégie avec le nom `simple` qui réfère à notre schéma `basic`.
Nous passons aussi un objet d'options qui est passé au schéma et nous permet
de configurer son comportement.

La dernière chose que nous faisons est de déclarer une route qui utilise la
stratégie appelée `simple` pour l'authentification.

## Schémas

Un `schéma` est une méthode avec la signature `function (server, options)`. Le
paramètre `server` est une référence au serveur auquel le schéma est ajouté,
tandis que le paramètre `options` est l'objet de configuration fourni quand on
enregistre une stratégie qui utilise ce schéma.

Cette méthode doit retourner un objet avec *au moins* la clé `authenticate`.
D'autres méthodes facultatives qui peuvent être utilisées sont `payload` et
`response`.

### `authenticate`

La méthode `authenticate` a la signature `function (request, reply)`, et est la
seule méthode *obligatoire* dans un schéma.

Dans ce contexte, `request` est l'objet `request` créé par le server. C'est le
même objet qui devient disponible dans un handler de route, et est documenté dans
[la référence API](/api#request-object).

`reply` est l'interface Hapi standard `reply`, qui accepte les paramètres `err`
et `result` dans cet ordre.

Si `err` est une valeur non nulle, cela indique un échec dans l'authentification
et l'erreur sera utilisée comme réponse à l'utilisateur final. Il est conseillé
d'utiliser [boom](https://github.com/hapijs/boom) pour créer cette erreur et
pour fournir de façon simple les code et statut et message appropriés.

Le paramètre `result` doit être un objet, bien que l'objet lui même ainsi que
toutes ses clés sont optionnels si `err` est fourni.

Si vous souhaitez donner plus de détails dans le cas d'un échec, l'objet
`result` doit avoir une propriété `credentials` qui est un objet représentant
l'utilisateur authentifié (ou les credentials avec lesquels l'utilisateur a
tenté de s'authentifier) et doit être appeler ainsi `reply(error, null,
result);`.

Quand l'authentification a réussi, vous devez appeler `reply.continue(result)`
où result est un objet avec une propriété `credentials`.

De plus, vous pouvez avoir une clé `artifacts`, qui contient toutes les données
d'authentification qui ne font pas parti des credentials de l'utilisateurs.

Les propriétés `credentials` et `artifacts` peuvent être accéder plus tard (dans
un handler de route, par exemple) comme partie de l'objet `request.auth`.

### `payload`

La méthode `payload` a la signature `function (request, reply)`.

Encore une fois l'interface standard d'Hapi `reply` est disponible ici. Pour
lancer un échec, appeler `reply(error, result)` ou plus simplement
`reply(error)` (de nouveau, il est recommandé d'utiliser
[boom](https://github.com/hapijs/boom)) pour les erreurs.

Pour lancer une authentification réussie, appeler `reply.continue()` sans
paramètres.

### `response`

La méthode `response` a aussi la signature `function (request, reply)` et utilise
l'interface standard `reply`.

Cette méthode est destinée à décorer l'objet de réponse (`request.response`)
avec des entêtes supplémentaires, avant que la réponse ne soit envoyée à
l'utilisateur.

Une fois que toutes les décorations sont faites, vous devez appeler
`reply.continue()`, et la réponse sera envoyée.

Si une erreur survient, vous devriez plutôt appeler `reply(error)` où `error`
est recommandée d'être [boom](https://github.com/hapijs/boom).

### Enregistrement

Pour déclarer un schéma, utilisez `server.auth.scheme(name, scheme)`. Le
paramètre `name` est une chaîne de caractères utilisée pour identifier ce schéma
spécifique, le paramètre `scheme` est une méthode comme décrite plus haut.

## Stratégies

Une fois que vous avez déclaré votre schéma, vous avez besoin d'une façon de
l'utiliser. C'est là que les stratégies interviennent.

Comme mentionné au dessus, une stratégie est essentiellement une copie
pré-configurée d'un schéma.

Pour déclarer une stratégie, nous devons d'abord avec un schéma enregistré. Une
fois que c'est fait, utilisez `server.auth.strategy(name, scheme, [mode],
[options])` pour déclarer votre stratégie.

Le paramètre `name` doit être une chaîne de caractères, et sera utilisé plus
tard pour identifier cette stratégie spécifique. `scheme` est aussi une chaîne,
et c'est le nom du schéma dont la stratégie sera une instance.

### Mode

`mode` est le premierparamètre optionel, et peut être `true`, `false`,
`'required'`, `'optional'`, ou `'try'`.

Le mode par défaut est `false`, ce qui signifie que la stratégie sera déclarée
mais pas appliquée partout sauf si vous le faites manuellement.

Si défini à `true` ou `'required'`, qui sont les mêmes, la stratégie sera
automatiquement assignée à toutes les routes qui ne contiennent pas une config `auth`.
Ce réglage signifie que pour accéder à ces routes, l'utilisateur doit être
authentifié, et l'authentification doit être valide, sinon il recevra une
erreur.

Si le mode est défini à `'optional'` la stratégie sera toujours appliquée à
toutes les routes qui n'ont pas la config `auth`, mais dans ce cas l'utilisateur
n'a *pas* besoin d'être authentifié. Les données d'authentification sont
optionelles, mais doivent être valides si fournies.

Le dernier réglage de mode est `'try'` qui, encore, s'applique à toutes les
routes qui n'ont pas la config `auth`. La différence entre `'try'` et
`'optional'` est que `'try'` accepte l'authentification invalide, et que
l'utilisateur attendra toujours le handler de la route.

### Options

Le dernier paramètre est `options`, qui sera passé diréctement au schéma nommé.

### Définir une stratégie par défaut

Comme mentionné précédement, le paramètre `mode` peut être utilisé avec
`server.auth.strategy()` pour définir une stratégie par défaut. Vous pouvez
aussi définie une stratégie par défaut explicitement en utilisant
`server.auth.default()`.

Cette méthode accepte un paramètre, qui peut être une chaîne avec le nom de la
stratégie utilisée par défaut, ou un objet dans le même format que les
[auth options](#route-configuration) du handler de la route.

Notez que toutes les routes ajoutées *avant* que `server.auth.default()` ne soit
appelé n'auront pas le défaut appliquées. Si vous avec besoin d'être sûr que
toutes les routes ont la stratégie par défaut, vous devez soit appeler
`server.auth.default()` avant d'ajouter vos routes, soit définir le mode par
défaut quand vous déclarez la stratégie.

## Configuration des routes

L'authentification peut aussi être configurée sur une route, avec le paramètre
`config.auth`. Si défini à `false`, l'authentification est désactivée pour la
route.

Elle peut être aussi défini par une chaîne avec le nom de la stratégie à
utiliser, ou un objet avec les paramètres `mode`, `strategies`, et `payload`.

Le paramètre `mode` doit être défini à `'required'`, `'optional'`, ou `'try'` et
fonctionne de la même façon quand lors de la déclaraction d'une stratégie.

Quand vous spécifiez une stratégie, vous devez mettre la propriété `strategy` à
une chaîne avec le nom de la stratégie. Quand vous spécifiez plus d'une
stratégie, le paramètre name doit être `strategies` et doit être un tableau de
chaînes chacune étant un nom de stratégie à essayer. Les stratégies seront
tentées dans l'ordre jusqu'à ce qu'une réussisse, ou qu'elles échouent toutes.

Pour finir, le paramètre `payload` peut être défini à `false` pour dire que le
payload ne doit pas être authentifié, `'required'` ou `true` signifient qu'il
*doit* être authentifié, ou `'optional'` que le client inclue l'information
d'authentification du payload, l'authentification doit être valide.

le paramètre `payload` n'est possible qu'avec une stratégie qui support la
méthode `payload` dans son schéma.
