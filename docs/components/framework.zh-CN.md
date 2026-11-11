<p align="right">
  <a href="framework.md">English</a> | <strong>中文文档</strong>
</p>

# SuperKernel Framework

包：[`super-kernel/framework`](https://github.com/super-kernel/framework) · [Packagist](https://packagist.org/packages/super-kernel/framework) · [文档索引](../README.zh-CN.md)

## 用途

`super-kernel/framework` 提供入口脚本所运行的应用。容器把它创建为 `SuperKernel\Contract\ApplicationInterface`：一个以根包命名的 Symfony Console 应用，接入应用的 PSR-14 事件分发器和共享的异常分发器。

本包既不决定有哪些命令，也不决定命令做什么。命令由 `super-kernel/command` 注册；服务器、协议回调和日志记录器属于提供它们的包，本包也不依赖 Swoole。异常处理器用 `super-kernel/exception-dispatcher` 声明。

## 能力

### 运行应用

共享接口只声明 `run(): int`，因此按 `ApplicationInterface` 编写类型的代码不需要任何 Symfony 类。该对象是一个 `SuperKernel\Framework\Application`，它继承 `Symfony\Component\Console\Application`，保留其输入、输出、命令和执行 API，包括 `run()` 的可选输入和输出参数。

入口脚本从容器中解析应用并运行它。入口脚本，保存为 `bin/bootstrap.php`：

```php
#!/usr/bin/env php
<?php

declare(strict_types=1);

use SuperKernel\Context\ApplicationContext;
use SuperKernel\Contract\ApplicationInterface;

require dirname(__DIR__) . '/vendor/autoload.php';

exit(ApplicationContext::getContainer()->get(ApplicationInterface::class)->run());
```

Symfony Console 的自动退出得到保留：`run()` 以命令的状态结束进程。嵌入应用的测试或其他调用方在具体类上调用 `setAutoExit(false)` 关闭自动退出，并传入自己的输入和输出。

运行应用并读取其输出的脚本，保存为 `bin/version.php`：

```php
<?php

declare(strict_types=1);

use SuperKernel\Context\ApplicationContext;
use SuperKernel\Contract\ApplicationInterface;
use Symfony\Component\Console\Application;
use Symfony\Component\Console\Input\ArrayInput;
use Symfony\Component\Console\Output\BufferedOutput;

require dirname(__DIR__) . '/vendor/autoload.php';

$application = ApplicationContext::getContainer()->get(ApplicationInterface::class);
assert($application instanceof Application);

$application->setAutoExit(false);

$output = new BufferedOutput();
$status = $application->run(new ArrayInput(['--version' => true]), $output);

echo trim($output->fetch()), " (status {$status})", PHP_EOL;
```

构建项目并运行脚本：

```bash
skernel build
php bin/version.php
```

它会打印：

```text
acme/billing 1.2.3 (status 0)
```

### 为应用命名并指定版本

根 `composer.json` 为应用命名。它的 `name` 成为应用名称，存在时它的 `version` 成为应用版本：

```json
{
    "name": "acme/billing",
    "version": "1.2.3"
}
```

构建项目并查询版本：

```bash
skernel build
php bin/bootstrap.php --version
```

它会打印：

```text
acme/billing 1.2.3
```

名称和版本也出现在 `list` 和 `help` 的标题中，以及 Symfony 描述器的身份字段中。没有 `version` 时保留 Symfony 的默认值，只显示名称；`super-kernel/skeleton` 模板没有声明版本，因此它的 `--version` 打印 `super-kernel/skeleton`。

两个值都按原样使用：本包不会修剪、校验、规范化这两个值，也不会就它们发出警告，并且从不从 Git、已安装的包、`InstalledVersions` 或 `COMPOSER_ROOT_VERSION` 获取它们。没有任何类、注解或 main 方法用来标记应用。如何显示由 Symfony 决定：等于 `UNKNOWN` 的版本会被隐藏，空版本保留分隔符，格式化标记遵循 Symfony 的输出规则，而 `getVersion()` 返回原样的字符串。请使用 Composer 接受、并且你希望控制台显示的值。

无论应用从源码启动还是从打包后的应用启动，这两个值都来自 `skernel build` 生成的元数据。修改 `name` 或 `version` 之后，请重新运行 `skernel build`，或者用会先构建的 `skernel run` 启动；在此之前，应用保持之前的身份。

### 在应用运行之前配置它

容器在第一次解析 `ApplicationInterface` 时创建一次应用。它设置名称和版本，安装 Symfony 的事件分发器，并在返回应用之前携带该应用同步发布 `SuperKernel\Event\BeforeApplicationStart`；它从不调用 `run()`。该事件的监听器用 Symfony 的设置方法配置应用，`super-kernel/command` 就是这样安装它的命令加载器的。监听器抛出的异常原样向外传播，应用不会被返回。[`super-kernel/event`](event.zh-CN.md) 页面展示了这样的监听器。

自行提供 `ApplicationInterface` 的应用要对其实现的行为负责；本页描述的行为都不适用于它。

### 处理命令的失败

`Application` 把到达 Symfony 错误渲染的每个异常（包括 PHP 错误）交给 `SuperKernel\Contract\ExceptionDispatcherInterface`。运行仍以非零状态结束。例外是 Symfony Console 自己的异常，它们实现 `Symfony\Component\Console\Exception\ExceptionInterface`，例如未知命令、缺少参数或未知选项：Symfony 在错误输出上渲染它们，包括消息、正在运行的命令的用法（如果有命令正在运行），以及在详细级别下的异常类和调用栈。异常处理器永远不会收到它们。

一个会失败的命令，保存为 `src/Command/ImportCommand.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Command;

use RuntimeException;
use SuperKernel\Command\Attribute\Command;
use Symfony\Component\Console\Command\Command as SymfonyCommand;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

#[Command(name: 'invoice:import', description: 'Import invoices.')]
final class ImportCommand extends SymfonyCommand
{
    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        throw new RuntimeException('The invoice feed is unavailable.');
    }
}
```

一个异常处理器，保存为 `src/ExceptionHandler/ReportFailure.php`：

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

构建项目，运行命令并打印其状态：

```bash
skernel build
php bin/bootstrap.php invoice:import; echo "status $?"
```

它会打印：

```text
Failed: The invoice feed is unavailable.
status 1
```

没有匹配的处理器时，`super-kernel/exception-dispatcher` 通过标准输出日志记录器写出完整的异常。[`super-kernel/exception-dispatcher`](exception-dispatcher.zh-CN.md) 页面介绍了处理器。

### 处理未被捕获的异常

应用启动时，本包把 `SuperKernel\Framework\ExceptionHandler\ProcessExceptionHandler` 注册为进程的 PHP 异常处理器。它以 `PHP_INT_MAX` 优先级监听 `BeforeApplicationStart`，先于所有普通监听器，并且每个进程最多注册一次。此后未被捕获的异常会交给同一个异常分发器，然后进程以状态 `1` 退出。该处理器不处理警告，不恢复执行，也不决定任何协议特定的操作。入口脚本不需要为它编写代码。

`handle()` 把一个异常交给分发器而不结束进程。

容器和异常分发器就绪之前没有异常处理：此时的失败是需要修复的启动缺陷，由 PHP 报告。

在应用创建之后失败的脚本，保存为 `bin/late-failure.php`：

```php
<?php

declare(strict_types=1);

use SuperKernel\Context\ApplicationContext;
use SuperKernel\Contract\ApplicationInterface;

require dirname(__DIR__) . '/vendor/autoload.php';

ApplicationContext::getContainer()->get(ApplicationInterface::class);

throw new RuntimeException('The invoice archive is locked.');
```

运行脚本并打印其状态：

```bash
php bin/late-failure.php; echo "status $?"
```

它会打印：

```text
Failed: The invoice archive is locked.
status 1
```

### 监听 Symfony Console 事件

容器通过 `SuperKernel\Framework\EventDispatcher\SymfonyEventDispatcher` 提供 `Symfony\Contracts\EventDispatcher\EventDispatcherInterface`，它把每个事件对象原样交给 `Psr\EventDispatcher\EventDispatcherInterface` 并返回其结果。应用使用它，因此 Symfony Console 事件（例如 `ConsoleCommandEvent`、`ConsoleErrorEvent` 和 `ConsoleTerminateEvent`）会到达带有 `#[Listener]` 的类。

Symfony 的可选字符串事件名称会被忽略：监听器按事件对象的类或接口选择，不支持按事件名称订阅。

命令事件的监听器，保存为 `src/Listener/AnnounceCommand.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Listener;

use SuperKernel\Attribute\Listener;
use SuperKernel\Contract\ListenerInterface;
use Symfony\Component\Console\Event\ConsoleCommandEvent;

#[Listener]
final readonly class AnnounceCommand implements ListenerInterface
{
    public function listen(): array
    {
        return [
            ConsoleCommandEvent::class,
        ];
    }

    public function process(object $event): void
    {
        assert($event instanceof ConsoleCommandEvent);

        $event->getOutput()->writeln('Running ' . $event->getCommand()?->getName());
    }
}
```

构建项目并再次运行命令：

```bash
skernel build
php bin/bootstrap.php invoice:import
```

它会打印：

```text
Running invoice:import
Failed: The invoice feed is unavailable.
```

### 不通过容器创建应用

`SuperKernel\Framework\Provider\ApplicationProvider` 是容器调用的工厂。持有这三个服务的显式装配边界或测试可以用 `new ApplicationProvider()($events, $exceptions, $packageManager)` 直接创建应用，参数依次是 Symfony 的 `EventDispatcherInterface`、`SuperKernel\Contract\ExceptionDispatcherInterface` 和 `SuperKernel\Contract\PackageManagerInterface`。每次调用都会创建一个新应用，并为它发布 `BeforeApplicationStart`。其他任何地方都应从容器接收应用。

## 与其他包的关系

- [`super-kernel/contract`](contract.zh-CN.md) 声明 `ApplicationInterface`、`ExceptionDispatcherInterface`、`PackageManagerInterface` 和 `ListenerInterface`；本包实现或使用它们，不依赖任何实现。
- [`super-kernel/event`](event.zh-CN.md) 声明本包发布的 `BeforeApplicationStart`。
- [`super-kernel/di`](di.zh-CN.md) 和 [`super-kernel/context`](context.zh-CN.md) 创建用来创建应用的容器；[`super-kernel/metadata`](metadata.zh-CN.md) 从构建元数据中提供根包的名称和版本。
- [`super-kernel/event-dispatcher`](event-dispatcher.zh-CN.md) 提供 Symfony 分发器契约背后的 PSR-14 分发器，并发现监听器。
- [`super-kernel/exception-dispatcher`](exception-dispatcher.zh-CN.md) 提供异常分发器及其处理器；[`super-kernel/logger`](logger.zh-CN.md) 提供它兜底使用的标准输出日志记录器。
- [`super-kernel/command`](command.zh-CN.md) 监听 `BeforeApplicationStart` 来安装它的命令加载器。
- `skernel build` 生成元数据；`skernel run` 先构建再运行应用。

## 不要这样做

**不要调用 `ApplicationProvider` 来获取应用。** 每次调用都会创建第二个应用并发布它自己的启动事件，其余代码不会共享它。请从容器接收 `ApplicationInterface`，并把直接调用限制在显式装配边界或测试中。

**不要按名称订阅 Symfony Console 事件。** 事件名称会被忽略，因此期望 `console.command` 之类名称的监听器永远不会匹配。请在 `listen()` 中列出事件类，例如 `ConsoleCommandEvent::class`。

**不要指望容器就绪之前的失败会被处理。** 异常分发器和进程处理器只有在容器创建应用之后才存在。更早的失败（例如缺少构建元数据）由 PHP 报告；请修复其原因。
