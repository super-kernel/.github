<p align="right">
  <strong>English</strong> | <a href="framework.zh-CN.md">中文文档</a>
</p>

# SuperKernel Framework

Package: [`super-kernel/framework`](https://github.com/super-kernel/framework) · [Packagist](https://packagist.org/packages/super-kernel/framework) · [Documentation index](../README.md)

## Purpose

`super-kernel/framework` provides the application that an entry script runs. The container creates it as `SuperKernel\Contract\ApplicationInterface`: a Symfony Console application named after the root package, connected to the application's PSR-14 event dispatcher and to the shared exception dispatcher.

The package decides neither which commands exist nor what a command does. Commands are registered by `super-kernel/command`; servers, protocol callbacks, and loggers belong to the packages that provide them, and the package does not depend on Swoole. Exception handlers are declared with `super-kernel/exception-dispatcher`.

## Capabilities

### Run the application

The shared interface declares only `run(): int`, so code typed against `ApplicationInterface` needs no Symfony class. The object is a `SuperKernel\Framework\Application`, which extends `Symfony\Component\Console\Application` and keeps its input, output, command, and execution APIs, including the optional input and output parameters of `run()`.

An entry script resolves the application from the container and runs it. Entry script, saved as `bin/bootstrap.php`:

```php
#!/usr/bin/env php
<?php

declare(strict_types=1);

use SuperKernel\Context\ApplicationContext;
use SuperKernel\Contract\ApplicationInterface;

require dirname(__DIR__) . '/vendor/autoload.php';

exit(ApplicationContext::getContainer()->get(ApplicationInterface::class)->run());
```

Symfony Console's auto-exit is kept: `run()` ends the process with the command's status. A test or another caller that embeds the application turns auto-exit off with `setAutoExit(false)` on the concrete class, and passes its own input and output.

A script that runs the application and reads its output, saved as `bin/version.php`:

```php
<?php

declare(strict_types=1);

use SuperKernel\Context\ApplicationContext;
use SuperKernel\Contract\ApplicationInterface;
use Symfony\Component\Console\Application;
use Symfony\Component\Console\Input\ArrayInput;
use Symfony\Component\Console\Output\BufferedOutput;

require dirname(__DIR__) . '/vendor/autoload.php';

$application = ApplicationContext::getContainer()->get(ApplicationInterface::class);
assert($application instanceof Application);

$application->setAutoExit(false);

$output = new BufferedOutput();
$status = $application->run(new ArrayInput(['--version' => true]), $output);

echo trim($output->fetch()), " (status {$status})", PHP_EOL;
```

Build the project and run the script:

```bash
skernel build
php bin/version.php
```

It prints:

```text
acme/billing 1.2.3 (status 0)
```

### Name and version the application

The root `composer.json` names the application. Its `name` becomes the application name and its `version`, when present, the application version:

```json
{
    "name": "acme/billing",
    "version": "1.2.3"
}
```

Build the project and ask for the version:

```bash
skernel build
php bin/bootstrap.php --version
```

It prints:

```text
acme/billing 1.2.3
```

The name and version also appear in the headers of `list` and `help` and in the identity fields of Symfony's descriptors. Without a `version`, Symfony's default is kept and only the name is shown; the `super-kernel/skeleton` template declares no version, so its `--version` prints `super-kernel/skeleton`.

Both values are used exactly as written: the package does not trim, validate, normalize, or warn about them, and it never takes them from Git, installed packages, `InstalledVersions`, or `COMPOSER_ROOT_VERSION`. No class, Attribute, or main method marks the application. Symfony decides how they are shown: a version equal to `UNKNOWN` is hidden, an empty version keeps the separator, and formatter markup follows Symfony's output rules, while `getVersion()` returns the string as written. Use values that Composer accepts and that you want your console to show.

The values come from the metadata that `skernel build` generated, both when the application starts from source and when it starts from a packaged application. After changing `name` or `version`, run `skernel build` again, or start with `skernel run`, which builds first; until then the application keeps the previous identity.

### Configure the application before it runs

The container creates the application once, the first time `ApplicationInterface` is resolved. It sets the name and version, installs Symfony's event dispatcher, and publishes `SuperKernel\Event\BeforeApplicationStart` synchronously with that application before it returns it; it never calls `run()`. A listener of that event configures the application with Symfony's setters, as `super-kernel/command` does to install its command loader. An exception from a listener propagates unchanged, and the application is not returned. The [`super-kernel/event`](event.md) page shows such a listener.

An application that provides its own `ApplicationInterface` is responsible for the behavior of its implementation; none of the behavior on this page applies to it.

### Handle command failures

`Application` sends every throwable that reaches Symfony's error rendering to `SuperKernel\Contract\ExceptionDispatcherInterface`, PHP errors included. The run still ends with a non-zero status. The exception is Symfony Console's own exceptions, which implement `Symfony\Component\Console\Exception\ExceptionInterface`, such as an unknown command, a missing argument, or an unknown option: Symfony renders those on the error output with the message, the running command's synopsis when a command is running, and the exception class and trace at verbose level. Exception handlers never receive them.

A command that fails, saved as `src/Command/ImportCommand.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Command;

use RuntimeException;
use SuperKernel\Command\Attribute\Command;
use Symfony\Component\Console\Command\Command as SymfonyCommand;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

#[Command(name: 'invoice:import', description: 'Import invoices.')]
final class ImportCommand extends SymfonyCommand
{
    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        throw new RuntimeException('The invoice feed is unavailable.');
    }
}
```

An exception handler, saved as `src/ExceptionHandler/ReportFailure.php`:

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

Build the project, run the command, and print its status:

```bash
skernel build
php bin/bootstrap.php invoice:import; echo "status $?"
```

It prints:

```text
Failed: The invoice feed is unavailable.
status 1
```

Without a matching handler, `super-kernel/exception-dispatcher` writes the complete exception through the stdout logger. The [`super-kernel/exception-dispatcher`](exception-dispatcher.md) page describes handlers.

### Handle throwables that nothing catches

When the application starts, the package registers `SuperKernel\Framework\ExceptionHandler\ProcessExceptionHandler` as PHP's exception handler for the process. It listens to `BeforeApplicationStart` at priority `PHP_INT_MAX`, before every ordinary listener, and registers at most once per process. A throwable that nothing catches afterwards is sent to the same exception dispatcher, and the process then exits with status `1`. The handler does not handle warnings, does not resume execution, and decides no protocol-specific action. An entry script needs no code for it.

`handle()` sends a throwable to the dispatcher without ending the process.

There is no exception handling before the container and the exception dispatcher are ready: a failure at that point is a startup bug to repair, and PHP reports it.

A script that fails after the application is created, saved as `bin/late-failure.php`:

```php
<?php

declare(strict_types=1);

use SuperKernel\Context\ApplicationContext;
use SuperKernel\Contract\ApplicationInterface;

require dirname(__DIR__) . '/vendor/autoload.php';

ApplicationContext::getContainer()->get(ApplicationInterface::class);

throw new RuntimeException('The invoice archive is locked.');
```

Run the script and print its status:

```bash
php bin/late-failure.php; echo "status $?"
```

It prints:

```text
Failed: The invoice archive is locked.
status 1
```

### Listen to Symfony Console events

The container provides `Symfony\Contracts\EventDispatcher\EventDispatcherInterface` through `SuperKernel\Framework\EventDispatcher\SymfonyEventDispatcher`, which passes every event object unchanged to `Psr\EventDispatcher\EventDispatcherInterface` and returns its result. The application uses it, so Symfony Console events, such as `ConsoleCommandEvent`, `ConsoleErrorEvent`, and `ConsoleTerminateEvent`, reach `#[Listener]` classes.

Symfony's optional string event name is ignored: listeners are selected by the event object's class or interface, and subscriptions by event name are not supported.

A listener of the command event, saved as `src/Listener/AnnounceCommand.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Listener;

use SuperKernel\Attribute\Listener;
use SuperKernel\Contract\ListenerInterface;
use Symfony\Component\Console\Event\ConsoleCommandEvent;

#[Listener]
final readonly class AnnounceCommand implements ListenerInterface
{
    public function listen(): array
    {
        return [
            ConsoleCommandEvent::class,
        ];
    }

    public function process(object $event): void
    {
        assert($event instanceof ConsoleCommandEvent);

        $event->getOutput()->writeln('Running ' . $event->getCommand()?->getName());
    }
}
```

Build the project and run the command again:

```bash
skernel build
php bin/bootstrap.php invoice:import
```

It prints:

```text
Running invoice:import
Failed: The invoice feed is unavailable.
```

### Create the application without the container

`SuperKernel\Framework\Provider\ApplicationProvider` is the factory the container calls. An explicit assembly boundary or a test that has the three services can create the application directly with `new ApplicationProvider()($events, $exceptions, $packageManager)`, where the arguments are Symfony's `EventDispatcherInterface`, `SuperKernel\Contract\ExceptionDispatcherInterface`, and `SuperKernel\Contract\PackageManagerInterface`. Each call creates a new application and publishes `BeforeApplicationStart` for it. Everywhere else, receive the application from the container.

## Relation to other packages

- [`super-kernel/contract`](contract.md) declares `ApplicationInterface`, `ExceptionDispatcherInterface`, `PackageManagerInterface`, and `ListenerInterface`; the package implements or consumes them and depends on no implementation.
- [`super-kernel/event`](event.md) declares `BeforeApplicationStart`, which this package publishes.
- [`super-kernel/di`](di.md) and [`super-kernel/context`](context.md) create the container that creates the application; [`super-kernel/metadata`](metadata.md) supplies the root package's name and version from the build metadata.
- [`super-kernel/event-dispatcher`](event-dispatcher.md) provides the PSR-14 dispatcher behind Symfony's dispatcher contract and discovers the listeners.
- [`super-kernel/exception-dispatcher`](exception-dispatcher.md) provides the exception dispatcher and its handlers; [`super-kernel/logger`](logger.md) provides the stdout logger it falls back to.
- [`super-kernel/command`](command.md) listens to `BeforeApplicationStart` to install its command loader.
- `skernel build` generates the metadata; `skernel run` builds and then runs the application.

## What not to do

**Do not call `ApplicationProvider` to obtain the application.** Each call creates a second application with its own startup event, which the rest of the code does not share. Receive `ApplicationInterface` from the container, and keep direct calls to an explicit assembly boundary or a test.

**Do not subscribe to Symfony Console events by name.** The event name is ignored, so a listener that expects names such as `console.command` never matches. List the event class, such as `ConsoleCommandEvent::class`, in `listen()`.

**Do not expect failures before the container is ready to be handled.** The exception dispatcher and the process handler exist only after the container creates the application. A failure earlier, such as missing build metadata, is reported by PHP; repair its cause.
