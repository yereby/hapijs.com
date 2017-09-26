## Caching côté client

_Ce tutoriel est compatible avec hapi v11.x.x._

Le protocole HTTP fourni plusieurs entêtes différents pour controler comment les
navigateurs mettent en cache les ressources. Pour décider quels entêtes sont
adaptés pour votre cas, regardez le
[guide](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/http-caching) or other [resources](https://www.google.com/search?q=HTTP+caching)
de Google developers. Ce tutoriel est une vue d'ensemble sur comment utiliser
ces techniques avec Hapi.

### Cache-Control

L'entête `Cache-control` dit au navigateur et tout cache intermédiaire  si
une ressource peut être cachée et pour quelle durée. Par exemple,
`Cache-Control:max-age=30, must-revalidate, private` signifie que le navigateur
peut cacher la ressource associée pendant trente secondes et `private` qu'elle
ne doit pas être cachée par les caches intermédiares, seulement par le
navigateur. `must-revalidate` signifie qu'une fois expiré, il doit requéter la
ressource à nouveau depuis le serveur.

Voyons comment ça peut être fait depuis Hapi :


```javascript
server.route({
    path: '/hapi/{ttl?}',
    method: 'GET',
    handler: function (request, reply) {

        const response = reply({ be: 'hapi' });
        if (request.params.ttl) {
            response.ttl(request.params.ttl);
        }
    },
    config: {
        cache: {
            expiresIn: 30 * 1000,
            privacy: 'private'
        }
    }
});
```
**Listing 1** Setting Cache-Control header

Le listing 1 illustre aussi que la valeur `expiresIn` peut être surchargée par
la méthode `ttl(msec)` fournie par l'interface [objet réponse](http://hapijs.com/api#response-object)

Voir [route-options](http://hapijs.com/api#route-options) pour plus
d'informations à propos des options communes de configuration du `cache`.

### Last-Modified

Dans certains cas le serveur fourni des l'information de quand les ressources
ont été modifiés pour la dernière fois. Quand on utilise le plugin
[inert](https://github.com/hapijs/inert) pour servir du contenu statique, un
entête `Last-Modified` est ajouté à chaque paylod de réponse.

Quand l'entête `Last-Modified` est indiqué sur une réponse, Hapi le compare avec
l'entête `If-Modified-Since` envoyé par le client pour décider si le status code
de la réponse doit être `304`.

Avec `lastModified` étant un objet Date vous pouvez définir cet entête via
l'interface [response object](http://hapijs.com/api#response-object) comme
indiqué ci-dessous dans le listing 2.

```javascript
   reply(result).header('Last-Modified', lastModified.toUTCString());
```
**Listing 2** Setting Last-Modified header

Il y a un exemple suplémentaire utilisant `Last-Modified` dans la
[dernière partie](###########) de ce tutoriel.

### ETag

L'entête ETag est une alternative à `Last-Modified` où le serveur fourni un
token (habituellement le checksum de la ressource) plutôt qu'un timestamp de la
dernière modification. Le navigateur pourra utiliser ce token pour définir
l'entête `If-None-Match` dans la prochaine requête. Le serveur compare cet
entête avec le nouveau checksum `ETag` et répond un `304` s'il sont identiques.

Vous devez juste définir `ETag` dans votre handler via la fonction `etag(tag, options)` :

```javascript
   reply(result).etag('xxxxxxxxx');
```
**Listing 3** Setting ETag header

Vérifiez la documentation d'`etag` sur [response object](http://hapijs.com/api#response-object)
pour plus de détails sur les paramètre et options disponibles.

## Cache côté serveur

Hapi fourni du cache côté serveur puissant via [catbox](https://www.github.com/hapijs/catbox)
et facilite l'utilisation du cache. Cette section du tutoriel va vous aider à
comprendre comment catbox est utilisé dans Hapi et comment vous pouvez en
profiter.

Catbox a deux interfaces ; client et politique.

### Client

[Client](https://github.com/hapijs/catbox#client) est une interface de bas
niveau qui défini/récupère des paires de clé-valeur. Il est initialisé avec un
des adapteurs disponibles : 
([Memory](https://github.com/hapijs/catbox-memory),
[Redis](https://github.com/hapijs/catbox-redis),
[mongoDB](https://github.com/hapijs/catbox-mongodb),
[Memcached](https://github.com/hapijs/catbox-memcached),
ou [Riak](https://github.com/DanielBarnes/catbox-riak)).

Hapi initialise toujours un [client](https://github.com/hapijs/catbox#client)
par défaut avec l'adapteur [memory](https://github.com/hapijs/catbox-memory).
Voyons comment nous pouvons en définir plus.

```javascript
'use strict';

const Hapi = require('hapi');

const server = new Hapi.Server({
    cache: [
        {
            name: 'mongoCache',
            engine: require('catbox-mongodb'),
            host: '127.0.0.1',
            partition: 'cache'
        },
        {
            name: 'redisCache',
            engine: require('catbox-redis'),
            host: '127.0.0.1',
            partition: 'cache'
        }
    ]
});

server.connection({
    port: 8000
});
```
**Listing 4** Defining catbox clients

Dans le listing 4, nous avons défini deux clients catbox ; mongoCache et
redisCache. En incluant memory cache créé par hapi, il y a trois caches
disponibles. Vous pouvez remplacer le client par défaut en omettant la propriété
`name` à la déclaration d'un nouveau client de cache. `partition` indique à
l'adapteur comment le cache doit être appelé (`catbox` par défaut). Dans le cas
de [mongoDB](http://www.mongodb.org/) c'est un nom de base de données et dans le
cas de [redis](http://redis.io/) il est utilisé comme un préfixe de clé.

### Policy

[Policy](https://github.com/hapijs/catbox#policy) est une interface plus haut
niveau que Client. Imaginons que nous traitons avec quelque chose plus compliqué
qu'ajouter deux nombres et que nous voulons cacher le résultat.
[server.cache](http://hapijs.com/api#servercacheoptions) crée un nouveau
[policy](https://github.com/hapijs/catbox#policy), qui est utilisé dans le
handler de route.

```javascript
const add = function (a, b, next) {

    return next(null, Number(a) + Number(b));
};

const sumCache = server.cache({
    cache: 'mongoCache',
    expiresIn: 20 * 1000,
    segment: 'customSegment',
    generateFunc: function (id, next) {

        add(id.a, id.b, next);
    },
    generateTimeout: 100
});

server.route({
    path: '/add/{a}/{b}',
    method: 'GET',
    handler: function (request, reply) {

        const id = request.params.a + ':' + request.params.b;
        sumCache.get({ id: id, a: request.params.a, b: request.params.b }, (err, result) => {

            if (err) {
                return reply(err);
            }
            reply(result);
        });
    }
});
```
**Listing 5** Using `server.cache`

`cache` indique à hapi quel [client](https://github.com/hapijs/catbox#client)
utiliser.

Le premier paramètre de la méthode `sumCache.get` est un objet avec la clé
obligatoire `id`, qui est un chaine unique d'identifieur.

En plus de **partitions**, il y a **segments** qui vous permettent d'isoler les
caches dans un [client](https://github.com/hapijs/catbox#client). Si vous voulez
cacher les résultats de deux méthodes différentes, habituellement vous ne
souhaitez pas mixer les résultats ensemble. Dans les adapteurs
[mongoDB](http://www.mongodb.org/), `segment` représente une collection et dans
[redis](http://redis.io/) c'est un préfix additionel selon l'option `partition`.

La valeur par défaut de `segment` quand
[server.cache](http://hapijs.com/api#servercacheoptions) est appelée dans un
plugin sera `'!pluginName'`. Quand on crée des
[server methods](http://hapijs.com/tutorials/server-methods), la valeur de
`segment` sera `'#methodName'`. Si vous avez un cas d'utilisation pour de
multiples policies qui partagent un segment il y a aussi des options
[shared](http://hapijs.com/api#servercacheoptions) disponibles.

### Méthodes serveur

Mais cela peut être encore mieux que ça ! Dans 95% des cas vous utiliserez des
[server methods](http://hapijs.com/tutorials/server-methods) dans le but de
cacher, parce que ça réduit le boilerplate au minimum. Regardons le Listing 6
qui utilise des méthodes serveur :

```javascript
const add = function (a, b, next) {

    return next(null, Number(a) + Number(b));
};

server.method('sum', add, {
    cache: {
        cache: 'mongoCache',
        expiresIn: 30 * 1000,
        generateTimeout: 100
    }
});

server.route({
    path: '/add/{a}/{b}',
    method: 'GET',
    handler: function (request, reply) {

        server.methods.sum(request.params.a, request.params.b, (err, result) => {

            if (err) {
                return reply(err);
            }
            reply(result);
        });
    }
});
```
**Listing 6** Using cache via server methods

[server.method()](http://hapijs.com/api#servermethodname-method-options) crée un
nouveau [policy](https://github.com/hapijs/catbox#policy) avec le `segment:
'#sum'` automatiquement pour nous. l'`id` a aussi été généré automatiqment
depuis les paramètres. Par défaut, il comprend les paramètres `string`, `number` et
`boolean`. Pour des paramètres plus complexes, vous devez fournir votre propre
fonction `generateKey` pour créer des ids uniques basés sur les paramètres —
regardez lu tutoriel [server methods](http://hapijs.com/tutorials/server-methods) pour
plus d'informations

### En ensuite ?

* Regardez les options de politique de [catbox](https://github.com/hapijs/catbox#policy)
et faites très attention à `staleIn`, `staleTimeout`, `generateTimeout`, qui
augmentent le potentiel de cache de catbox.
* Regardez le [tutoriel server methods](http://hapijs.com/tutorials/server-methods)
pour plus d'exemples.

## Cache client et serveur

[Catbox Policy](https://github.com/hapijs/catbox#policy) fourni deux paramètres
supplémentaires `cached` et `report` qui fournissent des détails supplémentaires
quand un objet est caché.

Dans ce cas nous utilisons le timestamp `cached.stored` pour définir l'entête
`last-modified`.

```javascript
server.route({
    path: '/add/{a}/{b}',
    method: 'GET',
    handler: function (request, reply) {

        server.methods.sum(request.params.a, request.params.b, (err, result, cached, report) => {

            if (err) {
                return reply(err);
            }
            const lastModified = cached ? new Date(cached.stored) : new Date();
            return reply(result).header('last-modified', lastModified.toUTCString());
        });
    }
});
```
**Listing 7** Server and client side caching used together
