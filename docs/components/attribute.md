<p align="right">
  <strong>English</strong> | <a href="attribute.zh-CN.md">中文文档</a>
</p>

# SuperKernel Attribute

Package: [`super-kernel/attribute`](https://github.com/super-kernel/attribute) · [Packagist](https://packagist.org/packages/super-kernel/attribute) · [Documentation index](../README.md)

## Purpose

`super-kernel/attribute` holds the native PHP Attributes that SuperKernel packages, third-party packages, and other framework implementations share. Each one fixes a declaration's shape and the meaning it carries: where it can be placed, whether it repeats, its parameters, and which values are valid. Classes declare what they provide with these Attributes, and the consuming packages act on them.

The package contains declarations only and depends on nothing but PHP. `skernel build` collects the Attributes, `super-kernel/annotator` stores and restores them, and the consuming packages, such as `super-kernel/di`, `super-kernel/config`, and `super-kernel/event-dispatcher`, interpret them. Every Attribute lives in the `SuperKernel\Attribute` namespace.

## Capabilities

### Declare services

- `#[Provider(Target::class)]` on a class maps the identifier `Target` to that class, which `super-kernel/di` then constructs. A class may map itself, `#[Provider(OwnClass::class)]`, to be constructed as a shared service. The Attribute is not repeatable.
- `#[FactoryProvider(Target::class)]` on a class declares that the class's public, non-static `__invoke()` creates `Target`. The class may have no constructor or a public one; the container fills the parameters of both. The Attribute is repeatable: each declaration maps one more identifier to the same class, in declaration order, and the container keeps a separate product for each identifier.
- The identifier of both Attributes must be a non-empty string without NUL bytes. Whether the class is compatible with the identifier, and whether an identifier has two providers, the consuming container checks.

The interface, saved as `src/Clock/Clock.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Clock;

interface Clock
{
    public function zone(): string;
}
```

The implementation, saved as `src/Clock/ZonedClock.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Clock;

final readonly class ZonedClock implements Clock
{
    public function __construct(private string $zone)
    {
    }

    public function zone(): string
    {
        return $this->zone;
    }
}
```

A factory for both identifiers, saved as `src/Clock/ClockFactory.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Clock;

use SuperKernel\Attribute\FactoryProvider;

use function getenv;

#[FactoryProvider(Clock::class)]
#[FactoryProvider(ZonedClock::class)]
final class ClockFactory
{
    public function __invoke(): ZonedClock
    {
        return new ZonedClock(getenv('APP_TIMEZONE') ?: 'UTC');
    }
}
```

A script that looks both up, saved as `bin/clock.php`:

```php
<?php

declare(strict_types=1);

use Src\Clock\Clock;
use Src\Clock\ZonedClock;
use SuperKernel\Context\ApplicationContext;

require dirname(__DIR__) . '/vendor/autoload.php';

$container = ApplicationContext::getContainer();

echo $container->get(Clock::class)->zone(), PHP_EOL;
var_dump($container->get(Clock::class) === $container->get(ZonedClock::class));
```

Build the project and run the script:

```bash
skernel build
APP_TIMEZONE=Asia/Shanghai php bin/clock.php
```

It prints the following; the last line shows that each identifier has its own product:

```text
Asia/Shanghai
bool(false)
```

The rules of `super-kernel/di` for both Attributes are on its page: [Declare services](di.md#declare-services).

### Inject properties

`#[Autowired]` on a property asks the container to set it after the constructor returns.

- Without an argument, the property's type selects the entry. `#[Autowired(Target::class)]` names the class or interface to inject instead; it must exist, or constructing the Attribute fails with `Injectable identifier <identifier> does not exist.`
- `#[Autowired]` takes no other parameter. Whether the property receives a lazy proxy is the container's choice; `super-kernel/di` injects lazy proxies by default.
- The Attribute may also be placed on a method, where `super-kernel/di` treats it as metadata only and never calls the method.

A class with an injected property, saved as `src/Clock/Schedule.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Clock;

use SuperKernel\Attribute\Autowired;

final class Schedule
{
    #[Autowired(ZonedClock::class)]
    public Clock $clock;
}
```

A script that uses it, saved as `bin/schedule.php`:

```php
<?php

declare(strict_types=1);

use Src\Clock\Schedule;
use SuperKernel\Context\ApplicationContext;

require dirname(__DIR__) . '/vendor/autoload.php';

echo ApplicationContext::getContainer()->get(Schedule::class)->clock->zone(), PHP_EOL;
```

After `skernel build`, `php bin/schedule.php` prints `UTC`.

The injection order and the lazy proxies are described on the DI page: [Property injection](di.md#property-injection).

### Extend the container

- `#[Definer(priority: N)]` registers a class that turns identifiers into definitions; `#[Resolver(priority: N)]` registers a class that turns definitions into values. Both take one integer `priority`, `0` by default, and are not repeatable. A higher priority is asked first; equal priorities keep the order of discovery.
- `#[PropertyHandler(TheAttribute::class)]` registers the annotated class as the handler of a custom property Attribute. It is repeatable, so one handler may serve several Attributes. The mapped Attribute declares its own target with PHP's `#[Attribute(Attribute::TARGET_PROPERTY)]`; the name must be a non-empty string without NUL bytes.

The interfaces these classes implement and complete examples are on the DI page: [Custom property Attributes](di.md#custom-property-attributes) and [Definers and resolvers](di.md#definers-and-resolvers).

### Declare configuration

`#[Configuration]` serves `super-kernel/config`. It is not repeatable, and takes `?string $className = null` and `bool $default = false`:

- On a class, it declares a configuration source. `className` names the class or interface the source provides; when omitted, `super-kernel/config` infers it from the interfaces the class implements. A given `className` must be a non-empty string without NUL bytes.
- On a method of a source that implements `SuperKernel\Contract\MultipleConfigInterface`, it declares a named item; the method name is the item's name. `className` may be omitted or repeat the source's item type. Pass `default: true` by name to choose the default among several items.

`#[Provision]` on a method declares that the method creates a container service. It takes any number of names, `#[Provision('a', 'b')]`, kept in order in its public `name` list; each must be a non-empty string without NUL bytes. Without names, the service's identifier is the method's return type. The class needs no `#[Configuration]`, and `super-kernel/config` calls the method through the container.

The configuration interface, saved as `src/Mail/MailConfig.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Mail;

use SuperKernel\Contract\SingleConfigInterface;

interface MailConfig extends SingleConfigInterface
{
    public function host(): string;
}
```

The application's source, saved as `src/Mail/AppMailConfig.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Mail;

use SuperKernel\Attribute\Configuration;

#[Configuration(MailConfig::class)]
final readonly class AppMailConfig implements MailConfig
{
    public function host(): string
    {
        return 'smtp.example.com';
    }
}
```

A factory method for a named service, saved as `src/Mail/MailServices.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Mail;

use SuperKernel\Attribute\Provision;
use SuperKernel\Contract\ConfigInterface;

final readonly class MailServices
{
    public function __construct(private ConfigInterface $config)
    {
    }

    #[Provision('mail.dsn')]
    public function dsn(): string
    {
        $mail = $this->config->get(MailConfig::class);
        assert($mail instanceof MailConfig);

        return 'smtp://' . $mail->host();
    }
}
```

A script that reads the service, saved as `bin/mail.php`:

```php
<?php

declare(strict_types=1);

use SuperKernel\Context\ApplicationContext;

require dirname(__DIR__) . '/vendor/autoload.php';

echo ApplicationContext::getContainer()->get('mail.dsn'), PHP_EOL;
```

After `skernel build`, `php bin/mail.php` prints `smtp://smtp.example.com`.

Sources, items, defaults, and provisions are described on the Config page: [SuperKernel Config](config.md).

### Register event listeners

`#[Listener]` on a class that implements `SuperKernel\Contract\ListenerInterface` registers it with `super-kernel/event-dispatcher`. It takes one integer `priority`, `0` by default, and is not repeatable. A higher priority runs first; equal priorities keep the order of discovery. The events a listener receives are returned by its `listen()` method, not declared by the Attribute.

The event, saved as `src/Signup/UserRegistered.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Signup;

final readonly class UserRegistered
{
    public function __construct(public string $email)
    {
    }
}
```

A listener with a high priority, saved as `src/Signup/RecordAuditEntry.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Signup;

use SuperKernel\Attribute\Listener;
use SuperKernel\Contract\ListenerInterface;

#[Listener(priority: 100)]
final class RecordAuditEntry implements ListenerInterface
{
    public function listen(): array
    {
        return [
            UserRegistered::class,
        ];
    }

    public function process(object $event): void
    {
        assert($event instanceof UserRegistered);

        echo "Audit: {$event->email} registered", PHP_EOL;
    }
}
```

A listener with the default priority, saved as `src/Signup/AddToNewsletter.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Signup;

use SuperKernel\Attribute\Listener;
use SuperKernel\Contract\ListenerInterface;

#[Listener]
final class AddToNewsletter implements ListenerInterface
{
    public function listen(): array
    {
        return [
            UserRegistered::class,
        ];
    }

    public function process(object $event): void
    {
        assert($event instanceof UserRegistered);

        echo "Newsletter: {$event->email} added", PHP_EOL;
    }
}
```

A script that publishes the event, saved as `bin/signup.php`:

```php
<?php

declare(strict_types=1);

use Psr\EventDispatcher\EventDispatcherInterface;
use Src\Signup\UserRegistered;
use SuperKernel\Context\ApplicationContext;

require dirname(__DIR__) . '/vendor/autoload.php';

ApplicationContext::getContainer()
    ->get(EventDispatcherInterface::class)
    ->dispatch(new UserRegistered('ada@example.com'));
```

After `skernel build`, `php bin/signup.php` prints:

```text
Audit: ada@example.com registered
Newsletter: ada@example.com added
```

### Use the Attributes in a package

A package whose own code uses these Attributes requires `super-kernel/attribute` directly. Receiving it through `super-kernel/di` would tie the package to one container, while the Attributes are meant to be interpreted by any implementation; a package that uses them only in tests may require it for development only.

`super-kernel/di` interprets `Provider`, `FactoryProvider`, `Autowired`, `Definer`, `Resolver`, and `PropertyHandler`; `super-kernel/config` interprets `Configuration` and `Provision`; `super-kernel/event-dispatcher` interprets `Listener`. What another container does with them is up to that container.

## Errors

The Attributes raise `InvalidArgumentException` when they are constructed, which happens during `skernel build`. `skernel build` then skips the class, prints it with the message, and records it in the skipped-class diagnostics of `super-kernel/annotator`.

| Situation | Message |
| --- | --- |
| `Provider` with an empty identifier or one with a NUL byte | `Provider className must be non-empty and contain no NUL bytes.` |
| `FactoryProvider` with an empty identifier or one with a NUL byte | `FactoryProvider className must be non-empty and contain no NUL bytes.` |
| `PropertyHandler` with an empty Attribute name or one with a NUL byte | `PropertyHandler attribute must be non-empty and contain no NUL bytes.` |
| `Autowired` naming a class or interface that does not exist | `Injectable identifier <identifier> does not exist.` |
| `Configuration` with an empty `className` or one with a NUL byte | `Configuration className must be non-empty and contain no NUL bytes when provided.` |
| `Provision` with an empty name or one with a NUL byte | `Provision name must be non-empty and contain no NUL bytes.` |

## Relation to other packages

- [`super-kernel/di`](di.md) interprets the service, injection, and extension Attributes.
- [`super-kernel/config`](config.md) interprets `Configuration` and `Provision`.
- [`super-kernel/event-dispatcher`](event-dispatcher.md) interprets `Listener`.
- [`super-kernel/annotator`](annotator.md) stores the Attributes that `skernel build` collects and answers queries about them.
- [`super-kernel/contract`](contract.md) declares the interfaces that annotated classes implement, such as `ListenerInterface` and `MultipleConfigInterface`.
- Exception handlers are declared with an Attribute of [`super-kernel/exception-dispatcher`](exception-dispatcher.md), not of this package.

## What not to do

**Do not rely on `super-kernel/di` to install the Attributes.** A package whose code uses them then breaks in an application that installs another container instead. Require `super-kernel/attribute` directly.

**Do not ignore the classes that `skernel build` skips.** An invalid Attribute value, such as `#[Provider('')]`, fails with `Provider className must be non-empty and contain no NUL bytes.`, and the whole class is left out: its other Attributes are lost too, and the application behaves as if the class had none. Read what `skernel build` prints and fix the declaration.

**Do not expect the Attribute to check what its consumer checks.** An Attribute only checks its own values. Whether a `Provider`'s class is compatible with its identifier, or whether two classes provide one identifier, is reported by `super-kernel/di` at the first lookup that needs it.
