<p align="right">
  <strong>English</strong> | <a href="support.zh-CN.md">中文文档</a>
</p>

# SuperKernel Support

Package: [`super-kernel/support`](https://github.com/super-kernel/support) · [Packagist](https://packagist.org/packages/super-kernel/support) · [Documentation index](../README.md)

## Purpose

`super-kernel/support` holds convenience functions that SuperKernel packages and applications share. Its one function, `SuperKernel\Support\make()`, creates an object that the caller owns, such as a value for one request or one job, while its dependencies still come from the application container. The container itself keeps everything it creates; `make()` is the way to construct with injection without keeping the result.

The package requires only PHP. Composer loads the function with the package's autoloader; `super-kernel/context`, when installed, supplies the container.

## Capabilities

### Create objects the caller owns

`make(string $className, array $parameters = []): mixed` creates a new instance of `$className` with `new` on every call and returns it. The instance is neither taken from nor stored in the container. Its constructor receives:

- the values in `$parameters`, by parameter name or by position;
- for every other parameter, the container's entry for the class or interface in the parameter's type;
- otherwise the parameter's default value, then `null` when the type allows it.

Dependencies from the container are its shared services, so two objects made one after another share them. `make()` does nothing else: no `#[Provider]` or `#[FactoryProvider]` mapping of the requested class, no property injection, no lifecycle hooks, and no scope for the dependencies. It reflects the constructor again on every call and keeps no cache. For static analysis, the return type is the requested class whenever `$className` is a class string.

A service the container provides, saved as `src/Report/Formatter.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Report;

use function strtoupper;

final class Formatter
{
    public function format(string $format, string $title): string
    {
        return strtoupper($format) . ': ' . $title;
    }
}
```

An object the caller owns, saved as `src/Report/Report.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Report;

final readonly class Report
{
    public function __construct(public Formatter $formatter, public string $format = 'text')
    {
    }

    public function render(string $title): string
    {
        return $this->formatter->format($this->format, $title);
    }
}
```

A script that creates two reports, saved as `bin/report.php`:

```php
<?php

declare(strict_types=1);

use Src\Report\Report;

use function SuperKernel\Support\make;

require dirname(__DIR__) . '/vendor/autoload.php';

$json = make(Report::class, ['format' => 'json']);
$text = make(Report::class);

echo $json->render('Sales'), PHP_EOL;
echo $text->render('Sales'), PHP_EOL;
var_dump($json === $text, $json->formatter === $text->formatter);
```

Build the project and run the script:

```bash
skernel build
php bin/report.php
```

It prints:

```text
JSON: Sales
TEXT: Sales
bool(false)
bool(true)
```

### Know how parameters are filled

With a container, `make()` fills the constructor exactly as `super-kernel/di` fills a constructor, and asks the container only `has()` and `get()`:

1. A class without a constructor receives no arguments, whatever `$parameters` holds.
2. Variadic parameters are skipped, also when a value for them is given. A by-reference parameter is an error.
3. A value given by name wins over a value at the parameter's position. A given `null` counts as a value; PHP checks its type when the object is created.
4. The class, interface, and enum names in the parameter's type are the candidates. `self` means the requested class and `parent` its direct parent, also in a constructor the class inherits. A union contributes its named branches in order, without duplicates; an intersection contributes nothing, so in `(A&B)|C` only `C` is a candidate. Built-in types such as `string` or `object` are never looked up, and neither is the parameter's name.
5. `has()` is asked for every candidate. Exactly one available candidate is fetched with `get()`; more than one is an error before any `get()`, even when they hold the same object, and a default does not resolve it.
6. With no available candidate, the default is used, then `null` when the type allows it; a required `mixed` parameter receives `null`, a required parameter without a type is an error.
7. Extra positional values and unknown names in `$parameters` are ignored.

A dependency must therefore be registered under the fully qualified name of a class or interface in the type. An entry registered as `mailer` is never found for a `MailerInterface` parameter; register the interface, or pass the value.

An interface, saved as `src/Storage/Store.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Storage;

interface Store
{
    public function name(): string;
}
```

One implementation, saved as `src/Storage/MemoryStore.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Storage;

final class MemoryStore implements Store
{
    public function name(): string
    {
        return 'memory';
    }
}
```

Another implementation, saved as `src/Storage/BackupStore.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Storage;

final class BackupStore implements Store
{
    public function name(): string
    {
        return 'backup';
    }
}
```

A job that needs a store, saved as `src/Storage/Job.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Storage;

final readonly class Job
{
    public function __construct(public Store $store, public string $label = 'default')
    {
    }
}
```

A class whose parameter accepts either implementation, saved as `src/Storage/Mirror.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Storage;

final readonly class Mirror
{
    public function __construct(public MemoryStore|BackupStore $target)
    {
    }
}
```

A minimal PSR-11 container, saved as `src/Storage/ArrayContainer.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Storage;

use Psr\Container\ContainerInterface;
use RuntimeException;

final readonly class ArrayContainer implements ContainerInterface
{
    /**
     * @param array<string, mixed> $entries
     */
    public function __construct(private array $entries)
    {
    }

    public function get(string $id): mixed
    {
        return $this->entries[$id] ?? throw new RuntimeException("No entry {$id}.");
    }

    public function has(string $id): bool
    {
        return isset($this->entries[$id]);
    }
}
```

A script that shows two failures and how a given value avoids one, saved as `bin/errors.php`:

```php
<?php

declare(strict_types=1);

use Src\Storage\ArrayContainer;
use Src\Storage\BackupStore;
use Src\Storage\Job;
use Src\Storage\MemoryStore;
use Src\Storage\Mirror;
use SuperKernel\Context\ApplicationContext;

use function SuperKernel\Support\make;

require dirname(__DIR__) . '/vendor/autoload.php';

ApplicationContext::setContainer(new ArrayContainer([
    MemoryStore::class => new MemoryStore(),
    BackupStore::class => new BackupStore(),
]));

foreach ([Job::class, Mirror::class] as $class) {
    try {
        make($class);
    } catch (InvalidArgumentException $exception) {
        echo $exception->getMessage(), PHP_EOL;
    }
}

echo make(Mirror::class, ['target' => new BackupStore()])->target->name(), PHP_EOL;
```

After `skernel build`, `php bin/errors.php` prints:

```text
Unable to resolve constructor parameter Src\Storage\Job::__construct($store) of type Src\Storage\Store.
Ambiguous constructor parameter Src\Storage\Mirror::__construct($target): Src\Storage\MemoryStore, Src\Storage\BackupStore.
backup
```

### Use another container

`make()` works with whatever PSR-11 container `SuperKernel\Context\ApplicationContext` holds; it needs no service of `super-kernel/di`. To use another container, create and configure it as that implementation requires, then pass it to `ApplicationContext::setContainer()` before the first `make()`. Without a container set, `ApplicationContext::getContainer()` creates the default one from `super-kernel/di` and the build metadata, or raises `SuperKernel\Context\Exception\ContainerNotSetException` when `super-kernel/di` is not installed.

A script that uses the container above, saved as `bin/third-party.php`:

```php
<?php

declare(strict_types=1);

use Src\Storage\ArrayContainer;
use Src\Storage\Job;
use Src\Storage\MemoryStore;
use Src\Storage\Store;
use SuperKernel\Context\ApplicationContext;

use function SuperKernel\Support\make;

require dirname(__DIR__) . '/vendor/autoload.php';

ApplicationContext::setContainer(new ArrayContainer([Store::class => new MemoryStore()]));

$first = make(Job::class, ['label' => 'first']);
$second = make(Job::class, ['label' => 'second']);

echo $first->label, ' and ', $second->label, ' use ', $first->store->name(), PHP_EOL;
var_dump($first->store === $second->store);
```

`php bin/third-party.php` prints:

```text
first and second use memory
bool(true)
```

### Use it without SuperKernel

When the class `SuperKernel\Context\ApplicationContext` does not exist, `make()` simply runs `new $className(...$parameters)` with PHP's own rules for arguments: the container rules above do not apply, so variadic values and extra keys reach the constructor as PHP passes them. The function file `src/functions.php` can be required on its own.

A script that loads only the function, saved as `bin/native.php`:

```php
<?php

declare(strict_types=1);

use Src\Report\Formatter;
use Src\Report\Report;

use function SuperKernel\Support\make;

require dirname(__DIR__) . '/vendor/super-kernel/support/src/functions.php';
require dirname(__DIR__) . '/src/Report/Formatter.php';
require dirname(__DIR__) . '/src/Report/Report.php';

echo make(Report::class, [new Formatter(), 'csv'])->render('Sales'), PHP_EOL;
```

`php bin/native.php` prints `CSV: Sales`.

## Errors

`make()` raises PHP's `InvalidArgumentException` for the failures it detects. It never catches or retries: exceptions of `super-kernel/context`, of the container's `has()` and `get()`, of reflection, and of the constructor pass through unchanged, and a failed lookup never falls back to a default, `null`, another candidate, or plain construction. Before reporting a parameter it cannot fill, `make()` asks the container's `get()` for each of the parameter's types once: the first failure other than not found, such as a class that fails to load, becomes the previous exception, because a container's `has()` reports such a failure only as `false`. An abstract class or a wrong argument type remains a PHP error.

| Situation | Message |
| --- | --- |
| No value, entry, default, or `null` for a parameter | `Unable to resolve constructor parameter <class>::__construct($<parameter>) of type <type>.`, without ` of type <type>` when the parameter has no type; a lookup failure of one of its types is the previous exception |
| More than one candidate has an entry | `Ambiguous constructor parameter <class>::__construct($<parameter>): <id>, <id>.` |
| A by-reference parameter | `Constructor parameter <class>::__construct($<parameter>) cannot be resolved because references are unsupported.` |

## Relation to other packages

- [`super-kernel/context`](context.md) supplies the container; `make()` uses it only when the package is installed.
- [`super-kernel/di`](di.md) applies the same argument rules to the constructors it calls; its own way to create an object it does not keep is described in [Resolve an object the caller owns](di.md#resolve-an-object-the-caller-owns). Where `super-kernel/di` raises `ResolutionException`, `make()` raises `InvalidArgumentException`.
- Other packages, such as [`super-kernel/config`](config.md) and [`super-kernel/http-server`](http-server.md), use `make()` to create objects that take arguments of their own besides injected dependencies.

## What not to do

**Do not use `make()` for shared services.** Every call creates another instance, which no other code sees. Receive shared services by injection or from the container.

**Do not register dependencies under names of your own.** `make()` looks up only the class and interface names in a parameter's type, so an entry under `mailer` stays unused and the parameter fails with `Unable to resolve constructor parameter ...`. Register the dependency under its interface, or pass it in `$parameters`.

**Do not leave two candidates of a union type in the container.** `make()` refuses to choose, with `Ambiguous constructor parameter ...`, even when both entries are the same object. Pass the value explicitly, or narrow the type.
