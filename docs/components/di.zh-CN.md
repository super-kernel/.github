<p align="right">
  <a href="di.md">English</a> | <strong>中文文档</strong>
</p>

# SuperKernel DI

包：[`super-kernel/di`](https://github.com/super-kernel/di) · [Packagist](https://packagist.org/packages/super-kernel/di) · [文档索引](../README.zh-CN.md)

## 用途

`super-kernel/di` 是 SuperKernel 的依赖注入容器，实现了 PSR-11（`psr/container-implementation` 2.0）。类用注解声明自己提供什么；容器在每个服务第一次被请求时创建它，注入它的依赖，并在容器的整个生命周期内保留它，因此所有使用方共享同一个实例。

容器不会扫描项目。`skernel build` 一次性收集注解并把它们写成元数据；应用启动时，容器恢复这些元数据。

## 能力

### 声明服务

有三类标识符具有定义：

- 可实例化类上的 `#[Provider(Target::class)]` 把 `Target` 映射到该类。该类必须是 `Target` 本身或它的子类型。映射到自身的类会注册为共享类。映射可以串联，例如从一个接口到一个更窄的接口再到一个类，串联中的每个标识符都会得到同一个实例。
- `#[FactoryProvider(Target::class)]` 通过调用被注解的类来创建 `Target`。该类不需要构造函数或者只有公开的构造函数，并且必须自己声明一个公开的非静态 `__invoke()`，其参数会被注入。该注解可以重复，因此一个类可以创建多个标识符。需要用代码完成构造（例如校验或读取设置）时使用它；只靠构造函数就足够时，请使用 `Provider`。
- 其他任何可实例化的类在被请求时都通过其构造函数创建；它不需要注解。

没有 `Provider` 或 `FactoryProvider` 的接口或抽象类没有定义。

接口，保存为 `src/Clock/Clock.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Clock;

use DateTimeImmutable;

interface Clock
{
    public function now(): DateTimeImmutable;
}
```

提供它的类，保存为 `src/Clock/SystemClock.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Clock;

use DateTimeImmutable;
use DateTimeZone;
use SuperKernel\Attribute\Provider;

#[Provider(Clock::class)]
final readonly class SystemClock implements Clock
{
    public function __construct(private DateTimeZone $timeZone)
    {
    }

    public function now(): DateTimeImmutable
    {
        return new DateTimeImmutable('now', $this->timeZone);
    }
}
```

为 `SystemClock` 所需的时区创建实例的工厂，保存为 `src/Clock/TimeZoneProvider.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Clock;

use DateTimeZone;
use SuperKernel\Attribute\FactoryProvider;

use function getenv;

#[FactoryProvider(DateTimeZone::class)]
final class TimeZoneProvider
{
    public function __invoke(): DateTimeZone
    {
        return new DateTimeZone(getenv('APP_TIMEZONE') ?: 'UTC');
    }
}
```

查找这些服务的脚本，保存为 `bin/clock.php`：

```php
<?php

declare(strict_types=1);

use Src\Clock\Clock;
use Src\Clock\SystemClock;
use SuperKernel\Di\Provider\ContainerProvider;

require dirname(__DIR__) . '/vendor/autoload.php';

$container = new ContainerProvider()();

echo $container->get(Clock::class)->now()->getTimezone()->getName(), PHP_EOL;
var_dump($container->get(Clock::class) === $container->get(SystemClock::class));
```

构建项目并运行脚本：

```bash
skernel build
APP_TIMEZONE=Asia/Shanghai php bin/clock.php
```

它会打印：

```text
Asia/Shanghai
bool(true)
```

### 构造函数注入与工厂注入

容器按顺序填充构造函数以及 `FactoryProvider` 的 `__invoke()` 的参数。每个参数取以下来源中第一个可用的：

1. 调用方放入 `SuperKernel\Di\Definition\MethodDefinition` 的值，先按参数名匹配，再按位置匹配；这只适用于由解析器或调用方创建的定义；
2. 当参数类型中恰好有一个类或接口在容器中存在条目时，取该类或接口的条目；
3. 参数的默认值；
4. 当类型允许时，取 `null`。

可变参数不会得到任何值。引用传递的参数、类型中有多个类或接口存在条目的参数，以及上述来源都无法填充的必需参数，都是错误。`string`、`int` 等标量类型永远不会从容器中填充；请为这样的参数提供默认值，或者在 `FactoryProvider` 中创建该对象。

### 属性注入

属性上的 `#[Autowired]` 要求容器在构造函数返回之后为它赋值。容器按以下顺序选择要注入的内容：

1. `#[Autowired(Target::class)]` 指定的类或接口；
2. 否则，属性类型中唯一存在的类或接口；
3. 否则，属性的默认值；
4. 否则，抛出错误。

一旦选定了标识符，就使用容器中的条目；无法创建的条目是一个错误，不会改用默认值。支持公开、受保护、私有以及未初始化的只读属性；静态属性会被拒绝。方法上的 `#[Autowired]` 只是元数据：容器从不调用该方法。

默认情况下，属性得到的是一个 PHP 8.4 延迟代理，而不是条目本身。代理在其状态第一次被读取或写入时通过 `Container::get()` 获取条目，因此依赖只在被使用时才创建。初始化失败会在下一次访问时重试。初始化是同步执行的，没有防止两个协程同时初始化同一个代理的保护。代理的行为与条目相同，但与它并不全等（`===`）。在以下情况下，容器直接注入条目本身，不使用代理：标识符没有已知的具体类（例如只由 `FactoryProvider` 创建的接口），以及除 `stdClass` 之外的内置类、继承它们的类和没有存储型实例属性的类。接口或抽象类只有在 `Provider` 把它映射到具体类时才会被代理。

带有注入属性的类，保存为 `src/Clock/Report.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Clock;

use SuperKernel\Attribute\Autowired;

final class Report
{
    #[Autowired]
    private Clock $clock;

    public function generatedAt(): string
    {
        return $this->clock->now()->format(DATE_ATOM);
    }
}
```

使用它的脚本，保存为 `bin/report.php`：

```php
<?php

declare(strict_types=1);

use Src\Clock\Report;
use SuperKernel\Di\Provider\ContainerProvider;

require dirname(__DIR__) . '/vendor/autoload.php';

echo new ContainerProvider()()->get(Report::class)->generatedAt(), PHP_EOL;
```

运行 `skernel build` 之后，`php bin/report.php` 会打印 UTC 当前时间，例如 `2026-11-10T10:30:00+00:00`。

### 自定义属性注解

包或应用可以为自己的属性注解赋予含义，而无需依赖本包的类：声明一个实现 `SuperKernel\Contract\PropertyAttributeHandlerInterface` 的处理器，并用 `#[PropertyHandler(TheAttribute::class)]` 注解它。

- 被映射的类必须是可实例化、可以作用于属性的原生注解，并且每个注解最多只有一个处理器。`#[PropertyHandler]` 可以重复，因此一个处理器可以服务多个注解；容器为每个处理器创建一个实例，并把根容器传给它的构造函数。
- 容器按反射列出的顺序处理一个类的属性，并在每个属性上按声明顺序处理已映射的注解。没有处理器的属性注解会被忽略。静态属性上带有已处理的注解是一个错误。
- 处理器会收到一个 `SuperKernel\Contract\PropertyContextInterface`，其中包含类名和反射得到的属性。调用 `assign()` 会为属性赋值；不调用它则属性保持原样；`assign(null)` 赋值为 `null`；对同一个属性第二次赋值是一个错误。

注解，保存为 `src/Env/FromEnv.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Env;

use Attribute;

#[Attribute(Attribute::TARGET_PROPERTY)]
final readonly class FromEnv
{
    public function __construct(public string $name)
    {
    }
}
```

它的处理器，保存为 `src/Env/FromEnvPropertyHandler.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Env;

use SuperKernel\Attribute\PropertyHandler;
use SuperKernel\Contract\ContainerInterface;
use SuperKernel\Contract\PropertyAttributeHandlerInterface;
use SuperKernel\Contract\PropertyContextInterface;

use function getenv;

#[PropertyHandler(FromEnv::class)]
final readonly class FromEnvPropertyHandler implements PropertyAttributeHandlerInterface
{
    public function __construct(private ContainerInterface $container)
    {
    }

    public function handle(PropertyContextInterface $context, object $attribute): void
    {
        assert($attribute instanceof FromEnv);

        $value = getenv($attribute->name);

        if (false !== $value) {
            $context->assign($value);
        }
    }
}
```

使用该注解的类，保存为 `src/Mail/MailSettings.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Mail;

use Src\Env\FromEnv;

final class MailSettings
{
    #[FromEnv('MAIL_HOST')]
    public string $host = 'localhost';
}
```

读取该设置的脚本，保存为 `bin/mail.php`：

```php
<?php

declare(strict_types=1);

use Src\Mail\MailSettings;
use SuperKernel\Di\Provider\ContainerProvider;

require dirname(__DIR__) . '/vendor/autoload.php';

echo new ContainerProvider()()->get(MailSettings::class)->host, PHP_EOL;
```

运行 `skernel build` 之后，`php bin/mail.php` 会打印 `localhost`，而 `MAIL_HOST=smtp.example.com php bin/mail.php` 会打印 `smtp.example.com`。

### 定义器与解析器

容器分两步把一个标识符转换为值。定义器（definer）把标识符转换为定义，即描述如何创建该值的对象；解析器（resolver）把定义转换为值。每一步中，容器按优先级从高到低询问已注册的实现，并使用第一个支持该标识符或定义的实现。优先级相同的实现保持它们被发现的顺序，因此请为每个扩展设置各自的优先级。

- `#[Definer(priority: N)]` 注册一个实现 `SuperKernel\Di\Contract\DefinerInterface` 的可实例化类。它的构造函数没有必需参数，或者恰好有一个必需参数，用于接收 `SuperKernel\Contract\AnnotationCollectorInterface`。内置定义器的优先级为 Provider `200`、FactoryProvider `100` 和 Class `0`。容器会记住每个标识符的定义，以及某个标识符没有定义，因此对每个容器而言，一个定义器对同一个标识符最多被询问一次。
- `#[Resolver(priority: N)]` 注册一个实现 `SuperKernel\Di\Contract\ResolverInterface` 的类；容器用根容器创建它，其类型为 `SuperKernel\Contract\ContainerInterface`。内置解析器的优先级为 Factory `400`、Provider `300`、Class `200`、Property `100` 和 Method `0`；高于 `400` 的优先级排在它们所有之前。需要解析另一个定义的解析器把它交给从容器获得的 `SuperKernel\Di\Contract\ResolverFactoryInterface`，而不是直接交给另一个解析器。优先级只用于为定义选择解析器；它不决定构造函数注入或属性注入的顺序。
- 容器在第一次查找时发现定义器和解析器。如果某个扩展的构造函数可能挂起协程，请在容器被并发工作共享之前完成第一次查找。

这些值和其他条目一样会被保留：下面示例中的标识符 `env.APP_ENV` 在每个容器中只读取一次。

定义，保存为 `src/Env/EnvDefinition.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Env;

use SuperKernel\Di\Contract\DefinitionInterface;

final readonly class EnvDefinition implements DefinitionInterface
{
    public function __construct(public string $name)
    {
    }

    public function identifier(): string
    {
        return 'env.' . $this->name;
    }
}
```

处理以 `env.` 开头的标识符的定义器，保存为 `src/Env/EnvDefiner.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Env;

use SuperKernel\Attribute\Definer;
use SuperKernel\Di\Contract\DefinerInterface;
use SuperKernel\Di\Contract\DefinitionInterface;

use function str_starts_with;
use function substr;

#[Definer(priority: 300)]
final class EnvDefiner implements DefinerInterface
{
    public function support(string $id): bool
    {
        return str_starts_with($id, 'env.');
    }

    public function get(string $id): DefinitionInterface
    {
        return new EnvDefinition(substr($id, 4));
    }
}
```

解析器，保存为 `src/Env/EnvResolver.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Env;

use SuperKernel\Attribute\Resolver;
use SuperKernel\Contract\ContainerInterface;
use SuperKernel\Di\Contract\DefinitionInterface;
use SuperKernel\Di\Contract\ResolverInterface;

use function getenv;

#[Resolver(priority: 500)]
final readonly class EnvResolver implements ResolverInterface
{
    public function __construct(private ContainerInterface $container)
    {
    }

    public function support(DefinitionInterface $definition): bool
    {
        return $definition instanceof EnvDefinition;
    }

    public function resolve(DefinitionInterface $definition): mixed
    {
        assert($definition instanceof EnvDefinition);

        $value = getenv($definition->name);

        return false === $value ? null : $value;
    }
}
```

读取一个标识符的脚本，保存为 `bin/env.php`：

```php
<?php

declare(strict_types=1);

use SuperKernel\Di\Provider\ContainerProvider;

require dirname(__DIR__) . '/vendor/autoload.php';

$container = new ContainerProvider()();

var_dump($container->has('env.APP_ENV'));
var_dump($container->get('env.APP_ENV'));
```

运行 `skernel build` 之后，`APP_ENV=production php bin/env.php` 会打印：

```text
bool(true)
string(10) "production"
```

### 恢复根容器

`SuperKernel\Di\Provider\ContainerProvider` 是 `SuperKernel\Contract\ContainerInterface` 和 `Psr\Container\ContainerInterface` 两者的工厂提供者。传入一个容器调用时，它原样返回该容器。不传容器调用时，它恢复 `skernel build` 生成的内容（通过 `super-kernel/metadata` 恢复包元数据，通过 `super-kernel/annotator` 恢复注解记录），并返回一个新的根容器。它从不扫描源码、解析 Composer 文件或启动进程，只对 `composer.lock` 计算哈希以检测安装是否变化，因此项目移动之后以及在 Phar 中，同样的恢复都能工作。每次不传容器的调用都会创建另一个根容器，它拥有自己的共享服务。

根容器在各自的 `SuperKernel\Contract` 接口和具体类下保留这些服务：`AnnotationCollectorInterface`、`BasePathInterface`、`PackageManagerInterface`、`ReflectorInterface` 和 `VendorDirInterface`。需要依赖目录（vendor 目录）时请注入 `VendorDirInterface`；`PackageManagerInterface` 不提供它。

恢复时会抛出：

- `SuperKernel\Metadata\Exception\BuildRequiredException`：当构建标记、包元数据或注解记录缺失、无效、按另一个构建协议写出，或者是为另一个 `composer.lock` 生成的时候。其消息说明具体问题，并以 ``Run `skernel build` to generate the project metadata.`` 结尾，例如 ``The build stamp /srv/app/vendor/.cache/build.json is missing. Run `skernel build` to generate the project metadata.``
- `SuperKernel\Metadata\Exception\MetadataException`：当没有任何已注册的 Composer 类加载器加载 `super-kernel/metadata` 的时候。
- 当某个被恢复的注解或反射器无法构造时，抛出它们自己的异常。

`skernel build` 会打印无法收集其注解的类；容器在启动时不报告它们。

在应用中，`super-kernel/context` 的 `SuperKernel\Context\ApplicationContext::getContainer()` 调用一次 `ContainerProvider`，并在整个进程中保留该根容器；它把失败报告为 `SuperKernel\Context\Exception\ContainerInitializationException`，提供者的异常作为其前一个异常（previous）。

### 自行创建容器

`new SuperKernel\Di\Container($collector, $entries)` 用任意 `SuperKernel\Contract\AnnotationCollectorInterface` 创建一个容器，这在测试中很有用。`$entries` 把标识符映射到已经存在的服务；容器保留它们，就像是它自己创建的一样。不是类名或接口名的标识符可以保存任何值，包括 `null`；类或接口标识符必须保存该类型的实例，否则构造函数会抛出 `ResolutionException`。

每个容器都保留以下标识符，它们优先于传入的条目：

- `SuperKernel\Di\Container`、`SuperKernel\Contract\ContainerInterface` 和 `Psr\Container\ContainerInterface` 返回容器本身；
- `SuperKernel\Contract\AnnotationCollectorInterface` 返回收集器；
- `SuperKernel\Di\Contract\DefinitionFactoryInterface`、`SuperKernel\Di\Factory\DefinitionFactory`、`SuperKernel\Di\Contract\ResolverFactoryInterface` 和 `SuperKernel\Di\Factory\ResolverFactory` 返回容器自己的工厂。

一个问候接口，保存为 `src/Greeting/Greeter.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Greeting;

interface Greeter
{
    public function greet(string $name): string;
}
```

为它提供测试替身的脚本，保存为 `bin/entries.php`：

```php
<?php

declare(strict_types=1);

use Src\Clock\Clock;
use Src\Greeting\Greeter;
use SuperKernel\Contract\AnnotationCollectorInterface;
use SuperKernel\Di\Container;
use SuperKernel\Di\Provider\ContainerProvider;

require dirname(__DIR__) . '/vendor/autoload.php';

$root = new ContainerProvider()();

$greeter = new class implements Greeter {
    public function greet(string $name): string
    {
        return "Hi, {$name}.";
    }
};

$container = new Container($root->get(AnnotationCollectorInterface::class), [
    Greeter::class => $greeter,
    'app.name'     => 'Billing',
]);

echo $container->get(Greeter::class)->greet('SuperKernel'), PHP_EOL;
echo $container->get('app.name'), PHP_EOL;
var_dump($container->get(Clock::class) === $root->get(Clock::class));
```

运行 `skernel build` 之后，`php bin/entries.php` 会打印以下内容；最后一行表明两个容器不共享任何东西：

```text
Hi, SuperKernel.
Billing
bool(false)
```

### 解析调用方持有的对象

容器会保留它创建的所有东西。对于每次都应该是新的对象（例如为一个请求构建的值），请获取它的定义并自己解析：结果不会被保留，而它的依赖仍然来自容器并被共享。

发票，保存为 `src/Billing/Invoice.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Billing;

use Src\Clock\Clock;

final class Invoice
{
    /** @var list<string> */
    public array $lines = [];

    public function __construct(public readonly Clock $clock)
    {
    }
}
```

创建两张发票的脚本，保存为 `bin/invoice.php`：

```php
<?php

declare(strict_types=1);

use Src\Billing\Invoice;
use Src\Clock\Clock;
use SuperKernel\Di\Contract\DefinitionFactoryInterface;
use SuperKernel\Di\Contract\ResolverFactoryInterface;
use SuperKernel\Di\Provider\ContainerProvider;

require dirname(__DIR__) . '/vendor/autoload.php';

$container = new ContainerProvider()();
$definitions = $container->get(DefinitionFactoryInterface::class);
$resolvers = $container->get(ResolverFactoryInterface::class);

$definition = $definitions->get(Invoice::class)
    ?? throw new RuntimeException('Invoice has no definition.');

$first = $resolvers->resolve($definition);
$second = $resolvers->resolve($definition);

var_dump($first === $second);
var_dump($first->clock === $container->get(Clock::class));
```

运行 `skernel build` 之后，`php bin/invoice.php` 会打印：

```text
bool(false)
bool(true)
```

对于没有定义的标识符，`DefinitionFactoryInterface::get()` 返回 `null`。这不是瞬态作用域：创建该对象期间请求的每个依赖都会被容器保留。属性注入仍然作用于该对象。

## 错误

只有当标识符没有定义时，`get()` 才抛出实现了 `Psr\Container\NotFoundExceptionInterface` 的 `SuperKernel\Di\Exception\NotFoundException`。其他所有失败都抛出实现了 `Psr\Container\ContainerExceptionInterface` 的 `SuperKernel\Di\Exception\ResolutionException`，原始异常作为其前一个异常（previous）。你自己的构造函数、工厂或处理器抛出的 PSR-11 异常会原样传出，依赖的未找到异常也是如此。

| 情形 | 消息 |
| --- | --- |
| 没有定义 | `Container entry "<id>" has no definition.` |
| 同一个标识符有两个 `Provider`、无法实例化的自身映射、不是其目标子类型的提供者、同一个标识符有两个 `FactoryProvider`、工厂没有公开的构造方式或自己的公开非静态 `__invoke()`，或者定义器失败 | `Unable to obtain a definition for container entry "<id>".`，原因在前一个异常中，例如 `Provider <id> is declared by both <class> and <class>.` 或 `<class> must declare a public non-static __invoke method.` |
| 参数类型中没有任何类或接口存在条目，且没有默认值 | `Unable to resolve method parameter <class>::<method>($<parameter>) of type <type>.` |
| 参数类型中有多个类或接口存在条目 | `Method parameter <class>::<method>($<parameter>) is ambiguous: <id>, <id> are available.` |
| 引用传递的参数 | `Method parameter <class>::<method>($<parameter>) cannot be resolved because references are unsupported.` |
| 构造函数抛出异常 | `Unable to construct container entry "<class>".` |
| 工厂抛出异常 | `Unable to invoke factory "<factory>" for container entry "<id>".` |
| 类或接口标识符解析为其他类型 | `Container entry "<id>" resolved to <type> instead of an instance of <id>.` |
| `#[Autowired]` 属性没有可用的类型或默认值 | `Unable to resolve property <class>::$<property> of type <type>.` |
| 静态属性上带有已处理的注解 | `Static property <class>::$<property> cannot be injected.` |
| 属性处理器抛出异常或重复赋值 | `Property handler <handler> failed while processing <attribute> on <class>::$<property>.` |
| 无效的 `#[PropertyHandler]` 映射 | `Unable to inspect injectable properties on <class>.`，原因在前一个异常中 |
| 没有解析器支持某个定义 | `No resolver supports definition "<definition class>".` |

## 与其他包的关系

- [`super-kernel/attribute`](attribute.zh-CN.md) 声明了 `Provider`、`FactoryProvider`、`Autowired`、`PropertyHandler`、`Definer` 和 `Resolver`；本包赋予它们含义。
- [`super-kernel/contract`](contract.zh-CN.md) 声明了 `ContainerInterface`、`PropertyAttributeHandlerInterface`、`PropertyContextInterface` 以及根容器保留的各项服务。请针对这些接口编写代码，而不是针对本包的类。
- [`super-kernel/metadata`](metadata.zh-CN.md) 恢复包元数据、基础路径和依赖目录，并抛出 `BuildRequiredException`；[`super-kernel/annotator`](annotator.zh-CN.md) 恢复注解记录；[`super-kernel/reflector`](reflector.zh-CN.md) 提供共享的反射器。
- [`super-kernel/context`](context.zh-CN.md) 保存进程的根容器，并通过 `ContainerProvider` 创建它。
- `skernel build` 生成容器所恢复的元数据，并打印被跳过的类；`skernel run` 在启动应用之前先构建。
- 其他每个 SuperKernel 包都用同样的注解声明自己的服务，因此运行 `skernel build` 之后，已安装包的服务无需配置即可使用。

## 不要这样做

**不要用过期的构建启动应用。** 容器只能看到上一次 `skernel build` 收集到的内容。`composer.lock` 的变化会被检测到并抛出 `BuildRequiredException`，但新增或修改带注解的类不会被检测到：容器会悄悄地继续使用旧的映射。每次这样的修改之后都请运行 `skernel build`，或者用 `skernel run` 启动。

**不要为同一个标识符声明两个提供者。** 同一个标识符上的两个 `#[Provider]` 或两个 `#[FactoryProvider]`，会使内置定义器在容器第一次需要它时失败；从那以后，所有尚未创建的条目的查找都会抛出 `ResolutionException`，`has()` 也会报告 `false`，与之无关的标识符同样如此。`skernel build` 不会报告这个问题。每个标识符只保留一个提供者；要替换实现，请修改这一个声明，或者在测试中向 `new Container()` 传入一个条目。

**不要把接口或抽象类映射到自身。** 在无法实例化的类上使用 `#[Provider(Self::class)]` 会被拒绝，消息为 `Self-provided class <class> must be instantiable.` 请改为注解具体类：`#[Provider(TheInterface::class)]`。

**不要在构造函数足够时使用 `FactoryProvider`。** 只调用 `new` 的工厂会对容器和读者隐藏类的依赖。请使用 `#[Provider]`，让容器注入构造函数；把 `FactoryProvider` 留给需要用代码完成的构造。工厂必须自己声明 `__invoke()`；从父类继承的 `__invoke()` 会被拒绝。

**不要构造循环依赖。** 容器不检测循环：一个类直接或通过其他类经由构造函数依赖自身时，会一直递归，直到 PHP 以致命错误（例如内存耗尽）结束进程，任何 `catch` 都无法处理。请把两个类都需要的部分移到第三个类中。

**不要在构造函数中使用注入的属性。** 属性在构造函数返回之后才赋值，因此在构造函数中读取它会抛出 `Typed property <class>::$<property> must not be accessed before initialization`，并被包装在 `Unable to construct container entry "<class>".` 中。请改为通过构造函数参数接收该依赖。

**不要用 `===` 比较延迟属性。** `#[Autowired]` 属性通常持有一个代理，它的行为与条目相同，但它是另一个对象。请比较对象所携带的值，或者通过容器查找该条目。

**不要在容器服务中保存按请求的状态。** 每个条目的生命周期与容器相同，在长期运行的服务器中就是整个进程，并且被所有请求和协程共享。请把请求数据放在请求中，按[解析调用方持有的对象](#解析调用方持有的对象)中所示解析调用方持有的对象，或者注入一个创建这些对象的工厂类。

**不要让并发工作执行服务的第一次查找。** 容器不会串行化构造：两个协程同时请求同一个新条目时，可能各自构造一次，而只有第一个结果会被保留。请在服务器启动其工作进程或协程之前，请求它共享的服务。

**不要把 `has()` 理解为"可以创建"。** `has()` 只报告一个标识符已被保留或具有定义。构造函数参数无法解析的类仍然报告 `true`，而它的 `get()` 会失败。

**不要通过调用提供者或 `new` 获取服务。** 调用提供者的 `__invoke()`、实例化容器所提供的服务，或者用手动创建的对象替换注入的条目，都会让代码拿到一个应用其他部分不共享的第二个实例。请通过构造函数参数或 `#[Autowired]` 属性接收服务；只在入口点（例如脚本或服务器回调）中从容器查找服务。在应用中，入口点请使用 `ApplicationContext::getContainer()`，而不是再次调用 `new ContainerProvider()()`，后者会恢复出第二个拥有自己服务的根容器。

**不要按类型向容器索取标量值。** `string` 或 `int` 参数永远不会被注入。请为它提供默认值，在 `FactoryProvider` 中读取它，或者像[自定义属性注解](#自定义属性注解)中那样使用自定义属性注解。
