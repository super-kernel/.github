<p align="right">
  <a href="autoloader.md">English</a> | <strong>中文文档</strong>
</p>

# SuperKernel Autoloader

包：[`super-kernel/autoloader`](https://github.com/super-kernel/autoloader) · [Packagist](https://packagist.org/packages/super-kernel/autoloader) · [文档索引](../README.zh-CN.md)

## 用途

`super-kernel/autoloader` 为进程提供一个类映射已知且固定的类加载器。需要在类加载之前看到它的代码（例如 `super-kernel/aop` 在把切面织入某个类时）通过这个加载器读取待加载的类映射并拦截加载。长时间运行的运行时可以在所有类加载完成之后移除该加载器。

类映射来自 `skernel build` 生成、由 `super-kernel/metadata` 恢复的元数据；加载器不读取 Composer 文件。它替换 Composer 的加载器，而不是与之协作。

## 能力

### 接管类加载

从 SuperKernel 容器解析 `SuperKernel\Contract\AutoLoaderInterface`。它的提供者 `SuperKernel\Autoloader\Provider\AutoLoaderProvider` 随后会：

1. 按 `PackageManagerInterface::all()` 的顺序（根包在前）合并各个包的类映射；两个包映射了同一个类时，排在前面的包胜出，因此根包优先；
2. 在之前的加载器仍然可用时创建新的加载器；在此期间，已映射的类通过直接引入其文件来加载，文件缺失时报出 PHP 自身的错误；
3. 移除所有已注册的自动加载器，并只注册新的加载器。

接管是最终的：之前的加载器（包括 Composer 的）永远不会被恢复。如果第 2 步失败，提供者会恢复之前的加载器，异常向外传播；已经加载的类及其文件产生的影响仍然保留。

每次解析都会创建一个新的加载器，由容器保留；加载器本身不保留静态实例。在会派生工作进程的长时间运行的运行时中，请在每个工作进程中解析加载器。安装了 `super-kernel/aop` 时，应用会在启动时解析它。

项目中的一个类，保存为 `src/Billing/Invoice.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Billing;

final class Invoice
{
}
```

接管加载的脚本，保存为 `bin/loader.php`：

```php
<?php

declare(strict_types=1);

use Src\Billing\Invoice;
use SuperKernel\Context\ApplicationContext;
use SuperKernel\Contract\AutoLoaderInterface;

require dirname(__DIR__) . '/vendor/autoload.php';

$loader = ApplicationContext::getContainer()->get(AutoLoaderInterface::class);

var_dump($loader->isRegistered(), count(spl_autoload_functions()));

$loader->addInterceptor(static function (string $class, string $file): bool {
    echo 'Loading ', $class, ' from ', basename($file), PHP_EOL;

    return false;
});

new Invoice();

var_dump(isset($loader->getClassMap()[Invoice::class]));
```

构建项目并运行脚本：

```bash
skernel build
php bin/loader.php
```

它会打印：

```text
bool(true)
int(1)
Loading Src\Billing\Invoice from Invoice.php
bool(false)
```

### 读取待加载的类映射

`getClassMap()` 返回尚未加载的已映射类，即从类名到文件路径的映射。

- 映射在加载器创建时就已确定。任何代码都不能添加或替换映射，修改你传入的数组或你拿到的数组也不会改变加载器。
- 创建加载器时已经声明的类、接口、trait 或枚举不会包含在内。
- 每当一个文件或拦截器运行之后，所有现在已经声明的已映射类都会离开映射，包括同一个文件声明的其他类。
- 路径按给定的原样保留；相对路径取决于 PHP 的当前工作目录，因此在长时间运行的进程中请使用绝对路径。`skernel build` 的映射使用绝对路径。
- 映射文件缺失的类不会被加载，并且留在映射中。不会询问其他加载器。
- 不在映射中的类根本不会被加载。

### 拦截加载

`addInterceptor($interceptor)` 添加一个可调用对象，它在包含映射文件之前接收被请求的类及其映射文件，并返回一个正整数标识符。拦截器只对映射中的类运行，因为只有这些类才有文件。

- 拦截器按添加顺序运行。返回 `false` 会把该类交给下一个拦截器，最后交给映射文件。
- 返回 `true` 会停止其余的拦截器和映射文件。此时拦截器必须自己声明该类；如果没有声明，该类会留在待加载映射中。
- `removeInterceptor($id)` 移除一个拦截器并返回 `true`，对未知的标识符返回 `false`。
- 拦截器属于加载器实例；添加或移除拦截器从不改变已注册的自动加载器。
- 在加载过程中再次请求同一个类时，不会第二次加载它。

一个要被拦截的类，保存为 `src/Billing/Receipt.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Billing;

final class Receipt
{
}
```

带有一个自行加载该类的拦截器的脚本，保存为 `bin/intercept.php`：

```php
<?php

declare(strict_types=1);

use Src\Billing\Receipt;
use SuperKernel\Context\ApplicationContext;
use SuperKernel\Contract\AutoLoaderInterface;

require dirname(__DIR__) . '/vendor/autoload.php';

$loader = ApplicationContext::getContainer()->get(AutoLoaderInterface::class);

$interceptorId = $loader->addInterceptor(static function (string $class, string $file): bool {
    if (Receipt::class !== $class) {
        return false;
    }

    echo 'Loading ', $class, ' myself', PHP_EOL;
    require $file;

    return true;
});

new Receipt();

var_dump($loader->removeInterceptor($interceptorId));
var_dump($loader->removeInterceptor($interceptorId));
```

运行 `skernel build` 之后，`php bin/intercept.php` 会打印：

```text
Loading Src\Billing\Receipt myself
bool(true)
bool(false)
```

### 安全地移除加载器

长时间运行的运行时可以在开始提供服务之前加载所有类，然后移除加载器。`unregister()`：

- 只要还有已映射的类尚未加载，就抛出 `SuperKernel\Autoloader\Exception\UnloadedClassMapException`，并且不做任何更改。该异常的 `classMap()` 返回待加载的映射，`firstClassName()` 返回其中的第一个类，消息中也会给出这个类；
- 否则只移除这个加载器并清空它的拦截器。其他代码在接管之后注册的加载器保持不变；被接管移除的加载器不会回来，因此之后自动加载器栈可能为空。

`isRegistered()` 报告这个加载器当前是否已注册。

过早尝试移除加载器的脚本，保存为 `bin/unregister.php`：

```php
<?php

declare(strict_types=1);

use SuperKernel\Autoloader\Exception\UnloadedClassMapException;
use SuperKernel\Context\ApplicationContext;
use SuperKernel\Contract\AutoLoaderInterface;

require dirname(__DIR__) . '/vendor/autoload.php';

$loader = ApplicationContext::getContainer()->get(AutoLoaderInterface::class);

try {
    $loader->unregister();
} catch (UnloadedClassMapException $exception) {
    echo $exception->getMessage(), PHP_EOL;
    var_dump($exception->firstClassName() === array_key_first($loader->getClassMap()));
}

var_dump($loader->isRegistered());
```

运行 `skernel build` 之后，`php bin/unregister.php` 会打印项目中第一个待加载的类：

```text
Class "Src\Billing\Invoice" has not been loaded.
bool(true)
bool(true)
```

### 自行创建加载器

`new SuperKernel\Autoloader\AutoLoader($classMap)` 检查并准备一个类映射，但不注册任何内容；只有提供者才会接管。加载器是可调用的，因此用类名调用它就会加载该类，它也可以像任何加载器一样用 `spl_autoload_register()` 注册。这在测试中很有用。

通过自己的加载器加载一个类的脚本，保存为 `bin/standalone.php`：

```php
<?php

declare(strict_types=1);

use Src\Billing\Receipt;
use SuperKernel\Autoloader\AutoLoader;

require dirname(__DIR__) . '/vendor/autoload.php';

$loader = new AutoLoader([Receipt::class => dirname(__DIR__) . '/src/Billing/Receipt.php']);

var_dump($loader->isRegistered());

$loader(Receipt::class);

var_dump(class_exists(Receipt::class, false));
var_dump($loader->getClassMap());

$loader->unregister();
```

`php bin/standalone.php` 会打印：

```text
bool(false)
bool(true)
array(0) {
}
```

## 错误

本包的每个异常都实现了 `SuperKernel\Autoloader\Exception\ClassLoaderExceptionInterface`。面向 `AutoLoaderInterface` 编写的代码可以把 `unregister()` 的失败当作 `RuntimeException` 捕获。

| 情形 | 消息 |
| --- | --- |
| 映射中的类名不是不含空字节的非空字符串（`InvalidClassMapException`） | `Classmap keys must be non-empty strings without null bytes; received <type>.` |
| 映射中的路径为空或包含空字节（`InvalidClassMapException`） | `The classmap path for "<class>" must be a non-empty string without null bytes.` |
| 仍有待加载的已映射类时调用 `unregister()`（`UnloadedClassMapException`） | `Class "<class>" has not been loaded.` |
| PHP 拒绝移除该加载器（`ClassLoaderRegistrationException`） | `Unable to unregister the class loader from SPL.` |

## 与其他包的关系

- [`super-kernel/contract`](contract.zh-CN.md) 声明了 `AutoLoaderInterface`；请面向它编写代码。
- [`super-kernel/metadata`](metadata.zh-CN.md) 从 `skernel build` 的元数据恢复各个包的类映射。
- [`super-kernel/attribute`](attribute.zh-CN.md) 声明了提供者上的 `#[FactoryProvider]` 注解，[`super-kernel/di`](di.zh-CN.md) 解析并保留加载器。
- [`super-kernel/aop`](aop.zh-CN.md) 在应用启动时解析加载器，读取它的待加载类映射，并在本加载器引入之前加载它要织入的类。
- `skernel build` 生成类映射；添加、删除或移动类之后请重新运行它。

## 不要这样做

**不要期望其他加载器会加载类映射遗漏的类。** 接管之后不再注册其他任何自动加载器：上一次 `skernel build` 之后新增的类，或者文件已被删除的类，都会找不到。修改类之后请运行 `skernel build`，或者用 `skernel run` 启动。

**不要在仍有待加载的类时移除加载器。** `unregister()` 会以 `Class "<class>" has not been loaded.` 拒绝，并保留加载器。请先加载进程将要用到的所有类，或者保持加载器处于注册状态。

**不要在多个工作进程之间共享同一个加载器。** 请在长时间运行的运行时的每个工作进程中解析 `AutoLoaderInterface`，使每个进程拥有自己的加载器和待加载类映射。
