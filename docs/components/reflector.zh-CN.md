<p align="right">
  <a href="reflector.md">English</a> | <strong>中文文档</strong>
</p>

# SuperKernel Reflector

包：[`super-kernel/reflector`](https://github.com/super-kernel/reflector) · [Packagist](https://packagist.org/packages/super-kernel/reflector) · [文档索引](../README.zh-CN.md)

## 用途

`super-kernel/reflector` 为应用的所有包提供一个获取 PHP 反射对象的统一位置，并保留这些对象，使反复检查同一些类的容器、配置和其他代码不必再次反射它们。结果是 PHP 自己的 `Reflection*` 对象，带有完整的 API；名称、自动加载和失败的行为都与 PHP 一致。

本包实现了 `SuperKernel\Contract\ReflectorInterface`。它的缓存属于每个反射器实例，不会在进程之间保留。

## 能力

### 自行创建反射器

`new SuperKernel\Reflector\Reflector()` 创建一个缓存为空的反射器。在同一个反射器中，对同一个符号的重复请求会返回同一个对象，无论使用 PHP 接受的哪种写法：类别名、开头的反斜杠、类名或方法名的不同大小写，或者 `reflectMethod()` 的单参数 `Class::method` 形式。另一个反射器有它自己的缓存，返回的是其他对象。

项目中的一个类，保存为 `src/Billing/Invoice.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Billing;

final class Invoice
{
    public function __construct(public readonly string $number)
    {
    }

    public function total(): int
    {
        return 0;
    }
}
```

用多种写法反射同一个方法的脚本，保存为 `bin/reflect.php`：

```php
<?php

declare(strict_types=1);

use Src\Billing\Invoice;
use SuperKernel\Reflector\Reflector;

require dirname(__DIR__) . '/vendor/autoload.php';

$reflector = new Reflector();
$method = $reflector->reflectMethod(Invoice::class, 'total');

echo $method->getReturnType(), PHP_EOL;
var_dump($method === $reflector->reflectMethod('Src\Billing\Invoice::total'));
var_dump($method === $reflector->reflectMethod('\src\billing\INVOICE', 'TOTAL'));
var_dump($method === new Reflector()->reflectMethod(Invoice::class, 'total'));
```

构建项目并运行脚本：

```bash
skernel build
php bin/reflect.php
```

它会打印：

```text
int
bool(true)
bool(true)
bool(false)
```

### 反射各种符号

| 方法 | 返回 |
| --- | --- |
| `reflectClass($class)` | 类、接口、trait 或枚举的 `ReflectionClass`，按名称或对象 |
| `reflectClassConstant($class, $constant)` | `ReflectionClassConstant` |
| `reflectConstant($name)` | 全局常量的 `ReflectionConstant` |
| `reflectEnum($enum)` | `ReflectionEnum`，按名称或枚举成员 |
| `reflectEnumUnitCase($enum, $case)` | `ReflectionEnumUnitCase`；对于回退枚举返回 `ReflectionEnumBackedCase` |
| `reflectEnumBackedCase($enum, $case)` | `ReflectionEnumBackedCase` |
| `reflectExtension($name)` | 已加载 PHP 扩展的 `ReflectionExtension` |
| `reflectZendExtension($name)` | 已加载 Zend 扩展的 `ReflectionZendExtension` |
| `reflectFunction($function)` | 函数名或闭包的 `ReflectionFunction` |
| `reflectMethod($class, $method)` | `ReflectionMethod`；也支持 `reflectMethod('Class::method')` |
| `reflectParameter($callable, $parameter)` | 按位置或名称的 `ReflectionParameter`，适用于闭包、可调用对象、函数名或方法名，以及 `[class, method]` 数组 |
| `reflectObject($object)` | `ReflectionObject` |
| `reflectProperty($class, $property)` | `ReflectionProperty` |
| `reflectGenerator($generator)` | `ReflectionGenerator` |
| `reflectFiber($fiber)` | `ReflectionFiber` |
| `reflectReference($array, $key)` | `ReflectionReference`；元素不是引用时为 `null` |

`ReflectionAttribute` 对象和 `ReflectionType` 的子类来自拥有它们的反射对象，例如 `getAttributes()` 或 `getType()`，因为 PHP 没有提供直接创建它们的方法。

一个枚举，保存为 `src/Billing/Status.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Billing;

enum Status: string
{
    case Open = 'open';
    case Paid = 'paid';
}
```

反射多种符号的脚本，保存为 `bin/kinds.php`：

```php
<?php

declare(strict_types=1);

use Src\Billing\Invoice;
use Src\Billing\Status;
use SuperKernel\Reflector\Reflector;

require dirname(__DIR__) . '/vendor/autoload.php';

$reflector = new Reflector();

echo $reflector->reflectEnumUnitCase(Status::class, 'Paid')::class, PHP_EOL;
echo $reflector->reflectEnumBackedCase(Status::Open, 'Open')->getBackingValue(), PHP_EOL;
echo $reflector->reflectProperty(Invoice::class, 'number')->getType(), PHP_EOL;
echo $reflector->reflectParameter([Invoice::class, '__construct'], 'number')->getPosition(), PHP_EOL;
echo $reflector->reflectFunction('strlen')->getNumberOfParameters(), PHP_EOL;

$values = ['plain' => 1];
$values['shared'] = &$values['plain'];

var_dump($reflector->reflectReference($values, 'shared') !== null);
var_dump($reflector->reflectReference(['plain' => 1], 'plain'));
```

运行 `skernel build` 之后，`php bin/kinds.php` 会打印：

```text
ReflectionEnumBackedCase
open
string
0
1
bool(true)
NULL
```

### 依靠缓存

- 名称遵循 PHP 的规则：类名、方法名、函数名和扩展名的匹配不区分大小写，类别名会得到它所指向的类的反射。属性、常量、参数和枚举成员的名称按原样精确匹配。
- 闭包、可调用对象、对象、生成器和纤程的反射以弱键保存：它们不会使这些对象保持存活，并会随对象一起消失。
- 引用按 `ReflectionReference::getId()` 保存。不是引用的元素返回 `null`，不会在缓存中留下任何内容。
- 缓存与它的反射器同生命周期。容器或门面会在进程的整个生命周期内保留它的反射器及其缓存。PHP 在一个进程内不能重新声明类或函数，因此反射永远不会过期；进程之间不保留任何内容。
- PHP 会拒绝的名称在这里同样被拒绝，抛出 PHP 的 `ReflectionException`；它永远不会变成缓存命中。

### 从容器接收反射器

`SuperKernel\Reflector\Provider\ReflectorProvider` 带有 `#[FactoryProvider(ReflectorInterface::class)]`。每次调用都会创建一个新的 `Reflector`，让门面使用它，并返回它。`super-kernel/di` 在创建根容器时调用它一次，并把结果保存在 `ReflectorInterface` 和 `Reflector` 之下，因此各个服务共享同一个缓存。请把 `ReflectorInterface` 作为构造函数参数接收。

接收反射器的服务，保存为 `src/Billing/InvoiceInspector.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Billing;

use ReflectionMethod;
use SuperKernel\Contract\ReflectorInterface;

final readonly class InvoiceInspector
{
    public function __construct(private ReflectorInterface $reflector)
    {
    }

    public function totalMethod(): ReflectionMethod
    {
        return $this->reflector->reflectMethod(Invoice::class, 'total');
    }
}
```

把它与门面进行比较的脚本，保存为 `bin/inspect.php`：

```php
<?php

declare(strict_types=1);

use Src\Billing\Invoice;
use Src\Billing\InvoiceInspector;
use SuperKernel\Context\ApplicationContext;
use SuperKernel\Reflector\Facade\ReflectorFacade;

require dirname(__DIR__) . '/vendor/autoload.php';

$method = ApplicationContext::getContainer()->get(InvoiceInspector::class)->totalMethod();

echo $method->class, '::', $method->getName(), PHP_EOL;
var_dump($method === ReflectorFacade::reflectMethod(Invoice::class, 'total'));
```

运行 `skernel build` 之后，`php bin/inspect.php` 会打印：

```text
Src\Billing\Invoice::total
bool(true)
```

### 使用静态门面

`SuperKernel\Reflector\Facade\ReflectorFacade` 以静态调用的形式提供 `ReflectorInterface` 的每个方法，支持位置参数和具名参数，并把调用转发给最近一次 `ReflectorProvider` 调用所创建的反射器，共享该反射器的缓存。结果和失败都原样传出。

- 在进程中运行 `ReflectorProvider` 之前，每次调用都抛出 `LogicException`。门面没有设置方法；只有提供者能设置它的反射器。在应用中，创建根容器时会运行该提供者。
- 每次调用提供者都会替换门面的反射器；之前创建的反射器保留它们自己的缓存。
- `ReflectorInterface` 未声明的方法会作为反射器的未定义方法而失败。

优先使用注入；只在无法接收反射器的代码中（例如静态辅助方法）使用门面。

不使用容器而使用门面的脚本，保存为 `bin/facade.php`：

```php
<?php

declare(strict_types=1);

use Src\Billing\Invoice;
use SuperKernel\Reflector\Facade\ReflectorFacade;
use SuperKernel\Reflector\Provider\ReflectorProvider;

require dirname(__DIR__) . '/vendor/autoload.php';

try {
    ReflectorFacade::reflectClass(Invoice::class);
} catch (LogicException $exception) {
    echo $exception->getMessage(), PHP_EOL;
}

$reflector = new ReflectorProvider()();

var_dump(ReflectorFacade::reflectClass(Invoice::class) === $reflector->reflectClass(Invoice::class));
```

运行 `skernel build` 之后，`php bin/facade.php` 会打印：

```text
ReflectorFacade requires initialization through ReflectorProvider.
bool(true)
```

## 错误

| 情形 | 消息 |
| --- | --- |
| 在 `ReflectorProvider` 运行之前调用门面（`LogicException`） | `ReflectorFacade requires initialization through ReflectorProvider.` |
| 符号不存在（PHP 自己的 `ReflectionException`） | PHP 的消息，例如 `Class "<class>" does not exist` |

## 与其他包的关系

- [`super-kernel/contract`](contract.zh-CN.md) 声明了 `ReflectorInterface`；请面向它编写代码。
- [`super-kernel/attribute`](attribute.zh-CN.md) 声明了提供者上的 `#[FactoryProvider]` 注解。
- [`super-kernel/di`](di.zh-CN.md) 在创建根容器时调用该提供者，保留反射器，并用它检查构造函数、工厂和属性。
- 其他包（例如 [`super-kernel/config`](config.zh-CN.md)）从容器接收同一个反射器。

## 不要这样做

**不要在提供者运行之前调用门面。** 它会抛出 `ReflectorFacade requires initialization through ReflectorProvider.`。在应用中，请先获取容器，或者通过注入接收 `ReflectorInterface`。

**不要每次调用都创建一个反射器。** 每个 `new Reflector()` 都从空缓存开始，因此反射会被重新构建，本应相同的对象也不再相同。请接收容器的反射器，或者保留一个实例。

**不要为了获取共享的反射器而再次调用 `ReflectorProvider`。** 每次调用都会创建一个缓存为空的新反射器，并让门面改用它，而不再使用容器保留的那个。请从容器获取 `ReflectorInterface`。
