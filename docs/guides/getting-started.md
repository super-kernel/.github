<p align="right">
  <strong>English</strong> | <a href="getting-started.zh-CN.md">中文文档</a>
</p>

# Getting started

This guide takes you from installing SKERNEL to a running SuperKernel application, then through changing it, building it, and packaging it. [Documentation index](../README.md)

## Before you start

- A Linux or macOS machine.
- Network access, and the build tools that the first managed PHP needs: a C compiler, `perl`, `re2c`, `autoconf`, `automake`, `cmake`, `libtool`, and `gettext`. SKERNEL never installs system packages; when a build fails, the output names what is missing.
- Composer 2. SKERNEL downloads its own when it finds none, but changing dependencies needs Composer on your machine.

SKERNEL builds, runs, and packages a project with a static PHP of its own, the managed PHP, not with the PHP on your system. **The first command that needs a managed PHP on a machine builds it from source, and that takes a while.** SKERNEL keeps it in `~/.skernel`, or in the directory that `SKERNEL_HOME` names, and reuses it for every later command that needs the same PHP version and extensions.

## Install SKERNEL

Download the binary for your platform, one of `linux-x86_64`, `linux-aarch64`, `macos-aarch64`, and `macos-x86_64`, from the [latest release](https://github.com/super-kernel/skernel/releases/latest). It needs no PHP. For Linux on x86-64:

```bash
curl -fsSL -o skernel https://github.com/super-kernel/skernel/releases/latest/download/skernel-linux-x86_64
chmod +x skernel
./skernel --version
```

`--version` prints `SKERNEL` followed by its version. The rest of this guide calls the binary `skernel`, so put it in a directory on your `PATH`. `skernel self-update` later replaces it with the latest release.

## Create a project

```bash
skernel new my-app
```

`new` creates the project in `my-app` from the `super-kernel/skeleton` template released with the same version as your SKERNEL, using Composer's `create-project` command, and installs its dependencies. Without a Composer 2 on your machine, it downloads one and runs it with a managed PHP, which it builds the first time. On success it prints:

```text
Project created in /path/to/my-app.
```

The directory must not exist or must be empty.

## What the project contains

| Path | Content |
| --- | --- |
| `bin/bootstrap.php` | The PHP entry: it obtains the application container and runs the application with the command-line arguments. |
| `src/Controller/TestController.php` | An HTTP controller with two routes, `GET /index` and `GET /user`. |
| `src/Service/TestService.php` | The service that `/index` uses. |
| `src/Aspect/TestAspect.php` | An aspect that logs before and after `TestService::getA()`. |
| `src/Config/SwooleConfig.php`, `src/Config/SwooleServerConfig.php` | The Swoole settings and the HTTP listener on `127.0.0.1:9600`. |
| `src/Environment/ApplicationEnvironment.php` | Environment access that reads `.env` in the project directory, then the process environment. |
| `src/ExceptionHandler/Http/HttpServerExceptionHandler.php` | Turns HTTP errors, such as an unknown route, into a JSON `error` response. |
| `src/Process/TestProcess.php` | A background process example, disabled by default. |
| `tests/` | PHPUnit tests. |
| `composer.lock` | The locked dependency versions. Commit it: `skernel package` installs from it, and the build metadata is bound to it. |
| `composer.json` | The SuperKernel packages, the `Src\` namespace under `src/`, and the scripts `start` (`skernel run swoole:start`) and `test` (`skernel build`, then PHPUnit). |

The project requires PHP `>=8.4.27 <8.5` and `ext-swoole >=6.2.3 <6.3`; SKERNEL's managed PHP includes every extension the project requires.

Change `name` in `composer.json` to your own package name, such as `acme/my-app`, before you go further: the application takes its console name from it, and `skernel package` names its output after it. Until you do, the name is `super-kernel/skeleton`.

## Run the application

```bash
cd my-app
skernel run swoole:start
```

`run` builds the project, then starts it with the managed PHP and passes `swoole:start` to the application, which starts the Swoole HTTP server. The first run on a machine also builds the managed PHP. The log shows the listener:

```text
[NOTICE] Swoole main server started in base mode.
[INFO] Swoole server process started with PID 12345.
[INFO] Swoole listener "http" started on 127.0.0.1:9600 (http).
```

Each line starts with a timestamp. In another terminal, request the two routes:

```bash
curl http://127.0.0.1:9600/index
curl http://127.0.0.1:9600/user
```

The first prints `1`, a counter that grows with each request, and the server logs `Before Src\Service\TestService::getA().` and `After Src\Service\TestService::getA().` from the aspect. The second prints `{"id":1,"name":"Example"}`. An unknown path, such as `/missing`, returns status 404 with `{"error":"Not Found"}`.

Press Ctrl+C to stop the application; SKERNEL stops it before it exits. `composer start` runs the same `skernel run swoole:start`.

## Change a class and rebuild

The application does not scan the project when it starts: it loads the metadata that the last build collected from your classes and their Attributes. Every `skernel run` builds first, so restarting picks up your changes.

Add a controller. Save it as `src/Controller/HelloController.php`:

```php
<?php

declare(strict_types=1);

namespace Src\Controller;

use SuperKernel\HttpServer\Attribute\Controller;
use SuperKernel\HttpServer\Attribute\RequestMapping;

#[Controller('/hello')]
final readonly class HelloController
{
    #[RequestMapping('/world')]
    public function world(): string
    {
        return 'Hello, world!';
    }
}
```

Every build also regenerates Composer's autoloader, so a new class needs only the next build. Run the application again:

```bash
skernel run swoole:start
```

`curl http://127.0.0.1:9600/hello/world` now prints `Hello, world!`. A change to an existing class also needs only the next build.

To rebuild and restart on every change, turn on watching in the project's configuration file. SKERNEL reads a project's settings from `skernel.json` in the project directory, or from `skernel.json.dist` when there is no `skernel.json`; the two files are never merged, and the settings have no command-line options. Commit shared settings as `skernel.json.dist`, and keep a `skernel.json` for one machine out of version control. Save it as `skernel.json`:

```json
{
    "run": {
        "watch": true
    }
}
```

Then run the application as before:

```bash
skernel run swoole:start
```

SKERNEL then checks once a second for changes to the files whose name matches a pattern of `run.watch-files`, by default `*.php`, `.env`, `.env.*`, `composer.json`, and `composer.lock`, and builds and restarts the application after each change, including the addition of a class. It never watches the vendor directory, `target/`, or directories whose name starts with a dot. `run.watch` is `false` by default.

## Build, then start with PHP

To start the application with your own PHP, build it first:

```bash
skernel build
php bin/bootstrap.php swoole:start
```

`skernel build` writes the build artifacts to `vendor/.cache` and prints on success:

```text
Build artifacts written to /path/to/my-app/vendor/.cache.
```

Before that line it lists the classes whose Attributes it could not collect. In a new project, the list names classes of optional Symfony integrations whose dependencies are not installed, such as `Symfony\Component\Console\ConsoleBundle`; your own classes should not appear in it.

Your PHP must satisfy the project's requirements, including `ext-swoole`. Run `skernel build` again after `composer install` or `composer update`, after changing autoload rules, and after adding or changing annotated classes. An application started without a build, or with a build made for a different `composer.lock`, stops with a message that ends with ``Run `skernel build` to generate the project metadata.`` `composer test` builds the project before it runs the tests.

[Building and metadata](building-and-metadata.md) explains what the build generates and every message it can produce.

## Package the application

```bash
skernel package
```

`package` turns the project into a standalone executable that needs no PHP on the target machine. It works on a temporary copy, so the project itself is not changed: it installs the copy's dependencies from `composer.lock` without development dependencies, builds the copy, and archives it with the managed PHP's micro runtime. On success it prints:

```text
Package written to /path/to/my-app/target/my-app.
```

The file is named after the last part of the root package's name, so `target/my-app` for `acme/my-app` and `target/skeleton` until you rename the package. It runs the application directly:

```bash
./target/my-app swoole:start
```

The package leaves out `.env` and `.env.*` in the project directory, and `skernel.json` and `skernel.json.dist`; a packaged application reads its environment from the process environment. The `package` settings change what it writes: `package.phar` set to `true` writes a PHAR instead of an executable, `package.dev` set to `true` includes development dependencies, `package.output` names another path, relative to the project directory, and `package.runtime` names a micro runtime of your own instead of the managed one. For example, this configuration makes `skernel package` write a PHAR to `dist/my-app.phar` instead. Save it as `skernel.json`:

```json
{
    "run": {
        "watch": true
    },
    "package": {
        "output": "dist/my-app.phar",
        "phar": true
    }
}
```

`skernel clean` removes `target/` and the build artifacts in `vendor/.cache`; a package written elsewhere through `package.output` stays.

## Next steps

- [Building and metadata](building-and-metadata.md): what the build writes, and how a missing or outdated build is reported.
- [SuperKernel DI](../components/di.md): declaring services and injecting them.
- [SuperKernel Event](../components/event.md): acting when the application starts or is built.
- The [SKERNEL README](https://github.com/super-kernel/skernel#readme): every command, and every setting of the `skernel.json` configuration file with its default.
