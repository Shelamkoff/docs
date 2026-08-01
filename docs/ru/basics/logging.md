# Основы — Журналирование

Spiral предоставляет компонент `spiral/logger`, соответствующий стандарту
[PSR-3](https://www.php-fig.org/psr/psr-3/). С его помощью можно записывать ошибки, предупреждения, отладочные сообщения
и другие сведения, необходимые для поиска и устранения проблем в приложении.

По умолчанию фреймворк не содержит собственной реализации логгера. Вместо неё доступен компонент
`spiral/monolog-bridge`, полностью интегрированный с пакетом
[Seldaek/monolog](https://github.com/Seldaek/monolog) и поддерживающий различные мощные обработчики журналов.

Фреймворк упрощает настройку обработчиков и позволяет разделять обработку сообщений по разным каналам.

## Конфигурация

Компонент можно настроить через конфигурационный файл или загрузчик. Обычно его конфигурация находится в
`app/config/monolog.php`. В этом файле можно выбрать обработчик по умолчанию, задать глобальный уровень журналирования и
настроить обработчики и процессоры.

Пример конфигурационного файла:

```php app/config/monolog.php
use Monolog\Handler\ErrorLogHandler;
use Monolog\Handler\SyslogHandler;
use Monolog\Logger;
use Monolog\Processor\PsrLogMessageProcessor;

return [
    /**
     * -------------------------------------------------------------------------
     *  Default Monolog handler
     * -------------------------------------------------------------------------
     */
    'default' => env('MONOLOG_DEFAULT_CHANNEL', 'default'),

    /**
     * -------------------------------------------------------------------------
     *  Global logging level
     * -------------------------------------------------------------------------
     *
     * Monolog supports the logging levels described by RFC 5424.
     *
     * @see https://seldaek.github.io/monolog/doc/01-usage.html#log-levels
     */
    'globalLevel' => Logger::toMonologLevel(
        env('MONOLOG_DEFAULT_LEVEL', \Monolog\Logger::DEBUG)
    ),

    /**
     * -------------------------------------------------------------------------
     *  Handlers
     * -------------------------------------------------------------------------
     *
     * @see https://seldaek.github.io/monolog/doc/02-handlers-formatters-processors.html#handlers
     */
    'handlers' => [
        'default' => [
            [
                'class' => 'log.rotate',
                'options' => [
                    'filename' => directory('runtime') . 'logs/app.log',
                    'level' => \Monolog\Logger::DEBUG,
                ],
            ],
        ],
        'stderr' => [
            ErrorLogHandler::class,
        ],
        'stdout' => [
            [
                'class' => SyslogHandler::class,
                'options' => [
                    'ident' => 'app',
                    'facility' => LOG_USER,
                ],
            ],
        ],
    ],

    /**
     * -------------------------------------------------------------------------
     *  Processors
     * -------------------------------------------------------------------------
     *
     * Processors allows adding extra data for all records.
     *
     * @see https://seldaek.github.io/monolog/doc/02-handlers-formatters-processors.html#processors
     */
    'processors' => [
        'default' => [
            [
                'class' => PsrLogMessageProcessor::class,
                'options' => [
                    'dateFormat' => 'Y-m-d\TH:i:s.uP',
                ],
            ],
        ],
    ],
];
```

> **Примечание**
> Переменная окружения `MONOLOG_DEFAULT_CHANNEL` задаёт обработчик по умолчанию для приложения.

### Формат журнала

По умолчанию обработчик форматирует сообщение следующим образом:
`[%datetime%] %level_name%: %message% %context%\n`.

Изменить структуру сообщения можно переменной окружения `MONOLOG_FORMAT`:

```dotenv .env
MONOLOG_FORMAT=[%datetime%] %level_name%: %message% %context%\n
```

> **Смотрите также**
> Перечень доступных заполнителей приведён в
> [документации Monolog](https://seldaek.github.io/monolog/doc/message-structure.html).

## Регистрация обработчика

Помимо конфигурационного файла или собственного загрузчика, обработчики можно регистрировать через класс
`Spiral\Monolog\Bootloader\MonologBootloader`.

### Обработчик с ротацией журналов

```php app/src/Application/Bootloader/LoggingBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Monolog\Bootloader\MonologBootloader;

final class LoggingBootloader extends Bootloader
{
    public function boot(MonologBootloader $monolog): void
    {
        $monolog->addHandler(
            'my-channel',
            $monolog->logRotate(directory('runtime') . 'logs/my-channel.log')
        );
    }
}
```

> **Предупреждение**
> Не забудьте добавить загрузчик в начало списка загрузчиков в `app/src/Application/Kernel.php`.

:::: tabs

::: tab С помощью метода

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \App\Application\Bootloader\LoggingBootloader::class,
        // ...
    ];
}
```

Подробнее о загрузчиках читайте в разделе [Фреймворк — Загрузчики](../framework/bootloaders.md).
:::

::: tab С помощью константы

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \App\Application\Bootloader\LoggingBootloader::class,
    // ...
];
```

Подробнее о загрузчиках читайте в разделе [Фреймворк — Загрузчики](../framework/bootloaders.md).
:::

::::

### Обработчик RoadRunner

Пакет интеграции RoadRunner предоставляет обработчик `Spiral\RoadRunnerBridge\Logger\Handler`, отправляющий записи в
[логгер приложения RoadRunner](https://roadrunner.dev/docs/plugins-applogger).

Добавьте `Spiral\RoadRunnerBridge\Bootloader\LoggerBootloader` в начало списка загрузчиков.

:::: tabs

::: tab С помощью метода

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\RoadRunnerBridge\Bootloader\LoggerBootloader::class,
        // ...
    ];
}
```

Подробнее о загрузчиках читайте в разделе [Фреймворк — Загрузчики](../framework/bootloaders.md).
:::

::: tab С помощью константы

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\RoadRunnerBridge\Bootloader\LoggerBootloader::class,
    // ...
];
```

Подробнее о загрузчиках читайте в разделе [Фреймворк — Загрузчики](../framework/bootloaders.md).
:::

::::

> **Предупреждение**
> Убедитесь, что установлен пакет
> [spiral/roadrunner-bridge](../start/server.md#roadrunner-bridge). Он содержит классы, необходимые для интеграции
> RoadRunner с Monolog.

Затем измените канал по умолчанию на `roadrunner`.

:::: tabs

::: tab Окружение

```dotenv .env
MONOLOG_DEFAULT_CHANNEL=roadrunner
```

:::

::: tab Конфигурация

```php app/config/monolog.php
return [
    'default' => 'roadrunner',
    // ...
];
```

:::

::::

## Использование

### Отправка записей в канал по умолчанию

Для журналирования через канал по умолчанию фреймворк использует `Psr\Log\LoggerInterface`.

```php
use Psr\Log\LoggerInterface;

final class UserService
{
    public function __construct(
        private readonly LoggerInterface $logger
    ) {}

    public function register(string $email, string $password): void
    {
        // Register user ...
        
        $this->logger->info('User has been registered', ['email' => $email]);
    }
}
```

### Отправка записей в определённый канал

Получить логгер определённого канала можно несколькими способами:

- через фабрику логгеров, реализующую `Spiral\Logger\LogsInterface`;
- с помощью атрибута `Spiral\Logger\Attribute\LoggerChannel` у параметра `Psr\Log\LoggerInterface` при автоматическом
  разрешении зависимостей.

:::: tabs

::: tab Через фабрику

```php
use Psr\Log\LoggerInterface;
use Spiral\Logger\LogsInterface;

final class UserService
{
    private readonly LoggerInterface $logger;

    public function __construct(LogsInterface $logs) 
    {
        $this->logger = $logs->channel('my-channel');
    }

    public function register(string $email, string $password): void
    {
        // Register user ...
        
        $this->logger->info('User has been registered', ['email' => $email]);
    }
}
```

:::

::: tab Через атрибут

```php
use Psr\Log\LoggerInterface;
use Spiral\Logger\Attribute\LoggerChannel;

final class UserService
{
    public function __construct(
        #[LoggerChannel('my-channel')]
        private readonly LoggerInterface $logger
    ) {}

    public function register(string $email, string $password): void
    {
        // Register user ...

        $this->logger->info('User has been registered', ['email' => $email]);
    }
}
```

:::

::::

### Трейт логгера

Трейт `Spiral\Logger\Traits\LoggerTrait` позволяет быстро предоставить логгер любому классу. После подключения трейта
класс получает доступ к экземпляру логгера без явного внедрения в конструктор.

> **Предупреждение**
> По умолчанию в качестве **имени канала** используется имя класса.

```php
use Spiral\Logger\Traits\LoggerTrait;

final class UserService
{
    use LoggerTrait;

    public function register(string $email, string $password): void
    {
        // Register user ...
        
        $this->getLogger()->info('User has been registered', ['email' => $email]);
    }
}
```

Назначьте классу логгер:

```php app/src/Application/Bootloader/LoggingBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Monolog\Bootloader\MonologBootloader;

final class LoggingBootloader extends Bootloader
{
    // ...
    
    public function boot(MonologBootloader $monolog): void
    {
        $monolog->addHandler(
            UserService::class,
            $monolog->logRotate(directory('runtime') . 'logs/user-dervice.log')
        );
    }
}
```

> **Предупреждение**
> `LoggerTrait` работает только внутри глобальной [области IoC](../framework/scopes.md).

### Обработка только определённых уровней

Иногда требуется записывать только определённые уровни сообщений. Например, все ошибки приложения можно собирать в
одном файле.

**Для этого подпишитесь на канал по умолчанию:**

```php app/src/Application/Bootloader/LoggingBootloader.php
namespace App\Application\Bootloader;

use Monolog\Logger;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Monolog\Bootloader\MonologBootloader;

final class LoggingBootloader extends Bootloader
{
    // ...
    
    public function boot(MonologBootloader $monolog): void
    {
        $monolog->addHandler(
            'default',
            $monolog->logRotate(directory('runtime') . 'logs/errors.log', Logger::ERROR) // only ERROR and above
        );
    }
}
```
