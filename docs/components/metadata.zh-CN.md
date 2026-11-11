<p align="right">
  <a href="metadata.md">English</a> | <strong>中文文档</strong>
</p>

# SuperKernel Metadata

包：[`super-kernel/metadata`](https://github.com/super-kernel/metadata) · [Packagist](https://packagist.org/packages/super-kernel/metadata) · [文档索引](../README.zh-CN.md)

## 用途

`super-kernel/metadata` 告诉应用它安装在哪里、由哪些包组成，启动时无需解析 Composer 文件或扫描源码。`skernel build` 一次性把各个包、它们的环境要求和类映射写到依赖目录（vendor 目录）之下；本包恢复这些内容，检查它们是否仍与项目的 `composer.lock` 一致，并通过共享的 `SuperKernel\Contract` 接口提供它们。

收集元数据由 `skernel build` 负责；自动加载仍由 Composer 的 `vendor/autoload.php` 负责。`skernel build` 写在元数据旁边的注解记录属于 `super-kernel/annotator`。

## 能力

### 恢复元数据

三个提供者负责恢复元数据。每个提供者都带有 `#[FactoryProvider]` 注解，因此 SuperKernel 容器通过它们解析这些接口；在容器之外，请按以下顺序调用它们：

1. `new VendorDirProvider()()` 返回第一个加载本包的已注册 Composer 类加载器所对应的 `SuperKernel\Contract\VendorDirInterface`。本地目录会被规范化；`phar://` 路径保持注册时的样子。
2. `new BasePathProvider()($vendorDir)` 返回 `SuperKernel\Contract\BasePathInterface`：构建时记录、相对于依赖目录解析出的项目目录。
3. `new PackageManagerProvider()($vendorDir, $basePath)` 用项目的 `composer.lock` 核对构建戳，并返回 `SuperKernel\Contract\PackageManagerInterface`。

每次调用都会重新读取文件并返回一个新值。容器为每个接口保留一个实例；`super-kernel/di` 在创建根容器时、容器尚不存在之前自行调用这些提供者。

项目中的一个类，保存为 `src/Billing/Invoice.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Billing;

final class Invoice
{
}
```

恢复元数据的脚本，保存为 `bin/packages.php`：

```php
<?php

declare(strict_types=1);

use Src\Billing\Invoice;
use SuperKernel\Metadata\Provider\BasePathProvider;
use SuperKernel\Metadata\Provider\PackageManagerProvider;
use SuperKernel\Metadata\Provider\VendorDirProvider;

require dirname(__DIR__) . '/vendor/autoload.php';

$vendorDir = new VendorDirProvider()();
$basePath = new BasePathProvider()($vendorDir);
$packages = new PackageManagerProvider()($vendorDir, $basePath);

echo $packages->packageForClass(VendorDirProvider::class)?->name(), PHP_EOL;
echo $packages->root()->composerType(), PHP_EOL;
var_dump($packages->packageForClass(Invoice::class) === $packages->root());
var_dump($packages->root()->locatedPath() === (string) $basePath);
```

构建项目并运行脚本：

```bash
skernel build
php bin/packages.php
```

它会打印：

```text
super-kernel/metadata
project
bool(true)
bool(true)
```

### 查找包和类

`PackageManagerInterface` 只根据恢复的元数据作答：

- `all()` 先返回根包，然后是按名称排序的已安装包；`packages()` 返回不含根包的已安装包。
- `root()` 返回项目；`get($name)` 按任意大小写的名称查找根包或已安装的包，对未知的包返回 `null`；`ofComposerType($type)` 返回声明的 Composer 类型恰好为 `$type` 的包。
- `classes($name)` 返回一个包的类映射（从类名到绝对文件路径），对未知的包返回 `null`。`allClasses()` 返回所有包的类映射的合集；当两个包声明了同一个类时，在 `all()` 中排在前面的包拥有它。`packageForClass($class)` 返回拥有该类的包，或者 `null`。

每个包都是一个 `SuperKernel\Contract\PackageMetadataInterface`：

- `name()` 是小写的 Composer 名称。
- `version()` 是 Composer 记录的原样版本，根清单未声明版本时为 `null`；它不会被规范化或比较。
- `reference()` 是分发引用，缺失时退回到源码引用，或者为 `null`。
- `locatedPath()` 是包所在的目录；对于根包，它就是项目目录。
- 恢复出的包还提供 `composerType()`（包未声明类型时为 `library`）和 `environmentRequirements()`。

### 读取环境要求

包管理器的 `environmentRequirements()` 返回根包和每个已安装的包所声明的、对 Composer 平台包（例如 `php` 或 `ext-swoole`）的要求；`hasEnvironmentRequirements()` 报告是否存在任何要求。每一项都是一个 `SuperKernel\Contract\EnvironmentRequirementInterface`：

- `name()` 是小写的平台包名称，`constraint()` 是原样声明的约束，`requiredBy()` 是声明它的包。
- 只有根包 `require-dev` 中的要求，`isDevRequirement()` 才为 `true`；已安装的包的 `require-dev` 从不被记录。

`super-kernel/environment` 用正在运行的 PHP 检查这些要求。

打印某个包的要求的脚本，保存为 `bin/requirements.php`：

```php
<?php

declare(strict_types=1);

use SuperKernel\Metadata\Provider\BasePathProvider;
use SuperKernel\Metadata\Provider\PackageManagerProvider;
use SuperKernel\Metadata\Provider\VendorDirProvider;

require dirname(__DIR__) . '/vendor/autoload.php';

$vendorDir = new VendorDirProvider()();
$packages = new PackageManagerProvider()($vendorDir, new BasePathProvider()($vendorDir));

foreach ($packages->environmentRequirements() as $requirement) {
    if ($requirement->requiredBy() === 'super-kernel/metadata') {
        echo $requirement->name(), ' ', $requirement->constraint(), PHP_EOL;
    }
}
```

运行 `skernel build` 之后，`php bin/requirements.php` 会打印 `php >=8.4.27 <8.5`。

### 在服务中使用元数据

在应用中，把这些接口作为构造函数参数接收；容器会提供它保留的实例。

报告已安装包的服务，保存为 `src/Diagnostics/PackageReport.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Diagnostics;

use SuperKernel\Contract\PackageManagerInterface;
use SuperKernel\Contract\VendorDirInterface;

use function basename;

final readonly class PackageReport
{
    public function __construct(private PackageManagerInterface $packages, private VendorDirInterface $vendorDir)
    {
    }

    public function describe(string $name): string
    {
        $package = $this->packages->get($name);

        if (null === $package) {
            return "{$name} is not installed";
        }

        return $package->name() . ' is installed in ' . basename((string) $this->vendorDir);
    }
}
```

使用它的脚本，保存为 `bin/report.php`：

```php
<?php

declare(strict_types=1);

use Src\Diagnostics\PackageReport;
use SuperKernel\Context\ApplicationContext;

require dirname(__DIR__) . '/vendor/autoload.php';

$report = ApplicationContext::getContainer()->get(PackageReport::class);

echo $report->describe('Super-Kernel/Metadata'), PHP_EOL;
echo $report->describe('acme/missing'), PHP_EOL;
```

运行 `skernel build` 之后，`php bin/report.php` 会打印：

```text
super-kernel/metadata is installed in vendor
acme/missing is not installed
```

### 随应用分发生成的文件

`skernel build` 在依赖目录之下写出两个文件。`SuperKernel\Metadata\Constants\ArtifactLayout` 为它们命名。

| 文件 | 内容 |
| --- | --- |
| `.cache/build.json` | 构建戳：构建协议、SKERNEL 版本、`composer.lock` 的 SHA-256，以及相对于依赖目录的项目目录。 |
| `.cache/metadata.ser` | 每个包及其类映射，路径相对于项目目录和依赖目录，并由校验和保护。 |

构建戳的样子如下：

```json
{
    "protocol": 1,
    "skernel": "1.0.0-alpha.2",
    "lock": "3f9c5d0e6a1b2c4d8e7f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5",
    "base-path": ".."
}
```

- 请把这两个文件和项目的 `composer.lock` 一起随应用分发；`skernel package` 就是这样做的。恢复时会对 `composer.lock` 计算哈希，并拒绝为不同内容构建的元数据。
- 依赖目录位于项目之内时，`base-path` 是相对路径，因此项目可以连同其依赖目录一起移动。在 Phar 中，路径以 `phar://` 开头。依赖目录位于项目之外时，项目目录被记录为绝对路径，因此该项目不能移动。
- 构建会先删除构建戳、最后写出它，因此被中断的构建不会留下构建戳，恢复时会拒绝它。
- 这些文件是受信任的本地数据：恢复时会反序列化这些记录。永远不要让不受信任的写入者放置它们。
- 构建戳不记录构建时是否包含开发依赖。在 `composer install` 和 `composer install --no-dev` 之间切换之后，请重新构建。

### 从构建工具发布元数据

`skernel build` 在项目自己的进程中、使用项目所安装的本包版本发布元数据，因此格式总是与读取方一致。`SuperKernel\Metadata\Constants\Protocol::VERSION` 标识这种格式的协议；`skernel build` 只构建协议受它支持的项目。

`new SuperKernel\Metadata\Build\MetadataPublisher($vendorDir, $basePath)` 接收规范化的依赖目录和项目目录的绝对路径。请按以下顺序调用它的方法：

1. `invalidate()` 先删除构建戳，再删除包记录。
2. `publishPackages($manager)` 写出 `SuperKernel\Metadata\Manager\PackageManager` 的记录，其根包必须位于项目目录。
3. 用 `super-kernel/annotator` 发布注解记录。
4. `commit($skernelVersion)` 检查记录能否恢复，然后为当前的 `composer.lock` 写出构建戳。版本（例如 `1.0.0-alpha.2` 或 `dev`）只出现在消息中。

每个文件都先写入同一目录下的临时文件，再重命名到位，权限由 umask 决定。

再次发布已恢复的元数据的脚本，保存为 `bin/republish.php`：

```php
<?php

declare(strict_types=1);

use SuperKernel\Metadata\Build\MetadataPublisher;
use SuperKernel\Metadata\Manager\PackageManager;
use SuperKernel\Metadata\Provider\BasePathProvider;
use SuperKernel\Metadata\Provider\PackageManagerProvider;
use SuperKernel\Metadata\Provider\VendorDirProvider;

require dirname(__DIR__) . '/vendor/autoload.php';

$vendorDir = new VendorDirProvider()();
$basePath = new BasePathProvider()($vendorDir);
$packages = new PackageManagerProvider()($vendorDir, $basePath);
assert($packages instanceof PackageManager);

$publisher = new MetadataPublisher($vendorDir, $basePath);
$publisher->invalidate();
$publisher->publishPackages($packages);
$publisher->commit('dev');

echo count(new PackageManagerProvider()($vendorDir, $basePath)->all()), ' packages republished', PHP_EOL;
```

运行 `skernel build` 之后，`php bin/republish.php` 会打印包括根包在内的包数量，例如 `33 packages republished`。

## 错误

`SuperKernel\Metadata\Exception\BuildRequiredException` 和 `SuperKernel\Metadata\Exception\MetadataException` 都实现了 `SuperKernel\Metadata\Exception\MetadataExceptionInterface`。每条 `BuildRequiredException` 消息都以 ``Run `skernel build` to generate the project metadata.`` 结尾，下表省略了这一句。

| 情形 | 消息 |
| --- | --- |
| 没有构建戳 | `The build stamp <path> is missing.` |
| 构建戳无法解码或包含其他键 | `The build stamp is invalid.` 或 `The build stamp has an unexpected structure.` |
| 构建戳是在另一个协议下写出的 | `The metadata was generated by SKERNEL <version> under build protocol <n>, but this project expects <n>.` |
| 项目没有 `composer.lock` | `The project <path> has no readable composer.lock.` |
| 构建之后 `composer.lock` 发生了变化 | `The metadata generated by SKERNEL <version> does not match the current composer.lock.` |
| 没有包记录 | `The package records <path> are missing.` |
| 包记录已损坏 | `The package records are invalid.` 或 `The package records cannot be restored.` |
| 没有任何 Composer 类加载器加载本包（`MetadataException`） | `No registered Composer class loader serves super-kernel/metadata.` |
| 在记录能够恢复之前调用 `commit()`（`MetadataException`） | `The published package records do not restore.` |
| 用位于项目之外的根包调用 `publishPackages()`（`MetadataException`） | `The root package must be located at the base path.` |

## 与其他包的关系

- [`super-kernel/contract`](contract.zh-CN.md) 声明了 `VendorDirInterface`、`BasePathInterface`、`PackageManagerInterface`、`PackageMetadataInterface` 和 `EnvironmentRequirementInterface`。请面向这些接口编写代码。
- [`super-kernel/attribute`](attribute.zh-CN.md) 声明了提供者上的 `#[FactoryProvider]` 注解。
- [`super-kernel/di`](di.zh-CN.md) 在创建根容器时调用这些提供者并保留这三个服务；它把缺失的元数据报告为本包的 `BuildRequiredException`。
- [`super-kernel/annotator`](annotator.zh-CN.md) 用包管理器核对它的注解记录，并把它们写在元数据旁边。
- [`super-kernel/environment`](environment.zh-CN.md) 检查环境要求；[`super-kernel/autoloader`](autoloader.zh-CN.md) 使用类映射。
- `skernel build` 收集并发布元数据；`skernel package` 把它与 `composer.lock` 一起分发。

## 不要这样做

**不要在没有 `composer.lock` 的情况下分发生成的文件。** 恢复时会把构建戳与 `composer.lock` 的哈希进行比较，文件缺失时抛出 `The project <path> has no readable composer.lock.`。请像 `skernel package` 那样把 `composer.lock` 随应用一起分发。

**不要在开发安装和生产安装之间切换而不重新构建。** 在开发构建之后执行 `composer install --no-dev` 不会改变 `composer.lock`，因此过期的元数据不会被检测到，其中仍然列出已被移除的包。每次安装之后请运行 `skernel build`。

**不要编辑或合并生成的文件。** 记录带有校验和，会以 `The package records are invalid.` 被拒绝；修改过的构建戳会被拒绝或不再匹配。请改为运行 `skernel build`。

**不要在记录发布之前调用 `commit()`。** 它拒绝为无法恢复的记录写出构建戳，抛出 `The published package records do not restore.`，在完成一次完整构建之前应用都无法启动。请依次调用 `invalidate()`、`publishPackages()`、发布注解记录，最后调用 `commit()`。
