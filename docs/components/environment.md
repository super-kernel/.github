<p align="right">
  <strong>English</strong> | <a href="environment.zh-CN.md">中文文档</a>
</p>

# SuperKernel Environment

Package: [`super-kernel/environment`](https://github.com/super-kernel/environment) · [Packagist](https://packagist.org/packages/super-kernel/environment) · [Documentation index](../README.md)

## Purpose

`super-kernel/environment` is where an application reads values from outside its code: process environment variables, `.env` files, and arrays it builds itself. It combines these sources in a fixed order and converts their raw values strictly, so that a typo such as `MAIL_PORT=25x` fails loudly instead of becoming `25` or a silent default.

The package returns raw values only. Defaults and the checks a component needs belong to the application's configuration sources and to the components, typically through `super-kernel/config`.

## Capabilities

### Read typed values

`SuperKernel\Environment\Environment` implements `SuperKernel\Environment\Contract\EnvironmentInterface`:

| Method | Returns |
| --- | --- |
| `has($name)` | Whether any source has the property, also when its value is `null` |
| `get($name)` | The raw value: `string`, `int`, `float`, `bool`, or `null` |
| `getString($name)` | The value as a string; `true` and `false` become `'true'` and `'false'` |
| `getInt($name)` | An integer, from an integer or a string that is exactly an integer |
| `getFloat($name)` | A finite float, from an integer, a float, or a numeric string |
| `getBool($name)` | A boolean, from a boolean, `0` or `1`, or the strings `1`, `0`, `true`, `false`, `yes`, `no`, `on`, `off` in any case, with surrounding spaces ignored |

Every typed accessor returns `null` when no source has the property or the winning value is `null`. Any other value that does not fit raises `SuperKernel\Environment\Exception\InvalidEnvironmentValueException`, whose message names the property, the expected kind, and the type it received, never the value, since external values may be secrets. Nothing falls back to a default.

A script that reads typed values, saved as `bin/environment.php`:

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

Run it with a process variable:

```bash
APP_DEBUG=yes php bin/environment.php
```

It prints:

```text
bool(true)
int(4)
NULL
```

A script that shows the failures, saved as `bin/invalid.php`:

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

`php bin/invalid.php` prints:

```text
Environment property db.password must be an integer; string provided.
Property names must use non-empty lowercase dotted or kebab notation.
bool(true)
NULL
```

### Name properties

A property name is lowercase letters and digits, starting with a letter, in segments joined by `.` or `-`, such as `mail.port` or `app.feature-flag`. Every source uses the same name. The process and `.env` sources look it up as an environment variable: dots and hyphens become underscores and letters become uppercase, so `app.feature-flag` reads `APP_FEATURE_FLAG`. Any other name, such as `DB_PASSWORD` or `Mail.Port`, is rejected with `InvalidPropertyNameException` rather than guessed.

### Order sources

`new Environment($sources)` takes a list of `SuperKernel\Environment\Contract\PropertySourceInterface`, ordered from lowest to highest precedence: the last source that has a property wins. A source that holds `null` for a property still wins, so the result is `null`. Sources are read as they are; the environment changes none of them. Put defaults first and the process environment last, so that a variable set for one run overrides everything else.

- `ArrayPropertySource($name, $values)` holds an array keyed by property name; values must be scalars or `null`. Use it for defaults built in code and in tests.
- `ProcessEnvironmentPropertySource` reads the process environment through `getenv()` at each lookup.
- `DotenvPropertySource($path, $optional = true)` reads one `.env` file, described below.

### Read a `.env` file

`DotenvPropertySource` parses the file once, when it is created, with `symfony/dotenv`, and keeps that snapshot. It never writes to `$_ENV`, `$_SERVER`, or the process environment, so `getenv()` does not see the values.

- A missing file contributes nothing when the source is optional, the default; with `$optional` set to `false` it raises `PropertySourceLoadingException`.
- An unreadable or malformed file always raises `PropertySourceLoadingException`, with the parser's exception as its previous one.
- `name()` returns `dotenv:` followed by the path, for diagnostics.

The file, saved in the project directory as `.env`:

```dotenv
MAIL_HOST=smtp.example.com
MAIL_PORT=2525
```

A script that reads it, saved as `bin/dotenv.php`:

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

`php bin/dotenv.php` prints the following; the last line reads a file that does not exist:

```text
smtp.example.com:2525
bool(false)
bool(false)
```

### Use the environment in an application

`SuperKernel\Environment\Provider\EnvironmentProvider` carries `#[FactoryProvider(EnvironmentInterface::class)]` and supplies an environment over the process environment only. To read a `.env` file as well, the application declares its own `EnvironmentInterface` with `#[Provider(EnvironmentInterface::class)]`, which takes precedence over the factory provider, and puts the `.env` source below the process source.

The application reads the environment in its configuration sources: each typed value with its default, so that a missing property uses the default and an invalid one fails. Components then receive typed configuration, never raw properties.

An environment with a `.env` file, saved as `src/Env/AppEnvironment.php`:

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

A configuration interface, saved as `src/Mail/MailConfig.php`:

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

A configuration source that reads the environment with defaults, saved as `src/Mail/EnvironmentMailConfig.php`:

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

A script that reads the configuration, saved as `bin/mail.php`:

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

Build the project and run the script with and without a process variable:

```bash
skernel build
php bin/mail.php
MAIL_PORT=465 php bin/mail.php
```

With the `.env` file above, it prints `smtp.example.com:2525`, then `smtp.example.com:465`; without the file, `localhost:25`.

### Write a property source

A source implements `PropertySourceInterface`: `name()` returns a non-empty name for diagnostics, `has($name)` reports whether the source holds the property, also with a `null` value, and `get($name)` returns the raw `string`, `int`, `float`, `bool`, or `null`. `Environment` rejects anything in its list that is not a source with `InvalidPropertySourceException`.

## Errors

Every exception of the package implements `SuperKernel\Environment\Exception\EnvironmentExceptionInterface`.

| Situation | Message |
| --- | --- |
| A name that is not lowercase dotted or kebab notation (`InvalidPropertyNameException`) | `Property names must use non-empty lowercase dotted or kebab notation.` |
| A value that is not an integer (`InvalidEnvironmentValueException`) | `Environment property <name> must be an integer; <type> provided.` |
| A value that is not a finite number (`InvalidEnvironmentValueException`) | `Environment property <name> must be a finite number; <type> provided.` |
| A value that is not a recognized boolean (`InvalidEnvironmentValueException`) | `Environment property <name> must be a boolean; <type> provided.` |
| A missing required `.env` file (`PropertySourceLoadingException`) | `Required dotenv file <path> does not exist.` |
| An unreadable `.env` file (`PropertySourceLoadingException`) | `Unable to read dotenv file <path>.` |
| A malformed `.env` file (`PropertySourceLoadingException`, parser exception as previous) | `Unable to parse dotenv file <path>.` |
| Sources that are not a list (`InvalidPropertySourceException`) | `Environment property sources must be a list.` |
| An entry that is not a source (`InvalidPropertySourceException`) | `Environment property sources must implement SuperKernel\Environment\Contract\PropertySourceInterface; <type> provided.` |
| An empty array source name (`InvalidPropertySourceException`) | `A property source name must not be empty.` |
| An array source key that is not a string (`InvalidPropertySourceException`) | `Property source keys must be canonical strings; <type> provided.` |
| An array source value that is not a scalar or `null` (`InvalidPropertySourceException`) | `Property source value for <name> must be scalar or null; <type> provided.` |

## Relation to other packages

- [`super-kernel/attribute`](attribute.md) declares the `#[FactoryProvider]` Attribute on the provider; [`super-kernel/di`](di.md) creates the environment and lets an application `#[Provider]` take precedence.
- [`super-kernel/config`](config.md) sources read the environment and build the configuration objects that components receive.
- [`super-kernel/metadata`](metadata.md) supplies `BasePathInterface`, the project directory where an application keeps its `.env` file.

## What not to do

**Do not use environment-variable names.** `getInt('MAIL_PORT')` raises `Property names must use non-empty lowercase dotted or kebab notation.` Use the property name, `mail.port`, which the process and `.env` sources map to `MAIL_PORT`.

**Do not catch conversion errors to apply a default.** An explicit value of the wrong kind is a mistake in the deployment, reported as `Environment property <name> must be an integer; string provided.` Apply a default only to `null`, with `?? $default`, and let invalid values fail.

**Do not expect a `.env` file to set environment variables.** `DotenvPropertySource` keeps its values to itself, so `getenv()` and code that reads `$_ENV` do not see them. Read properties through `EnvironmentInterface`.

**Do not read the environment inside components.** Components receive typed configuration; the application's configuration sources read the environment and choose defaults, so each component's values come from one place.
