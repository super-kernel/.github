<p align="right">
  <strong>English</strong> | <a href="logger.zh-CN.md">中文文档</a>
</p>

# SuperKernel Logger

Package: [`super-kernel/logger`](https://github.com/super-kernel/logger) · [Packagist](https://packagist.org/packages/super-kernel/logger) · [Documentation index](../README.md)

## Purpose

`super-kernel/logger` decides where SuperKernel's own log records go and gives applications a short way to create Monolog loggers. The container provides `SuperKernel\Contract\StdoutLoggerInterface`, a PSR-3 logger that writes readable lines to the standard output; SuperKernel's packages, including the exception fallback of `super-kernel/exception-dispatcher`, write through it. `LoggerFactory` creates further loggers for the streams an application chooses.

Which logger an application uses for its own records, and how it is configured per deployment, stays with the application: the package does not bind `Psr\Log\LoggerInterface`.

## Capabilities

### Write to the standard output logger

Receive `StdoutLoggerInterface` through a constructor and call any PSR-3 method. The default service writes on the `super-kernel` channel to `php://stdout`, keeps `INFO` and above, and formats each record with `ConsoleFormatter`. Records reach Monolog unchanged, and the caller chooses the level.

Exception records use the same service: the caller picks the error level and formats the throwable, and there is no separate exception logger, channel, or standard error stream.

A service that logs, saved as `src/Billing/InvoiceImporter.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Billing;

use SuperKernel\Contract\StdoutLoggerInterface;

final readonly class InvoiceImporter
{
    public function __construct(private StdoutLoggerInterface $logger)
    {
    }

    public function import(string $invoiceNumber): void
    {
        $this->logger->debug("Reading invoice {$invoiceNumber}.");
        $this->logger->info("Invoice {$invoiceNumber} imported.");
    }
}
```

A script that uses it, saved as `bin/import.php`:

```php
<?php

declare(strict_types=1);

use Src\Billing\InvoiceImporter;
use SuperKernel\Context\ApplicationContext;

require dirname(__DIR__) . '/vendor/autoload.php';

ApplicationContext::getContainer()->get(InvoiceImporter::class)->import('INV-1001');
```

Build the project and run the script:

```bash
skernel build
php bin/import.php
```

It prints a line such as:

```text
2026-11-10T10:56:25.283+00:00 [INFO] Invoice INV-1001 imported.
```

The `DEBUG` record is below the threshold and is not written.

### Read the console format

`SuperKernel\Logger\Formatter\ConsoleFormatter` writes one line per record: a sortable ISO 8601 timestamp with milliseconds and offset, the PSR-3 level in square brackets, exactly one space, and the message, followed by one newline. It omits context and extra data, and has no channel, process, or thread column. The layout is inspired by Spring Boot's console output:

```text
2026-08-18T03:05:46.007+00:00 [NOTICE] Swoole main server started in base mode.
2026-08-18T03:05:46.008+00:00 [INFO] Swoole listener "http" started on 127.0.0.1:9501 (http).
```

The message is written as given. Placeholders such as `{workerId}` are not replaced from the context, so build the sentence you want to show before you log it. Structured output belongs on a logger of its own with another formatter, such as one created by `LoggerFactory` without a formatter.

`new ConsoleFormatter()` writes plain text. `new ConsoleFormatter(decorated: true)` adds ANSI colors: the timestamp is faint, error levels and above are red, `WARNING` is yellow, `NOTICE` is cyan, and `INFO` and `DEBUG` are green; messages are never colored. `isDecorated()` reports the choice. The standard output logger decorates only when the standard output is an interactive terminal, `NO_COLOR` is not set, and `TERM` is not `dumb`, so redirected output stays plain.

### Create a logger

`SuperKernel\Logger\Factory\LoggerFactory::create()` returns a `Psr\Log\LoggerInterface` backed by a synchronous Monolog logger with one stream handler:

- `channel` and `stream` are required; the stream is a path or a PHP stream such as `php://stderr`;
- `minimumLevel` is a PSR-3 level, `LogLevel::INFO` by default; an unknown level raises `Psr\Log\InvalidArgumentException`;
- `formatter` is an optional Monolog formatter; without one, Monolog's default line format is used.

Every call creates a new logger; keeping one is the container's job. Monolog opens the stream when the first record is written, so a stream that cannot be opened or written fails then, with Monolog's exception. The factory does not read environment files, redact sensitive context, rotate files, buffer records, send records to remote services, or deliver them asynchronously or coroutine-safely; the service that uses the factory decides those policies.

The package binds no `Psr\Log\LoggerInterface`. An application that wants one declares it, for example with a factory.

A provider of the application's `LoggerInterface`, saved as `src/Logger/AuditLoggerProvider.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Logger;

use Psr\Log\LoggerInterface;
use Psr\Log\LogLevel;
use SuperKernel\Attribute\FactoryProvider;
use SuperKernel\Contract\BasePathInterface;
use SuperKernel\Logger\Factory\LoggerFactory;

#[FactoryProvider(LoggerInterface::class)]
final readonly class AuditLoggerProvider
{
    public function __invoke(LoggerFactory $factory, BasePathInterface $basePath): LoggerInterface
    {
        return $factory->create(
            channel: 'audit',
            stream: $basePath . '/audit.log',
            minimumLevel: LogLevel::WARNING,
        );
    }
}
```

A script that writes to it, saved as `bin/audit.php`:

```php
<?php

declare(strict_types=1);

use Psr\Log\LoggerInterface;
use SuperKernel\Context\ApplicationContext;

require dirname(__DIR__) . '/vendor/autoload.php';

$audit = ApplicationContext::getContainer()->get(LoggerInterface::class);

$audit->info('Invoice INV-1001 viewed.');
$audit->warning('Invoice INV-1001 deleted.', ['user' => 'ada']);
```

Build the project and run the script:

```bash
skernel build
php bin/audit.php
```

It prints nothing; `audit.log` in the project directory then holds one line such as:

```text
[2026-11-10T10:56:25.306140+00:00] audit.WARNING: Invoice INV-1001 deleted. {"user":"ada"} []
```

### Replace the standard output logger

Every SuperKernel package writes through the registered `StdoutLoggerInterface`, so a replacement sets the threshold, format, and stream of all of their records, and of the application's records that use the same service. Declare the replacement as a class with `#[Provider(StdoutLoggerInterface::class)]`; the package's own `#[FactoryProvider]` then stays unused. Do not declare a second `#[FactoryProvider]` for the interface: two factories for one identifier conflict. Inject configuration or environment services into the replacement when the level or stream depends on the deployment.

A replacement that writes `WARNING` and above to the standard error, saved as `src/Logger/AppLogger.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Logger;

use Psr\Log\LoggerInterface;
use Psr\Log\LoggerTrait;
use Psr\Log\LogLevel;
use Stringable;
use SuperKernel\Attribute\Provider;
use SuperKernel\Contract\StdoutLoggerInterface;
use SuperKernel\Logger\Factory\LoggerFactory;
use SuperKernel\Logger\Formatter\ConsoleFormatter;

#[Provider(StdoutLoggerInterface::class)]
final readonly class AppLogger implements StdoutLoggerInterface
{
    use LoggerTrait;

    private LoggerInterface $logger;

    public function __construct(LoggerFactory $factory)
    {
        $this->logger = $factory->create(
            channel: 'application',
            stream: 'php://stderr',
            minimumLevel: LogLevel::WARNING,
            formatter: new ConsoleFormatter(),
        );
    }

    public function log(mixed $level, string|Stringable $message, array $context = []): void
    {
        $this->logger->log($level, $message, $context);
    }
}
```

Build the project and run the import script again:

```bash
skernel build
php bin/import.php
```

It prints nothing: the `INFO` record is below the replacement's threshold. A `WARNING` record would appear on the standard error. To keep the default threshold and destination, use `LogLevel::INFO` and `php://stdout`.

### Keep machine-readable output clean

The default logger writes to the standard output, so its lines can appear before or between a command's own output. When the standard output must be a machine-readable protocol, replace the logger with one that writes to `php://stderr` or a file, as above. There is no command-specific switch; the replacement applies to every record of the process.

## Relation to other packages

- [`super-kernel/contract`](contract.md) declares `StdoutLoggerInterface`; every package that logs depends on that interface, not on this package.
- [`super-kernel/exception-dispatcher`](exception-dispatcher.md) writes the exceptions that no handler takes, and handler failures, through the standard output logger.
- [`super-kernel/di`](di.md) creates the logger and keeps it; [`super-kernel/attribute`](attribute.md) declares the `#[Provider]` and `#[FactoryProvider]` used to replace or add loggers.
- Monolog provides the loggers, handlers, and formatters.

## What not to do

**Do not replace the standard output logger with a `#[FactoryProvider]`.** The package already declares one for `StdoutLoggerInterface`, so a second one makes every container lookup that needs a new definition fail with `Factory provider for SuperKernel\Contract\StdoutLoggerInterface is declared by both <class> and SuperKernel\Logger\Provider\StdoutLoggerProvider.` Use `#[Provider(StdoutLoggerInterface::class)]` on the replacement class.

**Do not rely on placeholders in the console format.** `$logger->warning('Worker {workerId} stopped.', ['workerId' => 7])` prints `Worker {workerId} stopped.` literally, and the context is not shown. Put the values into the message, or use a logger with a formatter or processor that renders them.
