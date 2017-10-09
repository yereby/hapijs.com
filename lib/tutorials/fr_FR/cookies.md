## Cookies

_Ce tutoriel est compatible avec hapi v11.x.x._

Lorsqu'on écrit une application web, utiliser des cookies est souvent une
nécessité. En utilisant hapi, les cookies sont flexibles, sûrs, et simples.

## Configurer le serveur

hapi a plusieurs options de configuration possibles quand on parle des cookies.
Celles par défaut sont probablement suffisantes pour la plupart des cas, mais
peuvent être changées quand c'est nécessaire.

Pour les configurer, appeler `server.state(name, options)` où `name` est le nom
du cookie, et `options` est un objet utilisé pour configurer ce cookie
spécifique.

```javascript
server.state('data', {
    ttl: null,
    isSecure: true,
    isHttpOnly: true,
    encoding: 'base64json',
    clearInvalid: false, // remove invalid cookies
    strictHeader: true // don't allow violations of RFC 6265
});
```

Cette configuration fera que le cookie appelé `data` a une durée de vie de
session (qui sera supprimée quand le navigateur sera fermé), est défini comme
sûr et seulement sur le HTTP (voir [RFC
6265](http://tools.ietf.org/html/rfc6265), les sections
 [4.1.2.5](http://tools.ietf.org/html/rfc6265#section-4.1.2.5)
et [4.1.2.6](http://tools.ietf.org/html/rfc6265#section-4.1.2.6) pour plus de
détails sur ces flags), et dit à hapi que la valeur est une chaîne JSON encodé
en base 64. La documentation complète peut être trouvée sur
[the API reference](/api#serverstatename-options).

Ajoutons à ça, que vous pouvez passer deux paramètres à `config` lors de l'ajout
d'une route :

```json5
{
    config: {
        state: {
            parse: true, // parse and store in request.state
            failAction: 'error' // may also be 'ignore' or 'log'
        }
    }
}
```

## Définir un cookie

Définir un cookie est fait vair [l'interface `reply()`](/api#reply-interface)
dans un handler de requête, un prérequis, ou la durée de vie de la requête
ressemble à ceci :

```javascript
reply('Hello').state('data', { firstVisit: false });
```

Dans cet exemple, hapi va répondre avec la chaîne `Hello` et définir un cookie
appelé `data` comme valeur une chaîne encodé en base 64 la représentation de
l'objet fourni.

### Surcharger les options

Lorsque vous définissez un cookie, vous pouvez aussi passer les même options
disponibles que `server.state()` comme un troisième paramètre ainsi :

```javascript
reply('Hello').state('data', 'test', { encoding: 'none' });
```

## Nettoyer un cookie

Le cookie peut être néttoyé en appelant la méthode `unstate()` sur l'objet
[`response`](/api#response-object) :


```javascript
reply('Hello').unstate('data');
```
