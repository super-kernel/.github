<p align="right">
  <a href="aop.md">English</a> | <strong>中文文档</strong>
</p>

# SuperKernel AOP

包：[`super-kernel/aop`](https://github.com/super-kernel/aop) · [Packagist](https://packagist.org/packages/super-kernel/aop) · [文档索引](../README.zh-CN.md)

## 用途

`super-kernel/aop` 让代码在其他类的方法周围运行而无需修改这些类：例如属于许多调用的日志、计时、缓存或检查。切面是一个用 `#[Aspect]` 声明其目标并实现 `AspectInterface` 的类。当应用已经启动、目标类首次被加载时，本包会改为加载该类的一份转换后的副本，其中每个方法都把调用交给切面。

转换后的源码由 `super-kernel/mimic` 准备和缓存；类映射来自 `SuperKernel\Contract\AutoLoaderInterface`，只要存在一个切面，本包就需要它。`super-kernel/autoloader` 提供该接口；本包建议（suggest）但并不依赖该包，请用 `composer require super-kernel/autoloader` 安装它。织入随 `super-kernel/framework` 的应用启动而开始，`skernel build` 会预先准备转换后的类。

## 能力

### 声明切面

切面是一个带有 `SuperKernel\Aop\Attribute\Aspect` 并实现 `SuperKernel\Aop\Contract\AspectInterface` 的类。它由容器创建，因此通过构造函数接收服务。`process()` 为每次选中的调用接收一个 `SuperKernel\Aop\Contract\ProceedingJoinPointInterface`，并返回调用的结果：调用 `proceed()` 会运行原始方法。

一个服务，保存为 `src/Billing/PriceCalculator.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Billing;

final class PriceCalculator
{
    public function total(int $net, int $taxPercent): int
    {
        return $net + intdiv($net * $taxPercent, 100);
    }
}
```

针对其 `total()` 方法的切面，保存为 `src/Aspect/LogCalculation.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Aspect;

use Src\Billing\PriceCalculator;
use SuperKernel\Aop\Attribute\Aspect;
use SuperKernel\Aop\Contract\AspectInterface;
use SuperKernel\Aop\Contract\ProceedingJoinPointInterface;

#[Aspect(methods: [PriceCalculator::class => ['total']])]
final readonly class LogCalculation implements AspectInterface
{
    public function process(ProceedingJoinPointInterface $point): mixed
    {
        $arguments = implode(', ', $point->getArguments());
        $result = $point->proceed();

        echo "{$point->getShortMethodName()}({$arguments}) = {$result}", PHP_EOL;

        return $result;
    }
}
```

使用该服务的命令，保存为 `src/Command/TotalCommand.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Command;

use Src\Billing\PriceCalculator;
use SuperKernel\Command\Attribute\Command;
use Symfony\Component\Console\Command\Command as SymfonyCommand;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

#[Command(name: 'invoice:total', description: 'Print an invoice total.')]
final class TotalCommand extends SymfonyCommand
{
    public function __construct(private readonly PriceCalculator $calculator)
    {
        parent::__construct();
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $output->writeln("Total: {$this->calculator->total(1000, 19)}");

        return self::SUCCESS;
    }
}
```

构建项目并运行命令：

```bash
skernel build
php bin/bootstrap.php invoice:total
```

它会打印：

```text
total(1000, 19) = 1190
Total: 1190
```

### 选择目标

`#[Aspect]` 是一个只能用于类、不可重复的注解；一个声明可以在三个参数中列出任意数量的目标：

- `classes`：类名列表；每个类的所有方法都会被选中；
- `methods`：从类名映射到非空方法名列表；
- `propertyHooks`：从类名映射到属性名（不带 `$`），每个属性名再映射到由 `get` 和 `set` 组成的非空列表；会被接受，但目前不会被织入，见[不要这样做](#不要这样做)。

类名可以以命名空间分隔符开头。不接受通配符，也不接受 `Class::method` 之类的选择器字符串。名称、它们的顺序以及重复项都按原样保留。该注解检查名称的语法和列表的结构，对格式错误的声明抛出 `InvalidArgumentException`；它从不加载目标，也不检查成员是否存在，因此声明可以指向很久之后才加载的类。目标按名称匹配，忽略大小写和开头的命名空间分隔符，并且只选中所指定的类：其子类以及接口的实现不会被选中。除非切面指定了自己，否则它不是自己的目标。

调用被选中的类时，方法选择器优先于同一个类的类选择器；当多个切面选中同一个方法或类时，构建最后发现的那个会替换之前的。

一个有两个方法的类，保存为 `src/Billing/InvoiceNumbers.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Billing;

use function sprintf;

final class InvoiceNumbers
{
    private int $last = 1000;

    public function next(): string
    {
        return $this->format(++$this->last);
    }

    public function format(int $number): string
    {
        return sprintf('INV-%d', $number);
    }
}
```

针对整个类的切面，保存为 `src/Aspect/TraceInvoiceNumbers.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Aspect;

use Src\Billing\InvoiceNumbers;
use SuperKernel\Aop\Attribute\Aspect;
use SuperKernel\Aop\Contract\AspectInterface;
use SuperKernel\Aop\Contract\ProceedingJoinPointInterface;

#[Aspect(classes: [InvoiceNumbers::class])]
final readonly class TraceInvoiceNumbers implements AspectInterface
{
    public function process(ProceedingJoinPointInterface $point): mixed
    {
        echo "> {$point->getShortMethodName()}", PHP_EOL;

        try {
            return $point->proceed();
        } finally {
            echo "< {$point->getShortMethodName()}", PHP_EOL;
        }
    }
}
```

启动应用并使用该类的脚本，保存为 `bin/numbers.php`：

```php
<?php

declare(strict_types=1);

use Src\Billing\InvoiceNumbers;
use SuperKernel\Context\ApplicationContext;
use SuperKernel\Contract\ApplicationInterface;

require dirname(__DIR__) . '/vendor/autoload.php';

$container = ApplicationContext::getContainer();
$container->get(ApplicationInterface::class);

echo $container->get(InvoiceNumbers::class)->next(), PHP_EOL;
```

构建项目并运行脚本：

```bash
skernel build
php bin/numbers.php
```

它会打印：

```text
> next
> format
< format
< next
INV-1001
```

从 `next()` 到 `format()` 的调用也会经过切面，因为被织入类的每个方法都会分发它的调用。

### 编写通知

连接点描述一次调用：

- `getInstance()` 对实例方法返回对象，对静态方法返回类名，即被调用的类；
- `getArguments()` 返回参数；`getClassReflection()` 和 `getMethodReflection()` 反射原始的类和方法；`getShortMethodName()` 返回方法名；
- `proceed()` 运行原始方法体并返回其结果。

通知决定结果：返回 `proceed()` 的返回值、修改它，或者不调用 `proceed()` 而直接返回一个值以跳过原始方法体。方法声明的返回类型仍会被强制检查。构造函数、析构函数和 `void` 方法的结果会被丢弃，并且在通知运行时 PHP 已经为提升的构造函数属性赋值。`never` 方法的通知不得正常返回。原始方法抛出异常时也必须运行的代码请放在 `finally` 块中，就像 `TraceInvoiceNumbers` 那样。

### 自行织入类

不带参数的 `#[Aspect]` 选中被注解的类本身。这样的类不需要 `AspectInterface`：它像目标一样被织入，在为其调用注册回调之前，调用运行原始方法体。`SuperKernel\Aop\Runtime\Dispatcher::intercept()` 用于注册回调；它接受一个类名、`Class::method` 或 `[Class, method]`，以及一个接收 `SuperKernel\Aop\Runtime\ProceedingJoinPoint` 并返回结果的可调用值。选择器忽略大小写，方法选择器优先于类选择器，再次注册同一个选择器会替换其回调。请在方法被调用之前注册回调。

自我织入的类，保存为 `src/Billing/Clock.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Billing;

use SuperKernel\Aop\Attribute\Aspect;

use function date;

#[Aspect]
final class Clock
{
    public function today(): string
    {
        return date('Y-m-d');
    }
}
```

固定日期的脚本，保存为 `bin/clock.php`：

```php
<?php

declare(strict_types=1);

use Src\Billing\Clock;
use SuperKernel\Aop\Runtime\Dispatcher;
use SuperKernel\Aop\Runtime\ProceedingJoinPoint;
use SuperKernel\Context\ApplicationContext;
use SuperKernel\Contract\ApplicationInterface;

require dirname(__DIR__) . '/vendor/autoload.php';

$container = ApplicationContext::getContainer();
$container->get(ApplicationInterface::class);

Dispatcher::intercept([Clock::class, 'today'], static fn(ProceedingJoinPoint $point): string => '2026-01-01');

echo $container->get(Clock::class)->today(), PHP_EOL;
```

构建项目并运行脚本：

```bash
skernel build
php bin/clock.php
```

它会打印：

```text
2026-01-01
```

### 了解类何时被织入

当容器创建 `ApplicationInterface` 时，`super-kernel/framework` 发布 `BeforeApplicationStart`。本包以优先级 `PHP_INT_MAX` 监听它，并在至少存在一个切面时解析 `AutoLoaderInterface`（这会让 `super-kernel/autoloader` 接管类加载），然后把自己的加载器放在它前面。没有切面时，什么都不会注册。再次发布该事件不会重复注册任何东西，并保留已注册的通知。

从那时起，当 PHP 加载一个类，而该类是目标或会自我织入、尚未加载并且在类映射中有文件时，加载器会：

1. 通过 `super-kernel/mimic` 准备转换后的源码；
2. 通过容器创建其声明选中了该类的切面，并检查每个切面都实现了 `AspectInterface`；其他切面不会被创建；
3. 运行转换后的源码，检查它声明了该类，然后注册这些切面的通知。

其他任何类都交给下一个加载器。前两步中的失败会让该类保持未声明状态，因此可以再次尝试加载它。请求一个已经加载的类不会改变任何东西，包括它的通知。通知是为整个进程注册的：长时间运行的服务器在请求和协程之间共享它，fork 出来的工作进程会继承其父进程在 fork 之前注册的通知。

当容器自己加载目标时（例如为了注入它），加载期间的失败会被报告为容器的 `Unable to resolve method parameter <class>::<method>($<parameter>) of type <target>.`，而不带本包的消息。要看到本包自己的异常，请在启动应用之后直接加载目标，例如使用 `new`。

### 在构建时准备代理

`skernel build` 在项目自己的进程中发布 `SuperKernel\Event\ApplicationBuild`。当至少存在一个切面时，本包随后为类映射中有文件的每个目标准备转换后的源码，而不加载目标、不创建切面，也不注册加载器。打包后的应用（例如由 `skernel package` 构建的应用）会加载这些代理，而不在运行时转换源码；没有预先准备代理的目标会在加载时被转换为一个 `mimic://` 流。解析或转换失败会使构建失败。

代理写入 `vendor/.cache/mimic/<以点分隔的类名>.tpl`，例如 `Src.Billing.PriceCalculator.tpl`；[`super-kernel/mimic`](mimic.zh-CN.md) 页面介绍了该缓存。`SuperKernel\Aop\Mutation\Mutator::getCacheKey()` 标识转换，因此改变生成代码的版本会替换这些代理。进程已经加载的类无法被替换：修改之后请重启长时间运行的工作进程。

### 编写可以被织入的源码

请织入可信的、以声明为主的文件。被选中类的每个具体方法都会被转换；抽象方法保持不变，`final` 和 `private` 也会保留。`SuperKernel\Aop\Mutation\Modifier` 是一个可选的 PHP-Parser 访问器，它移除 `final` 并把 `private` 方法和属性改为 `protected`；默认的转换不使用它。

参数值和提升参数的默认对象会被保留。`__FILE__` 和 `__DIR__` 仍然指向原始源码（本地的或 Phar 归档内的），在类常量、属性初始值和参数默认值中也是如此：生成的代码引用的常量由 `SuperKernel\Aop\Runtime\SourceMetadata` 在类加载之前根据类映射绑定，并以包名和相对于包的路径命名，因此在项目移动或打包之前准备的代理仍然指向实际运行的源码。直接把 `Mutator` 与 `super-kernel/mimic` 一起使用的集成，需要在运行准备好的源码之前对其注入的实例调用 `SourceMetadata::register($class)`。

本包在准备类时会拒绝它无法保留的方法：按引用返回的方法、引用或可变参数、带钩子的提升参数、生成器、静态局部变量，以及方法体中（包括方法体内闭包中）的 `__FUNCTION__`、`__METHOD__` 或 `__LINE__`。

被转换的方法抛出的异常保留其对象，并报告原始文件以及该方法中最后进入的顶层语句的行号；嵌套语句可能报告其外层语句。来自通知和其他代码的异常保留它们自己的位置。栈帧和反射仍然描述生成的代理，并且这些诊断行号在挂起的纤程或 Swoole 协程之间不是隔离的。本包不会替换全局异常处理器。

## 错误

| 情形 | 消息 |
| --- | --- |
| `classes` 不是列表 | `Aspect classes must be a list of class names.` |
| 目标不是有效的类名 | `Aspect targets must be valid class names.` |
| `methods` 的某一项不是非空列表 | `Aspect methods must be a non-empty list.` |
| 方法名不是有效的成员名 | `Method targets must be valid member names.` |
| `propertyHooks` 的某一项没有把属性名映射到钩子列表 | `Aspect property hooks must map property names to hook lists.` |
| 属性名不是有效的成员名 | `Property targets must be valid member names.` |
| 钩子列表为空或不是列表 | `Property hooks must be a non-empty list.` |
| `get` 或 `set` 以外的钩子 | `Aspect property hooks must be get or set.` |
| 带有目标的切面没有实现 `AspectInterface` | `Aspect handler <class> must implement SuperKernel\Aop\Contract\AspectInterface.` |
| 加载带有属性钩子声明的目标 | `AOP property hook interception is not implemented.` |
| 方法按引用返回 | `AOP does not support methods returning by reference.` |
| 引用、可变或带钩子的提升参数 | `AOP does not support reference, variadic, or hooked promoted parameters.` |
| 生成器、静态局部变量，或 `__FUNCTION__`、`__METHOD__`、`__LINE__` | `AOP cannot preserve generators, static locals, or method/line magic constants.` |
| 加载目标时再次需要同一个目标；通过容器时表现为无法解析的参数 | `Recursive instrumentation of <class>.` |
| 类映射中的文件没有声明该类 | `The mapped source did not declare <class>.` |
| 存在切面，但没有任何包提供 `AutoLoaderInterface` | 容器的 `Unable to resolve method parameter SuperKernel\Aop\AutoLoader::__construct($autoLoader) of type SuperKernel\Contract\AutoLoaderInterface.` |
| 传给 `Dispatcher::intercept()` 的选择器为空或格式错误 | `An interceptor selector must be a non-empty name.` 或 `A method selector must contain a class name and method name.` |

声明错误是在 `skernel build` 读取注解时抛出的 `InvalidArgumentException`，构建会跳过该类并报告它，因此该切面不存在。其他错误在准备或加载类时抛出：`InvalidArgumentException`、`LogicException` 或 `RuntimeException`。缺少 `AutoLoaderInterface` 的提供者会在织入任何类之前使 `skernel build` 和应用启动失败；安装 `super-kernel/autoloader` 即可解决。

## 与其他包的关系

- [`super-kernel/mimic`](mimic.zh-CN.md) 准备并缓存转换后的源码；本包提供它所用的转换。
- [`super-kernel/autoloader`](autoloader.zh-CN.md) 提供 `AutoLoaderInterface` 和类映射；本包建议（suggest）但并不依赖它，因此只要存在一个切面，就请与本包一起安装它。
- [`super-kernel/framework`](framework.zh-CN.md) 发布开始织入的 `BeforeApplicationStart`；[`super-kernel/event`](event.zh-CN.md) 声明它和 `ApplicationBuild`。
- [`super-kernel/di`](di.zh-CN.md) 创建切面；[`super-kernel/annotator`](annotator.zh-CN.md) 提供记录下来的 `#[Aspect]` 声明；[`super-kernel/metadata`](metadata.zh-CN.md) 提供用于命名源码常量的包位置。
- `skernel build` 收集切面并准备代理；`skernel run` 在启动应用之前先构建。

## 不要这样做

**不要在应用启动之前使用目标。** 织入在容器创建 `ApplicationInterface` 时开始。只在容器中查找服务的脚本，或者更早加载目标的代码，得到的是原始类，它的切面永远不会运行，并且没有任何错误。请先启动应用（就像 `bin/numbers.php` 那样），或者在命令中运行这些代码。

**不要织入本包无法保留的方法。** 被选中的类中如果有生成器、静态局部变量、`__FUNCTION__`、`__METHOD__` 或 `__LINE__`、引用或可变参数，或者按引用返回的方法，会使 `skernel build` 失败，例如报告 `AOP cannot preserve generators, static locals, or method/line magic constants.` 请把这些方法移到没有切面选中的类中，或者用 `methods` 缩小切面的范围。

**暂时不要声明属性钩子。** 该注解接受 `propertyHooks`，但加载这样的目标会抛出 `AOP property hook interception is not implemented.`，因此该类完全无法使用。请改为选中它的方法。

**不要让切面依赖它自己的目标。** 切面是在其目标加载期间创建的，因此构造函数（直接或通过其他服务）需要该目标的切面无法被创建：加载目标会以容器的 `Unable to resolve method parameter <aspect>::__construct($<parameter>) of type <target>.` 失败。请改用连接点的 `getInstance()` 返回的实例。

**不要指望同一个方法的两个切面都会运行。** 对于同一个方法或类选择器，构建最后发现的切面会替换之前的。请把通知合并到一个切面中，或者选中不同的方法。

**不要指望格式错误的 `#[Aspect]` 会在运行时失败。** `skernel build` 会报告声明错误并跳过该类，因此应用启动时没有该切面。修改声明之后，请检查构建的报告。
