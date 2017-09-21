# Migrating ZF2 to ZF3

> This quide should provide informations that might help you migrating ZF2 
MVC application to ZF3 in minimum steps.


### Where to start

Choose one module in your application and start refactoring while your 
changes will still be running **under ZF2 version**. 

## 1. Remove getServiceLocator() method from controllers, controller plugins ...

When you upgrade to 2.7 version you will probably start seeing PHP deprecated message: 

```
Deprecated: ServiceManagerAwareInterface is deprecated and will be removed in version 3.0, 
along with the ServiceManagerAwareInitializer. Please update your class XXX to remove the implementation, 
and start injecting your dependencies via factory instead.
```

To **hide** this message insert into index.php: 

```php 
error_reporting(E_ALL ^ E_USER_DEPRECATED);
```

> You may have different error reporting rules for production environment but 
this will show all messages except `E_USER_DEPRECATED`. 

### Move dependency to factory
Create factory for every class that's using `getServiceLocator()` implementation.

### 1. Controller factory example

OutfitControllerFactory.php

```php
<?php
namespace Outfit\Controller;

use Interop\Container\ContainerInterface; // since 2.6.0, required  by zendframework/zend-servicemanager
use Outfit\Model;

class OutfitControllerFactory
{

    public function __invoke(ContainerInterface $container)
    {
        $container = $container->getServiceLocator(); // in ZF3 you will delete this line

        return new OutfitController($container->get(Model\Shops::class), $container->get('config'));
    }
}
```

OutfitController.php

```php
<?php
namespace Outfit\Controller;

use Zend\Mvc\Controller\AbstractActionController;
use Zend\View\Model\ViewModel;
use Outfit\Model\Shops;

class OutfitController extends AbstractActionController
{

    public $config;

    public $modelShops;

    public function __construct(Shops $modelShops, $config)
    {
        $this->modelShops = $modelShops;
        $this->config = $config;
    }
    
    ...
}    
```

module.config.php

```php
<?php
namespace Outfit;

return array(
    'controllers' => array(
        'factories' => array(
            Controller\Outfit::class => Controller\OutfitControllerFactory::class
        )
    ),
    'service_manager' => array(
        'factories' => array(
            Model\Shops::class => Model\ShopsFactory::class
        )
    )
    
    ...
)    
```

### 2. View helper factory example

LastItemsFactory.php

```php
<?php
namespace Bazaar\View\Helper;

use Interop\Container\ContainerInterface;
use Bazaar\Model\Items;

class LastItemsFactory
{

    public function __invoke(ContainerInterface $container)
    {
        $container = $container->getServiceLocator();
        
        return new LastItems($container->get(Items::class));
    }
}
```

> In the controller and view helper factories call `getServiceLocator()` to access defined services. 
In a factories for controller plugins, models or module service you can access services from 
`$container` variable without the line `$container = $container->getServiceLocator();`

#### More to this topic

* [Deprecated error](https://github.com/zendframework/zend-mvc/issues/103)
* [ZF2 Upgrading to 2.7](https://zendframework.github.io/zend-mvc/migration/to-v2-7/)



## 2. Adopt PSR-4 directory structure for Composer autoloading

The StandardAutoloader is designed as a PSR-0 compliant autoloader. Changing structure to PSR-4 
standard brings autoloading via Composer on the module.
 
> It's possible to keep structure in PSR-0 but adopting PSR-4 is recommended and the structure is simplified. 
PSR-0 is [deprecated](http://www.php-fig.org/psr/psr-0/) since 2014-10-21. 

The recommended module structure of a typical MVC-oriented ZF2 module is as follows:

```
zf2-application
└─ module
   └── Album
       ├── config
       │   └── module.config.php
       │
       ├── src
       │   └── Album
       │       ├── Controller
       │       │   └── AlbumController.php
       │       ├── Form
       │       ├── Model
       │       ├── Service
       │       └── View
       │           └── Helper
       ├── view
       │   └── album
       │       └── album
       │           └── index.phtml
       │
       ├── autoload_classmap.php
       ├── autoload_function.php 
       ├── autoload_register.php 
       ├── Module.php 
       └── template_map.php
```

Now change the module structure as recommended in ZF3 version (while still running ZF2):

```
zf3-application
└─ module
   └── Album
       ├── config
       │   ├── module.config.php
       │   └── template_map.config.php
       │
       ├── src
       │   ├── Controller
       │   │   └── AlbumController.php
       │   ├── Form
       │   ├── Model
       │   ├── Service
       │   ├── View
       │   │   └── Helper
       │   └── Module.php 
       │    
       └── view
           └── album
               └── album
                   └── index.phtml
```

Move the *Module.php* into `src` directory. You can remove `getAutoloaderConfig()` method. The file may contain only `getConfig()`:

```php
<?php
namespace Album;

use Zend\Config;

class Module
{
    public function getConfig()
    {
        $config = new Config\Config(include __DIR__ . '/../config/module.config.php');
        $router = Config\Factory::fromFile(__DIR__ . '/../config/router.ini', true);
        $navigation = Config\Factory::fromFile(__DIR__ . '/../config/navigation.xml', true);
        
        $config->merge($router);
        $config->merge($navigation);
        
        return $config;
    }
}
```

Because the *template_map.php* is moved to config directory (and also renamed), you need to update `template_map` path.

module.config.php

```php
<?php
namespace Album;

return array(

    ...
    
    'view_manager' => array(
        'template_map' => include __DIR__ . '/template_map.config.php', // was 'template_map' => include __DIR__ . '/../template_map.php'
        'template_path_stack' => array(
            __DIR__ . '/../view'
        )
    ),
    
    ....
);
```

And in the *template_map.config.php* file change the path or rebuild the file.

> You should be using *template_map.config.php* for production to avoid performance expence. More informations in
[view_manager](http://zf2.readthedocs.io/en/latest/modules/zend.view.quick-start.html#configuration) configuration.
*templatemap_generator.php* was moved to the zend-view component with the 2.8.0 release, and is 
available via ./vendor/bin/templatemap_generator.php.

```php
<?php
return [
    'album/album/index' => __DIR__ . '/../view/album/album/index.phtml',
];
```

The last step is to update the *composer.json* autoload configuration as follows:

```
{
  "name" : "ZF2 api",
  "description" : "ZF management system",
  "require" : {
    "php" : ">=5.5.10",
    "zendframework/zendframework" : "^2.5"
  },
  "autoload" : {
    "psr-4" : {
      "Album\\" : "module/Album/src/"
    }
  }
}
```

and run `$ composer dump-autoload`.

After rebuild of composer autoload you should be able to run module in ZF3 recommended structure under ZF2 version 
and with composer autoloading. Don't forget to dump cache if you have `config_cache_enabled`.

#### More to this topic

* [ZF2 Module System](http://zf2.readthedocs.io/en/latest/modules/zend.module-manager.intro.html)
* [ZF3 Autoloading](https://docs.zendframework.com/tutorials/migration/to-v3/application/#autoloading)
* [Optimizing Composer's autoloader performance](http://mouf-php.com/optimizing-composer-autoloader-performance)


# Switching to ZF3 ~ Zend Framework

At first read the upgrading and migration guide. After that check zendframework/ZendSkeletonApplication to see composer.json in particular.

* [Tutorial Upgrading applications](https://docs.zendframework.com/tutorials/migration/to-v3/application/) 
* [Migration guide](https://zendframework.github.io/zend-servicemanager/migration/)
* [zendframework/ZendSkeletonApplication](https://github.com/zendframework/ZendSkeletonApplication)

Insert into *composer.json* `autoload` section paths for modules your application has.
If you are not sure what Zend Framework components you should include in composer `require` section, start with skeleton example and
you will be promted later about missing components (zend-navigation, zend-validator, zend-paginator ...).

> Case sensitive: In v3, service names are case sensitive, and are not normalized in any way.


### FactoryInterface implementation

Next step is to edit again all factories and let them implement `FactoryInterface`. So the `OutfitControllerFactory` 
from previous example will be updated as:

OutfitControllerFactory.php before

```php
<?php
namespace Outfit\Controller;

use Interop\Container\ContainerInterface; // since 2.6.0, required  by zendframework/zend-servicemanager
use Outfit\Model;

class OutfitControllerFactory
{

    public function __invoke(ContainerInterface $container)
    {
        $container = $container->getServiceLocator(); // in ZF3 you will delete this line

        return new OutfitController($container->get(Model\Shops::class), $container->get('config'));
    }
}
``` 

Updated

```php
<?php
namespace Outfit\Controller;

use Zend\ServiceManager\Factory\FactoryInterface;
use Interop\Container\ContainerInterface;
use Outfit\Model;

class OutfitControllerFactory implements FactoryInterface
{

    public function __invoke(ContainerInterface $container, $requestedName, array $options = null)
    {
        return new OutfitController($container->get(Model\Shops::class), $container->get('Config'));
    }
}
``` 

If you have controller without factory definition, use *Invokable factory* for that:

module.config.php

```php
<?php
namespace Outfit;

use Zend\ServiceManager\Factory\InvokableFactory;

return array(
    'controllers' => array(
        'factories' => array(
            Controller\IndexController::class => InvokableFactory::class
        ),
    ),
    
    ...
    
```

And that should be basically all steps to **finish migration**.

## Some tips and recommendations

* Avoid closures for implementing dependencies in Module.php and create factory file for that.
* Avoid using initializers - `InitializerInterface`. Use delegators `DelegatorFactoryInterface` for better
performance. 

#### More to this topic

* [Delegator Factories in Zend Framework 2](http://ocramius.github.io/blog/zend-framework-2-delegator-factories-explained/)
* [zend-servicemanager - Delegators](https://docs.zendframework.com/zend-servicemanager/delegators/)


## Contributions

* Open pull request with improvements
* Discuss ideas and experiences in issues
