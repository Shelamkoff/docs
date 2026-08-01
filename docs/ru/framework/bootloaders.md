# Фреймворк — Загрузчики

Загрузчики — это классы, которые настраивают приложение при запуске. Они выполняются один раз во время начальной
загрузки, поэтому добавленный в них код не влияет на производительность приложения во время выполнения.

**Типичные задачи загрузчиков:**

- регистрация сервисов и связывание интерфейсов;
- настройка окружения, режима отладки и обработки ошибок;
- настройка подключений к базе данных;
- объявление правил маршрутизации;
- инициализация кеширования, журналирования и системы событий.

![Этапы управления приложением](https://user-images.githubusercontent.com/773481/180768689-c711e6f0-3523-4330-a496-f78088504b29.png)

## Создание загрузчика

Создайте загрузчик с помощью генератора:

```terminal
php app.php create:bootloader GithubClient
```

Будет создан файл `app/src/Application/Bootloader/GithubClientBootloader.php`.

> **Смотрите также**
> [Основы — Генерация кода](../basics/scaffolding.md#bootloader)

## Регистрация загрузчиков

Зарегистрируйте загрузчики в `app/src/Application/Kernel.php`:

```php app/src/Application/Kernel.php
namespace App\Application;

class Kernel extends \Spiral\Framework\Kernel
{
    public function defineBootloaders(): array
    {
        return [
            RoutesBootloader::class,
            // Framework bootloaders
        ];
    }

    public function defineAppBootloaders(): array
    {
        return [
            LoggingBootloader::class,
            MyBootloader::class,
            
            // You can even use anonymous classes
            new class extends Bootloader {
                // ...
            },
        ];
    }
}
```

> **Примечание**
> `defineAppBootloaders()` выполняется после `defineBootloaders()`. Загрузчики конкретного приложения следует размещать
> именно в нём.

## Условная загрузка

Класс `BootloadConfig` позволяет управлять тем, когда и как загружаются загрузчики. Это полезно для возможностей,
зависящих от окружения, необязательных компонентов и динамической конфигурации.

### Параметры конфигурации

| Параметр   | Тип     | По умолчанию | Описание                                                                  |
|------------|---------|---------------|---------------------------------------------------------------------------|
| `args`     | `array` | `[]`          | Аргументы, передаваемые конструктору загрузчика                            |
| `enabled`  | `bool`  | `true`        | Включает или отключает загрузчик                                           |
| `allowEnv` | `array` | `[]`          | Переменные окружения, которые должны совпасть с заданными значениями       |
| `denyEnv`  | `array` | `[]`          | Переменные окружения, которые не должны совпасть с заданными значениями    |
| `override` | `bool`  | `true`        | Может ли конфигурация ядра переопределить конфигурацию атрибута            |

### Основное использование

Управление загрузкой в зависимости от окружения:

```php app/src/Application/Kernel.php
use Spiral\Boot\Attribute\BootloadConfig;

class Kernel extends \Spiral\Framework\Kernel
{
    public function defineBootloaders(): array
    {
        return [
            // Load only in local/dev environments
            PrototypeBootloader::class => new BootloadConfig(
                allowEnv: ['APP_ENV' => ['local', 'dev']]
            ),
            
            // Disable in production
            DebugBootloader::class => new BootloadConfig(
                denyEnv: ['APP_ENV' => 'production']
            ),
            
            // Pass constructor arguments
            CacheBootloader::class => new BootloadConfig(
                args: ['driver' => 'redis', 'ttl' => 3600]
            ),
        ];
    }
}
```

### Использование замыканий

При построении конфигурации замыкание позволяет получать сервисы контейнера:

```php
use Spiral\Boot\Environment\AppEnvironment;

PrototypeBootloader::class => static fn(AppEnvironment $env) => 
    new BootloadConfig(
        enabled: $env->isLocal(),
        args: ['debug' => $env->get('DEBUG')]
    ),
```

**Когда использовать:** для динамической конфигурации, зависящей от сложного состояния приложения или нескольких
сервисов.

### Использование атрибутов

Конфигурацию можно объявить непосредственно в классе загрузчика:

```php app/src/Application/Bootloader/DevToolsBootloader.php
use Spiral\Boot\Attribute\BootloadConfig;
use Spiral\Boot\Bootloader\Bootloader;

#[BootloadConfig(
    allowEnv: ['APP_ENV' => ['local', 'development']],
    denyEnv: ['TESTING' => true]
)]
final class DevToolsBootloader extends Bootloader
{
    public function __construct(
        private readonly bool $profiling = false
    ) {}
}
```

**Когда использовать:** для самодостаточных загрузчиков со встроенными условиями загрузки.

### Приоритет конфигурации

Если загрузчик имеет и атрибут, и конфигурацию времени выполнения, параметр `override` определяет приоритет:

```php
// In the bootloader
#[BootloadConfig(
    args: ['debug' => true],
    override: false  // Prevent runtime override
)]
final class MyBootloader extends Bootloader
{
    public function __construct(
        private readonly bool $debug
    ) {}
}

// In the Kernel - this will be IGNORED due to override: false
MyBootloader::class => new BootloadConfig(args: ['debug' => false]),
```

**Когда использовать:** чтобы зафиксировать конфигурацию критически важных загрузчиков, которую нельзя изменять во время
выполнения.

### Пользовательские классы конфигурации

Для повторного использования конфигурации унаследуйтесь от `BootloadConfig`:

```php app/src/Application/Attribute/TargetRRWorker.php
use Spiral\Boot\Attribute\BootloadConfig;

/**
 * Load bootloader only for specific RoadRunner workers
 */
class TargetRRWorker extends BootloadConfig 
{
    public function __construct(array $modes)
    {
        parent::__construct(allowEnv: ['RR_MODE' => $modes]);
    }
}
```

Использование в загрузчике:

```php
#[TargetRRWorker(modes: ['http', 'grpc'])]
final class ApiBootloader extends Bootloader {}
```

Или в ядре:

```php
public function defineBootloaders(): array
{
    return [
        HttpBootloader::class => new TargetRRWorker(['http']),
        GrpcBootloader::class => new TargetRRWorker(['grpc']),
        TemporalBootloader::class => new TargetRRWorker(['temporal']),
    ];
}
```

**Когда использовать:** в микросервисах и многопротокольных приложениях, где разным воркерам требуются разные
загрузчики.

### Сопоставление окружения

И `allowEnv`, и `denyEnv` поддерживают несколько значений:

```php
#[BootloadConfig(
    // Load if APP_ENV is local OR development
    allowEnv: ['APP_ENV' => ['local', 'development']],
    
    // Don't load if any of these are true
    denyEnv: [
        'TESTING' => [true, 1, 'true', 'yes'],
        'CI' => true
    ]
)]
final class DevToolsBootloader extends Bootloader {}
```

**Правила работы:**

- `allowEnv`: должно совпасть хотя бы одно условие — логика ИЛИ;
- `denyEnv`: при совпадении любого условия загрузчик пропускается — логика ИЛИ;
- если заданы оба параметра, сначала проверяется `allowEnv`, затем `denyEnv`.

## Методы инициализации

Загрузчики предоставляют несколько способов выполнения инициализирующего кода. Можно использовать традиционные методы,
методы с атрибутами или сочетать оба подхода.

### Традиционные методы

#### init()

Метод `init()` выполняется первым, до вызова метода `boot()` любого загрузчика.

**Параметры:**

- поддерживается внедрение зависимостей: любой сервис можно запросить через параметры метода.

**Когда использовать:**

- для установки значений конфигурации по умолчанию до их чтения другими загрузчиками;
- для регистрации связей контейнера, которые не зависят от конфигурации;
- для инициализации сервисов, необходимых другим загрузчикам.

```php
final class GithubClientBootloader extends Bootloader
{
    public function __construct(
        private readonly ConfiguratorInterface $config
    ) {}

    public function init(EnvironmentInterface $env): void 
    {
        // Set defaults before configuration is accessed
        $this->config->setDefaults(GithubConfig::CONFIG, [
            'access_token' => $env->get('GITHUB_ACCESS_TOKEN'),
            'secret' => $env->get('GITHUB_SECRET'),
            'timeout' => 30,
        ]);
    }
}
```

> **Смотрите также**
> [Конфигурация](../start/configuration.md) • [Ядро](../framework/kernel.md) •
> [Объекты конфигурации](../framework/config.md)

#### boot()

Метод `boot()` выполняется после завершения всех методов `init()` во всех загрузчиках.

**Параметры:**

- поддерживается внедрение зависимостей через параметры метода;
- можно получать скомпилированные объекты конфигурации.

**Когда использовать:**

- для настройки сервисов на основе окончательной конфигурации;
- для регистрации маршрутов, middleware и слушателей событий;
- для настройки интеграции между компонентами.

```php
final class GithubClientBootloader extends Bootloader
{
    public function boot(
        GithubConfig $config,
        HttpBootloader $http
    ): void {
        // Configuration is now compiled and ready
        $client = new GithubClient($config->getAccessToken());
        
        // Other bootloaders are initialized
        $http->addMiddleware(GithubAuthMiddleware::class);
    }
}
```

### Методы с атрибутами

Методы с атрибутами позволяют точнее управлять порядком выполнения и объявлять несколько методов инициализации и
загрузки в одном загрузчике.

#### #[InitMethod]

Методы с атрибутом `#[InitMethod]` выполняются на этапе инициализации.

**Параметры:**

| Параметр   | Тип   | По умолчанию | Описание                                       |
|------------|-------|---------------|------------------------------------------------|
| `priority` | `int` | `0`           | Приоритет выполнения: больший выполняется раньше |

**Возможности:**

- поддержка внедрения зависимостей;
- несколько методов `#[InitMethod]` в одном загрузчике;
- порядок выполнения: приоритет 10 → 5 → 0 → -5 → -10.

**Когда использовать:**

- для разделения логики инициализации на небольшие специализированные методы;
- для точного управления порядком инициализации разных загрузчиков;
- для регистрации групп связанных связей контейнера.

```php
use Spiral\Boot\Attribute\InitMethod;

final class DatabaseBootloader extends Bootloader
{
    // Critical: runs first
    #[InitMethod(priority: 10)]
    public function registerDrivers(DatabaseManager $manager): void
    {
        $manager->addDriver('mysql', MySQLDriver::class);
        $manager->addDriver('postgres', PostgresDriver::class);
    }
    
    // Normal: runs after high priority
    #[InitMethod]
    public function registerConnections(Container $container): void
    {
        $container->bindSingleton(
            ConnectionInterface::class,
            DefaultConnection::class
        );
    }
    
    // Low priority: runs last
    #[InitMethod(priority: -10)]
    public function registerExtensions(): void
    {
        // Optional extensions that depend on core setup
    }
}
```

#### #[BootMethod]

Методы с атрибутом `#[BootMethod]` выполняются на этапе загрузки после завершения всей инициализации.

**Параметры:**

| Параметр   | Тип   | По умолчанию | Описание                                       |
|------------|-------|---------------|------------------------------------------------|
| `priority` | `int` | `0`           | Приоритет выполнения: больший выполняется раньше |

**Возможности:**

- поддержка внедрения зависимостей;
- несколько методов `#[BootMethod]` в одном загрузчике;
- доступ к полностью настроенным сервисам и скомпилированной конфигурации.

**Когда использовать:**

- для настройки маршрутов, middleware и слушателей событий;
- для реализации сквозной функциональности;
- для регистрации сервисов уровня приложения.

```php
use Spiral\Boot\Attribute\BootMethod;

final class ApplicationBootloader extends Bootloader
{
    // Critical services first
    #[BootMethod(priority: 10)]
    public function configureErrorHandling(ErrorHandler $handler): void
    {
        $handler->addRenderer(new JsonErrorRenderer());
    }
    
    // Standard configuration
    #[BootMethod]
    public function configureRoutes(RouterInterface $router): void
    {
        $router->addRoute('home', new Route('/', HomeController::class));
    }
    
    // Non-critical features last
    #[BootMethod(priority: -10)]
    public function registerEventListeners(
        EventDispatcherInterface $dispatcher
    ): void {
        $dispatcher->addListener(
            ApplicationStarted::class, 
            fn() => $this->onStart()
        );
    }
}
```

### Порядок выполнения

Полная последовательность выполнения:

```
1. #[InitMethod(priority: 10)]  ← Highest priority init methods
2. #[InitMethod(priority: 0)]   ← Default priority init methods  
3. #[InitMethod(priority: -10)] ← Lowest priority init methods
4. init()                       ← Traditional init method
5. #[BootMethod(priority: 10)]  ← Highest priority boot methods
6. #[BootMethod(priority: 0)]   ← Default priority boot methods
7. #[BootMethod(priority: -10)] ← Lowest priority boot methods
8. boot()                       ← Traditional boot method
```

**Этот порядок применяется ко всем загрузчикам:**

- все методы `#[InitMethod(priority: 10)]` выполняются раньше любых `#[InitMethod(priority: 0)]`;
- все методы `init()` выполняются до любого `#[BootMethod]`;
- традиционные методы и методы с атрибутами можно сочетать в одном загрузчике.

## Настройка контейнера

Загрузчики предоставляют несколько способов настройки контейнера зависимостей. Выберите вариант, подходящий конкретной
задаче.

### Прямая настройка

Для максимальной гибкости используйте `BinderInterface` напрямую:

```php
use Spiral\Core\BinderInterface;

final class GithubClientBootloader extends Bootloader
{
    public function boot(BinderInterface $binder, GithubConfig $config): void 
    {
        // Singleton - created once, reused
        $binder->bindSingleton(
            ClientInterface::class, 
            static fn(GithubConfig $config) => new Client(
                $config->getAccessToken(),
                $config->getSecret(),
            )
        );
        
        // Factory - created each time
        $binder->bind(
            RequestInterface::class,
            static fn() => new Request()
        );
    }
}
```

> **Смотрите также**
> [Контейнер и фабрики](../container/overview.md)

### Декларативные связи

Связи можно объявить в структурированной декларативной форме.

#### С помощью методов

**Доступные методы:**

| Метод                | Возвращает | Описание                                      |
|----------------------|------------|-----------------------------------------------|
| `defineBindings()`   | `array`    | Фабричные связи: новый экземпляр при запросе  |
| `defineSingletons()` | `array`    | Singleton-связи: один переиспользуемый объект |

**Форматы связей:**

```php
return [
    // Simple class binding
    InterfaceA::class => ClassA::class,
    
    // Method callback
    InterfaceB::class => [self::class, 'createServiceB'],
    
    // Closure with dependencies
    InterfaceC::class => static fn(Config $config) => new ServiceC($config),
];
```

**Пример:**

```php
final class ServicesBootloader extends Bootloader
{
    public function defineBindings(): array
    {
        return [
            // New instance each resolution
            RequestInterface::class => Request::class,
            TokenGeneratorInterface::class => [self::class, 'createTokenGenerator'],
        ];
    }

    public function defineSingletons(): array
    {
        return [
            // Shared instance
            CacheInterface::class => RedisCache::class,
            
            // Lazy initialization with dependencies
            LoggerInterface::class => static fn(Config $config) => 
                new Logger($config->get('logging.channel')),
        ];
    }
    
    private function createTokenGenerator(): TokenGeneratorInterface
    {
        return new TokenGenerator(hash_algo: 'sha256', length: 32);
    }
}
```

**Когда использовать:**

- для чистых и легко просматриваемых объявлений связей;
- для группировки связанных зависимостей;
- для простых связей класс-класс и класс-фабрика.

#### С помощью атрибутов

Атрибуты PHP позволяют объявлять типобезопасные и самодокументируемые связи с дополнительными возможностями: псевдонимами
и областями.

##### #[SingletonMethod]

Создаёт singleton-связь: метод вызывается один раз, а результат кешируется и переиспользуется.

**Параметры:**

| Параметр                | Тип            | По умолчанию | Описание                                                   |
|-------------------------|----------------|---------------|------------------------------------------------------------|
| `alias`                 | `string\|null` | `null`        | Связать с этим псевдонимом вместо возвращаемого типа       |
| `aliasesFromReturnType` | `bool`         | `false`       | При заданном псевдониме также связать с возвращаемым типом |

**Когда использовать:**

- для сервисов, хранящих состояние;
- для дорогих в создании объектов: подключений к БД, HTTP-клиентов;
- для общих ресурсов: кеша, диспетчера событий.

```php
use Spiral\Boot\Attribute\SingletonMethod;

final class ServicesBootloader extends Bootloader
{
    // Binds to return type: HttpClientInterface
    #[SingletonMethod]
    public function createHttpClient(GithubConfig $config): HttpClientInterface
    {
        return new Client(
            $config->getAccessToken(),
            $config->getSecret(),
        );
    }
    
    // Binds to DbFactory, NOT DatabaseFactory
    #[SingletonMethod(alias: DbFactory::class)]
    public function createDatabaseFactory(): DatabaseFactory
    {
        return new DatabaseFactory();
    }
    
    // Binds to BOTH LogManagerInterface AND LogManager
    #[SingletonMethod(
        alias: LogManagerInterface::class, 
        aliasesFromReturnType: true
    )]
    public function createLogManager(): LogManager
    {
        return new LogManager();
    }
}
```

##### #[BindMethod]

Создаёт фабричную связь: метод вызывается при каждом разрешении зависимости и каждый раз возвращает новый экземпляр.

**Параметры:**

| Параметр                | Тип            | По умолчанию | Описание                                                   |
|-------------------------|----------------|---------------|------------------------------------------------------------|
| `alias`                 | `string\|null` | `null`        | Связать с этим псевдонимом вместо возвращаемого типа       |
| `aliasesFromReturnType` | `bool`         | `false`       | При заданном псевдониме также связать с возвращаемым типом |

**Когда использовать:**

- для сервисов без состояния;
- для объектов области запроса;
- для объектов, которые нельзя разделять между запросами.

```php
use Spiral\Boot\Attribute\BindMethod;

final class ServicesBootloader extends Bootloader
{
    // New instance each time
    #[BindMethod]
    public function createHttpClient(): HttpClientInterface
    {
        return new HttpClient();
    }
    
    // New request factory each time
    #[BindMethod(alias: RequestFactory::class)]
    public function createRequestFactory(): RequestFactoryInterface
    {
        return new RequestFactory();
    }
}
```

##### #[InjectorMethod]

Создаёт пользовательский инжектор, управляющий разрешением типа. Сам метод становится функцией разрешения.

**Параметры:**

| Параметр | Тип      | Обязателен | Описание              |
|----------|----------|------------|-----------------------|
| `alias`  | `string` | да         | Внедряемый тип        |

**Когда использовать:**

- для типов, создание которых зависит от контекста;
- для сервисов, принимающих параметры при создании;
- для сложной логики инициализации, меняющейся при каждом разрешении.

```php
use Spiral\Boot\Attribute\InjectorMethod;

final class LoggingBootloader extends Bootloader
{
    // Each logger resolution can specify a different channel
    #[InjectorMethod(LoggerInterface::class)]
    public function createLogger(string $channel = 'default'): LoggerInterface
    {
        return new Logger($channel);
    }
    
    // Connection name can be provided at resolution time
    #[InjectorMethod(ConnectionInterface::class)]
    public function createConnection(?string $name = null): ConnectionInterface
    {
        return $name === null
            ? new DefaultConnection()
            : $this->connectionPool->get($name);
    }
}

// Later, in another class:
class UserRepository
{
    public function __construct(
        // Will call createConnection(null) -> DefaultConnection
        ConnectionInterface $connection,
        
        // Will call createLogger('user') -> Logger with 'user' channel
        #[LogChannel('user')] LoggerInterface $logger
    ) {}
}
```

##### #[BindAlias]

Добавляет методу дополнительные псевдонимы связей. Атрибут можно применять несколько раз.

**Параметры:**

| Параметр      | Тип        | Обязателен | Описание                                       |
|---------------|------------|------------|------------------------------------------------|
| `...$aliases` | `string[]` | да         | Дополнительные имена классов и интерфейсов     |

**Когда использовать:**

- для связывания одной реализации с несколькими интерфейсами;
- для поддержки устаревших интерфейсов;
- для нескольких способов разрешения одного сервиса.

```php
use Spiral\Boot\Attribute\{SingletonMethod, BindAlias};

final class LoggingBootloader extends Bootloader
{
    #[SingletonMethod]
    #[BindAlias(LoggerInterface::class, PsrLoggerInterface::class)]
    #[BindAlias(MonologLoggerInterface::class)]
    public function createLogger(): Logger
    {
        return new Logger();
    }
}
```

**Будут созданы связи со всеми перечисленными типами:**

- `Logger` — возвращаемый тип;
- `LoggerInterface`;
- `PsrLoggerInterface`;
- `MonologLoggerInterface`.

При внедрении любого из этих типов будет передан один и тот же экземпляр `Logger`.

##### #[BindScope]

Связывает сервис с определённой областью контейнера. Атрибут можно применить несколько раз для нескольких областей.

**Параметры:**

| Параметр | Тип                    | Обязателен | Описание                       |
|----------|------------------------|------------|--------------------------------|
| `scope`  | `string\|\BackedEnum` | да         | Имя области или перечисление   |

**Когда использовать:**

- для HTTP-сервисов: запроса, ответа, сессии;
- для консольных сервисов: ввода и вывода;
- для сервисов конкретного воркера;
- для связей, зависящих от окружения.

```php
use Spiral\Boot\Attribute\{SingletonMethod, BindScope};

final class ServicesBootloader extends Bootloader
{
    // Only available in 'http' scope
    #[SingletonMethod]
    #[BindScope('http')]
    public function createHttpClient(): HttpClientInterface
    {
        return new HttpClient();
    }
    
    // Only in 'console' scope
    #[SingletonMethod]
    #[BindScope('console')]
    public function createConsoleOutput(): OutputInterface
    {
        return new ConsoleOutput();
    }
    
    // Available in BOTH 'http' and 'grpc' scopes
    #[SingletonMethod]
    #[BindScope('http')]
    #[BindScope('grpc')]
    public function createSharedService(): SharedServiceInterface
    {
        return new SharedService();
    }
}
```

**Как работают области:**

- связи без `#[BindScope]` глобальны и доступны везде;
- связь с областью доступна только при выполнении в этой области;
- попытка разрешить связь за пределами её области приводит к исключению.

##### Сочетание атрибутов

Атрибуты можно сочетать для сложных сценариев связывания:

```php
use Spiral\Boot\Attribute\{SingletonMethod, BindAlias, BindScope};

final class LoggingBootloader extends Bootloader
{
    // Singleton logger with multiple aliases, only in HTTP scope
    #[SingletonMethod]
    #[BindAlias(LoggerInterface::class, PsrLoggerInterface::class)]
    #[BindScope('http')]
    public function createHttpLogger(): Logger
    {
        return new Logger('http');
    }
    
    // Different logger for console scope
    #[SingletonMethod]
    #[BindAlias(LoggerInterface::class, PsrLoggerInterface::class)]
    #[BindScope('console')]
    public function createConsoleLogger(): Logger
    {
        return new Logger('console');
    }
}
```

**Результат:**

- HTTP-запросы получают HTTP-логгер через `LoggerInterface` или `PsrLoggerInterface`;
- консольные команды получают консольный логгер через те же интерфейсы;
- оба логгера являются singleton-объектами в своих областях;
- за пределами соответствующих областей разрешить их нельзя.

## Зависимости загрузчиков

Можно гарантировать, что другие загрузчики будут загружены и инициализированы раньше текущего. Каждая зависимость
загружается только один раз, даже если она требуется нескольким загрузчикам.

### Способ 1: внедрение параметром

Запросите загрузчик непосредственно в методе `init()` или `boot()`:

```php
use Spiral\Bootloader\Http\HttpBootloader;

class ApiBootloader extends Bootloader 
{
    public function boot(HttpBootloader $http): void
    {
        // HttpBootloader is guaranteed to be initialized
        $http->addMiddleware(ApiAuthMiddleware::class);
        $http->addMiddleware(RateLimitMiddleware::class);
    }
}
```

**Когда использовать:** если необходимо обращаться к методам или свойствам зависимости.

### Способ 2: метод defineDependencies

Объявите зависимости, не обращаясь к ним напрямую:

```php
class ApiBootloader extends Bootloader 
{
    public function defineDependencies(): void
    {
        return [
            HttpBootloader::class,
            CorsBootloader::class,
            AuthBootloader::class,
        ];
    }
    
    public function boot(): void
    {
        // All dependencies are initialized before this runs
        // Use when you need dependencies loaded but don't interact with them directly
    }
}
```

**Когда использовать:** когда зависимости должны быть загружены ради побочных эффектов, но вызывать их методы не
требуется.

### Как выбрать способ

```php
// Use injection when you need to call methods
public function boot(HttpBootloader $http): void
{
    $http->addMiddleware(MyMiddleware::class);  // Interacting with dependency
}

// Use defineDependencies when you just need initialization
public function defineDependencies(): void
{
    return [ 
        DatabaseBootloader::class,  // Just needs to set up database
        CacheBootloader::class,     // Just needs to configure cache
    ];
}
```

## Динамическая загрузка

Загрузчики можно подключать во время выполнения в зависимости от состояния приложения.

**Параметры метода `bootload()`:**

| Параметр            | Тип     | Описание                                         |
|---------------------|---------|--------------------------------------------------|
| `classes`           | `array` | Массив имён классов загружаемых загрузчиков      |
| `bootingCallbacks`  | `array` | Обратные вызовы до запуска загрузчиков            |
| `bootedCallbacks`   | `array` | Обратные вызовы после запуска загрузчиков         |

**Когда использовать:**

- для флагов возможностей;
- для загрузчиков, зависящих от окружения;
- для систем плагинов;
- для условного подключения функций.

```php
use Spiral\Boot\BootloadManagerInterface;

class AppBootloader extends Bootloader
{
    public function boot(
        BootloadManagerInterface $bootloadManager, 
        EnvironmentInterface $env
    ): void {
        // Load debug tools only when DEBUG is enabled
        if ($env->get('DEBUG')) {
            $bootloadManager->bootload([
                DebugBootloader::class,
                ProfilerBootloader::class,
            ]);
        }
        
        // Load feature bootloaders based on feature flags
        if ($this->isFeatureEnabled('new-api')) {
            $bootloadManager->bootload([
                NewApiBootloader::class,
            ]);
        }
        
        // Load different bootloaders per environment
        match($env->get('APP_ENV')) {
            'production' => $bootloadManager->bootload([
                ProductionCacheBootloader::class,
                ProductionLoggingBootloader::class,
            ]),
            'local' => $bootloadManager->bootload([
                DevToolsBootloader::class,
                LocalCacheBootloader::class,
            ]),
            default => null,
        };
    }
}
```

> **Предупреждение**
> Динамическая загрузка происходит на этапе `boot`, поэтому динамически подключаемые загрузчики не могут содержать
> методы `init()` или атрибуты `#[InitMethod]`: этап инициализации к этому моменту уже завершён.

<hr>

## Что дальше?

* [HTTP — Перехватчики](../http/interceptors.md);
* [Генерация кода](../basics/scaffolding.md);
* [Контейнер — Атрибуты](../container/attributes.md).
