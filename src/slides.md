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

This talk will assume you have some understanding of PHP, but if you get lost then please ask me to talk you throught stuff afterwards.
-->
---

## What Is A Service?

- Used in all parts of Drupal and many modules.
* Built on the <strong>Symfony Service Container</strong> system.
* A service describes an object in Drupal.
* Dependency injection is used to inject services into other services.
* Forms, controllers, and plugins have services built in.
* Simple to use and powerful.

---

```php
\Drupal::service('thing');
```

---

## What Services Exist?

- Lots!

```bash
drush eval "print_r(\Drupal::getContainer()->getServiceIds());"
```

- Prints a list of over 600 services.
- Most are in the form `date.formatter`.
- Some are in the form `Drupal\Core\Datetime\DateFormatterInterface`, and are used in autoloading.

<!--
The simple version of date.formatter is what is normal in Drupal.
The interface form is used for autoloading.
-->
---

# Using A Service

---
## Using A Service

- Grab the service.
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

## Using A Service

- However! Most of the time you don't want to be using `\Drupal::service()`.
- Drupal will inject the services you need into your service.
- This is called <strong>dependency innjection</strong>.

---

# Dependecny Injection

A quick introduction.

---

## Dependency Injection

Dependecy injection sounds complicated, but its just the practice of <strong>injecting the things the object needs</strong>, instead of <strong>baking them into the class</strong>.

<!--
Some OOP wording here. Make sure this is clear.
-->
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
- Drupal handles all of the object creation for us and will create services with all of the required objects in place.

- All we need to do is ask for our service.
---

## Why Use Dependency Injection In Drupal?

Let's try to create the `path_alias.manager` service to translate a path <em>without</em> using Drupal's dependency injection system.

---

The `path_alias.manager` service wraps the `\Drupal\path_alias\AliasManager` class.


```php
use Drupal\path_alias\AliasManager;

$aliasManager = new AliasManager($pathAliasRepository, $pathPrefixes, $languageManager, $cache, $time);
```

We need to fill in the missing dependencies of `$pathAliasRepository`, `$pathPrefixes`, `$languageManager`, `$cache`, and `$time`.

Let's start with `$pathAliasRepository`.

---

The `$pathAliasRepository` property is an instance of `\Drupal\path_alias\AliasRepository`.

```php
use Drupal\path_alias\AliasManager;
use Drupal\path_alias\AliasRepository;

$pathAliasRepository = new AliasRepository($connection);

$aliasManager = new AliasManager($pathAliasRepository, $pathPrefixes, $languageManager, $cache, $time);
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

$aliasManager = new AliasManager($pathAliasRepository, $pathPrefixes, $languageManager, $cache, $time);
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

$aliasManager = new AliasManager($pathAliasRepository, $pathPrefixes, $languageManager, $cache, $time);
```
<!--
We have made a decision here about where the data will come from.
We are assuming that all the paths are stored in our local database.
Which is an ok assumption to make, but this is how hard coded.
-->
---

The `$cid` property of `AliasPrefixList` is easy as that's just a string.

```php
use Drupal\path_alias\AliasManager;
use Drupal\path_alias\AliasPrefixList;
use Drupal\path_alias\AliasRepository;
use Drupal\Core\Database\Database;

$connection = Database::getConnection();
$pathAliasRepository = new AliasRepository($connection);

$cid = 'path_alias_whitelist';
$pathPrefixes = new AliasPrefixList($cid, $cache, $lock, $state, $alias_repository);

$aliasManager = new AliasManager($pathAliasRepository, $pathPrefixes, $languageManager, $cache, $time);
```

---
<!-- _footer: "" -->

The `$cache` property is an object of type `\Drupal\Core\Cache\CacheFactoryInterface`, so we can use `\Drupal\Core\Cache\CacheFactory` to create this. We first need to create to create a `\Drupal\Core\Site\Settings` object to create that.

```php
use Drupal\path_alias\AliasManager;
use Drupal\path_alias\AliasPrefixList;
use Drupal\path_alias\AliasRepository;
use Drupal\Core\Database\Database;
use Drupal\Core\Site\Settings;

$connection = Database::getConnection();
$pathAliasRepository = new AliasRepository($connection);

$cid = 'path_alias_whitelist';
$settings = Settings::getInstance();
$default_bin_backends = ['bootstrap' => 'cache.backend.chainedfast'];
$cacheFactory = new CacheFactory($settings, $default_bin_backends);
$cache = $cacheFactory->get('bootstrap');
$pathPrefixes = new AliasPrefixList($cid, $cache, $lock, $state, $alias_repository);

$aliasManager = new AliasManager($pathAliasRepository, $pathPrefixes, $languageManager, $cache, $time);
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

Anyone else lost?

```php
$pathManager = \Drupal::service('path_alias.manager');
$normalPath = $pathManager->getPathByAlias('somepath');
```

Seems easier, right?

---

# Creating Your Own Services

---

## Creating Services

- All services are defined in a `[module].services.yml` file in your module directory.

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

- Most commonly, we want to inject our service dependencies.

```yml
services:
  services_argument_example.single_argument:
    class: \Drupal\services_argument_example\SingleArgument
    arguments: ['@serialization.json']
```

---

## Creating Services - Arguments

- Our service class needs to accept the arguments.

```php
<?php

namespace Drupal\services_argument_example;

use Drupal\Component\Serialization\SerializationInterface;

class SingleArgument implements SingleArgumentInterface {

  public function __construct(protected SerializationInterface $serializer) {}
}
```
<!--
This is call constructor property promotion. The $serialiser property is created in our class.
This follows SOLID principles again, in this case the dependency inversion principle.
-->

---

## Creating Services - Arguments

- The service can be used within the class.

```php
  public function removeItemFromPayload(string $payload, int $id):string {
    $payload = $this->serializer->decode($payload);
    foreach ($payload as $key => $item) {
      if ($item['id'] === $id) {
        unset($payload[$key]);
      }
    }
    return $this->serializer->encode($payload);
  }
```

---

autowire?

---

# Controllers And Forms

---
<!-- _footer: "" -->
## Controllers And Forms

- Some types of Drupal object don't use the services system.
* Instead they implement `\Drupal\Core\DependencyInjection\ContainerInjectionInterface`.
* Drupal will see this and use a method called `create()` to create the service.
* The `create()` method must return an instance of the service object.  

---
<!-- _footer: "" -->
## Controllers And Forms

- Best practice is to assign the properties you need in the create() method.

```php
class ControllerExample extends ControllerBase {

  protected $dateFormatter;

  public static function create(ContainerInterface $container) {
    $instance = new static();
    $instance->dateFormatter = $container->get('date.formatter');
    return $instance;
  }

  /// .. 
}
```

---
## Plugins

- Plugins have a similar interface called `\Drupal\Core\Plugin\ContainerFactoryPluginInterface`
- This has the same `create()` method.

---

<!-- _footer: "" -->
## Tips For Creating Services

- Don't use `\Drupal::sercices()` inside your service classes, use depedency injection instead.
- Use <strong>SOLID</strong> principles. Create small service classes that perform one task.
- Keep constructors as simple as possible. Just assign your dependencies to properties.
- Don't "hand off" dependencies to internal classes, use additional services.

---

## Tips For Creating Services

- Services make unit testing much easier.
  - Test your services on their own with unit testing.
  - Then move up to functional testing for test services working together.
  - Functional tests can test your module controllers, forms, drush commands etc.

---

# Altering Services

---



---

# Next Steps

There's much more to Drupal services, try looking up

- Decoration of services
- Tagged services
- Access control
- Logging
- Event handlers

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
