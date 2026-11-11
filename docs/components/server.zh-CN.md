<p align="right">
  <a href="server.md">English</a> | <strong>中文文档</strong>
</p>

# SuperKernel Server

包：[`super-kernel/server`](https://github.com/super-kernel/server) · [Packagist](https://packagist.org/packages/super-kernel/server) · [文档索引](../README.zh-CN.md)

## 用途

`super-kernel/server` 把应用的服务器配置变成一个正在运行的 Swoole 服务器。它创建原生服务器及其监听器，给协议包和应用一个挂上回调的时机，把 Swoole 的主回调作为事件发布，挂上为服务器声明的用户进程，然后启动它。

本包不处理任何协议：读取连接的回调由 `super-kernel/http-server` 和应用挂上。它不提供命令，也不监管进程：`super-kernel/swoole` 的 `swoole:start` 在其主进程的子进程中启动服务器，守护进程模式、PID 文件和信号都由该主进程负责。

## 能力

### 配置服务器

应用声明一个实现 `SuperKernel\Server\Config\SwooleServerConfigInterface` 的类，并用 `#[Configuration(SwooleServerConfigInterface::class)]` 标记它。容器校验该类，然后在每个注入 `SwooleServerConfigInterface` 的地方提供同一个对象。需要时会再次调用这些获取方法，因此请保持它们的结果稳定。

- `enable()` 决定 `swoole:start` 是否启动服务器。没有声明配置时，不会启动任何内容，也不会读取配置；声明了的配置即使在 `enable()` 返回 `false` 时也会被校验。`enable()` 不影响对 `ServerInterface::start()` 的直接调用。
- `mode()` 返回一个 `Mode`，参见[选择模式](#选择模式)。
- `listeners()` 返回由监听器数组组成的非空列表。第一个是主监听器；其他监听器按顺序添加。
- `settings()` 返回以字符串为键的 Swoole 服务器设置。这些设置由 Swoole 自己校验并报告失败。

监听器数组包含以下字段；任何其他键都会被拒绝，显式的 `null` 也会被拒绝：

| 字段 | 类型 | 必需 | 省略时使用 |
| --- | --- | --- | --- |
| `name` | 非空 `string` | 是 | |
| `host` | 非空 `string` | 是 | |
| `port` | `int` | 是 | |
| `socketType` | `int` | 否 | `SWOOLE_SOCK_TCP` |
| `ssl` | `bool` | 否 | `false` |
| `reusePort` | `bool` | 否 | `false` |
| `protocol` | `Protocol` | 否 | `Protocol::Raw` |

监听器名称在事件和日志记录中标识监听器；它区分大小写，并且必须唯一。这些数组会原样到达事件，与配置返回的完全一致：省略的字段不会被补上。在 base、process 和 thread 模式下，主监听器必须具有所有监听器中最高的协议能力，顺序为 `WebSocket`、`Http`、`Raw`，因为主监听器决定创建哪个原生服务器类；TLS 是 `socketType` 的一部分，例如 `SWOOLE_SOCK_TCP | SWOOLE_SSL`。协程模式只允许一个使用 `Raw` 协议的监听器，并使用它的 `ssl` 和 `reusePort` 字段。

带有一个 HTTP 监听器和一个原始监听器的配置，保存为 `src/Config/SwooleServerConfig.php`：

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

### 选择模式

| 模式 | 原生服务器 | 说明 |
| --- | --- | --- |
| `Mode::Base` | 由主监听器的协议决定的 `Swoole\Server`、`Swoole\Http\Server` 或 `Swoole\WebSocket\Server` | Swoole 的 `SWOOLE_BASE`。 |
| `Mode::Process` | 与 `Base` 相同 | Swoole 的 `SWOOLE_PROCESS`。 |
| `Mode::Thread` | 与 `Base` 相同 | Swoole 的 `SWOOLE_THREAD`；需要 ZTS 版 PHP 和支持线程的 Swoole 构建。 |
| `Mode::Coroutine` | `Swoole\Coroutine\Server` | 一个原始监听器；没有主回调，也没有用户进程。 |

在 base、process 和 thread 模式下，容器以 `Swoole\Server` 提供原生服务器，此时设置已经应用，额外的监听器已经添加，主回调和启动日志记录也已经注册。在协程模式下，它改为提供 `Swoole\Coroutine\Server`。Server 把协程监听器视为主监听器，并且不为它设置 `handle()` 回调：服务连接的代码在 `AfterServerCreated` 监听器中设置它。

### 挂上协议处理器

`ServerInterface::start()` 在服务器启动之前同步地按以下顺序发布两个事件：

1. `AfterMainServerCreated` 一次；`server()` 返回原生服务器。
2. 为每个监听器发布 `AfterServerCreated` 一次，主监听器最先；对主监听器，`server()` 返回原生服务器，对其他监听器则返回其 `Swoole\Server\Port`，`configuration()` 返回该监听器的数组。

请在这些监听器中挂上读取连接的回调，例如 `receive` 或 `request`。用 `configuration()['name']` 读取监听器的名称，用 `configuration()['protocol'] ?? Protocol::Raw` 读取它的协议。监听器抛出的异常会阻止启动。

在上面配置的 `echo` 监听器上进行应答的监听器，保存为 `src/Listener/AttachEcho.php`：

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

客户端，保存为 `bin/echo-client.php`：

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

构建项目，启动它，然后在另一个终端中运行客户端：

```bash
skernel build
php bin/bootstrap.php swoole:start
php bin/echo-client.php
```

客户端会打印 `echo: hello`。

### 响应服务器回调

Server 注册原生服务器的全部 14 个主回调，并通过 `Psr\EventDispatcher\EventDispatcherInterface` 把每一个作为事件发布。这些事件位于 `SuperKernel\Server\Event\Callbacks`，以公开属性携带 Swoole 的参数：

| 事件 | Swoole 回调 | 属性 |
| --- | --- | --- |
| `OnStart` | `start` | `server` |
| `OnBeforeShutdown` | `beforeShutdown` | `server` |
| `OnShutdown` | `shutdown` | `server` |
| `OnManagerStart` | `managerStart` | `server` |
| `OnManagerStop` | `managerStop` | `server` |
| `OnWorkerStart` | `workerStart` | `server`、`workerId` |
| `OnWorkerStop` | `workerStop` | `server`、`workerId` |
| `OnWorkerExit` | `workerExit` | `server`、`workerId` |
| `OnWorkerError` | `workerError` | `server`、`status`、`workerPid`、`exitCode`、`signal` |
| `OnTask` | `task` | `server`、`task`、`sourceWorkerId`、`data`，以及可写的 `result` |
| `OnFinish` | `finish` | `server`、`result`、`data` |
| `OnPipeMessage` | `pipeMessage` | `server`、`message`、`data` |
| `OnBeforeReload` | `beforeReload` | `server` |
| `OnAfterReload` | `afterReload` | `server` |

当 Swoole 既可能传入一个对象、也可能传入多个单独的值时（例如启用 `task_enable_coroutine` 时的任务），事件保留 Swoole 的形式：`task` 是一个 `Swoole\Server\Task` 或任务 ID，后面的属性只在第二种形式中才有值。`OnTask` 的监听器通过给 `$event->result` 赋值来答复任务；Server 把该值返回给 Swoole。监听器抛出的异常会传播到原生回调。

README 的快速开始监听了 `OnStart`。服务器真正启动时，Server 在 `OnStart` 监听器运行之后通过 `StdoutLoggerInterface` 写出以下记录；在协程模式下，它在协程服务器运行之后写出这些记录：

```text
[NOTICE] Swoole main server started in process mode.
[INFO] Swoole server process started with PID 958085.
[INFO] Swoole listener "http" started on 127.0.0.1:9501 (http).
[INFO] Swoole listener "echo" started on 127.0.0.1:9502 (raw).
```

只有当容器通过 Server 自己的提供者创建 `Swoole\Server` 时，才会注册这些回调和记录。`receive`、`request` 和 `message` 等连接回调属于协议处理器。

### 声明用户进程

用户进程是 Swoole 服务器通过 `Swoole\Server::addProcess()` 在它自己的工作进程旁边启动、并在其结束时重新启动的工作进程。用 `#[SuperKernel\Server\Attribute\UserProcess]` 标记一个类，并实现 `SuperKernel\Server\Contract\UserProcessInterface`：

- 该注解接收 `name`（默认为类名）、`workNums`（默认为 `1`，范围为 `0` 到 `1024`）和 `enableCoroutine`（默认为 `false`）。有多个工作进程时，名称为 `name:0`、`name:1`，依此类推。
- 该接口声明四个可读属性：`enabled`，为 `false` 时排除该类；`redirectStdinStdout`；`pipeType`，取值为 `Swoole\Process::PIPE_TYPE_NONE`、`PIPE_TYPE_STREAM` 或 `PIPE_TYPE_DGRAM` 之一；`enableCoroutine`，注解的 `enableCoroutine` 会覆盖它，但不会修改对象。
- `execute(Swoole\Process $process, Swoole\Server $server)` 在工作进程中运行。从中返回会结束该工作进程，Swoole 服务器可能会再次启动它。

该类在工作进程存在之前、在服务器的父进程中创建。请让它的构造函数和属性不涉及连接和其他资源，并在 `execute()` 中打开它们。

每秒报告一次的用户进程，保存为 `src/Worker/Heartbeat.php`：

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

运行 `skernel build` 之后，`php bin/bootstrap.php swoole:start` 每秒打印一行 `Heartbeat from worker …`，直到服务器停止。

容器提供的 `SuperKernel\Server\Contract\UserProcessManagerInterface` 已经按构建发现它们的顺序注册了每个 `#[UserProcess]` 类。`workNums` 为 `0` 的类不会被创建；被禁用的类会被创建一次，但不会添加工作进程。在服务器启动之前运行的代码（例如 `AfterMainServerCreated` 监听器）可以注册更多：

- `add($process, $enableCoroutine = null, $name = null)` 注册一个工作进程；`addBatch($workerNum, $process, $enableCoroutine = null, $name = null)` 一次注册 `0` 到 `1024` 个工作进程，只要其中任何一个被拒绝，就一个也不注册。二者都返回管理器。
- `$process` 是一个 `UserProcessInterface` 或 `callable(Swoole\Process, Swoole\Server): void`。可调用对象在不启用协程、不重定向、使用数据报管道的情况下运行；要选择其他选项，请用 `SuperKernel\Server\Definition\UserProcessDefinition` 包装它。为 `null` 的 `$enableCoroutine` 保留进程自己的选择。
- 没有名称时，工作进程以其注册序号命名，从 `0` 开始。名称区分大小写并且必须唯一，管理器最多保存 1024 个工作进程。
- `all()` 按顺序返回已注册的进程，以名称为键；`has($name)` 报告某个名称是否已注册；`get($name)` 返回该进程，或者抛出 `SuperKernel\Server\Exception\UserProcessNotFoundException`。

管理器在注册进程时捕获其选项，此后不再读取它们。在创建事件和 `daemonize` 重置之后，Server 按顺序为每个已注册的进程挂上一个原生工作进程，然后启动服务器；之后注册的进程不会被挂上。管理器没有启动或停止操作；工作进程的启动、回收和重启由 Swoole 服务器负责，哪些模式可以使用用户进程也由它决定。协程服务器不会挂上用户进程。工作进程名称不是操作系统的进程标题。

### 自行启动服务器

`SuperKernel\Server\Contract\ServerInterface` 就是 `swoole:start` 在其子进程中启动的对象。脚本可以不经过主进程直接启动服务器。保存为 `bin/serve.php`：

```php
<?php

declare(strict_types=1);

use SuperKernel\Context\ApplicationContext;
use SuperKernel\Server\Contract\ServerInterface;

require dirname(__DIR__) . '/vendor/autoload.php';

ApplicationContext::getContainer()->get(ServerInterface::class)->start();
```

运行 `skernel build` 之后运行它：

```bash
php bin/serve.php
```

无论 `enable()` 返回什么，它都会在前台启动服务器，并阻塞到服务器停止；按 Ctrl+C 停止它。`ServerInterface` 只接受一次启动：第二次调用会抛出 `The configured Swoole service may only be started once.`，启动失败之后也是如此。在 base、process 和 thread 模式下，`start()` 在服务器启动之前把 `daemonize` 设为 `false`，覆盖设置以及事件监听器所做的任何修改；需要在后台运行时，请改用 `super-kernel/swoole` 的守护进程模式。在协程模式下，如果 `start()` 不在协程中，它会启动协程调度器，并阻塞到服务器停止。`getServer()` 每次都返回同一个原生服务器，供需要 Swoole 自身 API 的代码使用。

## 错误

本包的每个异常都实现 `SuperKernel\Server\Exception\SwooleServerExceptionInterface`。

| 情形 | 消息 |
| --- | --- |
| 没有声明配置，而代码需要它 | `No configuration source is registered for SuperKernel\Server\Config\SwooleServerConfigInterface.` |
| 监听器缺少必需字段 | `Listener configuration at position <n> is missing required field "<field>".` |
| 监听器含有未知字段 | `Listener configuration at position <n> contains unknown field "<field>".` |
| 监听器字段类型错误 | `Listener field "<field>" at position <n> must be <type>; <type> provided.` |
| 名称或主机为空 | `Listener configuration at position <n> must define a non-empty <field>.` |
| 两个监听器使用同一个名称 | `Listener name "<name>" at position <n> duplicates the listener at position <n>.` |
| 没有监听器 | `The Swoole server configuration must define at least one listener.` |
| 后面的监听器需要比主监听器更高的协议 | `Listener "<name>" at position <n> requires the "<protocol>" protocol, which exceeds primary listener "<name>" using "<protocol>"; place a listener with the highest protocol capability first.` |
| 协程模式下有多个监听器，或者使用其他协议 | `Swoole coroutine mode supports exactly one listener.` 或 `Swoole coroutine mode supports only the raw listener protocol; listener "<name>" requires "<protocol>".` |
| 设置使用整数键 | `Swoole server settings must use string keys.` |
| 不支持线程时使用线程模式 | `Swoole thread mode requires a ZTS PHP runtime and an ext-swoole build with thread support.` |
| 无法创建原生服务器，例如端口已被占用 | `Unable to resolve the configured Swoole service from container entry Swoole\Server.`，原因例如 `Unable to create the Swoole process-mode server for listener "<name>".` |
| 无法注册主回调 | `Unable to register Swoole Server event "<event>" through callback handler <class>.` |
| 第二次启动 | `The configured Swoole service may only be started once.` |
| 无效的 `#[UserProcess]` | `UserProcess name must not be empty.` 或 `UserProcess count must be between 0 and 1024.` |
| 没有实现该接口的 `#[UserProcess]` 类 | `<class> must implement SuperKernel\Server\Contract\UserProcessInterface.` |
| 重复的工作进程名称、空白名称或工作进程过多 | `Duplicate user process name: <name>.`、`User process names must not be blank.` 或 `At most 1024 user workers may be registered.` |
| 无效的管道类型 | `Invalid user process pipe type: <type>` |
| 用未知名称调用 `get()` | `User process not found: "<name>".` |

配置错误是 `SuperKernel\Server\Exception\InvalidServerConfigurationException`，创建错误是 `ServerCreationException`，启动错误是 `ServerStartException`，模式错误是 `UnsupportedServerModeException`，它们都位于 `SuperKernel\Server\Exception`；用户进程错误是 `InvalidArgumentException` 和 `LogicException`。当容器创建配置或服务器时，会把这些异常包装为 `SuperKernel\Di\Exception\ResolutionException` 的前一个异常（previous）。

## 与其他包的关系

- [`super-kernel/swoole`](swoole.zh-CN.md) 运行 `swoole:start`；本包监听它的 `MainProcessCreated` 事件，并挂载一个启动服务器的子进程。
- [`super-kernel/http-server`](http-server.zh-CN.md) 在 `AfterServerCreated` 监听器中挂上 HTTP 的 `request` 回调。
- [`super-kernel/process`](process.zh-CN.md) 在服务器旁边的独立进程池中运行后台进程；本包的用户进程则属于服务器。
- [`super-kernel/config`](config.zh-CN.md) 找到 `#[Configuration]` 配置源，[`super-kernel/logger`](logger.zh-CN.md) 为启动记录提供 `StdoutLoggerInterface` 服务。
- [`super-kernel/event-dispatcher`](event-dispatcher.zh-CN.md) 把这些事件交给 `skernel build` 发现的监听器。

## 不要这样做

**不要用 `on()` 设置 `start` 或 `workerStart` 等主回调。** Server 注册了全部 14 个主回调并把它们作为事件发布；再次设置某个回调会替换 Server 的处理器，于是它的事件以及（对 `start` 而言）启动记录都会消失。请改为监听对应的 `On*` 事件。

**不要用自己的提供者提供 `Swoole\Server`。** 为 `Swoole\Server` 提供的提供者会绕过 Server 的提供者，同时绕过主回调、它们的事件和启动记录。请通过配置和创建事件来修改服务器。

**不要用 `#[Provider]` 或 `#[FactoryProvider]` 替换配置。** 本包自己的工厂会校验配置；为 `SwooleServerConfigInterface` 提供另一个提供者会跳过校验，并与该工厂冲突。请用 `#[Configuration(SwooleServerConfigInterface::class)]` 声明该类。

**不要在设置中启用 `daemonize`。** Server 会在服务器启动之前把它重置为 `false`。需要在后台运行时，请使用 `super-kernel/swoole` 的守护进程模式。

**不要在用户进程的构造函数或属性中打开连接。** 该对象在工作进程存在之前于父进程中创建，因此连接会被每个工作进程共享，或者在派生（fork）之后失效。请在 `execute()` 中打开它。

**不要在 `execute()` 中启动、等待进程或把进程转为守护进程。** 用户工作进程归 Swoole 服务器所有：不要在其中调用 `Process::start()`、`wait()`、`daemon()` 或 `Server::start()`。

**不要在服务器启动之后注册用户进程。** 它们只在启动之前被挂上一次；之后的注册只会改变管理器的列表。
