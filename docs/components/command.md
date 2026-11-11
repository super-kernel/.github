<p align="right">
  <strong>English</strong> | <a href="command.zh-CN.md">中文文档</a>
</p>

# SuperKernel Command

Package: [`super-kernel/command`](https://github.com/super-kernel/command) · [Packagist](https://packagist.org/packages/super-kernel/command) · [Documentation index](../README.md)

## Purpose

`super-kernel/command` turns classes into Symfony Console commands of the application. A class declares its command with the `#[Command]` Attribute; the package registers the names it declares and gives the application a command loader that creates each command through the container only when the command is used.

Symfony Console runs the commands and owns their input, output, and lazy wrappers. The application itself comes from `super-kernel/framework`, and the container from `super-kernel/di`, which owns each command's lifetime.

## Capabilities

### Declare a command

A command is a class that extends `Symfony\Component\Console\Command\Command` and carries `SuperKernel\Command\Attribute\Command`. Both are named `Command`, so import Symfony's class under an alias such as `SymfonyCommand`. The Attribute takes:

- `name`: the primary name, required and non-empty;
- `description`: shown by `list` and `help`, `null` by default;
- `aliases`: a list of further non-empty names, empty by default;
- `hidden`: hides the command from `list` when `true`, `false` by default.

Every name and alias must be unique across the application, also within one declaration. Only the `#[Command]` Attribute on a class registers a command: Symfony's `#[AsCommand]` is ignored, and the Attribute cannot be placed on a method. Which package declares a class does not matter.

A command, saved as `src/Command/HelloCommand.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Command;

use SuperKernel\Command\Attribute\Command;
use Symfony\Component\Console\Command\Command as SymfonyCommand;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

#[Command(name: 'app:hello', description: 'Print a greeting.', aliases: ['hello'])]
final class HelloCommand extends SymfonyCommand
{
    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $output->writeln('Hello');

        return self::SUCCESS;
    }
}
```

Build the project and run the command by its name and by its alias:

```bash
skernel build
php bin/bootstrap.php app:hello
php bin/bootstrap.php hello
```

It prints:

```text
Hello
Hello
```

### Inject services into a command

The container creates a command, so the command receives its services through its constructor, like any other class the container creates. Call the parent constructor without arguments: the name, description, and aliases come from the Attribute.

Creation is lazy. Discovery creates no command; `list` and the suggestions for an unknown name read the declarations only, so they work even when a command's dependencies cannot be resolved. Symfony initializes a command to run it, show its help, or complete its input, and only then does the container create it. A failure to create it, such as an unresolvable constructor parameter, is raised at that moment: the application sends it to the exception dispatcher, and the run ends with a non-zero status. The container keeps the command it created, as it keeps every entry.

A service, saved as `src/Billing/InvoiceRepository.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Billing;

use function count;

final class InvoiceRepository
{
    /** @var list<string> */
    private array $open = ['INV-1001', 'INV-1002'];

    public function countOpen(): int
    {
        return count($this->open);
    }
}
```

A command that uses it, saved as `src/Command/CountInvoicesCommand.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Command;

use Src\Billing\InvoiceRepository;
use SuperKernel\Command\Attribute\Command;
use Symfony\Component\Console\Command\Command as SymfonyCommand;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

#[Command(name: 'invoice:count', description: 'Count open invoices.')]
final class CountInvoicesCommand extends SymfonyCommand
{
    public function __construct(private readonly InvoiceRepository $invoices)
    {
        parent::__construct();
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $output->writeln("{$this->invoices->countOpen()} open invoices");

        return self::SUCCESS;
    }
}
```

Build the project and run the command:

```bash
skernel build
php bin/bootstrap.php invoice:count
```

It prints:

```text
2 open invoices
```

### Use built-in and tool commands

Symfony's application provides `help`, `list`, `completion`, and `_complete` itself; they need no declaration. A tool that ships its own executable, such as a code formatter, runs through that executable. To run a tool inside the application, declare a command with `#[Command]` that uses the tool's public services.

### How commands reach the application

The container provides `Symfony\Component\Console\CommandLoader\CommandLoaderInterface` with the registered commands. When `super-kernel/framework` publishes `SuperKernel\Event\BeforeApplicationStart`, a listener of this package installs that loader in the event's application if it is a `Symfony\Component\Console\Application`; an application of another type is left unchanged. Installing the loader creates no command.

The loader is created together with that listener, so an invalid declaration fails wherever the listeners are created: `skernel build` fails at its `ApplicationBuild` step, and the application fails to start. The `InvalidArgumentException` that names the problem is the previous exception of a `SuperKernel\Di\Exception\ResolutionException` with the message `Unable to invoke factory "SuperKernel\Command\Provider\CommandLoaderProvider" for container entry "Symfony\Component\Console\CommandLoader\CommandLoaderInterface".` Declaration errors that PHP itself reports, such as a wrong argument type, follow the rules of the build's Attribute collection.

## Errors

The declaration errors are `InvalidArgumentException`s raised when the command loader is created; the last one is raised when a command is initialized.

| Situation | Message |
| --- | --- |
| An empty name | `SuperKernel\Command\Attribute\Command on <class> must declare a non-empty command name.` |
| Aliases that are not a list of non-empty strings | `Command aliases on <class> must be a list of non-empty strings.` |
| A name or alias declared twice, also within one class | `Command name "<name>" is declared by both <class> and <class>.` |
| `#[Command]` on a class that does not extend Symfony's `Command` | `<class> declares SuperKernel\Command\Attribute\Command but does not extend Symfony\Component\Console\Command\Command.` |
| The container entry of a command class is not a Symfony command (`UnexpectedValueException`) | `Container entry <class> must resolve to Symfony\Component\Console\Command\Command, <type> returned.` |

An unknown name is reported by Symfony Console, for example `Command "invoice:imprt" is not defined.`, together with the closest names.

## Relation to other packages

- [`super-kernel/framework`](framework.md) creates the application and publishes `BeforeApplicationStart`, which this package listens to; Symfony Console runs the commands.
- [`super-kernel/event`](event.md) declares `BeforeApplicationStart`; [`super-kernel/event-dispatcher`](event-dispatcher.md) discovers the listener that installs the loader.
- [`super-kernel/di`](di.md) creates the commands and their dependencies and keeps them; [`super-kernel/annotator`](annotator.md) provides the recorded `#[Command]` declarations.
- `skernel build` collects the declarations, and a build fails when one is invalid; `skernel run` builds before it starts the application.

## What not to do

**Do not register a command with Symfony's `#[AsCommand]` alone.** The package ignores it, so the command does not exist: running it makes Symfony report `Command "<name>" is not defined.`, or `There are no commands defined in the "<namespace>" namespace.` when no command shares its namespace. Declare the command with `#[Command]`.

**Do not reuse a name or an alias.** Two declarations of one name, including a name repeated in a class's own aliases, make `skernel build` and every application start fail with `Command name "<name>" is declared by both <class> and <class>.` Give each command its own names.
