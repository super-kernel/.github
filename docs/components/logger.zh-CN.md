<p align="right">
  <a href="logger.md">English</a> | <strong>中文文档</strong>
</p>

# SuperKernel Logger

包：[`super-kernel/logger`](https://github.com/super-kernel/logger) · [Packagist](https://packagist.org/packages/super-kernel/logger) · [文档索引](../README.zh-CN.md)

## 用途

`super-kernel/logger` 决定 SuperKernel 自己的日志记录写到哪里，并为应用提供一种创建 Monolog 日志记录器的简便方式。容器提供 `SuperKernel\Contract\StdoutLoggerInterface`，这是一个把易读的行写到标准输出的 PSR-3 日志记录器；SuperKernel 的各个包（包括 `super-kernel/exception-dispatcher` 的异常兜底）都通过它写入。`LoggerFactory` 为应用选择的流创建更多的日志记录器。

应用用哪个日志记录器写自己的记录，以及如何按部署进行配置，都由应用决定：本包不绑定 `Psr\Log\LoggerInterface`。

## 能力

### 写入标准输出日志记录器

通过构造函数接收 `StdoutLoggerInterface`，然后调用任何 PSR-3 方法。默认服务在 `super-kernel` 通道上写到 `php://stdout`，保留 `INFO` 及以上级别，并用 `ConsoleFormatter` 格式化每条记录。记录原样到达 Monolog，级别由调用方选择。

异常记录使用同一个服务：调用方选择错误级别并格式化异常，没有单独的异常日志记录器、通道或标准错误流。

一个写日志的服务，保存为 `src/Billing/InvoiceImporter.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Billing;

use SuperKernel\Contract\StdoutLoggerInterface;

final readonly class InvoiceImporter
{
    public function __construct(private StdoutLoggerInterface $logger)
    {
    }

    public function import(string $invoiceNumber): void
    {
        $this->logger->debug("Reading invoice {$invoiceNumber}.");
        $this->logger->info("Invoice {$invoiceNumber} imported.");
    }
}
```

使用它的脚本，保存为 `bin/import.php`：

```php
<?php

declare(strict_types=1);

use Src\Billing\InvoiceImporter;
use SuperKernel\Context\ApplicationContext;

require dirname(__DIR__) . '/vendor/autoload.php';

ApplicationContext::getContainer()->get(InvoiceImporter::class)->import('INV-1001');
```

构建项目并运行脚本：

```bash
skernel build
php bin/import.php
```

它会打印类似这样的一行：

```text
2026-11-10T10:56:25.283+00:00 [INFO] Invoice INV-1001 imported.
```

`DEBUG` 记录低于阈值，不会被写出。

### 读懂控制台格式

`SuperKernel\Logger\Formatter\ConsoleFormatter` 为每条记录写一行：一个可排序的、带毫秒和时区偏移的 ISO 8601 时间戳，方括号中的 PSR-3 级别，恰好一个空格，然后是消息，最后是一个换行符。它省略上下文和附加数据，也没有通道、进程或线程列。该布局受 Spring Boot 的控制台输出启发：

```text
2026-08-18T03:05:46.007+00:00 [NOTICE] Swoole main server started in base mode.
2026-08-18T03:05:46.008+00:00 [INFO] Swoole listener "http" started on 127.0.0.1:9501 (http).
```

消息按原样写出。`{workerId}` 之类的占位符不会用上下文替换，因此请在记录之前构造好你想显示的句子。结构化输出应当使用另一个带其他格式化器的日志记录器，例如由 `LoggerFactory` 创建且不指定格式化器的日志记录器。

`new ConsoleFormatter()` 写出纯文本。`new ConsoleFormatter(decorated: true)` 添加 ANSI 颜色：时间戳为暗色，错误及以上级别为红色，`WARNING` 为黄色，`NOTICE` 为青色，`INFO` 和 `DEBUG` 为绿色；消息从不着色。`isDecorated()` 报告所做的选择。只有当标准输出是交互式终端、`NO_COLOR` 未设置并且 `TERM` 不是 `dumb` 时，标准输出日志记录器才会着色，因此重定向的输出保持纯文本。

### 创建日志记录器

`SuperKernel\Logger\Factory\LoggerFactory::create()` 返回一个 `Psr\Log\LoggerInterface`，其背后是一个带一个流处理器的同步 Monolog 日志记录器：

- `channel` 和 `stream` 是必填的；流是一个路径或 `php://stderr` 之类的 PHP 流；
- `minimumLevel` 是一个 PSR-3 级别，默认为 `LogLevel::INFO`；未知级别会抛出 `Psr\Log\InvalidArgumentException`；
- `formatter` 是可选的 Monolog 格式化器；没有它时使用 Monolog 默认的行格式。

每次调用都会创建一个新的日志记录器；保留它是容器的职责。Monolog 在写入第一条记录时才打开流，因此无法打开或写入的流在那时以 Monolog 的异常失败。该工厂不会读取环境文件、对敏感上下文脱敏、轮转文件、缓冲记录、把记录发送到远程服务，也不会异步或以协程安全的方式投递记录；这些策略由使用该工厂的服务决定。

本包不绑定任何 `Psr\Log\LoggerInterface`。需要它的应用自己声明，例如使用一个工厂。

提供应用的 `LoggerInterface` 的工厂，保存为 `src/Logger/AuditLoggerProvider.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Logger;

use Psr\Log\LoggerInterface;
use Psr\Log\LogLevel;
use SuperKernel\Attribute\FactoryProvider;
use SuperKernel\Contract\BasePathInterface;
use SuperKernel\Logger\Factory\LoggerFactory;

#[FactoryProvider(LoggerInterface::class)]
final readonly class AuditLoggerProvider
{
    public function __invoke(LoggerFactory $factory, BasePathInterface $basePath): LoggerInterface
    {
        return $factory->create(
            channel: 'audit',
            stream: $basePath . '/audit.log',
            minimumLevel: LogLevel::WARNING,
        );
    }
}
```

写入它的脚本，保存为 `bin/audit.php`：

```php
<?php

declare(strict_types=1);

use Psr\Log\LoggerInterface;
use SuperKernel\Context\ApplicationContext;

require dirname(__DIR__) . '/vendor/autoload.php';

$audit = ApplicationContext::getContainer()->get(LoggerInterface::class);

$audit->info('Invoice INV-1001 viewed.');
$audit->warning('Invoice INV-1001 deleted.', ['user' => 'ada']);
```

构建项目并运行脚本：

```bash
skernel build
php bin/audit.php
```

它什么都不打印；随后项目目录中的 `audit.log` 包含类似这样的一行：

```text
[2026-11-10T10:56:25.306140+00:00] audit.WARNING: Invoice INV-1001 deleted. {"user":"ada"} []
```

### 替换标准输出日志记录器

每个 SuperKernel 包都通过已注册的 `StdoutLoggerInterface` 写入，因此替换实现决定了这些包的所有记录（以及使用同一服务的应用记录）的阈值、格式和流。请把替换实现声明为带有 `#[Provider(StdoutLoggerInterface::class)]` 的类；本包自己的 `#[FactoryProvider]` 随即不再使用。不要为该接口再声明一个 `#[FactoryProvider]`：同一个标识符的两个工厂会冲突。当级别或流取决于部署时，请把配置或环境服务注入替换实现。

把 `WARNING` 及以上级别写到标准错误的替换实现，保存为 `src/Logger/AppLogger.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Logger;

use Psr\Log\LoggerInterface;
use Psr\Log\LoggerTrait;
use Psr\Log\LogLevel;
use Stringable;
use SuperKernel\Attribute\Provider;
use SuperKernel\Contract\StdoutLoggerInterface;
use SuperKernel\Logger\Factory\LoggerFactory;
use SuperKernel\Logger\Formatter\ConsoleFormatter;

#[Provider(StdoutLoggerInterface::class)]
final readonly class AppLogger implements StdoutLoggerInterface
{
    use LoggerTrait;

    private LoggerInterface $logger;

    public function __construct(LoggerFactory $factory)
    {
        $this->logger = $factory->create(
            channel: 'application',
            stream: 'php://stderr',
            minimumLevel: LogLevel::WARNING,
            formatter: new ConsoleFormatter(),
        );
    }

    public function log(mixed $level, string|Stringable $message, array $context = []): void
    {
        $this->logger->log($level, $message, $context);
    }
}
```

构建项目并再次运行导入脚本：

```bash
skernel build
php bin/import.php
```

它什么都不打印：`INFO` 记录低于替换实现的阈值。`WARNING` 记录会出现在标准错误上。要保留默认的阈值和目标，请使用 `LogLevel::INFO` 和 `php://stdout`。

### 保持机器可读的输出干净

默认日志记录器写到标准输出，因此它的行可能出现在命令自己的输出之前或之间。当标准输出必须是机器可读的协议时，请像上面那样把日志记录器替换为写到 `php://stderr` 或文件的实现。没有针对单个命令的开关；替换实现作用于进程的每条记录。

## 与其他包的关系

- [`super-kernel/contract`](contract.zh-CN.md) 声明 `StdoutLoggerInterface`；每个写日志的包都依赖该接口，而不是本包。
- [`super-kernel/exception-dispatcher`](exception-dispatcher.zh-CN.md) 通过标准输出日志记录器写出没有处理器接手的异常以及处理器的失败。
- [`super-kernel/di`](di.zh-CN.md) 创建并保留日志记录器；[`super-kernel/attribute`](attribute.zh-CN.md) 声明用于替换或添加日志记录器的 `#[Provider]` 和 `#[FactoryProvider]`。
- Monolog 提供日志记录器、处理器和格式化器。

## 不要这样做

**不要用 `#[FactoryProvider]` 替换标准输出日志记录器。** 本包已经为 `StdoutLoggerInterface` 声明了一个，因此再声明一个会使每个需要新定义的容器查找都失败，并报告 `Factory provider for SuperKernel\Contract\StdoutLoggerInterface is declared by both <class> and SuperKernel\Logger\Provider\StdoutLoggerProvider.` 请在替换类上使用 `#[Provider(StdoutLoggerInterface::class)]`。

**不要在控制台格式中依赖占位符。** `$logger->warning('Worker {workerId} stopped.', ['workerId' => 7])` 会原样打印 `Worker {workerId} stopped.`，上下文也不会显示。请把值放进消息中，或者使用带有能渲染它们的格式化器或处理器的日志记录器。
