<p align="right">
  <strong>English</strong> | <a href="config.zh-CN.md">中文文档</a>
</p>

# SuperKernel Config

Package: [`super-kernel/config`](https://github.com/super-kernel/config) · [Packagist](https://packagist.org/packages/super-kernel/config) · [Documentation index](../README.md)

## Purpose

`super-kernel/config` connects the configuration an application declares with the components that read it. A component publishes a configuration contract, a class or interface; the application declares a source for it with `#[Configuration]`; the component reads the item through `SuperKernel\Contract\ConfigInterface`, which loads it on first use and returns the same object afterwards. A source can provide one object, or several named items with a default, such as several database connections.

Config works with objects only. Reading raw values from files or the environment belongs to the sources, for example with `super-kernel/environment`; checking that a value suits a component belongs to that component. The package also gives `#[Provision]` methods their meaning: each becomes a factory for a container service.

## Capabilities

### Declare a single source

`#[Configuration(Contract::class)]` on a class declares it as the source of `Contract`, a class or interface. The source must be an instantiable class compatible with the contract, and is itself the contract's only item, its default, also available under the name `default`.

- Without an argument, `#[Configuration]` uses the one most-specific interface the class implements, ignoring `SingleConfigInterface` and interfaces inherited through others. When that is not exactly one interface, the declaration is invalid.
- Configuration interfaces of components usually extend `SuperKernel\Contract\SingleConfigInterface`; a single source does not need it.
- A contract has at most one source.
- Config creates the source through the container, so its constructor may receive services and other configuration.

The configuration interface, saved as `src/Mail/MailConfig.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Mail;

use SuperKernel\Contract\SingleConfigInterface;

interface MailConfig extends SingleConfigInterface
{
    public function host(): string;

    public function port(): int;
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

    public function port(): int
    {
        return 587;
    }
}
```

A script that reads it, saved as `bin/config.php`:

```php
<?php

declare(strict_types=1);

use Src\Mail\MailConfig;
use SuperKernel\Context\ApplicationContext;
use SuperKernel\Contract\ConfigInterface;

require dirname(__DIR__) . '/vendor/autoload.php';

$config = ApplicationContext::getContainer()->get(ConfigInterface::class);
$mail = $config->get(MailConfig::class);

echo $mail->host(), ':', $mail->port(), PHP_EOL;
var_dump($mail === $config->get(MailConfig::class, 'default'));
```

Build the project and run the script:

```bash
skernel build
php bin/config.php
```

It prints:

```text
smtp.example.com:587
bool(true)
```

### Declare named items

A source that implements `SuperKernel\Contract\MultipleConfigInterface` provides several items of one contract, which its class-level `#[Configuration(Contract::class)]` must name explicitly. Each item is a method marked with `#[Configuration]`:

- The method name, case-sensitive, is the item's name. Unmarked methods are not items.
- The method must be public, non-static, and without parameters, and declare a non-nullable class or interface return type compatible with the contract. A method-level `#[Configuration]` may repeat the contract or leave it out, never name another.
- The source does not implement the contract itself; the items it returns must be instances of it.
- With one item, that item is the default. With several, `#[Configuration(default: true)]` marks the default; at most one may be marked. Without a marked default, named lookups work and only an unnamed lookup fails. The order of the methods never selects a default.
- A source with no item, or one that implements both `SingleConfigInterface` and `MultipleConfigInterface`, is invalid.

The contract may be a plain class, such as a final readonly class with public properties; it needs no interface or container entry. Config calls the item method itself and never asks the container for the item.

The option class, saved as `src/Database/DatabaseConfig.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Database;

final readonly class DatabaseConfig
{
    public function __construct(public string $host, public string $database)
    {
    }
}
```

The source of two connections, saved as `src/Database/DatabaseConnections.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Database;

use SuperKernel\Attribute\Configuration;
use SuperKernel\Contract\MultipleConfigInterface;

#[Configuration(DatabaseConfig::class)]
final readonly class DatabaseConnections implements MultipleConfigInterface
{
    #[Configuration(default: true)]
    public function primary(): DatabaseConfig
    {
        echo 'Loading primary', PHP_EOL;

        return new DatabaseConfig('db.internal', 'billing');
    }

    #[Configuration]
    public function reporting(): DatabaseConfig
    {
        echo 'Loading reporting', PHP_EOL;

        return new DatabaseConfig('replica.internal', 'billing');
    }
}
```

A script that reads the connections, saved as `bin/database.php`:

```php
<?php

declare(strict_types=1);

use Src\Database\DatabaseConfig;
use SuperKernel\Context\ApplicationContext;
use SuperKernel\Contract\ConfigInterface;

require dirname(__DIR__) . '/vendor/autoload.php';

$config = ApplicationContext::getContainer()->get(ConfigInterface::class);

var_dump($config->has(DatabaseConfig::class, 'reporting'), $config->has(DatabaseConfig::class, 'archive'));

$primary = $config->get(DatabaseConfig::class);

echo $primary->host, PHP_EOL;
var_dump($primary === $config->get(DatabaseConfig::class, 'primary'));
echo $config->get(DatabaseConfig::class, 'reporting')->host, PHP_EOL;
echo $config->get(DatabaseConfig::class, 'reporting')->host, PHP_EOL;
var_dump($config->get(DatabaseConfig::class, 'archive'));
```

After `skernel build`, `php bin/database.php` prints the following; each item method runs once, when its item is first read:

```text
bool(true)
bool(false)
Loading primary
db.internal
bool(true)
Loading reporting
replica.internal
replica.internal
NULL
```

### Read items

`ConfigInterface` is created by the container. Creating it checks every declaration of the application and its packages, without creating a source or calling an item method.

- `has($contract)` reports whether the contract has a source, also one with several items and no default; `has($contract, $name)` reports whether that item is declared. Neither loads anything, and neither says whether loading will succeed.
- `get($contract, $name = null)` returns the item. On the first lookup it creates the source through the container, checks that it is an instance of its own class, calls only the requested item method, and keeps the result; later lookups by the name or as the default return that object.
- `get()` returns `null` for an undeclared contract or name without touching the container. For several items without a default, `get($contract)` raises `ConfigurationNotFoundException` before creating the source.
- A failed load keeps nothing and is retried by the next lookup; a source that was created stays when one of its item methods fails.
- A source or item that needs itself while it is being loaded, directly or through other configuration, raises `CircularConfigurationDependencyException` with the path of contracts and items.
- Exceptions of the package pass through unchanged. Other failures, and an item of the wrong type, are reported as `ConfigurationLoadingException` with the cause as its previous exception.

Items are kept, not copied: keep their getters returning the same values while the application runs.

### Validate configuration in a component

Config checks types only. A component that needs valid values, such as a port in range, binds its configuration interface with a `#[FactoryProvider]` that loads the item through `ConfigInterface`, rejects a missing one with `ConfigurationNotFoundException`, checks the values, and returns the same object. Code that needs the configuration then receives the interface by injection, already checked. Config creates sources by their class, never by the interface, so this provider does not call itself.

- When the check fails, the container keeps no entry for the interface; Config may already keep the raw item, so the provider checks it again on the next attempt.
- A component that offers named items, such as several connections, shares one loader that checks each item between its default provider and its factories.
- An optional component checks `has()` first and skips itself when nothing is declared; a declared but invalid source, or a missing default, must still fail.

A validating provider, saved as `src/Mail/MailConfigProvider.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Mail;

use InvalidArgumentException;
use SuperKernel\Attribute\FactoryProvider;
use SuperKernel\Config\Exception\ConfigurationNotFoundException;
use SuperKernel\Contract\ConfigInterface;

#[FactoryProvider(MailConfig::class)]
final readonly class MailConfigProvider
{
    public function __invoke(ConfigInterface $config): MailConfig
    {
        $mail = $config->get(MailConfig::class)
            ?? throw new ConfigurationNotFoundException('No mail configuration is declared.');

        if ($mail->port() < 1 || $mail->port() > 65535) {
            throw new InvalidArgumentException("Mail port {$mail->port()} is out of range.");
        }

        return $mail;
    }
}
```

A service that receives the checked configuration, saved as `src/Mail/Mailer.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Mail;

final readonly class Mailer
{
    public function __construct(private MailConfig $config)
    {
    }

    public function describe(): string
    {
        return "Sending through {$this->config->host()}:{$this->config->port()}";
    }
}
```

A script that uses the service, saved as `bin/mailer.php`:

```php
<?php

declare(strict_types=1);

use Src\Mail\Mailer;
use SuperKernel\Context\ApplicationContext;

require dirname(__DIR__) . '/vendor/autoload.php';

echo ApplicationContext::getContainer()->get(Mailer::class)->describe(), PHP_EOL;
```

After `skernel build`, `php bin/mailer.php` prints `Sending through smtp.example.com:587`.

### Create services with factory methods

A method marked `#[Provision]` becomes the factory of a container service. Its class needs no `#[Configuration]`; the container creates the class once, injecting its constructor, and calls the method with injected parameters.

- Without names, the service's identifier is the method's return type, which must be one non-nullable class or interface; `self`, `static`, and `parent` are resolved against the declaring class. Any other return type needs names.
- `#[Provision('a', 'b')]` registers exactly those identifiers. Each keeps its own product, so `a` and `b` are different objects, while both share the factory class. A repeated name on one method counts once; two methods with one identifier are an error.
- A provision cannot provide its own class.
- The class must be instantiable, and the method public, non-static, and neither a constructor nor a destructor.
- `#[Provider]` and `#[FactoryProvider]` take precedence over a provision for the same identifier; a provision takes precedence over plain construction of a class.
- The container keeps each product, including `null` for an identifier that is not a class. Parameters are filled by the rules of `super-kernel/di`. PSR-11 exceptions pass through; other failures are wrapped in `ConfigurationLoadingException`, and the next lookup tries again. Dependency cycles between provisions are not detected.

A class with factory methods, saved as `src/Clock/ClockServices.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Clock;

use DateTimeImmutable;
use DateTimeZone;
use SuperKernel\Attribute\Provision;
use Src\Mail\MailConfig;

final readonly class ClockServices
{
    public function __construct(private MailConfig $mail)
    {
    }

    #[Provision]
    public function timeZone(): DateTimeZone
    {
        return new DateTimeZone('Europe/Berlin');
    }

    #[Provision('clock.started', 'clock.now')]
    public function now(DateTimeZone $timeZone): DateTimeImmutable
    {
        return new DateTimeImmutable('2026-11-10 09:00', $timeZone);
    }

    #[Provision('mail.dsn')]
    public function mailDsn(): string
    {
        return "smtp://{$this->mail->host()}:{$this->mail->port()}";
    }
}
```

A script that reads the services, saved as `bin/provisions.php`:

```php
<?php

declare(strict_types=1);

use SuperKernel\Context\ApplicationContext;

require dirname(__DIR__) . '/vendor/autoload.php';

$container = ApplicationContext::getContainer();

echo $container->get(DateTimeZone::class)->getName(), PHP_EOL;
echo $container->get('clock.now')->format(DATE_ATOM), PHP_EOL;
var_dump($container->get('clock.now') === $container->get('clock.now'));
var_dump($container->get('clock.now') === $container->get('clock.started'));
echo $container->get('mail.dsn'), PHP_EOL;
```

After `skernel build`, `php bin/provisions.php` prints:

```text
Europe/Berlin
2026-11-10T09:00:00+01:00
bool(true)
bool(false)
smtp://smtp.example.com:587
```

`SuperKernel\Config\Collector\ProvisionCollector` lists the declarations without creating or calling anything: `getMethods()` returns, by class and method name, the `Provision` instance as `attribute` and the method's native return type, or `null`, as `returnType`. A class or method that cannot be reflected raises `ReflectionException` when the collector is created.

## Errors

Every exception of the package implements `SuperKernel\Config\Exception\ConfigExceptionInterface`, a PSR-11 `ContainerExceptionInterface`; `ConfigurationNotFoundException` is also a PSR-11 not-found exception.

| Situation | Message |
| --- | --- |
| Two sources for one contract | `Configuration contract <contract> is declared by both <source> and <source>.` |
| `#[Configuration]` without a contract on a class with not exactly one most-specific interface | `Configuration source <source> must implement exactly one most-specific interface when SuperKernel\Attribute\Configuration has no explicit argument; found <n>: <interfaces>.` |
| A source not compatible with its contract | `Configuration source <source> must be compatible with its declared contract <contract>.` |
| A source that cannot be instantiated | `Configuration source <source> must be an instantiable class.` |
| A contract that is not an existing class or interface | `Configuration source <source> explicitly names <contract>, which is not an existing class or interface.` |
| A source with named items that does not name its contract | `Multiple configuration source <source> must explicitly name its item contract in Configuration.` |
| A source that is single and multiple | `A configuration source cannot be both single and multiple.` |
| An item method with a wrong signature or two `#[Configuration]` | `Configuration item <source>::<method> must be a public non-static parameterless method returning <contract>, marked exactly once with Configuration.` |
| An item method naming another contract | `Configuration item <source>::<method> must use its source contract <contract> instead of <other>.` |
| Two default items | `Multiple configuration source <source> declares more than one default item.` |
| A source without items | `Multiple configuration source <source> must declare at least one configuration item.` |
| Several items, no default, lookup without a name (`ConfigurationNotFoundException`) | `Configuration contract <contract> has multiple items but no default item is declared.` |
| A source or item that needs itself (`CircularConfigurationDependencyException`) | `Circular configuration dependency detected: <path>.` |
| A source or item method that fails (`ConfigurationLoadingException`) | `Unable to load configuration item <contract> from source <source>.` |
| An item of the wrong type (`ConfigurationLoadingException`) | `Configuration item <contract> resolved to <type> instead of <contract>.` |
| Two provisions for one identifier | `Provision for <id> is declared by both <class>::<method> and <class>::<method>.` |
| An unnamed provision without a usable return type | `Unnamed provision <class>::<method> must declare a non-nullable class or interface return type; otherwise specify Provision names explicitly.` |
| A provision of its own class | `Provision <class>::<method> cannot replace its own source class; use an explicit service name.` |
| A provision method that is not public and non-static on an instantiable class | `Provision method <class>::<method> must be public and non-static on an instantiable class, and cannot be a constructor or destructor.` |
| A provision method that fails (`ConfigurationLoadingException`) | `Unable to invoke provision <class>::<method> for container entry "<id>".` |

Invalid provisions are reported by the container's definer: the container raises `Unable to obtain a definition for container entry "<id>".` with the Config exception among its previous exceptions.

## Relation to other packages

- [`super-kernel/attribute`](attribute.md) declares `#[Configuration]` and `#[Provision]`; [`super-kernel/contract`](contract.md) declares `ConfigInterface`, `SingleConfigInterface`, and `MultipleConfigInterface`.
- [`super-kernel/di`](di.md) creates sources and provision classes, fills their parameters, and keeps provision products; this package registers a definer and a resolver in it.
- [`super-kernel/annotator`](annotator.md) supplies the collected declarations; [`super-kernel/reflector`](reflector.md) the shared reflector; [`super-kernel/support`](support.md)'s `make()` creates `ConfigInterface`.
- [`super-kernel/environment`](environment.md) reads raw properties, such as `.env` values, that sources may use.
- Components such as [`super-kernel/server`](server.md) and [`super-kernel/redis`](redis.md) publish configuration contracts and check their values.

## What not to do

**Do not declare two sources for one contract.** Creating `ConfigInterface` fails with `Configuration contract <contract> is declared by both <source> and <source>.`, and nothing that needs configuration can start. Keep one source per contract; offer alternatives as named items of one source.

**Do not declare two provisions for one identifier.** The container's definer fails, and from then on every lookup of an entry not yet created fails with `Unable to obtain a definition for container entry "<id>".`, also for unrelated services. Give each provision its own name.

**Do not rely on method order for a default.** With several items and no `default: true`, `get($contract)` raises `Configuration contract <contract> has multiple items but no default item is declared.` Mark exactly one default, or always read by name.

**Do not replace a component's validating provider to set values.** A `#[Provider]` or `#[FactoryProvider]` of your own for the component's configuration interface bypasses the component's checks. Declare a `#[Configuration]` source with your values instead.

**Do not snapshot or mutate items.** Config returns the objects your sources build and keeps them; a copy taken by a consumer goes stale, and an item whose getters change at run time gives different components different answers. Keep items immutable and read them through `ConfigInterface` or injection.

**Do not let sources depend on each other in a circle.** A source whose constructor reads a configuration that needs the first one fails with `Circular configuration dependency detected: <path>.` Move the shared value into a third source. Cycles between provisions are not detected at all and end the process.

**Do not read raw environment values inside components.** Components read typed configuration; the application's source turns raw values, for example from `super-kernel/environment`, into the configuration object.
