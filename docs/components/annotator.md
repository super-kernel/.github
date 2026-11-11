<p align="right">
  <strong>English</strong> | <a href="annotator.zh-CN.md">中文文档</a>
</p>

# SuperKernel Annotator

Package: [`super-kernel/annotator`](https://github.com/super-kernel/annotator) · [Packagist](https://packagist.org/packages/super-kernel/annotator) · [Documentation index](../README.md)

## Purpose

`super-kernel/annotator` gives an application the native PHP Attributes of its classes, methods, and properties, and of the classes of its installed packages, without scanning anything when the application starts. `skernel build` collects and constructs the Attributes once; this package publishes them as records below the vendor directory and restores them into an immutable collector that answers queries.

Deciding which classes are collected, constructing the Attributes, and reporting the classes that failed belong to `skernel build`. The package supports native PHP Attributes only, not DocBlock annotations.

## Capabilities

### Find classes and methods by Attribute

`SuperKernel\Contract\AnnotationCollectorInterface` is the collector. A SuperKernel container supplies it, so a service receives it as a constructor parameter, and a script looks it up through `ApplicationContext::getContainer()`.

- `getClassesWithAttribute(Attribute::class)` returns a map from each class carrying that exact Attribute class to the list of its matching entries. Classes come in the build order of their packages and by name within a package.
- `getMethodsWithAttribute(Attribute::class)` returns a map keyed by the declaring class and then by the method name, each with the list of matching entries. Methods of a class are listed in the order of their names.
- Each entry is a `SuperKernel\Contract\AttributeMetadataInterface`. `attribute()` returns the Attribute instance that `skernel build` constructed; `name()` returns its class. A repeatable Attribute gives one entry per declaration, in declaration order.
- An Attribute that violates its target or repeatability fails when `skernel build` constructs it, so it never appears in a result. Use the entries directly: a result already guarantees an instance of the queried class, and the Attribute is never constructed again.

The collector does not check relationships of your own domain, such as a marked command extending Symfony's `Command`; the consumer that gives the Attribute its meaning checks them.

The Attribute, saved as `src/Routing/Route.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Routing;

use Attribute;

#[Attribute(Attribute::TARGET_METHOD | Attribute::IS_REPEATABLE)]
final readonly class Route
{
    public function __construct(public string $method, public string $path)
    {
    }
}
```

A controller that uses it, saved as `src/Http/PageController.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Http;

use Src\Routing\Route;

final class PageController
{
    #[Route('GET', '/')]
    public function home(): string
    {
        return 'Home';
    }

    #[Route('GET', '/about')]
    #[Route('GET', '/about-us')]
    public function about(): string
    {
        return 'About';
    }
}
```

A script that lists the routes, saved as `bin/routes.php`:

```php
<?php

declare(strict_types=1);

use Src\Routing\Route;
use SuperKernel\Context\ApplicationContext;
use SuperKernel\Contract\AnnotationCollectorInterface;

require dirname(__DIR__) . '/vendor/autoload.php';

$collector = ApplicationContext::getContainer()->get(AnnotationCollectorInterface::class);

foreach ($collector->getMethodsWithAttribute(Route::class) as $class => $methods) {
    foreach ($methods as $method => $routes) {
        foreach ($routes as $route) {
            echo $route->attribute()->method, ' ', $route->attribute()->path, ' -> ', $class, '::', $method, PHP_EOL;
        }
    }
}
```

Build the project and run the script:

```bash
skernel build
php bin/routes.php
```

It prints:

```text
GET /about -> Src\Http\PageController::about
GET /about-us -> Src\Http\PageController::about
GET / -> Src\Http\PageController::home
```

### Find Attributes by family

`getClassesWithAttributeAssignableTo(Base::class)` and `getMethodsWithAttributeAssignableTo(Base::class)` return the same shapes as the exact queries, but include every Attribute whose class extends `Base` or implements it. `skernel build` captures each Attribute's parent classes and interfaces, so the application does not discover the family by reflection. Exact queries stay exact: asking `getClassesWithAttribute()` for the base class finds only Attributes of that very class.

The base class of the family, saved as `src/Jobs/Schedule.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Jobs;

abstract readonly class Schedule
{
    abstract public function describe(): string;
}
```

One Attribute of the family, saved as `src/Jobs/Daily.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Jobs;

use Attribute;

#[Attribute(Attribute::TARGET_CLASS)]
final readonly class Daily extends Schedule
{
    public function __construct(public string $at)
    {
    }

    public function describe(): string
    {
        return "daily at {$this->at}";
    }
}
```

Another, saved as `src/Jobs/Hourly.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Jobs;

use Attribute;

#[Attribute(Attribute::TARGET_CLASS)]
final readonly class Hourly extends Schedule
{
    public function describe(): string
    {
        return 'hourly';
    }
}
```

A job, saved as `src/Jobs/SendReports.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Jobs;

#[Daily('06:00')]
final class SendReports
{
}
```

Another job, saved as `src/Jobs/RefreshRates.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Jobs;

#[Hourly]
final class RefreshRates
{
}
```

A script that lists the schedules, saved as `bin/jobs.php`:

```php
<?php

declare(strict_types=1);

use Src\Jobs\Daily;
use Src\Jobs\Schedule;
use SuperKernel\Context\ApplicationContext;
use SuperKernel\Contract\AnnotationCollectorInterface;

require dirname(__DIR__) . '/vendor/autoload.php';

$collector = ApplicationContext::getContainer()->get(AnnotationCollectorInterface::class);

foreach ($collector->getClassesWithAttributeAssignableTo(Schedule::class) as $class => $schedules) {
    echo $class, ' runs ', $schedules[0]->attribute()->describe(), PHP_EOL;
}

echo count($collector->getClassesWithAttribute(Daily::class)), ' daily job', PHP_EOL;
echo count($collector->getClassesWithAttribute(Schedule::class)), ' exact Schedule matches', PHP_EOL;
```

After `skernel build`, `php bin/jobs.php` prints:

```text
Src\Jobs\RefreshRates runs hourly
Src\Jobs\SendReports runs daily at 06:00
1 daily job
0 exact Schedule matches
```

### Read the Attributes of one class or member

- `getClassAttributes($class)`, `getMethodAttributes($class, $method)`, and `getPropertyAttributes($class, $property)` return the list of entries declared there, or an empty list.
- Only members that a class declares are recorded. An inherited method or property is listed under the class that declares it, not under each subclass.
- `types()` of an entry returns the Attribute's class first, followed by the parent classes and interfaces captured at build time, without duplicates.
- `getPackages()` returns the record of each package, keyed by the lowercase Composer package name; `getPackage($name)` accepts the name in any case and returns `null` for an unknown package. Each record is a `SuperKernel\Contract\PackageAnnotationsInterface` with `classAttributes()`, `methodAttributes()`, and `propertyAttributes()`.
- When two packages declare the same class, the first package in build order owns it, also when its record of that class is empty.
- Queries never construct an Attribute again and never load or reflect the annotated classes. Repeated queries return the same instances.

A script that reads single declarations, saved as `bin/inspect.php`:

```php
<?php

declare(strict_types=1);

use Src\Http\PageController;
use Src\Jobs\SendReports;
use SuperKernel\Context\ApplicationContext;
use SuperKernel\Contract\AnnotationCollectorInterface;

require dirname(__DIR__) . '/vendor/autoload.php';

$collector = ApplicationContext::getContainer()->get(AnnotationCollectorInterface::class);

foreach ($collector->getMethodAttributes(PageController::class, 'about') as $metadata) {
    echo $metadata->name(), ' ', $metadata->attribute()->path, PHP_EOL;
}

echo implode(', ', $collector->getClassAttributes(SendReports::class)[0]->types()), PHP_EOL;
```

After `skernel build`, `php bin/inspect.php` prints:

```text
Src\Routing\Route /about
Src\Routing\Route /about-us
Src\Jobs\Daily, Src\Jobs\Schedule
```

### Read the skipped-class diagnostics

When `skernel build` cannot collect the Attributes of a class, for example because its parent class is not installed, it skips the class, prints it, and records one diagnostic row. The collector and each package record also implement `SuperKernel\Contract\ScanDiagnosticsInterface`, whose `getDiagnostics()` returns the rows: a package its own, the collector all of them in build order. Nothing prints the rows when the application starts.

Every row has exactly these fields:

| Field | Value |
| --- | --- |
| `sequence` | A positive integer; the rows of one build run from 1 through N. |
| `package` | The lowercase Composer name of the package that owns the class. |
| `class` | The class name as captured; reading it never loads the class. |
| `exception_type` | The class of the exception that stopped the collection. |
| `message` | The exception message with its original bytes, which may be empty or not UTF-8. |

The rows store no exception object, trace, or timestamp. Messages are not redacted and can contain paths or application data.

A script that prints the rows, saved as `bin/skipped.php`:

```php
<?php

declare(strict_types=1);

use SuperKernel\Context\ApplicationContext;
use SuperKernel\Contract\AnnotationCollectorInterface;
use SuperKernel\Contract\ScanDiagnosticsInterface;

require dirname(__DIR__) . '/vendor/autoload.php';

$collector = ApplicationContext::getContainer()->get(AnnotationCollectorInterface::class);

if ($collector instanceof ScanDiagnosticsInterface) {
    foreach ($collector->getDiagnostics() as $row) {
        echo $row['sequence'], ' ', $row['package'], ' ', $row['class'], ': ', $row['exception_type'], PHP_EOL;
    }
}
```

The rows depend on the installed packages. In a project whose `symfony/console` and `symfony/dotenv` contain classes that need packages it does not install, `php bin/skipped.php` prints:

```text
1 symfony/console Symfony\Component\Console\ConsoleBundle: Error
2 symfony/dotenv Symfony\Component\Dotenv\Command\DotenvDumpCommand: Error
```

### Restore the records without a container

`new SuperKernel\Annotator\Cache\AnnotationCache($vendorDir)` with `read($packages)` restores the collector itself. It takes the `SuperKernel\Contract\VendorDirInterface` and `SuperKernel\Contract\PackageManagerInterface` that the providers of `super-kernel/metadata` supply, and is what `super-kernel/di` calls when it creates the root container.

- Each call reads and validates every record and returns a new collector; nothing is kept between calls.
- It accepts the records only when they were written for exactly the installed packages and their versions.
- It raises `SuperKernel\Annotator\Exception\AnnotationCacheException` and writes nothing to any output.

A script that restores the collector, saved as `bin/restore.php`:

```php
<?php

declare(strict_types=1);

use Src\Routing\Route;
use SuperKernel\Annotator\Cache\AnnotationCache;
use SuperKernel\Metadata\Provider\BasePathProvider;
use SuperKernel\Metadata\Provider\PackageManagerProvider;
use SuperKernel\Metadata\Provider\VendorDirProvider;

require dirname(__DIR__) . '/vendor/autoload.php';

$vendorDir = new VendorDirProvider()();
$packages = new PackageManagerProvider()($vendorDir, new BasePathProvider()($vendorDir));

$collector = new AnnotationCache($vendorDir)->read($packages);

echo count($collector->getMethodsWithAttribute(Route::class)), ' class with routes', PHP_EOL;
```

After `skernel build`, `php bin/restore.php` prints `1 class with routes`.

In a test, `new SuperKernel\Annotator\AnnotationCollector($packages)` creates a collector from any list of `PackageAnnotationsInterface` records, including your own implementations. It keeps the records and entries it receives as they are. Only `PackageAnnotations` records can be published.

### Publish records from a build tool

`skernel build` publishes the records in the project's own process. A build tool that collects Attributes itself publishes with `new SuperKernel\Annotator\Cache\AnnotationPublisher($vendorDir)` and `publish($packages, $records)`:

- `$records` holds exactly one `SuperKernel\Annotator\Model\PackageAnnotations` for every package of `$packages`, with the package's name and `reference()`. The records may come in any order, with unsorted maps; equal content publishes equal files.
- `new PackageAnnotations($name, $reference, $classes, $methods, $properties, $diagnostics)` takes the class Attributes keyed by class, the method and property Attributes keyed by class and member name, and optionally the package's diagnostic rows. The sequences of all records together must run from 1 through N.
- `new SuperKernel\Annotator\Model\AttributeMetadata($attribute, $types)` takes a constructed Attribute and the parent classes and interfaces to capture. It raises `InvalidArgumentException` when the Attribute cannot be serialized and restored, when its object graph is nested more than 64 levels deep, or when it is not an instance of a captured type. Create it inside the code that recovers from a failing class, as `skernel build` does.
- A failed publication removes every annotation record before the exception propagates, so a later `read()` fails instead of returning partial data.

A script that adds one class record to what `skernel build` published, saved as `bin/publish.php`:

```php
<?php

declare(strict_types=1);

use Src\Jobs\Hourly;
use Src\Jobs\Schedule;
use SuperKernel\Annotator\Cache\AnnotationCache;
use SuperKernel\Annotator\Cache\AnnotationPublisher;
use SuperKernel\Annotator\Model\AttributeMetadata;
use SuperKernel\Annotator\Model\PackageAnnotations;
use SuperKernel\Metadata\Provider\BasePathProvider;
use SuperKernel\Metadata\Provider\PackageManagerProvider;
use SuperKernel\Metadata\Provider\VendorDirProvider;

require dirname(__DIR__) . '/vendor/autoload.php';

$vendorDir = new VendorDirProvider()();
$packages = new PackageManagerProvider()($vendorDir, new BasePathProvider()($vendorDir));
$root = $packages->root()->name();

$records = [];

foreach (new AnnotationCache($vendorDir)->read($packages)->getPackages() as $name => $record) {
    assert($record instanceof PackageAnnotations);

    if ($name === $root) {
        $classes = $record->classAttributes();
        $classes['Src\Jobs\PurgeCache'] = [new AttributeMetadata(new Hourly(), [Schedule::class])];

        $record = new PackageAnnotations(
            $name,
            $record->reference(),
            $classes,
            $record->methodAttributes(),
            $record->propertyAttributes(),
            $record->getDiagnostics(),
        );
    }

    $records[] = $record;
}

new AnnotationPublisher($vendorDir)->publish($packages, $records);

$collector = new AnnotationCache($vendorDir)->read($packages);

echo implode(', ', array_keys($collector->getClassesWithAttributeAssignableTo(Schedule::class))), PHP_EOL;
```

After `skernel build`, `php bin/publish.php` prints `Src\Jobs\PurgeCache, Src\Jobs\RefreshRates, Src\Jobs\SendReports`. The next `skernel build` replaces the records again.

### Ship the generated files

The records live below the vendor directory that `VendorDirInterface` names, also when it is not `vendor/`:

```text
<vendor>/.cache/
├── packages.ser
└── annotations/
    ├── super-kernel/metadata.ser
    ├── super-kernel/reflector.ser
    └── vendor/package.ser
```

- `annotations/<package>.ser` holds the record of one package with its diagnostics. `packages.ser` maps every package name to its Composer reference and is written last. `SuperKernel\Annotator\Constants\CacheLayout` names both: `IndexPath` is `packages.ser`, `PackageDirectory` is `annotations`, relative to `<vendor>/.cache/`.
- Restoration loads the classes of the Attributes, including enums they use, but the records never contain the annotated classes.
- The files are trusted local data: restoration deserializes them. Never let an untrusted writer place them.
- Each file is replaced atomically, but a publication is not a transaction across packages and takes no lock. A failed publication removes `packages.ser` and `annotations/` only; the files of `super-kernel/metadata` next to them stay.
- The files are derived data. Ship them with the application, as `skernel package` does, and rebuild them instead of repairing them: a record that cannot be decoded or validated is rejected, never skipped. A Phar must carry records built for the code it contains.

## Errors

`read()` and `publish()` raise `SuperKernel\Annotator\Exception\AnnotationCacheException`, which implements `SuperKernel\Annotator\Exception\AnnotationCollectorExceptionInterface`. When `super-kernel/di` restores the records, it raises `SuperKernel\Metadata\Exception\BuildRequiredException` with the message ``The generated annotation records cannot be restored. Run `skernel build` to generate the project metadata.`` and the annotator exception as its previous one.

| Situation | Message |
| --- | --- |
| `packages.ser` is missing | `The annotation package-reference stub is unavailable.` |
| `packages.ser` cannot be decoded or is invalid | `The annotation package-reference stub cannot be decoded.` or `The annotation package-reference stub is invalid.` |
| The records were built for other packages or versions | `The annotation package-reference stub does not match current Composer metadata.` |
| A package record is missing | `The annotation cache for package <package> is unavailable.` |
| A package record cannot be decoded or is invalid | `The annotation cache for package <package> cannot be decoded.` or `The annotation cache for package <package> is invalid.` |
| Diagnostic sequences are not 1 through N | `Annotation diagnostic sequences must be unique and contiguous.` |
| A published record is not a `PackageAnnotations` | `Every published annotation record must be PackageAnnotations.` |
| Two published records for one package | `Package <package> has more than one published annotation record.` |
| The published records do not cover exactly the installed packages | `Collected annotation packages do not match Composer package metadata.` |
| A published record has another reference than its package | `Collected annotation reference mismatch for package <package>.` |
| An Attribute cannot be serialized and restored, or is nested too deeply | `Attribute <class> must support trusted serialization and restoration.` (`InvalidArgumentException`) |
| An Attribute is not an instance of a captured type | `Attribute <class> is not assignable to captured type <type>.` (`InvalidArgumentException`) |

## Relation to other packages

- [`super-kernel/contract`](contract.md) declares `AnnotationCollectorInterface`, `AttributeMetadataInterface`, `PackageAnnotationsInterface`, and `ScanDiagnosticsInterface`; this package depends only on it. Type your code against these interfaces.
- [`super-kernel/metadata`](metadata.md) supplies the vendor directory and the package manager that restoration and publication check the records against.
- [`super-kernel/di`](di.md) restores the collector when it creates the root container and keeps it under `AnnotationCollectorInterface`; it uses the collector to find providers, definers, resolvers, and property handlers.
- [`super-kernel/attribute`](attribute.md) declares the Attributes that SuperKernel packages look up through the collector.
- `skernel build` chooses the classes, constructs their Attributes, prints the skipped classes, and publishes the records; `skernel package` ships them.

## What not to do

**Do not copy records between installations.** Records carry the reference of every package. Records built for other package versions are rejected with `The annotation package-reference stub does not match current Composer metadata.` Run `skernel build` in each installation, and after every `composer install` or `composer update`.

**Do not repair or hand-edit the generated files.** A record that cannot be decoded or validated is rejected, and the application does not start. Run `skernel build`, which rewrites them.

**Do not publish anything but `PackageAnnotations`, or two records for one package.** `publish()` rejects them with `Every published annotation record must be PackageAnnotations.` and `Package <package> has more than one published annotation record.`, and removes every record already published. Publish one `PackageAnnotations` per installed package.

**Do not expect DocBlock annotations to be collected.** Only native PHP Attributes are recorded; a `@Route` comment is ignored. Declare an Attribute class and use `#[...]`.
