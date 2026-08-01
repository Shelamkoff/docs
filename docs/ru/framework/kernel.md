# Фреймворк — Ядро и окружение

Spiral использует объект ядра, содержащий набор сервисов конкретного приложения. В отличие от Symfony, Spiral требуется
одно ядро для всех способов обработки запросов: HTTP, очередей, gRPC, консоли и других. Ядро автоматически выбирает
подходящий способ обработки в зависимости от подключённого [диспетчера](../framework/dispatcher.md).

> **Примечание**
> Базовая реализация ядра находится в репозитории `spiral/boot`.

## Обязанности ядра

Класс `Spiral\Boot\AbstractKernel` отвечает за следующие части приложения:

- инициализацию контейнера набором загрузчиков приложения;
- инициализацию загрузчиков;
- инициализацию окружения и структуры каталогов;
- инициализацию обработчика исключений, если он необходим;
- выбор подходящего диспетчера.

Для создания ядра приложения необходимо унаследоваться от `Spiral\Boot\AbstractKernel`:

```php app/src/Application/MyApp.php
namespace App\Application;

use Spiral\Boot\AbstractKernel;
use Spiral\Boot\Exception\BootException;

final class MyApp extends AbstractKernel
{
    protected const LOAD = [
        // bootloaders to initialize
    ];

    protected function bootstrap(): void
    {
        // custom initialization code
        // invoked after all bootloaders are loaded
    }

    protected function mapDirectories(array $directories): array
    {
        if (!isset($directories['root'])) {
            throw new BootException('Missing required directory `root`');
        }

        if (!isset($directories['app'])) {
            $directories['app'] = $directories['root'] . '/app/';
        }

        return \array_merge(
            [
                // public root
                'public'    => $directories['root'] . '/public/',

                // vendor libraries
                'vendor'    => $directories['root'] . '/vendor/',

                // data directories
                'runtime'   => $directories['root'] . '/runtime/',
                'cache'     => $directories['root'] . '/runtime/cache/',

                // application directories
                'config'    => $directories['app'] . '/config/',
                'resources' => $directories['app'] . '/resources/',
            ],
            $directories
        );
    }
}
```

> **Примечание**
> `Spiral\Framework\Kernel` определяет карту каталогов по умолчанию.

## Инициализация ядра

Для инициализации ядра вызовите статический метод `create`:

```php app.php
$myapp = MyApp::create(
    directories: [
        'root' => __DIR__,
    ],
    handleErrors: false // do not mount error handler
);

$myapp->run(environment: null); // use default env 

\dump($myapp->get(\Spiral\Boot\DirectoriesInterface::class)->getAll());
```

> **Примечание**
> Во время инициализации `MyApp` будет связан в контейнере с `Spiral\Boot\KernelInterface` как singleton.

### Обратные вызовы

Класс `Spiral\Boot\AbstractKernel` предоставляет несколько обратных вызовов, выполняемых на разных этапах инициализации
приложения: `running`, `booting`, `booted` и `bootstrapped`. Класс `Spiral\Framework\Kernel`, наследующий
`AbstractKernel`, добавляет обратные вызовы `appBooting` и `appBooted`. Это позволяет выполнять пользовательские действия
на определённых этапах инициализации.

> **Примечание**
> В наборе приложения стандартный класс `App\Application\Kernel` наследуется от `Spiral\Framework\Kernel` и использует
> эти обратные вызовы.

#### Running

`running` — первый обратный вызов при инициализации приложения. Он выполняется при вызове метода `run`, сразу после
связывания `EnvironmentInterface` с контейнером приложения.

Пример:

```php app.php
$app = MyApp::create(directories: ['root' => __DIR__]);

$app->running(static function (): void {
    // Do something
});

$app->run();
```

> **Примечание**
> Метод регистрации можно вызывать несколько раз. Обратные вызовы выполняются в порядке регистрации.

#### Booting

Обратный вызов `booting` выполняется до запуска всех загрузчиков фреймворка из секции `LOAD`.

Зарегистрировать его можно двумя способами.

:::: tabs

::: tab При инициализации ядра

Метод `booting` можно вызвать у созданного экземпляра приложения.

```php app.php
$app = MyApp::create(
    directories: ['root' => __DIR__]
);

$app->booting(function () {
    // ...
});

$app->run();
```

:::

::: tab Через загрузчик ядра

Обратный вызов этапа `booting` также можно зарегистрировать в методе `init` загрузчика. Сам метод `init` выполняется до
срабатывания обратного вызова.

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Boot\KernelInterface;

class AppBootloader extends Bootloader
{
    public function init(KernelInterface $app): void
    {
        $app->booting(function () {
            // ...
        });
    }
}
```

:::

::::

#### Booted

Обратный вызов `booted` выполняется после завершения инициализации всех загрузчиков фреймворка из секции `LOAD`.

```php
$app->booted(function () {
    // ...
});
```

#### AppBooting

Обратный вызов `appBooting` выполняется до запуска всех загрузчиков приложения из секции `APP`.

```php
$app->appBooting(function () {
    // ...
});
```

#### AppBooted

Обратный вызов `appBooted` выполняется после завершения инициализации всех загрузчиков приложения из секции `APP`.

```php
$app->appBooted(function () {
    // ...
});
```

## Окружение

Spiral интегрируется с [Dotenv](https://github.com/vlucas/phpdotenv) через класс
`Spiral\DotEnv\Bootloader\DotenvBootloader`. Этот загрузчик читает переменные окружения из файла `.env` и делает их
доступными приложению.

### Переменные окружения

Интерфейс `Spiral\Boot\EnvironmentInterface` предоставляет доступ к переменным окружения. По умолчанию фреймворк
использует значения системного уровня. Их можно переопределить при инициализации ядра, передав пользовательский объект
`Spiral\Boot\Environment` методу `run`.

> **Смотрите также**
> Подробнее об окружениях приложения читайте в разделе
> [Начало работы — Конфигурация](../start/configuration.md).

Пример:

```php app.php
use \Spiral\Boot\Environment;

// Create an application instance ...

$app->run(new Environment(['DEBUG' => true]));

\dump($app->get(\Spiral\Boot\EnvironmentInterface::class)->getAll());
```

> **Примечание**
> Такой подход можно использовать для начальной загрузки приложения в тестах.

### Расположение файла .env

По умолчанию загрузчик ищет `.env` в корне проекта. Расположение можно изменить, задав переменную окружения
`DOTENV_PATH` при запуске ядра:

```php app.php
use Spiral\Boot\Environment;

$app = App\Application\Kernel::create(...);

$app->run(new Environment(['DOTENV_PATH' => __DIR__ . '/.env.production']));
```

> **Примечание**
> Можно создать собственную реализацию класса `DotenvBootloader` и изменить поведение загрузки переменных окружения:
> например, каталог поиска `.env` или дополнительную обработку. Это полезно, когда стандартный загрузчик не соответствует
> требованиям приложения.

### Перезапись переменных

По умолчанию Spiral не перезаписывает ранее установленные переменные окружения значениями из `.env`. Поведение можно
изменить, передав `true` параметру `overwrite` при создании `Environment`.

```php app.php
use Spiral\Boot\Environment;

$app = App\Application\Kernel::create(...);

$app->run(new Environment([
    'APP_ENV' => 'production'
], overwrite: true));
```

## События

| Событие                              | Описание                                                                                                      |
|--------------------------------------|---------------------------------------------------------------------------------------------------------------|
| Spiral\Boot\Event\Bootstrapped       | Вызывается `после` инициализации всех загрузчиков из секций `SYSTEM`, `LOAD` и `APP`                          |
| Spiral\Boot\Event\Serving            | Вызывается `до` поиска диспетчера для обработки входящих запросов в текущем окружении                         |
| Spiral\Boot\Event\DispatcherFound    | Вызывается после обнаружения диспетчера для обработки входящих запросов в текущем окружении                   |
| Spiral\Boot\Event\DispatcherNotFound | Вызывается, если диспетчер приложения не найден                                                               |
| Spiral\Boot\Event\Finalizing         | Вызывается перед выполнением финализаторов                                                                    |

> **Примечание**
> Подробнее о диспетчеризации событий читайте в разделе [События](../advanced/events.md).
