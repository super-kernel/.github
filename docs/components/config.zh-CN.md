<p align="right">
  <a href="config.md">English</a> | <strong>中文文档</strong>
</p>

# SuperKernel Config

包：[`super-kernel/config`](https://github.com/super-kernel/config) · [Packagist](https://packagist.org/packages/super-kernel/config) · [文档索引](../README.zh-CN.md)

## 用途

`super-kernel/config` 把应用声明的配置与读取配置的组件连接起来。组件发布一个配置契约（一个类或接口）；应用用 `#[Configuration]` 为它声明一个配置来源；组件通过 `SuperKernel\Contract\ConfigInterface` 读取配置项，第一次使用时加载，之后返回同一个对象。一个来源可以提供一个对象，也可以提供多个带有默认项的具名配置项，例如多个数据库连接。

Config 只处理对象。从文件或环境中读取原始值属于配置来源的工作（例如借助 `super-kernel/environment`）；检查某个值是否适合某个组件属于该组件。本包还赋予 `#[Provision]` 方法含义：每个这样的方法都成为一个容器服务的工厂。

## 能力

### 声明单一来源

类上的 `#[Configuration(Contract::class)]` 把它声明为 `Contract`（一个类或接口）的来源。该来源必须是一个与契约兼容的可实例化类，它本身就是该契约唯一的配置项，即默认项，也可以用名称 `default` 获取。

- 不带参数时，`#[Configuration]` 使用该类实现的唯一一个最具体的接口，忽略 `SingleConfigInterface` 以及通过其他接口继承而来的接口。如果不恰好是一个接口，该声明无效。
- 组件的配置接口通常扩展 `SuperKernel\Contract\SingleConfigInterface`；单一来源本身不需要它。
- 一个契约最多只有一个来源。
- Config 通过容器创建来源，因此它的构造函数可以接收服务和其他配置。

配置接口，保存为 `src/Mail/MailConfig.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Mail;

use SuperKernel\Contract\SingleConfigInterface;

interface MailConfig extends SingleConfigInterface
{
    public function host(): string;

    public function port(): int;
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

    public function port(): int
    {
        return 587;
    }
}
```

读取它的脚本，保存为 `bin/config.php`：

```php
<?php

declare(strict_types=1);

use Src\Mail\MailConfig;
use SuperKernel\Context\ApplicationContext;
use SuperKernel\Contract\ConfigInterface;

require dirname(__DIR__) . '/vendor/autoload.php';

$config = ApplicationContext::getContainer()->get(ConfigInterface::class);
$mail = $config->get(MailConfig::class);

echo $mail->host(), ':', $mail->port(), PHP_EOL;
var_dump($mail === $config->get(MailConfig::class, 'default'));
```

构建项目并运行脚本：

```bash
skernel build
php bin/config.php
```

它会打印：

```text
smtp.example.com:587
bool(true)
```

### 声明具名配置项

实现了 `SuperKernel\Contract\MultipleConfigInterface` 的来源提供同一契约的多个配置项，其类级别的 `#[Configuration(Contract::class)]` 必须显式指定该契约。每个配置项都是一个标有 `#[Configuration]` 的方法：

- 方法名（区分大小写）就是配置项的名称。没有标记的方法不是配置项。
- 方法必须是公开、非静态、无参数的，并声明一个与契约兼容、不可为空的类或接口返回类型。方法级别的 `#[Configuration]` 可以重复该契约或省略它，但不能指定另一个契约。
- 来源本身不实现该契约；它返回的配置项必须是该契约的实例。
- 只有一个配置项时，它就是默认项。有多个时，用 `#[Configuration(default: true)]` 标记默认项，最多只能标记一个。没有标记默认项时，按名称查找可用，只有不带名称的查找会失败。方法的顺序从不决定默认项。
- 没有配置项的来源，或者同时实现了 `SingleConfigInterface` 和 `MultipleConfigInterface` 的来源，都是无效的。

契约可以是一个普通类，例如带有公开属性的 final readonly 类；它不需要接口，也不需要容器条目。Config 自己调用配置项方法，从不向容器请求配置项。

选项类，保存为 `src/Database/DatabaseConfig.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Database;

final readonly class DatabaseConfig
{
    public function __construct(public string $host, public string $database)
    {
    }
}
```

提供两个连接的来源，保存为 `src/Database/DatabaseConnections.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Database;

use SuperKernel\Attribute\Configuration;
use SuperKernel\Contract\MultipleConfigInterface;

#[Configuration(DatabaseConfig::class)]
final readonly class DatabaseConnections implements MultipleConfigInterface
{
    #[Configuration(default: true)]
    public function primary(): DatabaseConfig
    {
        echo 'Loading primary', PHP_EOL;

        return new DatabaseConfig('db.internal', 'billing');
    }

    #[Configuration]
    public function reporting(): DatabaseConfig
    {
        echo 'Loading reporting', PHP_EOL;

        return new DatabaseConfig('replica.internal', 'billing');
    }
}
```

读取这些连接的脚本，保存为 `bin/database.php`：

```php
<?php

declare(strict_types=1);

use Src\Database\DatabaseConfig;
use SuperKernel\Context\ApplicationContext;
use SuperKernel\Contract\ConfigInterface;

require dirname(__DIR__) . '/vendor/autoload.php';

$config = ApplicationContext::getContainer()->get(ConfigInterface::class);

var_dump($config->has(DatabaseConfig::class, 'reporting'), $config->has(DatabaseConfig::class, 'archive'));

$primary = $config->get(DatabaseConfig::class);

echo $primary->host, PHP_EOL;
var_dump($primary === $config->get(DatabaseConfig::class, 'primary'));
echo $config->get(DatabaseConfig::class, 'reporting')->host, PHP_EOL;
echo $config->get(DatabaseConfig::class, 'reporting')->host, PHP_EOL;
var_dump($config->get(DatabaseConfig::class, 'archive'));
```

运行 `skernel build` 之后，`php bin/database.php` 会打印以下内容；每个配置项方法只在其配置项第一次被读取时运行一次：

```text
bool(true)
bool(false)
Loading primary
db.internal
bool(true)
Loading reporting
replica.internal
replica.internal
NULL
```

### 读取配置项

`ConfigInterface` 由容器创建。创建它时会检查应用及其各个包的所有声明，但不创建任何来源，也不调用任何配置项方法。

- `has($contract)` 报告该契约是否有来源，即使该来源有多个配置项且没有默认项；`has($contract, $name)` 报告该配置项是否已声明。两者都不加载任何内容，也都不表示加载一定会成功。
- `get($contract, $name = null)` 返回配置项。第一次查找时，它通过容器创建来源，检查来源是其自身类的实例，只调用被请求的配置项方法，并保留结果；之后按该名称或作为默认项查找都返回这个对象。
- 对于未声明的契约或名称，`get()` 返回 `null`，不会访问容器。对于有多个配置项但没有默认项的情况，`get($contract)` 在创建来源之前就抛出 `ConfigurationNotFoundException`。
- 失败的加载不保留任何内容，下一次查找会重试；如果来源已经创建，而它的某个配置项方法失败，来源仍会保留。
- 在加载过程中直接或通过其他配置需要自身的来源或配置项，会抛出 `CircularConfigurationDependencyException`，并给出契约和配置项的路径。
- 本包的异常会原样传出。其他失败以及类型错误的配置项会被报告为 `ConfigurationLoadingException`，以失败原因作为其前一个异常（previous）。

配置项被保留而不是被复制：在应用运行期间，请让它们的读取方法始终返回相同的值。

### 在组件中校验配置

Config 只检查类型。需要有效值（例如端口在范围内）的组件用一个 `#[FactoryProvider]` 绑定它的配置接口：该提供者通过 `ConfigInterface` 加载配置项，缺失时以 `ConfigurationNotFoundException` 拒绝，检查各个值，然后返回同一个对象。需要该配置的代码随后通过注入接收这个已经检查过的接口。Config 按类创建来源，从不按接口创建，因此这个提供者不会调用到自身。

- 检查失败时，容器不会为该接口保留条目；Config 可能已经保留了原始配置项，因此提供者会在下一次尝试时再次检查它。
- 提供具名配置项（例如多个连接）的组件，在其默认提供者和各个工厂之间共享一个逐项检查的加载器。
- 可选组件先检查 `has()`，没有任何声明时跳过自身；但已声明却无效的来源，或者缺失的默认项，仍然必须失败。

一个负责校验的提供者，保存为 `src/Mail/MailConfigProvider.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Mail;

use InvalidArgumentException;
use SuperKernel\Attribute\FactoryProvider;
use SuperKernel\Config\Exception\ConfigurationNotFoundException;
use SuperKernel\Contract\ConfigInterface;

#[FactoryProvider(MailConfig::class)]
final readonly class MailConfigProvider
{
    public function __invoke(ConfigInterface $config): MailConfig
    {
        $mail = $config->get(MailConfig::class)
            ?? throw new ConfigurationNotFoundException('No mail configuration is declared.');

        if ($mail->port() < 1 || $mail->port() > 65535) {
            throw new InvalidArgumentException("Mail port {$mail->port()} is out of range.");
        }

        return $mail;
    }
}
```

接收已检查配置的服务，保存为 `src/Mail/Mailer.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Mail;

final readonly class Mailer
{
    public function __construct(private MailConfig $config)
    {
    }

    public function describe(): string
    {
        return "Sending through {$this->config->host()}:{$this->config->port()}";
    }
}
```

使用该服务的脚本，保存为 `bin/mailer.php`：

```php
<?php

declare(strict_types=1);

use Src\Mail\Mailer;
use SuperKernel\Context\ApplicationContext;

require dirname(__DIR__) . '/vendor/autoload.php';

echo ApplicationContext::getContainer()->get(Mailer::class)->describe(), PHP_EOL;
```

运行 `skernel build` 之后，`php bin/mailer.php` 会打印 `Sending through smtp.example.com:587`。

### 用工厂方法创建服务

标有 `#[Provision]` 的方法成为一个容器服务的工厂。它所在的类不需要 `#[Configuration]`；容器创建该类一次（注入其构造函数），并用注入的参数调用该方法。

- 不给名称时，服务的标识符就是方法的返回类型，它必须是一个不可为空的类或接口；`self`、`static` 和 `parent` 相对于声明它的类解析。其他返回类型都需要给出名称。
- `#[Provision('a', 'b')]` 恰好注册这些标识符。每个标识符保留自己的产物，因此 `a` 和 `b` 是不同的对象，而两者共享同一个工厂类。同一个方法上重复的名称只算一次；两个方法使用同一个标识符是错误。
- 工厂方法不能提供它自己所在的类。
- 该类必须可实例化，方法必须是公开、非静态的，并且既不是构造函数也不是析构函数。
- 对同一个标识符，`#[Provider]` 和 `#[FactoryProvider]` 优先于工厂方法；工厂方法优先于对类的普通构造。
- 容器保留每个产物，对于不是类的标识符也会保留 `null`。参数按 `super-kernel/di` 的规则填充。PSR-11 异常原样传出；其他失败被包装为 `ConfigurationLoadingException`，下一次查找会重试。工厂方法之间的依赖循环不会被检测。

带有工厂方法的类，保存为 `src/Clock/ClockServices.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Clock;

use DateTimeImmutable;
use DateTimeZone;
use SuperKernel\Attribute\Provision;
use Src\Mail\MailConfig;

final readonly class ClockServices
{
    public function __construct(private MailConfig $mail)
    {
    }

    #[Provision]
    public function timeZone(): DateTimeZone
    {
        return new DateTimeZone('Europe/Berlin');
    }

    #[Provision('clock.started', 'clock.now')]
    public function now(DateTimeZone $timeZone): DateTimeImmutable
    {
        return new DateTimeImmutable('2026-11-10 09:00', $timeZone);
    }

    #[Provision('mail.dsn')]
    public function mailDsn(): string
    {
        return "smtp://{$this->mail->host()}:{$this->mail->port()}";
    }
}
```

读取这些服务的脚本，保存为 `bin/provisions.php`：

```php
<?php

declare(strict_types=1);

use SuperKernel\Context\ApplicationContext;

require dirname(__DIR__) . '/vendor/autoload.php';

$container = ApplicationContext::getContainer();

echo $container->get(DateTimeZone::class)->getName(), PHP_EOL;
echo $container->get('clock.now')->format(DATE_ATOM), PHP_EOL;
var_dump($container->get('clock.now') === $container->get('clock.now'));
var_dump($container->get('clock.now') === $container->get('clock.started'));
echo $container->get('mail.dsn'), PHP_EOL;
```

运行 `skernel build` 之后，`php bin/provisions.php` 会打印：

```text
Europe/Berlin
2026-11-10T09:00:00+01:00
bool(true)
bool(false)
smtp://smtp.example.com:587
```

`SuperKernel\Config\Collector\ProvisionCollector` 列出这些声明，但不创建或调用任何东西：`getMethods()` 按类和方法名返回 `Provision` 实例（`attribute`）以及方法原生的返回类型或 `null`（`returnType`）。无法反射的类或方法会在创建收集器时抛出 `ReflectionException`。

## 错误

本包的每个异常都实现了 `SuperKernel\Config\Exception\ConfigExceptionInterface`，它是一个 PSR-11 `ContainerExceptionInterface`；`ConfigurationNotFoundException` 同时也是 PSR-11 未找到异常。

| 情形 | 消息 |
| --- | --- |
| 一个契约有两个来源 | `Configuration contract <contract> is declared by both <source> and <source>.` |
| 不带契约的 `#[Configuration]` 所在的类不恰好有一个最具体的接口 | `Configuration source <source> must implement exactly one most-specific interface when SuperKernel\Attribute\Configuration has no explicit argument; found <n>: <interfaces>.` |
| 来源与其契约不兼容 | `Configuration source <source> must be compatible with its declared contract <contract>.` |
| 来源无法实例化 | `Configuration source <source> must be an instantiable class.` |
| 契约不是已存在的类或接口 | `Configuration source <source> explicitly names <contract>, which is not an existing class or interface.` |
| 带有具名配置项的来源没有指定其契约 | `Multiple configuration source <source> must explicitly name its item contract in Configuration.` |
| 来源既是单一的又是多项的 | `A configuration source cannot be both single and multiple.` |
| 配置项方法签名错误或带有两个 `#[Configuration]` | `Configuration item <source>::<method> must be a public non-static parameterless method returning <contract>, marked exactly once with Configuration.` |
| 配置项方法指定了另一个契约 | `Configuration item <source>::<method> must use its source contract <contract> instead of <other>.` |
| 两个默认项 | `Multiple configuration source <source> declares more than one default item.` |
| 没有配置项的来源 | `Multiple configuration source <source> must declare at least one configuration item.` |
| 多个配置项、没有默认项、不带名称查找（`ConfigurationNotFoundException`） | `Configuration contract <contract> has multiple items but no default item is declared.` |
| 需要自身的来源或配置项（`CircularConfigurationDependencyException`） | `Circular configuration dependency detected: <path>.` |
| 来源或配置项方法失败（`ConfigurationLoadingException`） | `Unable to load configuration item <contract> from source <source>.` |
| 类型错误的配置项（`ConfigurationLoadingException`） | `Configuration item <contract> resolved to <type> instead of <contract>.` |
| 一个标识符有两个工厂方法 | `Provision for <id> is declared by both <class>::<method> and <class>::<method>.` |
| 不带名称的工厂方法没有可用的返回类型 | `Unnamed provision <class>::<method> must declare a non-nullable class or interface return type; otherwise specify Provision names explicitly.` |
| 工厂方法提供它自己所在的类 | `Provision <class>::<method> cannot replace its own source class; use an explicit service name.` |
| 工厂方法不是可实例化类上的公开非静态方法 | `Provision method <class>::<method> must be public and non-static on an instantiable class, and cannot be a constructor or destructor.` |
| 工厂方法失败（`ConfigurationLoadingException`） | `Unable to invoke provision <class>::<method> for container entry "<id>".` |

无效的工厂方法由容器的定义器报告：容器抛出 `Unable to obtain a definition for container entry "<id>".`，Config 的异常位于其前一个异常链中。

## 与其他包的关系

- [`super-kernel/attribute`](attribute.zh-CN.md) 声明了 `#[Configuration]` 和 `#[Provision]`；[`super-kernel/contract`](contract.zh-CN.md) 声明了 `ConfigInterface`、`SingleConfigInterface` 和 `MultipleConfigInterface`。
- [`super-kernel/di`](di.zh-CN.md) 创建来源和工厂方法所在的类，填充它们的参数，并保留工厂方法的产物；本包在其中注册一个定义器和一个解析器。
- [`super-kernel/annotator`](annotator.zh-CN.md) 提供收集到的声明；[`super-kernel/reflector`](reflector.zh-CN.md) 提供共享的反射器；[`super-kernel/support`](support.zh-CN.md) 的 `make()` 创建 `ConfigInterface`。
- [`super-kernel/environment`](environment.zh-CN.md) 读取来源可以使用的原始属性，例如 `.env` 中的值。
- [`super-kernel/server`](server.zh-CN.md) 和 [`super-kernel/redis`](redis.zh-CN.md) 等组件发布配置契约并检查它们的值。

## 不要这样做

**不要为一个契约声明两个来源。** 创建 `ConfigInterface` 会以 `Configuration contract <contract> is declared by both <source> and <source>.` 失败，任何需要配置的代码都无法启动。每个契约只保留一个来源；把备选项作为同一个来源的具名配置项提供。

**不要为一个标识符声明两个工厂方法。** 容器的定义器会失败，此后所有尚未创建的条目的查找都会以 `Unable to obtain a definition for container entry "<id>".` 失败，与之无关的服务也不例外。请为每个工厂方法使用各自的名称。

**不要依赖方法顺序来决定默认项。** 有多个配置项且没有 `default: true` 时，`get($contract)` 会抛出 `Configuration contract <contract> has multiple items but no default item is declared.`。请恰好标记一个默认项，或者始终按名称读取。

**不要为了设置值而替换组件负责校验的提供者。** 你自己为组件的配置接口编写的 `#[Provider]` 或 `#[FactoryProvider]` 会绕过组件的检查。请改为声明一个带有你的值的 `#[Configuration]` 来源。

**不要为配置项制作快照或修改配置项。** Config 返回并保留你的来源构建的对象；使用方复制的副本会过期，而读取方法在运行时变化的配置项会给不同的组件不同的答案。请保持配置项不可变，并通过 `ConfigInterface` 或注入读取它们。

**不要让来源之间形成循环依赖。** 构造函数读取了一个反过来需要它自己的配置的来源，会以 `Circular configuration dependency detected: <path>.` 失败。请把共享的值移到第三个来源中。工厂方法之间的循环根本不会被检测，会终止进程。

**不要在组件内部读取原始的环境值。** 组件读取类型化的配置；由应用的来源把原始值（例如来自 `super-kernel/environment` 的值）转换为配置对象。
