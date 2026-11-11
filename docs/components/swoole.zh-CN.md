<p align="right">
  <a href="swoole.md">English</a> | <strong>中文文档</strong>
</p>

# SuperKernel Swoole

包：[`super-kernel/swoole`](https://github.com/super-kernel/swoole) · [Packagist](https://packagist.org/packages/super-kernel/swoole) · [文档索引](../README.zh-CN.md)

## 用途

`super-kernel/swoole` 启动 SuperKernel 应用的长时间运行的工作。它的 `swoole:start` 命令创建一个主进程，让每个已安装的包和应用在它上面挂载工作，然后监管这些工作：初始化回调在主进程中运行，每个运行时（例如 HTTP 服务器或进程池）在各自的子进程中与其他运行时并列运行。主进程负责守护进程模式、PID 文件、信号和关闭。

子进程中运行什么由挂载它的包负责：`super-kernel/server` 挂载服务器，`super-kernel/process` 挂载进程池。本包从不重启子进程，也不保存进程注册表。

## 能力

### 启动运行时

`swoole:start` 不接收参数，也没有自己的选项；Symfony Console 的帮助和输出选项照常可用。创建命令、列出命令或显示它的帮助不会解析任何内容。运行时，该命令：

1. 从容器解析 `MainProcess`，这会读取并校验 `SwooleConfigInterface` 配置；
2. 同步分发 `new MainProcessCreated($mainProcess)`，使每个监听器挂载自己的工作；
3. 启动同一个主进程，并在每个已挂载的子进程都退出并被回收后返回。

配置失败会在事件分发之前终止命令，即使没有任何内容会被挂载也是如此。每次运行应用时该命令只运行一次；主进程会拒绝第二次启动。

`SuperKernel\Swoole\Runtime\ProcessRuntime` 不经过命令完成同样的事情：它在构造函数中接收 PSR-14 事件调度器和 `StdoutLoggerInterface` 日志记录器，`start(SwooleConfigInterface $config)` 创建新的 `MainProcess`、分发其事件并启动它。

其他包可以用 `super-kernel/command` 的 `#[Command]` 注解在 `swoole:` 下声明更多命令，例如 `swoole:watch`。本包不保留这个前缀；每个完整的命令名称和别名都必须唯一。

### 挂载工作

`SuperKernel\Swoole\Event\MainProcessCreated` 的监听器通过事件的 `mainProcess` 属性接收主进程；事件不携带其他内容。分发事件时，任何已挂载的内容都还没有运行。

- `add(callable $callback)` 挂载在主进程中运行的初始化回调。
- `addProcess(callable $callback)` 挂载运行该回调的子进程。

`start()` 按挂载顺序把每个已挂载的回调各运行一次，并忽略它们的返回值；回调不接收参数。初始化回调必须返回之后，下一个已挂载的项才会开始。子进程按挂载顺序创建，然后并发运行。子进程没有通往主进程的管道，也没有协程调度器：需要协程的子进程自己启动调度器，并在自己的回调中而不是在监听器中创建服务器、进程池、连接和信号处理器。正常返回的子进程会被回收，不会被替换。

只有创建主进程的代码可以挂载工作或启动它，并且必须在它启动之前。在 `start()` 开始之后挂载、从派生（fork）出的副本中挂载或第二次启动都会抛出 `LogicException`。

README 的快速开始挂载了一个初始化回调和一个子进程。下面的监听器挂载一个运行协程的子进程，保存为 `src/Listener/MountTicker.php`：

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

在由 `super-kernel/skeleton` 创建的应用中，构建项目并启动它：

```bash
skernel build
php bin/bootstrap.php swoole:start
```

它会在服务器的日志行旁边打印 `Tick 1`、`Tick 2` 和 `Tick 3`。随后计时子进程退出并被回收，而服务器继续运行；按 Ctrl+C 停止它。

### 配置运行时

应用声明一个实现 `SuperKernel\Swoole\Config\SwooleConfigInterface` 的类，并用 `#[Configuration(SwooleConfigInterface::class)]` 标记它。容器校验该类，然后在每个注入 `SwooleConfigInterface` 的地方提供同一个对象。需要时会再次调用这些获取方法，因此请在整个运行期间保持它们的结果稳定。

不带 PID 文件的前台应用配置，保存为 `src/Config/SwooleConfig.php`：

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

| 方法 | `super-kernel/skeleton` 中的值 | 含义 |
| --- | --- | --- |
| `daemon()` | `false` | 主进程在运行任何内容之前是否转入后台。 |
| `pidFile()` | 绝对路径形式的 `vendor/.cache/super-kernel.pid`，在 Phar 中为 `null` | PID 文件的绝对路径，或者表示不使用 PID 文件的 `null`；守护进程模式需要它。 |
| `displayErrors()` | `true` | 设置 `swoole.display_errors`，与 PHP 的 `display_errors` 和日志记录器无关。 |
| `socketBufferSize()` | `8388608`（8 MiB） | Swoole 默认的套接字缓冲区大小，单位为字节，范围为 `1` 到 `4294967295`。 |
| `enablePreemptiveScheduler()` | `false` | 在子进程启动其协程时启用抢占式调度。 |

本包没有自己的默认值：每个值都由应用的类提供。

读取配置不会改变进程中的任何内容。主进程在挂载了内容的情况下启动时，会在第一个回调之前、任何子进程存在之前，设置 INI 选项 `swoole.display_errors`、`swoole.socket_buffer_size` 和 `swoole.enable_preemptive_scheduler`，并把这三个值传给 `Swoole\Coroutine::set()`，从而刷新套接字默认值并替换之前的抢占式调度选项，其他协程选项保持不变。这不会启动任何协程或事件循环。子进程继承这些设置，并由每个子进程决定何时启动它的调度器。这些设置作用于整个进程，在 `start()` 返回后仍然有效；没有挂载任何内容时不会应用它们。

这些选项遵循 Swoole 6.2.3。`swoole.enable_coroutine` 不是 INI 指令。`swoole.use_shortname` 和 `swoole.enable_library` 在 PHP 启动时设置（在 `php.ini` 中或通过 `-d`），不属于该接口；之后修改 `swoole.enable_library` 无法卸载已经加载的库。参见 Swoole 的 [INI 注册与初始化](https://github.com/swoole/swoole-src/blob/v6.2.3/ext-src/php_swoole.cc) 和 [协程配置](https://github.com/swoole/swoole-src/blob/v6.2.3/ext-src/swoole_coroutine_scheduler.cc)。

### 以守护进程运行

要在后台运行，请让 `daemon()` 返回 `true`，并让 `pidFile()` 返回一个绝对路径。`super-kernel/skeleton` 的配置已经返回绝对路径，因此只需把 `daemon()` 改为 `return true;`。此后，命令在主进程转入后台后立即返回，而主进程会一直运行到被停止：

```bash
skernel build
php bin/bootstrap.php swoole:start
kill -TERM "$(cat vendor/.cache/super-kernel.pid)"
```

PID 文件遵守以下规则：

- 主进程在转入后台之前对该文件加排他锁；当另一个进程持有该锁时，它会以 `Cannot acquire PID file <path>.` 失败。
- 转入后台之后，它在第一个回调运行或第一个子进程启动之前，把自己最终的 PID 写入该文件。该文件从不保存服务器、进程池或工作进程的 PID。
- 子进程关闭它们继承的文件句柄，但不会释放该锁。主进程一直持有该锁，直到每个子进程都被回收，然后删除该文件。
- 命令成功返回或 PID 文件存在，并不表示子进程已经可以提供服务。

守护进程模式只把标准输入重定向到 `/dev/null`；标准输出和标准错误保持原样，因此如果终端会关闭，请在启动应用时重定向它们。

### 停止运行时

向 PID 文件中的 PID 发送 `SIGTERM` 或 `SIGINT`，或者在前台按 Ctrl+C。主进程按挂载顺序向它的每个仍在运行的子进程发送 `SIGTERM`，等待它们全部结束，然后删除 PID 文件。发送顺序并不决定子进程处理信号或退出的顺序。每个子进程停止它自己启动的内容：进程池的父进程停止并回收它的工作进程，服务器停止它自己的工作进程。主进程只等待它创建的子进程，因此同一个 PHP 进程的其他子进程会保留它们的退出状态。等待期间，它通过 PHP 的[信号掩码](https://www.php.net/manual/zh/function.pcntl-sigprocmask.php)阻塞这些信号并[等待它们](https://www.php.net/manual/zh/function.pcntl-sigwaitinfo.php)；`start()` 返回时会恢复原来的掩码。

当子进程以异常、非零退出码或关闭信号以外的信号结束时，主进程会停止并回收其他子进程，并抛出 `SuperKernel\Swoole\Exception\SupervisorException`。异常无法作为同一个 PHP 对象跨越进程边界，因此子进程会记录它并以退出码 1 退出。初始化回调抛出的异常会在子进程停止并被回收之后原样传出。

主进程没有重启策略，也没有关闭期限：忽略 `SIGTERM` 的子进程运行多久，关闭就延迟多久。进程池会替换它自己的工作进程，参见 [`super-kernel/process`](process.zh-CN.md)。`SIGKILL` 之后的重启和资源限制由应用之外的服务管理器负责。

### 日志与输出

主进程只记录失败，使用 `StdoutLoggerInterface` 服务的 error 级别：子进程中的异常（`Mounted process failed: <message>`），以及在守护进程模式下主进程自身的失败（`Main process failed: <message>`）。记录写到哪里由该服务决定；`super-kernel/logger` 把它们写到标准输出，它的文档页介绍了如何替换它。主进程不会捕获或转发子进程打印的内容，也没有日志文件设置。没有挂载任何内容时，`start()` 立即返回，不记录任何内容。

## 错误

| 情形 | 消息 |
| --- | --- |
| 没有声明配置源 | `No configuration source is registered for SuperKernel\Swoole\Config\SwooleConfigInterface.`（`SuperKernel\Config\Exception\ConfigurationNotFoundException`） |
| 相对的 PID 路径，或者没有 PID 路径的守护进程模式 | `Use an absolute PID file path; daemon mode requires one.`（`InvalidArgumentException`） |
| 不在 `1` 到 `4294967295` 之间的套接字缓冲区大小 | `Use a socket buffer size between 1 and 4294967295 bytes.`（`InvalidArgumentException`） |
| 启动开始之后挂载，或者从派生出的副本中挂载 | `Main process tasks must be added by their owner before startup.`（`LogicException`） |
| 第二次启动、从回调中启动或从派生出的副本中启动 | `The main process may only be started once by its owner.`（`LogicException`） |
| 在协程中或异步工作之后启动 | `Start MainProcess in a clean CLI before reactors or asynchronous work.` |
| PID 文件被另一个进程锁定 | `Cannot acquire PID file <path>.` |
| 转入后台失败 | `Unable to daemonize MainProcess.` |
| 子进程失败 | `A mounted process failed with exit code <code> and signal <signal>.` |
| Swoole INI 选项被拒绝 | `Unable to apply Swoole INI setting: <directive>.` |

没有标注类的消息属于 `SuperKernel\Swoole\Exception\SupervisorException`。配置错误在容器创建 `SwooleConfigInterface` 时抛出；无效值错误作为 `SuperKernel\Di\Exception\ResolutionException` 的前一个异常（previous）到达调用方。

## 与其他包的关系

- [`super-kernel/server`](server.zh-CN.md) 监听 `MainProcessCreated`，并在其配置启用服务器时挂载一个启动服务器的子进程。
- [`super-kernel/process`](process.zh-CN.md) 监听同一个事件，并在注册了进程时挂载另一个运行进程池的子进程。
- [`super-kernel/command`](command.zh-CN.md) 把 `swoole:start` 注册到 [`super-kernel/framework`](framework.zh-CN.md) 的控制台应用中。
- [`super-kernel/config`](config.zh-CN.md) 找到应用的 `#[Configuration]` 配置源。
- [`super-kernel/logger`](logger.zh-CN.md) 提供记录失败的 `StdoutLoggerInterface` 服务。
- [`super-kernel/event-dispatcher`](event-dispatcher.zh-CN.md) 把 `MainProcessCreated` 交给 `skernel build` 发现的监听器。

## 不要这样做

**不要在监听器或初始化回调中创建服务器、进程池或连接。** 它们会属于主进程，并被复制到每个子进程中。请在运行它们的 `addProcess()` 回调中创建它们。

**不要在启动协程或事件循环之后运行 `swoole:start`。** 主进程会派生子进程，必须在干净的 CLI 中启动，否则会抛出 `Start MainProcess in a clean CLI before reactors or asynchronous work.` 请在任何异步工作之前，从控制台命令启动它。

**不要在主进程启动之后挂载工作。** 迟到的 `add()` 或 `addProcess()`（包括在回调中调用）会抛出 `Main process tasks must be added by their owner before startup.` 请在 `MainProcessCreated` 监听器中挂载所有内容。

**不要用 `#[Provider]` 或 `#[FactoryProvider]` 替换配置。** 本包自己的工厂会校验 PID 路径和套接字缓冲区大小；为 `SwooleConfigInterface` 提供另一个提供者会跳过这些校验，并与本包的工厂冲突。请用 `#[Configuration(SwooleConfigInterface::class)]` 声明该类。

**不要把配置用于按请求或运行期间的修改。** 这些设置在子进程启动之前对整个进程应用一次，而且之后还会再次调用获取方法。请返回固定的值，要修改它们就重启应用。

**不要依赖主进程重启子进程。** 正常退出的子进程不会被替换，失败的子进程会停止整个运行时。需要重启时，请在应用之前放置 systemd 等服务管理器。
