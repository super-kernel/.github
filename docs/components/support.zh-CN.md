<p align="right">
  <a href="support.md">English</a> | <strong>中文文档</strong>
</p>

# SuperKernel Support

包：[`super-kernel/support`](https://github.com/super-kernel/support) · [Packagist](https://packagist.org/packages/super-kernel/support) · [文档索引](../README.zh-CN.md)

## 用途

`super-kernel/support` 收录 SuperKernel 各个包和应用共享的便利函数。它唯一的函数 `SuperKernel\Support\make()` 创建一个由调用方持有的对象（例如为一次请求或一个任务创建的值），而其依赖仍然来自应用容器。容器本身会保留它创建的一切；`make()` 是在不保留结果的前提下进行注入式构造的方式。

本包只依赖 PHP。Composer 通过本包的自动加载配置加载该函数；安装了 `super-kernel/context` 时，由它提供容器。

## 能力

### 创建由调用方持有的对象

`make(string $className, array $parameters = []): mixed` 在每次调用时都用 `new` 创建 `$className` 的一个新实例并返回它。该实例既不从容器中获取，也不保存到容器中。它的构造函数接收：

- `$parameters` 中按参数名或按位置给出的值；
- 对于其他每个参数，参数类型中某个类或接口在容器中的条目；
- 否则取参数的默认值，然后在类型允许时取 `null`。

来自容器的依赖是容器的共享服务，因此先后创建的两个对象共享它们。`make()` 不做其他任何事：不对被请求的类应用 `#[Provider]` 或 `#[FactoryProvider]` 映射，不做属性注入，没有生命周期钩子，也不为依赖创建作用域。它每次调用都重新反射构造函数，不保留任何缓存。对于静态分析，只要 `$className` 是类字符串，返回类型就是被请求的类。

由容器提供的服务，保存为 `src/Report/Formatter.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Report;

use function strtoupper;

final class Formatter
{
    public function format(string $format, string $title): string
    {
        return strtoupper($format) . ': ' . $title;
    }
}
```

由调用方持有的对象，保存为 `src/Report/Report.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Report;

final readonly class Report
{
    public function __construct(public Formatter $formatter, public string $format = 'text')
    {
    }

    public function render(string $title): string
    {
        return $this->formatter->format($this->format, $title);
    }
}
```

创建两份报告的脚本，保存为 `bin/report.php`：

```php
<?php

declare(strict_types=1);

use Src\Report\Report;

use function SuperKernel\Support\make;

require dirname(__DIR__) . '/vendor/autoload.php';

$json = make(Report::class, ['format' => 'json']);
$text = make(Report::class);

echo $json->render('Sales'), PHP_EOL;
echo $text->render('Sales'), PHP_EOL;
var_dump($json === $text, $json->formatter === $text->formatter);
```

构建项目并运行脚本：

```bash
skernel build
php bin/report.php
```

它会打印：

```text
JSON: Sales
TEXT: Sales
bool(false)
bool(true)
```

### 了解参数如何填充

有容器时，`make()` 填充构造函数的方式与 `super-kernel/di` 完全相同，并且只向容器调用 `has()` 和 `get()`：

1. 没有构造函数的类不接收任何参数，无论 `$parameters` 中有什么。
2. 可变参数会被跳过，即使为它给出了值。引用传递的参数是错误。
3. 按名称给出的值优先于参数所在位置上给出的值。给出的 `null` 也算作一个值；PHP 在创建对象时检查其类型。
4. 参数类型中的类名、接口名和枚举名就是候选项。`self` 指被请求的类，`parent` 指它的直接父类，在类继承来的构造函数中也是如此。联合类型按顺序贡献其具名分支，并去除重复项；交集类型不贡献任何候选项，因此在 `(A&B)|C` 中只有 `C` 是候选项。`string` 或 `object` 等内置类型永远不会被查找，参数名也不会。
5. 对每个候选项都调用 `has()`。恰好有一个可用候选项时用 `get()` 获取它；多于一个时在任何 `get()` 之前报错，即使它们保存的是同一个对象，默认值也无法消除这种歧义。
6. 没有可用候选项时，使用默认值，然后在类型允许时使用 `null`；必需的 `mixed` 参数得到 `null`，没有类型的必需参数是错误。
7. `$parameters` 中多余的位置值和未知名称会被忽略。

因此，依赖必须以类型中某个类或接口的完全限定名注册。以 `mailer` 注册的条目永远不会被用于 `MailerInterface` 参数；请注册该接口，或者直接传入值。

一个接口，保存为 `src/Storage/Store.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Storage;

interface Store
{
    public function name(): string;
}
```

一个实现，保存为 `src/Storage/MemoryStore.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Storage;

final class MemoryStore implements Store
{
    public function name(): string
    {
        return 'memory';
    }
}
```

另一个实现，保存为 `src/Storage/BackupStore.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Storage;

final class BackupStore implements Store
{
    public function name(): string
    {
        return 'backup';
    }
}
```

需要一个存储的任务，保存为 `src/Storage/Job.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Storage;

final readonly class Job
{
    public function __construct(public Store $store, public string $label = 'default')
    {
    }
}
```

参数接受任一实现的类，保存为 `src/Storage/Mirror.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Storage;

final readonly class Mirror
{
    public function __construct(public MemoryStore|BackupStore $target)
    {
    }
}
```

一个最小的 PSR-11 容器，保存为 `src/Storage/ArrayContainer.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Storage;

use Psr\Container\ContainerInterface;
use RuntimeException;

final readonly class ArrayContainer implements ContainerInterface
{
    /**
     * @param array<string, mixed> $entries
     */
    public function __construct(private array $entries)
    {
    }

    public function get(string $id): mixed
    {
        return $this->entries[$id] ?? throw new RuntimeException("No entry {$id}.");
    }

    public function has(string $id): bool
    {
        return isset($this->entries[$id]);
    }
}
```

展示两种失败以及如何通过给出值来避免其中一种的脚本，保存为 `bin/errors.php`：

```php
<?php

declare(strict_types=1);

use Src\Storage\ArrayContainer;
use Src\Storage\BackupStore;
use Src\Storage\Job;
use Src\Storage\MemoryStore;
use Src\Storage\Mirror;
use SuperKernel\Context\ApplicationContext;

use function SuperKernel\Support\make;

require dirname(__DIR__) . '/vendor/autoload.php';

ApplicationContext::setContainer(new ArrayContainer([
    MemoryStore::class => new MemoryStore(),
    BackupStore::class => new BackupStore(),
]));

foreach ([Job::class, Mirror::class] as $class) {
    try {
        make($class);
    } catch (InvalidArgumentException $exception) {
        echo $exception->getMessage(), PHP_EOL;
    }
}

echo make(Mirror::class, ['target' => new BackupStore()])->target->name(), PHP_EOL;
```

运行 `skernel build` 之后，`php bin/errors.php` 会打印：

```text
Unable to resolve constructor parameter Src\Storage\Job::__construct($store) of type Src\Storage\Store.
Ambiguous constructor parameter Src\Storage\Mirror::__construct($target): Src\Storage\MemoryStore, Src\Storage\BackupStore.
backup
```

### 使用其他容器

`make()` 可以与 `SuperKernel\Context\ApplicationContext` 持有的任何 PSR-11 容器一起工作；它不需要 `super-kernel/di` 的任何服务。要使用其他容器，请按该实现的要求创建并配置它，然后在第一次调用 `make()` 之前把它传给 `ApplicationContext::setContainer()`。没有设置容器时，`ApplicationContext::getContainer()` 用 `super-kernel/di` 和构建元数据创建默认容器；没有安装 `super-kernel/di` 时，抛出 `SuperKernel\Context\Exception\ContainerNotSetException`。

使用上面那个容器的脚本，保存为 `bin/third-party.php`：

```php
<?php

declare(strict_types=1);

use Src\Storage\ArrayContainer;
use Src\Storage\Job;
use Src\Storage\MemoryStore;
use Src\Storage\Store;
use SuperKernel\Context\ApplicationContext;

use function SuperKernel\Support\make;

require dirname(__DIR__) . '/vendor/autoload.php';

ApplicationContext::setContainer(new ArrayContainer([Store::class => new MemoryStore()]));

$first = make(Job::class, ['label' => 'first']);
$second = make(Job::class, ['label' => 'second']);

echo $first->label, ' and ', $second->label, ' use ', $first->store->name(), PHP_EOL;
var_dump($first->store === $second->store);
```

`php bin/third-party.php` 会打印：

```text
first and second use memory
bool(true)
```

### 不依赖 SuperKernel 使用

当 `SuperKernel\Context\ApplicationContext` 类不存在时，`make()` 只是按 PHP 自己的参数规则执行 `new $className(...$parameters)`：上述容器规则不适用，因此可变参数的值和多余的键会按 PHP 的方式传给构造函数。函数文件 `src/functions.php` 可以被单独引入。

只加载该函数的脚本，保存为 `bin/native.php`：

```php
<?php

declare(strict_types=1);

use Src\Report\Formatter;
use Src\Report\Report;

use function SuperKernel\Support\make;

require dirname(__DIR__) . '/vendor/super-kernel/support/src/functions.php';
require dirname(__DIR__) . '/src/Report/Formatter.php';
require dirname(__DIR__) . '/src/Report/Report.php';

echo make(Report::class, [new Formatter(), 'csv'])->render('Sales'), PHP_EOL;
```

`php bin/native.php` 会打印 `CSV: Sales`。

## 错误

对于它检测到的失败，`make()` 抛出 PHP 的 `InvalidArgumentException`。它从不捕获或重试：`super-kernel/context`、容器的 `has()` 和 `get()`、反射以及构造函数抛出的异常都会原样传出，查找失败也从不退回到默认值、`null`、其他候选项或普通构造。抽象类或错误的参数类型仍然是 PHP 错误。

| 情形 | 消息 |
| --- | --- |
| 某个参数没有值、条目、默认值或 `null` 可用 | `Unable to resolve constructor parameter <class>::__construct($<parameter>) of type <type>.`；参数没有类型时不含 ` of type <type>` |
| 多个候选项存在条目 | `Ambiguous constructor parameter <class>::__construct($<parameter>): <id>, <id>.` |
| 引用传递的参数 | `Constructor parameter <class>::__construct($<parameter>) cannot be resolved because references are unsupported.` |

## 与其他包的关系

- [`super-kernel/context`](context.zh-CN.md) 提供容器；只有安装了该包，`make()` 才会使用它。
- [`super-kernel/di`](di.zh-CN.md) 对它调用的构造函数应用相同的参数规则；它自己创建不保留的对象的方式见 [解析调用方持有的对象](di.zh-CN.md#解析调用方持有的对象)。`super-kernel/di` 抛出 `ResolutionException` 的地方，`make()` 抛出 `InvalidArgumentException`。
- 其他包，例如 [`super-kernel/config`](config.zh-CN.md) 和 [`super-kernel/http-server`](http-server.zh-CN.md)，用 `make()` 创建除注入的依赖外还接收自己参数的对象。

## 不要这样做

**不要用 `make()` 获取共享服务。** 每次调用都会创建另一个实例，其他代码看不到它。请通过注入或从容器接收共享服务。

**不要以自己起的名称注册依赖。** `make()` 只查找参数类型中的类名和接口名，因此以 `mailer` 注册的条目不会被使用，该参数会以 `Unable to resolve constructor parameter ...` 失败。请以接口注册该依赖，或者在 `$parameters` 中传入它。

**不要让联合类型的两个候选项都留在容器中。** `make()` 拒绝做出选择，抛出 `Ambiguous constructor parameter ...`，即使两个条目是同一个对象。请显式传入值，或者收窄类型。
