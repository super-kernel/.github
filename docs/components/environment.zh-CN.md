<p align="right">
  <a href="environment.md">English</a> | <strong>中文文档</strong>
</p>

# SuperKernel Environment

包：[`super-kernel/environment`](https://github.com/super-kernel/environment) · [Packagist](https://packagist.org/packages/super-kernel/environment) · [文档索引](../README.zh-CN.md)

## 用途

`super-kernel/environment` 是应用从代码之外读取值的地方：进程环境变量、`.env` 文件以及应用自己构建的数组。它按固定顺序组合这些来源，并严格转换它们的原始值，因此像 `MAIL_PORT=25x` 这样的笔误会明确地失败，而不是变成 `25` 或者悄悄地使用默认值。

本包只返回原始值。默认值以及组件所需的检查属于应用的配置来源和各个组件，通常借助 `super-kernel/config` 完成。

## 能力

### 读取类型化的值

`SuperKernel\Environment\Environment` 实现了 `SuperKernel\Environment\Contract\EnvironmentInterface`：

| 方法 | 返回 |
| --- | --- |
| `has($name)` | 是否有任何来源拥有该属性，即使它的值是 `null` |
| `get($name)` | 原始值：`string`、`int`、`float`、`bool` 或 `null` |
| `getString($name)` | 字符串形式的值；`true` 和 `false` 变为 `'true'` 和 `'false'` |
| `getInt($name)` | 整数，来自整数或恰好是整数的字符串 |
| `getFloat($name)` | 有限的浮点数，来自整数、浮点数或数字字符串 |
| `getBool($name)` | 布尔值，来自布尔值、`0` 或 `1`，或者不区分大小写的字符串 `1`、`0`、`true`、`false`、`yes`、`no`、`on`、`off`，忽略首尾空白 |

当没有任何来源拥有该属性，或者胜出的值是 `null` 时，每个类型化读取方法都返回 `null`。其他任何不符合要求的值都会抛出 `SuperKernel\Environment\Exception\InvalidEnvironmentValueException`，其消息给出属性名、期望的类型和收到的类型，但从不包含值本身，因为外部值可能是机密。任何情况下都不会退回到默认值。

读取类型化值的脚本，保存为 `bin/environment.php`：

```php
<?php

declare(strict_types=1);

use SuperKernel\Environment\Environment;
use SuperKernel\Environment\PropertySource\ArrayPropertySource;
use SuperKernel\Environment\PropertySource\ProcessEnvironmentPropertySource;

require dirname(__DIR__) . '/vendor/autoload.php';

$environment = new Environment([
    new ArrayPropertySource('defaults', ['app.debug' => 'off', 'app.workers' => '4']),
    new ProcessEnvironmentPropertySource(),
]);

var_dump($environment->getBool('app.debug'));
var_dump($environment->getInt('app.workers'));
var_dump($environment->getString('app.name'));
```

带一个进程变量运行它：

```bash
APP_DEBUG=yes php bin/environment.php
```

它会打印：

```text
bool(true)
int(4)
NULL
```

展示各种失败的脚本，保存为 `bin/invalid.php`：

```php
<?php

declare(strict_types=1);

use SuperKernel\Environment\Environment;
use SuperKernel\Environment\Exception\InvalidEnvironmentValueException;
use SuperKernel\Environment\Exception\InvalidPropertyNameException;
use SuperKernel\Environment\PropertySource\ArrayPropertySource;

require dirname(__DIR__) . '/vendor/autoload.php';

$environment = new Environment([
    new ArrayPropertySource('settings', ['db.password' => 's3cret', 'cache.ttl' => null]),
]);

try {
    $environment->getInt('db.password');
} catch (InvalidEnvironmentValueException $exception) {
    echo $exception->getMessage(), PHP_EOL;
}

try {
    $environment->get('DB_PASSWORD');
} catch (InvalidPropertyNameException $exception) {
    echo $exception->getMessage(), PHP_EOL;
}

var_dump($environment->has('cache.ttl'), $environment->getInt('cache.ttl'));
```

`php bin/invalid.php` 会打印：

```text
Environment property db.password must be an integer; string provided.
Property names must use non-empty lowercase dotted or kebab notation.
bool(true)
NULL
```

### 为属性命名

属性名称由小写字母和数字组成，以字母开头，各段之间用 `.` 或 `-` 连接，例如 `mail.port` 或 `app.feature-flag`。所有来源都使用同一个名称。进程和 `.env` 来源把它当作环境变量查找：点和连字符变为下划线，字母变为大写，因此 `app.feature-flag` 读取的是 `APP_FEATURE_FLAG`。其他任何名称（例如 `DB_PASSWORD` 或 `Mail.Port`）都会以 `InvalidPropertyNameException` 被拒绝，而不会被猜测。

### 排列来源

`new Environment($sources)` 接收一个 `SuperKernel\Environment\Contract\PropertySourceInterface` 列表，按优先级从低到高排列：拥有该属性的最后一个来源胜出。为某个属性保存了 `null` 的来源同样会胜出，因此结果是 `null`。来源按原样读取；环境不会修改其中任何一个。请把默认值放在最前面，把进程环境放在最后，使为某一次运行设置的变量覆盖其他所有值。

- `ArrayPropertySource($name, $values)` 保存一个以属性名称为键的数组；值必须是标量或 `null`。用于在代码中构建的默认值以及测试。
- `ProcessEnvironmentPropertySource` 在每次查找时通过 `getenv()` 读取进程环境。
- `DotenvPropertySource($path, $optional = true)` 读取一个 `.env` 文件，见下文。

### 读取 `.env` 文件

`DotenvPropertySource` 在创建时用 `symfony/dotenv` 解析文件一次，并保留这份快照。它从不写入 `$_ENV`、`$_SERVER` 或进程环境，因此 `getenv()` 看不到这些值。

- 来源是可选的（默认如此）时，缺失的文件不提供任何值；把 `$optional` 设为 `false` 时，缺失的文件会抛出 `PropertySourceLoadingException`。
- 不可读或格式错误的文件总是抛出 `PropertySourceLoadingException`，以解析器的异常作为其前一个异常（previous）。
- `name()` 返回 `dotenv:` 加上文件路径，用于诊断。

该文件，保存在项目目录中，名为 `.env`：

```dotenv
MAIL_HOST=smtp.example.com
MAIL_PORT=2525
```

读取它的脚本，保存为 `bin/dotenv.php`：

```php
<?php

declare(strict_types=1);

use SuperKernel\Environment\Environment;
use SuperKernel\Environment\PropertySource\DotenvPropertySource;
use SuperKernel\Environment\PropertySource\ProcessEnvironmentPropertySource;

require dirname(__DIR__) . '/vendor/autoload.php';

$dotenv = new DotenvPropertySource(dirname(__DIR__) . '/.env');
$environment = new Environment([$dotenv, new ProcessEnvironmentPropertySource()]);

echo $environment->getString('mail.host'), ':', $environment->getInt('mail.port'), PHP_EOL;
var_dump(getenv('MAIL_HOST'));
var_dump(new DotenvPropertySource(dirname(__DIR__) . '/.env.local')->has('mail.host'));
```

`php bin/dotenv.php` 会打印以下内容；最后一行读取的是一个不存在的文件：

```text
smtp.example.com:2525
bool(false)
bool(false)
```

### 在应用中使用环境

`SuperKernel\Environment\Provider\EnvironmentProvider` 带有 `#[FactoryProvider(EnvironmentInterface::class)]`，提供一个只基于进程环境的环境。要同时读取 `.env` 文件，应用用 `#[Provider(EnvironmentInterface::class)]` 声明自己的 `EnvironmentInterface`，它优先于工厂提供者，并把 `.env` 来源放在进程来源之下。

应用在它的配置来源中读取环境：每个类型化的值都配上默认值，使缺失的属性使用默认值，而无效的属性导致失败。随后各个组件接收类型化的配置，而从不接收原始属性。

带有 `.env` 文件的环境，保存为 `src/Env/AppEnvironment.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Env;

use SuperKernel\Attribute\Provider;
use SuperKernel\Contract\BasePathInterface;
use SuperKernel\Environment\Contract\EnvironmentInterface;
use SuperKernel\Environment\Environment;
use SuperKernel\Environment\PropertySource\DotenvPropertySource;
use SuperKernel\Environment\PropertySource\ProcessEnvironmentPropertySource;

#[Provider(EnvironmentInterface::class)]
final readonly class AppEnvironment implements EnvironmentInterface
{
    private Environment $environment;

    public function __construct(BasePathInterface $basePath)
    {
        $this->environment = new Environment([
            new DotenvPropertySource($basePath . '/.env'),
            new ProcessEnvironmentPropertySource(),
        ]);
    }

    public function has(string $name): bool
    {
        return $this->environment->has($name);
    }

    public function get(string $name): string|int|float|bool|null
    {
        return $this->environment->get($name);
    }

    public function getString(string $name): ?string
    {
        return $this->environment->getString($name);
    }

    public function getInt(string $name): ?int
    {
        return $this->environment->getInt($name);
    }

    public function getFloat(string $name): ?float
    {
        return $this->environment->getFloat($name);
    }

    public function getBool(string $name): ?bool
    {
        return $this->environment->getBool($name);
    }
}
```

配置接口，保存为 `src/Mail/MailConfig.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Mail;

use SuperKernel\Contract\SingleConfigInterface;

interface MailConfig extends SingleConfigInterface
{
    public function host(): string;

    public function port(): int;
}
```

读取环境并提供默认值的配置来源，保存为 `src/Mail/EnvironmentMailConfig.php`：

```php
<?php

declare(strict_types=1);

namespace Src\Mail;

use SuperKernel\Attribute\Configuration;
use SuperKernel\Environment\Contract\EnvironmentInterface;

#[Configuration(MailConfig::class)]
final readonly class EnvironmentMailConfig implements MailConfig
{
    private string $host;

    private int $port;

    public function __construct(EnvironmentInterface $environment)
    {
        $this->host = $environment->getString('mail.host') ?? 'localhost';
        $this->port = $environment->getInt('mail.port') ?? 25;
    }

    public function host(): string
    {
        return $this->host;
    }

    public function port(): int
    {
        return $this->port;
    }
}
```

读取该配置的脚本，保存为 `bin/mail.php`：

```php
<?php

declare(strict_types=1);

use Src\Mail\MailConfig;
use SuperKernel\Context\ApplicationContext;
use SuperKernel\Contract\ConfigInterface;

require dirname(__DIR__) . '/vendor/autoload.php';

$mail = ApplicationContext::getContainer()->get(ConfigInterface::class)->get(MailConfig::class);

echo $mail->host(), ':', $mail->port(), PHP_EOL;
```

构建项目，分别在不带和带进程变量的情况下运行脚本：

```bash
skernel build
php bin/mail.php
MAIL_PORT=465 php bin/mail.php
```

有上面的 `.env` 文件时，它先打印 `smtp.example.com:2525`，再打印 `smtp.example.com:465`；没有该文件时，打印 `localhost:25`。

### 编写属性来源

来源实现 `PropertySourceInterface`：`name()` 返回一个用于诊断的非空名称，`has($name)` 报告该来源是否拥有该属性（即使其值为 `null`），`get($name)` 返回原始的 `string`、`int`、`float`、`bool` 或 `null`。`Environment` 会以 `InvalidPropertySourceException` 拒绝其列表中任何不是来源的条目。

## 错误

本包的每个异常都实现了 `SuperKernel\Environment\Exception\EnvironmentExceptionInterface`。

| 情形 | 消息 |
| --- | --- |
| 名称不是小写的点或连字符形式（`InvalidPropertyNameException`） | `Property names must use non-empty lowercase dotted or kebab notation.` |
| 值不是整数（`InvalidEnvironmentValueException`） | `Environment property <name> must be an integer; <type> provided.` |
| 值不是有限的数字（`InvalidEnvironmentValueException`） | `Environment property <name> must be a finite number; <type> provided.` |
| 值不是可识别的布尔值（`InvalidEnvironmentValueException`） | `Environment property <name> must be a boolean; <type> provided.` |
| 必需的 `.env` 文件缺失（`PropertySourceLoadingException`） | `Required dotenv file <path> does not exist.` |
| `.env` 文件不可读（`PropertySourceLoadingException`） | `Unable to read dotenv file <path>.` |
| `.env` 文件格式错误（`PropertySourceLoadingException`，解析器异常作为前一个异常） | `Unable to parse dotenv file <path>.` |
| 来源不是列表（`InvalidPropertySourceException`） | `Environment property sources must be a list.` |
| 某个条目不是来源（`InvalidPropertySourceException`） | `Environment property sources must implement SuperKernel\Environment\Contract\PropertySourceInterface; <type> provided.` |
| 数组来源的名称为空（`InvalidPropertySourceException`） | `A property source name must not be empty.` |
| 数组来源的键不是字符串（`InvalidPropertySourceException`） | `Property source keys must be canonical strings; <type> provided.` |
| 数组来源的值不是标量或 `null`（`InvalidPropertySourceException`） | `Property source value for <name> must be scalar or null; <type> provided.` |

## 与其他包的关系

- [`super-kernel/attribute`](attribute.zh-CN.md) 声明了提供者上的 `#[FactoryProvider]` 注解；[`super-kernel/di`](di.zh-CN.md) 创建环境，并让应用的 `#[Provider]` 优先。
- [`super-kernel/config`](config.zh-CN.md) 的配置来源读取环境，并构建组件接收的配置对象。
- [`super-kernel/metadata`](metadata.zh-CN.md) 提供 `BasePathInterface`，即应用存放 `.env` 文件的项目目录。

## 不要这样做

**不要使用环境变量名。** `getInt('MAIL_PORT')` 会抛出 `Property names must use non-empty lowercase dotted or kebab notation.`。请使用属性名 `mail.port`，进程和 `.env` 来源会把它映射为 `MAIL_PORT`。

**不要捕获转换错误来套用默认值。** 类型不对的显式值是部署中的错误，会被报告为 `Environment property <name> must be an integer; string provided.`。只对 `null` 用 `?? $default` 套用默认值，让无效的值失败。

**不要期望 `.env` 文件会设置环境变量。** `DotenvPropertySource` 只自己保存这些值，因此 `getenv()` 和读取 `$_ENV` 的代码都看不到它们。请通过 `EnvironmentInterface` 读取属性。

**不要在组件内部读取环境。** 组件接收类型化的配置；由应用的配置来源读取环境并选择默认值，使每个组件的值都来自同一个地方。
