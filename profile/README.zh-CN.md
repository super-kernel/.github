<div align="center">

# SuperKernel

[![PHP Version](https://img.shields.io/badge/php-%3E%3D8.4-blue)](https://www.php.net/)
[![Swoole](https://img.shields.io/badge/swoole-%3E%3D6.0-green)](https://www.swoole.com/)
[![Platform](https://img.shields.io/badge/platform-linux-lightgrey.svg?logo=linux)](https://www.kernel.org)
[![License](https://img.shields.io/badge/license-MIT-orange)](../LICENSE)
[![Code Style](https://img.shields.io/badge/code%20style-PSR--12-lightgrey)](https://www.php-fig.org/psr/psr-12/)
[![Made with Love](https://img.shields.io/badge/made%20with-%F0%9F%A7%A1%20and%20%F0%9F%92%9A-blueviolet)]()

</div>

---

> **SuperKernel** 是一个以 **Swoole 扩展** 为核心驱动力的现代化 PHP 框架。  
> 框架的所有组件均遵循 **PSR 规范** 实现，并通过 **DI 容器** 与 **Skernel 工具集** 提供灵活的组件替换与扩展能力。  
> 致力于探索 **PHP 在高性能与系统编程层面的未来可能性**。

---

## 🚀 设计初衷

1. **免环境安装** —— 一键运行，无需繁琐配置。
2. **自动化启动** —— 框架全权接管生命周期。
3. **原生 AOP 支持** —— 面向切面编程的优雅实现。
4. **以 Swoole 为核心** —— 拥抱异步、协程与高并发。
5. **多协议统一支持** —— HTTP、WebSocket、TCP、UDP 等协议无缝接入。
6. **PSR 规范组件化** —— 可替换、可扩展、可独立运行。
7. **强类型配置系统** —— 严格约束与清晰定义。

---

## ⚙️ 安装

### Composer

如果你的系统尚未安装 PHP 环境，可以前往 👉 [wheakerd/skernel - Releases](https://github.com/wheakerd/skernel/releases)
下载可执行的 Composer 二进制文件（无需安装`php`环境）。

### Skernel

访问 [Skernel](https://github.com/wheakerd/skernel) 仓库,了解如何下载安装与使用该构建工具。

### 安装 SuperKernel

通过 Composer 创建一个全新的 SuperKernel 项目：

```bash
composer create-project super-kernel/super-kernel-skeleton
```

## 🧩 Quick Start

### 配置 skernel 工具

```json
{
  "description": "SuperKernel 框架的项目模板。",
  "type": "project",
  "license": "MIT",
  "extra": {
    "skernel": {
      "name": "skernel"
      // 构建时选择的名称，默认使用 `bin`。
    }
  }
}
```

```
skernel build \
&& chmod +x target/release/bin \
&& target/release/bin start
```

- 若定义了构建产物的名称，请以实际产物运行。

## 🧠 架构概览

```text
SuperKernel
 ├── Skernel Tools      # 工具与运行时支持、切面编程与类扫描机制
 ├── DI Container       # 定义器、解析器、惰性实例存储
 ├── Server Components  # HTTP / WebSocket / TCP 等服务模块
 ├── Event Dispatcher   # 生命周期与事件系统
 ├── PSR-Compatible     # PSR-3, PSR-7, PSR-11, PSR-15, PSR-17
```

## 📦 当前进展

SuperKernel 已实现核心运行机制，包括：

- 容器定义源与解析器工厂体系
- 生命周期调度器与自定义进程机制
- 组件自动注册与服务管理器
- 类映射生成

⚠️ 注意：
项目仍在持续开发中。若用于生产，请确保具备较高的 PHP 与 Swoole 开发经验。

## 🧭 路线图

- [ ] 完善测试体系
- [ ] 插件化扩展系统
- [x] 多服务协同调度
- [ ] AOP 扫描器优化
- [ ] 官方文档与示例完善

## 📜 许可证

本项目基于 MIT License 开源。

## 💬 致开发者

> SuperKernel 不仅是一个框架，更是一种理念：
> “让 PHP 回归系统编程，探索语言的极限。”