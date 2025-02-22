---
theme: uncover
paginate: true
class:
  - lead
  - invert
size: 16:9
style: |
  .small-text {
    font-size: 0.75rem;
  }
  p {
    text-align: left;
  }
  p.centre {
    text-align: center;
  }
  .hidden-bullets li {
    list-style-type: none
  }
footer: "Philip Norton [hashbangcode.com](https://www.hashbangcode.com) [fosstodon.org@hashbangcode](https://fosstodon.org/@hashbangcode) [fosstodon.org@philipnorton42](https://fosstodon.org/@philipnorton42)"
marp: true
---

# An Introduction To Drupal Services
<p class="centre">Philip Norton<br>
<small>DrupalCamp England 2025</small></p>
<!-- Speaker notes will appear here. -->

---

# Philip Norton
- Developer at Code Enigma
- Writer at `#! code` (www.hashbangcode.com)
- NWDUG co-organiser
![bg h:50% right:40%](../src/assets/images/lily58.png)

<!--
- Doing Drupal for about 15 years.
- Programming in general for about 20 now.
- That is a picture of a Lily58 mechanical keyboard.
-->

---
<!-- _footer: "" -->
## Source Code
- Talk is available at:
<small>https://github.com/hashbangcode/drupal-services-talk</small> 
- All code seen can be found at:
<small>[https://github.com/hashbangcode/drupal_services_example](https://github.com/hashbangcode/drupal_services_example)</small>
- I have also written about Drupal services on <small>[www.hashbangcode.com](www.hashbangcode.com)</small>

![bg h:50% right:40%](../src/assets/images/qr_slides.png)

<!-- 
- Scan the QR code for the talk repo. Which has links to all of the resources you need.
-->
---

# An Introduction To Drupal Services

<!--
Drupal services are an integral part of how Drupal is structured internally. Everything from connecting to a database to decoding JSON has a service you can use to do it.

This talk will introduce you to what services are and how to use them. We'll talk about why they are used and then go onto building our own services.

This talk will assume you have some understanding of PHP and OOP, but if you get lost then please ask me to talk you throught stuff afterwards.
-->
---

# Why Talk About Services?

<!--
- I gave a version of this talk at DrupalCamp London in 2018.
- A bit has changed since then so I thought it was time to an update.
- They are an essential part of Drupal.
- If you are going to build or maintain a module then you will use services.
- I quite like back to basics talks.
- Coming soon are service hooks. We will soon lose some hooks and need to use service based hooks instead.
-->

---

## What Is A Service?

- Used in all parts of Drupal and many modules.
* Built on the <strong>Symfony Service Container</strong> system.

* A service describes an object in Drupal.

* Dependency injection is used to inject services into other services.

* Simple to use and powerful.

---

# Using A Service

---

## Using A Service

- Services can be accessed via the Drupal object.

```php
$serviceObject = \Drupal::service('thing');
```

---
## Using A Service

- To use a service, pick the service you need.
- Use it.

```php
$pathManager = \Drupal::service('path_alias.manager');
$normalPath = $pathManager->getPathByAlias('somepath');
```
<!--
This is the path alias manger service. The above code will convert
"somepath" into a Drupal path, something like "node/123".
-->
---

## Using A Service
- You can also chain the call.

```php
$normalPath = \Drupal::service('path_alias.manager')->getPathByAlias('somepath');
```

---

## What Services Exist?

- There are lots!

```bash
drush eval "print_r(\Drupal::getContainer()->getServiceIds());"
```

- Prints a list of over 600 services in Drupal core.
- Most are in the form `date.formatter`.
- Some are in the form `Drupal\Core\Datetime\DateFormatterInterface`, and are used in autoloading.

<!--
The simple version of date.formatter is what is normal in Drupal.
The interface form is used for autoloading.
-->
---

## Using A Service

- However! Most of the time you don't want to be using `\Drupal::service()`.
* Unless you are running code in a hook you should be injecting services into your own code.
* Drupal will handle what services are needed to create your needed service.
* This is called <strong>dependency injection</strong>.

---

# Dependecny Injection

A quick introduction.

---

## Dependency Injection

Dependecy injection sounds complicated, but its just the practice of <strong>injecting the things the object needs</strong>, instead of <strong>baking them into the class</strong>.

---

## Dependency Injection

- Let's say you had this class (not a Drupal thing).

```php
class Page {
  protected $database;
  public function __construct() {
    $this->database = new PDO('mysql:host=localhost;dbname=test', 'username', 'password');
  }
}
$page = new Page();
```

- What happens if you want to change the credentials? Or change the database itself?
- You would need to edit the class. 
<!--
Think SOLID principles.
-->

---

## Dependency Injection

- We can change this to inject the database dependency as we create the Page object.

```php
class Page {
  protected $database;
  public function __construct(DbConnectionInterface $database) {
    $this->database = $database;
  }
}
$database = new MysqlDatabase();
$page = new Page($database);
```
<!--
Dependency Inversion principle.
-->
---

## Dependency Injection

- Drupal handles all the dependency creation automatically.
- It will create services with all of the required objects.
- All we need to do is ask for our service.

---

## Why Use Dependency Injection In Drupal?

Let's try to create the `path_alias.manager` service to translate a path <em>without</em> using Drupal's automatic dependency injection system.

---

The `path_alias.manager` service wraps the `\Drupal\path_alias\AliasManager` class.

```php
use Drupal\path_alias\AliasManager;

$aliasManager = new AliasManager($pathAliasRepository, $pathPrefixes,
$languageManager, $cache, $time);
```

<!-- 
We need to fill in the missing dependencies of `$pathAliasRepository`, `$pathPrefixes`, `$languageManager`, `$cache`, and `$time`.

Let's start with `$pathAliasRepository`.
-->
---

The `$pathAliasRepository` property is an instance of `\Drupal\path_alias\AliasRepository`.

```php
use Drupal\path_alias\AliasManager;
use Drupal\path_alias\AliasRepository;

$pathAliasRepository = new AliasRepository($connection);

$aliasManager = new AliasManager($pathAliasRepository, $pathPrefixes, 
$languageManager, $cache, $time);
```

The `AliasRepository` class takes a property of `$connection`.

---

The `$connection` property is a connection to the database, which we can create using the `\Drupal\Core\Database\Database` class.

```php
use Drupal\path_alias\AliasManager;
use Drupal\path_alias\AliasRepository;
use Drupal\Core\Database\Database;

$connection = Database::getConnection();
$pathAliasRepository = new AliasRepository($connection);

$aliasManager = new AliasManager($pathAliasRepository, $pathPrefixes, 
$languageManager, $cache, $time);
```

Next, let's look at `$pathPrefixes`.

---

The `$pathPrefixes` property is an instance of `AliasPrefixList`, which has more dependencies.

```php
use Drupal\path_alias\AliasManager;
use Drupal\path_alias\AliasPrefixList;
use Drupal\path_alias\AliasRepository;
use Drupal\Core\Database\Database;

$connection = Database::getConnection();
$pathAliasRepository = new AliasRepository($connection);

$pathPrefixes = new AliasPrefixList($cid, $cache, $lock, $state, $alias_repository);

$aliasManager = new AliasManager($pathAliasRepository, $pathPrefixes,
$languageManager, $cache, $time);
```
<!--
We have made a decision here about where the data will come from.
We are assuming that all the paths are stored in our local database.
Which is an ok assumption to make, but this is how hard coded.
-->
---

The `$cid` property of `AliasPrefixList` is just a string.

```php
use Drupal\path_alias\AliasManager;
use Drupal\path_alias\AliasPrefixList;
use Drupal\path_alias\AliasRepository;
use Drupal\Core\Database\Database;

$connection = Database::getConnection();
$pathAliasRepository = new AliasRepository($connection);

$cid = 'path_alias_prefix_list';
$pathPrefixes = new AliasPrefixList($cid, $cache, $lock, $state, $alias_repository);

$aliasManager = new AliasManager($pathAliasRepository, $pathPrefixes,
$languageManager, $cache, $time);
```

---
<!-- _footer: "" -->

```php
use Drupal\path_alias\AliasManager;
use Drupal\path_alias\AliasPrefixList;
use Drupal\path_alias\AliasRepository;
use Drupal\Core\Database\Database;
use Drupal\Core\Site\Settings;

$connection = Database::getConnection();
$pathAliasRepository = new AliasRepository($connection);

$cid = 'path_alias_prefix_list';

$settings = Settings::getInstance();
$default_bin_backends = ['bootstrap' => 'cache.backend.chainedfast'];
$cacheFactory = new CacheFactory($settings, $default_bin_backends);
$cache = $cacheFactory->get('bootstrap');

$pathPrefixes = new AliasPrefixList($cid, $cache, $lock, $state, $alias_repository);

$aliasManager = new AliasManager($pathAliasRepository, $pathPrefixes,
$languageManager, $cache, $time);
```
<!--

We have now made several assumptions about our settings, the type of cache we will use, where the cache bin is stored.
This has created very brittle code that is prone to breakage.
Also, we still have another 3 properties to do just to get AliasPrefixList created! All of which will need objects creating, and some of those objects will have more objects, some of which might need extra stuff adding.
Then, there's more properties we need to load for AliasManager!
We are only half way through creating objects.

Worse, every time you want to transform a path you'll need to copy this entire block of code.
-->
---

Anyone else lost? ...  :/

---

```php
$pathManager = \Drupal::service('path_alias.manager');
$normalPath = $pathManager->getPathByAlias('somepath');
```

Seems easier, right?

---

# Creating Your Own Services

---

## Creating Services

- Custom services are defined in a `[module].services.yml` file in your module directory.

```yml
services:
  services_simple_example.simple_service:
    class: \Drupal\services_simple_example\SimpleService
```
---

## Creating Services

- Create the class for your service.

```php
<?php

namespace Drupal\services_simple_example;

class SimpleService implements SimpleServiceInterface {

  public function getArray():array {
    return [];
  }
}
```

---

## Creating Services

- You can now use your service like any Drupal service.

```php
$simpleService = \Drupal::service('services_simple_example.simple_service');
$array = $simpleService->getArray();
```

---

## Creating Services - Arguments

- Most commonly, we want to inject our service dependencies.

```yml
services:
  services_argument_example.single_argument:
    class: \Drupal\services_argument_example\SingleArgument
    arguments: ['@password_generator']
```
<!--
We are injecting the service to generate passwords.
-->
---

## Creating Services - Arguments

- Your services can accept a number of arguments.

  - `@` for another service (`@config.factory`).
  - `%` for a parameter (`%site.path%`).
  - 'config' = A string, in this case ‘config’.

<!--
Parameters are set in services files, but can be overridden at runtime by Drupal.
Strings are used to pass static properties to factories to generate different types of object.
-->

---

## Creating Services - Arguments

- Our service class needs to accept the arguments in the constructor.

```php
<?php

namespace Drupal\services_argument_example;

use Drupal\Component\Serialization\SerializationInterface;

class SingleArgument implements SingleArgumentInterface {

  public function __construct(protected PasswordGeneratorInterface $passwordGenerator) {
  }
}
```
<!--
This is call constructor property promotion. The $passwordGenerator property is created in our class.
Requiring the interface follows SOLID principles, in this case the dependency inversion principle.
-->

---

## Creating Services - Arguments

- The service can be used within the class.

```php
class SingleArgument implements SingleArgumentInterface {

  public function __construct(protected PasswordGeneratorInterface $passwordGenerator) {
  }

  public function generate12CharacterPassword():string {
    return $this->passwordGenerator->generate(12);
  }
}
```

---

## Creating Services - Autowiring

- You don't need to add all of your dependencies by hand, you can use autowiring to do this for you.
- Autowiring works by nominating services that correspond to interfaces.

```yml
services:
  Drupal\Core\Password\PasswordGeneratorInterface: '@password_generator'
```
<!--
This is from the Drupal core.services.yml file, where a lot of services are defined.
-->
---

## Creating Services - Autowiring

- Then, we need to add the `autowire: true` directive to the service definition for our service.

```yml
services:
  services_autowire_example.autowire_example:
    class: \Drupal\services_autowire_example\AutowireExample
    autowire: true
```

---

## Creating Services - Autowiring

- Alternatively, you can set a default in your service file that all services will be autowired.

```yml
services:
  _defaults:
    autowire: true

  services_autowire_example.autowire_example:
    class: \Drupal\services_autowire_example\AutowireExample
```

---
<!-- _footer: "" -->
## Creating Services - Autowiring

- Create your class as normal. The interfaces you nominate will be translated into services and automatically injected into your constructor.

```php
namespace Drupal\services_autowire_example;

use Drupal\Core\Password\PasswordGeneratorInterface;

class AutowireExample implements AutowireExampleInterface {

  public function __construct(protected PasswordGeneratorInterface $passwordGenerator) {
  }

  public function generate12CharacterPassword():string {
    return $this->passwordGenerator->generate(12);
  }
}
```

---

# Controllers And Forms

---
<!-- _footer: "" -->
## Controllers And Forms

- Some types of Drupal object (especially Controllers and Forms) don't use `*.services.yml` files.
- Instead they implement 
```
\Drupal\Core\DependencyInjection\ContainerInjectionInterface
```
* Drupal will see this and use a method called `create()` in the class to create the service.
* The `create()` method must return an instance of the service object.  

---
<!-- _footer: "" -->
## Controllers And Forms

- Best practice is to assign the properties you need in the `create()` method.

```php
class ControllerExample extends ControllerBase {

  protected $passwordGenerator;

  public static function create(ContainerInterface $container) {
    $instance = new static();
    $instance->passwordGenerator = $container->get('password_generator');
    return $instance;
  }

  /// .. 
}
```
<!--
The static() here is a reference to the current class. This calls the constructor but you probably don't need to define the controller.
-->
---
## Plugins

- Plugins have a similar interface called
```
\Drupal\Core\Plugin\ContainerFactoryPluginInterface
```
- This has the same `create()` method system, although you need to pass the plugin arguments to the parent controller.

<!--
Plugins work in the same way, but plugins will have additional arguments that need to be passed upstream. 
-->
---

## Plugins

- The plugin controller does have a function in this instance.

```php
  public static function create(ContainerInterface $container, 
    array $configuration,
    $plugin_id,
    $plugin_definition
  ) {  
    $instance = new static($configuration, $plugin_id, $plugin_definition);
    $instance->passwordGenerator = $container->get('password_generator');
    return $instance;
  }
```

---

# Hook Service Classes

---

## Hook Service Classes

- New in Drupal 11.1.0!
- Procedural hooks are being replaced by an OOP approach.
- Not all hooks are being replaced. For example:
  - `hook_install()` `hook_update_N()`
  - `hook_preprocess_HOOK()`
- Built using services.
- See https://www.drupal.org/node/3442349

<!--
This isn't a rug pull. All your normal hooks will work for the time being.
-->

---

## Hook Service Classes

- Define the service to put our hooks in.

```yml
services:
  services_hooks_example.node_hooks:
    class: \Drupal\services_hooks_example\Hook\NodeHooks
    autowire: true
```

---
<!-- _footer: "" -->
## Hook Service Classes

- Define our hooks using PHP attributes.

```php
namespace Drupal\services_hooks_example\Hook;

use Drupal\Core\Hook\Attribute\Hook;
use Drupal\node\NodeInterface;

class NodeHooks {
  /**
   * Implements hook_ENTITY_TYPE_insert() for node entities.
   */
  #[Hook('node_insert')]
  public function nodeInsert(NodeInterface $node) {
    // Act on the hook being triggered.
  }
}
```
--- 

## Hook Service Classes

- For the time being, you are encouraged to add a shim procedural hook in the usual place.

```php
use Drupal\node\NodeInterface;

#[LegacyHook]
function mymodule_node_insert(NodeInterface $node) {
  \Drupal::service('services_hooks_example.node_hooks')->nodeInsert($NodeInterface $node);
}
```
<!--
There doesn't appear to be any word on deprications of procedural hooks.
In fact many theme and install hooks will remain.
The aim is to allow low level hooks that have no dependencies to operate as they always have.
-->
---

# Tagged Services

---

## Tagged Services

- Some services can be tagged.
- This gives them special abilities.
- Listening to events, controlling access, set up caching, etc. 

---

## Tagged Services - Events

- The simplest tagged service is the event handler.
- This service will be triggered when events happen.

```yml
services:
  eventsubscriber_example.eventlistner_service:
    class: \Drupal\eventsubscriber_example\EventSubscriber\EventListenerService
    tags:
      - { name: event_subscriber }
```

---

## Tagged Services - Events

```php
namespace Drupal\eventsubscriber_example\EventSubscriber;

use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\Event\RequestEvent;
use Symfony\Component\HttpKernel\KernelEvents;

class EventListenerService implements EventSubscriberInterface {

  public function onRequest(RequestEvent $event) {
    // Runs when a request is made.
  }

  public static function getSubscribedEvents(): array {
    return [KernelEvents::REQUEST => [['onRequest', 1]]];
  }
}

```

---
<!-- _footer: "" -->
## Tagged Services - Autoconfigure

- Use the `autoconfigure: true` directive to automatically tag classes. 

```yml
services:
  _defaults:
    autoconfigure: true
  services_autoconfigure_example.autoconfigured_service:
    class: \Drupal\services_autoconfigure_example\EventSubscriber\AutoconfiguredService
```
- For events, Drupal looks for services that extend <p class="small-text">\Symfony\Component\EventDispatcher\EventSubscriberInterface</p>

---

# Tips For Creating Services

---

<!-- _footer: "" -->
## Tips For Creating Services

- Don't use `\Drupal::services()` inside your service classes, use depedency injection instead.
* Use <strong>SOLID</strong> principles. Create small service classes that perform one task.
* Don't have services with lots of arguments. This tends to show that the service is doing too much.
* Keep constructors as simple as possible. Just assign your dependencies to properties.
 
---
## Tips For Creating Services

- Don't "hand off" dependencies to internal classes, use additional services.
* Consider not creating a service if the class has no dependencies.
* If you need to alter the class at runtime then make a service to allow Drupal to do that.

---

## Tips For Creating Services

- Services make unit testing much easier.
  - Test your services on their own with unit testing.
  - Then move up to functional testing for test services working together.
  - Functional tests can test your module controllers, forms, drush commands etc.

---

# Altering Services

---

## Altering Services

- All services can be modified to change their funcitonality.
- This can be done in two ways, depending on your needs:
  - Decorating
  - Altering

---

## Altering Services: Decorating

- Services can be decorated to create your own service that extends another service.
- The original service will still exist, but you will have a new service that accepts the same arguments.

```yml
services:
  services_decorator_example.decorated_password_generator:
    class: \Drupal\services_decorator_example\DecoratedPasswordGenerator
    decorates: password_generator
```
<!--
Here, we are decorating the password_generator and creating our own service called services_decorator_example.decorated_password_generator.
The critical thing is that the existing service class still exists. The new service just extends that.
-->

---
<!-- _footer: "" -->
## Altering Services: Decorating

- We just extend the original class.

```php
<?php

namespace Drupal\services_decorator_example;

use Drupal\Core\Password\DefaultPasswordGenerator;

/**
 * Decorates the DefaultPasswordGenerator class.
 */
class DecoratedPasswordGenerator extends DefaultPasswordGenerator {

  protected $allowedChars = 'abcdefghijkmnopqrstuvwxyz';

}

```

---
<!-- _footer: "" -->
## Altering Services: Altering

- Override the serivce completely and replace it with your own.
* Create a class that has the name `[ModuleName]ServiceProvider`, which extends the class <p class="small-text">\Drupal\Core\DependencyInjection\ServiceProviderBase</p>
* Drupal will pick up this class and run the `register()` and `alter()` methods.

<!-- 
Use register() to register a new service in Drupal. Useful for highly dynamic services where you want to automatically create your services at run time.

Use alter() to alter the service registry and change any registered service in the site.
-->
---
<!-- _footer: "" -->
## Altering Services: Altering

- The `alter()` method is can be used to alter an existing service.

```php
<?php

namespace Drupal\joke_api_stub;

use Drupal\Core\DependencyInjection\ContainerBuilder;
use Drupal\Core\DependencyInjection\ServiceProviderBase;

class JokeApiStubServiceProvider extends ServiceProviderBase {
  public function alter(ContainerBuilder $container) {
    // Replace the \Drupal\joke_api\JokeApi class with our own stub class.
    $definition = $container->getDefinition('joke_api.joke');
    $definition->setClass('Drupal\joke_api_stub\JokeApiStub');
  }
}
```
<!--
Here, we are altering the joke_api.joke service to replace it with our a stub service that doesn't integrate with the Joke API.
-->

---
<!-- _footer: "" -->

[![width:1100px Joke API Stub](../src/assets/images/joke_api_video.png)](https://www.youtube.com/watch?v=SBrF96uM0Z8 "Joke API Stub")

---

# Next Steps

There's much more to Drupal services, try looking up

- Access control services
- Factories
- Lazy services
- Public and private services
- Service aliases
---
<!-- _footer: "" -->
## Resources

- Services and DI on `#! code` <small>https://www.hashbangcode.com/tag/dependency-injection</small>
- Custom code seen is code available at <small>https://github.com/hashbangcode/drupal_services_example</small>
- Services and Dependency Injection - <small>https://www.drupalatyourfingertips.com/services</small>
- Structure of a service file https://www.drupal.org/docs/drupal-apis/services-and-dependency-injection/structure-of-a-service-file
<!--
- See the list at the repo page https://github.com/hashbangcode/drupal-services-talk.
-->
---

## Questions?

- Slides: https://github.com/hashbangcode/drupal-services-talk

![bg h:50% right:40%](../src/assets/images/qr_slides.png)

---

## Thanks!

- Slides: https://github.com/hashbangcode/drupal-services-talk

![bg h:50% right:40%](../src/assets/images/qr_slides.png)
