# Chapter 6: Example d'application #

## Un code déclaratif ##

Nous allons adopter une nouvelle attitude. À partir de maintenant,
nous cesserons de dire à l'ordinateur ce qu'il doit faire, et
spécifierons à la place le résultat que nous aimerions avoir. Je suis
sûr que vous trouverez ça bien plus reposant que d'essayer de tout
micro-gérer tout le temps.

En déclaratif, contrairement à l'impératif, nous écrivons des
expressions, au lieu d'instructions étape par étape.

Pensez à SQL. Il n'y a pas de "Fait d'abord ceci, puis fait cela". Il
y a une expression qui spécifie ce que l'on attend de la base de
données. On ne décide pas la manière dont est fait le travail, _elle_
décide. Quand la base de données est mise-à-jour et que le moteur SQL
est optimisé, nous n'avons pas besoin de changer nos requêtes. Ceci
parce qu'il existe de multiple façon d'interpréter nos spécifications
et d'atteindre un même résultat.

Pour certaines personnes, moi inclus, il est difficile de saisir du
premier coup l'idée de la programmation déclarative, alors
familiarisons nous avec quelques exemples.

```js
// impératif
var makes = [];
for (i = 0; i < cars.length; i++) {
  makes.push(cars[i].make);
}


// déclaratif
var makes = cars.map(function(car){ return car.make; });
```

La boucle impérative doit premièrement instancier le
tableau. L'interpréteur doit évaluer cette instruction avant de
continuer. Il itère ensuite directement sur la liste des voitures,
incrémentant manuellement un compteur et faisant l'exposition vulgaire
et explicite de ses méchanismes d'itération.

La version `map` est une seule expression. Elle n'impose pas d'ordre
d'évaluation. Elle donne beaucoup de liberté à la fonction map pour
itérer et assembler le tableau à retourner. Elle spécifie le *quoi* et
non le *comment*. Elle arbore ainsi fièrement l'insigne du code déclaratif.

En plus d'être plus clair et concise, les entrailles de la fonction
map peuvent être optimisées à souhait sans que jamais notre précieux
code applicatif ne doive changer.

Pour ceux parmi vous qui pensent "Certes, mais la boucle impérative
est bien plus rapide", je suggère que vous vous renseigniez sur la
manière dont le JIT optimise votre code. Voici une
[vidéo géniale qui pourrait vous éclairer](https://www.youtube.com/watch?v=65-RbBwZQdU)
(en anglais).

Voici un autre exemple :

```js
// impératif
var authenticate = function(form) {
  var user = toUser(form);
  return logIn(user);
};

// déclaratif
var authenticate = compose(logIn, toUser);
```

Bien que la version impérative ne soit pas nécessairement mauvaise,
elle garde une évaluation étape par étape inscrite dans son
code. L'expression `compose` expose simplement un fait :
l'authentification est la composition de `toUser` et `logIn`. Encore
une fois, ça laisse la place à des ajustements du code utilitaire, et
résume notre code applicatif à une spécification de haut niveau.

Parce qu'il n'est écrit nulle part l'ordre d'évaluation, la
programmation déclarative se prête aux calculs parallèles. Ceci,
couplé aux fonctions pures, fait de la programmation fonctionnelle une
bonne option pour une parallélisation future. Créer des systèmes
concurrents/parallèles ne requiert pas d'action spéciale de notre
part.

## Flickr en programmation fonctionnelle ##

Nous allons maintenant construire un exemple d'application de manière
déclarative et composable. Nous tricherons encore un peu et
utiliserons des effets de bords, mais nous les réduirons à leur
minimum et les garderons séparé de notre code pur. Nous allons
construire un widget pour navigateur qui pompe des images flickr et
les affiche. Commençons par la structure de l'application. Voici le HTML :

```html
<!DOCTYPE html>
<html>
  <head>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/require.js/2.1.11/require.min.js"></script>
    <script src="flickr.js"></script>
  </head>
  <body></body>
</html>
```

Et voici la structure du fichier `flickr.js` :

```js
requirejs.config({
  paths: {
    ramda: 'https://cdnjs.cloudflare.com/ajax/libs/ramda/0.13.0/ramda.min',
    jquery: 'https://ajax.googleapis.com/ajax/libs/jquery/2.1.1/jquery.min'
  }
});

require([
    'ramda',
    'jquery'
  ],
  function (_, $) {
    var trace = _.curry(function(tag, x) {
      console.log(tag, x);
      return x;
    });
    // Code de l'application...
  });
```

On utilisera [ramda](http://ramdajs.com) au lieu de lodash une autre librairie utilitaire. Elle inclut `compose`, `curry`, et d'autres. J'ai utilisé `requirejs`, ce qui peut sembler exagéré, mais nous l'utiliserons tout au long de ce livre, et l'uniformité est cruciale. Aussi, j'ai introduit directement notre belle fonction `trace` pour un débogage facile.

Maintenant que ceci est bouclé, attaquons les spécifications. Notre application fera 4 choses :

1. Construire une URL pour notre terme de recherche
2. Faire l'appel à l'API flickr
3. Transformer le JSON de sortie en images HTML
4. Placer les images sur l'écran

Deux des actions mentionnées sont impures. Pouvez-vous les repérer ? Ces bouts concernant la récupération de donnée depuis flickr et le placement sur l'écran. Allons pour les définir en premier, qu'on puisse ensuite les isoler.

```js
var Impure = {
  getJSON: _.curry(function(callback, url) {
    $.getJSON(url, callback);
  }),

  setHtml: _.curry(function(sel, html) {
    $(sel).html(html);
  })
};
```

Nous avons simplement englobé les méthodes jQuery pour les currifier et nous avons interverti les arguments pour avoir un configuration plus favorable. Elles sont préfixées par `Impure` pour nous rappeler que ce sont des fonctions dangereuses. Dans un futur exemple, nous rendrons ces deux fonctions pures.

Ensuite, nous devons construire une URL à passer à notre fonction `Impure.getJSON`.

```js
var url = function (term) {
  return 'https://api.flickr.com/services/feeds/photos_public.gne?tags=' +
    term + '&format=json&jsoncallback=?';
};
```

Il y a des manières élégantes mais inutilement complexes d'écrire la fonction `url` TODO (pointfree) en utilisant les monoïdes TODO (nous en apprendrons plus à ce propos plus tard) ou les combinateurs. Nous avons opté pour une version lisible où nous assemblons la chaîne normalement TODO pointful.

Écrivons une fonction `app` qui effectue l'appel et positionne le contenu à l'écran.

```js
var app = _.compose(Impure.getJSON(trace("response")), url);

app("cats");
```

Cela appelle notre fonction `url`, puis passe la chaîne à notre fonction `getJSON`, qui a été partiellement appliquée avec `trace`. Au chargement de l'application, la réponse de l'API sera affichée dans la console.

<img src="images/console_ss.png"/>

Nous aimerions construire des images depuis ce JSON. On dirait que les sources `src` sont enfouies dans `items` puis dans la propriété `m` de chaque `media`

Dans tous les cas, pour obtenir ces propriétés imbriquées on peut utiliser une fonction _getter_ universelle de ramda appelée `_.prop()`. En voici une version maison, pour que vous voyiez de quoi il en retourne :

```js
var prop = _.curry(function(property, object){
  return object[property];
});
```

Ce n'est pas bien compliqué. Nous utilisons juste la syntaxe `[]` pour accéder à la propriété d'un objet quelconque. Mettons cette fonction à l'oeuvre pour obtenir les `src`.

```js
var mediaUrl = _.compose(_.prop('m'), _.prop('media'));

var srcs = _.compose(_.map(mediaUrl), _.prop('items'));
```

Une fois les `item` rassemblé, nous devons les _mapper_ pour en extraire les URL `media`. Le résultat est un joli tableau de `src`. Incluons ça dans notre application et affichons les à l'écran.

```js
var renderImages = _.compose(Impure.setHtml("body"), srcs);
var app = _.compose(Impure.getJSON(renderImages), url);
```

Nous n'avons fait que créer une nouvelle composition qui va appeler nos `srcs` et les faire occuper le corps du HTML. Maintenant que nous avons autre chose à afficher qu'un JSON brut, nous avons remplacé l'appel à `trace` par `renderImages`. Les sources des images seront affichées vulgairement dans le `body`.

L'étape finale est de transformer ces sources en jolies images. Dans une application plus grosse, nous utiliserions une librairie de template/DOM telle que Handlebars ou React. Cependant, ici, nous n'avons besoin que d'un tag `img` alors gardons jQuery.

```js
var img = function (url) {
  return $('<img />', { src: url });
};
```

La méthode `html()` de jQuery accepte un tableau de tags. Nous avons juste besoin de transformer nos srcs en images et de les envoyer à travers `setHtml`.

jQuery's `html()` method will accept an array of tags. We only have to transform our srcs into images and send them along to `setHtml`.

```js
var images = _.compose(_.map(img), srcs);
var renderImages = _.compose(Impure.setHtml("body"), images);
var app = _.compose(Impure.getJSON(renderImages), url);
```

And we're done!

<img src="images/cats_ss.png" />

Here is the finished script:
```js
requirejs.config({
  paths: {
    ramda: 'https://cdnjs.cloudflare.com/ajax/libs/ramda/0.13.0/ramda.min',
    jquery: 'https://ajax.googleapis.com/ajax/libs/jquery/2.1.1/jquery.min'
  }
});

require([
    'ramda',
    'jquery'
  ],
  function (_, $) {
    ////////////////////////////////////////////
    // Utils

    var Impure = {
      getJSON: _.curry(function(callback, url) {
        $.getJSON(url, callback);
      }),

      setHtml: _.curry(function(sel, html) {
        $(sel).html(html);
      })
    };

    var img = function (url) {
      return $('<img />', { src: url });
    };

    var trace = _.curry(function(tag, x) {
      console.log(tag, x);
      return x;
    });

    ////////////////////////////////////////////

    var url = function (t) {
      return 'http://api.flickr.com/services/feeds/photos_public.gne?tags=' +
        t + '&format=json&jsoncallback=?';
    };

    var mediaUrl = _.compose(_.prop('m'), _.prop('media'));

    var srcs = _.compose(_.map(mediaUrl), _.prop('items'));

    var images = _.compose(_.map(img), srcs);

    var renderImages = _.compose(Impure.setHtml("body"), images);

    var app = _.compose(Impure.getJSON(renderImages), url);

    app("cats");
  });
```

Now look at that. A beautifully declarative specification of what things are, not how they come to be. We now view each line as an equation with properties that hold. We can use these properties to reason about our application and refactor.

## Un refactor s'impose ##

There is an optimization available - we map over each item to turn it into a media url, then we map again over those srcs to turn them into img tags. There is a law regarding map and composition:


```js
// map's composition law
var law = compose(map(f), map(g)) == map(compose(f, g));
```

We can use this property to optimize our code. Let's have a principled refactor.

```js
// original code
var mediaUrl = _.compose(_.prop('m'), _.prop('media'));

var srcs = _.compose(_.map(mediaUrl), _.prop('items'));

var images = _.compose(_.map(img), srcs);

```

Let's line up our maps. We can inline the call to `srcs` in `images` thanks to equational reasoning and purity.

```js
var mediaUrl = _.compose(_.prop('m'), _.prop('media'));

var images = _.compose(_.map(img), _.map(mediaUrl), _.prop('items'));
```

Now that we've lined up our `map`'s we can apply the composition law.

```js
var mediaUrl = _.compose(_.prop('m'), _.prop('media'));

var images = _.compose(_.map(_.compose(img, mediaUrl)), _.prop('items'));
```

Now the bugger will only loop once while turning each item into an img. Let's just make it a little more readable by extracting the function out.

```js
var mediaUrl = _.compose(_.prop('m'), _.prop('media'));

var mediaToImg = _.compose(img, mediaUrl);

var images = _.compose(_.map(mediaToImg), _.prop('items'));
```

## En bref ##

We have seen how to put our new skills into use with a small, but real world app. We've used our mathematical framework to reason about and refactor our code. But what about error handling and code branching? How can we make the whole application pure instead of merely namespacing destructive functions? How can we make our app safer and more expressive? These are the questions we will tackle in part 2.

[Chapter 7: Hindley-Milner et Moi](ch7.md)