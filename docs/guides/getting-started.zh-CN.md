<p align="right">
  <a href="getting-started.md">English</a> | <strong>中文文档</strong>
</p>

# 快速入门

本指南带你从安装 SKERNEL 走到一个运行中的 SuperKernel 应用，然后修改、构建并打包它。[文档索引](../README.zh-CN.md)

## 开始之前

- 一台 Linux 或 macOS 机器。
- 网络访问。没有预构建版本发布的托管 PHP 会在你的机器上构建，这需要构建工具：C 编译器、`perl`、`re2c`、`autoconf`、`automake`、`cmake`、`libtool` 和 `gettext`。SKERNEL 从不安装系统软件包；构建失败时，输出会指出缺少什么。
- Composer 2。找不到 Composer 时，SKERNEL 会下载自己的 Composer，但修改依赖需要你机器上的 Composer。

SKERNEL 用它自己的静态 PHP（托管 PHP）来构建、运行和打包项目，而不是使用系统上的 PHP。**在一台机器上，第一个需要托管 PHP 的命令会从 [super-kernel-tools/runtimes](https://github.com/super-kernel-tools/runtimes) 下载预构建的版本，本指南所创建项目的托管 PHP 就发布在那里。** 如果项目的 PHP 版本或扩展没有在那里发布，它的托管 PHP 就改为从源码构建，这需要一段时间，也需要上面的构建工具；`skernel.json` 中设置了 `"prebuilt": false` 的项目同样如此。SKERNEL 把托管 PHP 保存在 `~/.skernel` 或 `SKERNEL_HOME` 指定的目录中，之后凡是需要相同 PHP 版本和扩展的命令都会复用它。

## 安装 SKERNEL

从[最新发布](https://github.com/super-kernel-tools/skernel/releases/latest)下载适合你平台的二进制文件，平台为 `linux-x86_64`、`linux-aarch64`、`macos-aarch64` 和 `macos-x86_64` 之一。它不需要 PHP。以 x86-64 架构的 Linux 为例：

```bash
curl -fsSL -o skernel https://github.com/super-kernel-tools/skernel/releases/latest/download/skernel-linux-x86_64
chmod +x skernel
./skernel --version
```

`--version` 会打印 `SKERNEL` 及其版本。本指南其余部分把该二进制文件称为 `skernel`，因此请把它放到 `PATH` 中的某个目录里。之后可以用 `skernel self-update` 把它替换为最新发布。

## 创建项目

```bash
skernel new my-app
```

`new` 使用 Composer 的 `create-project` 命令，从与你的 SKERNEL 同一版本发布的 `super-kernel/skeleton` 模板在 `my-app` 中创建项目，并安装它的依赖。如果你的机器上没有 Composer 2，它会下载一个，并用托管 PHP 运行它，第一次时会下载托管 PHP。成功时它会打印：

```text
Project created in /path/to/my-app.
```

目标目录必须不存在或为空。

## 项目包含什么

| 路径 | 内容 |
| --- | --- |
| `bin/bootstrap.php` | PHP 入口：获取应用容器，并用命令行参数运行应用。 |
| `src/Controller/TestController.php` | 一个带有两个路由 `GET /index` 和 `GET /user` 的 HTTP 控制器。 |
| `src/Service/TestService.php` | `/index` 使用的服务。 |
| `src/Aspect/TestAspect.php` | 一个在 `TestService::getA()` 前后记录日志的切面。 |
| `src/Config/SwooleConfig.php`、`src/Config/SwooleServerConfig.php` | Swoole 设置以及 `127.0.0.1:9600` 上的 HTTP 监听器。 |
| `src/Environment/ApplicationEnvironment.php` | 环境访问：先读取项目目录中的 `.env`，再读取进程环境。 |
| `src/ExceptionHandler/Http/HttpServerExceptionHandler.php` | 把 HTTP 错误（例如未知路由）转换为带 `error` 字段的 JSON 响应。 |
| `src/Process/TestProcess.php` | 一个后台进程示例，默认禁用。 |
| `tests/` | PHPUnit 测试。 |
| `composer.lock` | 锁定的依赖版本。请提交它：`skernel package` 从它安装依赖，构建元数据也与它绑定。 |
| `composer.json` | SuperKernel 各包、`src/` 下的 `Src\` 命名空间，以及脚本 `start`（`skernel run swoole:start`）和 `test`（先 `skernel build`，再运行 PHPUnit）。 |

项目要求 PHP `>=8.4.27 <8.5` 和 `ext-swoole >=6.2.3 <6.3`；SKERNEL 的托管 PHP 包含项目要求的所有扩展。

继续之前，请把 `composer.json` 中的 `name` 改为你自己的包名，例如 `acme/my-app`：应用的控制台名称取自它，`skernel package` 也按它命名输出文件。在你修改之前，名称是 `super-kernel/skeleton`。

## 运行应用

```bash
cd my-app
skernel run swoole:start
```

`run` 先构建项目，然后用托管 PHP 启动它，并把 `swoole:start` 传给应用，由应用启动 Swoole HTTP 服务器。在一台机器上第一次运行时，还会下载托管 PHP。日志会显示监听器：

```text
[NOTICE] Swoole main server started in base mode.
[INFO] Swoole server process started with PID 12345.
[INFO] Swoole listener "http" started on 127.0.0.1:9600 (http).
```

每一行都以时间戳开头。在另一个终端中请求这两个路由：

```bash
curl http://127.0.0.1:9600/index
curl http://127.0.0.1:9600/user
```

第一个请求打印 `1`，这是一个随每次请求递增的计数器，同时服务器会记录切面输出的 `Before Src\Service\TestService::getA().` 和 `After Src\Service\TestService::getA().`。第二个请求打印 `{"id":1,"name":"Example"}`。未知路径（例如 `/missing`）返回状态码 404 和 `{"error":"Not Found"}`。

按 Ctrl+C 停止应用；SKERNEL 会在退出之前先停止它。`composer start` 运行的是同一条 `skernel run swoole:start`。

## 修改类并重新构建

应用启动时不会扫描项目：它加载上一次构建从你的类及其注解中收集到的元数据。每次 `skernel run` 都会先构建，因此重新启动就能获得你的修改。

添加一个控制器。将它保存为 `src/Controller/HelloController.php`：

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

每次构建都会重新生成 Composer 的自动加载器，因此新类只需要下一次构建。再次运行应用：

```bash
skernel run swoole:start
```

现在 `curl http://127.0.0.1:9600/hello/world` 会打印 `Hello, world!`。修改已有的类同样只需要下一次构建。

要在每次修改后重新构建并重启，请在项目的配置文件中开启监视。SKERNEL 从项目目录的 `skernel.json` 读取项目的设置，没有 `skernel.json` 时改读 `skernel.json.dist`；两个文件从不合并，这些设置也没有命令行选项。共享的设置请作为 `skernel.json.dist` 提交，只用于一台机器的 `skernel.json` 不要纳入版本控制。将它保存为 `skernel.json`：

```json
{
    "run": {
        "watch": true
    }
}
```

然后像之前一样运行应用：

```bash
skernel run swoole:start
```

这样 SKERNEL 每秒检查一次名称与 `run.watch-files` 中某个模式匹配的文件是否变化，默认模式为 `*.php`、`.env`、`.env.*`、`composer.json` 和 `composer.lock`，并在每次变化（包括添加类）之后构建并重启应用。它从不监视依赖目录、`target/` 以及名称以点开头的目录。`run.watch` 默认为 `false`。

## 先构建，再用 PHP 启动

要用你自己的 PHP 启动应用，请先构建它：

```bash
skernel build
php bin/bootstrap.php swoole:start
```

`skernel build` 把构建产物写入 `vendor/.cache`，成功时打印：

```text
Build artifacts written to /path/to/my-app/vendor/.cache.
```

在这一行之前，它会列出无法收集其注解的类。在新项目中，这个列表会列出依赖未安装的可选 Symfony 集成中的类，例如 `Symfony\Component\Console\ConsoleBundle`；你自己的类不应该出现在其中。

你的 PHP 必须满足项目的要求，包括 `ext-swoole`。在 `composer install` 或 `composer update` 之后、修改自动加载规则之后，以及添加或修改带注解的类之后，请重新运行 `skernel build`。没有构建、或者构建所对应的 `composer.lock` 已经不同时，应用启动会停止，并给出以 ``Run `skernel build` to generate the project metadata.`` 结尾的消息。`composer test` 会在运行测试之前构建项目。

[构建与元数据](building-and-metadata.zh-CN.md)说明了构建生成的内容以及它可能产生的每一条消息。

## 打包应用

```bash
skernel package
```

`package` 把项目变成一个独立的可执行文件，目标机器上不需要 PHP。它在一个临时副本上工作，因此项目本身不会被改动：它按 `composer.lock` 为副本安装依赖（不含开发依赖），构建该副本，再用托管 PHP 的 micro 运行时把它归档。成功时它会打印：

```text
Package written to /path/to/my-app/target/my-app.
```

文件以根包名称的最后一段命名，因此 `acme/my-app` 对应 `target/my-app`，在你重命名包之前则是 `target/skeleton`。它可以直接运行应用：

```bash
./target/my-app swoole:start
```

打包结果不包含项目目录中的 `.env` 和 `.env.*`，也不包含 `skernel.json` 和 `skernel.json.dist`；打包后的应用从进程环境读取它的环境。`package` 的设置决定写出什么：`package.phar` 为 `true` 时写出 PHAR 而不是可执行文件，`package.dev` 为 `true` 时包含开发依赖，`package.output` 指定另一个相对于项目目录的路径，`package.runtime` 用你自己的 micro 运行时代替托管的运行时。例如，下面的配置让 `skernel package` 改为把 PHAR 写到 `dist/my-app.phar`。将它保存为 `skernel.json`：

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

`skernel clean` 会删除 `target/` 和 `vendor/.cache` 中的构建产物；通过 `package.output` 写到其他位置的包会保留。

## 下一步

- [构建与元数据](building-and-metadata.zh-CN.md)：构建写出什么，以及缺失或过期的构建如何报告。
- [SuperKernel DI](../components/di.zh-CN.md)：声明服务并注入它们。
- [SuperKernel Event](../components/event.zh-CN.md)：在应用启动或构建时执行操作。
- [SKERNEL README](https://github.com/super-kernel-tools/skernel/blob/main/README.zh-CN.md)：每个命令，以及 `skernel.json` 配置文件的每个设置及其默认值。
