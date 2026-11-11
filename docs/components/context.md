<p align="right">
  <strong>English</strong> | <a href="context.zh-CN.md">中文文档</a>
</p>

# SuperKernel Context

Package: [`super-kernel/context`](https://github.com/super-kernel/context) · [Packagist](https://packagist.org/packages/super-kernel/context) · [Documentation index](../README.md)

## Purpose

`super-kernel/context` answers two separate questions of a running application. `ApplicationContext` answers "which container does this process use?": entry points, such as `bin/bootstrap.php` or a server callback, take the container from it. `Context` answers "which values belong to the work running right now?": infrastructure code hands short-lived values, such as a request identifier, to code further down the same coroutine without passing them through every call.

The package requires only PHP and `psr/container`. `super-kernel/di` and the Swoole extension are optional; with them, the container is created automatically and values are kept per coroutine and thread.

## Capabilities

### Keep values for the current execution

`SuperKernel\Context\Context` offers four static methods:

- `set($key, $value)` stores a value and returns it;
- `get($key, $default = null)` returns the value, or `$default` when the key is missing;
- `has($key)` reports whether the key exists, also when it holds `null`;
- `clear($key)` removes one key, and `clear()` removes every value of the current scope.

`Context` works without a container: it never accesses `ApplicationContext` or resolves a service.

Every call chooses the scope anew, in this order:

1. the current Swoole coroutine, when Swoole is loaded and a coroutine is running;
2. the current Swoole thread, when running in a Swoole thread;
3. the PHP process.

A scope that is not available is skipped, so the process is always the last choice. Native Fibers and Swow coroutines are not scopes of their own: inside them, values go to the Swoole coroutine or the process like any other code.

A script that uses the process scope, saved as `bin/context.php`:

```php
<?php

declare(strict_types=1);

use SuperKernel\Context\Context;

require dirname(__DIR__) . '/vendor/autoload.php';

Context::set('request-id', 'req-1');
Context::set('user', null);

var_dump(Context::get('request-id'));
var_dump(Context::has('user'), Context::get('locale', 'en'));

Context::clear();

var_dump(Context::has('request-id'));
```

Run the script:

```bash
php bin/context.php
```

It prints:

```text
string(5) "req-1"
bool(true)
string(2) "en"
bool(false)
```

### Separate values per coroutine and thread

With Swoole, each coroutine has its own values, kept in Swoole's native coroutine context and released when the coroutine ends. A child coroutine does not see its parent's values. Values in a Swoole thread are kept per thread for the life of the process. Values stored outside any coroutine or thread belong to the process and stay until you clear them.

A script with two coroutines, saved as `bin/coroutines.php`:

```php
<?php

declare(strict_types=1);

use Swoole\Coroutine;
use SuperKernel\Context\Context;

use function Swoole\Coroutine\run;

require dirname(__DIR__) . '/vendor/autoload.php';

Context::set('request-id', 'process');

run(static function (): void {
    foreach (['req-A', 'req-B'] as $requestId) {
        Coroutine::create(static function () use ($requestId): void {
            Context::set('request-id', $requestId);
            Coroutine::sleep(0.01);

            echo 'Coroutine sees ', Context::get('request-id'), PHP_EOL;
        });
    }
});

echo 'Process sees ', Context::get('request-id'), PHP_EOL;
```

With the Swoole extension loaded, `php bin/coroutines.php` prints:

```text
Coroutine sees req-A
Coroutine sees req-B
Process sees process
```

`SuperKernel\Context\Context\CoroutineContext`, `ThreadContext`, and `ProcessContext` are the three scopes, with the same static methods and a `support()` method that reports whether the scope is available. Use `Context`, which chooses for you; calling a scope directly where it is not available raises `LogicException`, such as `The Swoole coroutine context requires an active coroutine.`

### Reach the application container

`SuperKernel\Context\ApplicationContext::getContainer()` returns the process's container:

- When a container was set, it returns that container.
- Otherwise, when `super-kernel/di` is installed, it calls `SuperKernel\Di\Provider\ContainerProvider` once, which restores the metadata of `skernel build`, keeps the result as the process's container, and returns it.
- Otherwise it raises `SuperKernel\Context\Exception\ContainerNotSetException`.

When creating the container fails, it raises `SuperKernel\Context\Exception\ContainerInitializationException` with the cause as its previous exception, and keeps nothing: a later call tries again, and `setContainer()` can still set one.

The container is the process's, not the coroutine's. Each worker process of a server has its own.

A class of the project, saved as `src/Billing/Invoice.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Billing;

final class Invoice
{
}
```

A script that takes the container twice, saved as `bin/container.php`:

```php
<?php

declare(strict_types=1);

use Src\Billing\Invoice;
use SuperKernel\Context\ApplicationContext;

require dirname(__DIR__) . '/vendor/autoload.php';

$container = ApplicationContext::getContainer();

var_dump($container === ApplicationContext::getContainer());
var_dump($container->get(Invoice::class) === $container->get(Invoice::class));
```

Build the project and run the script:

```bash
skernel build
php bin/container.php
```

It prints:

```text
bool(true)
bool(true)
```

### Use another container

`ApplicationContext::setContainer($container)` sets any `Psr\Container\ContainerInterface` as the process's container, replacing the previous one, and returns it. Set it during start-up, before any code asks for the container. A container you set must return, for a class or interface identifier, an instance of that type, as SuperKernel packages rely on it.

A script with a container of its own, saved as `bin/own-container.php`:

```php
<?php

declare(strict_types=1);

use Psr\Container\ContainerInterface;
use Psr\Container\NotFoundExceptionInterface;
use SuperKernel\Context\ApplicationContext;

require dirname(__DIR__) . '/vendor/autoload.php';

$container = new class implements ContainerInterface {
    public function get(string $id): mixed
    {
        return 'app.name' === $id
            ? 'Billing'
            : throw new class("No entry {$id}.") extends RuntimeException implements NotFoundExceptionInterface {};
    }

    public function has(string $id): bool
    {
        return 'app.name' === $id;
    }
};

var_dump(ApplicationContext::setContainer($container) === $container);
echo ApplicationContext::getContainer()->get('app.name'), PHP_EOL;
```

`php bin/own-container.php` prints:

```text
bool(true)
Billing
```

## Errors

| Situation | Message |
| --- | --- |
| No container was set and `super-kernel/di` is not installed (`ContainerNotSetException`) | `The application context has no container.` |
| Creating the default container failed (`ContainerInitializationException`, cause as previous) | `Unable to initialize the application container through SuperKernel\Di\Provider\ContainerProvider.` |
| `CoroutineContext` used outside a coroutine (`LogicException`) | `The Swoole coroutine context requires an active coroutine.` |
| `CoroutineContext` used without Swoole (`LogicException`) | `The Swoole coroutine context requires the Swoole extension.` |
| `ThreadContext` used outside a Swoole thread (`LogicException`) | `The Swoole thread context requires an active Swoole thread.` |

## Relation to other packages

- [`super-kernel/di`](di.md) provides `ContainerProvider`, which `getContainer()` calls to create the default container from the metadata of `skernel build`.
- The skeleton's `bin/bootstrap.php` takes the container from `ApplicationContext` and runs the application of [`super-kernel/framework`](framework.md).
- [`super-kernel/support`](support.md) uses the container of `ApplicationContext` in `make()`.
- [`super-kernel/swoole`](swoole.md) and the server packages run the coroutines and threads whose values `Context` separates.

## What not to do

**Do not look services up in `ApplicationContext` inside your services.** A class that calls `ApplicationContext::getContainer()->get()` hides its dependencies and cannot be created with others in a test. Receive dependencies through the constructor or `#[Autowired]`; call `getContainer()` only at an entry point, such as a script or a server callback.

**Do not create a second root container.** Calling `new ContainerProvider()()` next to `ApplicationContext` restores another container with its own services, which the rest of the application does not share. Use `ApplicationContext::getContainer()`.

**Do not replace the container after start-up.** `setContainer()` changes the container for later callers only; services already created keep what they received from the old one. Set your container before anything asks for it.

**Do not leave values in the process scope of a long-running server.** Outside a coroutine, values stay for the life of the process, so a value stored while handling one request is still there for the next. Store request values inside the request's coroutine, or `clear()` them when the work ends.

**Do not expect a Fiber to have its own values.** A value that a Fiber stores lands in the surrounding coroutine or the process and replaces the value the code outside the Fiber stored. Run separate work in Swoole coroutines, or use distinct keys.

**Do not use `Context` in place of injection.** Values in `Context` are untyped and invisible in signatures. Keep them for short-lived values that infrastructure hands down within one execution, such as a request identifier, and pass services and configuration by injection.
