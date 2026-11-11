<p align="right">
  <a href="event-dispatcher.md">English</a> | <strong>中文文档</strong>
</p>

# SuperKernel Event Dispatcher

包：[`super-kernel/event-dispatcher`](https://github.com/super-kernel/event-dispatcher) · [Packagist](https://packagist.org/packages/super-kernel/event-dispatcher) · [文档索引](../README.zh-CN.md)

## 用途

`super-kernel/event-dispatcher` 让代码宣告某件事已经发生，而无需知道谁会做出反应。代码通过 PSR-14 的 `EventDispatcherInterface` 发布一个事件对象；本包按优先级顺序同步调用声明了该事件类的每个监听器，并交回同一个对象。

本包只实现 PSR-14，不依赖 Symfony。有哪些事件由发布它们的代码决定，例如生命周期事件由 `super-kernel/event` 声明；创建监听器的容器属于 `super-kernel/di`。

## 能力

### 声明监听器

监听器是一个带有 `SuperKernel\Attribute\Listener` 并实现 `SuperKernel\Contract\ListenerInterface` 的类：

- `listen()` 返回监听器接收的事件类或接口的列表；每个名称都必须存在；
- `process(object $event): void` 处理一个事件；其返回值会被忽略。

监听器只接收与 `listen()` 中某一项匹配的事件，因此只监听一个事件的监听器可以直接处理它，并在 PHPDoc 中写明具体类型。`#[Listener]` 接受一个 `priority`，默认为 `0`：优先级高的先运行，优先级相同的保持构建发现监听器的顺序。

监听器由容器创建，因此监听器通过构造函数接收服务。被注解的类本身不必可实例化：只要容器为接口或抽象类提供了实现，它们也可以使用，而容器返回的对象必须实现 `ListenerInterface`。

事件，保存为 `src/Account/UserRegistered.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Account;

final readonly class UserRegistered
{
    public function __construct(public string $email)
    {
    }
}
```

优先级较高的监听器，保存为 `src/Account/RecordAuditEntry.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Account;

use SuperKernel\Attribute\Listener;
use SuperKernel\Contract\ListenerInterface;

#[Listener(priority: 100)]
final readonly class RecordAuditEntry implements ListenerInterface
{
    public function listen(): array
    {
        return [
            UserRegistered::class,
        ];
    }

    /**
     * @param UserRegistered $event
     */
    public function process(object $event): void
    {
        echo "Audit: {$event->email} registered", PHP_EOL;
    }
}
```

使用默认优先级的监听器，保存为 `src/Account/SendWelcomeMail.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Account;

use SuperKernel\Attribute\Listener;
use SuperKernel\Contract\ListenerInterface;

#[Listener]
final readonly class SendWelcomeMail implements ListenerInterface
{
    public function listen(): array
    {
        return [
            UserRegistered::class,
        ];
    }

    /**
     * @param UserRegistered $event
     */
    public function process(object $event): void
    {
        echo "Welcome mail sent to {$event->email}", PHP_EOL;
    }
}
```

发布该事件的脚本，保存为 `bin/register.php`：

```php
<?php

declare(strict_types=1);

use Psr\EventDispatcher\EventDispatcherInterface;
use Src\Account\UserRegistered;
use SuperKernel\Context\ApplicationContext;

require dirname(__DIR__) . '/vendor/autoload.php';

$events = ApplicationContext::getContainer()->get(EventDispatcherInterface::class);

$events->dispatch(new UserRegistered('ada@example.com'));
```

构建项目并运行脚本：

```bash
skernel build
php bin/register.php
```

它会打印：

```text
Audit: ada@example.com registered
Welcome mail sent to ada@example.com
```

### 监听多个事件

监听器可以列出多个事件，列出的类或接口也会匹配其子类和实现，因此 `SuperKernel\Contract\EventInterface` 的监听器会收到每个 SuperKernel 生命周期事件。即使有多项匹配，监听器对每个事件也只被调用一次。列出多个事件时，由 `process()` 区分它们，例如使用 `instanceof` 或 `match`。

监听两个事件的监听器，保存为 `src/Audit/RecordEverything.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Audit;

use Src\Account\UserRegistered;
use Src\Order\OrderPlaced;
use SuperKernel\Attribute\Listener;
use SuperKernel\Contract\ListenerInterface;

#[Listener(priority: 200)]
final readonly class RecordEverything implements ListenerInterface
{
    public function listen(): array
    {
        return [
            UserRegistered::class,
            OrderPlaced::class,
        ];
    }

    public function process(object $event): void
    {
        $subject = match (true) {
            $event instanceof UserRegistered => "user {$event->email}",
            $event instanceof OrderPlaced => "order of {$event->total}",
        };

        echo "Recorded {$subject}", PHP_EOL;
    }
}
```

构建项目并再次运行脚本：

```bash
skernel build
php bin/register.php
```

它会打印：

```text
Recorded user ada@example.com
Audit: ada@example.com registered
Welcome mail sent to ada@example.com
```

### 分发事件

通过构造函数接收 `Psr\EventDispatcher\EventDispatcherInterface`，或者像这里的脚本那样在入口点从容器中查找它。`dispatch()` 接受任何对象：事件类不需要基类，也不必实现 `EventInterface`。它在调用方进程中逐个调用匹配的监听器，然后返回同一个事件对象，因此监听器可以把结果记录在事件上供发布者读取。

监听器抛出的异常原样从 `dispatch()` 向外传播，其余的监听器不会被调用。无论按接口还是按类查找，容器都共享同一个分发器。

### 停止传播

实现 `Psr\EventDispatcher\StoppableEventInterface` 的事件控制自己传播多远。分发器在第一个监听器之前以及每个监听器之后询问 `isPropagationStopped()`；一旦它返回 `true`，就不再调用后续监听器，`dispatch()` 返回该事件。

可停止的事件，保存为 `src/Order/OrderPlaced.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Order;

use Psr\EventDispatcher\StoppableEventInterface;

final class OrderPlaced implements StoppableEventInterface
{
    private bool $rejected = false;

    public function __construct(public readonly int $total)
    {
    }

    public function reject(): void
    {
        $this->rejected = true;
    }

    public function isRejected(): bool
    {
        return $this->rejected;
    }

    public function isPropagationStopped(): bool
    {
        return $this->rejected;
    }
}
```

停止它的监听器，保存为 `src/Order/CheckCreditLimit.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Order;

use SuperKernel\Attribute\Listener;
use SuperKernel\Contract\ListenerInterface;

#[Listener(priority: 10)]
final readonly class CheckCreditLimit implements ListenerInterface
{
    public function listen(): array
    {
        return [
            OrderPlaced::class,
        ];
    }

    /**
     * @param OrderPlaced $event
     */
    public function process(object $event): void
    {
        if ($event->total > 1000) {
            echo "Order of {$event->total} rejected", PHP_EOL;
            $event->reject();
        }
    }
}
```

优先级较低的监听器，保存为 `src/Order/ShipOrder.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Order;

use SuperKernel\Attribute\Listener;
use SuperKernel\Contract\ListenerInterface;

#[Listener]
final readonly class ShipOrder implements ListenerInterface
{
    public function listen(): array
    {
        return [
            OrderPlaced::class,
        ];
    }

    /**
     * @param OrderPlaced $event
     */
    public function process(object $event): void
    {
        echo "Order of {$event->total} shipped", PHP_EOL;
    }
}
```

下两个订单的脚本，保存为 `bin/order.php`：

```php
<?php

declare(strict_types=1);

use Psr\EventDispatcher\EventDispatcherInterface;
use Src\Order\OrderPlaced;
use SuperKernel\Context\ApplicationContext;

require dirname(__DIR__) . '/vendor/autoload.php';

$events = ApplicationContext::getContainer()->get(EventDispatcherInterface::class);

foreach ([250, 4800] as $total) {
    $order = $events->dispatch(new OrderPlaced($total));

    echo $order->isRejected() ? 'rejected' : 'accepted', PHP_EOL;
}
```

构建项目并运行脚本：

```bash
skernel build
php bin/order.php
```

它会打印：

```text
Recorded order of 250
Order of 250 shipped
accepted
Recorded order of 4800
Order of 4800 rejected
rejected
```

### 读取某个事件的监听器

容器还提供 `Psr\EventDispatcher\ListenerProviderInterface`。`getListenersForEvent()` 以 `[$listener, 'process']` 可调用值的形式，按分发器调用它们的顺序返回与事件匹配的监听器。它按事件类记住结果，不保留任何事件对象。

列出订单监听器的脚本，保存为 `bin/listeners.php`：

```php
<?php

declare(strict_types=1);

use Psr\EventDispatcher\ListenerProviderInterface;
use Src\Order\OrderPlaced;
use SuperKernel\Context\ApplicationContext;

require dirname(__DIR__) . '/vendor/autoload.php';

$provider = ApplicationContext::getContainer()->get(ListenerProviderInterface::class);

foreach ($provider->getListenersForEvent(new OrderPlaced(0)) as [$listener]) {
    echo $listener::class, PHP_EOL;
}
```

运行脚本：

```bash
php bin/listeners.php
```

它会打印：

```text
Src\Audit\RecordEverything
Src\Order\CheckCreditLimit
Src\Order\ShipOrder
```

## 错误

监听器提供者在容器第一次需要它时（也就是第一次查找分发器时）创建所有监听器。无效的监听器会在那时抛出 `InvalidArgumentException`，并被包装在一个 `SuperKernel\Di\Exception\ResolutionException` 中，其消息为 `Unable to invoke factory "SuperKernel\EventDispatcher\Provider\ListenerProviderProvider" for container entry "Psr\EventDispatcher\ListenerProviderInterface".` 分发 `ApplicationBuild` 的 `skernel build` 中也会发生同样的情况，因此构建失败。

| 情形 | 消息 |
| --- | --- |
| 为 `#[Listener]` 类创建的对象没有实现 `ListenerInterface` | `Resolved listener <class> must implement SuperKernel\Contract\ListenerInterface, <type> provided.` |
| `listen()` 返回的数组不是列表，或者包含不是已存在的类或接口的名称 | `<class>::listen() must return a list containing only existing event class or interface names.` |

容器无法创建的监听器会在同一时刻抛出容器自己的异常。

## 与其他包的关系

- [`super-kernel/attribute`](attribute.zh-CN.md) 声明 `#[Listener]`，[`super-kernel/contract`](contract.zh-CN.md) 声明 `ListenerInterface` 和 `EventInterface`；本包赋予它们含义。
- [`super-kernel/di`](di.zh-CN.md) 创建监听器并共享分发器；[`super-kernel/annotator`](annotator.zh-CN.md) 提供记录下来的 `#[Listener]` 声明。
- [`super-kernel/event`](event.zh-CN.md) 声明生命周期事件；[`super-kernel/framework`](framework.zh-CN.md) 发布 `BeforeApplicationStart`，并把 Symfony 的分发器契约适配到本分发器，使 Symfony Console 事件到达同样的监听器。
- `skernel build` 收集监听器，并通过本分发器发布 `ApplicationBuild`；`skernel run` 在启动应用之前先构建。

## 不要这样做

**不要在项目中留下无效的监听器。** 所有监听器是一起创建的，因此一个没有实现 `ListenerInterface`、列出了不存在的类，或者容器无法创建的 `#[Listener]` 类会使每次分发都失败（而不只是它自己的事件），并使 `skernel build` 失败。请修复或删除该声明；需要保持静默的监听器可以从 `listen()` 返回空列表。
