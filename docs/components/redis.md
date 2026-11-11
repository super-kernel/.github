<p align="right">
  <strong>English</strong> | <a href="redis.zh-CN.md">中文文档</a>
</p>

# SuperKernel Redis

Package: [`super-kernel/redis`](https://github.com/super-kernel/redis) · [Packagist](https://packagist.org/packages/super-kernel/redis) · [Documentation index](../README.md)

## Purpose

`super-kernel/redis` connects SuperKernel applications to Redis through phpredis. The application declares one or more named connections; the package validates them, keeps one pool of `super-kernel/pool` per connection, and gives each coroutine its own native `\Redis` connection for each connection name and database, which it returns to the pool when the coroutine ends.

Commands, transactions, serialization, and the meaning of replies stay with the application: the package returns phpredis objects as they are and adds no command layer. Each configuration targets one Redis server; Cluster, Sentinel, TLS, persistent connections, failover, and cache adapters are not provided.

## Capabilities

### Declare connections

`SuperKernel\Redis\Config\RedisConfig` is a final readonly class whose public properties hold one connection's settings:

| Property | Default | Meaning |
| --- | --- | --- |
| `poolOption` | required | The `PoolOption` of `super-kernel/pool` that this connection's pool uses, with a `RedisConnectionFactory` as its factory. |
| `host` | `'127.0.0.1'` | The server's host name or address. |
| `port` | `6379` | The TCP port, from `1` to `65535`. |
| `database` | `0` | The database selected on every new connection. |
| `username` | `null` | The ACL user name; it requires a password. |
| `password` | `null` | The password. |
| `connectTimeout` | `1.0` | Seconds allowed to connect, at least one microsecond. |
| `readTimeout` | `1.0` | Seconds allowed to read a reply, at least one microsecond. |

Declare the items in a class marked with `#[Configuration(RedisConfig::class)]` that implements `SuperKernel\Contract\MultipleConfigInterface`. Each public method without parameters that is marked with `#[Configuration]` returns one item, named by the method name, case-sensitively. With one item, it is the default; with several, `#[Configuration(default: true)]` marks the default. Without a default, named access still works and only unnamed access fails. An unknown name never falls back to another item. `super-kernel/config` finds the class, creates each item when it is first needed, and keeps that object; the items are not container entries.

`SuperKernel\Redis\Config\RedisOptionLoader::get($name)` returns an item after checking its settings, and returns the same object every time; `null` selects the default. The check runs on every request, because the configuration keeps an item even when it is invalid, and it opens no connection. The items are readonly: to change a setting, create a new item, which in practice means restarting the application.

The `PoolOption` is used as it is: the package neither copies it nor keeps a second set of pool settings. Give every item its own `PoolOption` and its own `RedisConnectionFactory`.

A configuration with a default connection and a cache connection on database `1`, saved as `src/Config/RedisConfigurations.php`:

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

### Get native connections

`SuperKernel\Redis\Factory\RedisFactory` has one method, `get(?string $name = null, ?int $database = null): \Redis`. `get()` and `get(null)` use the default item; a name selects that item's server, credentials, timeouts, database, and pool. `$database` replaces only the database selected on the connection, without changing the item. Receive the factory through constructor injection, and call `get()` where the connection is used.

The result is a real `\Redis` object with phpredis's own signatures, return values, and errors, including by-reference cursors for `scan()`, `hScan()`, `sScan()`, and `zScan()`. New connections are not persistent, have automatic retries disabled (`OPT_MAX_RETRIES` is `0`) and the serializer off, and are authenticated and switched to their database before the caller receives them; when that fails, the connection is discarded. The package does not change Swoole's runtime hooks: enable the socket hooks in the application so that Redis I/O yields to other coroutines.

A service that counts page visits on the cache connection, saved as `src/Redis/VisitCounter.php`:

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

A script that records two visits, saved as `bin/visits.php`:

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

Build the project and run the script on an empty database:

```bash
skernel build
php bin/visits.php
```

It prints `Visits: 2`.

### Sessions per coroutine

Every borrow happens inside a coroutine. Within one coroutine, the same item and database always return the same connection, and a name and the default alias of the same item share it; another coroutine receives its own. A connection keeps its pool slot until its coroutine ends, even after its last command, and is then discarded rather than reused, so transactions, pipelines, the selected database, and options set on it never reach another borrower. A long-running coroutine therefore holds its connections for its whole life: size the pool for all connections held at the same time, and never keep a connection after its coroutine ends or hand it to another coroutine.

Database overrides of one item share that item's pool and its limit; separate items have separate pools. If two item methods return the same `RedisConfig` object, they share one pool. One `PoolOption` given to two different items is rejected with `A pool option cannot be reused with another Redis configuration.`

A script that shows which calls share a connection and walks a key space with a cursor, saved as `bin/sessions.php`:

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

Run it after `skernel build`:

```bash
php bin/sessions.php
```

It prints:

```text
Default and primary share a session: true
Databases: 0 1 2
Scanned: cache:a, cache:b, cache:c
Another coroutine gets its own session: true
```

A failing command on a native connection is an ordinary phpredis failure; the connection stays in use until the coroutine ends. To retire it at once, pass it to `SuperKernel\Redis\Pool\RedisPool::discard($redis)`; a connection that belongs to another coroutine is rejected with `The Redis connection does not belong to the current coroutine.`

### Call Redis through the proxy

`SuperKernel\Redis\Proxy\Redis` forwards every static call, such as `Redis::get('key')`, and every instance call on an injected `Redis` object to the default item's connection of the current coroutine, on that item's configured database; both forms share that connection. The proxy is not a subclass of `\Redis`; its `@mixin \Redis` annotation lets IDEs offer the native methods. Positional and named arguments, return values, `false` replies, and the last error are kept, and fluent methods such as `multi()` return the native connection. For a named item or another database, use `RedisFactory::get()`.

The proxy is bound to the default factory and pool when the container creates `RedisFactory`, without opening a connection; before that, every call raises `The Redis proxy requires initialization through RedisFactoryProvider.` Creating or injecting a proxy object does not bind it. The binding is per process or thread; when the container creates another `RedisFactory`, later proxy calls use it, and connections already borrowed stay with their original pool.

A call through the proxy that throws retires the connection and rethrows the original exception without repeating the command; later use of the same item and database in that coroutine raises `The native Redis session has already been retired.` Calls made directly on a native object returned by a fluent method are ordinary phpredis calls that the proxy cannot see. PHP cannot pass a by-reference cursor through a magic call, so the `scan` family works only on the factory's native result. `close()` through the proxy closes only that native connection, not the pool.

A script that uses the proxy statically and through an injected instance, then closes the pools, saved as `bin/proxy.php`:

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

Run it after `skernel build`:

```bash
php bin/proxy.php
```

It prints:

```text
15
[true,true]
one
LogicException: The Redis pool is closed.
```

### Assemble a pool yourself

`RedisPool::get(RedisConfig $option, ?int $database = null)` borrows from the pool of an item given directly, after checking it; it is what `RedisFactory` calls. Before it creates the pool, it binds the item to the item's `RedisConnectionFactory`, without opening a connection or replacing the factory. A factory can be bound to one item only, and binding it again to the same item is allowed. Code that builds a `Pool` of `super-kernel/pool` from a `RedisConfig` itself calls `$factory->bind($option)` first, or creates `new RedisConnectionFactory($option)`; a factory without an item cannot create connections. No timer is started: run the pool's maintenance yourself if needed.

### Shut down

Inject `RedisPool` and call `close()` in its own coroutine while the coroutine runtime still runs, for example during application shutdown. It closes every pool for good: later borrows and proxy calls raise `The Redis pool is closed.` Connections that are still borrowed may finish their work in their coroutine and are closed when it ends. `RedisFactory` has no shutdown method.

Closing a connection when its coroutine ends happens in a deferred callback, which cannot throw. A failure there is kept by `RedisPool` and raised once, as a `SuperKernel\Redis\Exception\RedisConnectionException`, by the next borrow or `close()`; credentials never appear in such messages. A connection whose close failed keeps its pool slot. Call `close()` at shutdown even when no further command would surface a kept failure; calling it again retries every failed close, including those of connections retired after the first call, and leaves closed connections alone.

### Processes and threads

`RedisPool` belongs to the process or thread in which the container created it, and so do its pools and the `CoroutineCapacity` of each item. In applications that fork or start threads, resolve Redis services and load items only after the fork, separately in each thread; using a pool from another process or thread raises `A Redis pool cannot be shared between processes or threads.` Pools, factories, capacities, and sockets are never moved between processes or threads.

## Errors

| Situation | Message |
| --- | --- |
| An empty host or one with a NUL byte | `The Redis host must be non-empty and contain no NUL bytes.` |
| A port outside `1` to `65535` | `The Redis TCP port must be between 1 and 65535.` |
| A negative database | `The Redis database index must be non-negative.` |
| A user name without a password, or an empty one | `A Redis ACL username requires a password and cannot be empty.` |
| A timeout below one microsecond or not finite | `Redis connectTimeout must be finite and at least one microsecond.` or `Redis readTimeout must be finite and at least one microsecond.` |
| A pool limit below one | `The Redis pool must allow at least one connection.` |
| An invalid pool wait timeout | `The Redis pool wait timeout must be finite and non-negative.` |
| A capacity that differs from the pool limit | `The Redis pool capacity must match its connection limit.` |
| A configuration item of another type | `The Redis configuration item must be a RedisConfig.` |
| An unknown name | `No Redis configuration item is registered for this name.` (`ConfigurationNotFoundException`) |
| Several items, no default, unnamed access | `Configuration contract SuperKernel\Redis\Config\RedisConfig has multiple items but no default item is declared.` (`ConfigurationNotFoundException`) |
| A negative database override | `The Redis database index must not be negative.` (`InvalidArgumentException`) |
| One `PoolOption` in two items | `A pool option cannot be reused with another Redis configuration.` (`InvalidArgumentException`) |
| One `RedisConnectionFactory` in two items | `A Redis connection factory belongs to one Redis option.` (`InvalidArgumentException`) |
| A borrow outside a coroutine | `Redis connection acquisition requires an active Swoole coroutine.` (`PoolException`) |
| A pool used from another process or thread | `A Redis pool cannot be shared between processes or threads.` (`PoolException`) |
| `discard()` from another coroutine | `The Redis connection does not belong to the current coroutine.` (`PoolException`) |
| The proxy before `RedisFactory` exists | `The Redis proxy requires initialization through RedisFactoryProvider.` (`LogicException`) |
| A borrow or proxy call after `close()` | `The Redis pool is closed.` (`LogicException`) |
| Reuse after a failed proxy call | `The native Redis session has already been retired.` (`LogicException`) |
| Connecting, authenticating, or selecting the database fails | `Redis connection initialization failed.`, with a cause such as `Redis connection establishment failed.`, `Redis authentication failed.`, or `Selecting the Redis database failed.` (`RedisConnectionException`) |
| Closing a connection failed | `Redis connection cleanup failed.` (`RedisConnectionException`) |

Messages without a class are `SuperKernel\Redis\Exception\InvalidRedisConfigurationException`, raised before a pool is created. `RedisConnectionException` and `InvalidRedisConfigurationException` are in `SuperKernel\Redis\Exception` and implement its `RedisExceptionInterface`.

## Relation to other packages

- [`super-kernel/pool`](pool.md) provides the pools, `PoolOption`, the capacities, health checks, and maintenance policies used by each item.
- [`super-kernel/config`](config.md) finds the `#[Configuration]` source, names and keeps the items, and selects the default.
- [`super-kernel/di`](di.md) provides `RedisFactory`, `RedisPool`, `RedisOptionLoader`, and the proxy to your services.
- [`super-kernel/swoole`](swoole.md) and [`super-kernel/http-server`](http-server.md) run the coroutines in which connections are borrowed.

## What not to do

**Do not use a connection outside the coroutine that borrowed it.** It is discarded when that coroutine ends, and another coroutine using it would interleave commands on one socket. Call `RedisFactory::get()` in every coroutine that needs Redis.

**Do not keep a coroutine alive with idle connections.** A connection holds its pool slot until its coroutine ends, so a long-running loop that borrowed once blocks that slot for good. Borrow in short-lived coroutines, or size the pool for the connections that long-lived ones hold.

**Do not share a `PoolOption` or a `RedisConnectionFactory` between items.** They are rejected with `A pool option cannot be reused with another Redis configuration.` and `A Redis connection factory belongs to one Redis option.` Create both inside each item method.

**Do not use the proxy for the `scan` family or for named items.** A magic call cannot update a by-reference cursor, and the proxy always uses the default item. Use the native result of `RedisFactory::get()`.

**Do not call the proxy before the container has created `RedisFactory`.** It raises `The Redis proxy requires initialization through RedisFactoryProvider.` Resolve `RedisFactory`, or a service that depends on it, first.

**Do not treat the proxy's `close()` as shutdown.** It closes only the current native connection. Call `RedisPool::close()` to shut the pools down.

**Do not skip `RedisPool::close()` at shutdown.** A failure to close a connection is kept and reported only by the next borrow or `close()`, and its pool slot stays occupied until a close succeeds. Call it in the owning coroutine before the runtime ends.

**Do not share pools between processes or threads.** A pool belongs to the process or thread that created it and raises `A Redis pool cannot be shared between processes or threads.` elsewhere. Let each process or thread borrow through its own container.
