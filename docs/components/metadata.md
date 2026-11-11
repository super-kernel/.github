<p align="right">
  <strong>English</strong> | <a href="metadata.zh-CN.md">中文文档</a>
</p>

# SuperKernel Metadata

Package: [`super-kernel/metadata`](https://github.com/super-kernel/metadata) · [Packagist](https://packagist.org/packages/super-kernel/metadata) · [Documentation index](../README.md)

## Purpose

`super-kernel/metadata` tells an application where it is installed and which packages it consists of, without parsing Composer files or scanning sources when it starts. `skernel build` writes the packages, their environment requirements, and their class maps once, below the vendor directory; this package restores them, checks that they still match the project's `composer.lock`, and serves them through the shared `SuperKernel\Contract` interfaces.

Collecting the metadata belongs to `skernel build`; autoloading stays with Composer's `vendor/autoload.php`. The Attribute records that `skernel build` writes next to the metadata belong to `super-kernel/annotator`.

## Capabilities

### Restore the metadata

Three providers restore the metadata. Each carries the `#[FactoryProvider]` Attribute, so a SuperKernel container resolves the interfaces through them; outside a container, call them in this order:

1. `new VendorDirProvider()()` returns the `SuperKernel\Contract\VendorDirInterface` of the first registered Composer class loader that loads this package. A local directory is canonicalized; a `phar://` path is kept as registered.
2. `new BasePathProvider()($vendorDir)` returns the `SuperKernel\Contract\BasePathInterface`: the project directory that the build recorded, resolved against the vendor directory.
3. `new PackageManagerProvider()($vendorDir, $basePath)` checks the build stamp against the project's `composer.lock` and returns the `SuperKernel\Contract\PackageManagerInterface`.

Each call reads the files again and returns a new value. A container keeps one instance of each interface, and `super-kernel/di` calls the providers itself when it creates the root container, before the container exists.

A class of the project, saved as `src/Billing/Invoice.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Billing;

final class Invoice
{
}
```

A script that restores the metadata, saved as `bin/packages.php`:

```php
<?php

declare(strict_types=1);

use Src\Billing\Invoice;
use SuperKernel\Metadata\Provider\BasePathProvider;
use SuperKernel\Metadata\Provider\PackageManagerProvider;
use SuperKernel\Metadata\Provider\VendorDirProvider;

require dirname(__DIR__) . '/vendor/autoload.php';

$vendorDir = new VendorDirProvider()();
$basePath = new BasePathProvider()($vendorDir);
$packages = new PackageManagerProvider()($vendorDir, $basePath);

echo $packages->packageForClass(VendorDirProvider::class)?->name(), PHP_EOL;
echo $packages->root()->composerType(), PHP_EOL;
var_dump($packages->packageForClass(Invoice::class) === $packages->root());
var_dump($packages->root()->locatedPath() === (string) $basePath);
```

Build the project and run the script:

```bash
skernel build
php bin/packages.php
```

It prints:

```text
super-kernel/metadata
project
bool(true)
bool(true)
```

### Look up packages and classes

`PackageManagerInterface` answers from the restored metadata only:

- `all()` returns the root package first, followed by the installed packages sorted by name; `packages()` returns the installed packages without the root.
- `root()` returns the project; `get($name)` finds the root or an installed package by its name in any case and returns `null` for an unknown one; `ofComposerType($type)` returns the packages whose declared Composer type is exactly `$type`.
- `classes($name)` returns the class map of one package, from class name to absolute file path, or `null` for an unknown package. `allClasses()` returns the class maps of all packages together; when two packages declare the same class, the one earlier in `all()` owns it. `packageForClass($class)` returns the owning package, or `null`.

Each package is a `SuperKernel\Contract\PackageMetadataInterface`:

- `name()` is the lowercase Composer name.
- `version()` is the version exactly as Composer recorded it, or `null` when the root manifest declares none; it is not normalized or compared.
- `reference()` is the distribution reference, falling back to the source reference, or `null`.
- `locatedPath()` is the package directory; for the root it is the project directory.
- The restored packages also expose `composerType()`, which is `library` when the package declares no type, and `environmentRequirements()`.

### Read environment requirements

`environmentRequirements()` of the package manager returns the requirements on Composer platform packages, such as `php` or `ext-swoole`, declared by the root and by every installed package; `hasEnvironmentRequirements()` reports whether there are any. Each is a `SuperKernel\Contract\EnvironmentRequirementInterface`:

- `name()` is the lowercase platform package name, `constraint()` the constraint exactly as declared, and `requiredBy()` the declaring package.
- `isDevRequirement()` is `true` only for a requirement in the root's `require-dev`; the `require-dev` of installed packages is never recorded.

`super-kernel/environment` checks these requirements against the running PHP.

A script that prints the requirements of one package, saved as `bin/requirements.php`:

```php
<?php

declare(strict_types=1);

use SuperKernel\Metadata\Provider\BasePathProvider;
use SuperKernel\Metadata\Provider\PackageManagerProvider;
use SuperKernel\Metadata\Provider\VendorDirProvider;

require dirname(__DIR__) . '/vendor/autoload.php';

$vendorDir = new VendorDirProvider()();
$packages = new PackageManagerProvider()($vendorDir, new BasePathProvider()($vendorDir));

foreach ($packages->environmentRequirements() as $requirement) {
    if ($requirement->requiredBy() === 'super-kernel/metadata') {
        echo $requirement->name(), ' ', $requirement->constraint(), PHP_EOL;
    }
}
```

After `skernel build`, `php bin/requirements.php` prints `php >=8.4.27 <8.5`.

### Use the metadata in a service

In an application, take the interfaces as constructor parameters; the container supplies the instances it keeps.

A service that reports on installed packages, saved as `src/Diagnostics/PackageReport.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Diagnostics;

use SuperKernel\Contract\PackageManagerInterface;
use SuperKernel\Contract\VendorDirInterface;

use function basename;

final readonly class PackageReport
{
    public function __construct(private PackageManagerInterface $packages, private VendorDirInterface $vendorDir)
    {
    }

    public function describe(string $name): string
    {
        $package = $this->packages->get($name);

        if (null === $package) {
            return "{$name} is not installed";
        }

        return $package->name() . ' is installed in ' . basename((string) $this->vendorDir);
    }
}
```

A script that uses it, saved as `bin/report.php`:

```php
<?php

declare(strict_types=1);

use Src\Diagnostics\PackageReport;
use SuperKernel\Context\ApplicationContext;

require dirname(__DIR__) . '/vendor/autoload.php';

$report = ApplicationContext::getContainer()->get(PackageReport::class);

echo $report->describe('Super-Kernel/Metadata'), PHP_EOL;
echo $report->describe('acme/missing'), PHP_EOL;
```

After `skernel build`, `php bin/report.php` prints:

```text
super-kernel/metadata is installed in vendor
acme/missing is not installed
```

### Ship the generated files

`skernel build` writes two files below the vendor directory. `SuperKernel\Metadata\Constants\ArtifactLayout` names them.

| File | Content |
| --- | --- |
| `.cache/build.json` | The build stamp: the build protocol, the SKERNEL version, the SHA-256 of `composer.lock`, and the project directory relative to the vendor directory. |
| `.cache/metadata.ser` | Every package with its class map, with paths relative to the project and vendor directories, protected by a checksum. |

A build stamp looks like this:

```json
{
    "protocol": 1,
    "skernel": "1.0.0-alpha.2",
    "lock": "3f9c5d0e6a1b2c4d8e7f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5",
    "base-path": ".."
}
```

- Ship both files and the project's `composer.lock` with the application; `skernel package` does. Restoration hashes `composer.lock` and refuses metadata built for different contents.
- When the vendor directory lies inside the project, `base-path` is relative, so the project and its vendor directory can move together. Inside a Phar, the paths start with `phar://`. A vendor directory outside the project records the project directory as an absolute path, so that project cannot move.
- A build removes the stamp first and writes it last, so an interrupted build leaves no stamp and restoration refuses it.
- The files are trusted local data: restoration deserializes the records. Never let an untrusted writer place them.
- The stamp does not record whether the build ran with or without development packages. Build again after switching between `composer install` and `composer install --no-dev`.

### Publish metadata from a build tool

`skernel build` publishes the metadata in the project's own process, with the version of this package that the project installs, so the format always matches the reader. `SuperKernel\Metadata\Constants\Protocol::VERSION` names the protocol of that format; `skernel build` builds only a project whose protocol it supports.

`new SuperKernel\Metadata\Build\MetadataPublisher($vendorDir, $basePath)` takes the canonical absolute vendor and project directories. Call its methods in this order:

1. `invalidate()` removes the stamp, then the package records.
2. `publishPackages($manager)` writes the records of a `SuperKernel\Metadata\Manager\PackageManager`, whose root must be located at the project directory.
3. Publish the Attribute records with `super-kernel/annotator`.
4. `commit($skernelVersion)` checks that the records restore and then writes the stamp for the current `composer.lock`. The version, such as `1.0.0-alpha.2` or `dev`, appears only in messages.

Each file is written to a temporary file in the same directory and renamed into place, with the permissions the umask allows.

A script that publishes the restored metadata again, saved as `bin/republish.php`:

```php
<?php

declare(strict_types=1);

use SuperKernel\Metadata\Build\MetadataPublisher;
use SuperKernel\Metadata\Manager\PackageManager;
use SuperKernel\Metadata\Provider\BasePathProvider;
use SuperKernel\Metadata\Provider\PackageManagerProvider;
use SuperKernel\Metadata\Provider\VendorDirProvider;

require dirname(__DIR__) . '/vendor/autoload.php';

$vendorDir = new VendorDirProvider()();
$basePath = new BasePathProvider()($vendorDir);
$packages = new PackageManagerProvider()($vendorDir, $basePath);
assert($packages instanceof PackageManager);

$publisher = new MetadataPublisher($vendorDir, $basePath);
$publisher->invalidate();
$publisher->publishPackages($packages);
$publisher->commit('dev');

echo count(new PackageManagerProvider()($vendorDir, $basePath)->all()), ' packages republished', PHP_EOL;
```

After `skernel build`, `php bin/republish.php` prints the number of packages including the root, such as `33 packages republished`.

## Errors

`SuperKernel\Metadata\Exception\BuildRequiredException` and `SuperKernel\Metadata\Exception\MetadataException` implement `SuperKernel\Metadata\Exception\MetadataExceptionInterface`. Every `BuildRequiredException` message ends with ``Run `skernel build` to generate the project metadata.``, which the table leaves out.

| Situation | Message |
| --- | --- |
| No build stamp | `The build stamp <path> is missing.` |
| The stamp cannot be decoded or has other keys | `The build stamp is invalid.` or `The build stamp has an unexpected structure.` |
| The stamp was written under another protocol | `The metadata was generated by SKERNEL <version> under build protocol <n>, but this project expects <n>.` |
| The project has no `composer.lock` | `The project <path> has no readable composer.lock.` |
| `composer.lock` changed since the build | `The metadata generated by SKERNEL <version> does not match the current composer.lock.` |
| No package records | `The package records <path> are missing.` |
| The package records are damaged | `The package records are invalid.` or `The package records cannot be restored.` |
| No Composer class loader loads this package (`MetadataException`) | `No registered Composer class loader serves super-kernel/metadata.` |
| `commit()` before the records restore (`MetadataException`) | `The published package records do not restore.` |
| `publishPackages()` with a root outside the project (`MetadataException`) | `The root package must be located at the base path.` |

## Relation to other packages

- [`super-kernel/contract`](contract.md) declares `VendorDirInterface`, `BasePathInterface`, `PackageManagerInterface`, `PackageMetadataInterface`, and `EnvironmentRequirementInterface`. Type your code against these interfaces.
- [`super-kernel/attribute`](attribute.md) declares the `#[FactoryProvider]` Attribute on the providers.
- [`super-kernel/di`](di.md) calls the providers when it creates the root container and keeps the three services; it reports missing metadata as this package's `BuildRequiredException`.
- [`super-kernel/annotator`](annotator.md) checks its Attribute records against the package manager and writes them next to the metadata.
- [`super-kernel/environment`](environment.md) checks the environment requirements; [`super-kernel/autoloader`](autoloader.md) uses the class maps.
- `skernel build` collects and publishes the metadata; `skernel package` ships it with `composer.lock`.

## What not to do

**Do not ship the generated files without `composer.lock`.** Restoration compares the stamp with the hash of `composer.lock` and raises `The project <path> has no readable composer.lock.` when the file is missing. Ship `composer.lock` with the application, as `skernel package` does.

**Do not switch between development and production installs without building.** `composer install --no-dev` after a development build leaves `composer.lock` unchanged, so the stale metadata is not detected and still lists the removed packages. Run `skernel build` after every install.

**Do not edit or merge the generated files.** The records carry a checksum and are rejected with `The package records are invalid.`; a changed stamp is rejected or no longer matches. Run `skernel build` instead.

**Do not call `commit()` before the records are published.** It refuses to write a stamp for records that do not restore, raising `The published package records do not restore.`, and the application cannot start until a complete build. Call `invalidate()`, `publishPackages()`, publish the Attribute records, and `commit()`, in that order.
