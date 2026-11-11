<p align="right">
  <a href="mimic.md">English</a> | <strong>中文文档</strong>
</p>

# SuperKernel Mimic

包：[`super-kernel/mimic`](https://github.com/super-kernel/mimic) · [Packagist](https://packagist.org/packages/super-kernel/mimic) · [文档索引](../README.zh-CN.md)

## 用途

`super-kernel/mimic` 把一个 PHP 源文件变成 PHP 可以加载的转换后的副本，并为下一个进程保留这份副本。一个转换（即 `SuperKernel\Mimic\Contract\MutatorInterface`）重写源码；本包读取文件的一份快照，应用该转换，并返回可以加载结果的位置：依赖目录（vendor 目录）缓存中的代理文件，或者在无法这样做时，一个只读的 `mimic://` 流。

本包不决定转换哪些文件，不注册自动加载器，也不运行它准备的代码；这些由使用它的集成负责。`super-kernel/aop` 就是这样的集成：它提供织入方法的转换，并加载准备好的类。

## 能力

### 准备源文件

`SuperKernel\Mimic\MimicFactory` 由容器提供。`prepare($sourcePath)` 读取文件的一份快照（本地路径或 Phar 归档内的路径），并返回一个可加载的地址。它先尝试文件缓存，然后回退到流。用 `require` 加载该地址；在此之前，文件中的任何内容都没有运行。

该工厂需要三个服务：用于缓存位置的 `SuperKernel\Contract\VendorDirInterface`、用于类名和包位置的 `SuperKernel\Contract\PackageManagerInterface`（两者都从 `skernel build` 的元数据中恢复），以及一个 `MutatorInterface`。`super-kernel/aop` 绑定了它自己的转换；没有它时，请自行声明一个。源文件必须是可信的 PHP。

一个带有占位符的类，保存为 `src/Billing/Edition.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Billing;

final class Edition
{
    public const string NAME = '%EDITION%';
}
```

填充占位符的转换，保存为 `src/Mimic/FillEdition.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Mimic;

use SuperKernel\Attribute\Provider;
use SuperKernel\Mimic\Contract\MutatorInterface;

use function str_replace;

#[Provider(MutatorInterface::class)]
final readonly class FillEdition implements MutatorInterface
{
    public function getCacheKey(): string
    {
        return 'fill-edition-1';
    }

    public function mutate(string $code, ?string $sourcePath = null): string
    {
        return str_replace('%EDITION%', 'Community', $code);
    }
}
```

准备并加载该类的脚本，保存为 `bin/edition.php`：

```php
<?php

declare(strict_types=1);

use Src\Billing\Edition;
use SuperKernel\Context\ApplicationContext;
use SuperKernel\Contract\BasePathInterface;
use SuperKernel\Mimic\MimicFactory;

require dirname(__DIR__) . '/vendor/autoload.php';

$container = ApplicationContext::getContainer();
$source = $container->get(BasePathInterface::class) . '/src/Billing/Edition.php';

$address = $container->get(MimicFactory::class)->prepare($source);

require $address;

echo basename($address), PHP_EOL;
echo Edition::NAME, PHP_EOL;
```

构建项目，然后运行脚本两次：

```bash
skernel build
php bin/edition.php
php bin/edition.php
```

它会打印：

```text
Src.Billing.Edition.tpl
Community
Src.Billing.Edition.tpl
Community
```

第二次运行重用了第一次写入的代理。

### 重用代理缓存

代理直接写入 `vendor/.cache/mimic/<class-name>.tpl`，其中类名是文件所声明的类的完全限定名称，命名空间分隔符替换为点，因此 `Acme\Service\Test` 和 `Acme\Service_Test` 保持区分。包管理器在不加载文件的情况下提供该名称；当一个文件声明了多个映射的类时，按排序后的类映射中的第一个名称为代理命名。新代理和已缓存代理返回的地址相同。

代理的第一行是 `<?php // mimic-cache-sha256: <digest>`。摘要覆盖源码的位置、转换的缓存键以及源码内容的哈希。位置是所属包的名称加上文件在该包内的路径，因此项目移动或被打包进 Phar 归档之后代理仍然有效；原始路径从不写入代理。从不比较修改时间、inode 或大小。

- 匹配的代理会被重用，不写入、不创建目录，也不需要流协议。
- 内容变化、文件在包内移动或缓存键变化，都会替换同一个代理。
- 新代理先写入缓存目录中的临时文件，再重命名到位，不使用锁文件。并发的进程可能把同一个文件转换两次，但没有读取方会看到不完整的代理；替换之后，如果 OPcache 可用则会使其失效。
- 准备好的文件路径会跟随之后的替换；当你需要永不改变的快照时，请保留 `mimic://` 地址。
- 本包从不读取、重命名或删除缓存中的其他文件，例如旧的哈希目录、`.php` 或 `.php.tpl` 代理，或 `.lock` 文件。

每个类在一个依赖目录中只有一个代理，因此共享该缓存的每个集成都必须使用相同的源码位置和转换。

### 回退到流

当缓存无法创建、读取、写入或替换，或者文件不属于项目的包时，`prepare()` 返回一个 `mimic://` 地址。替换失败时保留已有的代理，第一行无法读取或无法识别的代理保持原样。流协议在第一次需要时注册；当另一个库已经占用 `mimic` 时，回退会失败。流地址包含源码路径、内容哈希和缓存键，因此准备另一个修订版本或转换永远不会改变之前的流。流存在于当前进程的内存中。

准备一个位于项目的包之外的文件的脚本，保存为 `bin/snapshot.php`：

```php
<?php

declare(strict_types=1);

use SuperKernel\Context\ApplicationContext;
use SuperKernel\Mimic\MimicFactory;

require dirname(__DIR__) . '/vendor/autoload.php';

$source = sys_get_temp_dir() . '/edition-banner.php';
file_put_contents($source, "<?php\n\nreturn 'Edition: %EDITION%';\n");

$address = ApplicationContext::getContainer()->get(MimicFactory::class)->prepare($source);

echo strtok($address, ':'), PHP_EOL;
echo require $address, PHP_EOL;
```

运行脚本：

```bash
php bin/snapshot.php
```

它会打印：

```text
mimic
Edition: Community
```

### 编写转换

`MutatorInterface` 有两个方法：

- `mutate($code, $sourcePath)` 返回完整的新源码，以 `<?php` 开头；它抛出的异常原样从 `prepare()` 向外传播；
- `getCacheKey()` 返回一个非空字符串，标识该转换、它的修订版本以及影响其输出的每项设置。等价的实例返回相同的键；每当相同的源码必须生成不同的代码时，请修改它。

输出只能通过源码相对于包的位置依赖源码所在的位置。请在代码运行时绑定绝对路径，而不是把它们写进输出，否则项目移动或打包之后代理就不再与源码匹配。两个驱动从容器接收同一个转换。本包在运行时不依赖 PHP-Parser；需要解析 PHP 的转换自行依赖其解析器。

### 直接使用驱动

`MimicFactory` 组合了两个实现 `SuperKernel\Mimic\Contract\MimicDriverInterface` 的驱动，其 `prepare()` 接受一个 `SuperKernel\Mimic\Source\SourceFile`。`SourceFile` 保存规范化的本地路径（Phar 路径按原样保存）、源码字节及其 SHA-256 哈希。`SuperKernel\Mimic\Driver\FileDriver` 返回代理路径，在缓存无法使用时返回 `null`；`SuperKernel\Mimic\Driver\StreamDriver` 返回 `mimic://` 地址，或者抛出异常。

### 恢复异常位置

`SuperKernel\Mimic\Diagnostics\ExceptionPatcher::patch($throwable, $originalFile, $originalLine)` 把从生成代码抛出的异常的文件和行号设置为原始位置，保持同一个异常对象，并返回它。本包自己从不替换全局异常处理器。

### 加载修改后的代码

进程已经加载的类无法被替换或卸载。请重启长时间运行的工作进程以加载新准备的代码，并在修改类或包之后重新运行 `skernel build`，使类名和包位置以及构建时准备的代理与源码一致。

## 错误

| 情形 | 消息 |
| --- | --- |
| 源码路径为空、包含空字节或不是可读的文件 | `The source file is not readable: <path>` |
| 无法注册 `mimic` 协议 | `Unable to register the mimic stream protocol.` |
| 另一个库占用了 `mimic` 协议 | `The mimic stream protocol is already registered by another library.` |

第一种是 `InvalidArgumentException`，其他的是 `RuntimeException`。转换抛出的异常（包括解析错误）原样向外传播。

## 与其他包的关系

- [`super-kernel/aop`](aop.zh-CN.md) 把它的 `Mutator` 绑定到 `MutatorInterface`，并通过本包加载它织入的类；`skernel build` 准备它们的代理。
- [`super-kernel/metadata`](metadata.zh-CN.md) 从构建元数据中提供 `VendorDirInterface` 和 `PackageManagerInterface`；[`super-kernel/contract`](contract.zh-CN.md) 声明它们。
- [`super-kernel/di`](di.zh-CN.md) 创建 `MimicFactory` 和两个驱动。
- 本包在其所有者授权下移植了 [wheakerd/mimic](https://github.com/wheakerd/mimic) 的修订版本 `443ad3e4a2c8dd9ce7b6028b0b7aea163be4e47d`：Composer 包 `wheakerd/php-mimic` 变为 `super-kernel/mimic`，命名空间 `PhpMimic\` 变为 `SuperKernel\Mimic\`。

## 不要这样做

**输出改变时不要保留原来的缓存键。** 源码和键都没有变化的代理会被原样重用，因此改变了输出却没有改变 `getCacheKey()` 的转换会继续加载旧代码。每个改变生成代码的修订版本都要修改键。

**不要把绝对路径写进生成的代码。** 只要相对于包的位置匹配，代理就会被重用，包括项目移动之后或从 Phar 归档运行时，因此嵌入的路径会指向一个已经不存在的位置。请在代码运行时绑定路径，就像 `super-kernel/aop` 用它的源码常量所做的那样。

**不要让不同的转换共享同一个缓存。** 每个类在一个依赖目录中只有一个代理，因此用不同转换准备同一个类的两个集成会在每次运行时互相替换对方的代理。请每个缓存只使用一个转换，或者使用 `StreamDriver` 获取独立的快照。
