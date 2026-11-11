<p align="right">
  <a href="filesystem.md">English</a> | <strong>中文文档</strong>
</p>

# SuperKernel Filesystem

包：[`super-kernel/filesystem`](https://github.com/super-kernel/filesystem) · [Packagist](https://packagist.org/packages/super-kernel/filesystem) · [文档索引](../README.zh-CN.md)

## 用途

`super-kernel/filesystem` 把要读取的东西描述为一个资源，并把句柄交给资源管理。代码调用 `read()` 并迭代其中的值：文件、流和内存产出字节块，目录产出条目。资源在迭代开始时打开它的 PHP 句柄，在迭代结束时关闭它，因此代码从不持有、共享句柄，也不会忘记关闭句柄。

本包是同步且只读的。写入、异步或 Swoole 专用的读取、路径策略（例如限制范围、规范化路径或 glob 模式）以及缓存都属于其他地方。它的类都是普通对象：本包不注册任何容器服务，也不需要构建。

## 能力

### 读取文件

`LocalResourceFactory::file()` 描述一个普通文件；第二个参数 `chunkSize` 是以字节为单位的最大块大小，默认为 8192。`read()` 以不超过该大小的字符串产出文件的字节。数据块不带任何编码，也没有行边界：一行的字节可能跨越两个块，一个块也可能在多字节字符的中间结束。当内容必须完整时，请读取整个资源。

路径在创建资源时检查，文件只在迭代开始时打开；不是普通文件的路径会在那时失败。

读取文件的脚本，保存为 `bin/read-file.php`：

```php
<?php

declare(strict_types=1);

use SuperKernel\Filesystem\Factory\LocalResourceFactory;

require dirname(__DIR__) . '/vendor/autoload.php';

$path = sys_get_temp_dir() . '/invoice-numbers.txt';
file_put_contents($path, "INV-1001\nINV-1002\n");

$file = new LocalResourceFactory()->file($path, chunkSize: 8);

foreach ($file->read() as $chunk) {
    echo '[', addcslashes($chunk, "\n"), ']', PHP_EOL;
}
```

运行脚本：

```bash
php bin/read-file.php
```

它会打印：

```text
[INV-1001]
[\nINV-100]
[2\n]
```

### 读取目录

`LocalResourceFactory::directory()` 描述一个目录。`read()` 为每个直接子条目产出一个 `SuperKernel\Filesystem\Model\DirectoryEntry`，不包括 `.` 和 `..`，也不会深入子目录。每个条目都有 `name()`、把目录路径和名称拼接起来的 `path()`，以及 `type()`，后者是 `SuperKernel\Filesystem\Constants\DirectoryEntryType` 的 `Directory`、`File`、`Link` 或 `Other` 之一；符号链接无论指向什么都报告为 `Link`。条目按文件系统自己的顺序出现，该顺序未指定，因此在顺序重要时请自行排序。

列出目录的脚本，保存为 `bin/read-directory.php`：

```php
<?php

declare(strict_types=1);

use SuperKernel\Filesystem\Factory\LocalResourceFactory;

require dirname(__DIR__) . '/vendor/autoload.php';

$path = sys_get_temp_dir() . '/invoice-archive';
@mkdir($path . '/2026', recursive: true);
file_put_contents($path . '/index.txt', "INV-1001\n");

$entries = [];

foreach (new LocalResourceFactory()->directory($path)->read() as $entry) {
    $entries[$entry->name()] = $entry->type()->value;
}

ksort($entries);

foreach ($entries as $name => $type) {
    echo "{$name}: {$type}", PHP_EOL;
}
```

运行脚本：

```bash
php bin/read-directory.php
```

它会打印：

```text
2026: directory
index.txt: file
```

### 读取流

`LocalResourceFactory::stream()` 描述 PHP 的 `fopen()` 能以读取方式打开的任何 URI，例如 `php://stdin`、`data:` URI 或某个流包装器，数据块规则与文件相同。

读取 `data:` URI 的脚本，保存为 `bin/read-stream.php`：

```php
<?php

declare(strict_types=1);

use SuperKernel\Filesystem\Factory\LocalResourceFactory;

require dirname(__DIR__) . '/vendor/autoload.php';

$stream = new LocalResourceFactory()->stream('data://text/plain;base64,SU5WLTEwMDEsSU5WLTEwMDI=', chunkSize: 9);

foreach ($stream->read() as $chunk) {
    echo "[{$chunk}]", PHP_EOL;
}
```

运行脚本：

```bash
php bin/read-stream.php
```

它会打印：

```text
[INV-1001,]
[INV-1002]
```

### 读取由你的代码打开的流

`LocalResourceFactory::streamFromFactory()` 或 `SuperKernel\Filesystem\Resource\StreamResource::fromFactory()` 接受一个打开流的闭包。资源在每个读取周期开始时调用它，并在之后关闭流，因此闭包每次调用都必须返回一个新打开的 PHP 流。`description` 在错误消息中为该流命名，默认为 `custom stream`。闭包抛出的异常会成为 `ResourceReadException` 的前一个异常（previous）。

两次读取生成的报告的脚本，保存为 `bin/read-report.php`：

```php
<?php

declare(strict_types=1);

use SuperKernel\Filesystem\Factory\LocalResourceFactory;

require dirname(__DIR__) . '/vendor/autoload.php';

$report = new LocalResourceFactory()->streamFromFactory(
    static function () {
        $stream = fopen('php://temp', 'w+b');
        fwrite($stream, 'Monthly invoice report');
        rewind($stream);

        return $stream;
    },
    description: 'report stream',
);

foreach ([1, 2] as $cycle) {
    echo "Read {$cycle}: ", implode('', iterator_to_array($report->read(), false)), PHP_EOL;
}
```

运行脚本：

```bash
php bin/read-report.php
```

它会打印：

```text
Read 1: Monthly invoice report
Read 2: Monthly invoice report
```

### 读取保存在内存中的数据

`LocalResourceFactory::binaryStream()` 或 `new SuperKernel\Filesystem\Resource\BinaryStreamResource($contents, $chunkSize)` 通过同一个契约按块产出一个字符串，因此可以把已经在内存中的数据交给消费资源的代码。PHP 字符串是二进制安全的，数据块也是如此。

读取二进制数据的脚本，保存为 `bin/read-memory.php`：

```php
<?php

declare(strict_types=1);

use SuperKernel\Filesystem\Factory\LocalResourceFactory;

require dirname(__DIR__) . '/vendor/autoload.php';

$signature = new LocalResourceFactory()->binaryStream("\x89PNG\r\n\x1a\n", chunkSize: 4);

foreach ($signature->read() as $chunk) {
    echo bin2hex($chunk), PHP_EOL;
}
```

运行脚本：

```bash
php bin/read-memory.php
```

它会打印：

```text
89504e47
0d0a1a0a
```

### 了解一次读取的生命周期

每次调用 `read()` 都会开始一个新的、独立的读取周期，因此资源可以从头再读一遍。句柄在迭代开始时打开，而不是在创建资源或迭代器时打开；当迭代耗尽、读取失败，或者部分消费的迭代器被释放（例如在 `foreach` 中 `break` 之后）时，句柄会被关闭。资源从不暴露它们的句柄。

### 依赖契约

`SuperKernel\Filesystem\Contract\ReadableResourceInterface` 只有一个方法 `read()`，并且对它产出的内容是泛型的：目录产出 `DirectoryEntry` 值，字节资源产出字符串。这些资源共有的是读取生命周期，而不是一种载荷类型。`SuperKernel\Filesystem\Contract\ResourceFactoryInterface` 声明了五个工厂方法，由 `SuperKernel\Filesystem\Factory\LocalResourceFactory` 实现。请让读取的代码依赖这些接口，并在组装代码的地方创建工厂，这样测试就可以传入另一个资源，例如内存资源。

## 错误

每个异常都实现 `SuperKernel\Filesystem\Exception\FilesystemExceptionInterface`。配置错误是在创建资源时抛出的 `InvalidResourceConfigurationException`；读取错误是在 `read()` 期间抛出的 `ResourceReadException`。消息中的 `<label>` 是 `File path`、`Directory path`、`Stream URI` 或 `Stream description`，`<description>` 对 URI 是 `stream "<uri>"`，对工厂流是传给它的描述。

| 情形 | 消息 |
| --- | --- |
| 空的路径、URI 或描述 | `<label> must not be empty.` |
| 包含空字节的路径、URI 或描述 | `<label> must not contain null bytes.` |
| 小于一的块大小 | `Chunk size must be greater than zero.` |
| 不是普通文件的文件路径 | `File "<path>" is not a regular file.` |
| 无法打开的目录 | `Unable to open directory "<path>".` |
| 无法打开的流，或者抛出异常、没有返回流的工厂 | `Unable to open <description>.` |
| 无法读取的流 | `Unable to read <description>.` |
| 在结束之前没有返回数据的流 | `Reading <description> made no progress before end of stream.` |

## 与其他包的关系

- 本包不依赖任何其他包；请在你的代码中直接使用它。
- 它不在 [`super-kernel/di`](di.zh-CN.md) 中注册任何服务；请在组装代码的地方创建 `LocalResourceFactory`。

## 不要这样做

**不要让工厂两次返回同一个流。** 资源在每个读取周期之后关闭流，因此返回自己保留的流的闭包会把一个已关闭的流交给第二次 `read()`，后者以 `Unable to open <description>.` 失败。请在每次调用闭包时在其中打开一个新的流。
