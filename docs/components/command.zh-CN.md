<p align="right">
  <a href="command.md">English</a> | <strong>中文文档</strong>
</p>

# SuperKernel Command

包：[`super-kernel/command`](https://github.com/super-kernel/command) · [Packagist](https://packagist.org/packages/super-kernel/command) · [文档索引](../README.zh-CN.md)

## 用途

`super-kernel/command` 把类变成应用的 Symfony Console 命令。类用 `#[Command]` 注解声明它的命令；本包注册所声明的名称，并为应用提供一个命令加载器，只在命令被使用时才通过容器创建它。

Symfony Console 运行命令，并负责命令的输入、输出和延迟包装。应用本身来自 `super-kernel/framework`，容器来自 `super-kernel/di`，由容器负责每个命令的生命周期。

## 能力

### 声明命令

命令是一个继承 `Symfony\Component\Console\Command\Command` 并带有 `SuperKernel\Command\Attribute\Command` 的类。两者都名为 `Command`，因此请以别名（例如 `SymfonyCommand`）导入 Symfony 的类。该注解接受：

- `name`：主名称，必填且不能为空；
- `description`：由 `list` 和 `help` 显示，默认为 `null`；
- `aliases`：更多非空名称的列表，默认为空；
- `hidden`：为 `true` 时在 `list` 中隐藏命令，默认为 `false`。

每个名称和别名在整个应用中必须唯一，在同一个声明内也是如此。只有类上的 `#[Command]` 注解才会注册命令：Symfony 的 `#[AsCommand]` 会被忽略，该注解也不能放在方法上。类由哪个包声明并不重要。

一个命令，保存为 `src/Command/HelloCommand.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Command;

use SuperKernel\Command\Attribute\Command;
use Symfony\Component\Console\Command\Command as SymfonyCommand;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

#[Command(name: 'app:hello', description: 'Print a greeting.', aliases: ['hello'])]
final class HelloCommand extends SymfonyCommand
{
    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $output->writeln('Hello');

        return self::SUCCESS;
    }
}
```

构建项目，然后分别用名称和别名运行命令：

```bash
skernel build
php bin/bootstrap.php app:hello
php bin/bootstrap.php hello
```

它会打印：

```text
Hello
Hello
```

### 向命令注入服务

命令由容器创建，因此与容器创建的其他类一样，命令通过构造函数接收服务。调用父类构造函数时不传参数：名称、描述和别名来自注解。

创建是延迟的。发现阶段不创建任何命令；`list` 以及对未知名称的建议只读取声明，因此即使某个命令的依赖无法解析，它们也能正常工作。Symfony 为了运行命令、显示它的帮助或补全它的输入而初始化命令，只有到那时容器才会创建它。创建失败（例如无法解析的构造函数参数）在那一刻抛出：应用把它交给异常分发器，运行以非零状态结束。容器会保留它创建的命令，就像保留每个条目一样。

一个服务，保存为 `src/Billing/InvoiceRepository.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Billing;

use function count;

final class InvoiceRepository
{
    /** @var list<string> */
    private array $open = ['INV-1001', 'INV-1002'];

    public function countOpen(): int
    {
        return count($this->open);
    }
}
```

使用它的命令，保存为 `src/Command/CountInvoicesCommand.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Command;

use Src\Billing\InvoiceRepository;
use SuperKernel\Command\Attribute\Command;
use Symfony\Component\Console\Command\Command as SymfonyCommand;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

#[Command(name: 'invoice:count', description: 'Count open invoices.')]
final class CountInvoicesCommand extends SymfonyCommand
{
    public function __construct(private readonly InvoiceRepository $invoices)
    {
        parent::__construct();
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $output->writeln("{$this->invoices->countOpen()} open invoices");

        return self::SUCCESS;
    }
}
```

构建项目并运行命令：

```bash
skernel build
php bin/bootstrap.php invoice:count
```

它会打印：

```text
2 open invoices
```

### 使用内置命令和工具命令

Symfony 的应用自己提供 `help`、`list`、`completion` 和 `_complete`，它们不需要声明。自带可执行文件的工具（例如代码格式化工具）通过该可执行文件运行。要在应用内运行工具，请用 `#[Command]` 声明一个使用该工具公开服务的命令。

### 命令如何到达应用

容器提供带有已注册命令的 `Symfony\Component\Console\CommandLoader\CommandLoaderInterface`。当 `super-kernel/framework` 发布 `SuperKernel\Event\BeforeApplicationStart` 时，本包的一个监听器会在事件中的应用是 `Symfony\Component\Console\Application` 时把该加载器安装进去；其他类型的应用保持不变。安装加载器不会创建任何命令。

加载器与该监听器一起创建，因此无效的声明会在创建监听器的任何地方失败：`skernel build` 在其 `ApplicationBuild` 步骤失败，应用也无法启动。指出问题的 `InvalidArgumentException` 是一个 `SuperKernel\Di\Exception\ResolutionException` 的前一个异常（previous），后者的消息是 `Unable to invoke factory "SuperKernel\Command\Provider\CommandLoaderProvider" for container entry "Symfony\Component\Console\CommandLoader\CommandLoaderInterface".` 由 PHP 自己报告的声明错误（例如参数类型错误）遵循构建时注解收集的规则。

## 错误

声明错误是在创建命令加载器时抛出的 `InvalidArgumentException`；最后一种在初始化命令时抛出。

| 情形 | 消息 |
| --- | --- |
| 空名称 | `SuperKernel\Command\Attribute\Command on <class> must declare a non-empty command name.` |
| 别名不是非空字符串的列表 | `Command aliases on <class> must be a list of non-empty strings.` |
| 一个名称或别名被声明两次（包括在同一个类内） | `Command name "<name>" is declared by both <class> and <class>.` |
| `#[Command]` 位于没有继承 Symfony `Command` 的类上 | `<class> declares SuperKernel\Command\Attribute\Command but does not extend Symfony\Component\Console\Command\Command.` |
| 命令类的容器条目不是 Symfony 命令（`UnexpectedValueException`） | `Container entry <class> must resolve to Symfony\Component\Console\Command\Command, <type> returned.` |

未知名称由 Symfony Console 报告，例如 `Command "invoice:imprt" is not defined.`，并附上最接近的名称。

## 与其他包的关系

- [`super-kernel/framework`](framework.zh-CN.md) 创建应用并发布本包监听的 `BeforeApplicationStart`；Symfony Console 运行命令。
- [`super-kernel/event`](event.zh-CN.md) 声明 `BeforeApplicationStart`；[`super-kernel/event-dispatcher`](event-dispatcher.zh-CN.md) 发现安装加载器的监听器。
- [`super-kernel/di`](di.zh-CN.md) 创建并保留命令及其依赖；[`super-kernel/annotator`](annotator.zh-CN.md) 提供记录下来的 `#[Command]` 声明。
- `skernel build` 收集声明，声明无效时构建失败；`skernel run` 在启动应用之前先构建。

## 不要这样做

**不要只用 Symfony 的 `#[AsCommand]` 注册命令。** 本包会忽略它，因此该命令不存在：运行它时 Symfony 报告 `Command "<name>" is not defined.`，或者在没有命令共享其命名空间时报告 `There are no commands defined in the "<namespace>" namespace.` 请用 `#[Command]` 声明命令。

**不要重复使用名称或别名。** 同一名称的两个声明（包括类在自己的别名中重复其名称）会使 `skernel build` 和每次应用启动失败，并报告 `Command name "<name>" is declared by both <class> and <class>.` 请为每个命令使用各自的名称。
