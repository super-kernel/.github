<p align="right">
  <a href="attribute.md">English</a> | <strong>中文文档</strong>
</p>

# SuperKernel Attribute

包：[`super-kernel/attribute`](https://github.com/super-kernel/attribute) · [Packagist](https://packagist.org/packages/super-kernel/attribute) · [文档索引](../README.zh-CN.md)

## 用途

`super-kernel/attribute` 收录 SuperKernel 各个包、第三方包以及其他框架实现共享的原生 PHP 注解。每个注解确定一种声明的形式和它所表达的含义：它可以放在哪里、能否重复、有哪些参数，以及哪些值有效。类用这些注解声明自己提供什么，由使用它们的包据此执行。

本包只包含声明，除 PHP 外不依赖任何东西。`skernel build` 收集这些注解，`super-kernel/annotator` 存储并恢复它们，而使用它们的包（例如 `super-kernel/di`、`super-kernel/config` 和 `super-kernel/event-dispatcher`）解释它们。所有注解都位于 `SuperKernel\Attribute` 命名空间中。

## 能力

### 声明服务

- 类上的 `#[Provider(Target::class)]` 把标识符 `Target` 映射到该类，随后由 `super-kernel/di` 构造它。类可以映射到自身，即 `#[Provider(OwnClass::class)]`，从而作为共享服务被构造。该注解不可重复。
- 类上的 `#[FactoryProvider(Target::class)]` 声明由该类公开的非静态 `__invoke()` 创建 `Target`。该类可以没有构造函数，或者只有公开的构造函数；容器会填充两者的参数。该注解可以重复：每次声明都把另一个标识符映射到同一个类，按声明顺序排列，容器为每个标识符分别保留一个产物。
- 这两个注解的标识符都必须是不含 NUL 字节的非空字符串。类是否与标识符兼容、一个标识符是否有两个提供者，由使用它们的容器检查。

接口，保存为 `src/Clock/Clock.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Clock;

interface Clock
{
    public function zone(): string;
}
```

实现，保存为 `src/Clock/ZonedClock.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Clock;

final readonly class ZonedClock implements Clock
{
    public function __construct(private string $zone)
    {
    }

    public function zone(): string
    {
        return $this->zone;
    }
}
```

同时服务于两个标识符的工厂，保存为 `src/Clock/ClockFactory.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Clock;

use SuperKernel\Attribute\FactoryProvider;

use function getenv;

#[FactoryProvider(Clock::class)]
#[FactoryProvider(ZonedClock::class)]
final class ClockFactory
{
    public function __invoke(): ZonedClock
    {
        return new ZonedClock(getenv('APP_TIMEZONE') ?: 'UTC');
    }
}
```

查找这两个标识符的脚本，保存为 `bin/clock.php`：

```php
<?php

declare(strict_types=1);

use Src\Clock\Clock;
use Src\Clock\ZonedClock;
use SuperKernel\Context\ApplicationContext;

require dirname(__DIR__) . '/vendor/autoload.php';

$container = ApplicationContext::getContainer();

echo $container->get(Clock::class)->zone(), PHP_EOL;
var_dump($container->get(Clock::class) === $container->get(ZonedClock::class));
```

构建项目并运行脚本：

```bash
skernel build
APP_TIMEZONE=Asia/Shanghai php bin/clock.php
```

它会打印以下内容；最后一行表明每个标识符都有自己的产物：

```text
Asia/Shanghai
bool(false)
```

`super-kernel/di` 对这两个注解的规则见其页面：[声明服务](di.zh-CN.md#声明服务)。

### 注入属性

属性上的 `#[Autowired]` 要求容器在构造函数返回之后为它赋值。

- 不带参数时，由属性的类型选择条目。`#[Autowired(Target::class)]` 则指定要注入的类或接口；它必须存在，否则构造该注解时失败，消息为 `Injectable identifier <identifier> does not exist.`。
- `#[Autowired]` 没有其他参数。属性是否得到延迟代理由容器决定；`super-kernel/di` 默认注入延迟代理。
- 该注解也可以放在方法上，`super-kernel/di` 把它仅视为元数据，从不调用该方法。

带有注入属性的类，保存为 `src/Clock/Schedule.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Clock;

use SuperKernel\Attribute\Autowired;

final class Schedule
{
    #[Autowired(ZonedClock::class)]
    public Clock $clock;
}
```

使用它的脚本，保存为 `bin/schedule.php`：

```php
<?php

declare(strict_types=1);

use Src\Clock\Schedule;
use SuperKernel\Context\ApplicationContext;

require dirname(__DIR__) . '/vendor/autoload.php';

echo ApplicationContext::getContainer()->get(Schedule::class)->clock->zone(), PHP_EOL;
```

运行 `skernel build` 之后，`php bin/schedule.php` 会打印 `UTC`。

注入顺序和延迟代理见 DI 页面：[属性注入](di.zh-CN.md#属性注入)。

### 扩展容器

- `#[Definer(priority: N)]` 注册一个把标识符转换为定义的类；`#[Resolver(priority: N)]` 注册一个把定义转换为值的类。两者都接收一个整数 `priority`（默认 `0`），都不可重复。优先级越高越先被询问；优先级相同的保持发现顺序。
- `#[PropertyHandler(TheAttribute::class)]` 把被注解的类注册为某个自定义属性注解的处理器。它可以重复，因此一个处理器可以服务多个注解。被映射的注解用 PHP 的 `#[Attribute(Attribute::TARGET_PROPERTY)]` 声明自己的目标；名称必须是不含 NUL 字节的非空字符串。

这些类要实现的接口以及完整示例见 DI 页面：[自定义属性注解](di.zh-CN.md#自定义属性注解) 和 [定义器与解析器](di.zh-CN.md#定义器与解析器)。

### 声明配置

`#[Configuration]` 服务于 `super-kernel/config`。它不可重复，接收 `?string $className = null` 和 `bool $default = false`：

- 在类上，它声明一个配置来源。`className` 指定该来源提供的类或接口；省略时，由 `super-kernel/config` 根据该类实现的接口推断。给出的 `className` 必须是不含 NUL 字节的非空字符串。
- 在实现了 `SuperKernel\Contract\MultipleConfigInterface` 的来源的方法上，它声明一个具名配置项；方法名就是配置项的名称。`className` 可以省略，也可以重复来源的配置项类型。在多个配置项中，以具名参数传入 `default: true` 来选择默认项。

方法上的 `#[Provision]` 声明该方法创建一个容器服务。它接收任意数量的名称，例如 `#[Provision('a', 'b')]`，按顺序保存在其公开的 `name` 列表中；每个名称都必须是不含 NUL 字节的非空字符串。不给名称时，服务的标识符就是方法的返回类型。该类不需要 `#[Configuration]`，`super-kernel/config` 通过容器调用该方法。

配置接口，保存为 `src/Mail/MailConfig.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Mail;

use SuperKernel\Contract\SingleConfigInterface;

interface MailConfig extends SingleConfigInterface
{
    public function host(): string;
}
```

应用的配置来源，保存为 `src/Mail/AppMailConfig.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Mail;

use SuperKernel\Attribute\Configuration;

#[Configuration(MailConfig::class)]
final readonly class AppMailConfig implements MailConfig
{
    public function host(): string
    {
        return 'smtp.example.com';
    }
}
```

为一个具名服务提供的工厂方法，保存为 `src/Mail/MailServices.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Mail;

use SuperKernel\Attribute\Provision;
use SuperKernel\Contract\ConfigInterface;

final readonly class MailServices
{
    public function __construct(private ConfigInterface $config)
    {
    }

    #[Provision('mail.dsn')]
    public function dsn(): string
    {
        $mail = $this->config->get(MailConfig::class);
        assert($mail instanceof MailConfig);

        return 'smtp://' . $mail->host();
    }
}
```

读取该服务的脚本，保存为 `bin/mail.php`：

```php
<?php

declare(strict_types=1);

use SuperKernel\Context\ApplicationContext;

require dirname(__DIR__) . '/vendor/autoload.php';

echo ApplicationContext::getContainer()->get('mail.dsn'), PHP_EOL;
```

运行 `skernel build` 之后，`php bin/mail.php` 会打印 `smtp://smtp.example.com`。

配置来源、配置项、默认项和工厂方法见 Config 页面：[SuperKernel Config](config.zh-CN.md)。

### 注册事件监听器

在实现了 `SuperKernel\Contract\ListenerInterface` 的类上使用 `#[Listener]`，会把它注册到 `super-kernel/event-dispatcher`。它接收一个整数 `priority`（默认 `0`），不可重复。优先级越高越先运行；优先级相同的保持发现顺序。监听器接收哪些事件由它的 `listen()` 方法返回，而不是由该注解声明。

事件，保存为 `src/Signup/UserRegistered.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Signup;

final readonly class UserRegistered
{
    public function __construct(public string $email)
    {
    }
}
```

高优先级的监听器，保存为 `src/Signup/RecordAuditEntry.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Signup;

use SuperKernel\Attribute\Listener;
use SuperKernel\Contract\ListenerInterface;

#[Listener(priority: 100)]
final class RecordAuditEntry implements ListenerInterface
{
    public function listen(): array
    {
        return [
            UserRegistered::class,
        ];
    }

    public function process(object $event): void
    {
        assert($event instanceof UserRegistered);

        echo "Audit: {$event->email} registered", PHP_EOL;
    }
}
```

默认优先级的监听器，保存为 `src/Signup/AddToNewsletter.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Signup;

use SuperKernel\Attribute\Listener;
use SuperKernel\Contract\ListenerInterface;

#[Listener]
final class AddToNewsletter implements ListenerInterface
{
    public function listen(): array
    {
        return [
            UserRegistered::class,
        ];
    }

    public function process(object $event): void
    {
        assert($event instanceof UserRegistered);

        echo "Newsletter: {$event->email} added", PHP_EOL;
    }
}
```

发布该事件的脚本，保存为 `bin/signup.php`：

```php
<?php

declare(strict_types=1);

use Psr\EventDispatcher\EventDispatcherInterface;
use Src\Signup\UserRegistered;
use SuperKernel\Context\ApplicationContext;

require dirname(__DIR__) . '/vendor/autoload.php';

ApplicationContext::getContainer()
    ->get(EventDispatcherInterface::class)
    ->dispatch(new UserRegistered('ada@example.com'));
```

运行 `skernel build` 之后，`php bin/signup.php` 会打印：

```text
Audit: ada@example.com registered
Newsletter: ada@example.com added
```

### 在包中使用这些注解

自己的代码使用这些注解的包应当直接依赖 `super-kernel/attribute`。通过 `super-kernel/di` 间接获得它会把该包绑定到一个容器上，而这些注解本应能被任何实现解释；只在测试中使用它们的包可以只把它作为开发依赖。

`super-kernel/di` 解释 `Provider`、`FactoryProvider`、`Autowired`、`Definer`、`Resolver` 和 `PropertyHandler`；`super-kernel/config` 解释 `Configuration` 和 `Provision`；`super-kernel/event-dispatcher` 解释 `Listener`。其他容器如何处理它们，由该容器自己决定。

## 错误

注解在被构造时抛出 `InvalidArgumentException`，而构造发生在 `skernel build` 期间。随后 `skernel build` 跳过该类、连同消息一起打印它，并把它记录在 `super-kernel/annotator` 的跳过类诊断中。

| 情形 | 消息 |
| --- | --- |
| `Provider` 的标识符为空或包含 NUL 字节 | `Provider className must be non-empty and contain no NUL bytes.` |
| `FactoryProvider` 的标识符为空或包含 NUL 字节 | `FactoryProvider className must be non-empty and contain no NUL bytes.` |
| `PropertyHandler` 的注解名称为空或包含 NUL 字节 | `PropertyHandler attribute must be non-empty and contain no NUL bytes.` |
| `Autowired` 指定的类或接口不存在 | `Injectable identifier <identifier> does not exist.` |
| `Configuration` 的 `className` 为空或包含 NUL 字节 | `Configuration className must be non-empty and contain no NUL bytes when provided.` |
| `Provision` 的名称为空或包含 NUL 字节 | `Provision name must be non-empty and contain no NUL bytes.` |

## 与其他包的关系

- [`super-kernel/di`](di.zh-CN.md) 解释服务、注入和扩展类注解。
- [`super-kernel/config`](config.zh-CN.md) 解释 `Configuration` 和 `Provision`。
- [`super-kernel/event-dispatcher`](event-dispatcher.zh-CN.md) 解释 `Listener`。
- [`super-kernel/annotator`](annotator.zh-CN.md) 存储 `skernel build` 收集的注解，并回答关于它们的查询。
- [`super-kernel/contract`](contract.zh-CN.md) 声明了被注解的类所实现的接口，例如 `ListenerInterface` 和 `MultipleConfigInterface`。
- 异常处理器用 [`super-kernel/exception-dispatcher`](exception-dispatcher.zh-CN.md) 的注解声明，而不是用本包的注解。

## 不要这样做

**不要依靠 `super-kernel/di` 来安装这些注解。** 代码使用了这些注解的包，在改为安装其他容器的应用中就会失效。请直接依赖 `super-kernel/attribute`。

**不要忽视 `skernel build` 跳过的类。** 无效的注解值（例如 `#[Provider('')]`）会以 `Provider className must be non-empty and contain no NUL bytes.` 失败，并且整个类都会被排除：它的其他注解也一并丢失，应用的行为就像该类没有任何注解一样。请阅读 `skernel build` 打印的内容并修正声明。

**不要期望注解检查其使用方负责检查的内容。** 注解只检查自身的值。`Provider` 的类是否与其标识符兼容、两个类是否提供了同一个标识符，由 `super-kernel/di` 在第一次需要它的查找时报告。
