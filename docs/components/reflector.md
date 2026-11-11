<p align="right">
  <strong>English</strong> | <a href="reflector.zh-CN.md">中文文档</a>
</p>

# SuperKernel Reflector

Package: [`super-kernel/reflector`](https://github.com/super-kernel/reflector) · [Packagist](https://packagist.org/packages/super-kernel/reflector) · [Documentation index](../README.md)

## Purpose

`super-kernel/reflector` gives every package of an application one place to obtain PHP's reflection objects, and keeps them, so that the container, the configuration, and other code that inspect the same classes repeatedly do not reflect them again. The results are PHP's own `Reflection*` objects with their complete API; names, autoloading, and failures behave as in PHP.

The package implements `SuperKernel\Contract\ReflectorInterface`. Its cache belongs to each reflector instance; it does not persist between processes.

## Capabilities

### Create a reflector yourself

`new SuperKernel\Reflector\Reflector()` creates a reflector with an empty cache. Within one reflector, a repeated request for the same symbol returns the same object, whichever spelling PHP accepts for it: a class alias, a leading backslash, a different case of a class or method name, or the single `Class::method` form of `reflectMethod()`. Another reflector has its own cache and returns other objects.

A class of the project, saved as `src/Billing/Invoice.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Billing;

final class Invoice
{
    public function __construct(public readonly string $number)
    {
    }

    public function total(): int
    {
        return 0;
    }
}
```

A script that reflects one method in several spellings, saved as `bin/reflect.php`:

```php
<?php

declare(strict_types=1);

use Src\Billing\Invoice;
use SuperKernel\Reflector\Reflector;

require dirname(__DIR__) . '/vendor/autoload.php';

$reflector = new Reflector();
$method = $reflector->reflectMethod(Invoice::class, 'total');

echo $method->getReturnType(), PHP_EOL;
var_dump($method === $reflector->reflectMethod('Src\Billing\Invoice::total'));
var_dump($method === $reflector->reflectMethod('\src\billing\INVOICE', 'TOTAL'));
var_dump($method === new Reflector()->reflectMethod(Invoice::class, 'total'));
```

Build the project and run the script:

```bash
skernel build
php bin/reflect.php
```

It prints:

```text
int
bool(true)
bool(true)
bool(false)
```

### Reflect any kind of symbol

| Method | Returns |
| --- | --- |
| `reflectClass($class)` | `ReflectionClass` of a class, interface, trait, or enum, by name or object |
| `reflectClassConstant($class, $constant)` | `ReflectionClassConstant` |
| `reflectConstant($name)` | `ReflectionConstant` of a global constant |
| `reflectEnum($enum)` | `ReflectionEnum`, by name or case |
| `reflectEnumUnitCase($enum, $case)` | `ReflectionEnumUnitCase`, or `ReflectionEnumBackedCase` for a backed enum |
| `reflectEnumBackedCase($enum, $case)` | `ReflectionEnumBackedCase` |
| `reflectExtension($name)` | `ReflectionExtension` of a loaded PHP extension |
| `reflectZendExtension($name)` | `ReflectionZendExtension` of a loaded Zend extension |
| `reflectFunction($function)` | `ReflectionFunction` of a function name or closure |
| `reflectMethod($class, $method)` | `ReflectionMethod`; also `reflectMethod('Class::method')` |
| `reflectParameter($callable, $parameter)` | `ReflectionParameter` by position or name, for a closure, invokable object, function or method name, or `[class, method]` pair |
| `reflectObject($object)` | `ReflectionObject` |
| `reflectProperty($class, $property)` | `ReflectionProperty` |
| `reflectGenerator($generator)` | `ReflectionGenerator` |
| `reflectFiber($fiber)` | `ReflectionFiber` |
| `reflectReference($array, $key)` | `ReflectionReference`, or `null` when the element is not a reference |

`ReflectionAttribute` objects and the `ReflectionType` subclasses come from the reflections that own them, such as `getAttributes()` or `getType()`, because PHP offers no way to create them directly.

An enum, saved as `src/Billing/Status.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Billing;

enum Status: string
{
    case Open = 'open';
    case Paid = 'paid';
}
```

A script that reflects several kinds of symbols, saved as `bin/kinds.php`:

```php
<?php

declare(strict_types=1);

use Src\Billing\Invoice;
use Src\Billing\Status;
use SuperKernel\Reflector\Reflector;

require dirname(__DIR__) . '/vendor/autoload.php';

$reflector = new Reflector();

echo $reflector->reflectEnumUnitCase(Status::class, 'Paid')::class, PHP_EOL;
echo $reflector->reflectEnumBackedCase(Status::Open, 'Open')->getBackingValue(), PHP_EOL;
echo $reflector->reflectProperty(Invoice::class, 'number')->getType(), PHP_EOL;
echo $reflector->reflectParameter([Invoice::class, '__construct'], 'number')->getPosition(), PHP_EOL;
echo $reflector->reflectFunction('strlen')->getNumberOfParameters(), PHP_EOL;

$values = ['plain' => 1];
$values['shared'] = &$values['plain'];

var_dump($reflector->reflectReference($values, 'shared') !== null);
var_dump($reflector->reflectReference(['plain' => 1], 'plain'));
```

After `skernel build`, `php bin/kinds.php` prints:

```text
ReflectionEnumBackedCase
open
string
0
1
bool(true)
NULL
```

### Rely on the cache

- Names follow PHP: class, method, function, and extension names match without regard to case, and a class alias reaches the reflection of the class it names. Property, constant, parameter, and enum case names match exactly.
- Reflections of closures, invokable objects, objects, generators, and fibers are kept under weak keys: they do not keep those objects alive, and they disappear with them.
- A reference is kept by `ReflectionReference::getId()`. An element that is not a reference returns `null` and leaves nothing in the cache.
- The cache lives as long as its reflector. A container or the facade keeps its reflector, and so its cache, for the life of the process. PHP cannot redeclare a class or function within a process, so a reflection never goes stale; across processes nothing is kept.
- A name PHP would reject is rejected here too, with PHP's `ReflectionException`; it is never turned into a cache hit.

### Receive the reflector from the container

`SuperKernel\Reflector\Provider\ReflectorProvider` carries `#[FactoryProvider(ReflectorInterface::class)]`. Each call creates a new `Reflector`, makes it the facade's reflector, and returns it. `super-kernel/di` calls it once when it creates the root container and keeps the result under `ReflectorInterface` and `Reflector`, so services share one cache. Take `ReflectorInterface` as a constructor parameter.

A service that receives the reflector, saved as `src/Billing/InvoiceInspector.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Billing;

use ReflectionMethod;
use SuperKernel\Contract\ReflectorInterface;

final readonly class InvoiceInspector
{
    public function __construct(private ReflectorInterface $reflector)
    {
    }

    public function totalMethod(): ReflectionMethod
    {
        return $this->reflector->reflectMethod(Invoice::class, 'total');
    }
}
```

A script that compares it with the facade, saved as `bin/inspect.php`:

```php
<?php

declare(strict_types=1);

use Src\Billing\Invoice;
use Src\Billing\InvoiceInspector;
use SuperKernel\Context\ApplicationContext;
use SuperKernel\Reflector\Facade\ReflectorFacade;

require dirname(__DIR__) . '/vendor/autoload.php';

$method = ApplicationContext::getContainer()->get(InvoiceInspector::class)->totalMethod();

echo $method->class, '::', $method->getName(), PHP_EOL;
var_dump($method === ReflectorFacade::reflectMethod(Invoice::class, 'total'));
```

After `skernel build`, `php bin/inspect.php` prints:

```text
Src\Billing\Invoice::total
bool(true)
```

### Use the static facade

`SuperKernel\Reflector\Facade\ReflectorFacade` offers every method of `ReflectorInterface` as a static call, with positional or named arguments, and forwards it to the reflector of the most recent `ReflectorProvider` call, sharing that reflector's cache. Results and failures pass through unchanged.

- Before `ReflectorProvider` has run in the process, every call raises `LogicException`. The facade has no setter; only the provider sets its reflector. In an application, the root container's creation runs the provider.
- Each provider call replaces the facade's reflector; reflectors created before keep their own caches.
- A method that `ReflectorInterface` does not declare fails as an undefined method of the reflector.

Prefer injection; use the facade only where code cannot receive a reflector, such as a static helper.

A script that uses the facade without a container, saved as `bin/facade.php`:

```php
<?php

declare(strict_types=1);

use Src\Billing\Invoice;
use SuperKernel\Reflector\Facade\ReflectorFacade;
use SuperKernel\Reflector\Provider\ReflectorProvider;

require dirname(__DIR__) . '/vendor/autoload.php';

try {
    ReflectorFacade::reflectClass(Invoice::class);
} catch (LogicException $exception) {
    echo $exception->getMessage(), PHP_EOL;
}

$reflector = new ReflectorProvider()();

var_dump(ReflectorFacade::reflectClass(Invoice::class) === $reflector->reflectClass(Invoice::class));
```

After `skernel build`, `php bin/facade.php` prints:

```text
ReflectorFacade requires initialization through ReflectorProvider.
bool(true)
```

## Errors

| Situation | Message |
| --- | --- |
| The facade is called before `ReflectorProvider` ran (`LogicException`) | `ReflectorFacade requires initialization through ReflectorProvider.` |
| A symbol does not exist (`ReflectionException`, PHP's own) | PHP's message, such as `Class "<class>" does not exist` |

## Relation to other packages

- [`super-kernel/contract`](contract.md) declares `ReflectorInterface`; type your code against it.
- [`super-kernel/attribute`](attribute.md) declares the `#[FactoryProvider]` Attribute on the provider.
- [`super-kernel/di`](di.md) calls the provider when it creates the root container, keeps the reflector, and uses it to inspect constructors, factories, and properties.
- Other packages, such as [`super-kernel/config`](config.md), receive the same reflector from the container.

## What not to do

**Do not call the facade before the provider has run.** It raises `ReflectorFacade requires initialization through ReflectorProvider.` In an application, obtain the container first, or receive `ReflectorInterface` by injection.

**Do not create a reflector per call.** Each `new Reflector()` starts with an empty cache, so the reflections are built again and objects that should be identical are not. Receive the container's reflector, or keep one instance.

**Do not call `ReflectorProvider` again to get the shared reflector.** Each call creates a new reflector with an empty cache and points the facade at it, away from the one the container keeps. Take `ReflectorInterface` from the container.
