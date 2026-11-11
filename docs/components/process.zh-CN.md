<p align="right">
  <a href="process.md">English</a> | <strong>中文文档</strong>
</p>

# SuperKernel Process

包：[`super-kernel/process`](https://github.com/super-kernel/process) · [Packagist](https://packagist.org/packages/super-kernel/process) · [文档索引](../README.zh-CN.md)

## 用途

`super-kernel/process` 运行属于服务器旁边、而不属于请求内部的工作：队列消费者、调度器、维护循环。每个后台进程都是一个带有一个或多个工作进程的命名条目，所有条目都作为同一个 `Swoole\Process\Pool` 的工作进程运行。它的管理器遵循 Swoole [`Process\Manager`](https://github.com/swoole/library/blob/master/src/core/Process/Manager.php) 的 PHP 实现：每次启动一个进程池，每个已注册条目一个工作进程，并按条目选择直接执行或协程执行。

在 `swoole:start` 下，进程池在 `super-kernel/swoole` 主进程的独立子进程中运行，与 `super-kernel/server` 的服务器并列；守护进程模式、PID 文件和信号由主进程负责。服务器自己的用户进程属于 `super-kernel/server`。

## 能力

### 声明后台进程

用 `#[SuperKernel\Process\Attribute\Process]` 标记一个类，并实现 `SuperKernel\Process\Contract\ProcessInterface`：

- 该注解接收 `name`（默认为类名，即使容器返回的是另一个实现）、`workNums`（默认为 `1`，范围为 `0` 到 `1024`）和 `enableCoroutine`（默认为 `null`）。有多个工作进程时，名称为 `name:0`、`name:1`，依此类推。
- 该接口声明两个可读属性 `enabled` 和 `enableCoroutine`，以及在工作进程中运行的 `execute(Swoole\Process\Pool $pool, int $workerId): void`。
- `enabled` 为 `false` 的类不会添加工作进程，也不占用名称。`workNums: 0` 的类根本不会被创建；其他类会在父进程中从容器创建一次，以读取它的属性。

该类的创建及其属性的读取发生在派生（fork）工作进程之前。请让它的构造函数和属性不涉及连接、事件循环和其他资源，并在 `execute()` 中打开它们。

容器提供的 `SuperKernel\Process\Contract\ProcessManagerInterface` 已经按构建发现它们的顺序注册了每个 `#[Process]` 类。准备它时从不运行任务，也不启动进程池。README 的快速开始就是这样声明进程的。

骨架的 `Src\Process\TestProcess` 就是这样的声明，并保持禁用：它的 `enabled` 属性为 `false`。启用后，它在协程中通过 `Swoole\Coroutine::sleep(30)` 每 30 秒用 `StdoutLoggerInterface` 记录一次心跳，并且从不返回，因为返回的工作进程会被替换。启用之前，请把心跳替换为应用的工作。

### 选择直接执行或协程执行

进程池本身不使用协程运行。每个条目的有效 `enableCoroutine` 决定它的任务是直接运行，还是在 `Swoole\Coroutine\run($callback, $pool, $workerId)` 中运行。注解的 `enableCoroutine` 或管理器的 `$enableCoroutine` 参数只有在为 `true` 或 `false` 时才会覆盖任务的属性；`null` 保留该属性：

| 覆盖值 | 属性为 `false` | 属性为 `true` |
| --- | --- | --- |
| 省略或 `null` | 直接 | 协程 |
| `false` | 直接 | 直接 |
| `true` | 协程 | 协程 |

覆盖不会改变任务对象：管理器把任务包装在带有有效设置的 `SuperKernel\Process\Definition\ProcessDefinition` 中，`all()` 和 `get()` 返回该包装。没有覆盖时，保留注册的对象本身。自行注册声明的集成以 `$attribute->enableCoroutine ?? $task->enableCoroutine` 计算该设置。

### 用管理器注册进程

`SuperKernel\Process\Manager\ProcessManager` 实现 `ProcessManagerInterface`，可以在没有容器的情况下使用：

| 方法 | 行为 |
| --- | --- |
| `add($process, $enableCoroutine = null, $name = null)` | 注册一个工作进程并返回管理器。 |
| `addBatch($workerNum, $process, $enableCoroutine = null, $name = null)` | 一次注册 `0` 到 `1024` 个工作进程，只要其中任何一个被拒绝就一个也不注册，并返回管理器。 |
| `all()` | 按工作进程 ID 的顺序返回已注册的任务，以名称为键。 |
| `has($name)` | 报告某个名称是否已注册；它不说明工作进程是否在运行。 |
| `get($name)` | 返回已注册的任务，每次都是同一个对象，在其工作进程被替换之后也是如此。 |
| `setIPCType()` / `getIPCType()` | 设置或读取进程池的 IPC 模式。 |
| `setMsgQueueKey()` / `getMsgQueueKey()` | 设置或读取进程池的 System V 消息队列键。 |
| `start($beforeStart = null, $afterFork = null)` | 启动进程池，并在关闭之后、每个工作进程都被回收时返回。 |
| `stop()` | 从父进程或工作进程请求关闭整个进程池。 |

`$process` 是一个 `ProcessInterface` 或 `callable(Swoole\Process\Pool, int): void`。可调用对象会被包装在 `ProcessDefinition` 中，除非 `$enableCoroutine` 为 `true`，否则直接运行；`new ProcessDefinition($callback, enableCoroutine: true, enabled: true)` 可以自行创建这样的包装。`enabled` 为 `false` 的任务会被 `add()` 和 `addBatch()` 跳过，不占用名称或工作进程 ID。没有名称时，工作进程以其注册序号命名，从 `0` 开始；有名称的单个工作进程保留其名称。名称区分大小写，不能为空，可以是完整的类名；管理器最多保存 1024 个工作进程。`add()`、`addBatch()` 和设置方法可以链式调用。

只有创建管理器的进程可以修改或启动它，而第一次 `start()` 会关闭注册和设置，即使没有注册任何内容或启动失败也是如此。没有注册任何内容的管理器，或者在启动之前就被停止的管理器，不会创建进程池。`start()` 在进程池准备好之后、启动之前于父进程中调用 `$beforeStart`，并在每个工作进程中于其任务之前调用 `$afterFork`；在 `$beforeStart` 中调用 `stop()` 会取消启动。

运行一个由两个协程工作进程和一个维护工作进程组成的进程池的脚本，保存为 `bin/maintenance.php`：

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

运行脚本：

```bash
php bin/maintenance.php
```

它会打印：

```text
mailer:0, mailer:1, maintenance
Maintenance ran in worker 2
Pool stopped
```

### 进程池的生命周期

请在干净的 CLI 中、任何协程、事件循环或异步工作之前启动进程池。进程池运行期间，每当任务返回、失败或什么也不做时，Swoole 都会再次启动该工作进程：有限的作业在返回时并不算"完成"。没有重试次数限制，没有延迟，也没有一个工作进程失败就停止其他工作进程的规则。应当结束进程池的作业需要与其他工作进程协调，然后请求关闭。参见[原生进程池生命周期](https://github.com/swoole/swoole-src/blob/v6.2.3/src/os/process_pool.cc)。

工作进程中的 `$pool->shutdown()`、管理器的 `stop()`，或者向进程池父进程发送的 `SIGTERM`，都会停止整个进程池：Swoole 向工作进程发送 `SIGTERM`，每个任务都必须结束并完成清理。没有期限，也不会转发 `SIGINT`，因此忽略该信号的工作进程运行多久，关闭就延迟多久。Swoole 的 `Pool::start()` 返回之后，管理器会等待其余工作进程退出。进程池会回收其父进程的每个子进程，因此不要从进程池的父进程运行其他子进程；它们的退出状态会丢失。关闭之后 `all()`、`has()` 和 `get()` 仍然可用，但它们从不报告工作进程是否存活。

### 工作进程之间的通信

进程池的 IPC 作用于每个工作进程，无论是否使用协程。默认为 `SWOOLE_IPC_NONE`；也接受 `SWOOLE_IPC_UNIXSOCK` 和 `SWOOLE_IPC_MSGQUEUE`，消息队列键默认为 `0`，即私有队列。用 `new ProcessManager(ipcType: SWOOLE_IPC_MSGQUEUE, msgQueueKey: 0x5301)` 或设置方法设置它们；构造函数不接收其他内容。`SWOOLE_IPC_SOCKET` 会被拒绝，因为它需要在启动之前执行管理器没有提供的 `listen()` 步骤。工作进程使用它们收到的 `Pool` 的 IPC 方法；没有按工作进程划分的管道，工作进程的端点在它被替换后也不会保留。

### 在主进程下运行

`swoole:start` 分发 `MainProcessCreated`；本包以最低优先级监听，在其他监听器之后运行。它从 `ProcessManagerInterface` 复制已注册的任务、它们的名称和协程设置以及 IPC 设置，并在主进程上挂载一个子进程。该子进程根据副本创建新的管理器并启动进程池。没有注册任何内容时，不会挂载任何内容。事件之后对原管理器的注册、IPC 修改或 `stop()` 不会影响正在运行的进程池。

进程池的父进程是主进程的一个子进程：它和工作进程都不写 PID 文件，PID 文件保存的是主进程的 PID。向主进程发送 `SIGTERM` 或 `SIGINT` 会向其子进程发送 `SIGTERM`；进程池的父进程停止并回收它的工作进程后返回，主进程再回收它。当服务器的配置启用服务器时，服务器在进程池旁边的另一个子进程中运行，永远不会成为进程池的工作进程。`super-kernel/swoole` 的 `ProcessRuntime::start($config)` 不经过命令启动同样的主进程。

直接用主进程监管进程池的脚本，保存为 `bin/supervise.php`：

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

在由 `super-kernel/skeleton` 创建的应用中，运行 `skernel build` 之后运行它：

```bash
php bin/supervise.php
```

它会打印：

```text
Worker 0 of the supervised pool
Supervisor finished
```

管理器在子进程中创建，因此进程池属于该子进程。主进程的失败（例如子进程失败）会抛出 `SuperKernel\Swoole\Exception\SupervisorException`；管理器的失败会抛出 `SuperKernel\Process\Exception\ProcessException`。

## 错误

| 情形 | 消息 |
| --- | --- |
| `workNums` 不在 `0` 到 `1024` 之间 | `Process count must be between 0 and 1024.`（`InvalidArgumentException`） |
| 没有实现该接口的 `#[Process]` 类 | `<class> must implement SuperKernel\Process\Contract\ProcessInterface.`（`InvalidArgumentException`） |
| 容器为 `#[Process]` 类返回了其他内容 | `Invalid process container binding: <class>`（`InvalidArgumentException`） |
| 批量数量不在 `0` 到 `1024` 之间 | `Worker count must be between 0 and 1024.`（`InvalidArgumentException`） |
| 超过 1024 个工作进程 | `At most 1024 workers may be registered.`（`LogicException`） |
| 空名称 | `Process names must not be empty.`（`LogicException`） |
| 已经注册的名称 | `Duplicate process name: <name>.`（`LogicException`） |
| `SWOOLE_IPC_SOCKET` 或其他不支持的 IPC 类型 | `Worker IPC must be disabled or use Unix domain sockets or a System V message queue. Network sockets require listen().`（`InvalidArgumentException`） |
| 用未知名称调用 `get()` | `Process not found: "<name>".`（`ProcessNotFoundException`） |
| 从另一个进程修改或启动 | `Only the creating process may configure or start this manager.`（`LogicException`） |
| 第一次启动之后修改或启动 | `The manager may only start once; registrations and settings are now closed.`（`LogicException`） |
| 在协程中或异步工作之后启动 | `Start the Pool in a clean CLI before reactors or asynchronous work.`（`ProcessException`） |
| 在工作进程中启动进程池 | `A process pool instance has already been created and cannot be created again`（Swoole 的 `Error`） |
| Swoole 无法启动进程池 | `Unable to start the process Pool.`（`ProcessException`） |
| 无法请求关闭 | `Unable to request Pool shutdown.`（`ProcessException`） |

`ProcessException` 和 `ProcessNotFoundException` 位于 `SuperKernel\Process\Exception`；`ProcessNotFoundException` 继承 `OutOfBoundsException`，因此名称可能缺失时，请在 `get()` 之前调用 `has()`。

## 与其他包的关系

- [`super-kernel/swoole`](swoole.zh-CN.md) 运行 `swoole:start` 和主进程；本包把进程池挂载为它的一个子进程。
- [`super-kernel/server`](server.zh-CN.md) 在同一个主进程的另一个子进程中运行服务器；它的用户进程是服务器的工作进程，而不是本进程池的。
- [`super-kernel/di`](di.zh-CN.md) 创建 `#[Process]` 类及其依赖；这些声明来自 `skernel build` 的构建元数据。
- [`super-kernel/logger`](logger.zh-CN.md) 为需要记录日志的任务提供 `StdoutLoggerInterface`。

## 不要这样做

**不要在工作进程中启动第二个进程池。** Swoole 只允许一个进程及其工作进程中有一个进程池，并以 `A process pool instance has already been created and cannot be created again` 拒绝另一个。请改为用同一个管理器注册每个后台进程。

**不要通过从 `execute()` 返回来结束作业。** 进程池运行期间，返回的工作进程会被再次启动，因此有限的作业会一遍又一遍地运行。请把长时间运行的工作放在循环中，并在工作完成后用 `$pool->shutdown()` 结束有限的进程池。

**不要在进程池刚启动时就请求关闭。** 在 Swoole 6.2.3 中，观察到在工作进程 `0` 一启动、其他工作进程尚未启动时调用 `$pool->shutdown()`，会让进程池继续运行。请在进程池启动之后运行的工作中请求关闭，就像 `bin/maintenance.php` 在它的最后一个工作进程中所做的那样。

**不要在工作进程中忽略 `SIGTERM`。** 关闭没有期限：不因该信号而结束的工作进程会使进程池以及 `swoole:start` 无法停止。在协程工作进程中用 `Swoole\Coroutine::sleep()` 等待，或者在工作进程收到信号时结束阻塞循环。

**不要在进程的构造函数或属性中打开连接。** 该对象在派生工作进程之前于父进程中创建，因此连接会被每个工作进程共享，或者在派生之后失效。请在 `execute()` 中打开它。

**不要在 `MainProcessCreated` 之后注册进程。** `swoole:start` 在分发该事件时复制注册；之后的注册永远不会运行。请用 `#[Process]` 声明进程，或者在该事件之前注册它们。

**不要从进程池的父进程运行其他子进程。** 进程池会回收其父进程的每个子进程，因此其他子进程的退出状态会丢失。请在主进程的独立子进程中运行它们。

**不要使用 `SWOOLE_IPC_SOCKET`。** 它会以 `Worker IPC must be disabled or use Unix domain sockets or a System V message queue. Network sockets require listen().` 被拒绝。请使用 `SWOOLE_IPC_UNIXSOCK` 或 `SWOOLE_IPC_MSGQUEUE`。
