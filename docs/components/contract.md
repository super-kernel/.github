<p align="right">
  <strong>English</strong> | <a href="contract.zh-CN.md">中文文档</a>
</p>

# SuperKernel Contract

Package: [`super-kernel/contract`](https://github.com/super-kernel/contract) · [Packagist](https://packagist.org/packages/super-kernel/contract) · [Documentation index](../README.md)

## Purpose

`super-kernel/contract` is the set of interfaces through which SuperKernel packages work together without depending on each other. A package that needs configuration types its code against `ConfigInterface`, not against `super-kernel/config`; the container supplies whichever implementation the application installs. Applications use the same interfaces for the services they receive by injection.

The package contains interfaces only, in the `SuperKernel\Contract` namespace, and requires only PHP, the `reflection` extension, `psr/container`, and `psr/log`. Behavior, validation, and errors belong to the implementing packages.

## Capabilities

### Depend on a capability, not a package

Type constructor parameters and properties against the contract, and let the container supply the implementation. In an application created from `super-kernel/skeleton`, these packages implement the contracts:

| Contract | Implemented by |
| --- | --- |
| `ContainerInterface` | [`super-kernel/di`](di.md) |
| `ApplicationInterface` | [`super-kernel/framework`](framework.md) |
| `ConfigInterface` | [`super-kernel/config`](config.md) |
| `BasePathInterface`, `VendorDirInterface`, `PackageManagerInterface` | [`super-kernel/metadata`](metadata.md) |
| `AnnotationCollectorInterface` | [`super-kernel/annotator`](annotator.md) |
| `ReflectorInterface` | [`super-kernel/reflector`](reflector.md) |
| `AutoLoaderInterface` | [`super-kernel/autoloader`](autoloader.md) |
| `ExceptionDispatcherInterface` | [`super-kernel/exception-dispatcher`](exception-dispatcher.md) |
| `StdoutLoggerInterface` | [`super-kernel/logger`](logger.md) |

The other interfaces describe values these services return, or are implemented by your own classes: `ListenerInterface` by listeners, `PropertyAttributeHandlerInterface` by property handlers, `SingleConfigInterface` and `MultipleConfigInterface` by configuration.

A service typed against `ReflectorInterface`, saved as `src/Report/ClassReport.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Report;

use ReflectionMethod;
use SuperKernel\Contract\ReflectorInterface;

use function count;

final readonly class ClassReport
{
    public function __construct(private ReflectorInterface $reflector)
    {
    }

    /**
     * @param class-string $class
     */
    public function describe(string $class): string
    {
        $reflection = $this->reflector->reflectClass($class);
        $methods = $reflection->getMethods(ReflectionMethod::IS_PUBLIC);

        return $reflection->getShortName() . ' has ' . count($methods) . ' public methods';
    }
}
```

A class to describe, saved as `src/Billing/Invoice.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Billing;

final class Invoice
{
    public function total(): int
    {
        return 0;
    }

    public function currency(): string
    {
        return 'EUR';
    }
}
```

A script that uses the service, saved as `bin/report.php`:

```php
<?php

declare(strict_types=1);

use Src\Billing\Invoice;
use Src\Report\ClassReport;
use SuperKernel\Context\ApplicationContext;

require dirname(__DIR__) . '/vendor/autoload.php';

echo ApplicationContext::getContainer()->get(ClassReport::class)->describe(Invoice::class), PHP_EOL;
```

Build the project and run the script:

```bash
skernel build
php bin/report.php
```

It prints `Invoice has 2 public methods`.

### Use the container

`ContainerInterface` extends `Psr\Container\ContainerInterface` and adds no method. It describes a container of shared services that live as long as the application: a class or interface identifier returns an instance of that type, and any other identifier may return any value. Objects that a caller creates for itself, such as a value for one request, do not belong in it; create them yourself or through a factory class. Accept `ContainerInterface` where code needs SuperKernel's container and `Psr\Container\ContainerInterface` where any PSR-11 container will do.

### Run the application

`ApplicationInterface` has one method, `run(): int`, which runs the application and returns its process status. How it runs and when the process ends belong to the implementation; `super-kernel/framework` supplies a Symfony Console application.

### Read configuration

`ConfigInterface` gives typed access to configuration declared with `#[Configuration]`:

- `has($contract, $name = null)` reports whether an item is declared, without loading anything. Without a name, it checks that the contract has a source, even one with several items and no default.
- `get($contract, $name = null)` returns the item, an instance of `$contract`, loading it at the first successful lookup and returning the same object afterwards. Without a name it returns the default item. A single source is its own item, also available under the name `default`; a source with several items exposes each under its method name, and a sole item is the default automatically. It returns `null` for an undeclared contract or name, and raises a `Psr\Container\ContainerExceptionInterface` when loading fails or several items have no default. A failed load may be retried.

`SingleConfigInterface` marks a configuration object that is itself the default item; configuration interfaces of packages extend it. `MultipleConfigInterface` marks a source whose methods provide named items. Neither has methods. The rules are on the page of [`super-kernel/config`](config.md).

### Locate the project and its packages

- `BasePathInterface` and `VendorDirInterface` are `Stringable`: cast them to a string for the project directory, or the `phar://` root of a packaged application, and for the Composer vendor directory, which need not exist.
- `PackageManagerInterface` lists the root first and then the installed packages by name, finds a package by its name in any case, by its exact Composer type, or by one of its classes, and returns the class maps and the environment requirements.
- `PackageMetadataInterface` is one package; `version()` returns the version exactly as declared, or `null` when the root declares none, without normalizing or comparing it. `EnvironmentRequirementInterface` is one requirement on a Composer platform package, the package that declares it, and whether it is a development requirement of the root.

The values come from the metadata of `skernel build`; see [`super-kernel/metadata`](metadata.md).

### Read collected Attributes

`AnnotationCollectorInterface` finds the classes and methods that carry an Attribute, by its exact class or by a parent class or interface, and returns the Attributes declared on one class, method, or property. Every result is an `AttributeMetadataInterface`: `attribute()` returns the instance constructed at build time, `name()` its class, and `types()` its class followed by its captured parent types. `PackageAnnotationsInterface` holds one package's name, reference, and class, method, and property maps.

`ScanDiagnosticsInterface` is a separate capability with one method, `getDiagnostics()`, which returns the classes that `skernel build` could not collect. It is not a container entry of its own: check whether a collector or a package implements it. Reading a row never loads the class it names, and the rows hold no exception objects.

You may implement these interfaces yourself, for example in tests; `super-kernel/annotator` publishes only its own models. The queries are described on the page of [`super-kernel/annotator`](annotator.md).

### Handle a custom property Attribute

A handler implements `PropertyAttributeHandlerInterface`: its constructor takes exactly the `ContainerInterface`, and `handle($context, $attribute)` processes one Attribute on one property. The `PropertyContextInterface` it receives names the class and the reflected property and accepts one assignment: `assign($value)` sets the value, a second `assign()` raises `LogicException`, and not assigning leaves the property unchanged. `hasAssignment()` and `assignment()` read the assignment; `assignment()` raises `LogicException` when there is none. A complete example is on the DI page: [Custom property Attributes](di.md#custom-property-attributes).

### Publish and receive events

- `EventInterface` marks the events of SuperKernel packages. It has no methods and prescribes no payload; [`super-kernel/event`](event.md) declares the lifecycle events.
- `ListenerInterface` is implemented by a listener: `listen()` returns the event classes it handles, and `process($event)` receives each matching event. A listener of several event types distinguishes them inside `process()`. [`super-kernel/event-dispatcher`](event-dispatcher.md) registers and calls listeners.

### Report exceptions

`ExceptionDispatcherInterface::dispatch($throwable)` passes a throwable to the exception handlers that match it and reports it when none handles it. See [`super-kernel/exception-dispatcher`](exception-dispatcher.md).

### Reflect, load classes, and log

- `ReflectorInterface` returns PHP's native `Reflection*` objects for classes, methods, properties, constants, enums, functions, parameters, extensions, generators, fibers, and references, reused between calls. Failures are PHP's own `ReflectionException`. See [`super-kernel/reflector`](reflector.md).
- `AutoLoaderInterface` is the class loader of the process: its pending class map, its interceptors, and its removal, whose failures are `RuntimeException`. See [`super-kernel/autoloader`](autoloader.md).
- `StdoutLoggerInterface` extends `Psr\Log\LoggerInterface` without adding methods. It names the logger that SuperKernel packages write their records to; the default writes to standard output, and an application may replace its destination, format, and minimum level. See [`super-kernel/logger`](logger.md).

## Relation to other packages

- Every SuperKernel package that offers or uses a shared service depends on this package; this package depends on no other SuperKernel package.
- The implementing packages are listed in [Depend on a capability, not a package](#depend-on-a-capability-not-a-package).
- [`super-kernel/attribute`](attribute.md) declares the Attributes that classes implementing these interfaces carry, such as `#[Listener]` on a `ListenerInterface`.
- [`super-kernel/context`](context.md) keeps the root container of the process, which `super-kernel/di` creates as a `ContainerInterface`.
