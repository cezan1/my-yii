Conteneur d'injection de dépendances
====================================

Un conteneur d'injection de dépendances (DI container) est un objet qui sait comment instancier et configurer des objets et tous leurs objets dépendants. [Cet article de Martin Fowler](https://martinfowler.com/articles/injection.html) explique très bien en quoi un conteneur d'injection de dépendances est utile. Ici nous expliquons essentiellement l'utilisation qui est faite du conteneur d'injection de dépendances que fournit Yii.


Injection de dépendances <span id="dependency-injection"></span>
------------------------

Yii fournit la fonctionnalité conteneur d'injection de dépendances via la classe [[yii\di\Container]]. Elle prend en charge les sortes d'injections de dépendance suivantes :

* Injection par le constructeur ;
* Injection par les méthodes ;
* Injection par les méthodes d'assignation et les propriétés ;
* Injection par une méthode de rappel PHP ;


### Injection par le constructeur <span id="constructor-injection"></span>

Le conteneur d'injection de dépendances prend en charge l'injection dans le constructeur grâce à l'allusion au type pour les paramètres du constructeur. L'allusion au type indique au conteneur de quelles classes ou de quelles interfaces dépend l'objet concerné par la construction. Le conteneur essaye de trouver les instances des classes dont l'objet dépend pour les injecter dans le nouvel objet via le constructeur. Par exemple :

```php
class Foo
{
    public function __construct(Bar $bar)
    {
    }
}

$foo = $container->get('Foo');
// qui est équivalent à ce qui suit
$bar = new Bar;
$foo = new Foo($bar);
```


### Injection par les méthodes <span id="method-injection"></span>

Ordinairement, les classes dont une classe dépend sont passées à son constructeur et sont disponibles dans la classe durant tout son cycle de vie. Avec l'injection par les méthodes, il est possible de fournir une classe qui est seulement nécessaire à une unique méthode de la classe, et qu'il est impossible de passer au constructeur, ou qui pourrait entraîner trop de surplus de travail dans la majorité des classes qui l'utilisent. 

Une méthode de classe peut être définie comme la méthode `doSomething()` de l'exemple suivant :

```php
class MyClass extends \yii\base\Component
{
    public function __construct(/*ici, quelques classes légères dont la classe dépend*/, $config = [])
    {
        // ...
    }

    public function doSomething($param1, \ma\dependance\Lourde $something)
    {
        // faire quelque chose avec $something
    }
}
```

Vous pouvez appeler la méthode, soit en passant une instance de `\ma\dependance\Lourde` vous-même, soit en utilisant [[yii\di\Container::invoke()]] comme ceci :

```php
$obj = new MyClass(/*...*/);
Yii::$container->invoke([$obj, 'doSomething'], ['param1' => 42]); // $something est fournie par le conteneur d'injection de dépendances
```

### Injection par les méthodes d'assignation et les propriétés <span id="setter-and-property-injection"></span>

L'injection par les méthodes d'assignation et les propriétés est prise en charge via les [configurations](concept-configurations.md). Lors de l'enregistrement d'une dépendance ou lors de la création d'un nouvel objet, vous pouvez fournir une configuration qui est utilisée par le conteneur pour injecter les dépendances via les méthodes d'assignation ou les propriétés correspondantes. Par exemple :

```php
use yii\base\BaseObject;

class Foo extends BaseObject
{
    public $bar;

    private $_qux;

    public function getQux()
    {
        return $this->_qux;
    }

    public function setQux(Qux $qux)
    {
        $this->_qux = $qux;
    }
}

$container->get('Foo', [], [
    'bar' => $container->get('Bar'),
    'qux' => $container->get('Qux'),
]);
```

> Info: la méthode [[yii\di\Container::get()]] accepte un tableau de configurations qui peut être appliqué à l'objet en création comme troisième paramètre. Si la classe implémente l'interface [[yii\base\Configurable]] (p. ex. [[yii\base\BaseObject]]), le tableau de configuration est passé en tant que dernier paramètre du constructeur de la classe ; autrement le tableau de configuration serait appliqué *après* la création de l'objet. 

### Injection par une méthode de rappel PHP <span id="php-callable-injection"></span>

Dans ce cas, le conteneur utilise une fonction de rappel PRP enregistrée pour construire de nouvelles instances d'une classe. À chaque fois que [[yii\di\Container::get()]] est appelée, la fonction de rappel correspondante est invoquée. Cette fonction de rappel est chargée de la résolution des dépendances et de leur injection appropriée dans les objets nouvellement créés. Par exemple :

```php
$container->set('Foo', function ($container, $params, $config) {
    $foo = new Foo(new Bar);
    // ... autres initialisations ...
    return $foo;
});

$foo = $container->get('Foo');
```

Pour cacher la logique complexe de construction des nouveaux objets, vous pouvez utiliser un méthode de classe statique en tant que fonction de rappel. Par exemple :

```php
class FooBuilder
{
    public static function build($container, $params, $config)
    {
        $foo = new Foo(new Bar);
        // ... autres initialisations ...
        return $foo;
    }
}

$container->set('Foo', ['app\helper\FooBuilder', 'build']);

$foo = $container->get('Foo');
```

En procédant de cette manière, la personne qui désire configurer la classe `Foo` n'a plus besoin de savoir comment elle est construite.


Enregistrement des dépendances <span id="registering-dependencies"></span>
------------------------------

Vous pouvez utiliser [[yii\di\Container::set()]] pour enregistrer les dépendances. L'enregistrement requiert un nom de dépendance et une définition de dépendance. Un nom de dépendance peut être un nom de classe, un nom d'interface, ou un nom d'alias ; et une définition de dépendance peut être une nom de classe, un tableau de configuration, ou une fonction de rappel PHP.

```php
$container = new \yii\di\Container;

// enregistre un nom de classe tel quel. Cela peut être sauté. 
$container->set('yii\db\Connection');

// enregistre une interface
// Lorsqu'une classe dépend d'une interface, la classe correspondante
// est instanciée en tant qu'objet dépendant
$container->set('yii\mail\MailInterface', 'yii\swiftmailer\Mailer');

// enregistre un nom d'alias. Vous pouvez utiliser $container->get('foo')
// pour créer une instance de Connection
$container->set('foo', 'yii\db\Connection');

// enregistre une classe avec une configuration. La configuration
// est appliquée lorsque la classe est instanciée par  get()
$container->set('yii\db\Connection', [
    'dsn' => 'mysql:host=127.0.0.1;dbname=demo',
    'username' => 'root',
    'password' => '',
    'charset' => 'utf8',
]);

// enregistre un nom d'alias avec une configuration de classe
// Dans ce cas, un élément "class" est requis pour spécifier la classe
$container->set('db', [
    'class' => 'yii\db\Connection',
    'dsn' => 'mysql:host=127.0.0.1;dbname=demo',
    'username' => 'root',
    'password' => '',
    'charset' => 'utf8',
]);

// enregistre une fonction de rappel PHP 
// La fonction de rappel est exécutée à chaque fois que $container->get('db') est appelée
$container->set('db', function ($container, $params, $config) {
    return new \yii\db\Connection($config);
});

// enregistre une interface de composant 
// $container->get('pageCache') retourne la même instance à chaque fois qu'elle est appelée
$container->set('pageCache', new FileCache);
```

> Tip: si un nom de dépendance est identique à celui de la définition de dépendance correspondante, vous n'avez pas besoin de l'enregistrer dans le conteneur d'injection de dépendances.

Une dépendance enregistrée via `set()` génère une instance à chaque fois que la dépendance est nécessaire. Vous pouvez utiliser [[yii\di\Container::setSingleton()]] pour enregistrer une dépendance qui ne génère qu'une seule instance :

```php
$container->setSingleton('yii\db\Connection', [
    'dsn' => 'mysql:host=127.0.0.1;dbname=demo',
    'username' => 'root',
    'password' => '',
    'charset' => 'utf8',
]);
```


Résolution des dépendances <span id="resolving-dependencies"></span>
--------------------------

Une fois que vous avez enregistré des dépendances, vous pouvez utiliser le conteneur d'injection de dépendances pour créer de nouveau objets, et le conteneur résout automatiquement les dépendances en les instanciant et en les injectant dans les nouveaux objets. Le résolution des dépendances est récursive, ce qui signifie que si une dépendance a d'autres dépendances, ces dépendances sont aussi résolue automatiquement. 

Vous pouvez utiliser [[yii\di\Container::get()]] soit pour créer, soit pour obtenir une instance d'un objet. La méthode accepte un nom de dépendance qui peut être un nom de classe, un nom d'interface ou un nom d'alias. Le nom de dépendance, peut être enregistré [[yii\di\Container::set()|set()]] 
ou [[yii\di\Container::setSingleton()|setSingleton()]]. En option, vous pouvez fournir une liste de paramètres du constructeur de la classe et une [configuration](concept-configurations.md) pour configurer l'objet nouvellement créé. Par exemple :

```php
// "db" est un nom d'alias enregistré préalablement
$db = $container->get('db');

// équivalent à : $engine = new \app\components\SearchEngine($apiKey, $apiSecret, ['type' => 1]);
$engine = $container->get('app\components\SearchEngine', [$apiKey, $apiSecret], ['type' => 1]);
```

En coulisses, le conteneur d'injection de dépendances ne fait rien de plus que de créer l'objet. Le conteneur inspecte d'abord le constructeur de la classe pour trouver les classes dépendantes ou les noms d'interface et résout ensuite ces dépendances récursivement. 

Le code suivant montre un exemple plus sophistiqué. La classe `UserLister` dépend d'un objet implémentant l'interface `UserFinderInterface` ; la classe `UserFinder` implémente cet interface et dépend de l'objet `Connection`. Toutes ces dépendances sont déclarées via l'allusion au type des paramètres du constructeur de la classe. Avec l'enregistrement des dépendances de propriétés, le conteneur d'injection de dépendances est capable de résoudre ces dépendances automatiquement et de créer une nouvelle instance de `UserLister` par un simple appel à `get('userLister')`.

```php
namespace app\models;

use yii\base\BaseObject;
use yii\db\Connection;
use yii\di\Container;

interface UserFinderInterface
{
    function findUser();
}

class UserFinder extends BaseObject implements UserFinderInterface
{
    public $db;

    public function __construct(Connection $db, $config = [])
    {
        $this->db = $db;
        parent::__construct($config);
    }

    public function findUser()
    {
    }
}

class UserLister extends BaseObject
{
    public $finder;

    public function __construct(UserFinderInterface $finder, $config = [])
    {
        $this->finder = $finder;
        parent::__construct($config);
    }
}

$container = new Container;
$container->set('yii\db\Connection', [
    'dsn' => '...',
]);
$container->set('app\models\UserFinderInterface', [
    'class' => 'app\models\UserFinder',
]);
$container->set('userLister', 'app\models\UserLister');

$lister = $container->get('userLister');

// qui est équivalent à :

$db = new \yii\db\Connection(['dsn' => '...']);
$finder = new UserFinder($db);
$lister = new UserLister($finder);
```


Utilisation pratique <span id="practical-usage"></span>
--------------------

Yii crée un conteneur d'injection de dépendances lorsque vous incluez le fichier `Yii.php` dans le [script d'entrée](structure-entry-scripts.md) de votre application. Le conteneur d'injection de dépendances est accessible via [[Yii::$container]]. Lorsque vous appelez [[Yii::createObject()]], la méthode appelle en réalité la méthode [[yii\di\Container::get()|get()]] du conteneur pour créer le nouvel objet. Comme c'est dit plus haut, le conteneur d'injection de dépendances résout automatiquement les dépendances (s'il en existe) et les injecte dans l'objet obtenu. Parce que Yii utilise [[Yii::createObject()]] dans la plus grande partie du code de son noyau pour créer de nouveaux objets, cela signifie que vous pouvez personnaliser ces objets globalement en utilisant [[Yii::$container]].

Par exemple, personnalisons globalement le nombre de boutons de pagination par défaut de l'objet graphique [[yii\widgets\LinkPager]] :

```php
\Yii::$container->set('yii\widgets\LinkPager', ['maxButtonCount' => 5]);
```

Maintenant, si vous utilisez l'objet graphique dans une vue avec le code suivant, la propriété `maxButtonCount` est initialisée à la valeur 5 au lieu de la valeur par défaut 10 qui est définie dans la classe.
```php
echo \yii\widgets\LinkPager::widget();
```

Vous pouvez encore redéfinir la valeur définie par le conteneur d'injection de dépendances via :

```php
echo \yii\widgets\LinkPager::widget(['maxButtonCount' => 20]);
```

> Tip: peu importe de quel type de valeur il s'agit, elle est redéfinie, c'est pourquoi vous devez vous montrer prudent avec les tableaux d'options. Ils ne sont pas fusionnés. 

Un autre exemple est de profiter de l'injection automatique par le constructeur du conteneur d'injection de dépendances. Supposons que votre classe de contrôleur dépende de quelques autres objets, comme un service de réservation d'hôtel. Vous pouvez déclarer la dépendance via un paramètre de constructeur et laisser le conteneur d'injection de dépendances la résoudre pour vous. 

```php
namespace app\controllers;

use yii\web\Controller;
use app\components\BookingInterface;

class HotelController extends Controller
{
    protected $bookingService;

    public function __construct($id, $module, BookingInterface $bookingService, $config = [])
    {
        $this->bookingService = $bookingService;
        parent::__construct($id, $module, $config);
    }
}
```

Si vous accédez au contrôleur à partir du navigateur, vous verrez un message d'erreur se plaignant que l'interface `BookingInterface` ne peut pas être instanciée. Cela est dû au fait que vous devez dire au conteneur d'injection de dépendances comment s'y prendre avec cette dépendance :

```php
\Yii::$container->set('app\components\BookingInterface', 'app\components\BookingService');
```

Maintenant, si vous accédez à nouveau au contrôleur, une instance de `app\components\BookingService` est créée et injectée en tant que troisième paramètre du constructeur. 

Utilisation pratique avancée <span id="advanced-practical-usage"></span>
---------------

Supposons que nous travaillions sur l'API de l'application et ayons :S

- la classe `app\components\Request` qui étende `yii\web\Request` et fournisse une fonctionnalité additionnelle, 
- la classe `app\components\Response` qui étende `yii\web\Response` et devrait avoir une propriété `format` définie à `json` à la création,
- des classes `app\storage\FileStorage` et `app\storage\DocumentsReader` qui mettent en œuvre une certaine logique pour travailler sur des documents qui seraient situés dans un dossier :
  
  
  ```php
  class FileStorage
  {
      public function __construct($root) {
          // whatever
      }
  }
  
  class DocumentsReader
  {
      public function __construct(FileStorage $fs) {
          // whatever
      }
  }
  ```

Il est possible de configurer de multiples définitions à la fois, en passant un tableau de configurations à la méthode  
[[yii\di\Container::setDefinitions()|setDefinitions()]] ou à la méthode [[yii\di\Container::setSingletons()|setSingletons()]].
En itérant sur le tableau de configuration, les méthodes appellent [[yii\di\Container::set()|set()]]
ou [[yii\di\Container::setSingleton()|setSingleton()]] respectivement pour chacun des items.

Le format du tableau de  configurations est :

 - `key`: nom de classe, nom d'interface ou alias. La clé est passée à la méthode
 [[yii\di\Container::set()|set()]] comme premier argument `$class`.
 - `value`: la définition associée à `$class`. Les valeurs possibles sont décrites dans la documentation [[yii\di\Container::set()|set()]]
 du paramètre `$definition`. Est passé à la méthode [[set()]] comme deuxième argument `$definition`.

Par exemple, configurons notre conteneur pour répondre aux exigences mentionnées précédemment :

```php
$container->setDefinitions([
    'yii\web\Request' => 'app\components\Request',
    'yii\web\Response' => [
        'class' => 'app\components\Response',
        'format' => 'json'
    ],
    'app\storage\DocumentsReader' => function ($container, $params, $config) {
        $fs = new app\storage\FileStorage('/var/tempfiles');
        return new app\storage\DocumentsReader($fs);
    }
]);

$reader = $container->get('app\storage\DocumentsReader'); 
// Crée un objet DocumentReader avec ses dépendances tel que décrit dans la configuration.
```

> Tip: le conteneur peut être configuré dans le style déclaratif en utilisant la configuration de l'application depuis la version 2.0.11. 
Consultez la sous-section [Configurations des applications](concept-configurations.md#application-configurations) de l'article du guide  [Configurations](concept-configurations.md).

Tout fonctionne, mais au cas où, nous devons créer une classe  `DocumentWriter`, nous devons copier-coller la ligne qui crée un objet  `FileStorage`, ce qui n'est pas la manière la plus élégante, c'est évident. 


Comme cela est décrit à la sous-section [Résolution des dépendances](#resolving-dependencies) subsection, [[yii\di\Container::set()|set()]]
et [[yii\di\Container::setSingleton()|setSingleton()]] peuvent facultativement des paramètres du constructeur de dépendances en tant que troisième argument. Pour définir les paramètres du constructeur, vous pouvez utiliser le format de tableau de configuration suivant :

 - `key`: nom de classe, nom d'interface ou alias. La clé est passée à la méthode 
 [[yii\di\Container::set()|set()]] comme premier argument `$class`.
 - `value`: un tableau de deux éléments. Le premier élément est passé à la méthode [[yii\di\Container::set()|set()]] comme deuxième argument `$definition`, le second — comme `$params`.

Modifions notre exemple :

```php
$container->setDefinitions([
    'tempFileStorage' => [ // we've created an alias for convenience
        ['class' => 'app\storage\FileStorage'],
        ['/var/tempfiles'] // pourrait être extrait de certains fichiers de configuration
    ],
    'app\storage\DocumentsReader' => [
        ['class' => 'app\storage\DocumentsReader'],
        [Instance::of('tempFileStorage')]
    ],
    'app\storage\DocumentsWriter' => [
        ['class' => 'app\storage\DocumentsWriter'],
        [Instance::of('tempFileStorage')]
    ]
]);

$reader = $container->get('app\storage\DocumentsReader'); 
// Se comporte exactement comme l'exemple précédent
```

Vous noterez la notation `Instance::of('tempFileStorage')`. cela siginifie que  le [[yii\di\Container|Container]] fournit implicitement une dépendance enregistrée avec le nom de  `tempFileStorage` et la passe en tant que premier argument du constructeur
of `app\storage\DocumentsWriter`.

> Note: [[yii\di\Container::setDefinitions()|setDefinitions()]] and [[yii\di\Container::setSingletons()|setSingletons()]]
  methods are available since version 2.0.11.
  
Une autre étape de l'optimisation de la configuration est d'enregistrer certaines dépendances  sous forme de singletons. 
Une dépendance enregistrée via [[yii\di\Container::set()|set()]] est instanciée à chaque fois qu'on en a besoin.
Certaines classes ne changent pas l'état au moment de l'exécution, par conséquent elles peuvent être enregistrées sous forme de singletons afin d'augmenter la performance de l'application.

Un bon exemple serait la classe `app\storage\FileStorage`, qui effectue certaines opérations sur le système de fichiers avec une API simple (p. ex. `$fs->read()`, `$fs->write()`). Ces opération ne changent pas l'état interne de la classe, c'est pourquoi nous pouvons créer son instance une seule fois et l'utiliser de multiples fois.

```php
$container->setSingletons([
    'tempFileStorage' => [
        ['class' => 'app\storage\FileStorage'],
        ['/var/tempfiles']
    ],
]);

$container->setDefinitions([
    'app\storage\DocumentsReader' => [
        ['class' => 'app\storage\DocumentsReader'],
        [Instance::of('tempFileStorage')]
    ],
    'app\storage\DocumentsWriter' => [
        ['class' => 'app\storage\DocumentsWriter'],
        [Instance::of('tempFileStorage')]
    ]
]);

$reader = $container->get('app\storage\DocumentsReader');
```


À quel moment enregistrer les dépendances <span id="when-to-register-dependencies"></span>
-----------------------------------------

Comme les dépendances sont nécessaires lorsque de nouveaux objets sont créés, leur enregistrement doit être fait aussi tôt que possible. Les pratiques recommandées sont : 

* Si vous êtes le développeur d'une application, vous pouvez enregistrer les dépendances dans le [script d'entrée](structure-entry-scripts.md) de votre application ou dans un script qui est inclus par le script d'entrée. 
* Si vous êtes le développeur d'une [extension](structure-extensions.md) distribuable, vous pouvez enregistrer les dépendances dans la classe d'amorçage de l'extension. 


Résumé <span id="summary"></span>
-------

L'injection de dépendances et le [localisateur de services](concept-service-locator.md) sont tous deux des modèles de conception populaires qui permettent des construire des logiciels d'une manière faiblement couplée et plus testable. Nous vous recommandons fortement de lire [l'article de Martin](https://martinfowler.com/articles/injection.html) pour acquérir une compréhension plus profonde de l'injection de dépendances et du localisateur de services. 

Yii implémente son [localisateur de services](concept-service-locator.md) par dessus le conteneur d'injection de dépendances. Lorsqu'un localisateur de services essaye de créer une nouvelle instance d'un objet, il appelle le conteneur d'injection de dépendances. Ce dernier résout les dépendances automatiquement comme c'est expliqué plus haut.

