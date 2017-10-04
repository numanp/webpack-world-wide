# Webpack World Wide

## Lectura Introductoria

### IIFEs

Antes de que vayamos mucho mas, es importante entender el concepto de expresiones de funciones invocadas inmediatamente en Javascript (aka "IIFE"s).

Primero, nota que cualquier "expresión" en Javascript puede ser escrita dentro de paréntesis para asegurarte que sea evaluada como un todo. Un caso de uso muy común de esto es especificar el orden de operaciones cuando hacemos matemática simple:   

```js
var twelve = 2 * 4 + 4     // 12
var sixteen = 2 * (4 + 4); // 16
```

Segundo, nota que cuando guardas una función en una variable, esa función es una expresión de Javascript también - una expresión de función. Esto es opuesto a una declaración de función, que es el termino de cuando escribimos funciones usando el keyword `function` sin el `var`.


```js
// una declaración de función
function funcDeclaration () {
  // etc
}

// una variable asignada a una expresión de función
var someVariable = function funcExpression () {
  // etc
}
```

Una vez que la expresión de función es guardado dentro de una variable, podemos por supuesto invocarla:

```js
var algunaVariable = function funcExpression () {
  // etc
}

algunaVariable();
```

Sin embargo, si envolvés una expresión de función dentro de un par de paréntesis, eso va a causar la función que sea evaluada de la misma forma que sería evaluada si estuviésemos guardándolo en una variable. Esto significa que podemos invocar la expresión de función que contiene!

```js
(function funcExpression () {
  // etc
})();

// comportamiento idéntico, pequeña diferencia estilística:
(function funcExpression () {
  // etc
}());
```

Esto es a lo que nos referimos cuando hablamos de IFFEs - nos dan una forma de invocar una expresión de función "inmediatamente" sin necesitar guardarla en una variable.

Por mucho tiempo, este patrón fue fuertemente confiado en proveer modularidad en Javascript, porque cualquier variable declarada dentro de un IFFE pertenecería solo al scope de esa función - no contaminarían el scope global.

Vamos a ver porque eso es importante en un momento - por ahora, si te sentís cómodo en tu entendimiento de como operan las IFFE operan, estas listo para continuar leyendo! 

### Script Tags y Browser Javascript

Muchos lenguajes construidos para correr en sistemas operativos (e.g. Java) tienen una forma nativa de definir modulos - esto es, archivos de código individual que pueden definir variables y/o funciones para exportar (o "exponer") a otros archivos. Los módulos pueden luego importar las variables y funciones que fueron exportadas por otros módulos. Cualquier variable o función que no ha sido explícitamente elegida para ser exportada es "privada" (o "local") de ese módulo. 

Este tipo de modularidad hace nuestro código fácil de leer, escribir, y mantener en el tiempo.

Sin embargo, Javascript no fue originalmente destinado a ser corrido directamente en sistema operativo - fue enfocado a correr en un browser. Tampoco nos previmos usando Javascript para escribir aplicaciones grandes y escalables como hacemos hoy en día. Entonces, Javascript no vino originalmente con un sistema de módulos.  

En cambio, todo el Javascript del browser se ejecuta en un mismo ambiente, donde el objeto window es el contexto global compartido de cualquier JavaScript corriendo en una sola página web. Cualquier variable que es declarada con la keyword `var` fuera de el cuerpo de una función (o declarada sin el keyword `var` en cualquier lado cuando no estamos en strict-mode) se convierte en pares key-value en el objeto `window`. Si deseas escribir Javascript en archivos separados, escribís el path a tu archivo en un `<script>` tag, y el browser pide el script de esa dirección y ejecuta ese script en el mismo, ambiente compartido como cada otro script.

Esto significa que si tenemos el siguiente archivo Javascript:

```js
// fileA.js

var foo = 42;

function bar () {
  console.log(foo);
}
```

y

```js
// fileB.js

var baz = 74;
```

Y luego requerimos ambas en un archivo html de esta forma:

```html
<script src="/fileA.js"></script>
<script src="/fileB.js"></script>
```

Esto significa que fileA.js va a correr primero, seguido por fileB.js. Luego de que ambos archivos corrieron, nuesto objeto `window` se verá así:

```js
window.foo; // 42
typeof window.bar; // "function"
window.baz; // 74
```

Puedes ver como esto puede rápidamente volverse problemático! Miremos a los dos grandes problemas que nos enfrentamos:



#### PROBLEMA UNO: COLISIÓN DE NOMBRES

Por ejemplo, que pasa si agregamos una `fileC.js`...

```html
<script src="/fileA.js"></script>
<script src="/fileB.js"></script>
<script src="/fileC.js"></script>
```

...el cual se ve como esto:

```js
// fileC.js

var foo = "uh oh!"; // We've already declared a variable called "foo" in fileA.js! But we're redefining it here!
bar(); // this is our bar function from fileA.js!
```

Que pasa si invocamos bar? cuando originalmente escribimos en `fileA.js`, esperábamos loggear el número 42, pero ahora `fileC.js` hemos declarado otra variable con el mismo nombre - hemos encontrado un colisión de nombres. Mientras quizás queríamos re-definir el valor de `foo`, lo más probable es que nos olvidamos el nombre de la variable que usamos antes! 

De cualquier forma, el gran problema numero uno es que los archivos JavaScript en el browser no corren aislados - las variables/funciones que ponen en el scope pueden ser sobrescritas por el código en otros archivos JavaScript. Esto hace al código que escribimos extremadamente frágil - introduciendo un nuevo archivo Javascript puede inadvertidamente romper otra pieza del script al sobrescribir una variable!

#### PROBLEMA DOS: DEPENDENCIA OPACA

Miremos a nuestros mismos tres archivos JasvaScript otra vez:

```html
<script src="/fileA.js"></script>
<script src="/fileB.js"></script>
<script src="/fileC.js"></script>
```

Estos scripts son cargados y ejecutados en orden. Esto significa que `fileA.js` corre hasta completarse antes que `fileB.js` comience, y `fileB.js` va a correr hasta completarse antes que `fileC.js` empiece.

¿Qué pasa si movemos `fileC.js` primera?

```html
<script src="/fileC.js"></script>
<script src="/fileA.js"></script>
<script src="/fileA.js"></script>
```

Recuerda que `fileC.js` usa la función `bar` que `fileA.js` define. Si tratamos de invocar `bar` antes que fue definida, eso nos va a traer un gran y horrible `ReferenceError`! Esto nos da el problema numero dos: el orden en el cual nuestros archivos JavaScript se ejecutan importa, y es normalmente difícil decir como debería ser ese orden. Esto puede llevar a situaciones engañosas donde se vuelve difícil resolver las "dependencias" que cada archivo JavaScript tiene en los otros que son cargados en la misma página.

El patrón IIFE ayuda a aminorar alguno de estos problemas (pero no los soluciona). Continuá leyendo y veremos como.

### Patrón de Módulos IFFE 

Cómo una forma de proteger el scope de las variables dentro de un archivo JavaScript, los desarrolladores empezaron a envolver sus archivos JavaScript en IFFEs. Porque las variables/funciones declaradas dentro de un IFFE tienen su scope limitado al scope de la función que lo contiene, somos capaces de en su mayoría (pero no completamente) mitigan la amenaza de la colisión de nombres. Considerá, una vez más, nuestro `fileA.js` y `fileC.js` (anda y mirá las secciones previas si necesitas un recuerdo de como se ven).

```html
<script src="/fileA.js"></script>
<script src="/fileC.js"></script>
```

Cambiemos `fileA.js` para que su contenido este envuelto en una IFFE. En cambio de poner todas de nuestras variables directamente en el scope global, vamos a restringirlo al scope local de la función - si hay algún valor que queremos exponer, vamos a poner esos valor al objeto global `window`.

```js
// fileA.js

(function () {
  // Queremos mantener la variable foo privada a fileA.js
  var foo = 42;

  function bar () {
    console.log(foo);
  }

  // Queremos usar la función `bar` en otro lugar, por lo que explícitamente lo anexamos al `window`
  window.bar = bar;
})();
```


Ahora cuando `fileC.js` corren:

```js
// fileC.js

var foo = "uh oh";

bar(); // Ahora esto va a ser 42!
```

Cuando invocamos `bar` ahora, la función `bar` que invocamos va a tener closure sobre la variable `foo` dentro del IFFE - no va a colisionar con la variable `foo` declarada en `fileC.js`!

Mientras que todavía hay chances que `window.bar` pueda ser sobrescrito por otro módulo, este patrón hizo las cosas mucho más fácil por un tiempo, podemos escribir ahora archivos JavaScript separados, envueltos en IFFEs, y podemos razonar sobre ellos en términos de sus "exports" (los valores que ellos anexan al objeto global `window`),  y sus "imports"/"dependencias" (los valores que necesitan que existan en el objeto global `window` antes de que ese archivo corrió). Otra variación de este concepto, [el patrón de revelación de módulos](https://addyosmani.com/resources/essentialjsdesignpatterns/book/#revealingmodulepatternjavascript), en el cual retornamos en la función eso que queremos exponer, permitió a los desarrolladores ser mucho mas explícitos sobre los "exports" de sus "módulos".

Sin embargo, con el amanecer de la edad de AJAX y JQuery, las aplicaciones web se fueron haciendo más grandes y más sofisticadas, y el "patrón IFFE" esta lejos de ser a prueba de tontos. Si queremos escribir aplicaciones de browser sofisticadas como las que existen en la computadora, vamos a necesitar herramientas mejores.

Aquí es donde las tecnologías como Webpack aparecen. Pero antes de eso, vamos a divagar un poco sobre como NodeJS cambió el juego al popularizar un cierto sistema de módulos para JavaScript del **lado del servidor**. 


### Entra Node

En este punto, ya estas familiarizado con el sistema de módulos que viene con NodeJs. Cada archivo JavaScript en un programa de Node es un módulo - las variables y funciones que declares en un módulo nunca van a colisionar con esas que tu declarás en otro módulo.

```js
// fileA.js

var foo = 42;

function bar () {
  console.log(foo);
}

// Usamos `module.exports`para exportar los valores del módulo
module.exports = bar;
```

```js
// fileB.js

// usamos `require`para importar valores de otros módulos
var someFunc = require('./fileB');

var foo = "no sweat"; // no tiene idea que usamos el mismo nombre para una variable en fileA
someFunc(); // 42
```

Libertad del browser permite a Node implementar un sistema de módulos, basado en uno propuesto por CommonJS, que ayudo a elevar a JavaScript a ser un "lenguaje de programación real" en los ojos de muchos. 

Sin embargo, solo JavaScript del lado del servidor puede usar `require` y `module.exports` - JavaScript escrito en browsers todavía era renegado a los riesgos y fragilidades de tags `<scripts>`. Sin embargo, ahora que hemos experimentado modularidad real con Node, nos debemos preguntar - ¿podemos implementar un sistema de módulos real para JavaScript desde browser también? ¿Hay una herramienta que podamos crear para que podamos esencialmente usar `require` y `module.exports` en nuestro Javascript para el browser también?


### JavaScript Moderno

La respuesta a esa pregunta en la sección anterior es si, por supuesto! Herramientas como Webpack (como también otras como browserfy, el cual no vamos a estar usando) nos permiten escribir archivos Javascript para el browser como si fueran módulos de Node - podemos usar `require` y `module.exports`, y declarar variables sin tener que preocuparse sobre contaminar el objeto global window. Es realmente un gran momento para ser un desarrollador web! 


