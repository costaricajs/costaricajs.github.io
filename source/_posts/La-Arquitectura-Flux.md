title: "La Arquitectura Flux"
date: 2015-03-11 16:24:12
categories:
- artículos
tags:
- flux
- arquitectura
- patrones
author: Gabo Esquivel
authorBio:  Desarrollador de Software y Organizador de CostaRicaJS.
authorImg:  https://pbs.twimg.com/profile_images/463588456405286912/8pzG9zlt.jpeg
authorWebsite:  http://gaboesquivel.com
authorTwitter: gaboesquivel
---

Flux es una arquitectura diseñada por Facebook junto con [React](http://facebook.github.io/react/), la librería para vistas. Se enfoca en crear __flujos de datos explícitos y entendibles__ , lo cual hace más sencillo seguir los cambios en el estado de la applicación y por ende los errores más fáciles de encontrar y corregir. 

<div class='centered-img'>
{% asset_img data-flow.png %}
</div>

Para comprender mejor la arquitectura Flux comparémola con MVC o Modelo-Vista-Controlador, uno de los patrones más utilizados en el desarrollo de aplicaciones. En MVC el controlador es responsable de coordinar los cambios en 1 o más modelos y lo hace mediante llamadas a métodos en los modelos. Cuando los modelos cambian, se notifican las vistas las cuales a su vez leen los nuevos datos del modelo y se actualizan de acuerdo a esos cambios para que el usuario pueda ver los nuevo datos o estado.
<!-- more -->
<div class='centered-img'>
{% asset_img mvc-simple.png %}
</div>

Conforme la aplicación crece y se agregan más controladores, modelos y vistas las dependencias se vuelven más complejas

<div class='centered-img'>
{% asset_img mvc-complejo.png %}
</div>

Cuando el usuario interactúa con la interfaz gráfica múltiples ramas de código son ejecutadas y encontrar errores y correr pruebas sobre el estado de la apliación se vuelve una tarea muy difícil en la cual básicamente se tiene que tratar de adivinar en cual de todos los puntos se encuentra el error. En los peores casos una interacción causa actualizaciones adicionales en cascada haciendo la tarea aún mas dura.

Flux evita este diseño en favor de un flujo de datos en una sola direción. Todas las interaciones dentro de un vista llaman un "creador de acciones" que a su vez llama el "despachador" de tipo singleton que es responsable de emitir un evento de tipo "acción" al cual los "almacenes" se pueden subscribir. Estos "almacenes" responden a la acción y se autoactualizan.

<div class='centered-img'>
{% asset_img flux-simple.png %}
</div>

El flujo no cambia mucho cuando se agregan almacenes y vistas adicionales. El despachador simplemente envía cada acción a todos los almacenes en la aplicación y no conoce los detalles de como estos almacenes se actualizan, cada almacén contiene su propia lógica de negocios. Cada almacén es reponsable de área o dominio de la apliación y se solo se autoactualizan en respuesta a acciones.

<div class='centered-img'>
{% asset_img flux-complejo.png %}
</div>

Cuando los almacenes se actualizan emiten un evento "cambio" al cual los vistas reaccionan y actualizan la interfaz gráfica del usuario. En muchas aplicaciones basadas en React es común tener "contenedores" responsables for observar este evento, leer los nuevos datos en el almacén y pasar la información a través de propiedades a las vistas dentro del contenedor. Los contenedores encapsulan un componente de la interfaz gráfica, por ejemplo un caja de búsqueda con completación automática que se compone de varias vistas y componentes.

## Caraterística Principales

La arquitectura flux tiene propiedades que la hacen única y provee importantes garantías, todas giran alrededor de un flujo de datos explícito y fácil de entender, aumentando la capacidad de seguir, reproducir y realizar pruebas en estados de aplicación específicos.

### Sincronía

El despachador de acciones y las funciones dentro de los almacenes son síncronos. Todas las operaciones asincrónicas deben invocar una acción le comunica al sistema el resultado de la operación. Los creadores de acciones pueden llamar APIs asyncrónicamente, los almacenes idealmente no lo deben hacer. Esta regla hace que el flujo de información sea extremadamente explícito y en caso de errores facilmente se puede identificar la acción ese estado erróneo de la applicación.

### Inversión del Control

Los almacenes se autoactualizan en respuesta a acciones en lugar de ser acutalizados por un controllador o modulo similar, ningún otro componente de la aplicación contiene lógica sobre como actualizar el estado. Como las almacenes se autoactualizan y únicame sincrónicamente, realizar pruebas es tan sencillo como inicializar con un estado específico, invocar un acción y verificar que el estado final es el esperado. 

### Acciones Semánticas

Las acciones tienden a ser semanticamente descriptivas. Por ejemplo, en un chat basado en flux para marcar un conversación como leída probablemente se invocaría una acción de tipo "marcar_conversacion_como_leida". La acción y el componente que genera la acción no saben como hacer la actualización, pero describe lo que quiere que suceda.

Por esta propiedad raramente se tendrá que cambiar el tipo de las acciones, unicamente como los almacenes responden a ellas.

### Cero Acciones en Cascada

Flux no permite despachar una segunda acción como resultado de una primera acción, esto ayuda a prevenir actualizaciones en cascada que son difíciles de mantener y debuggear. Además ayuda a pensar en las interaciones de la applicación en una forma más semántica.

<div class="refs">
__Lectura Adicional en Inglés__
[Flux Application Architecture](http://facebook.github.io/flux/docs/overview.html)
[Texto Original en Inglés en fluxxor.com](http://fluxxor.com/what-is-flux.html)
[Video: Rethinking Web App Development at Facebook](https://www.youtube.com/watch?v=nYkdrAPrdcw)
</div>