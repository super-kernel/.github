<p align="right">
  <strong>English</strong> | <a href="autoloader.zh-CN.md">中文文档</a>
</p>

# SuperKernel Autoloader

Package: [`super-kernel/autoloader`](https://github.com/super-kernel/autoloader) · [Packagist](https://packagist.org/packages/super-kernel/autoloader) · [Documentation index](../README.md)

## Purpose

`super-kernel/autoloader` gives a process one class loader with a known, fixed class map. Code that needs to see a class before it loads, such as `super-kernel/aop` when it weaves aspects into a class, reads the pending class map and intercepts loading through this loader. A long-running runtime can remove the loader once every class has loaded.

The class maps come from the metadata that `skernel build` generates and `super-kernel/metadata` restores; the loader does not read Composer files. It replaces Composer's loader instead of cooperating with it.

## Capabilities

### Take over class loading

Resolve `SuperKernel\Contract\AutoLoaderInterface` from a SuperKernel container. Its provider, `SuperKernel\Autoloader\Provider\AutoLoaderProvider`, then:

1. combines the class maps of the packages in the order of `PackageManagerInterface::all()`, the root first; when two packages map the same class, the earlier one wins, so the root has priority;
2. creates the loader while the previous loaders still work; meanwhile, mapped classes load by requiring their files directly, with PHP's own errors for a missing file;
3. removes every registered autoloader and registers the new loader alone.

The takeover is final: the previous loaders, Composer's included, are never restored. If step 2 fails, the provider restores the previous loaders and the exception propagates; classes already loaded and the effects of their files remain.

Each resolution creates a new loader, and the container keeps it; the loader keeps no static instance. In a long-running runtime that forks worker processes, resolve the loader in each worker. With `super-kernel/aop` installed, an application resolves it when it starts.

A class of the project, saved as `src/Billing/Invoice.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Billing;

final class Invoice
{
}
```

A script that takes over loading, saved as `bin/loader.php`:

```php
<?php

declare(strict_types=1);

use Src\Billing\Invoice;
use SuperKernel\Context\ApplicationContext;
use SuperKernel\Contract\AutoLoaderInterface;

require dirname(__DIR__) . '/vendor/autoload.php';

$loader = ApplicationContext::getContainer()->get(AutoLoaderInterface::class);

var_dump($loader->isRegistered(), count(spl_autoload_functions()));

$loader->addInterceptor(static function (string $class, string $file): bool {
    echo 'Loading ', $class, ' from ', basename($file), PHP_EOL;

    return false;
});

new Invoice();

var_dump(isset($loader->getClassMap()[Invoice::class]));
```

Build the project and run the script:

```bash
skernel build
php bin/loader.php
```

It prints:

```text
bool(true)
int(1)
Loading Src\Billing\Invoice from Invoice.php
bool(false)
```

### Read the pending class map

`getClassMap()` returns the mapped classes that are not loaded yet, from class name to file path.

- The map is fixed when the loader is created. Nothing can add or replace a mapping, and changing the array you passed in or the array you received does not change the loader.
- A class, interface, trait, or enum that was already declared when the loader was created is left out.
- After a file or an interceptor runs, every mapped class that is now declared leaves the map, including other classes the same file declares.
- Paths are kept as given; a relative path follows PHP's current working directory, so use absolute paths in long-running processes. The maps of `skernel build` are absolute.
- A mapped class whose file is missing is not loaded and stays in the map. No other loader is asked.
- A class that is not in the map is not loaded at all.

### Intercept loading

`addInterceptor($interceptor)` adds a callable that receives the requested class and its mapped file before the file is included, and returns its positive identifier. Interceptors only run for classes in the map, since only those have a file.

- Interceptors run in the order they were added. Returning `false` passes the class to the next interceptor and finally to the mapped file.
- Returning `true` stops the remaining interceptors and the mapped file. The interceptor must then declare the class itself; if it does not, the class stays in the pending map.
- `removeInterceptor($id)` removes an interceptor and returns `true`, or `false` for an unknown identifier.
- Interceptors belong to the loader instance; adding or removing one never changes the registered autoloaders.
- A class requested again while it is being loaded is not loaded a second time.

A class to intercept, saved as `src/Billing/Receipt.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Billing;

final class Receipt
{
}
```

A script with an interceptor that loads the class itself, saved as `bin/intercept.php`:

```php
<?php

declare(strict_types=1);

use Src\Billing\Receipt;
use SuperKernel\Context\ApplicationContext;
use SuperKernel\Contract\AutoLoaderInterface;

require dirname(__DIR__) . '/vendor/autoload.php';

$loader = ApplicationContext::getContainer()->get(AutoLoaderInterface::class);

$interceptorId = $loader->addInterceptor(static function (string $class, string $file): bool {
    if (Receipt::class !== $class) {
        return false;
    }

    echo 'Loading ', $class, ' myself', PHP_EOL;
    require $file;

    return true;
});

new Receipt();

var_dump($loader->removeInterceptor($interceptorId));
var_dump($loader->removeInterceptor($interceptorId));
```

After `skernel build`, `php bin/intercept.php` prints:

```text
Loading Src\Billing\Receipt myself
bool(true)
bool(false)
```

### Remove the loader safely

A long-running runtime may load every class before it starts serving and then remove the loader. `unregister()`:

- raises `SuperKernel\Autoloader\Exception\UnloadedClassMapException` while any mapped class is not loaded, and changes nothing. `classMap()` of the exception returns the pending mappings and `firstClassName()` the first of them, which the message names;
- otherwise removes this loader only and clears its interceptors. Loaders that other code registered after the takeover stay; the loaders removed by the takeover do not come back, so the autoloader stack may be empty afterwards.

`isRegistered()` reports whether this loader is currently registered.

A script that tries to remove the loader too early, saved as `bin/unregister.php`:

```php
<?php

declare(strict_types=1);

use SuperKernel\Autoloader\Exception\UnloadedClassMapException;
use SuperKernel\Context\ApplicationContext;
use SuperKernel\Contract\AutoLoaderInterface;

require dirname(__DIR__) . '/vendor/autoload.php';

$loader = ApplicationContext::getContainer()->get(AutoLoaderInterface::class);

try {
    $loader->unregister();
} catch (UnloadedClassMapException $exception) {
    echo $exception->getMessage(), PHP_EOL;
    var_dump($exception->firstClassName() === array_key_first($loader->getClassMap()));
}

var_dump($loader->isRegistered());
```

After `skernel build`, `php bin/unregister.php` prints the first pending class of the project:

```text
Class "Src\Billing\Invoice" has not been loaded.
bool(true)
bool(true)
```

### Create a loader yourself

`new SuperKernel\Autoloader\AutoLoader($classMap)` checks and prepares a class map without registering anything; only the provider takes over. The loader is callable, so it loads a class when invoked with its name, and it can be registered with `spl_autoload_register()` like any loader. This is useful in tests.

A script that loads one class through its own loader, saved as `bin/standalone.php`:

```php
<?php

declare(strict_types=1);

use Src\Billing\Receipt;
use SuperKernel\Autoloader\AutoLoader;

require dirname(__DIR__) . '/vendor/autoload.php';

$loader = new AutoLoader([Receipt::class => dirname(__DIR__) . '/src/Billing/Receipt.php']);

var_dump($loader->isRegistered());

$loader(Receipt::class);

var_dump(class_exists(Receipt::class, false));
var_dump($loader->getClassMap());

$loader->unregister();
```

`php bin/standalone.php` prints:

```text
bool(false)
bool(true)
array(0) {
}
```

## Errors

Every exception of the package implements `SuperKernel\Autoloader\Exception\ClassLoaderExceptionInterface`. Code typed against `AutoLoaderInterface` can catch the failures of `unregister()` as `RuntimeException`.

| Situation | Message |
| --- | --- |
| A class name in the map is not a non-empty string without null bytes (`InvalidClassMapException`) | `Classmap keys must be non-empty strings without null bytes; received <type>.` |
| A path in the map is empty or has a null byte (`InvalidClassMapException`) | `The classmap path for "<class>" must be a non-empty string without null bytes.` |
| `unregister()` while mapped classes are pending (`UnloadedClassMapException`) | `Class "<class>" has not been loaded.` |
| PHP refuses to remove the loader (`ClassLoaderRegistrationException`) | `Unable to unregister the class loader from SPL.` |

## Relation to other packages

- [`super-kernel/contract`](contract.md) declares `AutoLoaderInterface`; type your code against it.
- [`super-kernel/metadata`](metadata.md) restores the class maps of the packages from the metadata of `skernel build`.
- [`super-kernel/attribute`](attribute.md) declares the `#[FactoryProvider]` Attribute on the provider, and [`super-kernel/di`](di.md) resolves and keeps the loader.
- [`super-kernel/aop`](aop.md) resolves the loader when an application starts, reads its pending class map, and loads the classes it weaves before this loader includes them.
- `skernel build` generates the class maps; run it again after adding, removing, or moving a class.

## What not to do

**Do not expect another loader to load what the class map misses.** After the takeover no other autoloader is registered: a class added after the last `skernel build`, or one whose file was removed, is simply not found. Run `skernel build` after changing classes, or start with `skernel run`.

**Do not remove the loader while classes are pending.** `unregister()` refuses with `Class "<class>" has not been loaded.` and keeps the loader. Load every class the process will need first, or keep the loader registered.

**Do not share one loader between worker processes.** Resolve `AutoLoaderInterface` in each worker of a long-running runtime, so each process owns its loader and its pending class map.
