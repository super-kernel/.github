<p align="right">
  <strong>English</strong> | <a href="CONTRIBUTING.zh-CN.md">中文文档</a>
</p>

# Contributing to SuperKernel

Each SuperKernel component is an independent repository and Composer package. Work in the checkout of the component you are changing.

## Requirements

- PHP 8.4.27 or a later 8.4 release.
- Composer.
- The PHP extensions listed in the component's `composer.json` under `require` and `require-dev`.

## Checks

Run these commands from the component's root:

```bash
composer install
composer validate --strict
composer test
composer format:check
```

`composer format` applies the formatting that `composer format:check` reports.

## Environment-gated tests

Some test suites contain tests that need a specific environment, such as ZTS PHP with Swoole thread support or a disposable Redis server. Without it, PHPUnit skips those tests and the skip message names the prerequisite.

## Pull requests

Changes reach a component's `main` branch through pull requests. Keep each pull request focused on one change, and update the component's `README.md` and `README.zh-CN.md` when the change alters what the package offers developers.
