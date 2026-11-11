<p align="right">
  <a href="annotator.md">English</a> | <strong>中文文档</strong>
</p>

# SuperKernel Annotator

包：[`super-kernel/annotator`](https://github.com/super-kernel/annotator) · [Packagist](https://packagist.org/packages/super-kernel/annotator) · [文档索引](../README.zh-CN.md)

## 用途

`super-kernel/annotator` 为应用提供其类、方法和属性上的原生 PHP 注解，以及已安装的包中各个类上的注解，应用启动时不扫描任何内容。`skernel build` 一次性收集并构造这些注解；本包把它们作为记录发布到依赖目录（vendor 目录）之下，并把它们恢复为一个回答查询的不可变收集器。

决定收集哪些类、构造注解以及报告失败的类，都由 `skernel build` 负责。本包只支持原生 PHP 注解，不支持 DocBlock 注释注解。

## 能力

### 按注解查找类和方法

`SuperKernel\Contract\AnnotationCollectorInterface` 就是收集器。SuperKernel 容器提供它，因此服务把它作为构造函数参数接收，脚本则通过 `ApplicationContext::getContainer()` 查找它。

- `getClassesWithAttribute(Attribute::class)` 返回一个映射：从每个带有该确切注解类的类，到它匹配的条目列表。类按其所属包的构建顺序排列，同一个包内按类名排列。
- `getMethodsWithAttribute(Attribute::class)` 返回一个先以声明类、再以方法名为键的映射，每一项是匹配的条目列表。一个类的方法按名称顺序列出。
- 每个条目都是一个 `SuperKernel\Contract\AttributeMetadataInterface`。`attribute()` 返回 `skernel build` 构造的注解实例；`name()` 返回它的类。可重复的注解每次声明对应一个条目，按声明顺序排列。
- 违反目标或可重复性限制的注解在 `skernel build` 构造它时就会失败，因此永远不会出现在结果中。请直接使用这些条目：结果已经保证是被查询类的实例，注解也不会被再次构造。

收集器不检查你自己领域中的关系，例如被标记的命令是否继承了 Symfony 的 `Command`；由赋予该注解含义的使用方检查它们。

注解，保存为 `src/Routing/Route.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Routing;

use Attribute;

#[Attribute(Attribute::TARGET_METHOD | Attribute::IS_REPEATABLE)]
final readonly class Route
{
    public function __construct(public string $method, public string $path)
    {
    }
}
```

使用它的控制器，保存为 `src/Http/PageController.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Http;

use Src\Routing\Route;

final class PageController
{
    #[Route('GET', '/')]
    public function home(): string
    {
        return 'Home';
    }

    #[Route('GET', '/about')]
    #[Route('GET', '/about-us')]
    public function about(): string
    {
        return 'About';
    }
}
```

列出路由的脚本，保存为 `bin/routes.php`：

```php
<?php

declare(strict_types=1);

use Src\Routing\Route;
use SuperKernel\Context\ApplicationContext;
use SuperKernel\Contract\AnnotationCollectorInterface;

require dirname(__DIR__) . '/vendor/autoload.php';

$collector = ApplicationContext::getContainer()->get(AnnotationCollectorInterface::class);

foreach ($collector->getMethodsWithAttribute(Route::class) as $class => $methods) {
    foreach ($methods as $method => $routes) {
        foreach ($routes as $route) {
            echo $route->attribute()->method, ' ', $route->attribute()->path, ' -> ', $class, '::', $method, PHP_EOL;
        }
    }
}
```

构建项目并运行脚本：

```bash
skernel build
php bin/routes.php
```

它会打印：

```text
GET /about -> Src\Http\PageController::about
GET /about-us -> Src\Http\PageController::about
GET / -> Src\Http\PageController::home
```

### 按家族查找注解

`getClassesWithAttributeAssignableTo(Base::class)` 和 `getMethodsWithAttributeAssignableTo(Base::class)` 返回与确切查询相同的结构，但包含所有类继承或实现了 `Base` 的注解。`skernel build` 捕获每个注解的父类和接口，因此应用不需要通过反射来发现这个家族。确切查询保持确切：用基类调用 `getClassesWithAttribute()` 只会找到正好属于该类的注解。

家族的基类，保存为 `src/Jobs/Schedule.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Jobs;

abstract readonly class Schedule
{
    abstract public function describe(): string;
}
```

家族中的一个注解，保存为 `src/Jobs/Daily.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Jobs;

use Attribute;

#[Attribute(Attribute::TARGET_CLASS)]
final readonly class Daily extends Schedule
{
    public function __construct(public string $at)
    {
    }

    public function describe(): string
    {
        return "daily at {$this->at}";
    }
}
```

另一个注解，保存为 `src/Jobs/Hourly.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Jobs;

use Attribute;

#[Attribute(Attribute::TARGET_CLASS)]
final readonly class Hourly extends Schedule
{
    public function describe(): string
    {
        return 'hourly';
    }
}
```

一个任务，保存为 `src/Jobs/SendReports.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Jobs;

#[Daily('06:00')]
final class SendReports
{
}
```

另一个任务，保存为 `src/Jobs/RefreshRates.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Jobs;

#[Hourly]
final class RefreshRates
{
}
```

列出调度的脚本，保存为 `bin/jobs.php`：

```php
<?php

declare(strict_types=1);

use Src\Jobs\Daily;
use Src\Jobs\Schedule;
use SuperKernel\Context\ApplicationContext;
use SuperKernel\Contract\AnnotationCollectorInterface;

require dirname(__DIR__) . '/vendor/autoload.php';

$collector = ApplicationContext::getContainer()->get(AnnotationCollectorInterface::class);

foreach ($collector->getClassesWithAttributeAssignableTo(Schedule::class) as $class => $schedules) {
    echo $class, ' runs ', $schedules[0]->attribute()->describe(), PHP_EOL;
}

echo count($collector->getClassesWithAttribute(Daily::class)), ' daily job', PHP_EOL;
echo count($collector->getClassesWithAttribute(Schedule::class)), ' exact Schedule matches', PHP_EOL;
```

运行 `skernel build` 之后，`php bin/jobs.php` 会打印：

```text
Src\Jobs\RefreshRates runs hourly
Src\Jobs\SendReports runs daily at 06:00
1 daily job
0 exact Schedule matches
```

### 读取一个类或成员的注解

- `getClassAttributes($class)`、`getMethodAttributes($class, $method)` 和 `getPropertyAttributes($class, $property)` 返回在该处声明的条目列表，或者空列表。
- 只记录类自己声明的成员。继承来的方法或属性列在声明它的类之下，而不是列在每个子类之下。
- 条目的 `types()` 先返回注解自身的类，然后是构建时捕获的父类和接口，没有重复项。
- `getPackages()` 返回每个包的记录，以小写的 Composer 包名为键；`getPackage($name)` 接受任意大小写的包名，对未知的包返回 `null`。每条记录都是一个带有 `classAttributes()`、`methodAttributes()` 和 `propertyAttributes()` 的 `SuperKernel\Contract\PackageAnnotationsInterface`。
- 当两个包声明了同一个类时，按构建顺序排在前面的包拥有它，即使它对该类的记录为空也是如此。
- 查询从不再次构造注解，也从不加载或反射被注解的类。重复查询返回相同的实例。

读取单个声明的脚本，保存为 `bin/inspect.php`：

```php
<?php

declare(strict_types=1);

use Src\Http\PageController;
use Src\Jobs\SendReports;
use SuperKernel\Context\ApplicationContext;
use SuperKernel\Contract\AnnotationCollectorInterface;

require dirname(__DIR__) . '/vendor/autoload.php';

$collector = ApplicationContext::getContainer()->get(AnnotationCollectorInterface::class);

foreach ($collector->getMethodAttributes(PageController::class, 'about') as $metadata) {
    echo $metadata->name(), ' ', $metadata->attribute()->path, PHP_EOL;
}

echo implode(', ', $collector->getClassAttributes(SendReports::class)[0]->types()), PHP_EOL;
```

运行 `skernel build` 之后，`php bin/inspect.php` 会打印：

```text
Src\Routing\Route /about
Src\Routing\Route /about-us
Src\Jobs\Daily, Src\Jobs\Schedule
```

### 读取跳过类诊断

当 `skernel build` 无法收集某个类的注解时（例如它的父类没有安装），它会跳过该类、打印它，并记录一条诊断行。收集器和每条包记录还实现了 `SuperKernel\Contract\ScanDiagnosticsInterface`，其 `getDiagnostics()` 返回这些行：包返回自己的行，收集器按构建顺序返回全部行。应用启动时没有任何代码打印这些行。

每一行恰好包含以下字段：

| 字段 | 值 |
| --- | --- |
| `sequence` | 正整数；一次构建的各行从 1 连续编号到 N。 |
| `package` | 拥有该类的包的小写 Composer 名称。 |
| `class` | 捕获到的类名；读取它从不加载该类。 |
| `exception_type` | 中止收集的异常的类。 |
| `message` | 保留原始字节的异常消息，可能为空，也可能不是 UTF-8。 |

这些行不保存异常对象、调用栈或时间戳。消息不会脱敏，可能包含路径或应用数据。

打印这些行的脚本，保存为 `bin/skipped.php`：

```php
<?php

declare(strict_types=1);

use SuperKernel\Context\ApplicationContext;
use SuperKernel\Contract\AnnotationCollectorInterface;
use SuperKernel\Contract\ScanDiagnosticsInterface;

require dirname(__DIR__) . '/vendor/autoload.php';

$collector = ApplicationContext::getContainer()->get(AnnotationCollectorInterface::class);

if ($collector instanceof ScanDiagnosticsInterface) {
    foreach ($collector->getDiagnostics() as $row) {
        echo $row['sequence'], ' ', $row['package'], ' ', $row['class'], ': ', $row['exception_type'], PHP_EOL;
    }
}
```

这些行取决于已安装的包。在一个 `symfony/console` 和 `symfony/dotenv` 中的某些类依赖于未安装的包的项目里，`php bin/skipped.php` 会打印：

```text
1 symfony/console Symfony\Component\Console\ConsoleBundle: Error
2 symfony/dotenv Symfony\Component\Dotenv\Command\DotenvDumpCommand: Error
```

### 不使用容器恢复记录

`new SuperKernel\Annotator\Cache\AnnotationCache($vendorDir)` 配合 `read($packages)` 自行恢复收集器。它接收 `super-kernel/metadata` 的提供者所提供的 `SuperKernel\Contract\VendorDirInterface` 和 `SuperKernel\Contract\PackageManagerInterface`，`super-kernel/di` 在创建根容器时调用的也正是它。

- 每次调用都会读取并校验所有记录，并返回一个新的收集器；调用之间不保留任何内容。
- 只有当记录正是为已安装的包及其版本写出时，它才接受这些记录。
- 它抛出 `SuperKernel\Annotator\Exception\AnnotationCacheException`，并且不向任何输出写入内容。

恢复收集器的脚本，保存为 `bin/restore.php`：

```php
<?php

declare(strict_types=1);

use Src\Routing\Route;
use SuperKernel\Annotator\Cache\AnnotationCache;
use SuperKernel\Metadata\Provider\BasePathProvider;
use SuperKernel\Metadata\Provider\PackageManagerProvider;
use SuperKernel\Metadata\Provider\VendorDirProvider;

require dirname(__DIR__) . '/vendor/autoload.php';

$vendorDir = new VendorDirProvider()();
$packages = new PackageManagerProvider()($vendorDir, new BasePathProvider()($vendorDir));

$collector = new AnnotationCache($vendorDir)->read($packages);

echo count($collector->getMethodsWithAttribute(Route::class)), ' class with routes', PHP_EOL;
```

运行 `skernel build` 之后，`php bin/restore.php` 会打印 `1 class with routes`。

在测试中，`new SuperKernel\Annotator\AnnotationCollector($packages)` 用任意 `PackageAnnotationsInterface` 记录的列表创建一个收集器，包括你自己的实现。它原样保留收到的记录和条目。只有 `PackageAnnotations` 记录可以被发布。

### 从构建工具发布记录

`skernel build` 在项目自己的进程中发布记录。自行收集注解的构建工具用 `new SuperKernel\Annotator\Cache\AnnotationPublisher($vendorDir)` 和 `publish($packages, $records)` 发布：

- `$records` 为 `$packages` 中的每个包恰好包含一个 `SuperKernel\Annotator\Model\PackageAnnotations`，带有该包的名称和 `reference()`。记录可以按任意顺序到达，其中的映射也可以未排序；内容相同则发布的文件相同。
- `new PackageAnnotations($name, $reference, $classes, $methods, $properties, $diagnostics)` 接收以类为键的类注解、以类和成员名为键的方法与属性注解，以及可选的该包的诊断行。所有记录的序号合起来必须从 1 连续到 N。
- `new SuperKernel\Annotator\Model\AttributeMetadata($attribute, $types)` 接收一个已构造的注解以及要捕获的父类和接口。当注解无法被序列化和恢复、其对象图嵌套超过 64 层，或者它不是某个被捕获类型的实例时，它抛出 `InvalidArgumentException`。请像 `skernel build` 那样，在从失败的类中恢复的代码里创建它。
- 发布失败时，会在异常向外传播之前删除所有注解记录，因此之后的 `read()` 会失败，而不是返回不完整的数据。

在 `skernel build` 已发布的记录中添加一条类记录的脚本，保存为 `bin/publish.php`：

```php
<?php

declare(strict_types=1);

use Src\Jobs\Hourly;
use Src\Jobs\Schedule;
use SuperKernel\Annotator\Cache\AnnotationCache;
use SuperKernel\Annotator\Cache\AnnotationPublisher;
use SuperKernel\Annotator\Model\AttributeMetadata;
use SuperKernel\Annotator\Model\PackageAnnotations;
use SuperKernel\Metadata\Provider\BasePathProvider;
use SuperKernel\Metadata\Provider\PackageManagerProvider;
use SuperKernel\Metadata\Provider\VendorDirProvider;

require dirname(__DIR__) . '/vendor/autoload.php';

$vendorDir = new VendorDirProvider()();
$packages = new PackageManagerProvider()($vendorDir, new BasePathProvider()($vendorDir));
$root = $packages->root()->name();

$records = [];

foreach (new AnnotationCache($vendorDir)->read($packages)->getPackages() as $name => $record) {
    assert($record instanceof PackageAnnotations);

    if ($name === $root) {
        $classes = $record->classAttributes();
        $classes['Src\Jobs\PurgeCache'] = [new AttributeMetadata(new Hourly(), [Schedule::class])];

        $record = new PackageAnnotations(
            $name,
            $record->reference(),
            $classes,
            $record->methodAttributes(),
            $record->propertyAttributes(),
            $record->getDiagnostics(),
        );
    }

    $records[] = $record;
}

new AnnotationPublisher($vendorDir)->publish($packages, $records);

$collector = new AnnotationCache($vendorDir)->read($packages);

echo implode(', ', array_keys($collector->getClassesWithAttributeAssignableTo(Schedule::class))), PHP_EOL;
```

运行 `skernel build` 之后，`php bin/publish.php` 会打印 `Src\Jobs\PurgeCache, Src\Jobs\RefreshRates, Src\Jobs\SendReports`。下一次 `skernel build` 会再次替换这些记录。

### 随应用分发生成的文件

记录位于 `VendorDirInterface` 指定的依赖目录之下，即使它不是 `vendor/` 也是如此：

```text
<vendor>/.cache/
├── packages.ser
└── annotations/
    ├── super-kernel/metadata.ser
    ├── super-kernel/reflector.ser
    └── vendor/package.ser
```

- `annotations/<package>.ser` 保存一个包的记录及其诊断。`packages.ser` 把每个包名映射到它的 Composer 引用，并且最后写出。`SuperKernel\Annotator\Constants\CacheLayout` 为两者命名：`IndexPath` 是 `packages.ser`，`PackageDirectory` 是 `annotations`，都相对于 `<vendor>/.cache/`。
- 恢复时会加载注解的类，包括它们使用的枚举，但记录中从不包含被注解的类。
- 这些文件是受信任的本地数据：恢复时会反序列化它们。永远不要让不受信任的写入者放置它们。
- 每个文件都以原子方式替换，但一次发布不是跨包的事务，也不加锁。发布失败只删除 `packages.ser` 和 `annotations/`；旁边属于 `super-kernel/metadata` 的文件保持不变。
- 这些文件是派生数据。请随应用一起分发它们（`skernel package` 就是这样做的），并通过重新构建而不是修补来更新它们：无法解码或校验的记录会被拒绝，而不会被跳过。Phar 必须携带为其所含代码构建的记录。

## 错误

`read()` 和 `publish()` 抛出 `SuperKernel\Annotator\Exception\AnnotationCacheException`，它实现了 `SuperKernel\Annotator\Exception\AnnotationCollectorExceptionInterface`。当 `super-kernel/di` 恢复记录时，它抛出 `SuperKernel\Metadata\Exception\BuildRequiredException`，消息为 ``The generated annotation records cannot be restored. Run `skernel build` to generate the project metadata.``，并以该注解收集异常作为其前一个异常（previous）。

| 情形 | 消息 |
| --- | --- |
| `packages.ser` 缺失 | `The annotation package-reference stub is unavailable.` |
| `packages.ser` 无法解码或无效 | `The annotation package-reference stub cannot be decoded.` 或 `The annotation package-reference stub is invalid.` |
| 记录是为其他包或其他版本构建的 | `The annotation package-reference stub does not match current Composer metadata.` |
| 某个包的记录缺失 | `The annotation cache for package <package> is unavailable.` |
| 某个包的记录无法解码或无效 | `The annotation cache for package <package> cannot be decoded.` 或 `The annotation cache for package <package> is invalid.` |
| 诊断序号不是从 1 连续到 N | `Annotation diagnostic sequences must be unique and contiguous.` |
| 发布的记录不是 `PackageAnnotations` | `Every published annotation record must be PackageAnnotations.` |
| 为一个包发布了两条记录 | `Package <package> has more than one published annotation record.` |
| 发布的记录没有恰好覆盖已安装的包 | `Collected annotation packages do not match Composer package metadata.` |
| 发布的记录与其包的引用不同 | `Collected annotation reference mismatch for package <package>.` |
| 注解无法被序列化和恢复，或嵌套过深 | `Attribute <class> must support trusted serialization and restoration.`（`InvalidArgumentException`） |
| 注解不是某个被捕获类型的实例 | `Attribute <class> is not assignable to captured type <type>.`（`InvalidArgumentException`） |

## 与其他包的关系

- [`super-kernel/contract`](contract.zh-CN.md) 声明了 `AnnotationCollectorInterface`、`AttributeMetadataInterface`、`PackageAnnotationsInterface` 和 `ScanDiagnosticsInterface`；本包只依赖它。请面向这些接口编写代码。
- [`super-kernel/metadata`](metadata.zh-CN.md) 提供依赖目录和包管理器，恢复和发布时用它们核对记录。
- [`super-kernel/di`](di.zh-CN.md) 在创建根容器时恢复收集器，并把它保存在 `AnnotationCollectorInterface` 之下；它用收集器查找提供者、定义器、解析器和属性处理器。
- [`super-kernel/attribute`](attribute.zh-CN.md) 声明了 SuperKernel 各个包通过收集器查找的注解。
- `skernel build` 选择类、构造它们的注解、打印被跳过的类并发布记录；`skernel package` 分发这些记录。

## 不要这样做

**不要在不同的安装之间复制记录。** 记录携带每个包的引用。为其他版本的包构建的记录会被拒绝，消息为 `The annotation package-reference stub does not match current Composer metadata.`。请在每个安装中运行 `skernel build`，并且在每次 `composer install` 或 `composer update` 之后运行它。

**不要修补或手工编辑生成的文件。** 无法解码或校验的记录会被拒绝，应用无法启动。请运行会重写它们的 `skernel build`。

**不要发布 `PackageAnnotations` 以外的记录，也不要为一个包发布两条记录。** `publish()` 会分别以 `Every published annotation record must be PackageAnnotations.` 和 `Package <package> has more than one published annotation record.` 拒绝它们，并删除所有已经发布的记录。请为每个已安装的包发布一条 `PackageAnnotations`。

**不要期望 DocBlock 注释注解会被收集。** 只有原生 PHP 注解会被记录；`@Route` 注释会被忽略。请声明一个注解类并使用 `#[...]`。
