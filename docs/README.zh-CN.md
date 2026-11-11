<p align="right">
  <a href="README.md">English</a> | <strong>中文文档</strong>
</p>

# SuperKernel 文档

SuperKernel 是一个面向长期运行的 Swoole 应用、以组件为中心的 PHP 框架。每个组件都是 `super-kernel/*` 下独立的 Composer 包，基于原生 PHP 注解、PSR-11 容器和标准 PSR 接口构建。它的命令行工具 SKERNEL 负责创建、构建、运行和打包 SuperKernel 项目。

## 如何阅读本文档

- 初次接触 SuperKernel：按照[快速入门](guides/getting-started.zh-CN.md)从安装 SKERNEL 一直走到运行中的应用。
- 部署或打包之前：阅读[构建与元数据](guides/building-and-metadata.zh-CN.md)，它说明 `skernel build` 生成什么以及何时必须重新运行。
- 使用某一个包：打开下面它的页面。每个页面都有相同的组成部分：包的用途、带完整示例的各项能力、与其他包的关系，以及在误用代价较高时列出的不要这样做的事情。
- 每个仓库的 README 都是一个简短的入口，包含安装、快速开始和包的公开接口；它会链接回这里的包页面。

示例使用由 `super-kernel/skeleton` 创建的项目的布局：应用类位于 `src/` 下的 `Src\` 命名空间中，脚本位于 `bin/` 下。

## 指南

- [快速入门](guides/getting-started.zh-CN.md)：安装 SKERNEL，创建项目，运行、修改、构建并打包它。
- [构建与元数据](guides/building-and-metadata.zh-CN.md)：`skernel build` 生成什么，为什么应用启动时不扫描，以及缺失或过期的构建如何报告。

## 组件

### 应用

- [framework](components/framework.zh-CN.md)：Symfony Console 应用运行时及其 PSR-14 分发器适配。
- [context](components/context.zh-CN.md)：进程的应用容器和透明的执行作用域上下文。
- [command](components/command.zh-CN.md)：Symfony Console 命令的延迟注册。
- [event](components/event.zh-CN.md)：在启动时和构建时发布的应用生命周期事件。
- [event-dispatcher](components/event-dispatcher.zh-CN.md)：PSR-14 事件分发，监听器通过注解发现。
- [exception-dispatcher](components/exception-dispatcher.zh-CN.md)：通过注解发现异常处理器并向其分发。

### 容器

- [di](components/di.zh-CN.md)：PSR-11 容器，创建、注入并共享类用注解声明的服务。
- [config](components/config.zh-CN.md)：带类型、延迟加载的配置源注册表。
- [environment](components/environment.zh-CN.md)：有序的外部属性源和严格类型化的环境访问。
- [contract](components/contract.zh-CN.md)：应用、基础设施、反射和事件的共享接口。
- [support](components/support.zh-CN.md)：无依赖的辅助函数，用于调用方持有的短生命周期实例。

### 注解

- [attribute](components/attribute.zh-CN.md)：用于配置、依赖注入、事件和集成的原生 PHP 注解。
- [annotator](components/annotator.zh-CN.md)：恢复并查询 `skernel build` 生成的注解记录。
- [reflector](components/reflector.zh-CN.md)：进程内带缓存的反射服务。
- [metadata](components/metadata.zh-CN.md)：只读地恢复 `skernel build` 生成的包元数据。
- [autoloader](components/autoloader.zh-CN.md)：由工厂创建的类映射自动加载与拦截。
- [composer-source-reference-plugin](components/composer-source-reference-plugin.zh-CN.md)：为 Composer 路径包提供确定的源引用。

### AOP

- [aop](components/aop.zh-CN.md)：切面声明、方法织入、连接点和拦截器分发。
- [mimic](components/mimic.zh-CN.md)：源码转换，带哈希校验的文件代理和流回退。

### Swoole

- [server](components/server.zh-CN.md)：所配置的 Swoole 服务器的创建和启动、其创建和回调的事件，以及用户进程。
- [swoole](components/swoole.zh-CN.md)：`swoole:start` 命令和主进程监管。
- [http-server](components/http-server.zh-CN.md)：用注解声明、由 Swoole 提供服务的 HTTP 路由和中间件。
- [http-message](components/http-message.zh-CN.md)：PSR-7 消息以及对当前 HTTP 消息的作用域访问。
- [process](components/process.zh-CN.md)：受监管的后台进程和直接的进程间通信。
- [pool](components/pool.zh-CN.md)：在协程、进程和线程之间协调的连接池。

### 集成

- [logger](components/logger.zh-CN.md)：基于 Monolog 的 PSR-3 日志器工厂，以及具名的标准输出日志。
- [redis](components/redis.zh-CN.md)：Redis 集成。
- [filesystem](components/filesystem.zh-CN.md)：生命周期安全的可读文件系统资源。

## 项目模板与工具

- [skeleton](https://github.com/super-kernel/skeleton)：`skernel new` 用来创建应用的项目模板；[快速入门](guides/getting-started.zh-CN.md)会逐步介绍它。
- [SKERNEL](https://github.com/super-kernel/skernel)：创建、运行、构建并打包项目的命令行工具，以独立二进制文件的形式通过 [GitHub Releases](https://github.com/super-kernel/skernel/releases) 发布。

## 许可证

SuperKernel 基于 [MIT 许可证](https://github.com/super-kernel/.github/blob/main/LICENSE)发布。
