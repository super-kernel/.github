<p align="right">
  <a href="README.md">English</a> | <strong>中文文档</strong>
</p>

<div align="center">

# SuperKernel

[![PHP](https://img.shields.io/badge/php-%3E%3D8.4.27%20%3C8.5-blue)](https://www.php.net/)
[![Swoole](https://img.shields.io/badge/swoole-%3E%3D6.2.3%20%3C6.3-green)](https://www.swoole.com/)
[![Platform](https://img.shields.io/badge/platform-linux-lightgrey.svg?logo=linux)](https://www.kernel.org)
[![License](https://img.shields.io/badge/license-MIT-orange)](https://github.com/super-kernel/.github/blob/main/LICENSE)
[![Code Style](https://img.shields.io/badge/code%20style-PSR--12-lightgrey)](https://www.php-fig.org/psr/psr-12/)

</div>

SuperKernel 是一个面向组件的 PHP 框架，用于常驻运行的 Swoole 应用。每个组件都是 `super-kernel/*` 下独立的
Composer 包，基于原生 PHP 注解（Attribute）、PSR-11 容器和标准 PSR 接口构建。

> [!NOTE]
> SuperKernel 仍在积极开发中，尚未正式发布。各仓库没有打任何版本标签，API 仍可能变化。
> 每个仓库的 `main` 分支就是当前的开发线。

## 特性

- **Swoole 运行时**：基于 `ext-swoole` 的服务启动、主进程监管、注解驱动的 HTTP 服务、后台进程和连接池。
- **先扫描，后运行**：应用启动前，在隔离的扫描阶段收集注解，并以 Composer 包元数据和缓存的类图为基础。
- **依赖注入**：支持构造函数自动装配的 PSR-11 共享服务容器、类型化的延迟配置，以及严格类型的环境变量访问。
- **面向切面编程**：在源码转换和带哈希校验的文件代理之上，提供切面声明、方法织入、连接点和拦截器调度。
- **PSR 互操作**：PSR-3 日志、PSR-7 HTTP 消息、PSR-11 容器、PSR-14 事件，以及 PSR-15 请求处理器和中间件。
- **命令行**：基于 Symfony Console 的应用运行时，命令延迟注册。

## 环境要求

- PHP `>=8.4.27 <8.5`
- 基于 Swoole 的组件需要 `ext-swoole >=6.2.3 <6.3`：`http-server`、`pool`、`process`、`server` 和 `swoole`
- `redis` 需要 `ext-redis >=6.3.0 <6.4`
- Linux；部分组件还需要 `ext-pcntl`、`ext-posix` 或 `ext-sockets`。确切要求以各仓库的 `composer.json` 为准。

## 组件

| 分类 | 组件 | 说明 |
| --- | --- | --- |
| 应用 | [skeleton](https://github.com/super-kernel/skeleton) | 用于创建新应用的 Composer 项目模板。 |
| 应用 | [framework](https://github.com/super-kernel/framework) | Symfony Console 应用运行时，以及 PSR-14 调度器适配。 |
| 应用 | [context](https://github.com/super-kernel/context) | 应用容器与透明的执行作用域上下文。 |
| 应用 | [command](https://github.com/super-kernel/command) | Symfony Console 命令的延迟注册。 |
| 应用 | [event](https://github.com/super-kernel/event) | 应用启动事件声明。 |
| 应用 | [event-dispatcher](https://github.com/super-kernel/event-dispatcher) | PSR-14 事件调度，通过原生注解发现监听器。 |
| 应用 | [exception-dispatcher](https://github.com/super-kernel/exception-dispatcher) | 注解驱动的异常处理器发现与调度。 |
| 容器 | [di](https://github.com/super-kernel/di) | 支持构造函数自动装配的 PSR-11 共享服务容器。 |
| 容器 | [config](https://github.com/super-kernel/config) | 类型化的延迟配置源注册表。 |
| 容器 | [environment](https://github.com/super-kernel/environment) | 有序的外部属性源，以及严格类型的环境变量访问。 |
| 容器 | [contract](https://github.com/super-kernel/contract) | 共享的纯接口契约：应用、基础设施、反射和事件。 |
| 容器 | [support](https://github.com/super-kernel/support) | 无依赖的命名空间辅助函数，用于调用方持有的短生命周期实例。 |
| 注解 | [attribute](https://github.com/super-kernel/attribute) | 原生 PHP 注解契约，覆盖配置、DI、事件和第三方集成。 |
| 注解 | [annotator](https://github.com/super-kernel/annotator) | 隔离、感知包边界的 PHP 注解收集。 |
| 注解 | [scan-isolation](https://github.com/super-kernel/scan-isolation) | 匿名扫描回调的隔离执行。 |
| 注解 | [tokenizer](https://github.com/super-kernel/tokenizer) | PHP 类级声明的词法分析器。 |
| 注解 | [reflector](https://github.com/super-kernel/reflector) | 进程内缓存的原生反射服务。 |
| 注解 | [composer](https://github.com/super-kernel/composer) | Composer 包元数据与缓存的类图。 |
| 注解 | [autoloader](https://github.com/super-kernel/autoloader) | 由工厂引导的类映射自动加载与拦截。 |
| 注解 | [composer-source-reference-plugin](https://github.com/super-kernel/composer-source-reference-plugin) | 为 Composer 路径包生成确定性的源码引用。 |
| AOP | [aop](https://github.com/super-kernel/aop) | 切面声明、方法织入、连接点，以及通过 Mimic 的拦截器调度。 |
| AOP | [mimic](https://github.com/super-kernel/mimic) | 可插拔转换的源码准备、带源码哈希校验的文件代理和流回退。 |
| Swoole | [server](https://github.com/super-kernel/server) | Swoole 服务启动服务与控制台命令。 |
| Swoole | [swoole](https://github.com/super-kernel/swoole) | Swoole 控制台命令与主进程监管。 |
| Swoole | [http-server](https://github.com/super-kernel/http-server) | 注解驱动的 HTTP 路由、中间件，以及 Swoole 集成。 |
| Swoole | [http-message](https://github.com/super-kernel/http-message) | PSR-7 消息与作用域内的 HTTP 消息访问。 |
| Swoole | [process](https://github.com/super-kernel/process) | 后台进程监管与直接 IPC。 |
| Swoole | [pool](https://github.com/super-kernel/pool) | 支持协程、进程和线程协调的连接池。 |
| 集成 | [logger](https://github.com/super-kernel/logger) | 基于 Monolog 的 PSR-3 日志工厂，以及具名的 stdout 日志。 |
| 集成 | [redis](https://github.com/super-kernel/redis) | Redis 集成。 |
| 集成 | [filesystem](https://github.com/super-kernel/filesystem) | 生命周期安全的只读文件系统资源抽象。 |

## 仓库说明

每个组件都作为独立仓库发布在这里。没有对应当前组件的仓库已归档。

## 许可证

所有组件都以 [MIT 许可证](https://github.com/super-kernel/.github/blob/main/LICENSE) 发布。AOP 和 Mimic 另附
`NOTICE` 文件，说明其源码来源。
