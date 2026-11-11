<p align="right">
  <a href="http-server.md">English</a> | <strong>中文文档</strong>
</p>

# SuperKernel HTTP Server

包：[`super-kernel/http-server`](https://github.com/super-kernel/http-server) · [Packagist](https://packagist.org/packages/super-kernel/http-server) · [文档索引](../README.zh-CN.md)

## 用途

`super-kernel/http-server` 是 SuperKernel 应用的 HTTP 层。对于 `super-kernel/server` 启动的服务器中的每个 HTTP 或 WebSocket 监听器，它接收每个 Swoole 请求，检查它，把它转换为 PSR-7 `ServerRequest`，让它经过 PSR-15 中间件到达控制器动作，把动作的结果转换为响应，再通过 Swoole 把该响应写回。途中出现的异常通过按监听器声明的异常处理器变成响应。

PSR-7 对象和当前请求服务属于 `super-kernel/http-message`。创建服务器及其监听器属于 `super-kernel/server`。WebSocket 消息、原始 TCP 和协程模式的服务器留给其他代码处理。

## 能力

### 把请求路由到控制器

用 `#[SuperKernel\HttpServer\Attribute\Controller]` 标记一个类，并用一个或多个 `#[SuperKernel\HttpServer\Attribute\RequestMapping]` 标记每个公开动作：

- `Controller` 首先接收路由前缀，然后接收监听器名称 `server`，默认为 `http`。该名称必须与服务器配置中某个监听器的 `name` 完全一致，包括大小写。每个监听器都有自己的路由表，因此同一个方法和路径可以存在于两个监听器上，而且控制器对其他监听器不可见。
- `RequestMapping` 接收路径和一个 `SuperKernel\HttpServer\Constants\HttpMethod` 枚举值，默认为 `HttpMethod::Get`；需要更多方法时重复使用它。不接受方法字符串。只有控制器类自己声明的公开非静态方法才会被路由；继承来的或静态的带有 `RequestMapping` 的方法会被忽略。
- 完整路径是前缀后接路由路径，二者都以 `/` 开头。因此路由路径 `''` 或 `/` 在 `<prefix>/`（带有结尾斜杠）上应答，而不是在单独的前缀上。
- 路径使用 FastRoute 语法：`{name}` 匹配一个路径段，`{name:\d+}` 匹配一个正则表达式。某个监听器路由表中的冲突（例如同一个方法和路径出现两次）会在启动时以 `Unable to build the HTTP route table.` 终止服务器。

动作参数按以下顺序填充：名称与路由变量相同的参数接收该值，并转换为参数的 `string`、`int`、`float`、`bool` 或回退枚举（backed enum）类型；类型为 `Psr\Http\Message\ServerRequestInterface` 的参数接收当前请求；类型为另一个类或接口的参数从容器中解析；否则参数保留默认值，或者在可为 null 时接收 `null`。其他参数会抛出 `Unable to resolve parameter $<name> of <class>::<method>().` 路由变量也是请求属性。无法转换的值（例如未知的枚举值）会抛出 `Route value <value> cannot be converted for parameter $<name>.`，并以 `500` 应答；可能出现其他值时，请用正则表达式约束路由变量。

控制器本身从容器中解析，结果必须是控制器类或其子类（例如代理）的实例。其他对象、标量或 `null` 会在解析动作参数之前抛出 `HTTP controller <class> must resolve to a compatible instance, <type> returned.` 容器抛出的异常原样传出；缺失的、私有的或受保护的动作，或者类型错误的参数，会抛出 PHP 自身的错误。

最小的控制器，其数组结果会成为 JSON 响应，保存为 `src/Controller/GreetingController.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Controller;

use SuperKernel\HttpServer\Attribute\Controller;
use SuperKernel\HttpServer\Attribute\RequestMapping;

#[Controller('/greetings')]
final readonly class GreetingController
{
    #[RequestMapping('/{name}')]
    public function show(string $name): array
    {
        return [
            'greeting' => "Hello, {$name}!",
        ];
    }
}
```

用于路由值的枚举，保存为 `src/Catalog/Widget.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Catalog;

enum Widget: string
{
    case Gear = 'gear';
    case Spring = 'spring';
}
```

带有类型化路由变量、当前请求以及一个动作上两种方法的控制器，保存为 `src/Controller/WidgetController.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Controller;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Src\Catalog\Widget;
use SuperKernel\HttpMessage\Response;
use SuperKernel\HttpServer\Attribute\Controller;
use SuperKernel\HttpServer\Attribute\RequestMapping;
use SuperKernel\HttpServer\Constants\HttpMethod;

#[Controller('/widgets')]
final readonly class WidgetController
{
    #[RequestMapping('/{widget}/{quantity:\d+}')]
    public function quote(Widget $widget, int $quantity, ServerRequestInterface $request): array
    {
        return [
            'widget'   => $widget->value,
            'quantity' => $quantity,
            'currency' => $request->getQueryParams()['currency'] ?? 'EUR',
        ];
    }

    #[RequestMapping('/{widget}', HttpMethod::Post)]
    #[RequestMapping('/{widget}', HttpMethod::Put)]
    public function store(Widget $widget): ResponseInterface
    {
        return new Response(201, ['Location' => "/widgets/{$widget->value}/1"]);
    }
}
```

在由 `super-kernel/skeleton` 创建的应用中构建并启动它，然后在另一个终端中发送请求：

```bash
skernel build
php bin/bootstrap.php swoole:start
curl 'http://127.0.0.1:9600/widgets/gear/3?currency=USD'
curl -i -X POST http://127.0.0.1:9600/widgets/spring
curl -i -X PATCH http://127.0.0.1:9600/widgets/spring
```

第一个请求打印 `{"widget":"gear","quantity":3,"currency":"USD"}`。第二个以 `201 Created` 和 `Location: /widgets/spring/1` 应答，第三个以 `405 Method Not Allowed` 和 `Allow: POST, PUT` 应答。

### 返回结果

动作可以返回：

- `Psr\Http\Message\ResponseInterface`，原样发送；
- 数组或 `JsonSerializable`，以 JSON 发送，带有 `Content-Type: application/json; charset=utf-8`，不转义斜杠和 Unicode；
- 标量或 `Stringable`，以文本发送，带有 `Content-Type: text/plain; charset=utf-8`；
- 不返回（`null`），发送 `ResponseContext` 中的当前响应。

其他值会抛出 `HTTP results must be a PSR-7 response, JSON-compatible data, a scalar, Stringable, or null; <type> provided.` 转换后的结果以当前响应为起点：保留其状态和自定义头，替换 `Content-Type` 和消息体，并删除过时的 `Content-Length`；转换后的响应不会被存回。因此，在返回数据之前通过 `ResponseContext::set()` 修改当前响应的代码（例如设置状态或 Cookie）会保留这些修改。结果在最内层中间件收到它之前完成转换，因此每个中间件收到的都是 `ResponseInterface`。

### 添加中间件

中间件实现 `Psr\Http\Server\MiddlewareInterface`：

- 不带名称的 `#[SuperKernel\HttpServer\Attribute\Middleware]` 把它应用到每个 HTTP 监听器，`#[Middleware(name: 'admin')]` 只应用到名为 `admin` 的监听器。二者也会为 `404` 和 `405` 响应运行。
- 控制器上的 `#[SuperKernel\HttpServer\Attribute\Middlewares([...])]` 为该控制器的路由列出中间件类，无论由哪个监听器服务。

中间件按以下顺序运行：所有全局中间件，然后是监听器的中间件，然后是控制器的中间件，每一层按构建发现它们的顺序或列表声明的顺序。没有数字优先级。中间件通过调用 `$handler->handle($request)` 继续，并且可以在调用之前处理请求、在调用之后处理响应。不调用处理器而直接返回响应会跳过其余的中间件和控制器。控制器抛出的异常不会经过中间件回溯：它直接进入异常处理器，因此中间件看不到那个响应。

中间件在服务器创建时由容器创建一次，并由所有请求和协程共享。不要在它的属性中保存请求数据；当服务需要当前消息时，请注入 `super-kernel/http-message` 的 `ServerRequestInterface` 或 `ResponseInterface`。

为每个请求和响应提供 ID 的全局中间件，保存为 `src/Middleware/RequestIdMiddleware.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Middleware;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;
use SuperKernel\HttpServer\Attribute\Middleware;

use function bin2hex;
use function random_bytes;

#[Middleware]
final readonly class RequestIdMiddleware implements MiddlewareInterface
{
    public function process(ServerRequestInterface $request, RequestHandlerInterface $handler): ResponseInterface
    {
        $requestId = $request->getHeaderLine('X-Request-Id') ?: bin2hex(random_bytes(8));

        return $handler->handle($request->withHeader('X-Request-Id', $requestId))
            ->withHeader('X-Request-Id', $requestId);
    }
}
```

用于一个控制器、无需注解的中间件，保存为 `src/Middleware/RequireApiKey.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Middleware;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;
use SuperKernel\HttpMessage\Response;

final readonly class RequireApiKey implements MiddlewareInterface
{
    public function process(ServerRequestInterface $request, RequestHandlerInterface $handler): ResponseInterface
    {
        if ('secret' !== $request->getHeaderLine('X-Api-Key')) {
            return new Response(401, ['Content-Type' => 'text/plain; charset=utf-8'], 'Missing API key');
        }

        return $handler->handle($request);
    }
}
```

使用它的控制器在接下来的两节中。

### 以流的方式发送响应

响应的消息体根据其 PSR-7 能力决定如何写出：

- `uri` 元数据指向可读普通文件的消息体（例如 `FileStream`），通过 Swoole 的 `sendfile()` 发送；
- 可定位且大小已知的消息体（例如 `StringStream`）会被回绕，并在一次写入中发送；
- 其他任何消息体（例如 `IterableStream`、`CallbackStream` 或管道）以 64 KiB 的块读取，每块读到后立即写出，因此事件流和大型生成下载会在生成过程中到达客户端，而慢速客户端只会挂起生成消息体的那个协程。

状态和每个头的值都按响应中的内容原样写出。对于文件和分块消息体，`Content-Length` 头会被删除，因为 Swoole 会计算文件的长度，并对写出的块使用分块传输编码；Swoole 只压缩在一次写入中发送的消息体。请求方法和状态永远不会改变消息体的写出方式。

写入失败会通过 `StdoutLoggerInterface` 记录，并结束响应而不再写入；`sendfile()` 失败时，只要响应仍可写，就改为分块发送消息体。准备或写出响应时出现的异常会被记录而不会抛出，响应也永远不会重新发送，因为它的一部分可能已经发出。一旦写出了一个块，状态和头就不能再改变。

带有事件流和一个会失败的动作、并由上面的中间件保护的控制器，保存为 `src/Controller/ReportController.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Controller;

use DomainException;
use Generator;
use Psr\Http\Message\ResponseInterface;
use Src\Middleware\RequireApiKey;
use SuperKernel\HttpMessage\Response;
use SuperKernel\HttpMessage\Stream\IterableStream;
use SuperKernel\HttpServer\Attribute\Controller;
use SuperKernel\HttpServer\Attribute\Middlewares;
use SuperKernel\HttpServer\Attribute\RequestMapping;
use Swoole\Coroutine;

#[Controller('/reports')]
#[Middlewares([
    RequireApiKey::class,
])]
final readonly class ReportController
{
    #[RequestMapping('/progress')]
    public function progress(): ResponseInterface
    {
        $events = static function (): Generator {
            foreach ([25, 50, 100] as $percent) {
                yield "data: {$percent}\n\n";
                Coroutine::sleep(0.2);
            }
        };

        return new Response(headers: ['Content-Type' => 'text/event-stream'], body: new IterableStream($events()));
    }

    #[RequestMapping('/archived')]
    public function archived(): never
    {
        throw new DomainException('Archived reports are read-only');
    }
}
```

运行 `skernel build` 和 `php bin/bootstrap.php swoole:start` 之后，分别带和不带密钥请求该流：

```bash
curl -i http://127.0.0.1:9600/reports/progress
curl -H 'X-Api-Key: secret' http://127.0.0.1:9600/reports/progress
```

第一个请求以 `401 Unauthorized`、`Missing API key` 和一个 `X-Request-Id` 头应答。第二个请求以五分之一秒的间隔打印三个事件，发送时带有 `Transfer-Encoding: chunked`：

```text
data: 25

data: 50

data: 100
```

### 处理异常

异常处理器实现 `SuperKernel\HttpServer\Contract\HttpExceptionHandlerInterface`，并用 `#[SuperKernel\HttpServer\Attribute\HttpExceptionHandler]` 标记，该注解接收：

- `name`：它服务的监听器，默认为 `http`，必须完全匹配；
- `priority`：默认为 `0`；优先级高的先运行，优先级相同的保持构建发现它们的顺序；
- `exception`：它处理的异常类，默认为 `Throwable::class`；子类也会匹配。

一个监听器可以有多个处理器。对于一个异常，请求所属监听器中匹配的处理器按顺序运行。`handle(Throwable $throwable, ResponseInterface $response)` 接收异常和当前为止的响应并返回一个响应，下一个处理器会收到它。第一个响应以纯文本形式从当前响应派生：对于 `HttpException`，它带有异常的状态和头，并以其消息作为消息体；对于其他任何异常，它是消息体为 `Internal Server Error` 的 `500`。同时实现了 `SuperKernel\HttpServer\Contract\StoppableHttpExceptionHandlerInterface` 的处理器会以同一个异常和它返回的响应被询问 `isPropagationStopped()`；只有 `true` 会结束处理链并发送该响应。失败的处理器，或者其决定失败的处理器，会被报告，处理链以最新的响应继续。不匹配的处理器完全不会被调用。

当没有处理器停止处理链时（包括根本没有处理器时），请求以安全的响应结束：`HttpException` 保留它的状态、消息和头，其他任何异常变为不含细节的纯文本 `500 Internal Server Error`。该异常会被报告给 `super-kernel/exception-dispatcher` 的通用异常分发器，由它记录；处理器对响应所做的修改会被丢弃。这个边界会捕获请求的每个异常，因此失败的请求永远不会结束工作进程，也不会影响其他请求；它无法捕获内存耗尽之类的致命错误。

抛出 `SuperKernel\HttpServer\Exception\HttpException` 以某个状态应答：`new HttpException(403, 'Forbidden')`、`HttpException::notFound()`，或者 `HttpException::methodNotAllowed(['GET', 'POST'])`，其头以列表形式携带这些方法。

处理 `http` 监听器上领域错误的处理器，保存为 `src/ExceptionHandler/DomainExceptionHandler.php`：

```php
<?php

declare(strict_types=1);

namespace Src\ExceptionHandler;

use DomainException;
use Psr\Http\Message\ResponseInterface;
use SuperKernel\HttpMessage\Stream\StringStream;
use SuperKernel\HttpServer\Attribute\HttpExceptionHandler;
use SuperKernel\HttpServer\Contract\StoppableHttpExceptionHandlerInterface;
use Throwable;

#[HttpExceptionHandler(exception: DomainException::class)]
final readonly class DomainExceptionHandler implements StoppableHttpExceptionHandlerInterface
{
    public function handle(Throwable $throwable, ResponseInterface $response): ResponseInterface
    {
        return $response
            ->withStatus(422)
            ->withHeader('Content-Type', 'text/plain; charset=utf-8')
            ->withBody(new StringStream($throwable->getMessage()));
    }

    public function isPropagationStopped(Throwable $throwable, ResponseInterface $response): bool
    {
        return true;
    }
}
```

运行 `skernel build` 之后，`curl -i -H 'X-Api-Key: secret' http://127.0.0.1:9600/reports/archived` 以 `422 Unprocessable Content` 和 `Archived reports are read-only` 应答。该响应没有 `X-Request-Id`，因为控制器异常不会回到中间件中。

骨架的 `Src\ExceptionHandler\Http\HttpServerExceptionHandler` 为 `http` 监听器处理 `HttpException`：它记录一条警告，以 JSON 的 `error` 字段和 `Content-Type: application/json; charset=utf-8` 应答，保留状态和头，并停止处理链。其他异常会绕过它，以安全的 `500` 结束。

### 未知的路径和方法

没有路由匹配时，请求变为 `new HttpException(404, 'Not Found')`；路径匹配但方法不同时，它变为 `new HttpException(405, 'Method Not Allowed', ['Allow' => 'POST, PUT'])`，允许的方法按路由顺序连接成一个字符串，因此响应只有一行 `Allow`。这些异常按相同的规则经过监听器的处理器。没有停止处理链的处理器时，默认响应是带有 `Content-Type: text/plain; charset=utf-8` 的纯文本 `404 Not Found` 或 `405 Method Not Allowed`，`405` 还带有 `Allow`；处理器所做的修改会被丢弃，路由异常也不会被报告。全局中间件和监听器的中间件会收到该响应，无论它来自处理器还是默认响应，并且不会解析任何控制器。

在骨架中，`curl -i http://127.0.0.1:9600/missing` 以 `404` 和 `{"error":"Not Found"}` 应答，`curl -i -X DELETE http://127.0.0.1:9600/user` 以 `405`、一个 `Allow: GET` 头和 `{"error":"Method Not Allowed"}` 应答，每个请求在日志中留下一条警告。

不带 `exception:` 的处理器会匹配每个异常，包括这些 `404` 和 `405`，因此会停止处理链的通配处理器会替换默认的文本响应。要保留它们，请声明处理器所针对的异常，或者让 `isPropagationStopped()` 对 `statusCode` 为 `404` 或 `405` 的 `HttpException` 返回 `false`。这个决定也会让应用代码抛出的 `404` 和 `405` 继续传递，并且不会撤销 `handle()` 已经做过的事情，例如写入日志记录。

### 在自己的代码中使用异常处理器

`SuperKernel\HttpServer\Factory\HttpExceptionDispatcherFactory` 在创建时收集所有处理器，并按监听器分组。`forListener($name)` 返回该名称对应的 `SuperKernel\HttpServer\Dispatcher\HttpExceptionDispatcher`，对于没有处理器的名称则返回空处理链。`dispatch(Throwable $throwable, ResponseInterface $response)` 按上述方式运行处理链，并返回停止它的处理器的响应；没有处理器停止它时，会原样重新抛出原来的异常。`recover()` 是请求边界使用的方法：它从不抛出异常，并返回安全的响应。也可以用 `new HttpExceptionDispatcher($registrations, $exceptionDispatcher)` 直接创建分发器，按给定顺序传入 `SuperKernel\ExceptionDispatcher\Registry\ExceptionHandlerRegistration` 对象和通用的 `SuperKernel\Contract\ExceptionDispatcherInterface`。

### 在没有服务器的情况下运行流水线

容器把 `Psr\Http\Server\RequestHandlerInterface` 作为路由和中间件流水线提供。请求必须在由 `SuperKernel\HttpServer\Constants\RequestAttributeName::Listener` 命名的属性 `super-kernel.server.listener` 中携带监听器的配置数组，其中包含非空字符串 `name`；服务器会把它加到每个请求上。当前响应必须保存在 `ResponseContext` 中，因为结果是从它转换来的。流水线自己会把请求保存到 `RequestContext`。

不启动服务器、让两个请求经过流水线的脚本，保存为 `bin/handle.php`：

```php
<?php

declare(strict_types=1);

use Psr\Http\Server\RequestHandlerInterface;
use SuperKernel\Context\ApplicationContext;
use SuperKernel\HttpMessage\Context\ResponseContext;
use SuperKernel\HttpMessage\Response;
use SuperKernel\HttpMessage\ServerRequest;
use SuperKernel\HttpServer\Constants\RequestAttributeName;

require dirname(__DIR__) . '/vendor/autoload.php';

$handler = ApplicationContext::getContainer()->get(RequestHandlerInterface::class);

foreach (['/greetings/Ada', '/missing'] as $path) {
    $request = new ServerRequest('GET', $path)
        ->withAttribute(RequestAttributeName::Listener->value, ['name' => 'http']);

    ResponseContext::set(new Response());

    $response = $handler->handle($request);

    echo $response->getStatusCode(), ' ', $response->getBody(), PHP_EOL;
}
```

在带有上面 `GreetingController` 的骨架中，运行 `skernel build` 之后运行它：

```bash
php bin/handle.php
```

它会打印以下内容，骨架处理器为第二个请求写出的警告出现在日志中：

```text
200 {"greeting":"Hello, Ada!"}
404 {"error":"Not Found"}
```

`SuperKernel\HttpServer\Provider\MiddlewareProvider` 是流水线的中间件；自行组装流水线的代码用容器、`SuperKernel\HttpServer\Response\ResponseConverter` 和 `HttpExceptionDispatcherFactory` 创建它。

### 请求如何到达流水线

当 `super-kernel/server` 为以 `Protocol::Http` 或 `Protocol::WebSocket` 配置的监听器发布 `AfterServerCreated` 时，本包设置该监听器的 Swoole `request` 回调，并绑定该监听器的配置。原始监听器和协程模式的服务器保持不变。带有非空 `Upgrade` 头的请求不会被处理，因此 WebSocket 握手会到达 WebSocket 代码。

对于每个请求，回调在 `ResponseContext` 中保存一个空的 `SuperKernel\HttpMessage\Response`，把 Swoole 请求转换为 `SuperKernel\HttpMessage\ServerRequest`，把监听器的配置数组作为 `super-kernel.server.listener` 属性加上，把请求保存到 `RequestContext`，然后运行流水线。之后它把响应写出一次，最后只清除上下文中的请求键和响应键；其他上下文值保持不变。请求把 Swoole 的数组复制为标量、`null` 和嵌套数组，以只读流提供消息体，并拒绝对象和资源。两个消息都不持有原生 Swoole 请求或响应，因此中间件、控制器和服务都无法访问它们。

`Host` 头在创建请求之前检查。它必须是 `host[:port]`：注册名（可以包含有效的百分号转义）、IPv4 地址，或者带方括号的 IPv6 地址。主机会转换为小写，结尾的点会保留；端口由 `0` 到 `65535` 的十进制数字组成，空端口会被接受并省略。以下内容会被拒绝：空主机、不带方括号的 IPv6 地址、格式错误的方括号、无效端口、空白字符、控制字符、原始的 `/`、`?`、`#`、`@` 或反斜杠、IPvFuture 字面量和 IPv6 区域标识符。Swoole 只保留最后一个 `Host` 值，因此对于 HTTP/1.0 和 HTTP/1.1，会检查原始头行中是否有不区分大小写的重复 `Host`，即使两行完全相同也会被拒绝；消息体不计算在内。这检查的是授权部分的格式，而不是允许的域名列表。

无效的或重复的 `Host`，或者 HTTP/1.1 中缺失的 `Host`（[RFC 9112 §3.2](https://www.rfc-editor.org/rfc/rfc9112.html#section-3.2) 要求必须有），会抛出 `HttpException(400, 'Bad Request')`，其消息从不包含输入内容。它在路由和中间件之前经过监听器的异常处理器，并且不会解析任何控制器；此时 `RequestContext` 尚未设置，因此处理器只能使用异常和初始的 `400` 响应。没有停止处理链的处理器时，应答为纯文本的 `400 Bad Request`。在骨架中，`curl -H 'Host: bad host' http://127.0.0.1:9600/user` 以 `400` 和 `{"error":"Bad Request"}` 应答。

URI 由各个部分通过 PSR-7 的 `with*()` 方法构建，从不通过解析拼接后的 URL。路径是带有 `Uri` 编码的 Swoole `request_uri`：路径中的字面 `#` 或 `?` 会被编码，已有的 `%23` 和 `%3F` 保持不变，开头的 `//` 会保留。Swoole 在第一个 `?` 处拆分请求目标；之后的问号属于查询，查询中的 `#` 会被编码，而不是被当作片段。绝对形式的请求目标（例如 `http://other.test/user`）保持为字面路径，既不会路由到 `/user`，也不会替换 `Host`。没有 `Host` 的 HTTP/1.0 请求依次使用 Swoole 的 `server_name`、`server_addr`、`localhost`，不带端口。Swoole 把 HTTP/2 的 `:authority` 作为 `host` 传入，因此适用相同的检查，没有它的 HTTP/2 请求使用相同的回退；除非服务器设置启用 HTTP/2，否则它是关闭的。

## 错误

本包的每个异常（包括 `HttpException`）都实现 `SuperKernel\HttpServer\Exception\HttpServerExceptionInterface`。

| 情形 | 消息 |
| --- | --- |
| HTTP 监听器的 `request` 回调已经被设置或无法设置 | `Unable to claim the HTTP request callback for the configured Swoole service "<name>".`（`RequestCallbackRegistrationException`） |
| `Controller` 中的监听器名称为空 | `HTTP controller server name must be non-empty.` |
| `Middleware` 中的名称为空 | `HTTP middleware server name must be non-empty when provided.` |
| `Middlewares` 中的列表无效 | `Middlewares must be a list of non-empty class names without NUL bytes.` |
| `HttpExceptionHandler` 中的名称为空 | `HTTP exception-handler service name must be non-empty.` |
| 无法实例化的控制器 | `HTTP controller <class> must be instantiable.` |
| 没有实现该接口的中间件 | `HTTP middleware <class> declared by <class> must implement Psr\Http\Server\MiddlewareInterface.` |
| 没有实现该接口的处理器 | `HTTP exception handler <class> must implement SuperKernel\HttpServer\Contract\HttpExceptionHandlerInterface; <type> provided.` |
| 一个监听器上的路由冲突 | `Unable to build the HTTP route table.`，FastRoute 给出的原因作为前一个异常（previous） |
| 容器返回的不是该控制器 | `HTTP controller <class> must resolve to a compatible instance, <type> returned.` |
| 无法转换的路由值 | `Route value <value> cannot be converted for parameter $<name>.` |
| 无法填充的动作参数 | `Unable to resolve parameter $<name> of <class>::<method>().` |
| 动作返回不支持的值 | `HTTP results must be a PSR-7 response, JSON-compatible data, a scalar, Stringable, or null; <type> provided.` |
| 没有监听器属性的请求 | `SuperKernel\HttpServer\Provider\RequestHandlerProvider requires the super-kernel.server.listener request attribute to be an array with a non-empty string name; <type> provided.` |

声明错误是 `SuperKernel\HttpServer\Exception\InvalidHttpDeclarationException`；注解错误是构建或容器构造注解时抛出的 `InvalidArgumentException`。在请求期间，每个错误都会到达监听器的异常处理器，没有停止处理链的处理器时以安全的 `500` 结束。

## 与其他包的关系

- [`super-kernel/server`](server.zh-CN.md) 创建监听器并发布 `AfterServerCreated`，本包在其中设置 `request` 回调。
- [`super-kernel/http-message`](http-message.zh-CN.md) 提供 PSR-7 消息、流、`RequestContext`、`ResponseContext`，以及可注入的 `ServerRequestInterface` 和 `ResponseInterface`。
- [`super-kernel/exception-dispatcher`](exception-dispatcher.zh-CN.md) 提供 `HttpExceptionHandler` 所继承的 `ExceptionHandler` 注解、处理器注册表，以及报告未处理异常的通用分发器。
- [`super-kernel/di`](di.zh-CN.md) 创建控制器、中间件、处理器和动作依赖；[`super-kernel/annotator`](annotator.zh-CN.md) 提供 `skernel build` 收集的声明。
- [`super-kernel/swoole`](swoole.zh-CN.md) 通过 `swoole:start` 启动服务器。

## 不要这样做

**不要为 `Psr\Http\Server\RequestHandlerInterface` 声明另一个提供者。** 本包的 `RequestHandlerProvider` 就是该标识符的提供者；第二个提供者会破坏容器的查找。请改用中间件添加行为。

**不要自己设置 HTTP 监听器的 `request` 回调。** 如果在本包占用它之前设置，服务器不会启动，并报告 `Unable to claim the HTTP request callback for the configured Swoole service "<name>".`；如果在之后设置，它会悄无声息地替换路由、中间件和异常处理。请为自己的协议使用 `Raw` 监听器。

**不要在中间件、控制器或处理器中保存请求数据。** 它们只创建一次，并由所有请求和协程共享，因此为一个请求写入的属性会被下一个请求看到。请用 `withAttribute()` 在请求上传递数据，或者注入 `super-kernel/http-message` 的当前请求服务。

**不要在中间件的构造函数中解析流水线或启动服务器。** 中间件在服务器创建期间、服务器运行之前创建；在那里向容器请求 `RequestHandlerInterface` 或启动 `ServerInterface` 会递归进入正在构建的服务。请在 `process()` 中做这些工作。

**不要把 `RequestMapping` 放在继承来的或静态的方法上。** 只有控制器类自己声明的公开非静态方法才会被路由；其他方法会被忽略，且不报错。请在控制器中声明该动作。

**不要依赖枚举或类型化参数来拒绝路由值。** 无法转换的值以 `500` 而不是 `404` 结束。请用正则表达式限制路由变量，例如 `{widget:gear|spring}`。

**除非确有此意，不要在通配处理器中停止所有异常。** 不带 `exception:` 的处理器也会收到路由的 `404` 和 `405`，以及错误 `Host` 引起的 `400`，会停止处理链的处理器会替换它们的响应。请声明它所针对的异常类。

**不要指望控制器异常会经过你的中间件。** 它直接进入异常处理器，因此中间件在返回途中添加的头不会出现在那个响应中。请也在异常处理器中添加这些头。
