<p align="right">
  <strong>English</strong> | <a href="exception-dispatcher.zh-CN.md">中文文档</a>
</p>

# SuperKernel Exception Dispatcher

Package: [`super-kernel/exception-dispatcher`](https://github.com/super-kernel/exception-dispatcher) · [Packagist](https://packagist.org/packages/super-kernel/exception-dispatcher) · [Documentation index](../README.md)

## Purpose

`super-kernel/exception-dispatcher` decides who hears about an exception that reaches one of the application's boundaries. A boundary, such as a console command run by `super-kernel/framework`, passes the throwable to `SuperKernel\Contract\ExceptionDispatcherInterface`; the package calls every handler declared for its class and, when none of them completes, records the complete exception through the stdout logger. That logger is a provider of `SuperKernel\Contract\StdoutLoggerInterface`, such as `super-kernel/logger`, which the package suggests but does not require; install it with `composer require super-kernel/logger` unless the application provides its own.

The package only reports. What happens next, such as an exit status, an HTTP response, or a retry, belongs to the boundary that caught the exception. It works once the container is ready and handles nothing that fails before.

## Capabilities

### Declare a handler

A handler is a class that carries `SuperKernel\ExceptionDispatcher\Attribute\ExceptionHandler` and implements `SuperKernel\ExceptionDispatcher\Contract\ExceptionHandlerInterface`. The Attribute takes the class it handles, `exception`, `Throwable::class` by default, and a `priority`, `0` by default. A handler takes the declared class and every subclass. The container creates the handlers, so a handler receives its services through its constructor.

A handler, saved as `src/ExceptionHandler/ReportFailure.php`:

```php
<?php

declare(strict_types=1);

namespace Src\ExceptionHandler;

use RuntimeException;
use SuperKernel\ExceptionDispatcher\Attribute\ExceptionHandler;
use SuperKernel\ExceptionDispatcher\Contract\ExceptionHandlerInterface;
use Throwable;

#[ExceptionHandler(RuntimeException::class)]
final readonly class ReportFailure implements ExceptionHandlerInterface
{
    public function handle(Throwable $throwable): void
    {
        echo 'Failed: ', $throwable->getMessage(), PHP_EOL;
    }
}
```

A script that dispatches an exception, saved as `bin/report.php`:

```php
<?php

declare(strict_types=1);

use SuperKernel\Context\ApplicationContext;
use SuperKernel\Contract\ExceptionDispatcherInterface;

require dirname(__DIR__) . '/vendor/autoload.php';

$exceptions = ApplicationContext::getContainer()->get(ExceptionDispatcherInterface::class);

$exceptions->dispatch(new RuntimeException('The invoice feed is unavailable.'));
```

Build the project and run the script:

```bash
skernel build
php bin/report.php
```

It prints:

```text
Failed: The invoice feed is unavailable.
```

### Order handlers

`dispatch()` calls every matching handler, not only the first: higher priorities run first, and equal priorities keep the order in which the build found the handlers. There is no result that stops the others. A handler of `Throwable` sees every exception.

A handler of every throwable, saved as `src/ExceptionHandler/NotifyOperator.php`:

```php
<?php

declare(strict_types=1);

namespace Src\ExceptionHandler;

use SuperKernel\ExceptionDispatcher\Attribute\ExceptionHandler;
use SuperKernel\ExceptionDispatcher\Contract\ExceptionHandlerInterface;
use Throwable;

#[ExceptionHandler(priority: 10)]
final readonly class NotifyOperator implements ExceptionHandlerInterface
{
    public function handle(Throwable $throwable): void
    {
        echo 'Operator notified about ', $throwable::class, PHP_EOL;
    }
}
```

Build the project and run the script again:

```bash
skernel build
php bin/report.php
```

It prints:

```text
Operator notified about RuntimeException
Failed: The invoice feed is unavailable.
```

### Rely on the fallback record

When no matching handler completes, because none matches or every matching one throws, the dispatcher writes the original throwable once through `SuperKernel\Contract\StdoutLoggerInterface` at the error level. The record is PHP's complete string of the throwable, with its file, line, stack trace, and previous exceptions. With `super-kernel/logger`, a record looks like this:

```text
2026-11-10T10:36:46.370+00:00 [ERROR] RuntimeException: The invoice feed is unavailable. in /app/src/Command/ImportCommand.php:18
Stack trace:
#0 ...
```

A handler that throws does not stop the dispatch. Its failure is written through the same logger as `Exception handler <handler> failed while processing <exception class>: <failure>`, where `<failure>` is the handler's complete exception, and the next matching handler runs. The failure is never dispatched again. An exception from the logger itself propagates.

The application needs no configuration for this. To change the output, declare a handler of `Throwable`, or provide another `StdoutLoggerInterface` as described on the [`super-kernel/logger`](logger.md) page, which changes the threshold, format, and stream of every record the framework writes.

### Know where exceptions come from

The package catches nothing itself. `super-kernel/framework` dispatches the exceptions of console commands and the throwables that nothing catches after the application starts; the [`super-kernel/framework`](framework.md) page shows both. Server and protocol packages dispatch the exceptions of their own callbacks, or keep a handler family of their own, as the next section describes. Code of your own can dispatch a throwable it caught, as `bin/report.php` does, to report it the same way.

### Keep a handler family for another boundary

A boundary that needs handlers with another contract, such as an HTTP handler that returns a response, declares its own Attribute that extends `ExceptionHandler`. The generic dispatcher calls only handlers declared with exactly `#[ExceptionHandler]`, so the derived family stays with its boundary. `super-kernel/http-server` keeps its HTTP exception handlers this way.

The derived Attribute must declare PHP's own `#[Attribute(Attribute::TARGET_CLASS)]`, because PHP does not inherit it, and a handler carries exactly one Attribute of the family. The container provides `SuperKernel\ExceptionDispatcher\Registry\ExceptionHandlerProvider`: `getHandlersForAttribute()` returns `SuperKernel\ExceptionDispatcher\Registry\ExceptionHandlerRegistration`s of exactly one Attribute class, in priority order. Each registration holds the `declaration` and the created `handler`, and `matches()` applies the same class rule as the dispatcher. The boundary checks its own handler contract and decides what a handler's result means.

The Attribute, saved as `src/Queue/QueueExceptionHandler.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Queue;

use Attribute;
use SuperKernel\ExceptionDispatcher\Attribute\ExceptionHandler;

#[Attribute(Attribute::TARGET_CLASS)]
final readonly class QueueExceptionHandler extends ExceptionHandler
{
}
```

The handler contract, saved as `src/Queue/QueueFailureHandler.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Queue;

use Throwable;

interface QueueFailureHandler
{
    public function handle(Throwable $throwable, string $job): bool;
}
```

A handler, saved as `src/Queue/RetryLater.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Queue;

use RuntimeException;
use Throwable;

#[QueueExceptionHandler(RuntimeException::class)]
final readonly class RetryLater implements QueueFailureHandler
{
    public function handle(Throwable $throwable, string $job): bool
    {
        echo "Job {$job} will be retried: {$throwable->getMessage()}", PHP_EOL;

        return true;
    }
}
```

The worker that uses the family and falls back to the generic dispatcher, saved as `src/Queue/Worker.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Queue;

use SuperKernel\Contract\ExceptionDispatcherInterface;
use SuperKernel\ExceptionDispatcher\Registry\ExceptionHandlerProvider;
use Throwable;

final readonly class Worker
{
    public function __construct(
        private ExceptionHandlerProvider     $handlers,
        private ExceptionDispatcherInterface $exceptions,
    )
    {
    }

    public function fail(string $job, Throwable $throwable): void
    {
        foreach ($this->handlers->getHandlersForAttribute(QueueExceptionHandler::class) as $registration) {
            if (!$registration->matches($throwable)) {
                continue;
            }

            $handler = $registration->handler;
            assert($handler instanceof QueueFailureHandler);

            if ($handler->handle($throwable, $job)) {
                return;
            }
        }

        $this->exceptions->dispatch($throwable);
    }
}
```

A script that reports two failed jobs, saved as `bin/worker.php`:

```php
<?php

declare(strict_types=1);

use Src\Queue\Worker;
use SuperKernel\Context\ApplicationContext;

require dirname(__DIR__) . '/vendor/autoload.php';

$worker = ApplicationContext::getContainer()->get(Worker::class);

$worker->fail('send-invoice', new RuntimeException('The mail server is busy.'));
$worker->fail('send-invoice', new LogicException('The invoice has no recipient.'));
```

Build the project and run the script:

```bash
skernel build
php bin/worker.php
```

It prints:

```text
Job send-invoice will be retried: The mail server is busy.
Operator notified about LogicException
```

## Errors

The handlers are collected and created when the container first creates the handler registry, which happens with the dispatcher. An invalid declaration raises `SuperKernel\ExceptionDispatcher\Exception\InvalidExceptionHandlerDeclarationException`, which implements `SuperKernel\ExceptionDispatcher\Exception\ExceptionDispatcherExceptionInterface`, wrapped in a `SuperKernel\Di\Exception\ResolutionException`. `super-kernel/framework` needs the dispatcher for its startup listener, so with it installed the error also fails `skernel build` and the application start.

| Situation | Message |
| --- | --- |
| A `#[ExceptionHandler]` class does not implement `ExceptionHandlerInterface` | `Exception handler <class> marked with SuperKernel\ExceptionDispatcher\Attribute\ExceptionHandler must implement SuperKernel\ExceptionDispatcher\Contract\ExceptionHandlerInterface; <type> provided.` |
| A class carries more than one Attribute of the family | `<class> must declare exactly one Attribute derived from SuperKernel\ExceptionDispatcher\Attribute\ExceptionHandler; <count> found.` |
| A derived Attribute is repeatable or allowed on more than classes | `<attribute> on <class> must be a non-repeatable class-only Attribute.` |
| `exception` is not a `Throwable` class | `<attribute> on <class> must declare a Throwable class; <class> provided.` |

A handler that the container cannot create raises the container's own exception at the same moment. Without a `StdoutLoggerInterface` provider, the dispatcher itself cannot be created, which fails `skernel build` and the application's start in the same way with `Unable to resolve method parameter SuperKernel\ExceptionDispatcher\Provider\ExceptionDispatcherProvider::__invoke($logger) of type SuperKernel\Contract\StdoutLoggerInterface.` Install `super-kernel/logger`, or provide your own logger. A derived Attribute that does not declare PHP's own `#[Attribute]` cannot be read at all: `skernel build` skips each class that uses it and reports it with PHP's message `Attempting to use non-attribute class "<attribute>" as attribute`, and the handler is never registered.

## Relation to other packages

- [`super-kernel/contract`](contract.md) declares `ExceptionDispatcherInterface` and `StdoutLoggerInterface`; [`super-kernel/logger`](logger.md) provides the stdout logger that receives the fallback records; the package suggests it but does not require it.
- [`super-kernel/framework`](framework.md) dispatches console and uncaught exceptions; [`super-kernel/http-server`](http-server.md) keeps its own handler family derived from `ExceptionHandler`.
- [`super-kernel/di`](di.md) creates the handlers; [`super-kernel/annotator`](annotator.md) provides the recorded declarations, including derived Attributes.
- `skernel build` collects the handlers; `skernel run` builds before it starts the application.

## What not to do

**Do not throw from a handler to stop the others.** A handler's exception is logged as a handler failure, and the remaining handlers still run. Every matching handler runs; give a handler a narrower `exception` class if it must not see an exception.

**Do not derive a handler Attribute without its own `#[Attribute(Attribute::TARGET_CLASS)]`.** PHP does not inherit the declaration, so `skernel build` skips every class that uses the Attribute, reporting `Attempting to use non-attribute class "<attribute>" as attribute`, and those handlers are silently never called. A repeatable Attribute, or one allowed on other targets, is rejected with `<attribute> on <class> must be a non-repeatable class-only Attribute.` Declare exactly `#[Attribute(Attribute::TARGET_CLASS)]` on every derived Attribute.
