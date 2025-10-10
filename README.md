<p align="right">
  <strong>English</strong> | <a href="README.zh-CN.md">中文文档</a>
</p>

# 🧩 SuperKernel

[![PHP Version](https://img.shields.io/badge/php-%3E%3D8.4-blue)](https://www.php.net/)
[![Swoole](https://img.shields.io/badge/swoole-%3E%3D6.*-green)](https://www.swoole.co.uk/)
[![License](https://img.shields.io/badge/license-MIT-orange)](LICENSE)
[![Code Style](https://img.shields.io/badge/code%20style-PSR--12-lightgrey)](https://www.php-fig.org/psr/psr-12/)
[![Made with Love](https://img.shields.io/badge/made%20with-%F0%9F%A7%A1%20and%20%F0%9F%92%9A-blueviolet)]()

---

> SuperKernel is a modern PHP framework powered by the Swoole extension.
>
> All components of the framework adhere to the PSR specification and provide flexible component replacement and extensibility through the DI container and Skernel toolkit.
>
> We are committed to exploring the future possibilities of PHP in high performance and system programming.

---

## 🚀 Design Philosophy

1. **Zero Environment Setup** – Run instantly without complex installation.
2. **Auto Boot Lifecycle** – Framework-managed startup and shutdown.
3. **Native AOP Support** – Elegant aspect-oriented design.
4. **Swoole-Centric** – Embrace async, coroutine, and concurrency.
5. **Multi-Protocol Support** – Unified abstraction for HTTP, WebSocket, TCP, and UDP.
6. **PSR-Based Components** – Replaceable, extensible, and standalone.
7. **Strongly Typed Configurations** – Clarity and strict validation.

---

## ⚙️ Installation

Use [Composer](https://getcomposer.org/)：

```bash
Template projects to be added.
```

Or install in an existing project:

```bash
composer require super-kernel/framework super-kernel/composer-plugin
```

## 🧩 Quick Start

```bash
composer skernel serve && php target/runtime/bin.php start
```

## 🧠 Architecture Overview

```text
SuperKernel
 ├── Skernel Tools      # Tool and runtime support, aspect programming and class scanning mechanism
 ├── DI Container       # Definers, Resolvers, Lazy Instance Storage
 ├── Server Components  # HTTP/WebSocket/TCP and other service modules
 ├── Event Dispatcher   # Lifecycle and event system
 ├── PSR-Compatible     # PSR-3, PSR-7, PSR-11, PSR-15, PSR-17
```

## 📦 Current Progress
SuperKernel has implemented core operational mechanisms, including:

- Container definition source and resolver factory system
- Lifecycle scheduler and custom process mechanism
- Automatic component registration and service manager
- Class map generation

⚠️ Note:
This project is still under development. If using this for production, please ensure you have advanced PHP and Swoole development experience.

## 🧭 Roadmap
- [ ] Improved testing system
- [ ] Plug-in extension system
- [x] Multi-service coordinated scheduling
- [ ] AOP scanner optimization
- [ ] Improved official documentation and examples

## 📜 License

This project is open source under the MIT License.

## 💬 To Developers
> SuperKernel is not just a framework, but a philosophy:
> "Return PHP to systems programming and explore the limits of the language."