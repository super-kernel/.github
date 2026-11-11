<p align="right">
  <strong>English</strong> | <a href="README.zh-CN.md">中文文档</a>
</p>

# SuperKernel documentation

SuperKernel is a component-oriented PHP framework for long-running Swoole applications. Each component is an independent Composer package under `super-kernel/*`, built on native PHP Attributes, a PSR-11 container, and standard PSR interfaces. SKERNEL, its command-line tool, creates, builds, runs, and packages SuperKernel projects.

## How to read this documentation

- New to SuperKernel: follow [Getting started](guides/getting-started.md) from installing SKERNEL to a running application.
- Before deploying or packaging: read [Building and metadata](guides/building-and-metadata.md), which explains what `skernel build` generates and when it must run again.
- Working with one package: open its page below. Every page has the same parts: the package's purpose, its capabilities with complete examples, its relation to other packages, and, where misuse is costly, what not to do.
- Each repository's README is a short entry point with installation, a quick start, and the package's public surface; it links back to the package's page here.

Examples use the layout of a project created from `super-kernel/skeleton`: application classes under `src/` in the `Src\` namespace, and scripts under `bin/`.

## Guides

- [Getting started](guides/getting-started.md): install SKERNEL, create a project, run it, change it, build it, and package it.
- [Building and metadata](guides/building-and-metadata.md): what `skernel build` generates, why an application does not scan at start, and how a missing or outdated build is reported.

## Components

### Application

- [framework](components/framework.md): the Symfony Console application runtime and its PSR-14 dispatcher adaptation.
- [context](components/context.md): the process's application container and transparent execution-scope context.
- [command](components/command.md): lazy registration of Symfony Console commands.
- [event](components/event.md): the application lifecycle events published at start and at build time.
- [event-dispatcher](components/event-dispatcher.md): PSR-14 event dispatching with listeners discovered from Attributes.
- [exception-dispatcher](components/exception-dispatcher.md): exception handlers discovered from Attributes and dispatch to them.

### Container

- [di](components/di.md): the PSR-11 container that creates, injects, and shares the services classes declare with Attributes.
- [config](components/config.md): a registry of typed, lazily loaded configuration sources.
- [environment](components/environment.md): ordered external property sources and strictly typed environment access.
- [contract](components/contract.md): the shared interfaces of applications, infrastructure, reflection, and events.
- [support](components/support.md): dependency-free helpers for short-lived instances that the caller owns.

### Attributes

- [attribute](components/attribute.md): the native PHP Attributes for configuration, dependency injection, events, and integrations.
- [annotator](components/annotator.md): restoration and lookup of the Attribute records that `skernel build` generates.
- [reflector](components/reflector.md): a process-local cached reflection service.
- [metadata](components/metadata.md): read-only restoration of the package metadata that `skernel build` generates.
- [autoloader](components/autoloader.md): class-map autoloading and interception, created by a factory.
- [composer-source-reference-plugin](components/composer-source-reference-plugin.md): deterministic source references for Composer path packages.

### AOP

- [aop](components/aop.md): aspect declarations, method weaving, join points, and interceptor dispatch.
- [mimic](components/mimic.md): source transformation with hash-validated file proxies and a stream fallback.

### Swoole

- [server](components/server.md): creation and startup of the configured Swoole server, events for its creation and callbacks, and user processes.
- [swoole](components/swoole.md): the `swoole:start` command and main process supervision.
- [http-server](components/http-server.md): HTTP routing and middleware declared with Attributes, served by Swoole.
- [http-message](components/http-message.md): PSR-7 messages and scoped access to the current HTTP message.
- [process](components/process.md): supervised background processes and direct inter-process communication.
- [pool](components/pool.md): connection pools coordinated across coroutines, processes, and threads.

### Integrations

- [logger](components/logger.md): PSR-3 logger factories backed by Monolog, and named stdout logging.
- [redis](components/redis.md): Redis integration.
- [filesystem](components/filesystem.md): readable filesystem resources with a safe lifecycle.

## Project template and tool

- [skeleton](https://github.com/super-kernel/skeleton): the project template that `skernel new` creates an application from; [Getting started](guides/getting-started.md) walks through it.
- [SKERNEL](https://github.com/super-kernel/skernel): the command-line tool that creates, runs, builds, and packages a project, published as a standalone binary through [GitHub Releases](https://github.com/super-kernel/skernel/releases).

## License

SuperKernel is released under the [MIT License](https://github.com/super-kernel/.github/blob/main/LICENSE).
