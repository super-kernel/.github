<p align="right">
  <a href="pool.md">English</a> | <strong>中文文档</strong>
</p>

# SuperKernel Pool

包：[`super-kernel/pool`](https://github.com/super-kernel/pool) · [Packagist](https://packagist.org/packages/super-kernel/pool) · [文档索引](../README.zh-CN.md)

## 用途

`super-kernel/pool` 保存一组数量有界的任意类型连接，并把它们借给调用方。调用方把自己的客户端包装在连接类中并提供工厂；连接池决定何时创建、检查、重用、替换和关闭这些连接，并让它们的数量保持在一个上限之内，该上限可以由一个进程内的协程、派生的多个进程或多个 Swoole 线程共享。

连接池对协议一无所知。打开、认证、检查、重置和关闭连接都由调用方的连接类负责，每个 I/O 期限也是如此。本包不依赖任何其他 SuperKernel 包，也不注册任何服务：由驱动（例如 `super-kernel/redis`）或应用组装每个连接池，并把它提供给使用方。

## 能力

### 编写连接及其工厂

连接实现 `SuperKernel\Pool\Contract\ConnectionInterface`：

- `connect()` 打开客户端。连接池为每个新连接调用它一次，在第一个借用方收到该连接之前。
- `isHealthy()` 报告连接是否可以再次使用。结果为 `false` 时，连接池会关闭该连接，并在需要时通过工厂创建替代连接；抛出的异常会向外传播。
- `reconnect()` 供当前借用方使用，借用方可以显式调用它。连接池自己从不重新连接已借出的连接，也从不重放请求。
- `close()` 释放客户端。它必须能够在失败之后再次被调用，因为连接池会重试失败的关闭。

连接类负责认证、清理半打开的客户端、在连接归还之前重置事务和会话，以及它自己的期限。工厂实现 `SuperKernel\Pool\Contract\ConnectionFactoryInterface`，其 `create()` 每次返回一个新的、尚未打开的连接；返回连接池已经持有的连接会抛出 `The factory returned a connection already owned by this pool.`

本页的示例使用下面的连接和工厂。只统计已打开连接数量的连接，保存为 `src/Pool/CounterConnection.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Pool;

use SuperKernel\Pool\Contract\ConnectionInterface;

final class CounterConnection implements ConnectionInterface
{
    private static int $opened = 0;

    public ?int $id = null;

    public bool $broken = false;

    public function connect(): void
    {
        $this->id = ++self::$opened;
    }

    public function isHealthy(): bool
    {
        return !$this->broken;
    }

    public function reconnect(): void
    {
        $this->close();
        $this->connect();
    }

    public function close(): void
    {
        $this->id = null;
    }
}
```

它的工厂，保存为 `src/Pool/CounterConnectionFactory.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Pool;

use SuperKernel\Pool\Contract\ConnectionFactoryInterface;

/**
 * @implements ConnectionFactoryInterface<CounterConnection>
 */
final class CounterConnectionFactory implements ConnectionFactoryInterface
{
    public int $created = 0;

    public function create(): CounterConnection
    {
        ++$this->created;

        return new CounterConnection();
    }
}
```

### 组装连接池

`SuperKernel\Pool\Config\PoolOption` 是保存连接池设置的 final readonly 类；用 `new Pool($option)` 创建连接池：

| 参数 | 默认值 | 含义 |
| --- | --- | --- |
| `factory` | 必需 | 创建连接的 `ConnectionFactoryInterface`。 |
| `capacity` | 必需 | 限制连接的容量：`CoroutineCapacity`、`ProcessCapacity`、`ThreadCapacity` 或自定义容量。 |
| `maxConnections` | 必需 | 固定的上限；它必须等于容量的上限。 |
| `waitTimeout` | `1.0` | `acquire()` 等待容量的秒数；`0` 表示不等待。 |
| `healthChecks` | `HealthCheckMode::OnBorrowAndReturn` | 何时检查连接。 |
| `frequency` | `new ConstantFrequency()` | 维护所遵循的需求策略。 |

选项保存固定的值和引用；工厂、容量和频率策略保留它们自己的状态。连接池在任何 I/O 之前检查上限为正数且与容量一致，以及超时是有限且非负的值。创建选项或连接池不会启动任何协程、定时器、工作进程或连接。请直接使用 `PoolOption`：没有需要实现的选项接口。它不是 `super-kernel/config` 的配置源；由驱动或应用的提供者根据自己的设置构建它，并把得到的连接池作为类型化服务提供出去。

### 借用和归还连接

`acquire(?float $timeout = null)` 返回一个连接。传入 `null` 时，它最多等待选项的超时时间；传入 `0` 时不等待。在时限内没有空出容量时，它会抛出 `SuperKernel\Pool\Exception\PoolTimeoutException`。只有等待容量的过程会计时；创建和打开连接使用连接自己的期限。每次调用都会作为一次借用尝试计入需求策略，即使之后超时也是如此。

每个连接都要恰好归还一次给借出它的连接池：可以重用时用 `release($connection)`，连接已损坏或无法重置时用 `discard($connection)`。第二次归还，或者来自另一个连接池的连接，会抛出 `The connection is not currently borrowed from this pool.` 连接池在检查或关闭连接之前就把它移出借出集合，因此即使第一次归还的清理仍在进行，第二次归还也会被拒绝；在清理结束之前，容量一直保持预留。关闭失败时，连接池会保留预留的容量，并在下一次维护或 `close()` 时重试关闭；永远不要为了重试而再次归还同一个对象。

连接池无法区分借用方。归还连接之前，请停止使用它并恢复其会话状态：没有自动归还，也没有租约；一旦该对象再次被借出，前一个借用方持有的过时引用与新借用方的引用就无法区分。

当创建、打开或检查连接失败时，`acquire()` 会关闭它、释放容量并抛出原来的异常，即使关闭也失败了也是如此。

三个协程争用两个连接、随后进行丢弃、第二次归还以及关闭后借用的脚本，保存为 `bin/contention.php`：

```php
<?php

declare(strict_types=1);

use Src\Pool\CounterConnection;
use Src\Pool\CounterConnectionFactory;
use SuperKernel\Pool\Capacity\CoroutineCapacity;
use SuperKernel\Pool\Config\PoolOption;
use SuperKernel\Pool\Exception\PoolTimeoutException;
use SuperKernel\Pool\Pool;
use Swoole\Coroutine;
use Swoole\Coroutine\WaitGroup;

use function Swoole\Coroutine\run;

require dirname(__DIR__) . '/vendor/autoload.php';

run(static function (): void {
    $pool = new Pool(new PoolOption(
        factory: new CounterConnectionFactory(),
        capacity: new CoroutineCapacity(2),
        maxConnections: 2,
        waitTimeout: 0.1,
    ));

    $group = new WaitGroup();

    foreach (['A', 'B', 'C'] as $borrower) {
        $group->add();

        Coroutine::create(static function () use ($pool, $group, $borrower): void {
            try {
                $connection = $pool->acquire();
                assert($connection instanceof CounterConnection);
                echo "{$borrower} holds connection {$connection->id}", PHP_EOL;
                Coroutine::sleep(0.2);
                $pool->release($connection);
            } catch (PoolTimeoutException $exception) {
                echo "{$borrower}: {$exception->getMessage()}", PHP_EOL;
            } finally {
                $group->done();
            }
        });
    }

    $group->wait();

    $connection = $pool->acquire();
    assert($connection instanceof CounterConnection);
    $connection->broken = true;
    echo "Discarding connection {$connection->id}", PHP_EOL;
    $pool->discard($connection);

    $connection = $pool->acquire();
    assert($connection instanceof CounterConnection);
    echo "After the discard, the next borrower gets connection {$connection->id}", PHP_EOL;
    $pool->release($connection);

    try {
        $pool->release($connection);
    } catch (Throwable $exception) {
        echo get_class($exception), ': ', $exception->getMessage(), PHP_EOL;
    }

    $pool->close();

    try {
        $pool->acquire();
    } catch (Throwable $exception) {
        echo get_class($exception), ': ', $exception->getMessage(), PHP_EOL;
    }
});
```

构建项目并运行脚本：

```bash
skernel build
php bin/contention.php
```

它会打印：

```text
A holds connection 1
B holds connection 2
C: Timed out waiting for a connection.
Discarding connection 2
After the discard, the next borrower gets connection 1
SuperKernel\Pool\Exception\PoolException: The connection is not currently borrowed from this pool.
SuperKernel\Pool\Exception\PoolClosedException: The connection pool is closed.
```

### 按上限统计连接

`maxConnections()` 是固定的上限，而不是空闲连接的数量。每个连接在空闲、借出、正在创建、正在检查或正在关闭时都计入上限，包括关闭失败、等待重试的连接。设 `M` 为上限、`B` 为已借出的连接数，则静止时的剩余容量为 `P = M - B`。就绪的空闲连接数 `I` 可能小于 `P`：缺少的连接由维护创建。

| 完成的转换 | 剩余容量 `P` | 就绪的空闲连接 `I` |
| --- | --- | --- |
| 借出一个健康的空闲连接 | -1 | -1 |
| 归还一个可重用的连接 | +1 | +1 |
| 归还一个随后被关闭的连接 | +1 | 不变 |
| 关闭一个不健康的空闲连接 | 不变 | -1 |
| 在维护期间创建一个连接 | 不变 | +1 |

`CapacityInterface::available()` 报告仍可预留的槽位，而不是 `I`；进行中的工作可能会暂时占用槽位。只有成功的创建才会产生就绪的连接。已借出的连接保持登记，以便检查归还，但维护从不触碰它们。

### 在协程、进程或线程之间共享上限

| 容量 | 共享者 | 空闲连接 | 组装方式 |
| --- | --- | --- | --- |
| `CoroutineCapacity` | 一个进程或线程内的协程 | 保留并重用 | 每个连接池一个容量；每个操作都在协程中运行。 |
| `ProcessCapacity` | 派生的进程 | 归还的连接由其所有者关闭 | 在派生之前创建容量，在派生之后创建每个连接池和工厂。 |
| `ThreadCapacity` | Swoole 线程 | 归还的连接由其所有者关闭 | 创建一次共享状态，并在每个线程中附加它。 |

`new CoroutineCapacity($limit, $channel)` 默认使用 `SuperKernel\Pool\Channel\SwooleChannel`；也可以传入另一个只由该容量使用的空 `ChannelInterface<bool>`。通道是延迟创建的，因此创建或关闭未使用的协程连接池不需要协程。

`ProcessCapacity` 把预留保存在 Swoole 共享内存中，`ThreadCapacity` 保存在原生线程映射中。只有预留是共享的：连接对象和套接字留在创建它们的连接池中，因此归还的连接会被关闭，而不是保留为空闲。等待方以单调时钟的截止时间轮询，每次最多暂停 1 ms，在协程中让出，否则休眠；没有先到先得的顺序，停滞的原生互斥锁可能使等待方超过其超时时间。把同一个容量传给多个连接池会共享一个上限和一次关闭；不同的容量保持各自的上限，因此多个独立的 `CoroutineCapacity` 对象不会限制整个应用。

对于进程，请在 `Swoole\Process::start()` 之前创建 `new ProcessCapacity($limit)`，并在每个进程中用它创建新的连接池。三个进程共享上限为二的脚本，保存为 `bin/processes.php`：

```php
<?php

declare(strict_types=1);

use Src\Pool\CounterConnectionFactory;
use SuperKernel\Pool\Capacity\ProcessCapacity;
use SuperKernel\Pool\Config\PoolOption;
use SuperKernel\Pool\Exception\PoolTimeoutException;
use SuperKernel\Pool\Pool;
use Swoole\Process;

require dirname(__DIR__) . '/vendor/autoload.php';

$capacity = new ProcessCapacity(2);

foreach ([1, 2, 3] as $number) {
    $process = new Process(static function () use ($capacity, $number): void {
        $pool = new Pool(new PoolOption(
            factory: new CounterConnectionFactory(),
            capacity: $capacity,
            maxConnections: 2,
            waitTimeout: 0.1,
        ));

        try {
            $connection = $pool->acquire();
            echo "Process {$number} holds a connection", PHP_EOL;
            usleep(300_000);
            $pool->release($connection);
        } catch (PoolTimeoutException $exception) {
            echo "Process {$number}: {$exception->getMessage()}", PHP_EOL;
        }
    });

    $process->start();
}

while (Process::wait()) {
}

echo 'Connections still reserved: ', $capacity->capacity() - $capacity->available(), PHP_EOL;
```

构建项目并运行脚本：

```bash
skernel build
php bin/processes.php
```

两个进程打印 `Process <n> holds a connection`，一个进程打印 `Process <n>: No connection capacity became available before the timeout.`，顺序可能不同；最后一行是 `Connections still reserved: 0`。

对于线程，创建方线程调用 `ThreadCapacity::create($limit)`，并把 `$capacity->state()` 和 `$capacity->mutex()` 作为 `Swoole\Thread` 参数传递。每个线程附加 `new ThreadCapacity($state, $mutex)`，并创建自己的工厂和连接池；只有这两个原生句柄会跨越线程边界。

预留不会过期。当持有容量的进程或线程终止时，等待并不能让该容量可以安全地重用：监管者必须在确认旧的所有者都已停止之后，废弃该容量并创建新的容量。本包不检测已终止的所有者，不重放请求，也不接管预留。

### 检查连接健康

`SuperKernel\Pool\Constants\HealthCheckMode` 决定何时用 `isHealthy()` 检查保留的连接：

| 模式 | 重用空闲连接之前 | 保留归还的连接之前 |
| --- | --- | --- |
| `OnBorrow` | 是 | 否 |
| `OnReturn` | 否 | 是 |
| `OnBorrowAndReturn`（默认） | 是 | 是 |
| `Manual` | 否 | 否 |

共享容量会关闭每个归还的连接，因此从不为了保留连接而检查它。借用检查发现不健康的空闲连接时，连接池会关闭它，并在创建一个替代连接之前尝试其他空闲连接；创建不会无限重试。每种模式都允许显式维护。

### 维护连接池

`maintain(int $minimumIdle = 0)` 运行一次维护：重试失败的关闭，检查空闲连接，并按需求策略的决定创建或移除空闲连接。它从不触碰已借出的连接，在另一次维护运行时跳过本次，并把正在创建或关闭的连接计入上限。`minimumIdle` 是尽力而为的下限，不能超过上限，对于不保留空闲连接的共享容量必须为 `0`。

选项的 `frequency` 提供需求策略，保留空闲连接的连接池在每次维护开始时读取它一次：

- 需求正常时，维护会填满借用方和进行中的工作留下的所有空闲容量，即使 `minimumIdle` 为 `0` 也是如此。
- 需求低时，它会关闭超出 `minimumIdle` 的空闲连接（为 `0` 时关闭全部），并创建连接直到该下限。

两种目标都不会挤占已借出的连接或进行中的工作。归还会释放槽位而不创建替代连接；由下一次维护创建。关闭失败时，其槽位保持预留，本次维护仍会尝试其余的连接，并抛出第一个错误；之后的每次维护会对这样的关闭重试一次，且绝不在它的另一次关闭进行时重试。创建失败会留下一个缺口，由下一次维护填补。已关闭的连接池和共享容量会跳过需求策略，归还连接也从不查询它。

`SuperKernel\Pool\Frequency\ConstantFrequency($frequency = 1.0, $lowFrequencyThreshold = 1.0)` 报告固定的每秒借用尝试次数；只有该次数低于阈值时（等于阈值不算）需求才算低。因此默认值总是填满连接池，而 `new ConstantFrequency(0.25, 1.0)` 总是收缩它。

`SuperKernel\Pool\Frequency\SmartLowFrequency($lowFrequencyThreshold = 1.0, $windowSeconds = 60.0, $clock = null)` 测量需求。它在调用等待容量之前统计每个有效的 `acquire()` 调用，包括之后超时或连接失败的调用；超时参数无效的调用、对已关闭连接池的调用以及来自其他执行上下文的调用都不计入。每次计入的尝试会在估计的每秒尝试次数上加 `1 / windowSeconds`，两次尝试之间，估计值按单调时钟以 `exp(-elapsedSeconds / windowSeconds)` 衰减，读取时也会衰减，因此没有新的借用时估计值会下降。在它的第一个窗口内，它从不报告低需求。该估计是平滑的，而不是滑动窗口内的精确计数，并在 `PHP_FLOAT_MAX` 处饱和。可选的 `$clock` 是一个返回有限单调秒数且不让出的 `Closure(): float`，用于确定性的测试；默认使用 `hrtime()`。失败、返回非有限值或倒退的时钟会抛出错误，并保持估计值不变。

策略不会启动定时器，也不执行 I/O。请给每个连接池各自的 `SmartLowFrequency`，它保存状态；`ConstantFrequency` 可以共享。自定义策略实现 `SuperKernel\Pool\Contract\FrequencyInterface`，并且不能让出或执行 I/O。

可以手动运行维护，也可以从应用事件或任务、任何调度器，或者用 `SuperKernel\Pool\Maintenance\SwooleTimerScheduler` 运行。该调度器接收连接池、以毫秒为单位的间隔、错误回调和 `minimumIdle`；`start()` 启动一个 Swoole 定时器，它在协程中运行每次维护，跳过重叠的维护，并把失败交给回调，回调不能抛出异常。`stop()` 取消之后的维护，并让正在运行的维护完成；已关闭的连接池会在下一次触发时停止定时器，已关闭的连接池也无法被调度。构造函数不会调度任何内容。

由定时器填满一个三连接连接池的脚本，保存为 `bin/maintenance.php`：

```php
<?php

declare(strict_types=1);

use Src\Pool\CounterConnectionFactory;
use SuperKernel\Pool\Capacity\CoroutineCapacity;
use SuperKernel\Pool\Config\PoolOption;
use SuperKernel\Pool\Frequency\SmartLowFrequency;
use SuperKernel\Pool\Maintenance\SwooleTimerScheduler;
use SuperKernel\Pool\Pool;
use Swoole\Coroutine;

use function Swoole\Coroutine\run;

require dirname(__DIR__) . '/vendor/autoload.php';

run(static function (): void {
    $factory = new CounterConnectionFactory();

    $pool = new Pool(new PoolOption(
        factory: $factory,
        capacity: new CoroutineCapacity(3),
        maxConnections: 3,
        frequency: new SmartLowFrequency(lowFrequencyThreshold: 1.0, windowSeconds: 60.0),
    ));

    $scheduler = new SwooleTimerScheduler(
        $pool,
        intervalMilliseconds: 100,
        onError: static function (Throwable $error): void {
            echo 'Maintenance failed: ', $error->getMessage(), PHP_EOL;
        },
        minimumIdle: 1,
    );

    echo "Before maintenance: {$factory->created} connections", PHP_EOL;

    $scheduler->start();
    Coroutine::sleep(0.25);
    $scheduler->stop();

    echo "After maintenance: {$factory->created} connections", PHP_EOL;

    $pool->close();
});
```

构建项目并运行脚本：

```bash
skernel build
php bin/maintenance.php
```

它会打印：

```text
Before maintenance: 0 connections
After maintenance: 3 connections
```

该策略仍处于它的第一个窗口内，因此需求不算低，维护会填满空闲容量。

### 关闭

`close()` 永久关闭容量（对共享它的每个连接池也是如此），并以 `PoolClosedException` 唤醒其等待方。它尝试关闭每个空闲连接，然后抛出第一个错误。已借出的连接在归还时关闭，因此每个借用方仍然必须归还它们；在关闭期间正在创建或检查的连接永远不会被借出。再次调用 `close()` 会重试失败的关闭，并保持已关闭的连接不变。析构函数不会关闭任何内容：请在协程运行时内部关闭协程连接池，并在所有者关闭时停止它的调度器。

### 使用通道

`SuperKernel\Pool\Channel\SwooleChannel($capacity)` 是供协程使用的有界先进先出通道，原样保存每个值，包括 `null` 和 `false`。对已满通道的 `push()` 会立即失败，`pop($timeout)` 传入 `0` 时从不等待，`close()` 会唤醒等待的读取方。原生通道在首次使用时创建，每次推入或弹出都需要协程。`CoroutineCapacity` 使用它；其他代码可以通过 `SuperKernel\Pool\Contract\ChannelInterface` 使用它。

### 编写自定义容量或连接池

自定义容量实现 `SuperKernel\Pool\Contract\CapacityInterface`；其核心方法是返回预留令牌的 `reserve(float $timeout, string $owner): string` 和 `release(string $reservation, string $owner)`。自定义连接池实现 `SuperKernel\Pool\Contract\PoolInterface`，包括 `SwooleTimerScheduler` 调用的 `maintain()`。

## 错误

| 情形 | 消息 |
| --- | --- |
| 上限不是正数或与容量不一致，或者超时无效 | `Positive matching capacity and a finite timeout are required.`（`InvalidPoolConfigurationException`） |
| 容量上限不是正数 | `Backend capacity must be positive.`（`InvalidArgumentException`） |
| `acquire()` 的超时为负数或非有限值 | `The connection wait timeout must be finite and non-negative.`（`InvalidArgumentException`） |
| 协程容量在时限内没有容量 | `Timed out waiting for a connection.`（`PoolTimeoutException`） |
| 共享容量在时限内没有容量 | `No connection capacity became available before the timeout.`（`PoolTimeoutException`） |
| 等待中的协程被取消 | `Connection acquisition was canceled.`（`PoolException`） |
| `close()` 之后借用 | `The connection pool is closed.`（`PoolClosedException`） |
| 在协程之外使用协程连接池 | `Connection pool operations require an active Swoole coroutine.`（`PoolException`） |
| 从另一个进程或线程使用连接池 | `Create each resource-owning pool in its own process or thread.`（`PoolException`） |
| 一个 `CoroutineCapacity` 被两个连接池使用 | `A coroutine backend belongs to one resource-owning pool.`（`PoolException`） |
| 第二次归还或外来的连接 | `The connection is not currently borrowed from this pool.`（`PoolException`） |
| 工厂返回了连接池持有的连接 | `The factory returned a connection already owned by this pool.`（`PoolException`） |
| `minimumIdle` 不在 `0` 到上限之间 | `The idle minimum must be between zero and capacity.`（`InvalidArgumentException`） |
| 共享容量使用正数的 `minimumIdle` | `Shared backends retire owner-local resources and cannot retain idle connections.`（`PoolException`） |
| 不支持线程时使用 `ThreadCapacity` | `The thread backend requires PHP ZTS and Swoole thread support.`（`PoolException`） |
| 无效的调度器间隔或下限 | `A positive interval and a non-negative idle minimum are required.`（`InvalidArgumentException`） |
| 调度已关闭的连接池 | `Cannot schedule maintenance on a closed pool.`（`PoolException`） |
| 无效的 `ConstantFrequency` 或 `SmartLowFrequency` 参数 | `A finite non-negative frequency and a positive threshold are required.` 或 `A finite positive frequency threshold and observation window are required.`（`InvalidArgumentException`） |
| 向已满的通道推入 | `The connection channel is full.`（`PoolException`） |
| 弹出超时 | `No connection channel value is available.`（`PoolTimeoutException`） |

`PoolException`、`PoolTimeoutException`、`PoolClosedException` 和 `InvalidPoolConfigurationException` 位于 `SuperKernel\Pool\Exception`；后三者继承 `PoolException`。

## 与其他包的关系

- [`super-kernel/redis`](redis.zh-CN.md) 用本包为每个 Redis 连接名称组装一个连接池。
- 其他包和应用以同样的方式为自己的客户端构建连接池；本包不依赖它们中的任何一个。

## 不要这样做

**不要在创建连接池的进程或线程之外使用它。** 连接池及其连接属于创建者；通过派生继承的、或传给线程的连接池会抛出 `Create each resource-owning pool in its own process or thread.` 请在派生之前创建共享容量，并在每个进程或线程中创建新的连接池。

**不要把连接归还两次或归还给另一个连接池。** 连接池会抛出 `The connection is not currently borrowed from this pool.`，而且一旦该对象再次被借出，迟到的第二次归还会把它从新的借用方手中拿走。请在 `finally` 块中把每个连接归还一次。

**不要在归还连接之后继续使用它。** 连接池无法撤销引用，下一个借用方会收到同一个对象。请在 `release()` 之前停止使用它并重置其会话状态。

**不要用 `release()` 归还已损坏或状态未清理的连接。** 带有未结束事务或已修改会话的连接会原样到达下一个借用方。对于无法重置的连接，请使用 `discard()`。

**不要在多个连接池之间共享一个 `CoroutineCapacity`。** 它属于一个连接池，对另一个连接池会抛出 `A coroutine backend belongs to one resource-owning pool.` 需要跨所有者的上限时，请使用 `ProcessCapacity` 或 `ThreadCapacity`。

**不要为共享容量设置正数的 `minimumIdle`。** 共享容量会关闭归还的连接，不保留空闲连接，因此维护会抛出 `Shared backends retire owner-local resources and cannot retain idle connections.` 请使用 `0`。

**不要在调度器的错误回调中抛出异常。** 该回调在定时器上运行；请报告错误并返回。

**不要在多个连接池之间共享 `SmartLowFrequency`。** 它测量它记录的借用，共享会混合多个连接池的需求。请给每个连接池各自的实例。

**不要在持有容量的进程终止后重用该容量。** 它的预留永远不会仅因时间而释放。请停止其余的所有者，然后创建新的容量和新的连接池。
