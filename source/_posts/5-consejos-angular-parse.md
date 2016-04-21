title: Consejos para utilizar Angular.js con Parse
date: 2015-03-07
author: Jose Enrique Bolaños
authorBio:  Desarrollador de Software. Cofundador y CTO de Slidebean
authorImg:  http://pbs.twimg.com/profile_images/378800000187243859/2a32f7d9e8b815a398d3412d46511d5c_400x400.jpeg
authorWebsite:  http://josebolanos.wordpress.com
authorTwitter: jozenwike
---

Actualmente estoy trabajando en [Slidebean](https://slidebean.com), una aplicación de la cual soy co-fundador. Slidebean permite crear presentaciones con excelente diseño en minutos. [Revísenlo aquí](https://www.slidebean.com), ¡es gratis!

Para Slidebean, decidí usar dos de los *frameworks* de aplicaciones más populares en este momento: [AngularJS](http://angularjs.org/) como *framework MVC* en javascript, y [Parse](http://parse.com/) como solución de *back-end* y almacenamiento en la nube. Si no han escuchado de ellos, los invito a que les echen un ojo, porque cada día se hacen más populares.

De entrada topé con algunos problemas de incompatibilidad al mezclar Parse con AngularJS. Las siguientes son algunas técnicas que descubrí sobre la marcha para combinarlos, que espero les ahorre tiempo si van a utilizarlos.

<!-- more -->

###1. Definir *getters* y *setters* para cada propiedad de sus clases de Parse

Parse permite obtener y almacenar propiedades en sus clases utilizando los métodos `get` y `set` de `Parse.Object`. En Slidebean, los usuarios crean presentaciones (`Presentation`), las cuales tienen una propiedad llamada `title` para el título. Digamos entonces que tenemos una variable `presentation` en el `$scope` de nuestro controlador, y queremos desplegar el título en una vista HTML. Haríamos algo como esto:

{% codeblock %}
  <div>{% raw %} {{ presentation.get("title") }} {% endraw %} </div>
{% endcodeblock %}

Sin embargo, AngularJS utiliza las propiedades simples de los objetos en javascript para leer y cambiar sus valores. Así que si quisiéramos cambiar el título de la presentación usando un campo `input`, la directiva `ng-model` no nos serviría:

{% codeblock %}
<input type="text" ng-model="presentation.set('title')">
<!-- Esto no tiene sentido, pero es para que se entienda el ejemplo: -->
{% endcodeblock %}

Una forma de solucionar este problema que sugiere [este artículo](https://github.com/jimrhoskins/angular-parse), es olvidarse de usar el *Javascript SDK* de Parse y usar su *REST API* directamente para trabajar con los modelos. Pero hay una solución mucho más simple, y es nada más definir `getters` y `setters` en cada una de sus clases de Parse. Así que usando AngularJS, usamos `factory` para definir una clase modelo llamada `Presentation`. El truco es que, adentro, también definimos un `getter` y un `setter` para cada propiedad. En este ejemplo, sólo tenemos una que se llama `title`:

{% codeblock %}
    angular.module('SlidebeanModels').
      factory('Presentation', function() {

        var Presentation = Parse.Object.extend("Presentation", {
          // Instance methods
        }, {
          // Class methods
        });

        // Title property
        Object.defineProperty(Presentation.prototype, "title", {
          get: function() {
            return this.get("title");
          },
          set: function(aValue) {
            this.set("title", aValue);
          }
        });

        return Presentation;
      });
{% endcodeblock %}

Ahora sí tenemos una clase modelo que funciona sin problemas con `ng-model`. Entonces, para actualizar el título de la presentación sólo tenemos que hacer esto:

{% codeblock %}   
<input type="text" ng-model="presentation.title">
{% endcodeblock %}

Mucho mejor :)

Esto funciona para propiedades con valores *simples* como *strings* o números. Pero, ¿qué pasa si tuviéramos una propiedad que contiene un puntero hacia otra clase de Parse? También se puede; nada más hay que asegurarse de incluir esa otra clase como dependencia de la primera para que así los *queries* funcionen sin problema.

Digamos que existe una clase `Comentario` y una clase `Noticia`, y que `Comentario` tiene una propiedad llamada `noticiaPadre`, la cual es un puntero hacia la `Noticia` a la que pertenece el `Comentario`. Si se agrega un `getter` en `Comentario` para `noticiaPadre` **sin** incluir la clase `Noticia` como una dependencia de `Comentario`, entonces ese `getter` siempre retornaría una instancia de `Parse.Object`. Pero si se incluye `Noticia` como dependencia, el `getter` retornará una instancia de `Noticia`.

Este truco suena más complicado de lo que es, jeje. Es similar a lo que se menciona en el punto #3  sobre la clase especial para `User`.

###2. Almacenar su usuario actual de Parse (`Parse.current()`) en el $rootScope

He leído que (casi) nunca se deben agregar variables al `$rootScope` de su aplicación de AngularJS. Parse provee un método muy conveniente para obtener el usuario que tiene una sesión actual, `Parse.User.current()`. El problema es que los cambios que suceden en esta variable de usuario suceden afuera del mundo de AngularJS, y entonces es difícil seguirles la pista y que sean *digeridos* por Angular. Me parece muy conveniente, entonces, almacenar el usuario actual de Parse en una variable en el `$rootScope` llamada `sessionUser` (o como quiera llamarla), y así también está disponible en el `$scope` de cualquier controlador de la aplicación. Esta variable es inicializada apenas comienza el app de AngularJS:

{% codeblock %}
    angular.module('SlidebeanApp')
      .config(function ($routeProvider, $locationProvider) {
        // Config goes here
      })
      .run(function($rootScope) {

        Parse.initialize("parse app", "parse credentials");

        $rootScope.sessionUser = Parse.User.current();

      });
{% endcodeblock %}

Para mantener las cosas un poco ordenadas, hice un *singleton* llamado `SessionService`, y la idea es que este sea el único lugar donde se manipula la variable `$rootScope.sessionUser`. Este servicio maneja inicios y finales de sesión (*log in & log out*), y actualiza la variable según corresponda.

Por ejemplo, un controlador de una barra de navegación que despliega el usuario actual puede reaccionar a esta variable, y modificar la forma en que se ve:

{% codeblock %}
    <!-- Mostrar botones de Ingresar and Registrarse cuando no hay una sesión -->
    <ul ng-show="sessionUser == null">
      <li><button type="button" ng-click="ingresar()">Ingresar</button></li>
      <li><button type="button" ng-click="registrarse()">Registrarse</button></li>
    </ul>

    <!-- Mostrar al usuario actual cuando haya una sesión -->
    <ul ng-show="sessionUser != null">
      <li class="dropdown">
        <a href="#" class="dropdown-toggle" data-toggle="dropdown">
          {{ sessionUser.name }}
          <b class="caret"></b>
        </a>
        <ul class="dropdown-menu">
          <li><a ng-href="/perfil">Perfil</a></li>
          <li><a ng-href="#" ng-click="salir()">Salir</a></li>
        </ul>
      </li>
    </ul>
{% endcodeblock %}

###3. Extender Parse.User e incluirlo desde el inicio

Me pareció que la documentación para extender la clase `Parse.User` no es muy clara. Pero básicamente, si uno quiere extenderla, es igual de fácil que extender cualquier otro `Parse.Object`. La clave es: **asegúrese de incluir su clase especial antes de llamar a `Parse.User.current()` por primera vez**, para que así reciba una instancia de su clase.

En Slidebean, quería tener un método especial llamado `getImage` para los usuarios, donde se abstrajera la complejidad de obtener la imagen del usuario ya sea de Facebook o de Gravatar. Así que extendí la clase `Parse.User` de esta manera:

{% codeblock %}
    angular.module('SlidebeanModels').
      factory('SlidebeanUser', function() {

        var User = Parse.User.extend({

          getImage : function() {
            // retornar la imagen de facebook o de gravatar
          }

        }, {
          // Métodos estáticos
        });

        return User;
      });
{% endcodeblock %}

Luego, nada más nos aseguramos de incluir nuestra clase `SlidebeanUser` como dependencia del método `run` de nuestro app:

{% codeblock %}
    .run(function($rootScope, $location, SlidebeanUser) {
      Parse.initialize("app id", "llave");

      // Ahora esto SI es una instancia de SlidebeanUser :)
      $rootScope.sessionUser = SlidebeanUser.current();

      // y esto sí funciona (si hay una sesión de usuario, obviamente):
      var imageUrl = $rootScope.sessionUser.getImage();
    })
{% endcodeblock %}

###4. Retrase la inicialización del SDK de Facebook hasta después de Parse.

Si permite que los usuarios inicien sesiones a su aplicación utilizando Facebook, entonces es buena idea cargar el SDK de Facebook hasta que se haya inicializado el SDK de Parse. Y también es buena idea inicializar el SDK de Parse hasta que el app de AngularJS haya arrancado. En general, entonces, es buena idea tener todo el código de inicialización en un solo lugar, y un buen lugar para hacerlo es en el método `run` de su app de Angular.

En Slidebean (y sospecho que en muchas aplicaciones), el orden de inicialización sigue así:

1. AngularJS
2. Parse
3. Facebook

y para lograrlo, nuestro método `run` luce así. Note cómo el código de inicialización del SDK de Facebook está aquí, en vez de estar en alguna parte del HTML donde normalmente se sugiere que se coloque:

{% codeblock %}
    .run(function($rootScope, $location, SlidebeanUser) {

      // 1) App de Angular ya está listo y corriendo.

      // 2) Inicializar Parse y poner el usuario actual en el $rootScope
      Parse.initialize("app id", "llave");

      $rootScope.sessionUser = SlidebeanUser.current();

      // 3) Finalmente, inicializar Facebook
      window.fbAsyncInit = function() {
        Parse.FacebookUtils.init({
          appId: 'facebook app id',
          channelUrl : '//www.slidebean.com/fbchannel.html',
          status: true,
          cookie: true,
          xfbml: true
        });
      };
      (function(d, s, id){
        var js, fjs = d.getElementsByTagName(s)[0];
        if (d.getElementById(id)) {return;}
        js = d.createElement(s); js.id = id;
        js.src = "//connect.facebook.net/en_US/all.js";
        fjs.parentNode.insertBefore(js, fjs);
      }(document, 'script', 'facebook-jssdk'));
    });
{% endcodeblock %}

Pero no olvide colocar esto en el HTML, ya que es algo que el SDK de Facebook requiere:

{% codeblock %}
    <div id="fb-root"></div>
{% endcodeblock %}

###5. Envolver los llamados asíncronos de Parse dentro de promesas $q

Se dará cuenta de que es buena idea envolver los llamados asíncronos de Parse dentro de promesas de AngularJS, en vez de nada más ejecutar los *queries* de Parse directamente en sus modelos y controladores. Uno de los beneficios es que los cambios en sus variables serán digeridos automáticamente por AngularJS.

Digamos que en Slidebean quisiéramos obtener todas las presentaciones que le pertenecen al usuario actual. Primero definimos una clase modelo llamada `Presentation` con un método estático para obtener las presentaciones según su dueño. Dentro de este método, envolvemos el *query* asíncrono usando una promesa `$q`:

{% codeblock %}
    angular.module('SlidebeanModels').
      factory('Presentation', function($q) {

        var Presentation = Parse.Object.extend("Presentation", {
          // Instance methods
        }, {
          // Class methods

          listByUser: function(aUser) {
            var defer = $q.defer();

            var query = new Parse.Query(this);
            query.equalTo("owner", aUser);
            query.find().then(function(aPresentations) {
              defer.resolve(aPresentations);
            }).fail(function(error) {
              defer.reject(error);
            });

            return defer.promise;
          }
        });

        // Properties
        Object.defineProperty(Presentation.prototype, "owner", {
          get: function() {
            return this.get("owner");
          },
          set: function(aValue) {
            this.set("owner", aValue);
          }
        });
        Object.defineProperty(Presentation.prototype, "title", {
          get: function() {
            return this.get("title");
          },
          set: function(aValue) {
            this.set("title", aValue);
          }
        });

     return Presentation;
     });
{% endcodeblock %}

Luego, dentro de cualquier controlador, si queremos obtener las presentaciones hacemos algo como esto:

{% codeblock %}
    angular.module('SlidebeanApp')
      .controller('DashboardCtrl', function($scope, Presentation) {

        Presentation.listByUser($scope.sessionUser).then(function(aPresentations) {
          $scope.presentations = aPresentations;
        }, function(aError) {
          // Something went wrong, handle the error
        });
      });
{% endcodeblock %}

Y listo. Ahora la lista de presentaciones se puede desplegar en HTML utilizando `ng-repeat`.

=====

Eso es todo por ahora. Si tienen más tips, preguntas o comentarios, no duden contactarme por Twitter [@jozenwike](http://twitter.com/jozenwike) ^____^

También pueden ver el original de esta noticia y otros artículos en mi [blog](https://josebolanos.wordpress.com/2013/09/30/5-tips-for-using-parse-with-angularjs/).
