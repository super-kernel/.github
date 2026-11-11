<p align="right">
  <strong>English</strong> | <a href="server.zh-CN.md">中文文档</a>
</p>

# SuperKernel Server

Package: [`super-kernel/server`](https://github.com/super-kernel/server) · [Packagist](https://packagist.org/packages/super-kernel/server) · [Documentation index](../README.md)

## Purpose

`super-kernel/server` turns the application's server configuration into one running Swoole server. It creates the native server and its listeners, gives protocol packages and the application a moment to attach their callbacks, publishes Swoole's main callbacks as events, attaches the user processes declared for the server, and starts it.

The package handles no protocol: `super-kernel/http-server` and the application attach the callbacks that read connections. It provides no command and does not supervise processes: `swoole:start` of `super-kernel/swoole` starts the server in a child of its main process, which owns daemon mode, the PID file, and signals.

## Capabilities

### Configure the server

The application declares one class that implements `SuperKernel\Server\Config\SwooleServerConfigInterface` and marks it with `#[Configuration(SwooleServerConfigInterface::class)]`. The container validates the class and then provides the same object wherever `SwooleServerConfigInterface` is injected. The getters are called again when they are needed, so keep their results stable.

- `enable()` decides whether `swoole:start` starts the server. Without a declared configuration, nothing is started and the configuration is not read; a declared configuration is validated even when `enable()` returns `false`. `enable()` does not affect a direct call of `ServerInterface::start()`.
- `mode()` returns a `Mode`, described in [Choose a mode](#choose-a-mode).
- `listeners()` returns a non-empty list of listener arrays. The first is the main listener; the others are added in order.
- `settings()` returns Swoole server settings with string keys. Swoole itself validates the settings and reports their failures.

A listener array has these fields; any other key is rejected, and so is an explicit `null`:

| Field | Type | Required | Used when omitted |
| --- | --- | --- | --- |
| `name` | non-empty `string` | yes | |
| `host` | non-empty `string` | yes | |
| `port` | `int` | yes | |
| `socketType` | `int` | no | `SWOOLE_SOCK_TCP` |
| `ssl` | `bool` | no | `false` |
| `reusePort` | `bool` | no | `false` |
| `protocol` | `Protocol` | no | `Protocol::Raw` |

Listener names identify listeners in events and log records; they are case-sensitive and must be unique. The arrays reach the events exactly as the configuration returns them: omitted fields are not added. In base, process, and thread mode, the main listener must have the highest protocol capability of all listeners, in the order `WebSocket`, `Http`, `Raw`, because the main listener decides which native server class is created; TLS is part of `socketType`, for example `SWOOLE_SOCK_TCP | SWOOLE_SSL`. Coroutine mode allows exactly one listener, with the `Raw` protocol, and uses its `ssl` and `reusePort` fields.

A configuration with an HTTP listener and a raw listener, saved as `src/Config/SwooleServerConfig.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Config;

use SuperKernel\Attribute\Configuration;
use SuperKernel\Server\Config\SwooleServerConfigInterface;
use SuperKernel\Server\Constants\Mode;
use SuperKernel\Server\Constants\Protocol;
use Swoole\Constant;

#[Configuration(SwooleServerConfigInterface::class)]
final readonly class SwooleServerConfig implements SwooleServerConfigInterface
{
    public function enable(): bool
    {
        return true;
    }

    public function mode(): Mode
    {
        return Mode::Process;
    }

    public function listeners(): array
    {
        return [
            [
                'name'     => 'http',
                'host'     => '127.0.0.1',
                'port'     => 9501,
                'protocol' => Protocol::Http,
            ],
            [
                'name' => 'echo',
                'host' => '127.0.0.1',
                'port' => 9502,
            ],
        ];
    }

    public function settings(): array
    {
        return [
            Constant::OPTION_WORKER_NUM       => 2,
            Constant::OPTION_ENABLE_COROUTINE => true,
        ];
    }
}
```

### Choose a mode

| Mode | Native server | Notes |
| --- | --- | --- |
| `Mode::Base` | `Swoole\Server`, `Swoole\Http\Server`, or `Swoole\WebSocket\Server`, chosen by the main listener's protocol | Swoole's `SWOOLE_BASE`. |
| `Mode::Process` | as for `Base` | Swoole's `SWOOLE_PROCESS`. |
| `Mode::Thread` | as for `Base` | Swoole's `SWOOLE_THREAD`; needs a ZTS PHP and a Swoole build with thread support. |
| `Mode::Coroutine` | `Swoole\Coroutine\Server` | One raw listener; no main callbacks and no user processes. |

In base, process, and thread mode, the container provides the native server as `Swoole\Server`, with the settings applied, the extra listeners added, and the main callbacks and startup log records registered. In coroutine mode it provides `Swoole\Coroutine\Server` instead. Server treats the coroutine listener as the main one and sets no `handle()` callback for it: the code that serves connections sets it in an `AfterServerCreated` listener.

### Attach a protocol handler

`ServerInterface::start()` publishes two events before the server starts, synchronously and in this order:

1. `AfterMainServerCreated` once; `server()` returns the native server.
2. `AfterServerCreated` once for every listener, the main listener first; `server()` returns the native server for the main listener and its `Swoole\Server\Port` for the others, and `configuration()` returns the listener's array.

Attach the callbacks that read connections, such as `receive` or `request`, in these listeners. Read a listener's name with `configuration()['name']` and its protocol with `configuration()['protocol'] ?? Protocol::Raw`. An exception from a listener prevents the start.

A listener that answers on the `echo` listener of the configuration above, saved as `src/Listener/AttachEcho.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Listener;

use SuperKernel\Attribute\Listener;
use SuperKernel\Contract\ListenerInterface;
use SuperKernel\Server\Event\AfterServerCreated;
use Swoole\Server;

#[Listener]
final readonly class AttachEcho implements ListenerInterface
{
    public function listen(): array
    {
        return [
            AfterServerCreated::class,
        ];
    }

    public function process(object $event): void
    {
        assert($event instanceof AfterServerCreated);

        if ('echo' !== $event->configuration()['name']) {
            return;
        }

        $event->server()->on('receive', static function (Server $server, int $fd, int $reactorId, string $data): void {
            $server->send($fd, 'echo: ' . $data);
        });
    }
}
```

A client, saved as `bin/echo-client.php`:

```php
<?php

declare(strict_types=1);

$client = stream_socket_client('tcp://127.0.0.1:9502', $errorCode, $errorMessage, 1.0);

if (false === $client) {
    exit("Unable to connect: {$errorMessage}" . PHP_EOL);
}

fwrite($client, 'hello');

echo fread($client, 1024), PHP_EOL;
```

Build the project, start it, and run the client from another terminal:

```bash
skernel build
php bin/bootstrap.php swoole:start
php bin/echo-client.php
```

The client prints `echo: hello`.

### React to server callbacks

Server registers all 14 main callbacks of the native server and publishes each as an event through `Psr\EventDispatcher\EventDispatcherInterface`. The events are in `SuperKernel\Server\Event\Callbacks` and carry Swoole's arguments as public properties:

| Event | Swoole callback | Properties |
| --- | --- | --- |
| `OnStart` | `start` | `server` |
| `OnBeforeShutdown` | `beforeShutdown` | `server` |
| `OnShutdown` | `shutdown` | `server` |
| `OnManagerStart` | `managerStart` | `server` |
| `OnManagerStop` | `managerStop` | `server` |
| `OnWorkerStart` | `workerStart` | `server`, `workerId` |
| `OnWorkerStop` | `workerStop` | `server`, `workerId` |
| `OnWorkerExit` | `workerExit` | `server`, `workerId` |
| `OnWorkerError` | `workerError` | `server`, `status`, `workerPid`, `exitCode`, `signal` |
| `OnTask` | `task` | `server`, `task`, `sourceWorkerId`, `data`, and the writable `result` |
| `OnFinish` | `finish` | `server`, `result`, `data` |
| `OnPipeMessage` | `pipeMessage` | `server`, `message`, `data` |
| `OnBeforeReload` | `beforeReload` | `server` |
| `OnAfterReload` | `afterReload` | `server` |

Where Swoole passes either an object or separate values, as for tasks with `task_enable_coroutine`, the event keeps Swoole's form: `task` is a `Swoole\Server\Task` or the task ID, and the following properties are filled only in the second form. A listener of `OnTask` answers a task by assigning `$event->result`; Server returns that value to Swoole. An exception from a listener propagates to the native callback.

The Quick start of the README listens to `OnStart`. When the server really starts, Server then writes these records through `StdoutLoggerInterface`, after the `OnStart` listeners have run; in coroutine mode it writes them once the coroutine server is running:

```text
[NOTICE] Swoole main server started in process mode.
[INFO] Swoole server process started with PID 958085.
[INFO] Swoole listener "http" started on 127.0.0.1:9501 (http).
[INFO] Swoole listener "echo" started on 127.0.0.1:9502 (raw).
```

The callbacks and the records are registered only when the container creates `Swoole\Server` through Server's own provider. Connection callbacks, such as `receive`, `request`, and `message`, belong to protocol handlers.

### Declare user processes

A user process is a worker that the Swoole server starts next to its own workers, with `Swoole\Server::addProcess()`, and restarts when it ends. Mark a class with `#[SuperKernel\Server\Attribute\UserProcess]` and implement `SuperKernel\Server\Contract\UserProcessInterface`:

- The Attribute takes `name` (default: the class name), `workNums` (default `1`, from `0` to `1024`), and `enableCoroutine` (default `false`). With more than one worker, the names are `name:0`, `name:1`, and so on.
- The interface declares four readable properties: `enabled`, which excludes the class when `false`; `redirectStdinStdout`; `pipeType`, one of `Swoole\Process::PIPE_TYPE_NONE`, `PIPE_TYPE_STREAM`, or `PIPE_TYPE_DGRAM`; and `enableCoroutine`, which the Attribute's `enableCoroutine` overrides without changing the object.
- `execute(Swoole\Process $process, Swoole\Server $server)` runs in the worker. Returning from it ends the worker, and the Swoole server may start it again.

The class is created in the server's parent process before the workers exist. Keep its constructor and properties free of connections and other resources, and open them in `execute()`.

A user process that reports once a second, saved as `src/Worker/Heartbeat.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Worker;

use SuperKernel\Server\Attribute\UserProcess;
use SuperKernel\Server\Contract\UserProcessInterface;
use Swoole\Process;
use Swoole\Server;

#[UserProcess('heartbeat')]
final class Heartbeat implements UserProcessInterface
{
    public bool $enabled = true;

    public bool $redirectStdinStdout = false;

    public int $pipeType = Process::PIPE_TYPE_NONE;

    public bool $enableCoroutine = false;

    public function execute(Process $process, Server $server): void
    {
        while (true) {
            echo "Heartbeat from worker {$process->pid} of server {$server->master_pid}", PHP_EOL;
            sleep(1);
        }
    }
}
```

After `skernel build`, `php bin/bootstrap.php swoole:start` prints a `Heartbeat from worker …` line every second until the server stops.

The container provides `SuperKernel\Server\Contract\UserProcessManagerInterface` with every `#[UserProcess]` class registered, in the order the build found them. A class with `workNums` of `0` is not created; a disabled class is created once but adds no workers. Code that runs before the server starts, such as an `AfterMainServerCreated` listener, can register more:

- `add($process, $enableCoroutine = null, $name = null)` registers one worker; `addBatch($workerNum, $process, $enableCoroutine = null, $name = null)` registers from `0` to `1024` workers at once, or none if any of them is rejected. Both return the manager.
- `$process` is a `UserProcessInterface` or a `callable(Swoole\Process, Swoole\Server): void`. A callable runs without coroutines, without redirection, and with a datagram pipe; wrap it in `SuperKernel\Server\Definition\UserProcessDefinition` to choose other options. A `null` `$enableCoroutine` keeps the process's own choice.
- Without a name, a worker is named by its registration index, starting at `0`. Names are case-sensitive and must be unique, and the manager holds at most 1024 workers.
- `all()` returns the registered processes in order, keyed by name; `has($name)` reports whether a name is registered; `get($name)` returns that process or raises `SuperKernel\Server\Exception\UserProcessNotFoundException`.

The manager captures the options when a process is registered and never reads them again. After the creation events and the `daemonize` reset, Server attaches one native worker per registered process, in order, and then starts the server; a process registered later is not attached. The manager has no start or stop operation; the Swoole server starts, reaps, and restarts the workers, and decides in which modes user processes are possible. A coroutine server attaches no user processes. A worker name is not the operating system's process title.

### Start the server yourself

`SuperKernel\Server\Contract\ServerInterface` is what `swoole:start` starts in its child process. A script can start the server directly, without the main process. Saved as `bin/serve.php`:

```php
<?php

declare(strict_types=1);

use SuperKernel\Context\ApplicationContext;
use SuperKernel\Server\Contract\ServerInterface;

require dirname(__DIR__) . '/vendor/autoload.php';

ApplicationContext::getContainer()->get(ServerInterface::class)->start();
```

Run it after `skernel build`:

```bash
php bin/serve.php
```

It starts the server in the foreground, whatever `enable()` returns, and blocks until the server stops; press Ctrl+C to stop it. A `ServerInterface` accepts one start: a second call raises `The configured Swoole service may only be started once.`, also after a failed start. In base, process, and thread mode, `start()` sets `daemonize` to `false` just before the server starts, overriding the settings and anything an event listener set; run in the background with daemon mode of `super-kernel/swoole` instead. In coroutine mode, `start()` starts a coroutine scheduler when it is not already inside a coroutine and blocks until the server stops. `getServer()` returns the same native server every time, for code that needs Swoole's own API.

## Errors

Every exception of this package implements `SuperKernel\Server\Exception\SwooleServerExceptionInterface`.

| Situation | Message |
| --- | --- |
| No configuration declared, for code that needs it | `No configuration source is registered for SuperKernel\Server\Config\SwooleServerConfigInterface.` |
| A listener without a required field | `Listener configuration at position <n> is missing required field "<field>".` |
| A listener with an unknown field | `Listener configuration at position <n> contains unknown field "<field>".` |
| A listener field of the wrong type | `Listener field "<field>" at position <n> must be <type>; <type> provided.` |
| An empty name or host | `Listener configuration at position <n> must define a non-empty <field>.` |
| Two listeners with one name | `Listener name "<name>" at position <n> duplicates the listener at position <n>.` |
| No listener | `The Swoole server configuration must define at least one listener.` |
| A later listener needs a higher protocol than the main one | `Listener "<name>" at position <n> requires the "<protocol>" protocol, which exceeds primary listener "<name>" using "<protocol>"; place a listener with the highest protocol capability first.` |
| Coroutine mode with several listeners, or with another protocol | `Swoole coroutine mode supports exactly one listener.` or `Swoole coroutine mode supports only the raw listener protocol; listener "<name>" requires "<protocol>".` |
| A setting with an integer key | `Swoole server settings must use string keys.` |
| Thread mode without thread support | `Swoole thread mode requires a ZTS PHP runtime and an ext-swoole build with thread support.` |
| The native server cannot be created, for example because the port is in use | `Unable to resolve the configured Swoole service from container entry Swoole\Server.`, with a cause such as `Unable to create the Swoole process-mode server for listener "<name>".` |
| A main callback cannot be registered | `Unable to register Swoole Server event "<event>" through callback handler <class>.` |
| A second start | `The configured Swoole service may only be started once.` |
| An invalid `#[UserProcess]` | `UserProcess name must not be empty.` or `UserProcess count must be between 0 and 1024.` |
| A `#[UserProcess]` class that does not implement the interface | `<class> must implement SuperKernel\Server\Contract\UserProcessInterface.` |
| A duplicate worker name, a blank name, or too many workers | `Duplicate user process name: <name>.`, `User process names must not be blank.`, or `At most 1024 user workers may be registered.` |
| An invalid pipe type | `Invalid user process pipe type: <type>` |
| `get()` with an unknown name | `User process not found: "<name>".` |

Configuration errors are `SuperKernel\Server\Exception\InvalidServerConfigurationException`, creation errors `ServerCreationException`, start errors `ServerStartException`, and mode errors `UnsupportedServerModeException`, all in `SuperKernel\Server\Exception`; the user-process errors are `InvalidArgumentException` and `LogicException`. When the container creates the configuration or the server, it wraps these as the previous exception of a `SuperKernel\Di\Exception\ResolutionException`.

## Relation to other packages

- [`super-kernel/swoole`](swoole.md) runs `swoole:start`; this package listens to its `MainProcessCreated` event and mounts a child that starts the server.
- [`super-kernel/http-server`](http-server.md) attaches the HTTP `request` callback in an `AfterServerCreated` listener.
- [`super-kernel/process`](process.md) runs background processes in a separate pool next to the server; user processes of this package belong to the server instead.
- [`super-kernel/config`](config.md) finds the `#[Configuration]` source, and [`super-kernel/logger`](logger.md) provides the `StdoutLoggerInterface` service for the startup records.
- [`super-kernel/event-dispatcher`](event-dispatcher.md) delivers the events to listeners found by `skernel build`.

## What not to do

**Do not set main callbacks such as `start` or `workerStart` with `on()`.** Server registers all 14 and publishes them as events; setting one again replaces Server's handler, so its event and, for `start`, the startup records disappear. Listen to the `On*` event instead.

**Do not provide `Swoole\Server` with your own provider.** A provider for `Swoole\Server` bypasses Server's provider, and with it the main callbacks, their events, and the startup records. Change the server through the configuration and the creation events.

**Do not replace the configuration with a `#[Provider]` or `#[FactoryProvider]`.** The package's own factory validates the configuration; another provider for `SwooleServerConfigInterface` skips the validation and conflicts with the factory. Declare the class with `#[Configuration(SwooleServerConfigInterface::class)]`.

**Do not enable `daemonize` in the settings.** Server resets it to `false` before the server starts. Run in the background with the daemon mode of `super-kernel/swoole`.

**Do not open connections in a user process's constructor or properties.** The object is created in the parent before the workers exist, so the connection would be shared by every worker or broken after the fork. Open it in `execute()`.

**Do not start, wait for, or daemonize processes inside `execute()`.** The Swoole server owns its user workers: do not call `Process::start()`, `wait()`, `daemon()`, or `Server::start()` there.

**Do not register user processes after the server has started.** They are attached once, just before the start; later registrations change only the manager's list.
