## Méthodes intégrées

Comme dans tout logiciel de serveur, le logging est très important. Hapi a
quelques méthodes de trace intégrées, ainsi que quelques capacités limités pour
visualisées ces logs.

Il y a deux méthodes de logging similaires, `server.log`, and `request.log`. La
différence entre les deux est en fonction de l'événement que vous lancez, de
quel object lance cet événement, et de quelle données est automatiquement
associée. La méthode `server.log` émet un événement `log` sur le server, et a
l'URI du serveur associée avec elle. `request.log` émet un événement `request`
sur le `server` et a l'id interne de la requête associé avec elle.

Elle acceptent toutes les deux jusqu'à trois paramètres. Il sont, dans l'ordre,
`tags`, `data`, et `timestamp`.

`tags` est une chaine de caractère ou un tableau de chaines utilisés pour
identifier brièvement l'événement. Pensez à eux comme de niveaux
de log, mais bien plus expressifs. Par exemple, vous pouvez tagger une erreur
récupérant de la donnée depuis votre base de données comme suit :

```javascript
server.log(['error', 'database', 'read']);
```

Tous événements de log que hapi génère de façon interne auront toujours le tag
`hapi` associé.

Le second paramètre `data`, est une chaîne optionnelle ou un objet à tracer avec
l'événement. C'est ici que vous passerez des choses telles qu'un message
d'erreur, ou n'importe quel autres détails que vous souhaitez avoir avec vos
tags. L'événement de log aura automatiquement l'URI du server associé en tant
que propriété.

Le dernier est le paramètre `timestamp`. La valeur par défaut est `Date.now()`,
et ne devra être passé que si vous avez besoin de surcharge le défaut pour
quelque raison.

### Récupérer les logs de requêtes

Hapi fourni aussi une méthode `request.getLog` pour obtenir les événements de
log d'une requête, en partant du principe que vous avez accès à l'objet request.
Si appelée sans paramètres, elle va renvoyer un tableau de tous les événements
de log associés à la requête. Vous pouvez aussi passer un tag ou un tableau de
tags pour filtrer les résultats. Cela peut être utile pour récupérer un
historique de tous les événements tracés sur une requête quand une erreur
apparaît à des fins d'analyse.

### Configuration

Par défaut, les seuls erreurs que Hapi va afficher dans la console sont des
erreurs non capturés dans le code externe, et des erreurs de runtime dues à une
implémentation incorrect de l'API de Hapi. Vous pouvez toutefois configurer
votre serveur pour afficher les événements de requêtes basés sur un tag. Par
exemple, si vous souhaitez afficher toutes les erreurs d'une requête vous pouvez
configurer votre serveur comme suit :

```javascript
const server = new Hapi.Server({ debug: { request: ['error'] } });
```

## Plugins de log

Les méthodes intégrées sont assez minimales, toutefois, et pour des traces plus
approfondies vous devriez penser à utiliser un plugin comme
[good](https://github.com/hapijs/good) ou [bucker](https://github.com/nlf/bucker).
