<p align="right">
  <strong>English</strong> | <a href="filesystem.zh-CN.md">中文文档</a>
</p>

# SuperKernel Filesystem

Package: [`super-kernel/filesystem`](https://github.com/super-kernel/filesystem) · [Packagist](https://packagist.org/packages/super-kernel/filesystem) · [Documentation index](../README.md)

## Purpose

`super-kernel/filesystem` describes something to read as a resource and leaves the handle to the resource. Code calls `read()` and iterates over the values: chunks of bytes for files, streams, and memory, and entries for directories. The resource opens its PHP handle when the iteration starts and closes it when the iteration ends, so the code never holds, shares, or forgets to close a handle.

The package is synchronous and read-only. Writing, asynchronous or Swoole-specific reading, path policy such as containment, canonical paths, or glob patterns, and caching belong elsewhere. Its classes are plain objects: the package registers no container service and needs no build.

## Capabilities

### Read a file

`LocalResourceFactory::file()` describes a regular file; the second argument, `chunkSize`, is the largest chunk in bytes, 8192 by default. `read()` yields the file's bytes as strings of up to that size. A chunk carries no encoding and no line boundary: the bytes of one line can span two chunks, and a chunk can end in the middle of a multibyte character. Read the whole resource when the content must be complete.

The path is checked when the resource is created, and the file is opened only when iteration starts; a path that is not a regular file fails then.

A script that reads a file, saved as `bin/read-file.php`:

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

Run the script:

```bash
php bin/read-file.php
```

It prints:

```text
[INV-1001]
[\nINV-100]
[2\n]
```

### Read a directory

`LocalResourceFactory::directory()` describes a directory. `read()` yields a `SuperKernel\Filesystem\Model\DirectoryEntry` for each immediate entry, without `.` and `..` and without descending into subdirectories. Each entry has a `name()`, a `path()` that joins the directory path and the name, and a `type()`, one of the `SuperKernel\Filesystem\Constants\DirectoryEntryType` cases `Directory`, `File`, `Link`, or `Other`; a symbolic link is reported as `Link`, whatever it points to. The entries come in the filesystem's own order, which is unspecified, so sort them when the order matters.

A script that lists a directory, saved as `bin/read-directory.php`:

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

Run the script:

```bash
php bin/read-directory.php
```

It prints:

```text
2026: directory
index.txt: file
```

### Read a stream

`LocalResourceFactory::stream()` describes any URI that PHP's `fopen()` opens for reading, such as `php://stdin`, `data:` URIs, or a stream wrapper, with the same chunk rules as a file.

A script that reads a `data:` URI, saved as `bin/read-stream.php`:

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

Run the script:

```bash
php bin/read-stream.php
```

It prints:

```text
[INV-1001,]
[INV-1002]
```

### Read a stream that your code opens

`LocalResourceFactory::streamFromFactory()`, or `SuperKernel\Filesystem\Resource\StreamResource::fromFactory()`, takes a closure that opens the stream. The resource calls it at the start of every read cycle and closes the stream afterwards, so the closure must return a newly opened PHP stream on every call. `description` names the stream in error messages, `custom stream` by default. An exception thrown by the closure becomes the previous exception of a `ResourceReadException`.

A script that reads a generated report twice, saved as `bin/read-report.php`:

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

Run the script:

```bash
php bin/read-report.php
```

It prints:

```text
Read 1: Monthly invoice report
Read 2: Monthly invoice report
```

### Read data held in memory

`LocalResourceFactory::binaryStream()`, or `new SuperKernel\Filesystem\Resource\BinaryStreamResource($contents, $chunkSize)`, yields a string in chunks through the same contract, so code that consumes a resource can be given data that is already in memory. PHP strings are binary-safe, and so are the chunks.

A script that reads binary data, saved as `bin/read-memory.php`:

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

Run the script:

```bash
php bin/read-memory.php
```

It prints:

```text
89504e47
0d0a1a0a
```

### Know the lifetime of a read

Each call to `read()` starts a new, independent read cycle, so a resource can be read again from the start. The handle is opened when iteration starts, not when the resource or the iterator is created, and it is closed when the iteration is exhausted, when reading fails, or when a partly consumed iterator is released, for example after a `break` out of a `foreach`. The resources never expose their handles.

### Depend on the contracts

`SuperKernel\Filesystem\Contract\ReadableResourceInterface` has one method, `read()`, and is generic over what it yields: a directory yields `DirectoryEntry` values and the byte resources yield strings. What the resources share is the reading lifecycle, not one payload type. `SuperKernel\Filesystem\Contract\ResourceFactoryInterface` declares the five factory methods, and `SuperKernel\Filesystem\Factory\LocalResourceFactory` implements it. Type code that reads against these interfaces, and create the factory where your code is composed, so that a test can pass another resource, such as an in-memory one.

## Errors

Every exception implements `SuperKernel\Filesystem\Exception\FilesystemExceptionInterface`. Configuration errors are `InvalidResourceConfigurationException`s raised when the resource is created; read errors are `ResourceReadException`s raised during `read()`. In the messages, `<label>` is `File path`, `Directory path`, `Stream URI`, or `Stream description`, and `<description>` is `stream "<uri>"` for a URI or the description given to a factory stream.

| Situation | Message |
| --- | --- |
| An empty path, URI, or description | `<label> must not be empty.` |
| A path, URI, or description with a null byte | `<label> must not contain null bytes.` |
| A chunk size below one | `Chunk size must be greater than zero.` |
| A file path that is not a regular file | `File "<path>" is not a regular file.` |
| A directory that cannot be opened | `Unable to open directory "<path>".` |
| A stream that cannot be opened, or a factory that throws or returns no stream | `Unable to open <description>.` |
| A stream that cannot be read | `Unable to read <description>.` |
| A stream that returns no data before its end | `Reading <description> made no progress before end of stream.` |

## Relation to other packages

- The package depends on no other package; use it directly in your code.
- It registers no service in [`super-kernel/di`](di.md); create `LocalResourceFactory` where your code is composed.

## What not to do

**Do not return the same stream from a factory twice.** The resource closes the stream after each read cycle, so a closure that returns a stream it keeps hands a closed stream to the second `read()`, which fails with `Unable to open <description>.` Open a new stream inside the closure every time it is called.

