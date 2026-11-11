<p align="right">
  <strong>English</strong> | <a href="http-server.zh-CN.md">中文文档</a>
</p>

# SuperKernel HTTP Server

Package: [`super-kernel/http-server`](https://github.com/super-kernel/http-server) · [Packagist](https://packagist.org/packages/super-kernel/http-server) · [Documentation index](../README.md)

## Purpose

`super-kernel/http-server` is the HTTP layer of a SuperKernel application. For every HTTP or WebSocket listener of the server that `super-kernel/server` starts, it takes each Swoole request, checks it, converts it into a PSR-7 `ServerRequest`, runs it through PSR-15 middleware to a controller action, converts the action's result into a response, and writes that response back through Swoole. Exceptions on the way become responses through exception handlers declared per listener.

The PSR-7 objects and the current-request services belong to `super-kernel/http-message`. Creating the server and its listeners belongs to `super-kernel/server`. WebSocket messages, raw TCP, and coroutine-mode servers are left to other code.

## Capabilities

### Route requests to controllers

Mark a class with `#[SuperKernel\HttpServer\Attribute\Controller]` and each public action with one or more `#[SuperKernel\HttpServer\Attribute\RequestMapping]`:

- `Controller` takes the route prefix first and the listener name `server`, `http` by default. The name must match a listener's `name` in the server configuration exactly, including case. Each listener has its own route table, so the same method and path may exist on two listeners, and a controller is invisible to the other listeners.
- `RequestMapping` takes the path and a `SuperKernel\HttpServer\Constants\HttpMethod` case, `HttpMethod::Get` by default; repeat it for more methods. A method string is not accepted. Only public, non-static methods that the controller class declares itself are routed; an inherited or static method with `RequestMapping` is ignored.
- The full path is the prefix followed by the route path, each starting with `/`. A route path `''` or `/` therefore answers at `<prefix>/` with a trailing slash, not at the bare prefix.
- Paths use the FastRoute syntax: `{name}` matches one segment and `{name:\d+}` a regular expression. A conflict in a listener's route table, such as the same method and path twice, stops the server at startup with `Unable to build the HTTP route table.`

Action parameters are filled in this order: a parameter named like a route variable receives that value, converted to the parameter's `string`, `int`, `float`, `bool`, or backed enum type; a parameter typed `Psr\Http\Message\ServerRequestInterface` receives the current request; a parameter typed with one other class or interface is resolved from the container; otherwise a parameter keeps its default value or, when nullable, receives `null`. Any other parameter raises `Unable to resolve parameter $<name> of <class>::<method>().` The route variables are also request attributes. A value that cannot be converted, such as an unknown enum value, raises `Route value <value> cannot be converted for parameter $<name>.` and is answered with `500`; constrain the route variable with a regular expression when other values are possible.

The controller itself is resolved from the container, and the result must be an instance of the controller class or a subclass, such as a proxy. Another object, a scalar, or `null` raises `HTTP controller <class> must resolve to a compatible instance, <type> returned.` before the action's parameters are resolved. Exceptions from the container propagate unchanged; a missing, private, or protected action, or an argument of the wrong type, raises PHP's own error.

The smallest controller, whose array result becomes a JSON response, saved as `src/Controller/GreetingController.php`:

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

An enum for route values, saved as `src/Catalog/Widget.php`:

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

A controller with typed route variables, the current request, and two methods on one action, saved as `src/Controller/WidgetController.php`:

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

In an application created from `super-kernel/skeleton`, build and start it, then send requests from another terminal:

```bash
skernel build
php bin/bootstrap.php swoole:start
curl 'http://127.0.0.1:9600/widgets/gear/3?currency=USD'
curl -i -X POST http://127.0.0.1:9600/widgets/spring
curl -i -X PATCH http://127.0.0.1:9600/widgets/spring
```

The first request prints `{"widget":"gear","quantity":3,"currency":"USD"}`. The second answers `201 Created` with `Location: /widgets/spring/1`, and the third `405 Method Not Allowed` with `Allow: POST, PUT`.

### Return results

An action may return:

- a `Psr\Http\Message\ResponseInterface`, which is sent as it is;
- an array or a `JsonSerializable`, sent as JSON with `Content-Type: application/json; charset=utf-8`, without escaped slashes or Unicode;
- a scalar or a `Stringable`, sent as text with `Content-Type: text/plain; charset=utf-8`;
- nothing (`null`), which sends the current response of `ResponseContext`.

Any other value raises `HTTP results must be a PSR-7 response, JSON-compatible data, a scalar, Stringable, or null; <type> provided.` A converted result starts from the current response: it keeps its status and custom headers, replaces `Content-Type` and the body, and drops a stale `Content-Length`; the converted response is not stored back. Code that changes the current response through `ResponseContext::set()` before returning data, for example to set a status or a cookie, therefore keeps those changes. The result is converted before the innermost middleware receives it, so every middleware receives a `ResponseInterface`.

### Add middleware

Middleware implements `Psr\Http\Server\MiddlewareInterface`:

- `#[SuperKernel\HttpServer\Attribute\Middleware]` without a name applies it to every HTTP listener, and `#[Middleware(name: 'admin')]` to the listener named `admin` only. Both also run for `404` and `405` responses.
- `#[SuperKernel\HttpServer\Attribute\Middlewares([...])]` on a controller lists middleware classes for that controller's routes, whichever listener serves them.

Middleware runs in this order: every global middleware, then the middleware of the listener, then the controller's middleware, each layer in the order the build found it or the list declares it. There is no numeric priority. A middleware continues by calling `$handler->handle($request)`, and can work on the request before and on the response after that call. Returning a response without calling the handler skips the remaining middleware and the controller. An exception from the controller does not unwind through the middleware: it goes to the exception handlers directly, so middleware does not see that response.

Middleware is created from the container once, when the server is created, and shared by every request and coroutine. Keep no request data in its properties; inject `ServerRequestInterface` or `ResponseInterface` from `super-kernel/http-message` where a service needs the current messages.

A global middleware that gives every request and response an ID, saved as `src/Middleware/RequestIdMiddleware.php`:

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

A middleware for one controller, which needs no Attribute, saved as `src/Middleware/RequireApiKey.php`:

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

The controller that uses it is in the next two sections.

### Stream responses

The response's body decides how it is written, from its PSR-7 capabilities:

- a body whose `uri` metadata names a readable regular file, such as a `FileStream`, is sent with Swoole's `sendfile()`;
- a seekable body with a known size, such as a `StringStream`, is rewound and sent in one write;
- any other body, such as an `IterableStream`, a `CallbackStream`, or a pipe, is read in chunks of 64 KiB and each chunk is written as soon as it is read, so event streams and large generated downloads reach the client while they are produced, and a slow client suspends only the coroutine that produces the body.

The status and every header value are written exactly as the response holds them. For file and chunked bodies a `Content-Length` header is dropped, because Swoole computes the length of a file and uses chunked transfer encoding for written chunks; Swoole compresses only bodies sent in one write. The request method and the status never change how the body is written.

A failed write is logged through `StdoutLoggerInterface` and ends the response without another write; when `sendfile()` fails, the body is sent in chunks while the response can still be written. An exception while the response is prepared or written is logged and not thrown, and the response is never sent again, because part of it may already be on the way. Once a chunk has been written, the status and headers can no longer change.

A controller with an event stream and an action that fails, protected by the middleware above, saved as `src/Controller/ReportController.php`:

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

After `skernel build` and `php bin/bootstrap.php swoole:start`, request the stream with and without the key:

```bash
curl -i http://127.0.0.1:9600/reports/progress
curl -H 'X-Api-Key: secret' http://127.0.0.1:9600/reports/progress
```

The first request answers `401 Unauthorized` with `Missing API key` and an `X-Request-Id` header. The second prints the three events a fifth of a second apart, sent with `Transfer-Encoding: chunked`:

```text
data: 25

data: 50

data: 100
```

### Handle exceptions

An exception handler implements `SuperKernel\HttpServer\Contract\HttpExceptionHandlerInterface` and is marked with `#[SuperKernel\HttpServer\Attribute\HttpExceptionHandler]`, which takes:

- `name`, the listener it serves, `http` by default, matched exactly;
- `priority`, `0` by default; higher priorities run first, and equal priorities keep the order the build found them in;
- `exception`, the exception class it handles, `Throwable::class` by default; subclasses match too.

A listener may have several handlers. For an exception, the matching handlers of the request's listener run in order. `handle(Throwable $throwable, ResponseInterface $response)` receives the exception and the response so far and returns a response, which the next handler receives. The first response is derived from the current response as plain text: for an `HttpException` it has the exception's status and headers and its message as the body; for any other exception it is `500` with the body `Internal Server Error`. A handler that also implements `SuperKernel\HttpServer\Contract\StoppableHttpExceptionHandlerInterface` is asked `isPropagationStopped()` with the same exception and the response it returned; only `true` ends the chain and sends that response. A handler that fails, or whose decision fails, is reported, and the chain continues with the latest response. Handlers that do not match are not called at all.

When no handler stops the chain, including when there are none, the request ends with a safe response: an `HttpException` keeps its status, message, and headers, and any other exception becomes `500 Internal Server Error` as plain text without details. The exception is reported to the generic exception dispatcher of `super-kernel/exception-dispatcher`, which logs it; changes that handlers made to the response are discarded. This boundary catches every exception of a request, so a failing request never ends the worker or affects other requests; it cannot catch a fatal error such as exhausted memory.

Throw `SuperKernel\HttpServer\Exception\HttpException` to answer with a status: `new HttpException(403, 'Forbidden')`, `HttpException::notFound()`, or `HttpException::methodNotAllowed(['GET', 'POST'])`, whose headers carry the methods as a list.

A handler for domain errors on the `http` listener, saved as `src/ExceptionHandler/DomainExceptionHandler.php`:

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

After `skernel build`, `curl -i -H 'X-Api-Key: secret' http://127.0.0.1:9600/reports/archived` answers `422 Unprocessable Content` with `Archived reports are read-only`. The response has no `X-Request-Id`, because a controller exception does not pass back through the middleware.

The skeleton's `Src\ExceptionHandler\Http\HttpServerExceptionHandler` handles `HttpException` for the `http` listener: it logs a warning, answers with a JSON `error` field and `Content-Type: application/json; charset=utf-8`, keeps the status and headers, and stops the chain. Other exceptions pass it by and end as the safe `500`.

### Unknown paths and methods

When no route matches, the request becomes `new HttpException(404, 'Not Found')`; when the path matches with another method, it becomes `new HttpException(405, 'Method Not Allowed', ['Allow' => 'POST, PUT'])`, with the allowed methods in route order joined into one string, so the response has one `Allow` line. These exceptions go through the listener's handlers with the same rules. Without a stopping handler, the default response is `404 Not Found` or `405 Method Not Allowed` as plain text with `Content-Type: text/plain; charset=utf-8`, plus `Allow` for `405`; handler changes are discarded, and the routing exception is not reported. Global middleware and the listener's middleware receive the response, whether it came from a handler or is the default, and no controller is resolved.

In the skeleton, `curl -i http://127.0.0.1:9600/missing` answers `404` with `{"error":"Not Found"}`, and `curl -i -X DELETE http://127.0.0.1:9600/user` answers `405` with one `Allow: GET` header and `{"error":"Method Not Allowed"}`, each with one warning in the log.

A handler without `exception:` matches every exception, including these `404` and `405`, so a stopping catch-all handler replaces the default text responses. To keep them, declare the exception the handler is for, or let `isPropagationStopped()` return `false` for an `HttpException` whose `statusCode` is `404` or `405`. That decision also lets `404` and `405` that application code throws continue, and it does not undo what `handle()` already did, such as writing a log record.

### Use the exception handlers in your own code

`SuperKernel\HttpServer\Factory\HttpExceptionDispatcherFactory` collects every handler when it is created and groups them by listener. `forListener($name)` returns the `SuperKernel\HttpServer\Dispatcher\HttpExceptionDispatcher` for that exact name, or an empty chain for a name without handlers. `dispatch(Throwable $throwable, ResponseInterface $response)` runs the chain as described above and returns the response of the handler that stopped it; when no handler stops it, it rethrows the original exception unchanged. `recover()` is what the request boundary uses: it never throws and returns the safe response. A dispatcher can also be created directly with `new HttpExceptionDispatcher($registrations, $exceptionDispatcher)`, from `SuperKernel\ExceptionDispatcher\Registry\ExceptionHandlerRegistration` objects in the order given and the generic `SuperKernel\Contract\ExceptionDispatcherInterface`.

### Run the pipeline without the server

The container provides `Psr\Http\Server\RequestHandlerInterface` as the routing and middleware pipeline. A request must carry the listener's configuration array, with a non-empty string `name`, in the attribute `super-kernel.server.listener`, named by `SuperKernel\HttpServer\Constants\RequestAttributeName::Listener`; the server adds it to every request. The current response must be stored in `ResponseContext`, because results are converted from it. The pipeline stores the request in `RequestContext` itself.

A script that sends two requests through the pipeline without starting a server, saved as `bin/handle.php`:

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

In the skeleton with the `GreetingController` above, run it after `skernel build`:

```bash
php bin/handle.php
```

It prints the following, with the skeleton handler's warning for the second request in the log:

```text
200 {"greeting":"Hello, Ada!"}
404 {"error":"Not Found"}
```

`SuperKernel\HttpServer\Provider\MiddlewareProvider` is the middleware of the pipeline; code that assembles a pipeline itself creates it with the container, a `SuperKernel\HttpServer\Response\ResponseConverter`, and the `HttpExceptionDispatcherFactory`.

### How requests reach the pipeline

When `super-kernel/server` publishes `AfterServerCreated` for a listener configured with `Protocol::Http` or `Protocol::WebSocket`, this package sets that listener's Swoole `request` callback, bound to the listener's configuration. Raw listeners and coroutine-mode servers are left alone. A request with a non-empty `Upgrade` header is not handled, so a WebSocket handshake reaches the WebSocket code.

For each request, the callback stores an empty `SuperKernel\HttpMessage\Response` in `ResponseContext`, converts the Swoole request into a `SuperKernel\HttpMessage\ServerRequest`, adds the listener's configuration array as the `super-kernel.server.listener` attribute, stores the request in `RequestContext`, and runs the pipeline. It then writes the response once and finally clears only the request and response keys of the context; other context values stay. The request copies Swoole's arrays as scalars, `null`, and nested arrays, offers the body as a read-only stream, and rejects objects and resources. Neither message holds the native Swoole request or response, so middleware, controllers, and services cannot reach them.

The `Host` header is checked before the request is created. It must be `host[:port]`: a registered name, which may contain valid percent escapes, an IPv4 address, or an IPv6 address in brackets. The host is lowercased and a trailing dot is kept; a port has decimal digits from `0` to `65535`, and an empty port is accepted and left out. Rejected are an empty host, an IPv6 address without brackets, malformed brackets, invalid ports, whitespace, control characters, a raw `/`, `?`, `#`, `@`, or backslash, IPvFuture literals, and IPv6 zone identifiers. Swoole keeps only the last `Host` value, so for HTTP/1.0 and HTTP/1.1 the raw header lines are checked for a repeated `Host`, in any case, and even two identical lines are rejected; the body is not counted. This checks the form of the authority; it is not a list of allowed domains.

An invalid or repeated `Host`, or a missing `Host` in HTTP/1.1, as [RFC 9112 §3.2](https://www.rfc-editor.org/rfc/rfc9112.html#section-3.2) requires, raises `HttpException(400, 'Bad Request')`, whose message never contains the input. It goes through the listener's exception handlers before routing and middleware, and no controller is resolved; `RequestContext` is not set at that point, so handlers work with the exception and the initial `400` response. Without a stopping handler, the answer is `400 Bad Request` as plain text. In the skeleton, `curl -H 'Host: bad host' http://127.0.0.1:9600/user` answers `400` with `{"error":"Bad Request"}`.

The URI is built from its parts with PSR-7 `with*()` methods and never by parsing a joined URL. The path is Swoole's `request_uri` with the encoding of `Uri`: a literal `#` or `?` in the path is encoded, existing `%23` and `%3F` stay as they are, and a leading `//` is kept. Swoole splits the target at the first `?`; later question marks belong to the query, and a `#` in the query is encoded, not taken as a fragment. A request target in absolute form, such as `http://other.test/user`, stays the literal path and neither routes to `/user` nor replaces `Host`. An HTTP/1.0 request without `Host` uses Swoole's `server_name`, then `server_addr`, then `localhost`, without a port. Swoole passes the HTTP/2 `:authority` as `host`, so the same checks apply, and an HTTP/2 request without one uses the same fallback; HTTP/2 is off unless the server settings enable it.

## Errors

Every exception of this package, including `HttpException`, implements `SuperKernel\HttpServer\Exception\HttpServerExceptionInterface`.

| Situation | Message |
| --- | --- |
| The `request` callback of an HTTP listener is already set or cannot be set | `Unable to claim the HTTP request callback for the configured Swoole service "<name>".` (`RequestCallbackRegistrationException`) |
| An empty listener name in `Controller` | `HTTP controller server name must be non-empty.` |
| An empty name in `Middleware` | `HTTP middleware server name must be non-empty when provided.` |
| An invalid list in `Middlewares` | `Middlewares must be a list of non-empty class names without NUL bytes.` |
| An empty name in `HttpExceptionHandler` | `HTTP exception-handler service name must be non-empty.` |
| A controller that cannot be instantiated | `HTTP controller <class> must be instantiable.` |
| Middleware that does not implement the interface | `HTTP middleware <class> declared by <class> must implement Psr\Http\Server\MiddlewareInterface.` |
| A handler that does not implement the interface | `HTTP exception handler <class> must implement SuperKernel\HttpServer\Contract\HttpExceptionHandlerInterface; <type> provided.` |
| Conflicting routes on one listener | `Unable to build the HTTP route table.`, with FastRoute's reason as the previous exception |
| The container returns something other than the controller | `HTTP controller <class> must resolve to a compatible instance, <type> returned.` |
| A route value that cannot be converted | `Route value <value> cannot be converted for parameter $<name>.` |
| An action parameter that cannot be filled | `Unable to resolve parameter $<name> of <class>::<method>().` |
| An action returns an unsupported value | `HTTP results must be a PSR-7 response, JSON-compatible data, a scalar, Stringable, or null; <type> provided.` |
| A request without the listener attribute | `SuperKernel\HttpServer\Provider\RequestHandlerProvider requires the super-kernel.server.listener request attribute to be an array with a non-empty string name; <type> provided.` |

Declaration errors are `SuperKernel\HttpServer\Exception\InvalidHttpDeclarationException`; the Attribute errors are `InvalidArgumentException` raised when the build or the container constructs the Attribute. During a request, every error reaches the listener's exception handlers and, without a stopping handler, ends as the safe `500`.

## Relation to other packages

- [`super-kernel/server`](server.md) creates the listeners and publishes `AfterServerCreated`, where this package sets the `request` callback.
- [`super-kernel/http-message`](http-message.md) provides the PSR-7 messages, streams, `RequestContext`, `ResponseContext`, and the injectable `ServerRequestInterface` and `ResponseInterface`.
- [`super-kernel/exception-dispatcher`](exception-dispatcher.md) provides the `ExceptionHandler` Attribute that `HttpExceptionHandler` extends, the handler registry, and the generic dispatcher that reports unhandled exceptions.
- [`super-kernel/di`](di.md) creates controllers, middleware, handlers, and action dependencies; [`super-kernel/annotator`](annotator.md) provides the declarations that `skernel build` collected.
- [`super-kernel/swoole`](swoole.md) starts the server with `swoole:start`.

## What not to do

**Do not declare another provider for `Psr\Http\Server\RequestHandlerInterface`.** The package's `RequestHandlerProvider` is that identifier's provider; a second one breaks the container's lookups. Add behavior with middleware instead.

**Do not set the `request` callback of an HTTP listener yourself.** If it is set before this package claims it, the server does not start, with `Unable to claim the HTTP request callback for the configured Swoole service "<name>".`; if it is set afterwards, it silently replaces routing, middleware, and exception handling. Give your own protocol a `Raw` listener instead.

**Do not keep request data in middleware, controllers, or handlers.** They are created once and shared by every request and coroutine, so a property written for one request is seen by the next. Pass data on the request with `withAttribute()`, or inject the current-request services of `super-kernel/http-message`.

**Do not resolve the pipeline or start the server in a middleware constructor.** Middleware is created while the server is created, before it runs; asking the container for `RequestHandlerInterface` or starting `ServerInterface` there recurses into the services being built. Do such work in `process()`.

**Do not put `RequestMapping` on inherited or static methods.** Only public, non-static methods that the controller class declares itself are routed; the others are ignored without an error. Declare the action in the controller.

**Do not rely on an enum or a typed parameter to reject a route value.** A value that cannot be converted ends in a `500`, not a `404`. Restrict the route variable with a regular expression, such as `{widget:gear|spring}`.

**Do not stop every exception in a catch-all handler unless you mean to.** A handler without `exception:` also receives the routing `404` and `405` and the `400` for a bad `Host`, and a stopping one replaces their responses. Declare the exception class it is for.

**Do not expect a controller exception to pass through your middleware.** It goes to the exception handlers directly, so headers that middleware adds on the way out are missing from that response. Add such headers in an exception handler too.
