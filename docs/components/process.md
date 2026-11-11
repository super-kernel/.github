<p align="right">
  <strong>English</strong> | <a href="process.zh-CN.md">中文文档</a>
</p>

# SuperKernel Process

Package: [`super-kernel/process`](https://github.com/super-kernel/process) · [Packagist](https://packagist.org/packages/super-kernel/process) · [Documentation index](../README.md)

## Purpose

`super-kernel/process` runs work that belongs next to a server rather than inside a request: queue consumers, schedulers, maintenance loops. Each background process is a named entry with one or more workers, and all entries run as the workers of one `Swoole\Process\Pool`. Its manager follows the PHP implementation of Swoole's [`Process\Manager`](https://github.com/swoole/library/blob/master/src/core/Process/Manager.php): one pool per start, one worker per registered entry, and a per-entry choice between direct and coroutine execution.

Under `swoole:start`, the pool runs in its own child of the main process of `super-kernel/swoole`, next to the server of `super-kernel/server`; the main process owns daemon mode, the PID file, and signals. The server's own user processes belong to `super-kernel/server`.

## Capabilities

### Declare background processes

Mark a class with `#[SuperKernel\Process\Attribute\Process]` and implement `SuperKernel\Process\Contract\ProcessInterface`:

- The Attribute takes `name` (default: the class name, even when the container returns another implementation), `workNums` (default `1`, from `0` to `1024`), and `enableCoroutine` (default `null`). With more than one worker, the names are `name:0`, `name:1`, and so on.
- The interface declares two readable properties, `enabled` and `enableCoroutine`, and `execute(Swoole\Process\Pool $pool, int $workerId): void`, which runs in the worker.
- A class whose `enabled` is `false` adds no workers and uses no name. A class with `workNums: 0` is not created at all; any other class is created once from the container, in the parent, to read its properties.

The class is created and its properties are read before the workers are forked. Keep its constructor and properties free of connections, event loops, and other resources, and open them in `execute()`.

The container provides `SuperKernel\Process\Contract\ProcessManagerInterface` with every `#[Process]` class registered, in the order the build found them. Preparing it never runs a task or starts a pool. The Quick start of the README declares a process this way.

The skeleton's `Src\Process\TestProcess` is such a declaration and stays disabled: its `enabled` property is `false`. When enabled, it logs a heartbeat through `StdoutLoggerInterface` every 30 seconds with `Swoole\Coroutine::sleep(30)` in a coroutine, and never returns, because a returning worker would be replaced. Replace the heartbeat with application work before enabling it.

### Choose direct or coroutine execution

The pool itself runs without coroutines. Each entry's effective `enableCoroutine` decides whether its task runs directly or inside `Swoole\Coroutine\run($callback, $pool, $workerId)`. The Attribute's `enableCoroutine`, or the `$enableCoroutine` argument of the manager, overrides the task's property only when it is `true` or `false`; `null` keeps the property:

| Override | Property `false` | Property `true` |
| --- | --- | --- |
| omitted or `null` | direct | coroutine |
| `false` | direct | direct |
| `true` | coroutine | coroutine |

An override does not change the task object: the manager wraps the task in a `SuperKernel\Process\Definition\ProcessDefinition` with the effective setting, and `all()` and `get()` return that wrapper. Without an override, the registered object itself is kept. Integrations that register declarations themselves compute the setting as `$attribute->enableCoroutine ?? $task->enableCoroutine`.

### Register processes with the manager

`SuperKernel\Process\Manager\ProcessManager` implements `ProcessManagerInterface` and can be used without the container:

| Method | Behavior |
| --- | --- |
| `add($process, $enableCoroutine = null, $name = null)` | Registers one worker and returns the manager. |
| `addBatch($workerNum, $process, $enableCoroutine = null, $name = null)` | Registers from `0` to `1024` workers at once, or none if any of them is rejected, and returns the manager. |
| `all()` | Returns the registered tasks keyed by name, in worker-ID order. |
| `has($name)` | Reports whether a name is registered; it says nothing about a running worker. |
| `get($name)` | Returns the registered task, the same object every time, also after its worker was replaced. |
| `setIPCType()` / `getIPCType()` | Sets or reads the pool's IPC mode. |
| `setMsgQueueKey()` / `getMsgQueueKey()` | Sets or reads the pool's System V message-queue key. |
| `start($beforeStart = null, $afterFork = null)` | Starts the pool and returns after shutdown, once every worker has been reaped. |
| `stop()` | Requests shutdown of the whole pool, from the parent or from a worker. |

`$process` is a `ProcessInterface` or a `callable(Swoole\Process\Pool, int): void`. A callable is wrapped in a `ProcessDefinition` and runs directly unless `$enableCoroutine` is `true`; `new ProcessDefinition($callback, enableCoroutine: true, enabled: true)` creates such a wrapper yourself. A task whose `enabled` is `false` is skipped by `add()` and `addBatch()` and uses no name or worker ID. Without a name, a worker is named by its registration index, starting at `0`; a named single worker keeps its name. Names are case-sensitive, must not be empty, and may be full class names; the manager holds at most 1024 workers. `add()`, `addBatch()`, and the setters can be chained.

Only the process that created the manager may change or start it, and the first `start()` closes registration and settings, also when nothing is registered or the start fails. A manager with nothing registered, or one stopped before it started, creates no pool. `start()` calls `$beforeStart` in the parent after the pool is prepared and before it starts, and `$afterFork` in every worker before its task; a `stop()` in `$beforeStart` cancels the start.

A script that runs a pool of two coroutine workers and one maintenance worker, saved as `bin/maintenance.php`:

```php
<?php

declare(strict_types=1);

use SuperKernel\Process\Manager\ProcessManager;
use Swoole\Coroutine;
use Swoole\Process\Pool;

require dirname(__DIR__) . '/vendor/autoload.php';

$manager = new ProcessManager();

$manager->addBatch(2, static function (Pool $pool, int $workerId): void {
    while (true) {
        Coroutine::sleep(1);
    }
}, enableCoroutine: true, name: 'mailer');

$manager->add(static function (Pool $pool, int $workerId): void {
    echo "Maintenance ran in worker {$workerId}", PHP_EOL;
    $pool->shutdown();
}, name: 'maintenance');

echo implode(', ', array_keys($manager->all())), PHP_EOL;

$manager->start();

echo 'Pool stopped', PHP_EOL;
```

Run the script:

```bash
php bin/maintenance.php
```

It prints:

```text
mailer:0, mailer:1, maintenance
Maintenance ran in worker 2
Pool stopped
```

### Pool lifecycle

Start a pool in a clean CLI, before any coroutine, event loop, or asynchronous work. While the pool runs, Swoole starts a worker again whenever its task returns, fails, or does nothing: a finite job is not "done" when it returns. There is no retry limit, no delay, and no rule that one failing worker stops the others. A job that should end the pool coordinates with its siblings and then requests shutdown. See the [native pool lifecycle](https://github.com/swoole/swoole-src/blob/v6.2.3/src/os/process_pool.cc).

`$pool->shutdown()` in a worker, `stop()` on the manager, or `SIGTERM` to the pool's parent process stops the whole pool: Swoole sends `SIGTERM` to the workers, and each task must end and clean up. There is no deadline and no forwarding of `SIGINT`, so a worker that ignores the signal delays shutdown for as long as it runs. After Swoole's `Pool::start()` returns, the manager waits for the remaining workers to exit. The pool reaps every child process of its parent, so do not run other child processes from the pool's parent; their exit status would be lost. `all()`, `has()`, and `get()` still work after shutdown, but they never report whether a worker is alive.

### Communicate between workers

The pool's IPC applies to every worker, coroutine or not. It is `SWOOLE_IPC_NONE` by default; `SWOOLE_IPC_UNIXSOCK` and `SWOOLE_IPC_MSGQUEUE` are also accepted, and the message-queue key defaults to `0`, a private queue. Set them with `new ProcessManager(ipcType: SWOOLE_IPC_MSGQUEUE, msgQueueKey: 0x5301)` or the setters; the constructor takes nothing else. `SWOOLE_IPC_SOCKET` is rejected, because it needs a `listen()` step before the start that the manager does not offer. Workers use the IPC methods of the `Pool` they receive; there is no pipe per worker, and a worker's endpoint does not survive its replacement.

### Run under the main process

`swoole:start` dispatches `MainProcessCreated`; this package listens at the lowest priority, after other listeners. It copies the registered tasks, their names and coroutine settings, and the IPC settings from `ProcessManagerInterface`, and mounts one child on the main process. That child creates a new manager from the copy and starts the pool. With nothing registered, nothing is mounted. Registrations, IPC changes, or a `stop()` on the original manager after the event do not reach the running pool.

The pool's parent is one child of the main process: neither it nor a worker writes the PID file, which holds the main process's PID. `SIGTERM` or `SIGINT` to the main process sends `SIGTERM` to its children; the pool's parent stops and reaps its workers and returns, and the main process reaps it. The server, when its configuration enables it, runs in another child next to the pool and is never a pool worker. `ProcessRuntime::start($config)` of `super-kernel/swoole` starts the same main process without the command.

A script that supervises a pool with the main process directly, saved as `bin/supervise.php`:

```php
<?php

declare(strict_types=1);

use SuperKernel\Context\ApplicationContext;
use SuperKernel\Contract\StdoutLoggerInterface;
use SuperKernel\Process\Manager\ProcessManager;
use SuperKernel\Swoole\Config\SwooleConfigInterface;
use SuperKernel\Swoole\MainProcess;
use Swoole\Process\Pool;

require dirname(__DIR__) . '/vendor/autoload.php';

$container = ApplicationContext::getContainer();

$main = new MainProcess(
    $container->get(StdoutLoggerInterface::class),
    $container->get(SwooleConfigInterface::class),
);

$main->addProcess(static function (): void {
    $manager = new ProcessManager();

    $manager->add(static function (Pool $pool, int $workerId): void {
        echo "Worker {$workerId} of the supervised pool", PHP_EOL;
        $pool->shutdown();
    }, name: 'worker');

    $manager->start();
});

$main->start();

echo 'Supervisor finished', PHP_EOL;
```

In an application created from `super-kernel/skeleton`, run it after `skernel build`:

```bash
php bin/supervise.php
```

It prints:

```text
Worker 0 of the supervised pool
Supervisor finished
```

The manager is created inside the child, so the pool belongs to that child. Failures of the main process, such as a failing child, raise `SuperKernel\Swoole\Exception\SupervisorException`; failures of the manager raise `SuperKernel\Process\Exception\ProcessException`.

## Errors

| Situation | Message |
| --- | --- |
| A `workNums` outside `0` to `1024` | `Process count must be between 0 and 1024.` (`InvalidArgumentException`) |
| A `#[Process]` class that does not implement the interface | `<class> must implement SuperKernel\Process\Contract\ProcessInterface.` (`InvalidArgumentException`) |
| The container returns something else for a `#[Process]` class | `Invalid process container binding: <class>` (`InvalidArgumentException`) |
| A batch outside `0` to `1024` | `Worker count must be between 0 and 1024.` (`InvalidArgumentException`) |
| More than 1024 workers | `At most 1024 workers may be registered.` (`LogicException`) |
| An empty name | `Process names must not be empty.` (`LogicException`) |
| A name that is already registered | `Duplicate process name: <name>.` (`LogicException`) |
| `SWOOLE_IPC_SOCKET` or another unsupported IPC type | `Worker IPC must be disabled or use Unix domain sockets or a System V message queue. Network sockets require listen().` (`InvalidArgumentException`) |
| `get()` with an unknown name | `Process not found: "<name>".` (`ProcessNotFoundException`) |
| A change or start from another process | `Only the creating process may configure or start this manager.` (`LogicException`) |
| A change or start after the first start | `The manager may only start once; registrations and settings are now closed.` (`LogicException`) |
| A start inside a coroutine or after asynchronous work | `Start the Pool in a clean CLI before reactors or asynchronous work.` (`ProcessException`) |
| A pool started inside a worker | `A process pool instance has already been created and cannot be created again` (Swoole's `Error`) |
| Swoole cannot start the pool | `Unable to start the process Pool.` (`ProcessException`) |
| Shutdown cannot be requested | `Unable to request Pool shutdown.` (`ProcessException`) |

`ProcessException` and `ProcessNotFoundException` are in `SuperKernel\Process\Exception`; `ProcessNotFoundException` extends `OutOfBoundsException`, so call `has()` before `get()` when a name may be missing.

## Relation to other packages

- [`super-kernel/swoole`](swoole.md) runs `swoole:start` and the main process; this package mounts the pool as one of its children.
- [`super-kernel/server`](server.md) runs the server in another child of the same main process; its user processes are workers of the server, not of this pool.
- [`super-kernel/di`](di.md) creates the `#[Process]` classes and their dependencies; the declarations come from the build metadata of `skernel build`.
- [`super-kernel/logger`](logger.md) provides `StdoutLoggerInterface` for tasks that log.

## What not to do

**Do not start a second pool inside a worker.** Swoole allows only one pool in a process and its workers, and rejects another with `A process pool instance has already been created and cannot be created again`. Register every background process with the one manager instead.

**Do not return from `execute()` to finish a job.** A returning worker is started again while the pool runs, so a finite job runs over and over. Keep long-running work in a loop, and end a finite pool with `$pool->shutdown()` once the work is done.

**Do not request shutdown at the very start of the pool.** In Swoole 6.2.3, a `$pool->shutdown()` called as soon as worker `0` starts, before the other workers had started, was observed to leave the pool running. Request shutdown from work that runs after the pool is up, as `bin/maintenance.php` does from its last worker.

**Do not ignore `SIGTERM` in a worker.** Shutdown has no deadline: a worker that does not end on the signal keeps the pool, and `swoole:start`, from stopping. Wait in `Swoole\Coroutine::sleep()` in coroutine workers, or end blocking loops when the worker is signalled.

**Do not open connections in a process's constructor or properties.** The object is created in the parent before the workers are forked, so the connection would be shared by every worker or broken after the fork. Open it in `execute()`.

**Do not register processes after `MainProcessCreated`.** `swoole:start` copies the registrations when it dispatches the event; later ones never run. Declare processes with `#[Process]` or register them before the event.

**Do not run other child processes from the pool's parent.** The pool reaps every child of its parent, so the exit status of other children is lost. Run them in their own child of the main process.

**Do not use `SWOOLE_IPC_SOCKET`.** It is rejected with `Worker IPC must be disabled or use Unix domain sockets or a System V message queue. Network sockets require listen().` Use `SWOOLE_IPC_UNIXSOCK` or `SWOOLE_IPC_MSGQUEUE`.
