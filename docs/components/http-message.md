<p align="right">
  <strong>English</strong> | <a href="http-message.zh-CN.md">中文文档</a>
</p>

# SuperKernel HTTP Message

Package: [`super-kernel/http-message`](https://github.com/super-kernel/http-message) · [Packagist](https://packagist.org/packages/super-kernel/http-message) · [Documentation index](../README.md)

## Purpose

`super-kernel/http-message` implements PSR-7 for SuperKernel: requests, server requests, responses, URIs, streams, and uploaded files that any PSR-7 consumer accepts. It also keeps the request and the response of the current scope, so that a shared service can read the request it is serving without receiving it as an argument.

The package has no server dependency. Turning a native server request into a `ServerRequest`, routing it, and sending the response belong to `super-kernel/http-server`. The package provides no `psr/http-factory` factories: create every object with its constructor.

## Capabilities

### Create messages

`Request`, `ServerRequest`, and `Response` take their data as constructor arguments: a request takes the method, the URI as a string or a `UriInterface`, headers, a body, and the protocol version; a server request adds server, cookie, query, uploaded-file, parsed-body, and attribute data; a response takes the status code, headers, a body, the protocol version, and a reason phrase. A body is a `StreamInterface`, a string, or `null` for an empty body.

The rules the objects keep:

- Every `with*()` and `without*()` method returns a changed copy and leaves the original unchanged. Copies share the body stream, and a stream has a cursor: after one copy reads the body, the other sees the moved cursor. Call `rewind()` before reading a body that another copy may have read.
- Header names keep the case they were given in `getHeaders()` and are found in any case. A header name must be an HTTP token, and a value must be a string or a list of strings without line breaks.
- A request without a `Host` header takes it from the URI. `withUri()` replaces `Host` from the new URI unless the request should preserve it, as PSR-7 specifies.
- A status code must be within `100` to `599`. Without a reason phrase, a response uses the standard one for its code, or none for a code without one; a custom reason phrase must not contain line breaks.
- A server request's attribute names must be strings, and its uploaded files must be `UploadedFileInterface` objects or nested arrays of them.

### Choose a stream

Each kind of body has its own stream class, created with its public constructor:

- `StringStream` copies a string into a temporary stream and rewinds it.
- `ResourceStream` takes an open native stream resource, such as the result of `fopen()`.
- `FileStream` opens a file with a mode, `rb` by default.
- `IterableStream` reads an iterable of string chunks, such as a generator, and skips empty chunks.
- `CallbackStream` calls a reader with the requested byte count; the reader returns a non-empty string, or `null` at the end of its source.
- `ReadOnlyStream` wraps another PSR-7 stream: it passes reads, seeks, metadata, closing, and detaching to that stream and rejects every write.

A stream that holds a native resource owns it. `close()` closes the resource, and `detach()` returns it to the caller, who then owns it. After either call, the stream reports no capabilities and raises `RuntimeException` for every operation that needs the resource.

`IterableStream` and `CallbackStream` are readable, forward-only, not seekable, not writable, and report an unknown size. Their `read()` returns as soon as the source has produced any bytes instead of waiting for the requested length, so a slow producer, such as a server-sent event source, reaches the consumer one chunk at a time; the part of a chunk larger than the request stays buffered for the next read. A chunk that is not a string, or an empty string from a reader, raises `RuntimeException` when it is read, and an exception from the source is kept as the previous exception. `close()` and `detach()` release the source; `detach()` returns `null`, because there is no resource.

A stream holds bytes only. Encode arrays and objects, for example with `json_encode()`, before putting them in a body. Casting any stream to a string never throws: it returns the remaining contents, or `''` after a failure or once the stream is closed.

A script that reads each kind of stream, saved as `bin/streams.php`:

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

Run the script:

```bash
php bin/streams.php
```

It prints:

```text
"data: first\n\n"
"data"
NULL false
321
The read-only stream is not writable.
payload
```

The first `read(1024)` returns only the first chunk, and the second read takes four bytes of the next one.

### Accept uploaded files

`UploadedFile` takes its source first: a `StreamInterface`, the path of a temporary file, or `null` for an upload that failed. Then it takes the size, the PHP upload error code (`UPLOAD_ERR_OK` by default), and the file name and media type the client sent. A successful upload needs a source; an unknown error code or a negative size is rejected.

`moveTo()` writes the file to a target path. A temporary path is moved with `move_uploaded_file()` when PHP received the file in this request, and renamed otherwise; a stream is copied and then closed. After a successful move, `getStream()` and another `moveTo()` raise `RuntimeException`. On a failed upload, both raise `RuntimeException` with the error code.

A script that moves an upload, saved as `bin/upload.php`:

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

Run the script:

```bash
php bin/upload.php
```

It prints:

```text
name,total
The uploaded file has already been moved.
```

### Work with URIs

`new Uri($string)` parses a URI; an empty string is an empty URI. `Uri` lowercases the scheme and the host, leaves out port 80 for `http` and port 443 for `https` from the authority and from `getPort()`, accepts ports from `0` to `65535`, keeps valid percent-encoded octets, and percent-encodes other invalid bytes once. It renders the authority and the path together as PSR-7 specifies. For example, `new Uri('HTTP://User@EXAMPLE.test:80/a b/%41?q=ü#f')` renders as `http://User@example.test/a%20b/%41?q=%C3%BC#f`.

### Read the current request in a service

`RequestContext` and `ResponseContext` keep the request and the response of the current scope in `SuperKernel\Context\Context`, under the keys `Psr\Http\Message\ServerRequestInterface` and `Psr\Http\Message\ResponseInterface`. Each has `set()`, `get()`, `has()`, and `clear()`; `clear()` removes only its own key. The scope is the current coroutine when code runs in one, and otherwise the current thread or process. `super-kernel/http-server` stores both messages before it handles a request and clears them afterwards.

The container provides `ServerRequestInterface` and `ResponseInterface` as services. Every call on such a service goes to the message stored in the current scope, so a shared service that keeps the injected object never sees another request's data. The services offer only the PSR-7 methods. A `with*()` call returns a new message and does not change the stored one; to change the current request for the code that follows, store the new message with `RequestContext::set()`.

A service that greets the caller of the current request, saved as `src/Http/Greeting.php`:

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

A script that stores two requests in turn, as a server does, saved as `bin/scope.php`:

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

Build the project and run the script:

```bash
skernel build
php bin/scope.php
```

It prints:

```text
Hello, Ada!
Hello, Linus!
The current HTTP request context has not been initialized.
```

## Errors

Invalid input raises `InvalidArgumentException`; a failed stream, upload, or context operation raises `RuntimeException`.

| Situation | Message |
| --- | --- |
| A method that is not an HTTP token | `Invalid HTTP method <method>.` |
| A header name that is not an HTTP token | `Invalid HTTP header name <name>.` |
| A header value with a line break or another prohibited control character | `HTTP header <name> contains a prohibited control character.` |
| A protocol version that is not dot-separated numbers, such as `1.1` | `Invalid HTTP protocol version <version>.` |
| A status code outside `100` to `599` | `Invalid HTTP status code <code>.` |
| A request target that is empty or contains whitespace | `Invalid HTTP request target <target>.` |
| A URI that cannot be parsed, including a port above `65535` | `Unable to parse URI <uri>.` |
| A file that cannot be opened | `Unable to open HTTP message stream file <filename>.` |
| A `ResourceStream` without a resource | `A native stream resource is required.` |
| A write to a read-only stream | `The read-only stream is not writable.` |
| An iterable chunk that is not a string | `Iterable stream chunks must be strings; got <type>.` |
| A reader that returns an empty string | `Callback stream sources must not return an empty chunk before end of source.` |
| An exception from an iterable or a reader | `Unable to read from the iterable stream source.` or `Unable to read from the callback stream source.`, with the exception as the previous one |
| An unknown upload error code | `Invalid upload error code <code>.` |
| A successful upload without a source | `A successful upload requires a stream or temporary file path.` |
| Using an upload after it was moved | `The uploaded file has already been moved.` |
| Using a failed upload | `The uploaded file is unavailable because upload error <code> occurred.` |
| An uploaded-file tree with another value | `<path> must contain only uploaded files or nested arrays.` |
| No request or response stored in the current scope | `The current HTTP request context has not been initialized.` or `The current HTTP response context has not been initialized.` |

## Relation to other packages

- [`super-kernel/http-server`](http-server.md) creates a `ServerRequest` from each native request, stores the current messages, and sends the response.
- [`super-kernel/context`](context.md) provides the coroutine or process scope in which the current messages are kept.
- [`super-kernel/di`](di.md) injects `ServerRequestInterface` and `ResponseInterface`; the services are found in the metadata that `skernel build` generates.
- Any library that accepts `psr/http-message` 2.0 objects accepts this package's messages, streams, URIs, and uploaded files.
