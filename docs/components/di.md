<p align="right">
  <strong>English</strong> | <a href="di.zh-CN.md">中文文档</a>
</p>

# SuperKernel DI

Package: [`super-kernel/di`](https://github.com/super-kernel/di) · [Packagist](https://packagist.org/packages/super-kernel/di) · [Documentation index](../README.md)

## Purpose

`super-kernel/di` is the dependency-injection container of SuperKernel and implements PSR-11 (`psr/container-implementation` 2.0). Classes declare what they provide with Attributes; the container creates each service the first time it is requested, injects the service's dependencies, and keeps it for the life of the container, so every consumer shares one instance.

The container does not scan the project. `skernel build` collects the Attributes once and writes them as metadata; the container restores that metadata when the application starts.

## Capabilities

### Declare services

Three kinds of identifiers have a definition:

- `#[Provider(Target::class)]` on an instantiable class maps `Target` to that class. The class must be `Target` itself or a subtype of it. A class mapped to itself is registered as a shared class. Mappings may chain, such as an interface to a narrower interface to a class, and every identifier in the chain receives the same instance.
- `#[FactoryProvider(Target::class)]` creates `Target` by calling the annotated class. The class needs no constructor or a public one, and must declare its own public, non-static `__invoke()`, whose parameters are injected. The Attribute is repeatable, so one class may create several identifiers. Use it for construction that needs code, such as validation or reading settings; when the constructor alone is enough, use `Provider`.
- Any other instantiable class is created from its constructor when it is requested; it needs no Attribute.

An interface or abstract class without a `Provider` or `FactoryProvider` has no definition.

The interface, saved as `src/Clock/Clock.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Clock;

use DateTimeImmutable;

interface Clock
{
    public function now(): DateTimeImmutable;
}
```

The class that provides it, saved as `src/Clock/SystemClock.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Clock;

use DateTimeImmutable;
use DateTimeZone;
use SuperKernel\Attribute\Provider;

#[Provider(Clock::class)]
final readonly class SystemClock implements Clock
{
    public function __construct(private DateTimeZone $timeZone)
    {
    }

    public function now(): DateTimeImmutable
    {
        return new DateTimeImmutable('now', $this->timeZone);
    }
}
```

A factory for the time zone that `SystemClock` needs, saved as `src/Clock/TimeZoneProvider.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Clock;

use DateTimeZone;
use SuperKernel\Attribute\FactoryProvider;

use function getenv;

#[FactoryProvider(DateTimeZone::class)]
final class TimeZoneProvider
{
    public function __invoke(): DateTimeZone
    {
        return new DateTimeZone(getenv('APP_TIMEZONE') ?: 'UTC');
    }
}
```

A script that looks the services up, saved as `bin/clock.php`:

```php
<?php

declare(strict_types=1);

use Src\Clock\Clock;
use Src\Clock\SystemClock;
use SuperKernel\Di\Provider\ContainerProvider;

require dirname(__DIR__) . '/vendor/autoload.php';

$container = new ContainerProvider()();

echo $container->get(Clock::class)->now()->getTimezone()->getName(), PHP_EOL;
var_dump($container->get(Clock::class) === $container->get(SystemClock::class));
```

Build the project and run the script:

```bash
skernel build
APP_TIMEZONE=Asia/Shanghai php bin/clock.php
```

It prints:

```text
Asia/Shanghai
bool(true)
```

### Constructor and factory injection

The container fills the parameters of a constructor, and of a `FactoryProvider`'s `__invoke()`, in order. Each parameter receives the first of:

1. a value that the caller put in a `SuperKernel\Di\Definition\MethodDefinition`, matched by parameter name and then by position; this applies only to definitions that a resolver or a caller creates;
2. the container's entry for the class or interface in the parameter's type, when exactly one class or interface in the type has an entry;
3. the parameter's default value;
4. `null`, when the type allows it.

A variadic parameter receives nothing. A by-reference parameter, a type in which more than one class or interface has an entry, and a required parameter that none of these sources fills are errors. Scalar types such as `string` and `int` are never filled from the container; give such a parameter a default value, or create the object in a `FactoryProvider`.

### Property injection

`#[Autowired]` on a property asks the container to set it after the constructor returns. The container chooses what to inject in this order:

1. the class or interface named by `#[Autowired(Target::class)]`;
2. otherwise, the single existing class or interface in the property's type;
3. otherwise, the property's default value;
4. otherwise, it raises an error.

Once an identifier is chosen, the container's entry is used; an entry that cannot be created is an error, and the default value is not used instead. Public, protected, private, and uninitialized readonly properties are supported; static properties are rejected. `#[Autowired]` on a method is only metadata: the container never calls that method.

By default the property receives a PHP 8.4 lazy proxy instead of the entry itself. The proxy obtains the entry through `Container::get()` the first time its state is read or written, so a dependency is created only when it is used. A failed initialization is retried at the next access. Initialization runs synchronously, without a guard against two coroutines initializing the same proxy at once. The proxy behaves as the entry but is not identical (`===`) to it. The container injects the entry itself, without a proxy, when no concrete class is known for the identifier, such as an interface created only by a `FactoryProvider`, and for internal classes other than `stdClass`, classes that extend them, and classes without stored instance properties. An interface or abstract class is proxied only when a `Provider` maps it to a concrete class.

A class with an injected property, saved as `src/Clock/Report.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Clock;

use SuperKernel\Attribute\Autowired;

final class Report
{
    #[Autowired]
    private Clock $clock;

    public function generatedAt(): string
    {
        return $this->clock->now()->format(DATE_ATOM);
    }
}
```

A script that uses it, saved as `bin/report.php`:

```php
<?php

declare(strict_types=1);

use Src\Clock\Report;
use SuperKernel\Di\Provider\ContainerProvider;

require dirname(__DIR__) . '/vendor/autoload.php';

echo new ContainerProvider()()->get(Report::class)->generatedAt(), PHP_EOL;
```

After `skernel build`, `php bin/report.php` prints the current time in UTC, such as `2026-11-10T10:30:00+00:00`.

### Custom property Attributes

A package or an application can give its own property Attribute a meaning, without depending on this package's classes: declare a handler that implements `SuperKernel\Contract\PropertyAttributeHandlerInterface` and annotate it with `#[PropertyHandler(TheAttribute::class)]`.

- The mapped class must be an instantiable native Attribute that can target properties, and each Attribute has at most one handler. `#[PropertyHandler]` is repeatable, so one handler may serve several Attributes; the container creates one instance of each handler and passes the root container to its constructor.
- The container processes the properties of a class in the order reflection lists them and, on each property, the mapped Attributes in declaration order. Property Attributes without a handler are ignored. A handled Attribute on a static property is an error.
- The handler receives a `SuperKernel\Contract\PropertyContextInterface` with the class name and the reflected property. Calling `assign()` sets the property; not calling it leaves the property as it is; `assign(null)` sets `null`; a second assignment to the same property is an error.

The Attribute, saved as `src/Env/FromEnv.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Env;

use Attribute;

#[Attribute(Attribute::TARGET_PROPERTY)]
final readonly class FromEnv
{
    public function __construct(public string $name)
    {
    }
}
```

Its handler, saved as `src/Env/FromEnvPropertyHandler.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Env;

use SuperKernel\Attribute\PropertyHandler;
use SuperKernel\Contract\ContainerInterface;
use SuperKernel\Contract\PropertyAttributeHandlerInterface;
use SuperKernel\Contract\PropertyContextInterface;

use function getenv;

#[PropertyHandler(FromEnv::class)]
final readonly class FromEnvPropertyHandler implements PropertyAttributeHandlerInterface
{
    public function __construct(private ContainerInterface $container)
    {
    }

    public function handle(PropertyContextInterface $context, object $attribute): void
    {
        assert($attribute instanceof FromEnv);

        $value = getenv($attribute->name);

        if (false !== $value) {
            $context->assign($value);
        }
    }
}
```

A class that uses the Attribute, saved as `src/Mail/MailSettings.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Mail;

use Src\Env\FromEnv;

final class MailSettings
{
    #[FromEnv('MAIL_HOST')]
    public string $host = 'localhost';
}
```

A script that reads the setting, saved as `bin/mail.php`:

```php
<?php

declare(strict_types=1);

use Src\Mail\MailSettings;
use SuperKernel\Di\Provider\ContainerProvider;

require dirname(__DIR__) . '/vendor/autoload.php';

echo new ContainerProvider()()->get(MailSettings::class)->host, PHP_EOL;
```

After `skernel build`, `php bin/mail.php` prints `localhost`, and `MAIL_HOST=smtp.example.com php bin/mail.php` prints `smtp.example.com`.

### Definers and resolvers

The container turns an identifier into a value in two steps. A definer turns the identifier into a definition, an object that describes how to create the value; a resolver turns the definition into the value. For each step the container asks the registered implementations in priority order, highest first, and uses the first one that supports the identifier or definition. Implementations with equal priorities keep the order in which they were discovered, so give each extension its own priority.

- `#[Definer(priority: N)]` registers an instantiable class that implements `SuperKernel\Di\Contract\DefinerInterface`. Its constructor has no required parameter, or exactly one, which receives the `SuperKernel\Contract\AnnotationCollectorInterface`. The built-in definers have the priorities Provider `200`, FactoryProvider `100`, and Class `0`. The container remembers each identifier's definition, and that an identifier has none, so a definer is asked about an identifier at most once per container.
- `#[Resolver(priority: N)]` registers a class that implements `SuperKernel\Di\Contract\ResolverInterface`; the container creates it with the root container, typed as `SuperKernel\Contract\ContainerInterface`. The built-in resolvers have the priorities Factory `400`, Provider `300`, Class `200`, Property `100`, and Method `0`; a priority above `400` comes before all of them. A resolver that needs another definition resolved passes it to the `SuperKernel\Di\Contract\ResolverFactoryInterface` from the container, never to another resolver directly. Priority only selects the resolver for a definition; it does not order constructor or property injection.
- The container discovers definers and resolvers at its first lookup. When an extension's constructor may suspend a coroutine, make the first lookup before the container is shared with concurrent work.

Values are kept like any other entry: the identifier `env.APP_ENV` in the example below is read once per container.

The definition, saved as `src/Env/EnvDefinition.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Env;

use SuperKernel\Di\Contract\DefinitionInterface;

final readonly class EnvDefinition implements DefinitionInterface
{
    public function __construct(public string $name)
    {
    }

    public function identifier(): string
    {
        return 'env.' . $this->name;
    }
}
```

The definer for identifiers that start with `env.`, saved as `src/Env/EnvDefiner.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Env;

use SuperKernel\Attribute\Definer;
use SuperKernel\Di\Contract\DefinerInterface;
use SuperKernel\Di\Contract\DefinitionInterface;

use function str_starts_with;
use function substr;

#[Definer(priority: 300)]
final class EnvDefiner implements DefinerInterface
{
    public function support(string $id): bool
    {
        return str_starts_with($id, 'env.');
    }

    public function get(string $id): DefinitionInterface
    {
        return new EnvDefinition(substr($id, 4));
    }
}
```

The resolver, saved as `src/Env/EnvResolver.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Env;

use SuperKernel\Attribute\Resolver;
use SuperKernel\Contract\ContainerInterface;
use SuperKernel\Di\Contract\DefinitionInterface;
use SuperKernel\Di\Contract\ResolverInterface;

use function getenv;

#[Resolver(priority: 500)]
final readonly class EnvResolver implements ResolverInterface
{
    public function __construct(private ContainerInterface $container)
    {
    }

    public function support(DefinitionInterface $definition): bool
    {
        return $definition instanceof EnvDefinition;
    }

    public function resolve(DefinitionInterface $definition): mixed
    {
        assert($definition instanceof EnvDefinition);

        $value = getenv($definition->name);

        return false === $value ? null : $value;
    }
}
```

A script that reads an identifier, saved as `bin/env.php`:

```php
<?php

declare(strict_types=1);

use SuperKernel\Di\Provider\ContainerProvider;

require dirname(__DIR__) . '/vendor/autoload.php';

$container = new ContainerProvider()();

var_dump($container->has('env.APP_ENV'));
var_dump($container->get('env.APP_ENV'));
```

After `skernel build`, `APP_ENV=production php bin/env.php` prints:

```text
bool(true)
string(10) "production"
```

### Restore the root container

`SuperKernel\Di\Provider\ContainerProvider` is the factory provider of both `SuperKernel\Contract\ContainerInterface` and `Psr\Container\ContainerInterface`. Called with a container, it returns that container unchanged. Called without one, it restores what `skernel build` generated, the package metadata through `super-kernel/metadata` and the annotation records through `super-kernel/annotator`, and returns a new root container. It never scans sources, parses Composer files, or starts a process; it only hashes `composer.lock` to detect a changed installation, so the same restoration works after the project moves and inside a Phar. Each call without a container creates another root, with its own shared services.

The root keeps these services under their `SuperKernel\Contract` interface and under their concrete class: `AnnotationCollectorInterface`, `BasePathInterface`, `PackageManagerInterface`, `ReflectorInterface`, and `VendorDirInterface`. Inject `VendorDirInterface` for the vendor directory; `PackageManagerInterface` does not expose it.

Restoration raises:

- `SuperKernel\Metadata\Exception\BuildRequiredException` when the build stamp, the package metadata, or the annotation records are missing, invalid, written for another build protocol, or made for a different `composer.lock`. Its message names the problem and ends with ``Run `skernel build` to generate the project metadata.``, for example ``The build stamp /srv/app/vendor/.cache/build.json is missing. Run `skernel build` to generate the project metadata.``
- `SuperKernel\Metadata\Exception\MetadataException` when no registered Composer class loader loads `super-kernel/metadata`.
- the exception of a restored Attribute or of the reflector when it cannot be constructed.

`skernel build` prints the classes whose Attributes it could not collect; the container does not report them at start.

In an application, `SuperKernel\Context\ApplicationContext::getContainer()` from `super-kernel/context` calls `ContainerProvider` once and keeps the root for the process; it reports a failure as `SuperKernel\Context\Exception\ContainerInitializationException` with the provider's exception as its previous one.

### Create a container yourself

`new SuperKernel\Di\Container($collector, $entries)` creates a container from any `SuperKernel\Contract\AnnotationCollectorInterface`, which is useful in tests. `$entries` maps identifiers to services that already exist; the container keeps them as if it had created them. Identifiers that are not class or interface names may hold any value, including `null`; a class or interface identifier must hold an instance of that type, or the constructor raises a `ResolutionException`.

Every container reserves these identifiers, which win over supplied entries:

- `SuperKernel\Di\Container`, `SuperKernel\Contract\ContainerInterface`, and `Psr\Container\ContainerInterface` return the container itself;
- `SuperKernel\Contract\AnnotationCollectorInterface` returns the collector;
- `SuperKernel\Di\Contract\DefinitionFactoryInterface`, `SuperKernel\Di\Factory\DefinitionFactory`, `SuperKernel\Di\Contract\ResolverFactoryInterface`, and `SuperKernel\Di\Factory\ResolverFactory` return the container's own factories.

A greeting interface, saved as `src/Greeting/Greeter.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Greeting;

interface Greeter
{
    public function greet(string $name): string;
}
```

A script that supplies a test double for it, saved as `bin/entries.php`:

```php
<?php

declare(strict_types=1);

use Src\Clock\Clock;
use Src\Greeting\Greeter;
use SuperKernel\Contract\AnnotationCollectorInterface;
use SuperKernel\Di\Container;
use SuperKernel\Di\Provider\ContainerProvider;

require dirname(__DIR__) . '/vendor/autoload.php';

$root = new ContainerProvider()();

$greeter = new class implements Greeter {
    public function greet(string $name): string
    {
        return "Hi, {$name}.";
    }
};

$container = new Container($root->get(AnnotationCollectorInterface::class), [
    Greeter::class => $greeter,
    'app.name'     => 'Billing',
]);

echo $container->get(Greeter::class)->greet('SuperKernel'), PHP_EOL;
echo $container->get('app.name'), PHP_EOL;
var_dump($container->get(Clock::class) === $root->get(Clock::class));
```

After `skernel build`, `php bin/entries.php` prints the following; the last line shows that the two containers share nothing:

```text
Hi, SuperKernel.
Billing
bool(false)
```

### Resolve an object the caller owns

The container keeps everything it creates. For an object that should be new each time, such as a value built for one request, obtain its definition and resolve it yourself: the result is not kept, while its dependencies still come from the container and are shared.

An invoice, saved as `src/Billing/Invoice.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Billing;

use Src\Clock\Clock;

final class Invoice
{
    /** @var list<string> */
    public array $lines = [];

    public function __construct(public readonly Clock $clock)
    {
    }
}
```

A script that creates two invoices, saved as `bin/invoice.php`:

```php
<?php

declare(strict_types=1);

use Src\Billing\Invoice;
use Src\Clock\Clock;
use SuperKernel\Di\Contract\DefinitionFactoryInterface;
use SuperKernel\Di\Contract\ResolverFactoryInterface;
use SuperKernel\Di\Provider\ContainerProvider;

require dirname(__DIR__) . '/vendor/autoload.php';

$container = new ContainerProvider()();
$definitions = $container->get(DefinitionFactoryInterface::class);
$resolvers = $container->get(ResolverFactoryInterface::class);

$definition = $definitions->get(Invoice::class)
    ?? throw new RuntimeException('Invoice has no definition.');

$first = $resolvers->resolve($definition);
$second = $resolvers->resolve($definition);

var_dump($first === $second);
var_dump($first->clock === $container->get(Clock::class));
```

After `skernel build`, `php bin/invoice.php` prints:

```text
bool(false)
bool(true)
```

`DefinitionFactoryInterface::get()` returns `null` for an identifier without a definition. This is not a transient scope: every dependency requested while the object is created is kept by the container. Property injection still applies to the object.

## Errors

`get()` raises `SuperKernel\Di\Exception\NotFoundException`, which implements `Psr\Container\NotFoundExceptionInterface`, only when an identifier has no definition. Every other failure raises `SuperKernel\Di\Exception\ResolutionException`, which implements `Psr\Container\ContainerExceptionInterface`, with the original exception as its previous one. A PSR-11 exception thrown by your own constructor, factory, or handler passes through unchanged, and so does a not-found exception for a dependency.

| Situation | Message |
| --- | --- |
| No definition | `Container entry "<id>" has no definition.` |
| Two `Provider`s for one identifier, a self mapping that is not instantiable, a provider that is not a subtype of its target, two `FactoryProvider`s for one identifier, a factory without public construction or its own public non-static `__invoke()`, or a failing definer | `Unable to obtain a definition for container entry "<id>".`, with the reason in a previous exception, such as `Provider <id> is declared by both <class> and <class>.` or `<class> must declare a public non-static __invoke method.` |
| No class or interface in a parameter's type has an entry, and there is no default | `Unable to resolve method parameter <class>::<method>($<parameter>) of type <type>.` |
| More than one class or interface in a parameter's type has an entry | `Method parameter <class>::<method>($<parameter>) is ambiguous: <id>, <id> are available.` |
| A by-reference parameter | `Method parameter <class>::<method>($<parameter>) cannot be resolved because references are unsupported.` |
| A constructor throws | `Unable to construct container entry "<class>".` |
| A factory throws | `Unable to invoke factory "<factory>" for container entry "<id>".` |
| A class or interface identifier resolves to another type | `Container entry "<id>" resolved to <type> instead of an instance of <id>.` |
| An `#[Autowired]` property without a usable type or default | `Unable to resolve property <class>::$<property> of type <type>.` |
| A handled Attribute on a static property | `Static property <class>::$<property> cannot be injected.` |
| A property handler throws or assigns twice | `Property handler <handler> failed while processing <attribute> on <class>::$<property>.` |
| An invalid `#[PropertyHandler]` mapping | `Unable to inspect injectable properties on <class>.`, with the reason in a previous exception |
| No resolver supports a definition | `No resolver supports definition "<definition class>".` |

## Relation to other packages

- [`super-kernel/attribute`](attribute.md) declares `Provider`, `FactoryProvider`, `Autowired`, `PropertyHandler`, `Definer`, and `Resolver`; this package gives them their meaning.
- [`super-kernel/contract`](contract.md) declares `ContainerInterface`, `PropertyAttributeHandlerInterface`, `PropertyContextInterface`, and the services the root keeps. Type your code against these interfaces, not against this package's classes.
- [`super-kernel/metadata`](metadata.md) restores the package metadata, the base path, and the vendor directory, and raises `BuildRequiredException`; [`super-kernel/annotator`](annotator.md) restores the annotation records; [`super-kernel/reflector`](reflector.md) provides the shared reflector.
- [`super-kernel/context`](context.md) keeps the process's root container and creates it through `ContainerProvider`.
- `skernel build` generates the metadata the container restores and prints skipped classes; `skernel run` builds before it starts the application.
- Every other SuperKernel package declares its services with the same Attributes, so after `skernel build` an installed package's services are available without configuration.

## What not to do

**Do not start an application from a stale build.** The container sees only what the last `skernel build` collected. A changed `composer.lock` is detected and raises `BuildRequiredException`, but a new or changed annotated class is not: the container silently keeps the old mappings. Run `skernel build` after every such change, or start with `skernel run`.

**Do not declare two providers for one identifier.** Two `#[Provider]`s, or two `#[FactoryProvider]`s, for the same identifier make the built-in definer fail when the container first needs it, and from then on every lookup of an entry not yet created raises `ResolutionException` and `has()` reports `false`, also for unrelated identifiers. `skernel build` does not report it. Keep exactly one provider per identifier and replace an implementation by changing that one declaration, or by supplying an entry to `new Container()` in a test.

**Do not map an interface or abstract class to itself.** `#[Provider(Self::class)]` on a class that cannot be instantiated is rejected with `Self-provided class <class> must be instantiable.` Annotate the concrete class instead: `#[Provider(TheInterface::class)]`.

**Do not use `FactoryProvider` where a constructor is enough.** A factory that only calls `new` hides the class's dependencies from the container and from readers. Use `#[Provider]` and let the container inject the constructor; keep `FactoryProvider` for construction that needs code. A factory must declare `__invoke()` itself; one inherited from a parent class is rejected.

**Do not build a circular dependency.** The container does not detect cycles: a class that needs itself through its constructor, directly or through other classes, recurses until PHP ends the process with a fatal error, such as exhausted memory, which no `catch` can handle. Move the part both classes need into a third class.

**Do not use an injected property in the constructor.** Properties are set after the constructor returns, so reading one in the constructor raises `Typed property <class>::$<property> must not be accessed before initialization`, wrapped in `Unable to construct container entry "<class>".` Take the dependency as a constructor parameter instead.

**Do not compare a lazy property with `===`.** An `#[Autowired]` property usually holds a proxy, which behaves as the entry but is a different object. Compare the values the objects carry, or look the entry up through the container.

**Do not keep per-request state in a container service.** Every entry lives as long as the container, which in a long-running server is the whole process, and is shared by every request and coroutine. Keep request data in the request, resolve the objects a caller owns as in [Resolve an object the caller owns](#resolve-an-object-the-caller-owns), or inject a factory class that creates them.

**Do not let concurrent work make the first lookup of a service.** The container does not serialize construction: two coroutines that request the same new entry at once may each construct it, and only the first result is kept. Request the services a server shares before it starts its workers or coroutines.

**Do not read `has()` as "can be created".** `has()` reports only that an identifier is kept or has a definition. A class with an unresolvable constructor parameter still reports `true`, and its `get()` fails.

**Do not obtain services by calling providers or `new`.** Calling a provider's `__invoke()`, instantiating a service the container provides, or replacing an injected entry with a manually created object gives the code a second instance that the rest of the application does not share. Receive services through constructor parameters or `#[Autowired]` properties; look services up in a container only at an entry point such as a script or a server callback. In an application, use `ApplicationContext::getContainer()` there instead of calling `new ContainerProvider()()` again, which would restore a second root with its own services.

**Do not ask the container for scalar values by type.** A `string` or `int` parameter is never injected. Give it a default, read it in a `FactoryProvider`, or use a custom property Attribute as in [Custom property Attributes](#custom-property-attributes).
