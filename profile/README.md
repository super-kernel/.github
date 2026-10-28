<p align="right">
  <strong>English</strong> | <a href="README.zh-CN.md">中文文档</a>
</p>

<div align="center">

# SuperKernel

[![PHP](https://img.shields.io/badge/php-%3E%3D8.4.27%20%3C8.5-blue)](https://www.php.net/)
[![Swoole](https://img.shields.io/badge/swoole-%3E%3D6.2.3%20%3C6.3-green)](https://www.swoole.com/)
[![Platform](https://img.shields.io/badge/platform-linux-lightgrey.svg?logo=linux)](https://www.kernel.org)
[![License](https://img.shields.io/badge/license-MIT-orange)](https://github.com/super-kernel/.github/blob/main/LICENSE)
[![Code Style](https://img.shields.io/badge/code%20style-PSR--12-lightgrey)](https://www.php-fig.org/psr/psr-12/)

</div>

SuperKernel is a component-oriented PHP framework for long-running Swoole applications. Each component is an
independent Composer package under `super-kernel/*`, built on native PHP attributes, a PSR-11 container, and
standard PSR interfaces.

> [!NOTE]
> SuperKernel is under active development and has not been released. The repositories carry no tagged releases,
> and APIs may still change. Each repository's `main` branch is the current development line.

## Highlights

- **Swoole runtime**: server startup, main-process supervision, attribute-driven HTTP serving, background processes,
  and connection pools on `ext-swoole`.
- **Scan before run**: attributes are collected in an isolated scan before the application starts, backed by
  Composer package metadata and cached class graphs.
- **Dependency injection**: a PSR-11 shared-service container with constructor autowiring, typed lazy configuration,
  and strict typed environment access.
- **Aspect-oriented programming**: aspect declarations, method weaving, join points, and interceptor dispatch on top
  of source transformation with hash-validated file proxies.
- **PSR interoperability**: PSR-3 logging, PSR-7 HTTP messages, PSR-11 containers, PSR-14 events, and PSR-15 request
  handlers and middleware.
- **Console**: a Symfony Console application runtime with lazy command registration.

## Requirements

- PHP `>=8.4.27 <8.5`
- `ext-swoole >=6.2.3 <6.3` for the Swoole-based components: `http-server`, `pool`, `process`, `server`, and `swoole`
- `ext-redis >=6.3.0 <6.4` for `redis`
- Linux; some components also require `ext-pcntl`, `ext-posix`, or `ext-sockets`. Each repository's `composer.json`
  lists its exact requirements.

## Components

| Area | Component | Description |
| --- | --- | --- |
| Application | [skeleton](https://github.com/super-kernel/skeleton) | Composer project template for new applications. |
| Application | [framework](https://github.com/super-kernel/framework) | Symfony Console application runtime and PSR-14 dispatcher adaptation. |
| Application | [context](https://github.com/super-kernel/context) | Application container and transparent execution-scope context. |
| Application | [command](https://github.com/super-kernel/command) | Lazy Symfony Console command registration. |
| Application | [event](https://github.com/super-kernel/event) | Application startup event declaration. |
| Application | [event-dispatcher](https://github.com/super-kernel/event-dispatcher) | PSR-14 event dispatching with native Attribute listener discovery. |
| Application | [exception-dispatcher](https://github.com/super-kernel/exception-dispatcher) | Attribute-driven exception handler discovery and dispatch. |
| Container | [di](https://github.com/super-kernel/di) | PSR-11 shared-service container with constructor autowiring. |
| Container | [config](https://github.com/super-kernel/config) | Typed lazy configuration source registry. |
| Container | [environment](https://github.com/super-kernel/environment) | Ordered external property sources and strict typed environment access. |
| Container | [contract](https://github.com/super-kernel/contract) | Shared interface-only application, infrastructure, reflection, and event contracts. |
| Container | [support](https://github.com/super-kernel/support) | Dependency-free namespaced helpers for caller-owned short-lived instances. |
| Attributes | [attribute](https://github.com/super-kernel/attribute) | Native PHP Attribute contracts for configuration, DI, events, and third-party integrations. |
| Attributes | [annotator](https://github.com/super-kernel/annotator) | Isolated package-aware PHP attribute collection. |
| Attributes | [scan-isolation](https://github.com/super-kernel/scan-isolation) | Anonymous scan callback isolation. |
| Attributes | [tokenizer](https://github.com/super-kernel/tokenizer) | PHP class-like declaration tokenizer. |
| Attributes | [reflector](https://github.com/super-kernel/reflector) | Process-local cached native reflection service. |
| Attributes | [composer](https://github.com/super-kernel/composer) | Composer package metadata and cached class graphs. |
| Attributes | [autoloader](https://github.com/super-kernel/autoloader) | Factory-bootstrapped classmap autoloading and interception. |
| Attributes | [composer-source-reference-plugin](https://github.com/super-kernel/composer-source-reference-plugin) | Deterministic source references for Composer path packages. |
| AOP | [aop](https://github.com/super-kernel/aop) | Aspect declarations, method weaving, join points, and interceptor dispatch through Mimic. |
| AOP | [mimic](https://github.com/super-kernel/mimic) | Source preparation with pluggable transformations, source-hash-validated file proxies, and stream fallback. |
| Swoole | [server](https://github.com/super-kernel/server) | Swoole server startup services and console command. |
| Swoole | [swoole](https://github.com/super-kernel/swoole) | Swoole console commands and main process supervision. |
| Swoole | [http-server](https://github.com/super-kernel/http-server) | Attribute-driven HTTP routing, middleware, and Swoole integration. |
| Swoole | [http-message](https://github.com/super-kernel/http-message) | PSR-7 messages and scoped HTTP message access. |
| Swoole | [process](https://github.com/super-kernel/process) | Background process supervision and direct IPC. |
| Swoole | [pool](https://github.com/super-kernel/pool) | Connection pools with coroutine, process, and thread coordination. |
| Integrations | [logger](https://github.com/super-kernel/logger) | Monolog-backed PSR-3 logger factories and named stdout logging. |
| Integrations | [redis](https://github.com/super-kernel/redis) | Redis integration. |
| Integrations | [filesystem](https://github.com/super-kernel/filesystem) | Lifecycle-safe readable filesystem resource abstractions. |

## Repositories

Each component is published here as an individual repository. Repositories without a current component are archived.

## License

Every component is released under the [MIT License](https://github.com/super-kernel/.github/blob/main/LICENSE).
AOP and Mimic also include a `NOTICE` file with their source attribution.
