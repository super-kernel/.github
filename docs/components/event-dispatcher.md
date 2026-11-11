<p align="right">
  <strong>English</strong> | <a href="event-dispatcher.zh-CN.md">中文文档</a>
</p>

# SuperKernel Event Dispatcher

Package: [`super-kernel/event-dispatcher`](https://github.com/super-kernel/event-dispatcher) · [Packagist](https://packagist.org/packages/super-kernel/event-dispatcher) · [Documentation index](../README.md)

## Purpose

`super-kernel/event-dispatcher` lets code announce that something happened without knowing who reacts. Code publishes an event object through PSR-14's `EventDispatcherInterface`; the package calls every listener that declared the event's class, synchronously and in priority order, and hands the same object back.

The package implements PSR-14 only and does not depend on Symfony. Which events exist belongs to the code that publishes them, such as `super-kernel/event` for the lifecycle events; the container that creates the listeners belongs to `super-kernel/di`.

## Capabilities

### Declare a listener

A listener is a class that carries `SuperKernel\Attribute\Listener` and implements `SuperKernel\Contract\ListenerInterface`:

- `listen()` returns a list of the event classes or interfaces the listener receives; every name must exist;
- `process(object $event): void` handles one event; its return value is ignored.

The listener receives only events that match an entry of `listen()`, so a listener of one event handles it directly and may name the concrete type in PHPDoc. `#[Listener]` takes a `priority`, `0` by default: higher priorities run first, and equal priorities keep the order in which the build found the listeners.

The container creates the listeners, so a listener receives its services through its constructor. The annotated class does not need to be instantiable itself: an interface or abstract class works when the container provides an implementation of it, and the object the container returns must implement `ListenerInterface`.

The event, saved as `src/Account/UserRegistered.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Account;

final readonly class UserRegistered
{
    public function __construct(public string $email)
    {
    }
}
```

A listener with a higher priority, saved as `src/Account/RecordAuditEntry.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Account;

use SuperKernel\Attribute\Listener;
use SuperKernel\Contract\ListenerInterface;

#[Listener(priority: 100)]
final readonly class RecordAuditEntry implements ListenerInterface
{
    public function listen(): array
    {
        return [
            UserRegistered::class,
        ];
    }

    /**
     * @param UserRegistered $event
     */
    public function process(object $event): void
    {
        echo "Audit: {$event->email} registered", PHP_EOL;
    }
}
```

A listener with the default priority, saved as `src/Account/SendWelcomeMail.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Account;

use SuperKernel\Attribute\Listener;
use SuperKernel\Contract\ListenerInterface;

#[Listener]
final readonly class SendWelcomeMail implements ListenerInterface
{
    public function listen(): array
    {
        return [
            UserRegistered::class,
        ];
    }

    /**
     * @param UserRegistered $event
     */
    public function process(object $event): void
    {
        echo "Welcome mail sent to {$event->email}", PHP_EOL;
    }
}
```

A script that publishes the event, saved as `bin/register.php`:

```php
<?php

declare(strict_types=1);

use Psr\EventDispatcher\EventDispatcherInterface;
use Src\Account\UserRegistered;
use SuperKernel\Context\ApplicationContext;

require dirname(__DIR__) . '/vendor/autoload.php';

$events = ApplicationContext::getContainer()->get(EventDispatcherInterface::class);

$events->dispatch(new UserRegistered('ada@example.com'));
```

Build the project and run the script:

```bash
skernel build
php bin/register.php
```

It prints:

```text
Audit: ada@example.com registered
Welcome mail sent to ada@example.com
```

### Listen to several events

A listener may list several events, and a listed class or interface also matches its subclasses and implementations, so a listener of `SuperKernel\Contract\EventInterface` receives every SuperKernel lifecycle event. A listener is called once per event even when several entries match. When it lists several events, `process()` tells them apart, for example with `instanceof` or `match`.

A listener of two events, saved as `src/Audit/RecordEverything.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Audit;

use Src\Account\UserRegistered;
use Src\Order\OrderPlaced;
use SuperKernel\Attribute\Listener;
use SuperKernel\Contract\ListenerInterface;

#[Listener(priority: 200)]
final readonly class RecordEverything implements ListenerInterface
{
    public function listen(): array
    {
        return [
            UserRegistered::class,
            OrderPlaced::class,
        ];
    }

    public function process(object $event): void
    {
        $subject = match (true) {
            $event instanceof UserRegistered => "user {$event->email}",
            $event instanceof OrderPlaced => "order of {$event->total}",
        };

        echo "Recorded {$subject}", PHP_EOL;
    }
}
```

Build the project and run the script again:

```bash
skernel build
php bin/register.php
```

It prints:

```text
Recorded user ada@example.com
Audit: ada@example.com registered
Welcome mail sent to ada@example.com
```

### Dispatch an event

Receive `Psr\EventDispatcher\EventDispatcherInterface` through a constructor, or look it up in the container at an entry point as the scripts here do. `dispatch()` accepts any object: an event class needs no base class and need not implement `EventInterface`. It calls the matching listeners one after another in the calling process, then returns the same event object, so listeners can record results on the event for the publisher to read.

An exception thrown by a listener propagates unchanged out of `dispatch()`, and the remaining listeners are not called. The container shares one dispatcher, whether it is looked up by the interface or by its class.

### Stop propagation

An event that implements `Psr\EventDispatcher\StoppableEventInterface` controls how far it travels. The dispatcher asks `isPropagationStopped()` before the first listener and after each one; once it returns `true`, no further listener is called and `dispatch()` returns the event.

A stoppable event, saved as `src/Order/OrderPlaced.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Order;

use Psr\EventDispatcher\StoppableEventInterface;

final class OrderPlaced implements StoppableEventInterface
{
    private bool $rejected = false;

    public function __construct(public readonly int $total)
    {
    }

    public function reject(): void
    {
        $this->rejected = true;
    }

    public function isRejected(): bool
    {
        return $this->rejected;
    }

    public function isPropagationStopped(): bool
    {
        return $this->rejected;
    }
}
```

A listener that stops it, saved as `src/Order/CheckCreditLimit.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Order;

use SuperKernel\Attribute\Listener;
use SuperKernel\Contract\ListenerInterface;

#[Listener(priority: 10)]
final readonly class CheckCreditLimit implements ListenerInterface
{
    public function listen(): array
    {
        return [
            OrderPlaced::class,
        ];
    }

    /**
     * @param OrderPlaced $event
     */
    public function process(object $event): void
    {
        if ($event->total > 1000) {
            echo "Order of {$event->total} rejected", PHP_EOL;
            $event->reject();
        }
    }
}
```

A listener with a lower priority, saved as `src/Order/ShipOrder.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Order;

use SuperKernel\Attribute\Listener;
use SuperKernel\Contract\ListenerInterface;

#[Listener]
final readonly class ShipOrder implements ListenerInterface
{
    public function listen(): array
    {
        return [
            OrderPlaced::class,
        ];
    }

    /**
     * @param OrderPlaced $event
     */
    public function process(object $event): void
    {
        echo "Order of {$event->total} shipped", PHP_EOL;
    }
}
```

A script that places two orders, saved as `bin/order.php`:

```php
<?php

declare(strict_types=1);

use Psr\EventDispatcher\EventDispatcherInterface;
use Src\Order\OrderPlaced;
use SuperKernel\Context\ApplicationContext;

require dirname(__DIR__) . '/vendor/autoload.php';

$events = ApplicationContext::getContainer()->get(EventDispatcherInterface::class);

foreach ([250, 4800] as $total) {
    $order = $events->dispatch(new OrderPlaced($total));

    echo $order->isRejected() ? 'rejected' : 'accepted', PHP_EOL;
}
```

Build the project and run the script:

```bash
skernel build
php bin/order.php
```

It prints:

```text
Recorded order of 250
Order of 250 shipped
accepted
Recorded order of 4800
Order of 4800 rejected
rejected
```

### Read the listeners of an event

The container also provides `Psr\EventDispatcher\ListenerProviderInterface`. `getListenersForEvent()` returns the listeners that match an event as `[$listener, 'process']` callables, in the order the dispatcher calls them. It remembers the result for each event class and keeps no event object.

A script that lists the listeners of an order, saved as `bin/listeners.php`:

```php
<?php

declare(strict_types=1);

use Psr\EventDispatcher\ListenerProviderInterface;
use Src\Order\OrderPlaced;
use SuperKernel\Context\ApplicationContext;

require dirname(__DIR__) . '/vendor/autoload.php';

$provider = ApplicationContext::getContainer()->get(ListenerProviderInterface::class);

foreach ($provider->getListenersForEvent(new OrderPlaced(0)) as [$listener]) {
    echo $listener::class, PHP_EOL;
}
```

Run the script:

```bash
php bin/listeners.php
```

It prints:

```text
Src\Audit\RecordEverything
Src\Order\CheckCreditLimit
Src\Order\ShipOrder
```

## Errors

The listener provider creates every listener the first time the container needs it, which is the first lookup of the dispatcher. An invalid listener raises `InvalidArgumentException` then, wrapped in a `SuperKernel\Di\Exception\ResolutionException` with the message `Unable to invoke factory "SuperKernel\EventDispatcher\Provider\ListenerProviderProvider" for container entry "Psr\EventDispatcher\ListenerProviderInterface".` The same happens in `skernel build`, which dispatches `ApplicationBuild`, so the build fails.

| Situation | Message |
| --- | --- |
| The object created for a `#[Listener]` class does not implement `ListenerInterface` | `Resolved listener <class> must implement SuperKernel\Contract\ListenerInterface, <type> provided.` |
| `listen()` returns an array that is not a list, or a name that is not an existing class or interface | `<class>::listen() must return a list containing only existing event class or interface names.` |

A listener that the container cannot create raises the container's own exception at the same moment.

## Relation to other packages

- [`super-kernel/attribute`](attribute.md) declares `#[Listener]` and [`super-kernel/contract`](contract.md) declares `ListenerInterface` and `EventInterface`; this package gives them their meaning.
- [`super-kernel/di`](di.md) creates the listeners and shares the dispatcher; [`super-kernel/annotator`](annotator.md) provides the recorded `#[Listener]` declarations.
- [`super-kernel/event`](event.md) declares the lifecycle events; [`super-kernel/framework`](framework.md) publishes `BeforeApplicationStart` and adapts Symfony's dispatcher contract to this dispatcher, so Symfony Console events reach the same listeners.
- `skernel build` collects the listeners and publishes `ApplicationBuild` through this dispatcher; `skernel run` builds before it starts the application.

## What not to do

**Do not leave an invalid listener in the project.** Every listener is created together, so a `#[Listener]` class that does not implement `ListenerInterface`, that lists a class that does not exist, or that the container cannot create makes every dispatch fail, not only its own events, and makes `skernel build` fail. Fix or remove the declaration; a listener that should stay silent returns an empty list from `listen()`.
