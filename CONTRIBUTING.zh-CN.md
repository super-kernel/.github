<p align="right">
  <a href="CONTRIBUTING.md">English</a> | <strong>中文文档</strong>
</p>

# 为 SuperKernel 做贡献

SuperKernel 的每个组件都是独立的仓库和 Composer 包。请在要修改的那个组件的检出目录中工作。

## 环境要求

- PHP 8.4.27 或更高的 8.4 版本。
- Composer。
- 组件 `composer.json` 中 `require` 和 `require-dev` 列出的 PHP 扩展。

## 检查

在组件根目录运行以下命令：

```bash
composer install
composer validate --strict
composer test
composer format:check
```

`composer format` 会应用 `composer format:check` 报告的格式修正。

## 受环境限制的测试

部分测试套件包含需要特定环境的测试，例如带 Swoole 线程支持的 ZTS PHP，或一个可随时丢弃的 Redis 服务器。缺少相应环境时，PHPUnit 会跳过这些测试，跳过信息会写明缺少的前提条件。

## Pull request

变更通过 pull request 进入组件的 `main` 分支。每个 pull request 只聚焦一项变更；当变更改变了包为开发者提供的能力时，同步更新组件的 `README.md` 和 `README.zh-CN.md`。
