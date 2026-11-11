<p align="right">
  <strong>English</strong> | <a href="building-and-metadata.zh-CN.md">中文文档</a>
</p>

# Building and metadata

A SuperKernel application does not discover its classes when it starts. `skernel build` discovers them once and writes what it found as metadata; the application restores that metadata at start. This guide explains what the build writes, when it must run again, and how a missing or outdated build is reported. [Documentation index](../README.md)

## Why the application does not scan at start

Discovering services means loading every class of every installed package and reading its Attributes. Doing that at every start would make the start slow, load code the application never uses, and make a packaged application depend on its sources being scannable. So the work is split:

- `skernel build` loads the project's classes in separate PHP processes, collects their Attributes, and writes the results.
- At start, `super-kernel/metadata` restores the package metadata, `super-kernel/annotator` restores the Attribute records, and `super-kernel/di` builds the container from them. Restoration only reads generated files and hashes `composer.lock`: it never scans sources, parses Composer files, or starts a process, so it also works after the project moves and inside a packaged application.

The cost is that the metadata must match the code. `skernel run` and `skernel package` build before they run or archive, so the question only arises when you start the PHP entry yourself.

## What `skernel build` generates

`skernel build` works on the project in the current directory, or the one given with `-d`, which must have its dependencies installed, including `super-kernel/metadata`, `super-kernel/context`, and `super-kernel/di`. It first regenerates the project's Composer autoloader, so classes added since the last build are found, then runs in two separate processes of the managed PHP:

1. The first loads the project's own autoloader, checks the build protocol of the installed `super-kernel/metadata`, maps the classes of every installed package, collects the Attributes of the classes that declare any, and publishes the records through the project's own `super-kernel/metadata` and `super-kernel/annotator`.
2. The second creates the container from those records and publishes `SuperKernel\Event\ApplicationBuild`, so that components write what they prepare at build time, such as the proxies of `super-kernel/aop`. It does not create or run the application.

Everything is written below the vendor directory, honouring Composer's `config.vendor-dir`:

| File | Content |
| --- | --- |
| `vendor/.cache/build.json` | The build stamp: the build protocol, the SKERNEL version, the SHA-256 of `composer.lock`, and the project directory relative to the vendor directory. |
| `vendor/.cache/metadata.ser` | Every installed package with its class map. |
| `vendor/.cache/packages.ser` | The index of the Attribute records: every package with its installed reference. |
| `vendor/.cache/annotations/<vendor>/<package>.ser` | The Attribute records of one package. |
| `vendor/.cache/mimic/` | The proxies that `super-kernel/aop` prepares, when the project declares aspects. |

A build stamp looks like this:

```json
{
    "protocol": 1,
    "skernel": "1.0.0-alpha.2",
    "lock": "ee15a742b47d0086ea577b57ea8a06b72e7a54a11564a449ac560c7c2a439961",
    "base-path": ".."
}
```

The files are derived data: keep them out of version control, ship them with the application, and let `skernel build` rewrite them. `skernel package` builds a fresh copy for the package it writes, and `skernel clean` removes them. Everything the build processes print goes to STDERR; on success STDOUT carries one line:

```text
Build artifacts written to /path/to/my-app/vendor/.cache.
```

## When to build again

Run `skernel build` again, or use `skernel run`, which builds first:

- after `composer install` or `composer update`;
- after changing autoload rules in `composer.json`;
- after adding, removing, or changing a class that carries Attributes, or an Attribute itself;
- after switching between an installation with and without development dependencies.

The application detects only some of these. At start it compares the build stamp's protocol with the installed `super-kernel/metadata` and the stamp's hash with the current `composer.lock`. A changed class, a changed autoload rule, or a switch between `composer install` and `composer install --no-dev` that leaves `composer.lock` unchanged is not detected: the application runs with the old metadata, so a new service, listener, or route is simply missing.

Adding a class needs only the next build: the build regenerates Composer's autoloader before it loads the project's classes, so the new class is included even though the skeleton makes Composer's class map authoritative.

## A missing or outdated build

When the generated files are missing, invalid, or do not match the installation, restoring the container raises `SuperKernel\Metadata\Exception\BuildRequiredException`. Its message states the problem and always ends with ``Run `skernel build` to generate the project metadata.``:

| Problem | Message |
| --- | --- |
| No build stamp | ``The build stamp <vendor>/.cache/build.json is missing. Run `skernel build` to generate the project metadata.`` |
| The stamp cannot be read | ``The build stamp is invalid. Run `skernel build` to generate the project metadata.`` or ``The build stamp has an unexpected structure. Run `skernel build` to generate the project metadata.`` |
| The stamp was written under another build protocol | ``The metadata was generated by SKERNEL <version> under build protocol <n>, but this project expects <n>. Run `skernel build` to generate the project metadata.`` |
| No `composer.lock` | ``The project <directory> has no readable composer.lock. Run `skernel build` to generate the project metadata.`` |
| `composer.lock` changed since the build | ``The metadata generated by SKERNEL <version> does not match the current composer.lock. Run `skernel build` to generate the project metadata.`` |
| The package metadata is missing or damaged | ``The package records <vendor>/.cache/metadata.ser are missing. Run `skernel build` to generate the project metadata.``, ``The package records are invalid. …``, or ``The package records cannot be restored. …`` |
| The Attribute records are missing, damaged, or from other package versions | ``The generated annotation records cannot be restored. Run `skernel build` to generate the project metadata.``, with the reason from `super-kernel/annotator` as its previous exception |

The application entry obtains its container through `SuperKernel\Context\ApplicationContext::getContainer()`, which reports the failure as `SuperKernel\Context\Exception\ContainerInitializationException` with the `BuildRequiredException` as its previous exception. Started without a build, the skeleton's entry ends with an uncaught exception whose first lines are:

```text
PHP Fatal error:  Uncaught SuperKernel\Metadata\Exception\BuildRequiredException: The build stamp /path/to/my-app/vendor/.cache/build.json is missing. Run `skernel build` to generate the project metadata. in …
Next SuperKernel\Context\Exception\ContainerInitializationException: Unable to initialize the application container through SuperKernel\Di\Provider\ContainerProvider. in …
```

## Skipped classes

The build skips a class that it cannot load or whose Attribute it cannot create, even when loading it ends the process with a fatal error. After the processes finish, it lists the skipped classes on STDERR, one per line, sorted by package and class, with the error type and message:

```text
SKERNEL skipped 2 classes whose attributes could not be collected:
  acme/app App\Broken\Listener: Error: Class "App\Missing" not found
  acme/app App\Broken\Proxy: fatal error: Declaration of App\Broken\Proxy::run() must be compatible with ...
```

The build still succeeds, but a skipped class has no metadata: its service mapping, listener, route, or other Attribute does not exist for the application. Fix the class and build again. A list in a new skeleton project names classes of optional Symfony integrations whose dependencies are not installed, such as `Symfony\Component\Console\ConsoleBundle`; those can be ignored.

The application does not print the list at start. The records keep it: the container's `SuperKernel\Contract\AnnotationCollectorInterface` also implements `SuperKernel\Contract\ScanDiagnosticsInterface`, whose `getDiagnostics()` returns one row per skipped class with the fields `sequence`, `package`, `class`, `exception_type`, and `message`.

## The build protocol

The format of the generated files has a version, the build protocol, which the installed `super-kernel/metadata` declares as `SuperKernel\Metadata\Constants\Protocol::VERSION`. Each SKERNEL release generates one protocol. Before it writes anything, `skernel build` compares the two and stops with a message that says which side to upgrade:

| Situation | Message |
| --- | --- |
| The project's packages are newer than SKERNEL | `The project's super-kernel/metadata uses build protocol <n>, but this SKERNEL generates protocol <n>. Upgrade SKERNEL, for example with "skernel self-update".` |
| SKERNEL is newer than the project's packages | `The project's super-kernel/metadata uses build protocol <n>, but this SKERNEL generates protocol <n>. Upgrade the project's SuperKernel packages, or use a SKERNEL release that generates protocol <n>.` |
| The project does not install `super-kernel/metadata` | `The project does not install super-kernel/metadata, so SKERNEL cannot publish its build artifacts. Require SuperKernel 1.0.0-alpha.2 or later.` |

The message is printed on STDERR, and `skernel build` exits with status 1. At start, the application checks the stamp's protocol the same way and raises the `BuildRequiredException` listed above when it differs.

## Moving and packaging a built project

The generated files store paths relative to the project and vendor directories. When the vendor directory is inside the project, the project can move with its build and stay valid; a vendor directory outside the project is recorded as an absolute path. Inside a packaged application the paths start with `phar://`, and the same restoration works. `skernel package` installs, builds, and archives a temporary copy, so the package always carries a build that matches its code.

## Related pages

- [Getting started](getting-started.md): the commands in the order a developer uses them.
- [SuperKernel Metadata](../components/metadata.md), [SuperKernel Annotator](../components/annotator.md), and [SuperKernel DI](../components/di.md): the packages that restore the build.
- [SuperKernel Event](../components/event.md): `ApplicationBuild` and writing build artifacts from a listener.
- The [SKERNEL README](https://github.com/super-kernel/skernel#readme): the `build`, `run`, `package`, and `clean` commands.
