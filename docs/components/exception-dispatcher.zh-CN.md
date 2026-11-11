<p align="right">
  <a href="exception-dispatcher.md">English</a> | <strong>中文文档</strong>
</p>

# SuperKernel Exception Dispatcher

包：[`super-kernel/exception-dispatcher`](https://github.com/super-kernel/exception-dispatcher) · [Packagist](https://packagist.org/packages/super-kernel/exception-dispatcher) · [文档索引](../README.zh-CN.md)

## 用途

`super-kernel/exception-dispatcher` 决定到达应用某个边界的异常由谁获知。边界（例如由 `super-kernel/framework` 运行的控制台命令）把异常交给 `SuperKernel\Contract\ExceptionDispatcherInterface`；本包调用为其类声明的每个处理器，并在它们都没有完成时通过标准输出日志记录器记录完整的异常。

本包只负责报告。接下来发生什么（例如退出状态、HTTP 响应或重试）由捕获异常的边界决定。它在容器就绪之后工作，不处理此前发生的任何失败。

## 能力

### 声明处理器

处理器是一个带有 `SuperKernel\ExceptionDispatcher\Attribute\ExceptionHandler` 并实现 `SuperKernel\ExceptionDispatcher\Contract\ExceptionHandlerInterface` 的类。该注解接受它处理的类 `exception`（默认为 `Throwable::class`）和 `priority`（默认为 `0`）。处理器接手所声明的类及其每个子类。处理器由容器创建，因此处理器通过构造函数接收服务。

一个处理器，保存为 `src/ExceptionHandler/ReportFailure.php`：

```php
<?php

declare(strict_types=1);

namespace Src\ExceptionHandler;

use RuntimeException;
use SuperKernel\ExceptionDispatcher\Attribute\ExceptionHandler;
use SuperKernel\ExceptionDispatcher\Contract\ExceptionHandlerInterface;
use Throwable;

#[ExceptionHandler(RuntimeException::class)]
final readonly class ReportFailure implements ExceptionHandlerInterface
{
    public function handle(Throwable $throwable): void
    {
        echo 'Failed: ', $throwable->getMessage(), PHP_EOL;
    }
}
```

分发一个异常的脚本，保存为 `bin/report.php`：

```php
<?php

declare(strict_types=1);

use SuperKernel\Context\ApplicationContext;
use SuperKernel\Contract\ExceptionDispatcherInterface;

require dirname(__DIR__) . '/vendor/autoload.php';

$exceptions = ApplicationContext::getContainer()->get(ExceptionDispatcherInterface::class);

$exceptions->dispatch(new RuntimeException('The invoice feed is unavailable.'));
```

构建项目并运行脚本：

```bash
skernel build
php bin/report.php
```

它会打印：

```text
Failed: The invoice feed is unavailable.
```

### 为处理器排序

`dispatch()` 调用每个匹配的处理器，而不只是第一个：优先级高的先运行，优先级相同的保持构建发现处理器的顺序。没有能阻止其他处理器的结果。`Throwable` 的处理器会看到每个异常。

处理所有异常的处理器，保存为 `src/ExceptionHandler/NotifyOperator.php`：

```php
<?php

declare(strict_types=1);

namespace Src\ExceptionHandler;

use SuperKernel\ExceptionDispatcher\Attribute\ExceptionHandler;
use SuperKernel\ExceptionDispatcher\Contract\ExceptionHandlerInterface;
use Throwable;

#[ExceptionHandler(priority: 10)]
final readonly class NotifyOperator implements ExceptionHandlerInterface
{
    public function handle(Throwable $throwable): void
    {
        echo 'Operator notified about ', $throwable::class, PHP_EOL;
    }
}
```

构建项目并再次运行脚本：

```bash
skernel build
php bin/report.php
```

它会打印：

```text
Operator notified about RuntimeException
Failed: The invoice feed is unavailable.
```

### 依靠兜底记录

当没有匹配的处理器完成时（因为没有处理器匹配，或者每个匹配的处理器都抛出了异常），分发器通过 `SuperKernel\Contract\StdoutLoggerInterface` 以错误级别写出一次原始异常。记录内容是 PHP 对该异常的完整字符串，包括它的文件、行号、调用栈和前一个异常。使用 `super-kernel/logger` 时，记录形如：

```text
2026-11-10T10:36:46.370+00:00 [ERROR] RuntimeException: The invoice feed is unavailable. in /app/src/Command/ImportCommand.php:18
Stack trace:
#0 ...
```

抛出异常的处理器不会中止分发。它的失败通过同一个日志记录器写出为 `Exception handler <handler> failed while processing <exception class>: <failure>`，其中 `<failure>` 是该处理器的完整异常，然后下一个匹配的处理器运行。这个失败永远不会被再次分发。日志记录器本身抛出的异常会向外传播。

应用不需要为此做任何配置。要改变输出，请声明一个 `Throwable` 的处理器，或者按照 [`super-kernel/logger`](logger.zh-CN.md) 页面的说明提供另一个 `StdoutLoggerInterface`，这会改变框架写出的每条记录的阈值、格式和流。

### 了解异常从何而来

本包自己不捕获任何异常。`super-kernel/framework` 分发控制台命令的异常，以及应用启动后未被捕获的异常；[`super-kernel/framework`](framework.zh-CN.md) 页面展示了这两种情况。服务器和协议包分发它们自己回调中的异常，或者像下一节所描述的那样维护自己的处理器家族。你自己的代码也可以像 `bin/report.php` 那样分发它捕获的异常，以同样的方式报告它。

### 为其他边界维护处理器家族

需要另一种契约的处理器（例如返回响应的 HTTP 处理器）的边界，会声明自己的、继承 `ExceptionHandler` 的注解。通用分发器只调用恰好用 `#[ExceptionHandler]` 声明的处理器，因此派生的家族留在它的边界内。`super-kernel/http-server` 就是这样维护它的 HTTP 异常处理器的。

派生注解必须声明 PHP 自己的 `#[Attribute(Attribute::TARGET_CLASS)]`，因为 PHP 不会继承它；一个处理器恰好带有该家族的一个注解。容器提供 `SuperKernel\ExceptionDispatcher\Registry\ExceptionHandlerProvider`：`getHandlersForAttribute()` 按优先级顺序返回恰好属于某一个注解类的 `SuperKernel\ExceptionDispatcher\Registry\ExceptionHandlerRegistration`。每个注册项持有 `declaration` 和创建好的 `handler`，`matches()` 应用与分发器相同的类规则。边界检查自己的处理器契约，并决定处理器的结果意味着什么。

注解，保存为 `src/Queue/QueueExceptionHandler.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Queue;

use Attribute;
use SuperKernel\ExceptionDispatcher\Attribute\ExceptionHandler;

#[Attribute(Attribute::TARGET_CLASS)]
final readonly class QueueExceptionHandler extends ExceptionHandler
{
}
```

处理器契约，保存为 `src/Queue/QueueFailureHandler.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Queue;

use Throwable;

interface QueueFailureHandler
{
    public function handle(Throwable $throwable, string $job): bool;
}
```

一个处理器，保存为 `src/Queue/RetryLater.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Queue;

use RuntimeException;
use Throwable;

#[QueueExceptionHandler(RuntimeException::class)]
final readonly class RetryLater implements QueueFailureHandler
{
    public function handle(Throwable $throwable, string $job): bool
    {
        echo "Job {$job} will be retried: {$throwable->getMessage()}", PHP_EOL;

        return true;
    }
}
```

使用该家族并回退到通用分发器的工作者，保存为 `src/Queue/Worker.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Queue;

use SuperKernel\Contract\ExceptionDispatcherInterface;
use SuperKernel\ExceptionDispatcher\Registry\ExceptionHandlerProvider;
use Throwable;

final readonly class Worker
{
    public function __construct(
        private ExceptionHandlerProvider     $handlers,
        private ExceptionDispatcherInterface $exceptions,
    )
    {
    }

    public function fail(string $job, Throwable $throwable): void
    {
        foreach ($this->handlers->getHandlersForAttribute(QueueExceptionHandler::class) as $registration) {
            if (!$registration->matches($throwable)) {
                continue;
            }

            $handler = $registration->handler;
            assert($handler instanceof QueueFailureHandler);

            if ($handler->handle($throwable, $job)) {
                return;
            }
        }

        $this->exceptions->dispatch($throwable);
    }
}
```

报告两个失败任务的脚本，保存为 `bin/worker.php`：

```php
<?php

declare(strict_types=1);

use Src\Queue\Worker;
use SuperKernel\Context\ApplicationContext;

require dirname(__DIR__) . '/vendor/autoload.php';

$worker = ApplicationContext::getContainer()->get(Worker::class);

$worker->fail('send-invoice', new RuntimeException('The mail server is busy.'));
$worker->fail('send-invoice', new LogicException('The invoice has no recipient.'));
```

构建项目并运行脚本：

```bash
skernel build
php bin/worker.php
```

它会打印：

```text
Job send-invoice will be retried: The mail server is busy.
Operator notified about LogicException
```

## 错误

容器第一次创建处理器注册表时（与分发器一起）收集并创建处理器。无效的声明会抛出 `SuperKernel\ExceptionDispatcher\Exception\InvalidExceptionHandlerDeclarationException`，它实现 `SuperKernel\ExceptionDispatcher\Exception\ExceptionDispatcherExceptionInterface`，并被包装在一个 `SuperKernel\Di\Exception\ResolutionException` 中。`super-kernel/framework` 的启动监听器需要分发器，因此安装了它时，该错误也会使 `skernel build` 和应用启动失败。

| 情形 | 消息 |
| --- | --- |
| 带有 `#[ExceptionHandler]` 的类没有实现 `ExceptionHandlerInterface` | `Exception handler <class> marked with SuperKernel\ExceptionDispatcher\Attribute\ExceptionHandler must implement SuperKernel\ExceptionDispatcher\Contract\ExceptionHandlerInterface; <type> provided.` |
| 一个类带有该家族的多个注解 | `<class> must declare exactly one Attribute derived from SuperKernel\ExceptionDispatcher\Attribute\ExceptionHandler; <count> found.` |
| 派生注解可重复，或者允许用于类以外的目标 | `<attribute> on <class> must be a non-repeatable class-only Attribute.` |
| `exception` 不是 `Throwable` 类 | `<attribute> on <class> must declare a Throwable class; <class> provided.` |

容器无法创建的处理器会在同一时刻抛出容器自己的异常。没有声明 PHP 自己的 `#[Attribute]` 的派生注解根本无法读取：`skernel build` 会跳过使用它的每个类，并用 PHP 的消息 `Attempting to use non-attribute class "<attribute>" as attribute` 报告，该处理器永远不会被注册。

## 与其他包的关系

- [`super-kernel/contract`](contract.zh-CN.md) 声明 `ExceptionDispatcherInterface` 和 `StdoutLoggerInterface`；[`super-kernel/logger`](logger.zh-CN.md) 提供接收兜底记录的标准输出日志记录器。
- [`super-kernel/framework`](framework.zh-CN.md) 分发控制台异常和未被捕获的异常；[`super-kernel/http-server`](http-server.zh-CN.md) 维护自己从 `ExceptionHandler` 派生的处理器家族。
- [`super-kernel/di`](di.zh-CN.md) 创建处理器；[`super-kernel/annotator`](annotator.zh-CN.md) 提供记录下来的声明，包括派生注解。
- `skernel build` 收集处理器；`skernel run` 在启动应用之前先构建。

## 不要这样做

**不要通过在处理器中抛出异常来阻止其他处理器。** 处理器的异常会被记录为处理器失败，其余的处理器仍会运行。每个匹配的处理器都会运行；如果某个处理器不应看到某个异常，请为它指定更窄的 `exception` 类。

**不要派生没有自己的 `#[Attribute(Attribute::TARGET_CLASS)]` 的处理器注解。** PHP 不会继承该声明，因此 `skernel build` 会跳过使用该注解的每个类，并报告 `Attempting to use non-attribute class "<attribute>" as attribute`，这些处理器会静默地永远不被调用。可重复的注解，或者允许用于其他目标的注解，会被拒绝并报告 `<attribute> on <class> must be a non-repeatable class-only Attribute.` 请在每个派生注解上恰好声明 `#[Attribute(Attribute::TARGET_CLASS)]`。
