## Installer hapi

_Ce tutoriel est compatible avec hapi v11.x.x._

Créer un nouveau répertoire `myproject`, depuis lequel :

* Exécuter : `npm init` et suivre le prompt , cela va générer un ficher package.json
pour vous.
* Exécuter : `npm install --save hapi` ce qui installe hapi, et l'enregistre
dans votre package.json comme dépendance de votre projet.

C'est tout ! Vous avez maintenant tout ce dont vous avez besoin pour créer
un serveur basé sur hapi.

## Créer un serveur

Le serveur le plus simple ressemble à ce qui suit :

```javascript
'use strict';

const Hapi = require('hapi');

const server = new Hapi.Server();
server.connection({ port: 3000, host: 'localhost' });

server.start((err) => {

    if (err) {
        throw err;
    }
    console.log(`Server running at: ${server.info.uri}`);
});
```

D'abord on require hapi. Puis nous créons un nouvel object serveur hapi.
Ensuite nous ajoutons une connexion au serveur, en passant un numéro de port
sur lequel écouter.
Après ça, nous lançons le serveur et loggons qu'il est en cours.

Quand nous ajoutons la connexion au serveur, nous pouvons aussi indiquer
un nom d'hôte, une adresse IP, ou même un fichier de socket Unix, ou Windows
nommé lié au server. Pour plus de détails, voir [la référence
d'API](/api/#serverconnectionoptions).

## Ajout des routes

Maintenant que nous avons un serveur nous pouvons ajouter une ou deux routes
pour que nous puissions faire quelque chose. Regardons à quoi ça ressemble :

```javascript
'use strict';

const Hapi = require('hapi');

const server = new Hapi.Server();
server.connection({ port: 3000, host: 'localhost' });

server.route({
    method: 'GET',
    path: '/',
    handler: function (request, reply) {
        reply('Hello, world!');
    }
});

server.route({
    method: 'GET',
    path: '/{name}',
    handler: function (request, reply) {
        reply('Hello, ' + encodeURIComponent(request.params.name) + '!');
    }
});

server.start((err) => {

    if (err) {
        throw err;
    }
    console.log(`Server running at: ${server.info.uri}`);
});
```

Sauver le code ci-dessus sous `server.js` et lancer le serveur avec la commande
`node server.js`. Maintenant vous verrez qu'en vous rendant sur
http://localhost:3000 dans votre navigateur, vous
verrez le text `Hello, world!`, et si vous visitez
http://localhost:3000/stimpy vous verrez `Hello, stimpy!`.

Notez que nous URI encodons le paramètre name, c'est pour prévenir des attaques
par injection de contenu. Souvenez-vous, ce n'est jamais une bonne idée
d'afficher des données fournies par l'utilisateur sans encoder la sortie
d'abord.

Le paramètre `method` peut être n'importe quelle méthode HTTP valide, tableau de
méthodes HTTP, ou un astérisque pour permettre n'importe quelle méthode. Le
paramètre `path` définit le chemin incluant les paramètres. Il peut contenir des
paramètres optionnels, des paramètres numérotés, et même des wildcards. Pour plus
de détails, voir [le tutoriel sur les routes](/tutorials/routing).

## Création des pages et du contenu statiques

Nous avons montré que nous pouvons lancer une application Hapi simple avec
notre application Hello World. Ensuite nous allons utiliser un plugin appelé
**inter** pour servir une page statique. Avant de commencer, arrêtez le serveur
avec **CTRL + C**.

Pour installer inert lancer cette commande sur la ligne de commande : `npm
install --save inert`. Cela va télécharger inert et l'ajouter à votre
`package.json`, qui documente quels paquets sont installés.

Ajouter ceci au fichier `server.js` :

``` javascript
server.register(require('inert'), (err) => {

    if (err) {
        throw err;
    }

    server.route({
        method: 'GET',
        path: '/hello',
        handler: function (request, reply) {
            reply.file('./public/hello.html');
        }
    });
});
```

La commande `server.register()` au dessus ajoute le plugin inert à votre
application Hapi. Si quelque chose se passe mal, nous voulons le savoir, donc
nous avons passé une fonction anonyme qui si invoquée va recevoir `err` et
lancer cette erreur. Cette fonction de callback est requise lorsqu'on enregistre
les plugins.

La commande `server.route()` déclare la route `/hello`, qui dit au server
d'accepter les requètes GET vers `/hello` et de répondre avec le contenu du
fichier `hello.html`. Nous avons mis la function de callback de routing dans la
déclaration d'inert parce que nous devons nous assurer qu'inert est déclaré
_avant_ que nous l'utilisions pour rendre la page static. Il est en général sage
d'éxécuter le code qui dépend d'un plugin dans la callback qui enregistre le
plugin pour être absolument sûr que le plugin existe quand le code se joue.

Démarrez le serveur avec `npm start` et allez sur `http://localhost:3000/hello`
dans votre navigateur. Oh non ! Nous obtenons une erreur 404 parce que nous
n'avons pas créé un fichier `hello.html`. Vous devez créer le fichier manquant
pour éviter cette erreur.

Créez un dossier appelé `public` à la racine de votre répertoire avec un fichier
appelé `hello.html` à l'intérieur. Dans `hello.html` mettez les HTML suivant :
`<h2>Hello World.</h2>`. Puis relancez la page dans le navigateur. Vous devriez
voir une entête disans "Hello World.".

Inert va servir tout contenu qui est sauvegardé sur votre disque dur quand la
requète est faite, ce qui amène à ce comportement de live reloading. Modifiez la
page à `/hello` à votre convenance.

Plus de détails sur comment le contenu statique est servi est détaillé sur
[Servir du contenu statique](/tutorials/serving-files). Cette technique est
habituellement utilisée pour servir des images, feuilles de style, et des pages
statiques dans votre application web.

## Utiliser les plugins

Une envie commune quand on crée une application web, est d'avoir des log
d'acces. Pour ajouter des logs basiques à notre application, nous allons charger
le plugin [good](https://github.com/hapijs/good) et son reporter
[good-console](https://github.com/hapijs/good-console) sur notre serveur. Nous
allons aussi avoir besoin d'un mécanisme basique de filtrage. Utilisons
[good-squeeze](https://github.com/hapijs/good-squeeze) parce qu'il contient le
filtrage basique des types d'event et des tags dont nous avons besoin pour commencer.

Installez les modules depuis npm pour démarrer :

```bash
npm install --save good
npm install --save good-console
npm install --save good-squeeze
```

Puis mettez à jour votre `server.js` :

```javascript
'use strict';

const Hapi = require('hapi');
const Good = require('good');

const server = new Hapi.Server();
server.connection({ port: 3000, host: 'localhost' });

server.route({
    method: 'GET',
    path: '/',
    handler: function (request, reply) {
        reply('Hello, world!');
    }
});

server.route({
    method: 'GET',
    path: '/{name}',
    handler: function (request, reply) {
        reply('Hello, ' + encodeURIComponent(request.params.name) + '!');
    }
});

server.register({
    register: Good,
    options: {
        reporters: {
            console: [{
                module: 'good-squeeze',
                name: 'Squeeze',
                args: [{
                    response: '*',
                    log: '*'
                }]
            }, {
                module: 'good-console'
            }, 'stdout']
        }
    }
}, (err) => {

    if (err) {
        throw err; // something bad happened loading the plugin
    }

    server.start((err) => {

        if (err) {
            throw err;
        }
        server.log('info', 'Server running at: ' + server.info.uri);
    });
});
```

Maintenant que le serveur est lancé vous pouvez voir :

```
140625/143008.751, [log,info], data: Server running at: http://localhost:3000
```

Et si vous visitez `http://localhost:3000/` dans le navigateur, vous verrez :

```
140625/143205.774, [response], http://localhost:3000: get / {} 200 (10ms)
```

Super ! Il s'agit juste d'un petit exemple de ce que les plugins sont capables,
pour plus d'information allez sur le [tutoriel des plugins](/tutorials/plugins).

## Tout le reste

Hapi a beaucoup, beaucoup d'autres possibilités et seulement quelques unes sont
documentées dans ces tutoriels. Veuillez utiliser la liste sur votre droite pour
les voir. Tout le reste est docummenté dans la [référence API](/api) et, comme
toujours, n'hésitez pas à poser des questions ou simplement nous visiter sur
freenode à [#hapi](http://webchat.freenode.net/?channels=hapi).
