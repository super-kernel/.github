<p align="right">
  <a href="http-message.md">English</a> | <strong>中文文档</strong>
</p>

# SuperKernel HTTP Message

包：[`super-kernel/http-message`](https://github.com/super-kernel/http-message) · [Packagist](https://packagist.org/packages/super-kernel/http-message) · [文档索引](../README.zh-CN.md)

## 用途

`super-kernel/http-message` 为 SuperKernel 实现 PSR-7：请求、服务器请求、响应、URI、流和上传文件，任何 PSR-7 使用方都能接受它们。它还保存当前作用域中的请求和响应，使共享服务无需以参数形式接收请求，就能读取它正在服务的请求。

本包不依赖服务器。把原生服务器请求转换为 `ServerRequest`、对它进行路由以及发送响应，由 `super-kernel/http-server` 负责。本包不提供 `psr/http-factory` 工厂：每个对象都用它的构造函数创建。

## 能力

### 创建消息

`Request`、`ServerRequest` 和 `Response` 通过构造函数参数接收数据：请求接收方法、字符串形式或 `UriInterface` 形式的 URI、头、消息体和协议版本；服务器请求另外接收服务器、Cookie、查询、上传文件、解析后的消息体和属性数据；响应接收状态码、头、消息体、协议版本和原因短语。消息体可以是 `StreamInterface`、字符串，或者表示空消息体的 `null`。

这些对象遵守的规则：

- 每个 `with*()` 和 `without*()` 方法都返回修改后的副本，原对象保持不变。副本共享消息体流，而流带有游标：一个副本读取消息体之后，另一个副本会看到移动后的游标。读取可能已被其他副本读过的消息体之前，请先调用 `rewind()`。
- 头名称在 `getHeaders()` 中保留给定时的大小写，并且可以用任意大小写查找。头名称必须是 HTTP 记号（token），值必须是不含换行的字符串或字符串列表。
- 没有 `Host` 头的请求从 URI 取得它。`withUri()` 会用新 URI 替换 `Host`，除非请求要求保留它，这与 PSR-7 的规定一致。
- 状态码必须在 `100` 到 `599` 之间。没有原因短语时，响应使用其状态码的标准原因短语；没有标准短语的状态码则为空。自定义原因短语不能包含换行。
- 服务器请求的属性名称必须是字符串，上传文件必须是 `UploadedFileInterface` 对象或由它们组成的嵌套数组。

### 选择流

每种消息体都有自己的流类，用其公开构造函数创建：

- `StringStream` 把字符串复制到临时流中并回绕。
- `ResourceStream` 接收一个已打开的原生流资源，例如 `fopen()` 的结果。
- `FileStream` 按指定模式打开文件，默认模式为 `rb`。
- `IterableStream` 读取由字符串分块组成的可迭代对象，例如生成器，并跳过空分块。
- `CallbackStream` 以请求的字节数调用读取器；读取器返回非空字符串，或者在数据源结束时返回 `null`。
- `ReadOnlyStream` 包装另一个 PSR-7 流：把读取、定位、元数据、关闭和分离交给该流，并拒绝所有写入。

持有原生资源的流拥有该资源。`close()` 关闭资源，`detach()` 把资源返回给调用方，此后由调用方拥有它。调用二者之一之后，流不再报告任何能力，并对每个需要资源的操作抛出 `RuntimeException`。

`IterableStream` 和 `CallbackStream` 可读、只进、不可定位、不可写，并报告未知大小。它们的 `read()` 在数据源产生任何字节后立即返回，而不是等待请求的长度，因此缓慢的生产者（例如服务器发送事件的数据源）会一块一块地到达使用方；分块中超出请求长度的部分会缓存到下一次读取。不是字符串的分块，或读取器返回的空字符串，会在读取时抛出 `RuntimeException`，数据源抛出的异常会作为前一个异常（previous）保留。`close()` 和 `detach()` 会释放数据源；由于没有资源，`detach()` 返回 `null`。

流只保存字节。把数组和对象放进消息体之前，请先对它们进行编码，例如使用 `json_encode()`。把任何流转换为字符串都不会抛出异常：它返回剩余内容，失败后或流关闭后返回 `''`。

读取各种流的脚本，保存为 `bin/streams.php`：

```php
<?php

declare(strict_types=1);

use SuperKernel\HttpMessage\Stream\CallbackStream;
use SuperKernel\HttpMessage\Stream\IterableStream;
use SuperKernel\HttpMessage\Stream\ReadOnlyStream;
use SuperKernel\HttpMessage\Stream\StringStream;

require dirname(__DIR__) . '/vendor/autoload.php';

$events = new IterableStream((static function (): Generator {
    yield "data: first\n\n";
    yield "data: second\n\n";
})());

echo json_encode($events->read(1024)), PHP_EOL;
echo json_encode($events->read(4)), PHP_EOL;
echo var_export($events->getSize(), true), ' ', var_export($events->isSeekable(), true), PHP_EOL;

$remaining = 3;
$countdown = new CallbackStream(static function (int $length) use (&$remaining): ?string {
    return $remaining > 0 ? (string) $remaining-- : null;
});

echo $countdown->getContents(), PHP_EOL;

$readOnly = new ReadOnlyStream(new StringStream('payload'));

try {
    $readOnly->write('more');
} catch (RuntimeException $exception) {
    echo $exception->getMessage(), PHP_EOL;
}

echo $readOnly, PHP_EOL;
```

运行脚本：

```bash
php bin/streams.php
```

它会打印：

```text
"data: first\n\n"
"data"
NULL false
321
The read-only stream is not writable.
payload
```

第一次 `read(1024)` 只返回第一个分块，第二次读取取得下一个分块的四个字节。

### 接收上传文件

`UploadedFile` 首先接收它的来源：`StreamInterface`、临时文件的路径，或者表示上传失败的 `null`。随后接收大小、PHP 上传错误码（默认为 `UPLOAD_ERR_OK`），以及客户端发送的文件名和媒体类型。成功的上传需要来源；未知的错误码或负数大小会被拒绝。

`moveTo()` 把文件写到目标路径。如果 PHP 在本次请求中接收了该临时文件，则使用 `move_uploaded_file()` 移动临时路径，否则重命名它；流会被复制，然后关闭。成功移动之后，`getStream()` 和再次调用 `moveTo()` 会抛出 `RuntimeException`。对于失败的上传，二者都会抛出带有错误码的 `RuntimeException`。

移动上传文件的脚本，保存为 `bin/upload.php`：

```php
<?php

declare(strict_types=1);

use SuperKernel\HttpMessage\Stream\StringStream;
use SuperKernel\HttpMessage\UploadedFile;

require dirname(__DIR__) . '/vendor/autoload.php';

$upload = new UploadedFile(
    new StringStream('name,total' . PHP_EOL),
    size: 11,
    clientFilename: 'report.csv',
    clientMediaType: 'text/csv',
);

$target = sys_get_temp_dir() . '/report.csv';
$upload->moveTo($target);

echo file_get_contents($target);

try {
    $upload->getStream();
} catch (RuntimeException $exception) {
    echo $exception->getMessage(), PHP_EOL;
}

unlink($target);
```

运行脚本：

```bash
php bin/upload.php
```

它会打印：

```text
name,total
The uploaded file has already been moved.
```

### 使用 URI

`new Uri($string)` 解析一个 URI；空字符串表示空 URI。`Uri` 把协议（scheme）和主机转换为小写，在授权部分和 `getPort()` 中省略 `http` 的 80 端口和 `https` 的 443 端口，接受 `0` 到 `65535` 的端口，保留有效的百分号编码字节，并对其他无效字节只做一次百分号编码。它按照 PSR-7 的规定组合授权部分和路径。例如，`new Uri('HTTP://User@EXAMPLE.test:80/a b/%41?q=ü#f')` 渲染为 `http://User@example.test/a%20b/%41?q=%C3%BC#f`。

### 在服务中读取当前请求

`RequestContext` 和 `ResponseContext` 把当前作用域中的请求和响应保存在 `SuperKernel\Context\Context` 中，键分别为 `Psr\Http\Message\ServerRequestInterface` 和 `Psr\Http\Message\ResponseInterface`。它们都有 `set()`、`get()`、`has()` 和 `clear()`；`clear()` 只删除自己的键。代码在协程中运行时，作用域是当前协程，否则是当前线程或进程。`super-kernel/http-server` 在处理请求之前保存这两个消息，处理之后清除它们。

容器把 `ServerRequestInterface` 和 `ResponseInterface` 作为服务提供。对这种服务的每次调用都会作用于当前作用域中保存的消息，因此保留注入对象的共享服务永远不会看到其他请求的数据。这些服务只提供 PSR-7 方法。`with*()` 调用返回新的消息，不会改变已保存的消息；要为后续代码改变当前请求，请用 `RequestContext::set()` 保存新消息。

向当前请求的调用方问候的服务，保存为 `src/Http/Greeting.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Http;

use Psr\Http\Message\ServerRequestInterface;
use SuperKernel\Attribute\Provider;

#[Provider(Greeting::class)]
final readonly class Greeting
{
    public function __construct(private ServerRequestInterface $request)
    {
    }

    public function text(): string
    {
        return 'Hello, ' . ($this->request->getQueryParams()['name'] ?? 'guest') . '!';
    }
}
```

像服务器那样依次保存两个请求的脚本，保存为 `bin/scope.php`：

```php
<?php

declare(strict_types=1);

use Src\Http\Greeting;
use SuperKernel\Context\ApplicationContext;
use SuperKernel\HttpMessage\Context\RequestContext;
use SuperKernel\HttpMessage\ServerRequest;

require dirname(__DIR__) . '/vendor/autoload.php';

$greeting = ApplicationContext::getContainer()->get(Greeting::class);

foreach (['Ada', 'Linus'] as $name) {
    RequestContext::set(new ServerRequest('GET', '/greet', queryParams: ['name' => $name]));

    echo $greeting->text(), PHP_EOL;

    RequestContext::clear();
}

try {
    $greeting->text();
} catch (RuntimeException $exception) {
    echo $exception->getMessage(), PHP_EOL;
}
```

构建项目并运行脚本：

```bash
skernel build
php bin/scope.php
```

它会打印：

```text
Hello, Ada!
Hello, Linus!
The current HTTP request context has not been initialized.
```

## 错误

无效输入会抛出 `InvalidArgumentException`；流、上传或上下文操作失败会抛出 `RuntimeException`。

| 情形 | 消息 |
| --- | --- |
| 不是 HTTP 记号的方法 | `Invalid HTTP method <method>.` |
| 不是 HTTP 记号的头名称 | `Invalid HTTP header name <name>.` |
| 含有换行或其他禁止的控制字符的头值 | `HTTP header <name> contains a prohibited control character.` |
| 不是以点分隔的数字（例如 `1.1`）的协议版本 | `Invalid HTTP protocol version <version>.` |
| 不在 `100` 到 `599` 之间的状态码 | `Invalid HTTP status code <code>.` |
| 为空或含有空白字符的请求目标 | `Invalid HTTP request target <target>.` |
| 无法解析的 URI，包括大于 `65535` 的端口 | `Unable to parse URI <uri>.` |
| 无法打开的文件 | `Unable to open HTTP message stream file <filename>.` |
| 没有资源的 `ResourceStream` | `A native stream resource is required.` |
| 写入只读流 | `The read-only stream is not writable.` |
| 不是字符串的可迭代分块 | `Iterable stream chunks must be strings; got <type>.` |
| 读取器返回空字符串 | `Callback stream sources must not return an empty chunk before end of source.` |
| 可迭代对象或读取器抛出异常 | `Unable to read from the iterable stream source.` 或 `Unable to read from the callback stream source.`，该异常作为前一个异常 |
| 未知的上传错误码 | `Invalid upload error code <code>.` |
| 没有来源的成功上传 | `A successful upload requires a stream or temporary file path.` |
| 移动之后再使用上传文件 | `The uploaded file has already been moved.` |
| 使用失败的上传 | `The uploaded file is unavailable because upload error <code> occurred.` |
| 含有其他值的上传文件树 | `<path> must contain only uploaded files or nested arrays.` |
| 当前作用域中没有保存请求或响应 | `The current HTTP request context has not been initialized.` 或 `The current HTTP response context has not been initialized.` |

## 与其他包的关系

- [`super-kernel/http-server`](http-server.zh-CN.md) 从每个原生请求创建 `ServerRequest`，保存当前消息，并发送响应。
- [`super-kernel/context`](context.zh-CN.md) 提供保存当前消息的协程或进程作用域。
- [`super-kernel/di`](di.zh-CN.md) 注入 `ServerRequestInterface` 和 `ResponseInterface`；这些服务从 `skernel build` 生成的元数据中发现。
- 任何接受 `psr/http-message` 2.0 对象的库都接受本包的消息、流、URI 和上传文件。
