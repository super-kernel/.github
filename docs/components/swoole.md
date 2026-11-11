<p align="right">
  <strong>English</strong> | <a href="swoole.zh-CN.md">中文文档</a>
</p>

# SuperKernel Swoole

Package: [`super-kernel/swoole`](https://github.com/super-kernel/swoole) · [Packagist](https://packagist.org/packages/super-kernel/swoole) · [Documentation index](../README.md)

## Purpose

`super-kernel/swoole` starts a SuperKernel application's long-running work. Its `swoole:start` command creates one main process, lets every installed package and the application mount work on it, and then supervises that work: initialization callbacks run in the main process, and each runtime, such as the HTTP server or the process pool, runs in its own child process next to the others. The main process owns daemon mode, the PID file, signals, and shutdown.

What runs in a child belongs to the package that mounts it: `super-kernel/server` mounts the server and `super-kernel/process` mounts the process pool. This package never restarts a child and keeps no registry of processes.

## Capabilities

### Start the runtime

`swoole:start` takes no arguments and no options of its own; Symfony Console's help and output options apply. Creating the command, listing commands, or showing its help resolves nothing. When it runs, the command:

1. resolves `MainProcess` from the container, which reads and validates the `SwooleConfigInterface` configuration;
2. dispatches `new MainProcessCreated($mainProcess)` synchronously, so that every listener mounts its work;
3. starts that same main process and returns when every mounted child has exited and been reaped.

A configuration failure stops the command before the event is dispatched, even when nothing would be mounted. The command runs once per application run; the main process rejects a second start.

`SuperKernel\Swoole\Runtime\ProcessRuntime` does the same without the command: it takes the PSR-14 event dispatcher and the `StdoutLoggerInterface` logger in its constructor, and `start(SwooleConfigInterface $config)` creates a new `MainProcess`, dispatches its event, and starts it.

Other packages may declare more commands under `swoole:`, such as `swoole:watch`, with the `#[Command]` Attribute of `super-kernel/command`. The package does not reserve the prefix; every full command name and alias must be unique.

### Mount work

A listener of `SuperKernel\Swoole\Event\MainProcessCreated` receives the main process as the event's `mainProcess` property; the event carries nothing else. Nothing mounted has run when the event is dispatched.

- `add(callable $callback)` mounts an initialization callback that runs in the main process.
- `addProcess(callable $callback)` mounts a child process that runs the callback.

`start()` runs the mounted callbacks once each, in the order they were mounted, and ignores their return values; callbacks take no arguments. An initialization callback must return before the next mounted item starts. Children are created in mounting order and then run concurrently. A child has no pipe to the main process and no coroutine scheduler: a child that needs coroutines starts the scheduler itself, and it creates its servers, pools, connections, and signal handlers inside its callback, not in the listener. A child that returns normally is reaped and not replaced.

Only the code that created the main process may mount work or start it, before it starts. Mounting after `start()` has begun, mounting from a forked copy, or starting twice raises `LogicException`.

The Quick start of the README mounts an initialization callback and a child. This listener mounts a child that runs coroutines, saved as `src/Listener/MountTicker.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Listener;

use SuperKernel\Attribute\Listener;
use SuperKernel\Contract\ListenerInterface;
use SuperKernel\Swoole\Event\MainProcessCreated;
use Swoole\Coroutine;

use function Swoole\Coroutine\run;

#[Listener]
final readonly class MountTicker implements ListenerInterface
{
    public function listen(): array
    {
        return [
            MainProcessCreated::class,
        ];
    }

    public function process(object $event): void
    {
        assert($event instanceof MainProcessCreated);

        $event->mainProcess->addProcess(static function (): void {
            run(static function (): void {
                for ($tick = 1; $tick <= 3; $tick++) {
                    Coroutine::sleep(0.1);
                    echo "Tick {$tick}", PHP_EOL;
                }
            });
        });
    }
}
```

In an application created from `super-kernel/skeleton`, build the project and start it:

```bash
skernel build
php bin/bootstrap.php swoole:start
```

It prints `Tick 1`, `Tick 2`, and `Tick 3` next to the server's log lines. The ticker child then exits and is reaped while the server keeps running; press Ctrl+C to stop it.

### Configure the runtime

The application declares one class that implements `SuperKernel\Swoole\Config\SwooleConfigInterface` and marks it with `#[Configuration(SwooleConfigInterface::class)]`. The container validates that class and then provides the same object wherever `SwooleConfigInterface` is injected. The getters are called again when they are needed, so keep their results stable for the whole run.

A configuration for a foreground application without a PID file, saved as `src/Config/SwooleConfig.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Config;

use SuperKernel\Attribute\Configuration;
use SuperKernel\Swoole\Config\SwooleConfigInterface;

#[Configuration(SwooleConfigInterface::class)]
final readonly class SwooleConfig implements SwooleConfigInterface
{
    public function daemon(): bool
    {
        return false;
    }

    public function pidFile(): ?string
    {
        return null;
    }

    public function displayErrors(): bool
    {
        return true;
    }

    public function socketBufferSize(): int
    {
        return 8 * 1024 * 1024;
    }

    public function enablePreemptiveScheduler(): bool
    {
        return false;
    }
}
```

| Method | Value in `super-kernel/skeleton` | Meaning |
| --- | --- | --- |
| `daemon()` | `false` | Whether the main process detaches into the background before it runs anything. |
| `pidFile()` | `vendor/.cache/super-kernel.pid` as an absolute path, or `null` in a Phar | The absolute path of the PID file, or `null` for none; daemon mode requires one. |
| `displayErrors()` | `true` | Sets `swoole.display_errors`, independently of PHP's `display_errors` and of the logger. |
| `socketBufferSize()` | `8388608` (8 MiB) | The default Swoole socket buffer size in bytes, from `1` to `4294967295`. |
| `enablePreemptiveScheduler()` | `false` | Enables preemptive scheduling when a child starts its coroutines. |

The package has no defaults of its own: the application's class supplies every value.

Reading the configuration changes nothing in the process. When the main process starts with something mounted, before the first callback and before any child exists, it sets the INI options `swoole.display_errors`, `swoole.socket_buffer_size`, and `swoole.enable_preemptive_scheduler`, and passes the same three values to `Swoole\Coroutine::set()`, which refreshes the socket default and replaces an earlier preemptive-scheduler option while other coroutine options stay as they are. This starts no coroutine or event loop. The children inherit the settings, and each child decides when to start its scheduler. The settings apply to the whole process and stay in effect after `start()` returns; with nothing mounted, they are not applied.

The options follow Swoole 6.2.3. `swoole.enable_coroutine` is not an INI directive. `swoole.use_shortname` and `swoole.enable_library` are set when PHP starts, in `php.ini` or with `-d`, and are not part of this interface; changing `swoole.enable_library` later cannot unload a library that is already loaded. See Swoole's [INI registration and initialization](https://github.com/swoole/swoole-src/blob/v6.2.3/ext-src/php_swoole.cc) and [coroutine configuration](https://github.com/swoole/swoole-src/blob/v6.2.3/ext-src/swoole_coroutine_scheduler.cc).

### Run as a daemon

To run in the background, return `true` from `daemon()` and an absolute path from `pidFile()`. The configuration of `super-kernel/skeleton` already returns an absolute path, so it is enough to change `daemon()` to `return true;`. Then the command returns as soon as the main process has detached, and the main process keeps running until it is stopped:

```bash
skernel build
php bin/bootstrap.php swoole:start
kill -TERM "$(cat vendor/.cache/super-kernel.pid)"
```

The PID file follows these rules:

- The main process takes an exclusive lock on the file before it detaches, and fails with `Cannot acquire PID file <path>.` when another process holds it.
- After it detaches, it writes its own final PID into the file before the first callback runs or the first child starts. The file never holds the PID of a server, a pool, or a worker.
- Children close their inherited handle of the file without releasing the lock. The main process keeps the lock until every child has been reaped, then removes the file.
- A command that returned successfully, or an existing PID file, does not mean that the children are ready to serve.

Daemon mode redirects only standard input, to `/dev/null`; standard output and standard error stay where they were, so redirect them when the application starts if the terminal will close.

### Stop the runtime

Send `SIGTERM` or `SIGINT` to the PID in the PID file, or press Ctrl+C in the foreground. The main process sends `SIGTERM` to each of its live children in mounting order, waits for all of them, and removes its PID file. The order of sending does not decide the order in which children handle the signal or exit. Each child stops what it started: the pool's parent process stops and reaps its workers, and the server stops its own workers. The main process waits only for the children it created, so other child processes of the same PHP process keep their exit status. While it waits, it blocks the signals with PHP's [signal mask](https://www.php.net/manual/en/function.pcntl-sigprocmask.php) and [waits for them](https://www.php.net/manual/en/function.pcntl-sigwaitinfo.php); the original mask is restored when `start()` returns.

When a child ends with an exception, a non-zero exit code, or a signal other than the shutdown signal, the main process stops and reaps the other children and raises `SuperKernel\Swoole\Exception\SupervisorException`. An exception cannot cross the process boundary as the same PHP object, so the child logs it and exits with code 1. An exception from an initialization callback propagates unchanged after the children are stopped and reaped.

The main process has no restart policy and no shutdown deadline: a child that ignores `SIGTERM` delays shutdown for as long as it runs. The process pool replaces its own workers; see [`super-kernel/process`](process.md). Restarting after `SIGKILL` and limiting resources belong to a service manager outside the application.

### Logs and output

The main process logs only failures, at the error level of the `StdoutLoggerInterface` service: an exception in a child (`Mounted process failed: <message>`) and, in daemon mode, a failure of the main process itself (`Main process failed: <message>`). That service decides where records go; `super-kernel/logger` writes them to standard output, and its page describes how to replace it. The main process does not capture or forward what the children print, and there is no log-file setting. With nothing mounted, `start()` returns at once and logs nothing.

## Errors

| Situation | Message |
| --- | --- |
| No configuration source declared | `No configuration source is registered for SuperKernel\Swoole\Config\SwooleConfigInterface.` (`SuperKernel\Config\Exception\ConfigurationNotFoundException`) |
| A relative PID path, or daemon mode without one | `Use an absolute PID file path; daemon mode requires one.` (`InvalidArgumentException`) |
| A socket buffer size outside `1` to `4294967295` | `Use a socket buffer size between 1 and 4294967295 bytes.` (`InvalidArgumentException`) |
| Mounting after startup began, or from a forked copy | `Main process tasks must be added by their owner before startup.` (`LogicException`) |
| Starting twice, from a callback, or from a forked copy | `The main process may only be started once by its owner.` (`LogicException`) |
| Starting inside a coroutine or after asynchronous work | `Start MainProcess in a clean CLI before reactors or asynchronous work.` |
| The PID file is locked by another process | `Cannot acquire PID file <path>.` |
| Detaching failed | `Unable to daemonize MainProcess.` |
| A child failed | `A mounted process failed with exit code <code> and signal <signal>.` |
| A Swoole INI option was rejected | `Unable to apply Swoole INI setting: <directive>.` |

Messages without a class are `SuperKernel\Swoole\Exception\SupervisorException`. The configuration errors are raised when the container creates `SwooleConfigInterface`; the invalid-value errors reach the caller as the previous exception of a `SuperKernel\Di\Exception\ResolutionException`.

## Relation to other packages

- [`super-kernel/server`](server.md) listens to `MainProcessCreated` and mounts a child that starts the server when its configuration enables it.
- [`super-kernel/process`](process.md) listens to the same event and mounts a separate child that runs the process pool when processes are registered.
- [`super-kernel/command`](command.md) registers `swoole:start` with the console application of [`super-kernel/framework`](framework.md).
- [`super-kernel/config`](config.md) finds the application's `#[Configuration]` source.
- [`super-kernel/logger`](logger.md) provides the `StdoutLoggerInterface` service that records failures.
- [`super-kernel/event-dispatcher`](event-dispatcher.md) delivers `MainProcessCreated` to listeners found by `skernel build`.

## What not to do

**Do not create servers, pools, or connections in a listener or an initialization callback.** They would belong to the main process and be copied into every child. Create them inside the `addProcess()` callback that runs them.

**Do not start `swoole:start` after starting coroutines or an event loop.** The main process forks its children and must start in a clean CLI; otherwise it raises `Start MainProcess in a clean CLI before reactors or asynchronous work.` Start it from the console command, before anything asynchronous.

**Do not mount work after the main process has started.** A late `add()` or `addProcess()`, also from a callback, raises `Main process tasks must be added by their owner before startup.` Mount everything from a `MainProcessCreated` listener.

**Do not replace the configuration with a `#[Provider]` or `#[FactoryProvider]`.** The package's own factory validates the PID path and the socket buffer size; another provider for `SwooleConfigInterface` skips that validation and conflicts with the package's factory. Declare the class with `#[Configuration(SwooleConfigInterface::class)]`.

**Do not use the configuration for per-request or runtime changes.** The settings apply once, to the whole process, before the children start, and the getters are called again later. Return fixed values, and restart the application to change them.

**Do not rely on the main process to restart children.** A child that exits normally is not replaced, and a child that fails stops the whole runtime. Put a service manager such as systemd in front of the application for restarts.
