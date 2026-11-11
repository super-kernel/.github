<p align="right">
  <strong>English</strong> | <a href="event.zh-CN.md">中文文档</a>
</p>

# SuperKernel Event

Package: [`super-kernel/event`](https://github.com/super-kernel/event) · [Packagist](https://packagist.org/packages/super-kernel/event) · [Documentation index](../README.md)

## Purpose

`super-kernel/event` declares the two moments of an application's life that SuperKernel announces to other code: the start of an application and the build of a project. Packages and applications listen to these events to configure the application before it runs, or to write files at build time that a packaged application loads instead of generating them.

The package contains the event classes only. Who publishes an event, when, and what happens when a listener fails belong to the publishing package; listener registration and dispatch belong to `super-kernel/event-dispatcher`.

## Capabilities

### Configure the application before it runs

`SuperKernel\Event\BeforeApplicationStart` is published by `super-kernel/framework` when its `ApplicationProvider` creates the application, the first time the container resolves `SuperKernel\Contract\ApplicationInterface`. By then the application has the root package's name and declared version and its Symfony event dispatcher. The event is published synchronously, before the provider returns the application; nothing calls `run()` until the application's caller does.

`application()` returns the application that was created. Its type is `SuperKernel\Contract\ApplicationInterface`; with `super-kernel/framework` the object is a `Symfony\Component\Console\Application`, so a listener checks for that class before it uses Symfony's setters. `super-kernel/command` installs its command loader this way.

In an application created from `super-kernel/skeleton`, this listener renames the console application. Save it as `src/Listener/NameTheApplication.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Listener;

use SuperKernel\Attribute\Listener;
use SuperKernel\Contract\ListenerInterface;
use SuperKernel\Event\BeforeApplicationStart;
use Symfony\Component\Console\Application;

#[Listener]
final readonly class NameTheApplication implements ListenerInterface
{
    public function listen(): array
    {
        return [
            BeforeApplicationStart::class,
        ];
    }

    public function process(object $event): void
    {
        assert($event instanceof BeforeApplicationStart);

        $application = $event->application();

        if ($application instanceof Application) {
            $application->setName('Billing');
        }
    }
}
```

Build the project and run the application:

```bash
skernel build
php bin/bootstrap.php --version
```

The output starts with `Billing`. A listener that throws stops the start: the container's lookup of `ApplicationInterface` fails with a `SuperKernel\Di\Exception\ResolutionException` whose previous exception is the listener's, and the application is neither returned nor run.

### Write build artifacts

`SuperKernel\Event\ApplicationBuild` is published by `skernel build`. After it generates the project's metadata, `skernel build` starts a second PHP process in the project, creates the container from that metadata, and publishes the event once through `Psr\EventDispatcher\EventDispatcherInterface`. It does not create or run the application, so a listener must not expect `ApplicationInterface` to have been created. The event carries no payload.

A listener writes the files that a packaged application, such as one built by `skernel package`, loads instead of generating them at run time. `super-kernel/aop` writes its prepared proxy classes this way. `skernel build` forwards what the process prints, and an exception from a listener ends the process with a non-zero status, which fails the build.

`skernel build` skips the event when the project does not install `super-kernel/event` or when its container has no `Psr\EventDispatcher\EventDispatcherInterface`.

This listener records when the project was built. Save it as `src/Listener/WriteBuildInfo.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Listener;

use SuperKernel\Attribute\Listener;
use SuperKernel\Contract\BasePathInterface;
use SuperKernel\Contract\ListenerInterface;
use SuperKernel\Event\ApplicationBuild;

use function date;
use function file_put_contents;

#[Listener]
final readonly class WriteBuildInfo implements ListenerInterface
{
    public function __construct(private BasePathInterface $basePath)
    {
    }

    public function listen(): array
    {
        return [
            ApplicationBuild::class,
        ];
    }

    public function process(object $event): void
    {
        file_put_contents($this->basePath . '/build-info.txt', 'Built at ' . date(DATE_ATOM) . PHP_EOL);

        echo 'Wrote build-info.txt', PHP_EOL;
    }
}
```

Run the build:

```bash
skernel build
```

The build prints `Wrote build-info.txt`, and `build-info.txt` in the project directory holds the build time.

### Listen to every SuperKernel event

Both events are final readonly classes that implement `SuperKernel\Contract\EventInterface`, a marker interface without methods. `super-kernel/event-dispatcher` matches a listener's `listen()` entries with `instanceof`, so a listener that returns `EventInterface::class` receives both events.

### Declare an application's own events

The package declares no other lifecycle boundary, such as after start or at stop. An application declares its own event classes for the moments it needs and publishes them through `Psr\EventDispatcher\EventDispatcherInterface`. `super-kernel/event-dispatcher` dispatches any object, so an application event needs no base class or interface.

The event, saved as `src/Billing/InvoicePaid.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Billing;

final readonly class InvoicePaid
{
    public function __construct(public string $invoiceNumber)
    {
    }
}
```

A listener, saved as `src/Billing/SendReceipt.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Billing;

use SuperKernel\Attribute\Listener;
use SuperKernel\Contract\ListenerInterface;

#[Listener]
final readonly class SendReceipt implements ListenerInterface
{
    public function listen(): array
    {
        return [
            InvoicePaid::class,
        ];
    }

    public function process(object $event): void
    {
        assert($event instanceof InvoicePaid);

        echo "Receipt sent for invoice {$event->invoiceNumber}", PHP_EOL;
    }
}
```

A service that publishes the event, saved as `src/Billing/PaymentService.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Billing;

use Psr\EventDispatcher\EventDispatcherInterface;

final readonly class PaymentService
{
    public function __construct(private EventDispatcherInterface $events)
    {
    }

    public function pay(string $invoiceNumber): void
    {
        $this->events->dispatch(new InvoicePaid($invoiceNumber));
    }
}
```

A script that uses the service, saved as `bin/pay.php`:

```php
<?php

declare(strict_types=1);

use Src\Billing\PaymentService;
use SuperKernel\Context\ApplicationContext;

require dirname(__DIR__) . '/vendor/autoload.php';

ApplicationContext::getContainer()->get(PaymentService::class)->pay('INV-1001');
```

Build the project and run the script:

```bash
skernel build
php bin/pay.php
```

It prints `Receipt sent for invoice INV-1001`.

## Relation to other packages

- [`super-kernel/contract`](contract.md) declares `EventInterface`, `ApplicationInterface`, and `ListenerInterface`; this package depends only on it.
- [`super-kernel/event-dispatcher`](event-dispatcher.md) discovers classes with the `#[Listener]` Attribute from [`super-kernel/attribute`](attribute.md) and dispatches events to them. Installing this package alone selects no dispatcher.
- [`super-kernel/framework`](framework.md) publishes `BeforeApplicationStart`. [`super-kernel/command`](command.md) and `super-kernel/framework` itself listen to it.
- `skernel build` publishes `ApplicationBuild`. [`super-kernel/aop`](aop.md) listens to it to prepare proxies.
- Listeners are found in the metadata that `skernel build` generates and [`super-kernel/di`](di.md) restores: run `skernel build` again after adding, removing, or changing a listener, or start with `skernel run`, which builds first.
