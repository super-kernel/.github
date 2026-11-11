<p align="right">
  <a href="contract.md">English</a> | <strong>中文文档</strong>
</p>

# SuperKernel Contract

包：[`super-kernel/contract`](https://github.com/super-kernel/contract) · [Packagist](https://packagist.org/packages/super-kernel/contract) · [文档索引](../README.zh-CN.md)

## 用途

`super-kernel/contract` 是 SuperKernel 各个包在互不依赖的情况下协同工作所用的一组接口。需要配置的包面向 `ConfigInterface` 编写代码，而不是面向 `super-kernel/config`；容器会提供应用所安装的任意实现。应用也用同样的接口接收注入的服务。

本包只包含位于 `SuperKernel\Contract` 命名空间中的接口，只依赖 PHP、`reflection` 扩展、`psr/container` 和 `psr/log`。行为、校验和错误都属于实现它们的包。

## 能力

### 依赖能力，而不是依赖包

面向契约声明构造函数参数和属性的类型，让容器提供实现。在由 `super-kernel/skeleton` 创建的应用中，以下包实现了这些契约：

| 契约 | 实现方 |
| --- | --- |
| `ContainerInterface` | [`super-kernel/di`](di.zh-CN.md) |
| `ApplicationInterface` | [`super-kernel/framework`](framework.zh-CN.md) |
| `ConfigInterface` | [`super-kernel/config`](config.zh-CN.md) |
| `BasePathInterface`、`VendorDirInterface`、`PackageManagerInterface` | [`super-kernel/metadata`](metadata.zh-CN.md) |
| `AnnotationCollectorInterface` | [`super-kernel/annotator`](annotator.zh-CN.md) |
| `ReflectorInterface` | [`super-kernel/reflector`](reflector.zh-CN.md) |
| `AutoLoaderInterface` | [`super-kernel/autoloader`](autoloader.zh-CN.md) |
| `ExceptionDispatcherInterface` | [`super-kernel/exception-dispatcher`](exception-dispatcher.zh-CN.md) |
| `StdoutLoggerInterface` | [`super-kernel/logger`](logger.zh-CN.md) |

其他接口描述这些服务返回的值，或者由你自己的类实现：`ListenerInterface` 由监听器实现，`PropertyAttributeHandlerInterface` 由属性处理器实现，`SingleConfigInterface` 和 `MultipleConfigInterface` 由配置实现。

面向 `ReflectorInterface` 编写的服务，保存为 `src/Report/ClassReport.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Report;

use ReflectionMethod;
use SuperKernel\Contract\ReflectorInterface;

use function count;

final readonly class ClassReport
{
    public function __construct(private ReflectorInterface $reflector)
    {
    }

    /**
     * @param class-string $class
     */
    public function describe(string $class): string
    {
        $reflection = $this->reflector->reflectClass($class);
        $methods = $reflection->getMethods(ReflectionMethod::IS_PUBLIC);

        return $reflection->getShortName() . ' has ' . count($methods) . ' public methods';
    }
}
```

一个要描述的类，保存为 `src/Billing/Invoice.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Billing;

final class Invoice
{
    public function total(): int
    {
        return 0;
    }

    public function currency(): string
    {
        return 'EUR';
    }
}
```

使用该服务的脚本，保存为 `bin/report.php`：

```php
<?php

declare(strict_types=1);

use Src\Billing\Invoice;
use Src\Report\ClassReport;
use SuperKernel\Context\ApplicationContext;

require dirname(__DIR__) . '/vendor/autoload.php';

echo ApplicationContext::getContainer()->get(ClassReport::class)->describe(Invoice::class), PHP_EOL;
```

构建项目并运行脚本：

```bash
skernel build
php bin/report.php
```

它会打印 `Invoice has 2 public methods`。

### 使用容器

`ContainerInterface` 扩展了 `Psr\Container\ContainerInterface`，没有增加任何方法。它描述一个保存共享服务的容器，这些服务与应用同生命周期：类或接口标识符返回该类型的实例，其他标识符可以返回任意值。调用方为自己创建的对象（例如为一次请求创建的值）不属于容器；请自己创建，或者通过工厂类创建。需要 SuperKernel 的容器时接收 `ContainerInterface`，任何 PSR-11 容器都可以时接收 `Psr\Container\ContainerInterface`。

### 运行应用

`ApplicationInterface` 只有一个方法 `run(): int`，它运行应用并返回进程状态。如何运行以及进程何时结束由实现决定；`super-kernel/framework` 提供一个 Symfony Console 应用。

### 读取配置

`ConfigInterface` 提供对用 `#[Configuration]` 声明的配置的类型化访问：

- `has($contract, $name = null)` 报告某个配置项是否已声明，不加载任何内容。不带名称时，它检查该契约是否有配置来源，即使该来源有多个配置项且没有默认项。
- `get($contract, $name = null)` 返回配置项，它是 `$contract` 的实例；第一次成功查找时加载它，之后返回同一个对象。不带名称时返回默认项。单一来源本身就是它的配置项，也可以用名称 `default` 获取；有多个配置项的来源以方法名公开每一项，唯一的配置项自动成为默认项。对于未声明的契约或名称，它返回 `null`；加载失败或多个配置项没有默认项时，抛出 `Psr\Container\ContainerExceptionInterface`。失败的加载可以重试。

`SingleConfigInterface` 标记一个自身就是默认项的配置对象；各个包的配置接口扩展它。`MultipleConfigInterface` 标记一个由其方法提供具名配置项的配置来源。两者都没有方法。具体规则见 [`super-kernel/config`](config.zh-CN.md) 的页面。

### 定位项目及其包

- `BasePathInterface` 和 `VendorDirInterface` 都是 `Stringable`：把它们转换为字符串，分别得到项目目录（或打包后应用的 `phar://` 根）和 Composer 的依赖目录，后者不一定存在。
- `PackageManagerInterface` 先列出根包，再按名称列出已安装的包；它可以按任意大小写的名称、按确切的 Composer 类型或按其中的某个类查找包，并返回类映射和环境要求。
- `PackageMetadataInterface` 是一个包；`version()` 返回原样声明的版本，根包未声明版本时返回 `null`，不做规范化或比较。`EnvironmentRequirementInterface` 是对某个 Composer 平台包的一项要求、声明它的包，以及它是否是根包的开发要求。

这些值来自 `skernel build` 的元数据；参见 [`super-kernel/metadata`](metadata.zh-CN.md)。

### 读取收集到的注解

`AnnotationCollectorInterface` 按注解的确切类或其父类、接口查找带有该注解的类和方法，并返回在某个类、方法或属性上声明的注解。每个结果都是一个 `AttributeMetadataInterface`：`attribute()` 返回构建时构造的实例，`name()` 返回它的类，`types()` 返回它的类以及捕获到的父类型。`PackageAnnotationsInterface` 保存一个包的名称、引用以及类、方法和属性映射。

`ScanDiagnosticsInterface` 是一种独立的能力，只有一个方法 `getDiagnostics()`，它返回 `skernel build` 无法收集的类。它本身不是容器条目：请检查收集器或包是否实现了它。读取一行诊断从不加载其中命名的类，这些行也不保存异常对象。

你可以自己实现这些接口，例如在测试中；`super-kernel/annotator` 只发布它自己的模型。查询的说明见 [`super-kernel/annotator`](annotator.zh-CN.md) 的页面。

### 处理自定义属性注解

处理器实现 `PropertyAttributeHandlerInterface`：它的构造函数恰好接收 `ContainerInterface`，`handle($context, $attribute)` 处理一个属性上的一个注解。它收到的 `PropertyContextInterface` 给出类和被反射的属性，并接受一次赋值：`assign($value)` 设置值，第二次 `assign()` 抛出 `LogicException`，不赋值则保持属性不变。`hasAssignment()` 和 `assignment()` 读取这次赋值；没有赋值时 `assignment()` 抛出 `LogicException`。完整示例见 DI 页面：[自定义属性注解](di.zh-CN.md#自定义属性注解)。

### 发布和接收事件

- `EventInterface` 标记 SuperKernel 各个包的事件。它没有方法，也不规定载荷；[`super-kernel/event`](event.zh-CN.md) 声明了生命周期事件。
- `ListenerInterface` 由监听器实现：`listen()` 返回它处理的事件类，`process($event)` 接收每个匹配的事件。处理多种事件类型的监听器在 `process()` 中区分它们。[`super-kernel/event-dispatcher`](event-dispatcher.zh-CN.md) 注册并调用监听器。

### 报告异常

`ExceptionDispatcherInterface::dispatch($throwable)` 把一个 throwable 交给与之匹配的异常处理器，没有处理器处理它时报告它。参见 [`super-kernel/exception-dispatcher`](exception-dispatcher.zh-CN.md)。

### 反射、加载类和记录日志

- `ReflectorInterface` 返回 PHP 原生的 `Reflection*` 对象，覆盖类、方法、属性、常量、枚举、函数、参数、扩展、生成器、纤程和引用，并在多次调用之间复用。失败时抛出 PHP 自己的 `ReflectionException`。参见 [`super-kernel/reflector`](reflector.zh-CN.md)。
- `AutoLoaderInterface` 是进程的类加载器：它的待加载类映射、拦截器以及移除，移除失败时抛出 `RuntimeException`。参见 [`super-kernel/autoloader`](autoloader.zh-CN.md)。
- `StdoutLoggerInterface` 扩展了 `Psr\Log\LoggerInterface`，没有增加方法。它为 SuperKernel 各个包写入日志记录所用的日志记录器命名；默认实现写到标准输出，应用可以替换它的目标、格式和最低级别。参见 [`super-kernel/logger`](logger.zh-CN.md)。

## 与其他包的关系

- 每个提供或使用共享服务的 SuperKernel 包都依赖本包；本包不依赖任何其他 SuperKernel 包。
- 实现这些契约的包列在 [依赖能力，而不是依赖包](#依赖能力而不是依赖包) 中。
- [`super-kernel/attribute`](attribute.zh-CN.md) 声明了实现这些接口的类所带的注解，例如 `ListenerInterface` 上的 `#[Listener]`。
- [`super-kernel/context`](context.zh-CN.md) 保存进程的根容器，该容器由 `super-kernel/di` 创建为一个 `ContainerInterface`。
