<p align="right">
  <strong>English</strong> | <a href="pool.zh-CN.md">中文文档</a>
</p>

# SuperKernel Pool

Package: [`super-kernel/pool`](https://github.com/super-kernel/pool) · [Packagist](https://packagist.org/packages/super-kernel/pool) · [Documentation index](../README.md)

## Purpose

`super-kernel/pool` keeps a bounded set of connections of any kind and lends them to callers. The caller wraps its client in a connection class and supplies a factory; the pool decides when to create, check, reuse, replace, and close those connections, and keeps their number within a limit that can be shared by the coroutines of one process, by forked processes, or by Swoole threads.

The pool knows nothing about protocols. Opening, authenticating, checking, resetting, and closing a connection belong to the caller's connection class, and so does every I/O deadline. The package depends on no other SuperKernel package and registers no services: a driver, such as `super-kernel/redis`, or the application assembles each pool and provides it to its consumers.

## Capabilities

### Write a connection and its factory

A connection implements `SuperKernel\Pool\Contract\ConnectionInterface`:

- `connect()` opens the client. The pool calls it once for every new connection, before the first borrower receives it.
- `isHealthy()` reports whether the connection can be used again. A `false` result makes the pool close the connection and create a replacement through the factory when one is needed; an exception propagates.
- `reconnect()` is for the current borrower, who may call it explicitly. The pool itself never reconnects a borrowed connection and never repeats a request.
- `close()` releases the client. It must tolerate being called again after a failure, because the pool retries a failed close.

The connection class owns authentication, cleanup of a half-opened client, resetting transactions and sessions before the connection goes back, and its own deadlines. A factory implements `SuperKernel\Pool\Contract\ConnectionFactoryInterface`, whose `create()` returns a new, unopened connection each time; returning a connection the pool already holds raises `The factory returned a connection already owned by this pool.`

The examples on this page use the following connection and factory. A connection that only counts how many connections were opened, saved as `src/Pool/CounterConnection.php`:

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

Its factory, saved as `src/Pool/CounterConnectionFactory.php`:

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

### Assemble a pool

`SuperKernel\Pool\Config\PoolOption` is a final readonly class that holds a pool's settings; a pool is created with `new Pool($option)`:

| Argument | Default | Meaning |
| --- | --- | --- |
| `factory` | required | The `ConnectionFactoryInterface` that creates connections. |
| `capacity` | required | The capacity that limits connections: `CoroutineCapacity`, `ProcessCapacity`, `ThreadCapacity`, or a custom one. |
| `maxConnections` | required | The fixed limit; it must equal the capacity's limit. |
| `waitTimeout` | `1.0` | Seconds that `acquire()` waits for capacity; `0` means no wait. |
| `healthChecks` | `HealthCheckMode::OnBorrowAndReturn` | When connections are checked. |
| `frequency` | `new ConstantFrequency()` | The demand policy that maintenance follows. |

The option keeps fixed values and references; the factory, the capacity, and the frequency policy keep their own state. The pool checks that the limit is positive and matches the capacity and that the timeout is finite and not negative, before any I/O. Creating an option or a pool starts no coroutine, timer, worker, or connection. Use `PoolOption` itself: there is no option interface to implement. It is not a configuration source of `super-kernel/config`; a driver or an application provider builds it from its own settings and exposes the resulting pool as a typed service.

### Borrow and return connections

`acquire(?float $timeout = null)` returns a connection. With `null` it waits up to the option's timeout; with `0` it does not wait. When no capacity becomes free in time, it raises `SuperKernel\Pool\Exception\PoolTimeoutException`. Only the wait for capacity is timed; creating and opening the connection uses the connection's own deadlines. Every call counts as a borrow attempt for the demand policy, also when it later times out.

Give every connection back exactly once, to the pool that lent it: `release($connection)` when it can be reused, `discard($connection)` when it is broken or cannot be reset. A second return, or a connection from another pool, raises `The connection is not currently borrowed from this pool.` The pool takes the connection out of the borrowed set before it checks or closes it, so a second return is rejected even while the first is still being cleaned up; capacity stays reserved until that cleanup ends. When closing fails, the pool keeps the capacity reserved and retries the close at the next maintenance pass or `close()`; never return the same object again to retry.

The pool cannot tell borrowers apart. Stop using a connection and restore its session state before returning it: there is no automatic return and no lease, and once the object is lent again, a stale reference held by the previous borrower is indistinguishable from the new borrower's.

When creating, opening, or checking a connection fails, `acquire()` closes it, frees the capacity, and raises the original exception, even if the close fails too.

A script in which three coroutines compete for two connections, followed by a discard, a second return, and a borrow after shutdown, saved as `bin/contention.php`:

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

Build the project and run the script:

```bash
skernel build
php bin/contention.php
```

It prints:

```text
A holds connection 1
B holds connection 2
C: Timed out waiting for a connection.
Discarding connection 2
After the discard, the next borrower gets connection 1
SuperKernel\Pool\Exception\PoolException: The connection is not currently borrowed from this pool.
SuperKernel\Pool\Exception\PoolClosedException: The connection pool is closed.
```

### Count connections against the limit

`maxConnections()` is a fixed upper bound, not the number of idle connections. Every connection counts against it while it is idle, borrowed, being created, checked, or closed, including one whose close failed and waits for a retry. With `M` the limit and `B` the borrowed connections, the remaining capacity at rest is `P = M - B`. The ready idle connections `I` may be fewer than `P`: missing ones are created by maintenance.

| Completed transition | Remaining capacity `P` | Ready idle connections `I` |
| --- | --- | --- |
| Borrow a healthy idle connection | -1 | -1 |
| Return a reusable connection | +1 | +1 |
| Return a connection that is then closed | +1 | unchanged |
| Close an unhealthy idle connection | unchanged | -1 |
| Create a connection during maintenance | unchanged | +1 |

`CapacityInterface::available()` reports the slots that can still be reserved, not `I`; work in progress can occupy slots for a while. Only a successful creation produces a ready connection. Borrowed connections stay registered so that returns can be checked, but maintenance never touches them.

### Share the limit across coroutines, processes, or threads

| Capacity | Shared by | Idle connections | Assembly |
| --- | --- | --- | --- |
| `CoroutineCapacity` | the coroutines of one process or thread | kept and reused | One capacity per pool; every operation runs in a coroutine. |
| `ProcessCapacity` | forked processes | returned connections are closed by their owner | Create the capacity before forking, and each pool and factory after. |
| `ThreadCapacity` | Swoole threads | returned connections are closed by their owner | Create the shared state once and attach it in each thread. |

`new CoroutineCapacity($limit, $channel)` uses a `SuperKernel\Pool\Channel\SwooleChannel` by default; another empty `ChannelInterface<bool>` used only by this capacity can be passed instead. The channel is created lazily, so creating or closing an unused coroutine pool needs no coroutine.

`ProcessCapacity` keeps its reservations in Swoole shared memory and `ThreadCapacity` in a native thread map. Only reservations are shared: connection objects and sockets stay with the pool that created them, so a returned connection is closed instead of kept idle. Waiters poll with a monotonic deadline and pauses of at most 1 ms, yielding inside a coroutine and sleeping otherwise; there is no first-come, first-served order, and a stalled native mutex can delay a waiter beyond its timeout. Passing the same capacity to several pools shares one limit and one shutdown; separate capacities keep separate limits, so separate `CoroutineCapacity` objects do not limit an application as a whole.

For processes, create `new ProcessCapacity($limit)` before `Swoole\Process::start()`, and create a new pool from it inside each process. A script in which three processes share a limit of two, saved as `bin/processes.php`:

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

Build the project and run the script:

```bash
skernel build
php bin/processes.php
```

Two processes print `Process <n> holds a connection` and one prints `Process <n>: No connection capacity became available before the timeout.`, in an order that can vary; the last line is `Connections still reserved: 0`.

For threads, the creating thread calls `ThreadCapacity::create($limit)` and passes `$capacity->state()` and `$capacity->mutex()` as `Swoole\Thread` arguments. Each thread attaches `new ThreadCapacity($state, $mutex)` and creates its own factory and pool; only the two native handles cross the thread boundary.

Reservations do not expire. When a process or thread that holds capacity dies, waiting does not make that capacity safe to reuse: the supervisor must retire the capacity and create a new one after making sure the old owners have stopped. The package does not detect dead owners, repeat requests, or take over reservations.

### Check connection health

`SuperKernel\Pool\Constants\HealthCheckMode` decides when kept connections are checked with `isHealthy()`:

| Mode | Before an idle connection is reused | Before a returned connection is kept |
| --- | --- | --- |
| `OnBorrow` | yes | no |
| `OnReturn` | no | yes |
| `OnBorrowAndReturn` (default) | yes | yes |
| `Manual` | no | no |

A shared capacity closes every returned connection and therefore never checks one to keep it. When a borrow check finds an unhealthy idle connection, the pool closes it and tries the other idle connections before it creates one replacement; creation is not retried endlessly. Every mode allows explicit maintenance.

### Maintain the pool

`maintain(int $minimumIdle = 0)` runs one maintenance pass: it retries closes that failed, checks the idle connections, and creates or removes idle connections as the demand policy decides. It never touches borrowed connections, skips a pass while another runs, and counts connections being created or closed against the limit. `minimumIdle` is a best-effort floor, never above the limit, and must be `0` for a shared capacity, which keeps no idle connections.

The option's `frequency` supplies the demand policy, which maintenance reads once at the start of each pass of a pool that keeps idle connections:

- At normal demand, maintenance fills all capacity that borrowers and work in progress leave free, also when `minimumIdle` is `0`.
- At low demand, it closes idle connections above `minimumIdle`, all of them when it is `0`, and creates connections up to that minimum.

Neither target displaces borrowed connections or work in progress. Returns free their slots without creating replacements; the next pass does. When a close fails, its slot stays reserved, the pass still tries the remaining connections, and the first error is raised; each later pass retries such a close once and never while another close of it runs. A failed creation leaves a gap for the next pass. A closed pool and a shared capacity skip the demand policy, and returning a connection never consults it.

`SuperKernel\Pool\Frequency\ConstantFrequency($frequency = 1.0, $lowFrequencyThreshold = 1.0)` reports a fixed number of borrow attempts per second; demand is low only when that number is below the threshold, not equal to it. The default therefore always fills the pool, and `new ConstantFrequency(0.25, 1.0)` always shrinks it.

`SuperKernel\Pool\Frequency\SmartLowFrequency($lowFrequencyThreshold = 1.0, $windowSeconds = 60.0, $clock = null)` measures demand. It counts every valid `acquire()` call before the call waits for capacity, including calls that later time out or fail to connect; a call with an invalid timeout, on a closed pool, or from another execution context does not count. Each counted attempt adds `1 / windowSeconds` to the estimated attempts per second, and between attempts the estimate decays by `exp(-elapsedSeconds / windowSeconds)` on a monotonic clock, also when it is read, so the estimate falls without new borrows. It never reports low demand during its first window. The estimate is smoothed, not an exact count over a sliding window, and saturates at `PHP_FLOAT_MAX`. The optional `$clock` is a `Closure(): float` that returns finite monotonic seconds without yielding, for deterministic tests; it defaults to `hrtime()`. A clock that fails, returns a non-finite value, or goes backwards raises an error and leaves the estimate unchanged.

Policies start no timer and perform no I/O. Give every pool its own `SmartLowFrequency`, which keeps state; a `ConstantFrequency` can be shared. A custom policy implements `SuperKernel\Pool\Contract\FrequencyInterface` and must not yield or perform I/O.

Run maintenance by hand, from an application event or task, from any scheduler, or with `SuperKernel\Pool\Maintenance\SwooleTimerScheduler`. The scheduler takes the pool, the interval in milliseconds, an error callback, and `minimumIdle`; `start()` starts a Swoole timer that runs each pass in a coroutine, skips overlapping passes, and passes failures to the callback, which must not throw. `stop()` cancels the next passes and lets a running one finish; a closed pool stops the timer at its next tick, and a closed pool cannot be scheduled. Nothing is scheduled by a constructor.

A script in which a timer fills a pool of three connections, saved as `bin/maintenance.php`:

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

Build the project and run the script:

```bash
skernel build
php bin/maintenance.php
```

It prints:

```text
Before maintenance: 0 connections
After maintenance: 3 connections
```

The policy is still in its first window, so demand is not low and the pass fills the free capacity.

### Shut down

`close()` closes the capacity for good, also for every pool that shares it, and wakes its waiters with `PoolClosedException`. It tries to close every idle connection and then raises the first error. Borrowed connections are closed when they are returned, so every borrower must still return them; a connection being created or checked during shutdown is never lent. Calling `close()` again retries failed closes and leaves closed connections alone. Nothing is closed by a destructor: close coroutine pools inside their coroutine runtime and stop their scheduler as part of the owner's shutdown.

### Use the channel

`SuperKernel\Pool\Channel\SwooleChannel($capacity)` is a bounded first-in, first-out channel for coroutines that keeps every value as it is, `null` and `false` included. `push()` on a full channel fails at once, `pop($timeout)` with `0` never waits, and `close()` wakes waiting readers. The native channel is created at first use, and every push or pop needs a coroutine. `CoroutineCapacity` uses it; other code may use it through `SuperKernel\Pool\Contract\ChannelInterface`.

### Write a custom capacity or pool

A custom capacity implements `SuperKernel\Pool\Contract\CapacityInterface`; its central methods are `reserve(float $timeout, string $owner): string`, which returns a reservation token, and `release(string $reservation, string $owner)`. A custom pool implements `SuperKernel\Pool\Contract\PoolInterface`, including `maintain()`, which `SwooleTimerScheduler` calls.

## Errors

| Situation | Message |
| --- | --- |
| A limit that is not positive or differs from the capacity, or an invalid timeout | `Positive matching capacity and a finite timeout are required.` (`InvalidPoolConfigurationException`) |
| A capacity limit that is not positive | `Backend capacity must be positive.` (`InvalidArgumentException`) |
| A negative or non-finite timeout for `acquire()` | `The connection wait timeout must be finite and non-negative.` (`InvalidArgumentException`) |
| No capacity in time, coroutine capacity | `Timed out waiting for a connection.` (`PoolTimeoutException`) |
| No capacity in time, shared capacity | `No connection capacity became available before the timeout.` (`PoolTimeoutException`) |
| The waiting coroutine was canceled | `Connection acquisition was canceled.` (`PoolException`) |
| A borrow after `close()` | `The connection pool is closed.` (`PoolClosedException`) |
| A coroutine pool used outside a coroutine | `Connection pool operations require an active Swoole coroutine.` (`PoolException`) |
| A pool used from another process or thread | `Create each resource-owning pool in its own process or thread.` (`PoolException`) |
| One `CoroutineCapacity` used by two pools | `A coroutine backend belongs to one resource-owning pool.` (`PoolException`) |
| A second return or a foreign connection | `The connection is not currently borrowed from this pool.` (`PoolException`) |
| The factory returns a connection the pool holds | `The factory returned a connection already owned by this pool.` (`PoolException`) |
| `minimumIdle` outside `0` to the limit | `The idle minimum must be between zero and capacity.` (`InvalidArgumentException`) |
| A positive `minimumIdle` with a shared capacity | `Shared backends retire owner-local resources and cannot retain idle connections.` (`PoolException`) |
| `ThreadCapacity` without thread support | `The thread backend requires PHP ZTS and Swoole thread support.` (`PoolException`) |
| An invalid scheduler interval or minimum | `A positive interval and a non-negative idle minimum are required.` (`InvalidArgumentException`) |
| Scheduling a closed pool | `Cannot schedule maintenance on a closed pool.` (`PoolException`) |
| Invalid `ConstantFrequency` or `SmartLowFrequency` arguments | `A finite non-negative frequency and a positive threshold are required.` or `A finite positive frequency threshold and observation window are required.` (`InvalidArgumentException`) |
| A push to a full channel | `The connection channel is full.` (`PoolException`) |
| A pop that times out | `No connection channel value is available.` (`PoolTimeoutException`) |

`PoolException`, `PoolTimeoutException`, `PoolClosedException`, and `InvalidPoolConfigurationException` are in `SuperKernel\Pool\Exception`; the last three extend `PoolException`.

## Relation to other packages

- [`super-kernel/redis`](redis.md) assembles one pool per Redis connection name with this package.
- Other packages and applications build pools for their own clients in the same way; this package depends on none of them.

## What not to do

**Do not use a pool in a process or thread other than the one that created it.** A pool and its connections belong to their creator; a pool inherited through a fork or passed to a thread raises `Create each resource-owning pool in its own process or thread.` Create the shared capacity before forking and a new pool in each process or thread.

**Do not return a connection twice or to another pool.** The pool raises `The connection is not currently borrowed from this pool.`, and once the object is lent again, a delayed second return would take it from its new borrower. Return each connection once, in a `finally` block.

**Do not keep using a connection after returning it.** The pool cannot revoke a reference, and the next borrower receives the same object. Stop using it and reset its session state before `release()`.

**Do not return a broken or dirty connection with `release()`.** A connection with an open transaction or a changed session would reach the next borrower as it is. Use `discard()` for connections that cannot be reset.

**Do not share one `CoroutineCapacity` between pools.** It belongs to one pool and raises `A coroutine backend belongs to one resource-owning pool.` for another. Use `ProcessCapacity` or `ThreadCapacity` for a limit across owners.

**Do not set a positive `minimumIdle` with a shared capacity.** Shared capacities close returned connections and keep none idle, so maintenance raises `Shared backends retire owner-local resources and cannot retain idle connections.` Use `0`.

**Do not throw from the scheduler's error callback.** The callback runs on the timer; report the error and return.

**Do not share a `SmartLowFrequency` between pools.** It measures the borrows it records, so a shared one mixes the demand of several pools. Give each pool its own.

**Do not reuse capacity after a process that held it died.** Its reservations are never released by time alone. Stop the remaining owners, then create a new capacity and new pools.
