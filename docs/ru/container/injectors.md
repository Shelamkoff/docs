# Контейнер — Инжекторы

Spiral предоставляет мощную систему инжекторов, позволяющую управлять созданием реализаций интерфейсов и потомков
абстрактных классов. Зависимости можно динамически разрешать с учётом контекста, что особенно полезно в сложных
сценариях внедрения.

## Что такое инжекторы?

Инжекторы — специальные компоненты, перехватывающие процесс разрешения определённых интерфейсов или классов контейнером.
Вместо использования только автоматического связывания они предоставляют точный контроль над созданием и настройкой
экземпляра с учётом контекста вызова.

**Основные преимущества:**

- **Контекстное разрешение:** выбор разных реализаций по имени параметра, типу или атрибутам.
- **Гибкость:** динамический выбор реализации во время выполнения без изменения клиентского кода.
- **Слабая связанность:** клиентский код не зависит от конкретных реализаций.
- **Управление жизненным циклом:** логика создания, инициализации и настройки сосредоточена в одном месте.

## Создание инжекторов классов

### Базовая реализация

Инжектор должен реализовывать `Spiral\Core\Container\InjectorInterface`, предоставляющий метод `createInjection`.
Он вызывается каждый раз, когда контейнер должен разрешить определённый класс или интерфейс.

Создадим простой инжектор кеша, возвращающий разные реализации в зависимости от контекста:

```php app/src/Application/Bootloader/CacheBootloader.php
namespace App\Application\Bootloader;

use Psr\Container\ContainerInterface;
use Psr\SimpleCache\CacheInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Core\BinderInterface;
use Spiral\Core\Container\InjectorInterface;

final class CacheBootloader extends Bootloader implements InjectorInterface
{
    public function __construct(
        private readonly ContainerInterface $container,
    ) {
    }

    public function boot(BinderInterface $binder): void
    {
        // Register this bootloader as the injector for CacheInterface
        $binder->bindInjector(CacheInterface::class, self::class);
    }

    public function createInjection(
        \ReflectionClass $class,
        \Stringable|string|null $context = null
    ): CacheInterface {
        return match ($context) {
            'redis' => new RedisCache(...),
            'memcached' => new MemcachedCache(...),
            default => new ArrayCache(...),
        };
    }
}
```

> **Примечание**
> Не забудьте активировать загрузчик в приложении.

### Параметры контекста

Метод `createInjection` получает два важных параметра:

| Параметр   | Тип                          | Описание                                                    |
|------------|------------------------------|-------------------------------------------------------------|
| `$class`   | `\ReflectionClass`           | Reflection-объект запрошенного класса или интерфейса        |
| `$context` | `\Stringable\|string\|null` | Сведения о месте, в котором запрашивается внедрение         |

Параметр `$context` может содержать разные виды данных:

- **String** — имя параметра конструктора или метода;
- **ReflectionParameter** — полный reflection-объект параметра для расширенных сценариев;
- **Stringable** — пользовательский объект контекста;
- **null** — контекст отсутствует.

### Строковый контекст

Самый простой вариант контекстного внедрения использует имена параметров:

```php app/src/Endpoint/Web/BlogController.php
namespace App\Endpoint\Web;

use Psr\SimpleCache\CacheInterface;

final class BlogController
{
    public function __construct(
        private readonly CacheInterface $redis,
        private readonly CacheInterface $memcached,
        private readonly CacheInterface $cache,
    ) {
        // $redis will be an instance of RedisCache
        // $memcached will be an instance of MemcachedCache  
        // $cache will be an instance of ArrayCache
    }
}
```

Инжектор получает имя параметра в `$context` и возвращает соответствующую реализацию.

## Расширенная обработка контекста

### Контекст ReflectionParameter

Начиная с версии `3.16.x`, инжекторы могут получать в качестве контекста объекты `ReflectionParameter`. Это позволяет
выбирать реализацию по атрибутам, типу и другим метаданным параметра.

Чтобы принимать `ReflectionParameter`, укажите его в сигнатуре `createInjection`:

```php
public function createInjection(
    \ReflectionClass $class,
    \ReflectionParameter|string|null $context = null
): object {
    // Now $context can be a ReflectionParameter
}
```

#### Внедрение на основе атрибутов

Этот подход особенно полезен совместно с атрибутами PHP:

```php app/src/Application/Attribute/DatabaseDriver.php
namespace App\Application\Attribute;

#[\Attribute(\Attribute::TARGET_PARAMETER)]
final readonly class DatabaseDriver
{
    public function __construct(
        public string $name,
    ) {
    }
}
```

Создадим инжектор, читающий атрибут параметра:

```php app/src/Application/Injector/DatabaseInjector.php
namespace App\Application\Injector;

use App\Application\Attribute\DatabaseDriver;
use Spiral\Core\Container\InjectorInterface;

final class DatabaseInjector implements InjectorInterface
{
    public function createInjection(
        \ReflectionClass $class,
        \ReflectionParameter|string|null $context = null
    ): object {
        // Extract attribute from parameter
        $driver = $context instanceof \ReflectionParameter
            ? $context->getAttributes(DatabaseDriver::class)[0]?->newInstance()?->name ?? 'mysql'
            : 'mysql';

        return match ($driver) {
            'sqlite' => new Sqlite(),
            'mysql' => new Mysql(),
            'postgres' => new PostgreSQL(),
            default => throw new \InvalidArgumentException("Unknown database driver: {$driver}"),
        };
    }
}
```

Зарегистрируем инжектор в загрузчике:

```php app/src/Application/Bootloader/DatabaseBootloader.php
namespace App\Application\Bootloader;

use App\Application\DatabaseInterface;
use App\Application\Injector\DatabaseInjector;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Core\BinderInterface;

final class DatabaseBootloader extends Bootloader
{
    public function boot(BinderInterface $binder): void
    {
        $binder->bindInjector(DatabaseInterface::class, DatabaseInjector::class);
    }
}
```

Теперь нужный драйвер базы данных можно указать атрибутом:

```php app/src/Application/Service/SomeService.php
namespace App\Application\Service;

use App\Application\Attribute\DatabaseDriver;
use App\Application\DatabaseInterface;

final class SomeService
{
    public function __construct(
        #[DatabaseDriver(name: 'mysql')]
        private readonly DatabaseInterface $mysql,

        #[DatabaseDriver(name: 'sqlite')]  
        private readonly DatabaseInterface $sqlite,
        
        #[DatabaseDriver(name: 'postgres')]
        private readonly DatabaseInterface $postgres,
    ) {
    }
}
```

### Контекст Stringable

Инжекторы поддерживают пользовательские объекты `Stringable`, позволяя передавать расширенные сведения о контексте:

```php
use Spiral\Core\FactoryInterface;

final class MyContextualClass implements \Stringable
{
    public function __construct(
        public readonly string $environment,
        public readonly array $options = [],
    ) {
    }

    public function __toString(): string
    {
        return $this->environment;
    }
}

// Using with FactoryInterface
$factory->make(
    CacheInterface::class, 
    context: new MyContextualClass('production', ['ttl' => 3600])
);
```

## Поддержка наследования классов

Инжекторы поддерживают наследование, поэтому реализацию можно выбирать по типу запрошенного класса.

> **Примечание**
> Сейчас наследование поддерживается только для классов, расширяющих базовый класс, но не для интерфейсов. В будущих
> версиях Spiral может появиться поддержка наследования интерфейсов.

### Пример с абстрактными классами

Объявим специализированные абстрактные классы:

```php
use Psr\SimpleCache\CacheInterface;

abstract class RedisCacheInterface implements CacheInterface
{
    // Redis-specific methods
}

abstract class MemcachedCacheInterface implements CacheInterface
{
    // Memcached-specific methods
}
```

Реализуем обработку наследования в инжекторе:

```php app/src/Application/Bootloader/CacheBootloader.php
public function createInjection(
    \ReflectionClass $class,
    \Stringable|string|null $context = null
): CacheInterface {
    // Check for specialized abstract classes first
    if ($class->isSubclassOf(RedisCacheInterface::class)) {
        return new RedisCache(...);
    }
    
    if ($class->isSubclassOf(MemcachedCacheInterface::class)) {
        return new MemcachedCache(...);
    }
    
    // Fall back to context-based resolution
    return match ($context) {
        'redis' => new RedisCache(...),
        'memcached' => new MemcachedCache(...),
        default => new ArrayCache(...),
    };
}
```

## Поддержка контекста в фабрике контейнера

Метод `make` интерфейса `FactoryInterface` принимает контекст `Stringable`, благодаря чему при разрешении зависимости
можно передавать пользовательские объекты контекста:

```php
use Spiral\Core\FactoryInterface;

final readonly class MyContext implements \Stringable
{
    public function __construct(
        public string $environment,
    ) {
    }
    
    public function __toString(): string
    {
        return $this->environment;
    }
}

final class MyService
{
    public function __construct(
        private readonly FactoryInterface $factory,
    ) {
    }
    
    public function createWithContext(): mixed
    {
        return $this->factory->make(
            SomeClass::class,
            context: new MyContext('production')
        );
    }
}
```

Инжектор получает этот контекст и использует его при выборе реализации:

```php
public function createInjection(
    \ReflectionClass $class,
    \Stringable|string|null $context = null
): object {
    if ($context instanceof MyContext) {
        // Use the environment from context
        return match ($context->environment) {
            'production' => new ProductionImplementation(),
            'staging' => new StagingImplementation(),
            default => new DevelopmentImplementation(),
        };
    }
    
    // Handle other context types
    return new DefaultImplementation();
}
```

## Инжекторы перечислений

Компонент `spiral/boot` позволяет разрешать перечисления с помощью интерфейса
`Spiral\Boot\Injector\InjectableEnumInterface`. При запросе перечисления контейнер вызывает указанный метод, определяет
текущее значение и внедряет необходимые зависимости этого метода.

### Преимущества внедряемых перечислений

- **Типобезопасность:** в методы и классы передаются значения правильного типа.
- **Динамическое разрешение:** значение определяется по текущему состоянию приложения.
- **Повторное использование:** одна логика внедрения применяется в разных частях приложения.
- **Читаемость:** назначение передаваемого значения явно выражено типом.
- **Централизованная конфигурация:** выбор значения находится в одном месте.

### Создание внедряемого перечисления

Атрибут `#[ProvideFrom]` указывает статический метод, возвращающий значение перечисления:

```php app/src/Application/Enum/AppEnvironment.php
namespace App\Application\Enum;

use Spiral\Boot\EnvironmentInterface;
use Spiral\Boot\Injector\ProvideFrom;
use Spiral\Boot\Injector\InjectableEnumInterface;

#[ProvideFrom(method: 'detect')]
enum AppEnvironment: string implements InjectableEnumInterface
{
    case Production = 'prod';
    case Stage = 'stage';
    case Testing = 'testing';
    case Local = 'local';

    public function isProduction(): bool
    {
        return $this === self::Production;
    }

    public function isTesting(): bool
    {
        return $this === self::Testing;
    }

    public function isLocal(): bool
    {
        return $this === self::Local;
    }

    public function isStage(): bool
    {
        return $this === self::Stage;
    }

    /**
     * Detect environment from configuration
     * Dependencies are automatically injected by the container
     */
    public static function detect(EnvironmentInterface $environment): self
    {
        $value = $environment->get('APP_ENV');

        return \is_string($value)
            ? (self::tryFrom($value) ?? self::Local)
            : self::Local;
    }
}
```

### Использование внедряемых перечислений

Контейнер автоматически передаёт перечисление с правильным значением:

```php app/src/Endpoint/Console/MigrateCommand.php
namespace App\Endpoint\Console;

use App\Application\Enum\AppEnvironment;
use Spiral\Console\Command;

final class MigrateCommand extends Command
{
    protected const NAME = 'migrate';

    public function perform(AppEnvironment $appEnv): int
    {
        if ($appEnv->isProduction()) {
            $this->error('Migrations are disabled in production');
            return self::FAILURE;
        }
        
        // Perform migration
        $this->info('Running migrations...');
        
        return self::SUCCESS;
    }
}
```

Перечисления также можно внедрять через конструктор:

```php app/src/Application/Service/ConfigService.php
namespace App\Application\Service;

use App\Application\Enum\AppEnvironment;

final readonly class ConfigService
{
    public function __construct(
        private AppEnvironment $environment,
    ) {
    }
    
    public function getCacheTTL(): int
    {
        return match ($this->environment) {
            AppEnvironment::Production, AppEnvironment::Stage => 3600,
            AppEnvironment::Testing => 60,
            AppEnvironment::Local => 0,
        };
    }
}
```

## Рекомендации

### Делайте инжекторы специализированными

Каждый инжектор должен обрабатывать один интерфейс или абстрактный класс. Не создавайте «мега-инжекторы», отвечающие за
несколько несвязанных типов.

```php
// ✅ Good: Focused injector
final class CacheInjector implements InjectorInterface
{
    public function createInjection(\ReflectionClass $class, $context = null): CacheInterface
    {
        // Handle only cache-related injection
    }
}

// ❌ Bad: Handles multiple unrelated types
final class MegaInjector implements InjectorInterface
{
    public function createInjection(\ReflectionClass $class, $context = null): object
    {
        if ($class->implementsInterface(CacheInterface::class)) { /* ... */ }
        if ($class->implementsInterface(LoggerInterface::class)) { /* ... */ }
        if ($class->implementsInterface(QueueInterface::class)) { /* ... */ }
        // Too many responsibilities
    }
}
```

### Предоставляйте разумное значение по умолчанию

Если контекст не соответствует известным вариантам, возвращайте реализацию по умолчанию:

```php
public function createInjection(\ReflectionClass $class, $context = null): CacheInterface
{
    return match ($context) {
        'redis' => new RedisCache(...),
        'memcached' => new MemcachedCache(...),
        // ✅ Sensible default prevents unexpected failures
        default => new ArrayCache(...),
    };
}
```

### Используйте типы

Всегда указывайте типы параметров и возвращаемого значения для поддержки IDE и типобезопасности:

```php
// ✅ Good: Explicit types
public function createInjection(
    \ReflectionClass $class,
    \ReflectionParameter|string|null $context = null
): CacheInterface {
    // ...
}

// ❌ Bad: Missing types
public function createInjection($class, $context = null) {
    // ...
}
```

### Документируйте ожидаемый контекст

Явно указывайте, какие значения контекста поддерживает инжектор:

```php
/**
 * Cache injector that resolves implementations based on parameter names.
 * 
 * Supported context values:
 * - 'redis': Returns RedisCache instance
 * - 'memcached': Returns MemcachedCache instance  
 * - 'file': Returns FileCache instance
 * - null or other: Returns ArrayCache instance (default)
 */
final class CacheInjector implements InjectorInterface
{
    public function createInjection(\ReflectionClass $class, $context = null): CacheInterface
    {
        // Implementation
    }
}
```

### Проверяйте контекст

При работе с `ReflectionParameter` проверяйте, что получены ожидаемые метаданные:

```php
public function createInjection(
    \ReflectionClass $class,
    \ReflectionParameter|string|null $context = null
): DatabaseInterface {
    if ($context instanceof \ReflectionParameter) {
        $attributes = $context->getAttributes(DatabaseDriver::class);
        
        if (empty($attributes)) {
            throw new \InvalidArgumentException(
                "Parameter {$context->getName()} must have DatabaseDriver attribute"
            );
        }
        
        $driver = $attributes[0]->newInstance()->name;
    } else {
        $driver = $context ?? 'mysql';
    }
    
    return $this->createDriver($driver);
}
```

## Решение проблем

### Инжектор не вызывается

Проверьте следующее:

1. инжектор зарегистрирован через `$binder->bindInjector()`;
2. содержащий его загрузчик активирован;
3. запрашивается именно тот интерфейс или класс, который обслуживает инжектор;
4. отсутствует обычная связь с более высоким приоритетом — связи разрешаются раньше инжекторов.

### Контекст всегда равен null

Проверьте следующее:

1. зависимость не запрашивается напрямую через `$container->get(CacheInterface::class)`, поскольку такой вызов не
   содержит контекста;
2. зависимость разрешается через внедрение в конструктор или метод, где доступно имя параметра;
3. при необходимости в `FactoryInterface::make()` передаётся параметр контекста.

### Несоответствие типов ReflectionParameter

При использовании контекста `ReflectionParameter` убедитесь, что:

1. сигнатура инжектора принимает `ReflectionParameter`: `\ReflectionParameter|string|null $context`;
2. тип контекста проверяется до обращения с ним как с `ReflectionParameter`;
3. для обратной совместимости обрабатываются строковый контекст и `null`.

## Смотрите также

- [Контейнер — Обзор](./overview.md) — основные возможности контейнера;
- [Контейнер — Конфигурация](./configuration.md) — настройка связей и зависимостей;
- [Контейнер — Атрибуты](./attributes.md) — управление зависимостями с помощью атрибутов;
- [Контейнер — Области IoC](./scopes.md) — разрешение зависимостей в областях.
