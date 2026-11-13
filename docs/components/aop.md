<p align="right">
  <strong>English</strong> | <a href="aop.zh-CN.md">中文文档</a>
</p>

# SuperKernel AOP

Package: [`super-kernel/aop`](https://github.com/super-kernel/aop) · [Packagist](https://packagist.org/packages/super-kernel/aop) · [Documentation index](../README.md)

## Purpose

`super-kernel/aop` lets code run around the methods of other classes without editing them: logging, timing, caching, or checks that belong to many calls. An aspect is a class that declares its targets with `#[Aspect]` and implements `AspectInterface`. When the application has started and a target class is loaded for the first time, the package loads a transformed copy of the class instead, in which every method passes its call to the aspect.

The transformed source is prepared and cached by `super-kernel/mimic`; the class map comes from `SuperKernel\Contract\AutoLoaderInterface`, which the package needs as soon as one aspect exists. `super-kernel/autoloader` provides it; the package suggests that package but does not require it, so install it with `composer require super-kernel/autoloader`. Weaving starts with the application of `super-kernel/framework`, and `skernel build` prepares the transformed classes in advance.

## Capabilities

### Declare an aspect

An aspect is a class that carries `SuperKernel\Aop\Attribute\Aspect` and implements `SuperKernel\Aop\Contract\AspectInterface`. The container creates it, so it receives its services through its constructor. `process()` receives a `SuperKernel\Aop\Contract\ProceedingJoinPointInterface` for each selected call and returns the call's result: calling `proceed()` runs the original method.

A service, saved as `src/Billing/PriceCalculator.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Billing;

final class PriceCalculator
{
    public function total(int $net, int $taxPercent): int
    {
        return $net + intdiv($net * $taxPercent, 100);
    }
}
```

An aspect of its `total()` method, saved as `src/Aspect/LogCalculation.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Aspect;

use Src\Billing\PriceCalculator;
use SuperKernel\Aop\Attribute\Aspect;
use SuperKernel\Aop\Contract\AspectInterface;
use SuperKernel\Aop\Contract\ProceedingJoinPointInterface;

#[Aspect(methods: [PriceCalculator::class => ['total']])]
final readonly class LogCalculation implements AspectInterface
{
    public function process(ProceedingJoinPointInterface $point): mixed
    {
        $arguments = implode(', ', $point->getArguments());
        $result = $point->proceed();

        echo "{$point->getShortMethodName()}({$arguments}) = {$result}", PHP_EOL;

        return $result;
    }
}
```

A command that uses the service, saved as `src/Command/TotalCommand.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Command;

use Src\Billing\PriceCalculator;
use SuperKernel\Command\Attribute\Command;
use Symfony\Component\Console\Command\Command as SymfonyCommand;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

#[Command(name: 'invoice:total', description: 'Print an invoice total.')]
final class TotalCommand extends SymfonyCommand
{
    public function __construct(private readonly PriceCalculator $calculator)
    {
        parent::__construct();
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $output->writeln("Total: {$this->calculator->total(1000, 19)}");

        return self::SUCCESS;
    }
}
```

Build the project and run the command:

```bash
skernel build
php bin/bootstrap.php invoice:total
```

It prints:

```text
total(1000, 19) = 1190
Total: 1190
```

### Select targets

`#[Aspect]` is a class-only, non-repeatable Attribute; one declaration lists any number of targets in three arguments:

- `classes`: a list of class names; every method of each class is selected;
- `methods`: class names mapped to non-empty lists of method names;
- `propertyHooks`: class names mapped to property names, without `$`, each mapped to a non-empty list of `get` and `set`; accepted, but not woven yet, see [What not to do](#what-not-to-do).

A class name may start with a namespace separator. Wildcards and selector strings such as `Class::method` are not accepted. Names, their order, and duplicates are kept as written. The Attribute checks the syntax of names and the shape of the lists, and raises `InvalidArgumentException` for a malformed declaration; it never loads a target or checks that a member exists, so a declaration can name classes that are loaded much later. Targets are matched by name, ignoring case and a leading namespace separator, and only the named class is selected: subclasses and implementations of an interface are not. An aspect is not its own target unless it names itself.

When the selected class is called, a method selector wins over a class selector of the same class, and when several aspects select the same method or class, the one found last by the build replaces the earlier ones.

A class with two methods, saved as `src/Billing/InvoiceNumbers.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Billing;

use function sprintf;

final class InvoiceNumbers
{
    private int $last = 1000;

    public function next(): string
    {
        return $this->format(++$this->last);
    }

    public function format(int $number): string
    {
        return sprintf('INV-%d', $number);
    }
}
```

An aspect of the whole class, saved as `src/Aspect/TraceInvoiceNumbers.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Aspect;

use Src\Billing\InvoiceNumbers;
use SuperKernel\Aop\Attribute\Aspect;
use SuperKernel\Aop\Contract\AspectInterface;
use SuperKernel\Aop\Contract\ProceedingJoinPointInterface;

#[Aspect(classes: [InvoiceNumbers::class])]
final readonly class TraceInvoiceNumbers implements AspectInterface
{
    public function process(ProceedingJoinPointInterface $point): mixed
    {
        echo "> {$point->getShortMethodName()}", PHP_EOL;

        try {
            return $point->proceed();
        } finally {
            echo "< {$point->getShortMethodName()}", PHP_EOL;
        }
    }
}
```

A script that starts the application and uses the class, saved as `bin/numbers.php`:

```php
<?php

declare(strict_types=1);

use Src\Billing\InvoiceNumbers;
use SuperKernel\Context\ApplicationContext;
use SuperKernel\Contract\ApplicationInterface;

require dirname(__DIR__) . '/vendor/autoload.php';

$container = ApplicationContext::getContainer();
$container->get(ApplicationInterface::class);

echo $container->get(InvoiceNumbers::class)->next(), PHP_EOL;
```

Build the project and run the script:

```bash
skernel build
php bin/numbers.php
```

It prints:

```text
> next
> format
< format
< next
INV-1001
```

The call from `next()` to `format()` passes through the aspect too, because every method of the woven class dispatches its calls.

### Write advice

The join point describes one call:

- `getInstance()` returns the object for an instance method, or the class name for a static method, which is the called class;
- `getArguments()` returns the arguments; `getClassReflection()` and `getMethodReflection()` reflect the original class and method; `getShortMethodName()` returns the method's name;
- `proceed()` runs the original method body and returns its result.

The advice decides the result: return what `proceed()` returned, change it, or return a value without proceeding to skip the original body. The method's declared return type is still enforced. The results of constructors, destructors, and `void` methods are discarded, and PHP has already assigned promoted constructor properties when the advice runs. Advice for a `never` method must not return normally. Put code that must also run when the original method throws in a `finally` block, as `TraceInvoiceNumbers` does.

### Weave a class yourself

`#[Aspect]` without arguments selects the annotated class itself. Such a class needs no `AspectInterface`: it is woven like a target, and its calls run their original bodies until a callback is registered for them. `SuperKernel\Aop\Runtime\Dispatcher::intercept()` registers one; it takes a class name, `Class::method`, or `[Class, method]`, and a callable that receives a `SuperKernel\Aop\Runtime\ProceedingJoinPoint` and returns the result. Selectors ignore case, a method selector wins over a class selector, and registering a selector again replaces its callback. Register callbacks before the methods are called.

A class that weaves itself, saved as `src/Billing/Clock.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Billing;

use SuperKernel\Aop\Attribute\Aspect;

use function date;

#[Aspect]
final class Clock
{
    public function today(): string
    {
        return date('Y-m-d');
    }
}
```

A script that fixes the date, saved as `bin/clock.php`:

```php
<?php

declare(strict_types=1);

use Src\Billing\Clock;
use SuperKernel\Aop\Runtime\Dispatcher;
use SuperKernel\Aop\Runtime\ProceedingJoinPoint;
use SuperKernel\Context\ApplicationContext;
use SuperKernel\Contract\ApplicationInterface;

require dirname(__DIR__) . '/vendor/autoload.php';

$container = ApplicationContext::getContainer();
$container->get(ApplicationInterface::class);

Dispatcher::intercept([Clock::class, 'today'], static fn(ProceedingJoinPoint $point): string => '2026-01-01');

echo $container->get(Clock::class)->today(), PHP_EOL;
```

Build the project and run the script:

```bash
skernel build
php bin/clock.php
```

It prints:

```text
2026-01-01
```

### Know when classes are woven

When the container creates `ApplicationInterface`, `super-kernel/framework` publishes `BeforeApplicationStart`. The package listens at priority `PHP_INT_MAX` and, when at least one aspect exists, resolves `AutoLoaderInterface`, which makes `super-kernel/autoloader` take over class loading, and puts its own loader in front of it. With no aspect, nothing is registered. Publishing the event again registers nothing twice and keeps the registered advice.

From then on, when PHP loads a class that is a target or weaves itself, that is not loaded yet, and that has a file in the class map, the loader:

1. prepares the transformed source through `super-kernel/mimic`;
2. creates, through the container, the aspects whose declarations select this class, and checks that each implements `AspectInterface`; other aspects are not created;
3. runs the transformed source, checks that it declared the class, and then registers the aspects' advice.

Any other class goes to the next loader. A failure in the first two steps leaves the class undeclared, so loading it can be tried again. Asking for a class that is already loaded changes nothing, including its advice. Advice is registered for the whole process: a long-running server shares it between requests and coroutines, and forked workers inherit the advice their parent registered before the fork.

When the container itself loads a target, for example to inject it, a failure while loading is reported as the container's `Unable to resolve method parameter <class>::<method>($<parameter>) of type <target>.`, without the package's message. Load the target directly, for example with `new`, after starting the application, to see the package's own exception.

### Prepare proxies at build

`skernel build` publishes `SuperKernel\Event\ApplicationBuild` in the project's own process. When at least one aspect exists, the package then prepares the transformed source of every target that has a file in the class map, without loading the target, creating aspects, or registering a loader. A packaged application, such as one built by `skernel package`, loads these proxies instead of transforming source at run time; a target without a prepared proxy is transformed into a `mimic://` stream when it is loaded. A parse or transformation failure fails the build.

The proxies are written to `vendor/.cache/mimic/<class name with dots>.tpl`, for example `Src.Billing.PriceCalculator.tpl`; the [`super-kernel/mimic`](mimic.md) page describes the cache. `SuperKernel\Aop\Mutation\Mutator::getCacheKey()` identifies the transformation, so a release that changes the generated code replaces the proxies. A class that a process has loaded cannot be replaced: restart long-running workers after a change.

### Write source that can be woven

Weave trusted, declaration-oriented files. Every concrete method of a selected class is transformed; abstract methods stay as they are, and `final` and `private` are kept. `SuperKernel\Aop\Mutation\Modifier` is an optional PHP-Parser visitor that removes `final` and makes `private` methods and properties `protected`; the default transformation does not use it.

Parameter values and promoted default objects are kept. `__FILE__` and `__DIR__` keep pointing to the original source, local or inside a Phar archive, also in class constants, property initializers, and parameter defaults: the generated code refers to constants that `SuperKernel\Aop\Runtime\SourceMetadata` binds from the class map before the class loads, named after the package and the package-relative path, so a proxy prepared before the project moved or was packaged still points to the source that runs. An integration that uses `Mutator` with `super-kernel/mimic` directly calls `SourceMetadata::register($class)` on its injected instance before it runs the prepared source.

The package refuses, when it prepares the class, methods it cannot preserve: methods that return by reference, reference or variadic parameters, hooked promoted parameters, generators, static local variables, and `__FUNCTION__`, `__METHOD__`, or `__LINE__` in a method body, including inside closures in the body.

An exception thrown by a transformed method keeps its object and reports the original file and the line of the last top-level statement entered in the method; a nested statement may report its enclosing statement. Exceptions from advice and other code keep their own locations. Stack frames and reflection still describe the generated proxy, and these diagnostic lines are not isolated between suspended fibers or Swoole coroutines. The package does not replace the global exception handler.

## Errors

| Situation | Message |
| --- | --- |
| `classes` is not a list | `Aspect classes must be a list of class names.` |
| A target is not a valid class name | `Aspect targets must be valid class names.` |
| A `methods` entry is not a non-empty list | `Aspect methods must be a non-empty list.` |
| A method name is not a valid member name | `Method targets must be valid member names.` |
| A `propertyHooks` entry does not map property names to hook lists | `Aspect property hooks must map property names to hook lists.` |
| A property name is not a valid member name | `Property targets must be valid member names.` |
| A hook list is empty or not a list | `Property hooks must be a non-empty list.` |
| A hook other than `get` or `set` | `Aspect property hooks must be get or set.` |
| An aspect with targets does not implement `AspectInterface` | `Aspect handler <class> must implement SuperKernel\Aop\Contract\AspectInterface.` |
| A target with a property hook declaration is loaded | `AOP property hook interception is not implemented.` |
| A method returns by reference | `AOP does not support methods returning by reference.` |
| A reference, variadic, or hooked promoted parameter | `AOP does not support reference, variadic, or hooked promoted parameters.` |
| A generator, static local, or `__FUNCTION__`, `__METHOD__`, or `__LINE__` | `AOP cannot preserve generators, static locals, or method/line magic constants.` |
| Loading a target needs the same target again; through the container it surfaces as an unresolvable parameter | `Recursive instrumentation of <class>.` |
| The class map's file does not declare the class | `The mapped source did not declare <class>.` |
| An aspect exists and no package provides `AutoLoaderInterface` | The container's `Unable to resolve method parameter SuperKernel\Aop\AutoLoader::__construct($autoLoader) of type SuperKernel\Contract\AutoLoaderInterface.` |
| An empty or malformed selector for `Dispatcher::intercept()` | `An interceptor selector must be a non-empty name.` or `A method selector must contain a class name and method name.` |

The declaration errors are `InvalidArgumentException`s raised while `skernel build` reads the Attribute, which skips the class and reports it, so the aspect does not exist. The others are raised when a class is prepared or loaded: `InvalidArgumentException`, `LogicException`, or `RuntimeException`. A missing `AutoLoaderInterface` provider fails `skernel build` and the application's start, before any class is woven; installing `super-kernel/autoloader` resolves it.

## Relation to other packages

- [`super-kernel/mimic`](mimic.md) prepares and caches the transformed source; this package provides its transformation.
- [`super-kernel/autoloader`](autoloader.md) provides `AutoLoaderInterface` and the class map; the package suggests it but does not require it, so install it with this package as soon as one aspect exists.
- [`super-kernel/framework`](framework.md) publishes `BeforeApplicationStart`, which starts weaving; [`super-kernel/event`](event.md) declares it and `ApplicationBuild`.
- [`super-kernel/di`](di.md) creates the aspects; [`super-kernel/annotator`](annotator.md) provides the recorded `#[Aspect]` declarations; [`super-kernel/metadata`](metadata.md) provides the package locations that name the source constants.
- `skernel build` collects the aspects and prepares the proxies; `skernel run` builds before it starts the application.

## What not to do

**Do not use a target before the application starts.** Weaving begins when the container creates `ApplicationInterface`. A script that only looks up services in the container, or code that loads a target earlier, gets the original class, and its aspects never run, without any error. Start the application first, as `bin/numbers.php` does, or run the code in a command.

**Do not weave methods the package cannot preserve.** A selected class with a generator, a static local, `__FUNCTION__`, `__METHOD__`, or `__LINE__`, a reference or variadic parameter, or a method that returns by reference fails `skernel build`, for example with `AOP cannot preserve generators, static locals, or method/line magic constants.` Move such methods to a class that no aspect selects, or narrow the aspect with `methods`.

**Do not declare property hooks yet.** The Attribute accepts `propertyHooks`, but loading such a target raises `AOP property hook interception is not implemented.`, so the class cannot be used at all. Select its methods instead.

**Do not let an aspect depend on its own target.** The aspect is created while its target is being loaded, so an aspect whose constructor needs the target, directly or through other services, cannot be created: loading the target fails with the container's `Unable to resolve method parameter <aspect>::__construct($<parameter>) of type <target>.` Use the instance that the join point's `getInstance()` returns instead.

**Do not expect two aspects of one method to both run.** For the same method or class selector, the aspect found last by the build replaces the earlier ones. Combine the advice in one aspect, or select different methods.

**Do not trust a malformed `#[Aspect]` to fail at run time.** `skernel build` reports the declaration error and skips the class, so the application starts without that aspect. Check the build's report after changing a declaration.
