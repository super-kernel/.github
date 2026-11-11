<p align="right">
  <a href="context.md">English</a> | <strong>中文文档</strong>
</p>

# SuperKernel Context

包：[`super-kernel/context`](https://github.com/super-kernel/context) · [Packagist](https://packagist.org/packages/super-kernel/context) · [文档索引](../README.zh-CN.md)

## 用途

`super-kernel/context` 回答运行中的应用的两个彼此独立的问题。`ApplicationContext` 回答“这个进程使用哪个容器？”：入口点（例如 `bin/bootstrap.php` 或服务器回调）从它获取容器。`Context` 回答“哪些值属于当前正在运行的工作？”：基础设施代码把短期值（例如请求标识符）交给同一协程中更下层的代码，而不必在每次调用中传递它们。

本包只依赖 PHP 和 `psr/container`。`super-kernel/di` 和 Swoole 扩展是可选的；有了它们，容器会被自动创建，值会按协程和线程分别保存。

## 能力

### 为当前执行保存值

`SuperKernel\Context\Context` 提供四个静态方法：

- `set($key, $value)` 存储一个值并返回它；
- `get($key, $default = null)` 返回该值，键不存在时返回 `$default`；
- `has($key)` 报告键是否存在，即使它保存的是 `null`；
- `clear($key)` 移除一个键，`clear()` 移除当前作用域的所有值。

`Context` 不需要容器即可工作：它从不访问 `ApplicationContext`，也从不解析服务。

每次调用都会按以下顺序重新选择作用域：

1. 当前的 Swoole 协程，前提是 Swoole 已加载并且有协程正在运行；
2. 当前的 Swoole 线程，前提是代码运行在 Swoole 线程中；
3. PHP 进程。

不可用的作用域会被跳过，因此进程总是最后的选择。原生 Fiber 和 Swow 协程没有自己的作用域：在其中，值像其他代码一样进入 Swoole 协程或进程。

使用进程作用域的脚本，保存为 `bin/context.php`：

```php
<?php

declare(strict_types=1);

use SuperKernel\Context\Context;

require dirname(__DIR__) . '/vendor/autoload.php';

Context::set('request-id', 'req-1');
Context::set('user', null);

var_dump(Context::get('request-id'));
var_dump(Context::has('user'), Context::get('locale', 'en'));

Context::clear();

var_dump(Context::has('request-id'));
```

运行脚本：

```bash
php bin/context.php
```

它会打印：

```text
string(5) "req-1"
bool(true)
string(2) "en"
bool(false)
```

### 按协程和线程分开保存值

有 Swoole 时，每个协程都有自己的值，保存在 Swoole 原生的协程上下文中，并在协程结束时释放。子协程看不到父协程的值。Swoole 线程中的值按线程保存，在进程的整个生命周期内保留。在任何协程或线程之外存储的值属于进程，会一直保留到你清除它们为止。

包含两个协程的脚本，保存为 `bin/coroutines.php`：

```php
<?php

declare(strict_types=1);

use Swoole\Coroutine;
use SuperKernel\Context\Context;

use function Swoole\Coroutine\run;

require dirname(__DIR__) . '/vendor/autoload.php';

Context::set('request-id', 'process');

run(static function (): void {
    foreach (['req-A', 'req-B'] as $requestId) {
        Coroutine::create(static function () use ($requestId): void {
            Context::set('request-id', $requestId);
            Coroutine::sleep(0.01);

            echo 'Coroutine sees ', Context::get('request-id'), PHP_EOL;
        });
    }
});

echo 'Process sees ', Context::get('request-id'), PHP_EOL;
```

加载了 Swoole 扩展时，`php bin/coroutines.php` 会打印：

```text
Coroutine sees req-A
Coroutine sees req-B
Process sees process
```

`SuperKernel\Context\Context\CoroutineContext`、`ThreadContext` 和 `ProcessContext` 就是这三个作用域，它们有相同的静态方法，以及一个报告该作用域是否可用的 `support()` 方法。请使用会替你做出选择的 `Context`；在作用域不可用的地方直接调用它会抛出 `LogicException`，例如 `The Swoole coroutine context requires an active coroutine.`。

### 获取应用容器

`SuperKernel\Context\ApplicationContext::getContainer()` 返回进程的容器：

- 已经设置过容器时，返回该容器。
- 否则，在安装了 `super-kernel/di` 时，它调用一次 `SuperKernel\Di\Provider\ContainerProvider`，由它恢复 `skernel build` 的元数据；结果被保存为进程的容器并返回。
- 否则抛出 `SuperKernel\Context\Exception\ContainerNotSetException`。

创建容器失败时，它抛出 `SuperKernel\Context\Exception\ContainerInitializationException`，以失败原因作为其前一个异常（previous），并且不保留任何内容：之后的调用会再次尝试，`setContainer()` 也仍然可以设置容器。

容器属于进程，而不属于协程。服务器的每个工作进程都有自己的容器。

项目中的一个类，保存为 `src/Billing/Invoice.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Billing;

final class Invoice
{
}
```

两次获取容器的脚本，保存为 `bin/container.php`：

```php
<?php

declare(strict_types=1);

use Src\Billing\Invoice;
use SuperKernel\Context\ApplicationContext;

require dirname(__DIR__) . '/vendor/autoload.php';

$container = ApplicationContext::getContainer();

var_dump($container === ApplicationContext::getContainer());
var_dump($container->get(Invoice::class) === $container->get(Invoice::class));
```

构建项目并运行脚本：

```bash
skernel build
php bin/container.php
```

它会打印：

```text
bool(true)
bool(true)
```

### 使用其他容器

`ApplicationContext::setContainer($container)` 把任意 `Psr\Container\ContainerInterface` 设置为进程的容器，替换之前的容器，并返回它。请在启动期间、任何代码请求容器之前设置它。你设置的容器对于类或接口标识符必须返回该类型的实例，因为 SuperKernel 的各个包依赖这一点。

使用自有容器的脚本，保存为 `bin/own-container.php`：

```php
<?php

declare(strict_types=1);

use Psr\Container\ContainerInterface;
use Psr\Container\NotFoundExceptionInterface;
use SuperKernel\Context\ApplicationContext;

require dirname(__DIR__) . '/vendor/autoload.php';

$container = new class implements ContainerInterface {
    public function get(string $id): mixed
    {
        return 'app.name' === $id
            ? 'Billing'
            : throw new class("No entry {$id}.") extends RuntimeException implements NotFoundExceptionInterface {};
    }

    public function has(string $id): bool
    {
        return 'app.name' === $id;
    }
};

var_dump(ApplicationContext::setContainer($container) === $container);
echo ApplicationContext::getContainer()->get('app.name'), PHP_EOL;
```

`php bin/own-container.php` 会打印：

```text
bool(true)
Billing
```

## 错误

| 情形 | 消息 |
| --- | --- |
| 没有设置容器且没有安装 `super-kernel/di`（`ContainerNotSetException`） | `The application context has no container.` |
| 创建默认容器失败（`ContainerInitializationException`，失败原因作为前一个异常） | `Unable to initialize the application container through SuperKernel\Di\Provider\ContainerProvider.` |
| 在协程之外使用 `CoroutineContext`（`LogicException`） | `The Swoole coroutine context requires an active coroutine.` |
| 在没有 Swoole 时使用 `CoroutineContext`（`LogicException`） | `The Swoole coroutine context requires the Swoole extension.` |
| 在 Swoole 线程之外使用 `ThreadContext`（`LogicException`） | `The Swoole thread context requires an active Swoole thread.` |

## 与其他包的关系

- [`super-kernel/di`](di.zh-CN.md) 提供 `ContainerProvider`，`getContainer()` 调用它，用 `skernel build` 的元数据创建默认容器。
- 骨架项目的 `bin/bootstrap.php` 从 `ApplicationContext` 获取容器，并运行 [`super-kernel/framework`](framework.zh-CN.md) 的应用。
- [`super-kernel/support`](support.zh-CN.md) 在 `make()` 中使用 `ApplicationContext` 的容器。
- [`super-kernel/swoole`](swoole.zh-CN.md) 和各个服务器包运行协程和线程，`Context` 为它们分开保存值。

## 不要这样做

**不要在你的服务内部从 `ApplicationContext` 查找服务。** 调用 `ApplicationContext::getContainer()->get()` 的类会隐藏它的依赖，在测试中也无法与其他对象一起创建。请通过构造函数或 `#[Autowired]` 接收依赖；只在入口点（例如脚本或服务器回调）调用 `getContainer()`。

**不要创建第二个根容器。** 在 `ApplicationContext` 之外调用 `new ContainerProvider()()` 会恢复另一个带有自己服务的容器，应用的其余部分并不共享它。请使用 `ApplicationContext::getContainer()`。

**不要在启动之后替换容器。** `setContainer()` 只为之后的调用方更改容器；已经创建的服务保留它们从旧容器得到的东西。请在任何代码请求容器之前设置你的容器。

**不要在长时间运行的服务器中把值留在进程作用域。** 在协程之外，值会在进程的整个生命周期内保留，因此处理一个请求时存储的值在下一个请求中依然存在。请在请求所在的协程中存储请求的值，或者在工作结束时 `clear()` 它们。

**不要期望 Fiber 拥有自己的值。** Fiber 存储的值会落入外围的协程或进程，并替换 Fiber 之外的代码存储的值。请在 Swoole 协程中运行彼此独立的工作，或者使用不同的键。

**不要用 `Context` 代替注入。** `Context` 中的值没有类型，也不出现在签名中。请只把它用于基础设施在一次执行中向下传递的短期值（例如请求标识符），服务和配置则通过注入传递。
