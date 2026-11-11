<p align="right">
  <a href="event.md">English</a> | <strong>中文文档</strong>
</p>

# SuperKernel Event

包：[`super-kernel/event`](https://github.com/super-kernel/event) · [Packagist](https://packagist.org/packages/super-kernel/event) · [文档索引](../README.zh-CN.md)

## 用途

`super-kernel/event` 声明 SuperKernel 向其他代码通告的应用生命周期中的两个时刻：应用的启动和项目的构建。各个包和应用监听这些事件，以便在应用运行之前配置它，或者在构建时写出打包后的应用直接加载、无需再生成的文件。

本包只包含事件类。由谁发布事件、何时发布、监听器失败时如何处理，都由发布事件的包负责；监听器的注册和分发由 `super-kernel/event-dispatcher` 负责。

## 能力

### 在应用运行之前配置它

`SuperKernel\Event\BeforeApplicationStart` 由 `super-kernel/framework` 在其 `ApplicationProvider` 创建应用时发布，即容器第一次解析 `SuperKernel\Contract\ApplicationInterface` 时。此时应用已经具有根包的名称和声明的版本，以及它的 Symfony 事件调度器。事件是同步发布的，发生在提供者返回应用之前；在应用的调用方调用 `run()` 之前，没有任何代码会调用它。

`application()` 返回被创建的应用。它的类型是 `SuperKernel\Contract\ApplicationInterface`；使用 `super-kernel/framework` 时，该对象是一个 `Symfony\Component\Console\Application`，因此监听器在使用 Symfony 的设置方法之前先检查这个类。`super-kernel/command` 就是这样安装它的命令加载器的。

在由 `super-kernel/skeleton` 创建的应用中，下面的监听器会为控制台应用改名。将它保存为 `src/Listener/NameTheApplication.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Listener;

use SuperKernel\Attribute\Listener;
use SuperKernel\Contract\ListenerInterface;
use SuperKernel\Event\BeforeApplicationStart;
use Symfony\Component\Console\Application;

#[Listener]
final readonly class NameTheApplication implements ListenerInterface
{
    public function listen(): array
    {
        return [
            BeforeApplicationStart::class,
        ];
    }

    public function process(object $event): void
    {
        assert($event instanceof BeforeApplicationStart);

        $application = $event->application();

        if ($application instanceof Application) {
            $application->setName('Billing');
        }
    }
}
```

构建项目并运行应用：

```bash
skernel build
php bin/bootstrap.php --version
```

输出以 `Billing` 开头。抛出异常的监听器会中止启动：容器对 `ApplicationInterface` 的查找以 `SuperKernel\Di\Exception\ResolutionException` 失败，其前一个异常（previous）就是监听器抛出的异常；应用既不会被返回，也不会运行。

### 写出构建产物

`SuperKernel\Event\ApplicationBuild` 由 `skernel build` 发布。生成项目的元数据之后，`skernel build` 在项目中启动第二个 PHP 进程，用这些元数据创建容器，并通过 `Psr\EventDispatcher\EventDispatcherInterface` 发布一次该事件。它不会创建或运行应用，因此监听器不能假定 `ApplicationInterface` 已被创建。该事件不携带载荷。

监听器写出打包后的应用（例如由 `skernel package` 构建的应用）直接加载、无需在运行时生成的文件。`super-kernel/aop` 就是这样写出它预先准备好的代理类的。`skernel build` 会转发该进程打印的内容；监听器抛出的异常会使该进程以非零状态退出，从而使构建失败。

当项目没有安装 `super-kernel/event`，或者其容器中没有 `Psr\EventDispatcher\EventDispatcherInterface` 时，`skernel build` 会跳过该事件。

下面的监听器记录项目的构建时间。将它保存为 `src/Listener/WriteBuildInfo.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Listener;

use SuperKernel\Attribute\Listener;
use SuperKernel\Contract\BasePathInterface;
use SuperKernel\Contract\ListenerInterface;
use SuperKernel\Event\ApplicationBuild;

use function date;
use function file_put_contents;

#[Listener]
final readonly class WriteBuildInfo implements ListenerInterface
{
    public function __construct(private BasePathInterface $basePath)
    {
    }

    public function listen(): array
    {
        return [
            ApplicationBuild::class,
        ];
    }

    public function process(object $event): void
    {
        file_put_contents($this->basePath . '/build-info.txt', 'Built at ' . date(DATE_ATOM) . PHP_EOL);

        echo 'Wrote build-info.txt', PHP_EOL;
    }
}
```

运行构建：

```bash
skernel build
```

构建会打印 `Wrote build-info.txt`，项目目录中的 `build-info.txt` 保存了构建时间。

### 监听所有 SuperKernel 事件

这两个事件都是实现了 `SuperKernel\Contract\EventInterface` 的 final readonly 类，该接口是一个没有方法的标记接口。`super-kernel/event-dispatcher` 用 `instanceof` 匹配监听器 `listen()` 返回的条目，因此返回 `EventInterface::class` 的监听器会收到这两个事件。

### 声明应用自己的事件

本包不声明其他生命周期边界，例如启动之后或停止时。应用为自己需要的时刻声明自己的事件类，并通过 `Psr\EventDispatcher\EventDispatcherInterface` 发布它们。`super-kernel/event-dispatcher` 可以分发任何对象，因此应用的事件不需要基类或接口。

事件，保存为 `src/Billing/InvoicePaid.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Billing;

final readonly class InvoicePaid
{
    public function __construct(public string $invoiceNumber)
    {
    }
}
```

监听器，保存为 `src/Billing/SendReceipt.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Billing;

use SuperKernel\Attribute\Listener;
use SuperKernel\Contract\ListenerInterface;

#[Listener]
final readonly class SendReceipt implements ListenerInterface
{
    public function listen(): array
    {
        return [
            InvoicePaid::class,
        ];
    }

    public function process(object $event): void
    {
        assert($event instanceof InvoicePaid);

        echo "Receipt sent for invoice {$event->invoiceNumber}", PHP_EOL;
    }
}
```

发布该事件的服务，保存为 `src/Billing/PaymentService.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Billing;

use Psr\EventDispatcher\EventDispatcherInterface;

final readonly class PaymentService
{
    public function __construct(private EventDispatcherInterface $events)
    {
    }

    public function pay(string $invoiceNumber): void
    {
        $this->events->dispatch(new InvoicePaid($invoiceNumber));
    }
}
```

使用该服务的脚本，保存为 `bin/pay.php`：

```php
<?php

declare(strict_types=1);

use Src\Billing\PaymentService;
use SuperKernel\Context\ApplicationContext;

require dirname(__DIR__) . '/vendor/autoload.php';

ApplicationContext::getContainer()->get(PaymentService::class)->pay('INV-1001');
```

构建项目并运行脚本：

```bash
skernel build
php bin/pay.php
```

它会打印 `Receipt sent for invoice INV-1001`。

## 与其他包的关系

- [`super-kernel/contract`](contract.zh-CN.md) 声明了 `EventInterface`、`ApplicationInterface` 和 `ListenerInterface`；本包只依赖它。
- [`super-kernel/event-dispatcher`](event-dispatcher.zh-CN.md) 发现带有 [`super-kernel/attribute`](attribute.zh-CN.md) 中 `#[Listener]` 注解的类，并把事件分发给它们。单独安装本包不会选定任何分发器。
- [`super-kernel/framework`](framework.zh-CN.md) 发布 `BeforeApplicationStart`。[`super-kernel/command`](command.zh-CN.md) 和 `super-kernel/framework` 自身监听它。
- `skernel build` 发布 `ApplicationBuild`。[`super-kernel/aop`](aop.zh-CN.md) 监听它来准备代理。
- 监听器是从 `skernel build` 生成、由 [`super-kernel/di`](di.zh-CN.md) 恢复的元数据中找到的：添加、删除或修改监听器之后，请重新运行 `skernel build`，或者用会先构建的 `skernel run` 启动。
