# Migrating ZF2 to ZF3

> This quide (in progress) should provide steps that might help you migrating ZF2 
application to ZF3 in minimum steps.


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

You may have different error reporting rules for production environment but 
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

* https://github.com/zendframework/zend-mvc/issues/103 
* https://zendframework.github.io/zend-mvc/migration/to-v2-7/



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

Now change the module structure as recommended in ZF3 version:

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
       │   │    └── Helper
       │   └── Module.php 
       │    
       └── view
           └── album
               └── album
                   └── index.phtml
```

TODO: Updates in Module.php file, composer.json structure ...

#### More to this topic

* http://zf2.readthedocs.io/en/latest/modules/zend.module-manager.intro.html
* https://docs.zendframework.com/tutorials/migration/to-v3/application/#autoloading


## Contributions

* Open pull request with improvements
* Discuss ideas and experiences in issues
