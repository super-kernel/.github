<p align="right">
  <strong>English</strong> | <a href="mimic.zh-CN.md">中文文档</a>
</p>

# SuperKernel Mimic

Package: [`super-kernel/mimic`](https://github.com/super-kernel/mimic) · [Packagist](https://packagist.org/packages/super-kernel/mimic) · [Documentation index](../README.md)

## Purpose

`super-kernel/mimic` turns a PHP source file into a transformed copy that PHP can load, and keeps that copy for the next process. A transformation, a `SuperKernel\Mimic\Contract\MutatorInterface`, rewrites the source; the package reads a snapshot of the file, applies the transformation, and returns where the result can be loaded: a proxy file in the vendor directory's cache or, when that is not possible, a read-only `mimic://` stream.

The package does not decide which files to transform, does not register autoloaders, and does not run the code it prepares; the integration that uses it does. `super-kernel/aop` is such an integration: it supplies the transformation that weaves methods and loads the prepared classes.

## Capabilities

### Prepare a source file

`SuperKernel\Mimic\MimicFactory` is provided by the container. `prepare($sourcePath)` reads one snapshot of the file, a local path or a path inside a Phar archive, and returns a loadable address. It tries the file cache first and falls back to a stream. Load the address with `require`; until then, nothing of the file has run.

The factory needs three services: `SuperKernel\Contract\VendorDirInterface` for the cache location, `SuperKernel\Contract\PackageManagerInterface` for class names and package locations, both restored from the metadata of `skernel build`, and a `MutatorInterface`. `super-kernel/aop` binds its own transformation; without it, declare one. Source files must be trusted PHP.

A class with a placeholder, saved as `src/Billing/Edition.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Billing;

final class Edition
{
    public const string NAME = '%EDITION%';
}
```

A transformation that fills the placeholder, saved as `src/Mimic/FillEdition.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Mimic;

use SuperKernel\Attribute\Provider;
use SuperKernel\Mimic\Contract\MutatorInterface;

use function str_replace;

#[Provider(MutatorInterface::class)]
final readonly class FillEdition implements MutatorInterface
{
    public function getCacheKey(): string
    {
        return 'fill-edition-1';
    }

    public function mutate(string $code, ?string $sourcePath = null): string
    {
        return str_replace('%EDITION%', 'Community', $code);
    }
}
```

A script that prepares and loads the class, saved as `bin/edition.php`:

```php
<?php

declare(strict_types=1);

use Src\Billing\Edition;
use SuperKernel\Context\ApplicationContext;
use SuperKernel\Contract\BasePathInterface;
use SuperKernel\Mimic\MimicFactory;

require dirname(__DIR__) . '/vendor/autoload.php';

$container = ApplicationContext::getContainer();
$source = $container->get(BasePathInterface::class) . '/src/Billing/Edition.php';

$address = $container->get(MimicFactory::class)->prepare($source);

require $address;

echo basename($address), PHP_EOL;
echo Edition::NAME, PHP_EOL;
```

Build the project and run the script twice:

```bash
skernel build
php bin/edition.php
php bin/edition.php
```

It prints:

```text
Src.Billing.Edition.tpl
Community
Src.Billing.Edition.tpl
Community
```

The second run reuses the proxy that the first one wrote.

### Reuse the proxy cache

Proxies are written directly to `vendor/.cache/mimic/<class-name>.tpl`, where the class name is the fully qualified name of the class the file declares, with namespace separators replaced by dots, so `Acme\Service\Test` and `Acme\Service_Test` stay distinct. The package manager provides the name without loading the file; when a file declares several mapped classes, the first name in its sorted class map names the proxy. The address returned for a new proxy and for a cached one is the same.

The first line of a proxy is `<?php // mimic-cache-sha256: <digest>`. The digest covers the source's location, the transformation's cache key, and the hash of the source content. The location is the owning package's name and the file's path inside that package, so a proxy stays valid when the project moves or is packaged into a Phar archive; the original path is never written into the proxy. Modification times, inodes, and sizes are never compared.

- A matching proxy is reused without writing, creating directories, or needing the stream protocol.
- Changed content, a file moved within its package, or a changed cache key replaces the same proxy.
- A new proxy is written to a temporary file in the cache directory and renamed into place, without lock files. Concurrent processes may transform the same file twice, but no reader sees a partial proxy, and OPcache is invalidated after a replacement when it is available.
- A prepared file path follows later replacements; keep `mimic://` addresses when you need snapshots that never change.
- The package never reads, renames, or deletes other files in the cache, such as older hash directories, `.php` or `.php.tpl` proxies, or `.lock` files.

Each class has one proxy in a vendor directory, so every integration that shares the cache must use the same source location and transformation.

### Fall back to a stream

`prepare()` returns a `mimic://` address when the cache cannot be created, read, written, or replaced, or when the file is not part of the project's packages. An existing proxy is kept when a replacement fails, and a proxy whose first line cannot be read or recognized is left as it is. The stream protocol is registered when it is first needed; when another library already owns `mimic`, the fallback fails. The stream address includes the source path, the content hash, and the cache key, so preparing another revision or transformation never changes an earlier stream. Streams live in the memory of the current process.

A script that prepares a file outside the project's packages, saved as `bin/snapshot.php`:

```php
<?php

declare(strict_types=1);

use SuperKernel\Context\ApplicationContext;
use SuperKernel\Mimic\MimicFactory;

require dirname(__DIR__) . '/vendor/autoload.php';

$source = sys_get_temp_dir() . '/edition-banner.php';
file_put_contents($source, "<?php\n\nreturn 'Edition: %EDITION%';\n");

$address = ApplicationContext::getContainer()->get(MimicFactory::class)->prepare($source);

echo strtok($address, ':'), PHP_EOL;
echo require $address, PHP_EOL;
```

Run the script:

```bash
php bin/snapshot.php
```

It prints:

```text
mimic
Edition: Community
```

### Write a transformation

`MutatorInterface` has two methods:

- `mutate($code, $sourcePath)` returns the complete new source, starting with `<?php`; an exception it throws propagates unchanged from `prepare()`;
- `getCacheKey()` returns a non-empty string that identifies the transformation, its revision, and every setting that affects its output. Equivalent instances return the same key; change it whenever the same source must produce different code.

The output may depend on where the source lives only through its package-relative location. Bind absolute paths when the code runs instead of writing them into the output, or the proxy stops matching the source after the project moves or is packaged. Both drivers receive the same transformation from the container. The package has no runtime dependency on PHP-Parser; a transformation that parses PHP requires its parser itself.

### Use the drivers directly

`MimicFactory` combines two drivers that implement `SuperKernel\Mimic\Contract\MimicDriverInterface`, whose `prepare()` takes a `SuperKernel\Mimic\Source\SourceFile`. A `SourceFile` holds the canonical local path, or a Phar path as given, the source bytes, and their SHA-256 hash. `SuperKernel\Mimic\Driver\FileDriver` returns a proxy path, or `null` when the cache cannot be used; `SuperKernel\Mimic\Driver\StreamDriver` returns a `mimic://` address or throws.

### Restore exception locations

`SuperKernel\Mimic\Diagnostics\ExceptionPatcher::patch($throwable, $originalFile, $originalLine)` sets the file and line of a throwable raised from generated code to the original location, keeping the same throwable object, and returns it. The package itself never replaces the global exception handler.

### Load changed code

A class that a process has loaded cannot be replaced or unloaded. Restart long-running workers to load newly prepared code, and run `skernel build` again after changing classes or packages, so that the class names and package locations, and the proxies prepared at build time, match the source.

## Errors

| Situation | Message |
| --- | --- |
| A source path that is empty, contains a null byte, or is not a readable file | `The source file is not readable: <path>` |
| The `mimic` protocol cannot be registered | `Unable to register the mimic stream protocol.` |
| Another library owns the `mimic` protocol | `The mimic stream protocol is already registered by another library.` |

The first is an `InvalidArgumentException`, the others are `RuntimeException`s. Exceptions from the transformation, including parse errors, propagate unchanged.

## Relation to other packages

- [`super-kernel/aop`](aop.md) binds its `Mutator` to `MutatorInterface` and loads the classes it weaves through this package; `skernel build` prepares their proxies.
- [`super-kernel/metadata`](metadata.md) provides `VendorDirInterface` and `PackageManagerInterface` from the build metadata; [`super-kernel/contract`](contract.md) declares them.
- [`super-kernel/di`](di.md) creates `MimicFactory` and both drivers.
- The package ports [wheakerd/mimic](https://github.com/wheakerd/mimic) at revision `443ad3e4a2c8dd9ce7b6028b0b7aea163be4e47d`, with its owner's authorization: the Composer package `wheakerd/php-mimic` became `super-kernel/mimic`, and the namespace `PhpMimic\` became `SuperKernel\Mimic\`.

## What not to do

**Do not keep the cache key when the output changes.** A proxy whose source and key are unchanged is reused as it is, so a transformation that changes its output without changing `getCacheKey()` keeps loading the old code. Change the key with every revision that changes the generated code.

**Do not write absolute paths into the generated code.** The proxy is reused wherever the package-relative location matches, including after the project moves or runs from a Phar archive, so an embedded path points to a place that no longer exists. Bind paths when the code runs, as `super-kernel/aop` does with its source constants.

**Do not share one cache between different transformations.** Each class has one proxy per vendor directory, so two integrations that prepare the same class with different transformations replace each other's proxy on every run. Use one transformation per cache, or use `StreamDriver` for independent snapshots.
