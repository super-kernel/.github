<p align="right">
  <a href="redis.md">English</a> | <strong>中文文档</strong>
</p>

# SuperKernel Redis

包：[`super-kernel/redis`](https://github.com/super-kernel/redis) · [Packagist](https://packagist.org/packages/super-kernel/redis) · [文档索引](../README.zh-CN.md)

## 用途

`super-kernel/redis` 通过 phpredis 把 SuperKernel 应用连接到 Redis。应用声明一个或多个命名连接；本包校验它们，为每个连接保留一个 `super-kernel/pool` 连接池，并为每个协程针对每个连接名称和数据库提供各自的原生 `\Redis` 连接，在协程结束时把它归还给连接池。

命令、事务、序列化以及回复的含义都由应用负责：本包原样返回 phpredis 对象，不添加命令层。每个配置指向一台 Redis 服务器；不提供 Cluster、Sentinel、TLS、持久连接、故障转移和缓存适配器。

## 能力

### 声明连接

`SuperKernel\Redis\Config\RedisConfig` 是一个 final readonly 类，其公开属性保存一个连接的设置：

| 属性 | 默认值 | 含义 |
| --- | --- | --- |
| `poolOption` | 必需 | 该连接的连接池所使用的 `super-kernel/pool` 的 `PoolOption`，其工厂为 `RedisConnectionFactory`。 |
| `host` | `'127.0.0.1'` | 服务器的主机名或地址。 |
| `port` | `6379` | TCP 端口，范围为 `1` 到 `65535`。 |
| `database` | `0` | 每个新连接上选择的数据库。 |
| `username` | `null` | ACL 用户名；需要同时提供密码。 |
| `password` | `null` | 密码。 |
| `connectTimeout` | `1.0` | 允许连接的秒数，至少一微秒。 |
| `readTimeout` | `1.0` | 允许读取回复的秒数，至少一微秒。 |

在一个用 `#[Configuration(RedisConfig::class)]` 标记、并实现 `SuperKernel\Contract\MultipleConfigInterface` 的类中声明这些条目。每个用 `#[Configuration]` 标记的无参数公开方法返回一个条目，以方法名命名，区分大小写。只有一个条目时，它就是默认条目；有多个条目时，用 `#[Configuration(default: true)]` 标记默认条目。没有默认条目时，按名称访问仍然可用，只有不带名称的访问会失败。未知名称永远不会回退到另一个条目。`super-kernel/config` 找到该类，在第一次需要时创建每个条目并保留该对象；这些条目不是容器条目。

`SuperKernel\Redis\Config\RedisOptionLoader::get($name)` 在检查条目的设置之后返回该条目，并且每次都返回同一个对象；`null` 选择默认条目。每次请求都会进行检查，因为即使条目无效，配置也会保留它；检查不会打开任何连接。条目是只读的：要修改设置，需要创建新的条目，实际上就是重启应用。

`PoolOption` 会被原样使用：本包既不复制它，也不保留另一套连接池设置。请为每个条目提供各自的 `PoolOption` 和各自的 `RedisConnectionFactory`。

带有一个默认连接和一个使用数据库 `1` 的缓存连接的配置，保存为 `src/Config/RedisConfigurations.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Config;

use SuperKernel\Attribute\Configuration;
use SuperKernel\Contract\MultipleConfigInterface;
use SuperKernel\Pool\Capacity\CoroutineCapacity;
use SuperKernel\Pool\Config\PoolOption;
use SuperKernel\Pool\Constants\HealthCheckMode;
use SuperKernel\Pool\Frequency\SmartLowFrequency;
use SuperKernel\Redis\Config\RedisConfig;
use SuperKernel\Redis\Factory\RedisConnectionFactory;

#[Configuration(RedisConfig::class)]
final class RedisConfigurations implements MultipleConfigInterface
{
    #[Configuration(default: true)]
    public function primary(): RedisConfig
    {
        return new RedisConfig(
            poolOption: new PoolOption(
                factory: new RedisConnectionFactory(),
                capacity: new CoroutineCapacity(8),
                maxConnections: 8,
                waitTimeout: 0.25,
                healthChecks: HealthCheckMode::OnBorrowAndReturn,
                frequency: new SmartLowFrequency(),
            ),
            host: '127.0.0.1',
            port: 6379,
            database: 0,
            connectTimeout: 1.0,
            readTimeout: 1.0,
        );
    }

    #[Configuration]
    public function cache(): RedisConfig
    {
        return new RedisConfig(
            poolOption: new PoolOption(
                factory: new RedisConnectionFactory(),
                capacity: new CoroutineCapacity(4),
                maxConnections: 4,
            ),
            database: 1,
        );
    }
}
```

### 获取原生连接

`SuperKernel\Redis\Factory\RedisFactory` 只有一个方法 `get(?string $name = null, ?int $database = null): \Redis`。`get()` 和 `get(null)` 使用默认条目；名称会选择该条目的服务器、凭据、超时、数据库和连接池。`$database` 只替换连接上选择的数据库，不会改变条目。请通过构造函数注入接收工厂，并在使用连接的地方调用 `get()`。

结果是真正的 `\Redis` 对象，带有 phpredis 自己的签名、返回值和错误，包括 `scan()`、`hScan()`、`sScan()` 和 `zScan()` 的引用传递游标。新连接不是持久连接，禁用了自动重试（`OPT_MAX_RETRIES` 为 `0`）并关闭了序列化器，在调用方收到之前已完成认证并切换到其数据库；失败时连接会被丢弃。本包不会修改 Swoole 的运行时钩子：请在应用中启用套接字钩子，使 Redis I/O 让出给其他协程。

在缓存连接上统计页面访问次数的服务，保存为 `src/Redis/VisitCounter.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Redis;

use SuperKernel\Attribute\Provider;
use SuperKernel\Redis\Factory\RedisFactory;

#[Provider(VisitCounter::class)]
final readonly class VisitCounter
{
    public function __construct(private RedisFactory $redisFactory)
    {
    }

    public function record(string $page): int
    {
        return $this->redisFactory->get('cache')->incr("visits:{$page}");
    }
}
```

记录两次访问的脚本，保存为 `bin/visits.php`：

```php
<?php

declare(strict_types=1);

use Src\Redis\VisitCounter;
use SuperKernel\Context\ApplicationContext;
use SuperKernel\Redis\Pool\RedisPool;

use function Swoole\Coroutine\run;

require dirname(__DIR__) . '/vendor/autoload.php';

run(static function (): void {
    $container = ApplicationContext::getContainer();
    $counter = $container->get(VisitCounter::class);

    $counter->record('home');

    echo 'Visits: ', $counter->record('home'), PHP_EOL;

    $container->get(RedisPool::class)->close();
});
```

构建项目，并在空数据库上运行脚本：

```bash
skernel build
php bin/visits.php
```

它会打印 `Visits: 2`。

### 按协程划分的会话

每次借用都发生在协程中。在同一个协程中，同一条目和数据库总是返回同一个连接，同一条目的名称和默认别名共享该连接；另一个协程会得到自己的连接。连接在其协程结束之前一直占用连接池的槽位（即使在最后一条命令之后），然后被丢弃而不是重用，因此在它上面设置的事务、管道、所选数据库和选项永远不会到达另一个借用方。因此，长时间运行的协程会在其整个生命期内持有它的连接：请按同时持有的所有连接来确定连接池的大小，永远不要在协程结束之后保留连接，也不要把它交给另一个协程。

同一条目的数据库覆盖共享该条目的连接池及其上限；不同的条目有各自的连接池。如果两个条目方法返回同一个 `RedisConfig` 对象，它们共享一个连接池。把一个 `PoolOption` 交给两个不同的条目会被拒绝，消息为 `A pool option cannot be reused with another Redis configuration.`

展示哪些调用共享连接、并用游标遍历键空间的脚本，保存为 `bin/sessions.php`：

```php
<?php

declare(strict_types=1);

use SuperKernel\Context\ApplicationContext;
use SuperKernel\Redis\Factory\RedisFactory;
use SuperKernel\Redis\Pool\RedisPool;
use Swoole\Coroutine;

use function Swoole\Coroutine\run;

require dirname(__DIR__) . '/vendor/autoload.php';

run(static function (): void {
    $container = ApplicationContext::getContainer();
    $factory = $container->get(RedisFactory::class);

    $default = $factory->get();
    $cache = $factory->get('cache');
    $queue = $factory->get('cache', database: 2);

    echo 'Default and primary share a session: ', var_export($default === $factory->get('primary'), true), PHP_EOL;
    echo 'Databases: ', $default->getDbNum(), ' ', $cache->getDbNum(), ' ', $queue->getDbNum(), PHP_EOL;

    $cache->mSet(['cache:a' => '1', 'cache:b' => '2', 'cache:c' => '3']);

    $keys = [];
    $iterator = null;

    do {
        foreach ($cache->scan($iterator, 'cache:*') ?: [] as $key) {
            $keys[] = $key;
        }
    } while (0 !== $iterator);

    sort($keys);
    echo 'Scanned: ', implode(', ', $keys), PHP_EOL;

    Coroutine::join([
        Coroutine::create(static function () use ($factory, $default): void {
            echo 'Another coroutine gets its own session: ', var_export($factory->get() !== $default, true), PHP_EOL;
        }),
    ]);

    $container->get(RedisPool::class)->close();
});
```

运行 `skernel build` 之后运行它：

```bash
php bin/sessions.php
```

它会打印：

```text
Default and primary share a session: true
Databases: 0 1 2
Scanned: cache:a, cache:b, cache:c
Another coroutine gets its own session: true
```

原生连接上失败的命令是普通的 phpredis 失败；该连接会一直使用到协程结束。要立即废弃它，请把它传给 `SuperKernel\Redis\Pool\RedisPool::discard($redis)`；属于另一个协程的连接会被拒绝，消息为 `The Redis connection does not belong to the current coroutine.`

### 通过代理调用 Redis

`SuperKernel\Redis\Proxy\Redis` 把每个静态调用（例如 `Redis::get('key')`）以及对注入的 `Redis` 对象的每个实例调用，都转发给当前协程中默认条目的连接，使用该条目配置的数据库；两种形式共享该连接。代理不是 `\Redis` 的子类；它的 `@mixin \Redis` 注释让 IDE 能够提示原生方法。位置参数和命名参数、返回值、`false` 回复和最后的错误都会保留，`multi()` 等流式方法返回原生连接。需要命名条目或其他数据库时，请使用 `RedisFactory::get()`。

当容器创建 `RedisFactory` 时，代理会绑定到默认的工厂和连接池，但不会打开连接；在此之前，每次调用都会抛出 `The Redis proxy requires initialization through RedisFactoryProvider.` 创建或注入代理对象并不会绑定它。绑定按进程或线程划分；当容器创建另一个 `RedisFactory` 时，之后的代理调用会使用它，已经借出的连接仍属于原来的连接池。

通过代理进行的调用抛出异常时，会废弃该连接并重新抛出原来的异常，而不会重复执行命令；之后在该协程中再使用同一条目和数据库会抛出 `The native Redis session has already been retired.` 直接在流式方法返回的原生对象上进行的调用是普通的 phpredis 调用，代理无法看到它们。PHP 无法通过魔术调用传递引用游标，因此 `scan` 系列只能在工厂的原生结果上使用。通过代理调用的 `close()` 只关闭那个原生连接，而不是连接池。

以静态方式和通过注入的实例使用代理、然后关闭连接池的脚本，保存为 `bin/proxy.php`：

```php
<?php

declare(strict_types=1);

use SuperKernel\Context\ApplicationContext;
use SuperKernel\Redis\Factory\RedisFactory;
use SuperKernel\Redis\Pool\RedisPool;
use SuperKernel\Redis\Proxy\Redis;

use function Swoole\Coroutine\run;

require dirname(__DIR__) . '/vendor/autoload.php';

run(static function (): void {
    $container = ApplicationContext::getContainer();
    $container->get(RedisFactory::class);

    Redis::set('counter', '10');
    echo Redis::incrBy('counter', 5), PHP_EOL;

    $result = Redis::multi()->set('first', 'one')->set('second', 'two')->exec();
    echo json_encode($result), PHP_EOL;

    $proxy = $container->get(Redis::class);
    echo $proxy->get('first'), PHP_EOL;

    $container->get(RedisPool::class)->close();

    try {
        Redis::get('first');
    } catch (Throwable $exception) {
        echo get_class($exception), ': ', $exception->getMessage(), PHP_EOL;
    }
});
```

运行 `skernel build` 之后运行它：

```bash
php bin/proxy.php
```

它会打印：

```text
15
[true,true]
one
LogicException: The Redis pool is closed.
```

### 自行组装连接池

`RedisPool::get(RedisConfig $option, ?int $database = null)` 在检查直接给出的条目之后从该条目的连接池借用；`RedisFactory` 调用的就是它。在创建连接池之前，它把条目绑定到条目的 `RedisConnectionFactory`，不会打开连接，也不会替换工厂。一个工厂只能绑定一个条目，再次绑定到同一个条目是允许的。自行从 `RedisConfig` 构建 `super-kernel/pool` 的 `Pool` 的代码，请先调用 `$factory->bind($option)`，或者创建 `new RedisConnectionFactory($option)`；没有条目的工厂无法创建连接。不会启动任何定时器：如有需要，请自行运行连接池的维护。

### 关闭

注入 `RedisPool`，并在其所属协程中、在协程运行时仍在运行时调用 `close()`，例如在应用关闭期间。它会永久关闭每个连接池：之后的借用和代理调用会抛出 `The Redis pool is closed.` 仍被借出的连接可以在其协程中完成工作，并在协程结束时关闭。`RedisFactory` 没有关闭方法。

协程结束时关闭连接发生在延迟回调中，该回调不能抛出异常。其中的失败由 `RedisPool` 保留，并由下一次借用或 `close()` 以 `SuperKernel\Redis\Exception\RedisConnectionException` 抛出一次；这类消息中永远不会出现凭据。关闭失败的连接会保留它的连接池槽位。即使之后没有命令会暴露保留的失败，也请在关闭时调用 `close()`；再次调用会重试每个失败的关闭（包括第一次调用之后废弃的连接），并保持已关闭的连接不变。

### 进程与线程

`RedisPool` 属于容器创建它时所在的进程或线程，它的连接池以及每个条目的 `CoroutineCapacity` 也是如此。在会派生进程或启动线程的应用中，请只在派生之后、在每个线程中分别解析 Redis 服务并加载条目；从另一个进程或线程使用连接池会抛出 `A Redis pool cannot be shared between processes or threads.` 连接池、工厂、容量和套接字永远不会在进程或线程之间移动。

## 错误

| 情形 | 消息 |
| --- | --- |
| 主机为空或含有 NUL 字节 | `The Redis host must be non-empty and contain no NUL bytes.` |
| 端口不在 `1` 到 `65535` 之间 | `The Redis TCP port must be between 1 and 65535.` |
| 负数的数据库 | `The Redis database index must be non-negative.` |
| 没有密码的用户名，或者空用户名 | `A Redis ACL username requires a password and cannot be empty.` |
| 小于一微秒或非有限值的超时 | `Redis connectTimeout must be finite and at least one microsecond.` 或 `Redis readTimeout must be finite and at least one microsecond.` |
| 连接池上限小于一 | `The Redis pool must allow at least one connection.` |
| 无效的连接池等待超时 | `The Redis pool wait timeout must be finite and non-negative.` |
| 容量与连接池上限不一致 | `The Redis pool capacity must match its connection limit.` |
| 其他类型的配置条目 | `The Redis configuration item must be a RedisConfig.` |
| 未知名称 | `No Redis configuration item is registered for this name.`（`ConfigurationNotFoundException`） |
| 多个条目、没有默认条目、不带名称的访问 | `Configuration contract SuperKernel\Redis\Config\RedisConfig has multiple items but no default item is declared.`（`ConfigurationNotFoundException`） |
| 负数的数据库覆盖 | `The Redis database index must not be negative.`（`InvalidArgumentException`） |
| 一个 `PoolOption` 用于两个条目 | `A pool option cannot be reused with another Redis configuration.`（`InvalidArgumentException`） |
| 一个 `RedisConnectionFactory` 用于两个条目 | `A Redis connection factory belongs to one Redis option.`（`InvalidArgumentException`） |
| 在协程之外借用 | `Redis connection acquisition requires an active Swoole coroutine.`（`PoolException`） |
| 从另一个进程或线程使用连接池 | `A Redis pool cannot be shared between processes or threads.`（`PoolException`） |
| 从另一个协程调用 `discard()` | `The Redis connection does not belong to the current coroutine.`（`PoolException`） |
| 在 `RedisFactory` 存在之前使用代理 | `The Redis proxy requires initialization through RedisFactoryProvider.`（`LogicException`） |
| `close()` 之后借用或调用代理 | `The Redis pool is closed.`（`LogicException`） |
| 代理调用失败之后再次使用 | `The native Redis session has already been retired.`（`LogicException`） |
| 连接、认证或选择数据库失败 | `Redis connection initialization failed.`，原因例如 `Redis connection establishment failed.`、`Redis authentication failed.` 或 `Selecting the Redis database failed.`（`RedisConnectionException`） |
| 关闭连接失败 | `Redis connection cleanup failed.`（`RedisConnectionException`） |

没有标注类的消息属于 `SuperKernel\Redis\Exception\InvalidRedisConfigurationException`，在创建连接池之前抛出。`RedisConnectionException` 和 `InvalidRedisConfigurationException` 位于 `SuperKernel\Redis\Exception`，并实现其 `RedisExceptionInterface`。

## 与其他包的关系

- [`super-kernel/pool`](pool.zh-CN.md) 提供每个条目使用的连接池、`PoolOption`、容量、健康检查和维护策略。
- [`super-kernel/config`](config.zh-CN.md) 找到 `#[Configuration]` 配置源，为条目命名并保留它们，并选择默认条目。
- [`super-kernel/di`](di.zh-CN.md) 把 `RedisFactory`、`RedisPool`、`RedisOptionLoader` 和代理提供给你的服务。
- [`super-kernel/swoole`](swoole.zh-CN.md) 和 [`super-kernel/http-server`](http-server.zh-CN.md) 运行借用连接的协程。

## 不要这样做

**不要在借用连接的协程之外使用它。** 它会在该协程结束时被丢弃，而另一个协程使用它会在一个套接字上交错执行命令。请在每个需要 Redis 的协程中调用 `RedisFactory::get()`。

**不要让持有空闲连接的协程一直存活。** 连接在其协程结束之前一直占用连接池槽位，因此借用过一次的长时间运行循环会永久占住该槽位。请在短生命期的协程中借用，或者按长生命期协程持有的连接数确定连接池大小。

**不要在条目之间共享 `PoolOption` 或 `RedisConnectionFactory`。** 它们会被拒绝，消息分别为 `A pool option cannot be reused with another Redis configuration.` 和 `A Redis connection factory belongs to one Redis option.` 请在每个条目方法中创建它们。

**不要把代理用于 `scan` 系列或命名条目。** 魔术调用无法更新引用游标，而且代理总是使用默认条目。请使用 `RedisFactory::get()` 的原生结果。

**不要在容器创建 `RedisFactory` 之前调用代理。** 它会抛出 `The Redis proxy requires initialization through RedisFactoryProvider.` 请先解析 `RedisFactory` 或依赖它的服务。

**不要把代理的 `close()` 当作关闭。** 它只关闭当前的原生连接。请调用 `RedisPool::close()` 关闭连接池。

**不要在关闭时省略 `RedisPool::close()`。** 关闭连接的失败会被保留，只由下一次借用或 `close()` 报告，在关闭成功之前其连接池槽位一直被占用。请在运行时结束之前于所属协程中调用它。

**不要在进程或线程之间共享连接池。** 连接池属于创建它的进程或线程，在其他地方会抛出 `A Redis pool cannot be shared between processes or threads.` 请让每个进程或线程通过自己的容器借用。
