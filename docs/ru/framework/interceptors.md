# Фреймворк — Перехватчики

Одна из ключевых возможностей Spiral — поддержка перехватчиков. Они позволяют добавлять функциональность, не изменяя
основной код приложения, благодаря чему кодовая база остаётся модульной и удобной для сопровождения.

**Преимущества перехватчиков:**

- **Разделение ответственности.** Разные части приложения остаются изолированными и упорядоченными. Например,
  аутентификацию можно реализовать в одном перехватчике вместо добавления одинакового кода во все защищённые участки.
- **Повторное использование.** Один раз написанный код можно применять в разных частях приложения, уменьшая дублирование.
- **Модульность.** Перехватчики можно добавлять, удалять и заменять, не затрагивая остальное приложение.
- **Производительность.** Через перехватчики можно кешировать ответы, уменьшать число запросов к базе данных и применять
  другие оптимизации.
- **Простота использования.** Добавление перехватчиков в приложение не требует сложной настройки.

Перехватчики поддерживаются различными компонентами:

- [HTTP](../http/interceptors.md);
- [события](../advanced/events.md#interceptors);
- [gRPC](../grpc/interceptors.md);
- [WebSocket](../websockets/interceptors.md);
- [очереди](../queue/interceptors.md);
- [Temporal](../temporal/interceptors.md).

## Интерфейс перехватчика

Перехватчики реализуют `Spiral\Interceptors\InterceptorInterface`:

```php
namespace Spiral\Interceptors;

use Spiral\Interceptors\Context\CallContextInterface;

interface InterceptorInterface
{
    public function intercept(CallContextInterface $context, HandlerInterface $handler): mixed;
}
```

Интерфейс позволяет перехватывать обращение к любой цели: методу, функции или пользовательскому обработчику.

## Контекст вызова

`CallContextInterface` содержит все сведения о перехватываемом вызове:

- **Target** — описание цели вызова: метода, функции, замыкания и так далее;
- **Arguments** — список аргументов вызова;
- **Attributes** — дополнительный контекст для передачи данных между перехватчиками.

```php
namespace Spiral\Interceptors\Context;

interface CallContextInterface extends AttributedInterface
{
    public function getTarget(): TargetInterface;
    public function getArguments(): array;
    public function withTarget(TargetInterface $target): static;
    public function withArguments(array $arguments): static;
    
    // Methods from AttributedInterface:
    public function getAttributes(): array;
    public function getAttribute(string $name, mixed $default = null): mixed;
    public function withAttribute(string $name, mixed $value): static;
    public function withoutAttribute(string $name): static;
}
```

> **Примечание**
> `CallContextInterface` неизменяем: методы `withTarget()` и `withArguments()` возвращают новый экземпляр с обновлёнными
> значениями.

## Интерфейс цели

`TargetInterface` описывает цель перехватываемого вызова. Это может быть метод, функция, замыкание или даже строковый путь
для RPC-обработчика либо конечной точки очереди сообщений.

```php
namespace Spiral\Interceptors\Context;

interface TargetInterface extends \Stringable
{
    public function getPath(): array;
    public function withPath(array $path, ?string $delimiter = null): static;
    public function getReflection(): ?\ReflectionFunctionAbstract;
    public function getObject(): ?object;
    public function getCallable(): callable|array|null;
}
```

### Создание целей

Статические фабричные методы класса `Target` упрощают создание целей разных типов:

```php
use Spiral\Interceptors\Context\Target;

// From a method reflection
$target = Target::fromReflectionMethod(new \ReflectionMethod(UserController::class, 'show'), UserController::class);

// From a function reflection
$target = Target::fromReflectionFunction(new \ReflectionFunction('array_map'));

// From a closure
$target = Target::fromClosure(fn() => 'Hello, World!');

// From a path string (for RPC endpoints or message queue handlers)
$target = Target::fromPathString('user.show');

// From a controller-action pair
$target = Target::fromPair(UserController::class, 'show');
```

## Создание перехватчика

Создадим простой перехватчик, регистрирующий время выполнения вызова:

```php
namespace App\Interceptor;

use Psr\Log\LoggerInterface;
use Spiral\Interceptors\Context\CallContextInterface;
use Spiral\Interceptors\HandlerInterface;
use Spiral\Interceptors\InterceptorInterface;

class ExecutionTimeInterceptor implements InterceptorInterface
{
    public function __construct(
        private readonly LoggerInterface $logger
    ) {
    }

    public function intercept(CallContextInterface $context, HandlerInterface $handler): mixed
    {
        $target = $context->getTarget();
        $startTime = \microtime(true);

        try {
            return $handler->handle($context);
        } finally {
            $executionTime = \microtime(true) - $startTime;

            $this->logger->debug(
                'Target executed',
                [
                    'target' => (string)$target,
                    'execution_time' => $executionTime,
                ]
            );
        }
    }
}
```

Этот перехватчик:

1. сохраняет время начала;
2. передаёт вызов следующему обработчику в цепочке;
3. после завершения обработчика вычисляет и записывает время выполнения.

## Построение конвейера перехватчиков

Для использования перехватчиков создайте конвейер через `PipelineBuilderInterface`:

```php
use Spiral\Interceptors\PipelineBuilder;
use Spiral\Interceptors\Context\CallContext;
use Spiral\Interceptors\Context\Target;
use Spiral\Interceptors\Handler\CallableHandler;
use App\Interceptor\ExecutionTimeInterceptor;
use App\Interceptor\AuthorizationInterceptor;

// Create interceptors
$interceptors = [
    new ExecutionTimeInterceptor($logger),
    new AuthorizationInterceptor($auth),
];

// Build the pipeline
$pipeline = (new PipelineBuilder())
    ->withInterceptors(...$interceptors)
    ->build(new CallableHandler());

// Create call context
$context = new CallContext(
    target: Target::fromPair(UserController::class, 'show'),
    arguments: [42],
    attributes: ['request' => $request]
);

// Execute the pipeline
$result = $pipeline->handle($context);
```

## Обработчики

Конвейер должен завершаться обработчиком, который вызывает цель. Spiral предоставляет несколько встроенных
обработчиков.

### CallableHandler

`CallableHandler` вызывает цель без дополнительной обработки:

```php
use Spiral\Interceptors\Handler\CallableHandler;

$handler = new CallableHandler();
```

### AutowireHandler

`AutowireHandler` разрешает отсутствующие аргументы через контейнер:

```php
use Spiral\Interceptors\Handler\AutowireHandler;

$handler = new AutowireHandler($container);
```

Это удобно для контроллеров, в которые зависимости сервисов должны внедряться автоматически.

## Расширенный пример

Более полный пример использования перехватчиков:

```php
namespace App\Controller;

use App\Interceptor\AuthorizationInterceptor;
use App\Interceptor\CacheInterceptor;
use App\Interceptor\ExecutionTimeInterceptor;
use App\User\UserService;
use Psr\Container\ContainerInterface;
use Spiral\Core\Attribute\Proxy;
use Spiral\Core\CompatiblePipelineBuilder;
use Spiral\Interceptors\Context\CallContext;
use Spiral\Interceptors\Context\Target;
use Spiral\Interceptors\Handler\AutowireHandler;

class UserController
{
    private $pipeline;

    public function __construct(
        private readonly UserService $userService,
        #[Proxy] ContainerInterface $container
    ) {
        // Build the pipeline with interceptors
        $this->pipeline = (new CompatiblePipelineBuilder())
            ->withInterceptors(
                new ExecutionTimeInterceptor($container->get(LoggerInterface::class)),
                new AuthorizationInterceptor($container->get(AuthInterface::class)),
                new CacheInterceptor($container->get(CacheInterface::class))
            )
            ->build(new AutowireHandler($container));
    }

    public function show(int $id)
    {
        // Create a context for the target method
        $context = new CallContext(
            target: Target::fromReflectionMethod(
                new \ReflectionMethod($this->userService, 'findUser'),
                $this->userService
            ),
            arguments: ['id' => $id]
        );

        // Execute the pipeline
        return $this->pipeline->handle($context);
    }
}
```

В этом примере:

1. создаётся конвейер с регистрацией времени выполнения, проверкой авторизации и кешированием результата;
2. `AutowireHandler` разрешает отсутствующие аргументы через контейнер;
3. метод контроллера создаёт контекст, целью которого является метод `findUser` сервиса `UserService`;
4. конвейер выполняет все перехватчики, а затем вызывает целевой метод.

> **Примечание**
> Подробнее об областях контейнера и прокси-объектах читайте в разделе
> [Области IoC](../container/scopes.md).

## Сравнение с устаревшими перехватчиками

> **Примечание**
> Старая реализация перехватчиков на основе `spiral/hmvc` больше не рекомендуется.
> Прежняя документация доступна по адресу
> [https://spiral.dev/docs/framework-interceptors/3.13](https://spiral.dev/docs/framework-interceptors/3.13).

В Spiral 3.14.0 появилась новая реализация перехватчиков в пакете `spiral/interceptors`. Ниже показаны отличия от
устаревшей реализации.

:::: tabs

::: tab Устаревшие перехватчики
```php
namespace App\Interceptor;

use Psr\SimpleCache\CacheInterface;
use Spiral\Core\CoreInterface;
use Spiral\Core\CoreInterceptorInterface;

class CacheInterceptor implements CoreInterceptorInterface
{
    public function __construct(
        private readonly CacheInterface $cache,
        private readonly int $ttl = 3600,
    ) {}

    public function process(string $controller, string $action, array $parameters, CoreInterface $core): mixed
    {
        // Step 1: Generate a cache key based on controller, action, and parameters
        $cacheKey = $this->generateCacheKey($controller, $action, $parameters);

        // Step 2: Check if the result is already cached
        if ($this->cache->has($cacheKey)) {
            // Return cached result if available
            return $this->cache->get($cacheKey);
        }

        // Step 3: Execute the controller action if no cached result
        $result = $core->callAction($controller, $action, $parameters);

        // Step 4: Cache the result for future requests
        if ($this->isCacheable($result)) {
            $this->cache->set($cacheKey, $result, $this->ttl);
        }

        return $result;
    }

    private function generateCacheKey(string $controller, string $action, array $parameters): string
    {
        // Create a deterministic cache key from controller, action, and parameters
        return \md5($controller . '::' . $action . '::' . \serialize($parameters));
    }

    private function isCacheable(mixed $result): bool
    {
        // Only cache serializable results
        return !\is_resource($result) && (
                \is_scalar($result) ||
                \is_array($result) ||
                $result instanceof \Serializable ||
                $result instanceof \stdClass
            );
    }
}
```
:::

::: tab Новые перехватчики
```php
namespace App\Interceptor;

use Psr\SimpleCache\CacheInterface;
use Spiral\Interceptors\Context\CallContextInterface;
use Spiral\Interceptors\Context\TargetInterface;
use Spiral\Interceptors\HandlerInterface;
use Spiral\Interceptors\InterceptorInterface;

final class CacheInterceptor implements InterceptorInterface
{
    public function __construct(
        private readonly CacheInterface $cache,
        private readonly int $ttl = 3600,
    ) {}

    public function intercept(CallContextInterface $context, HandlerInterface $handler): mixed
    {
        // Step 1: Generate a cache key using target path and arguments
        $cacheKey = $this->generateCacheKey($context->getTarget(), $context->getArguments());

        // Step 2: Check if the result is already cached
        if ($this->cache->has($cacheKey)) {
            // Return cached result if available
            return $this->cache->get($cacheKey);
        }

        // Step 3: Execute the target if no cached result
        $result = $handler->handle($context);

        // Step 4: Cache the result for future requests
        if ($this->isCacheable($result)) {
            $this->cache->set($cacheKey, $result, $this->ttl);
        }

        return $result;
    }

    private function generateCacheKey(TargetInterface $target, array $args): string
    {
        // Create a deterministic cache key from target string and arguments
        return \md5((string) $target . '::' . serialize($args));
    }

    private function isCacheable(mixed $result): bool
    {
        // Only cache serializable results
        return !\is_resource($result) && (
                \is_scalar($result) ||
                \is_array($result) ||
                $result instanceof \Serializable ||
                $result instanceof \stdClass
            );
    }
}
```
:::

::::

### Изменения интерфейса

**Устаревший интерфейс:**

```php
interface CoreInterceptorInterface
{
    public function process(string $controller, string $action, array $parameters, CoreInterface $core): mixed;
}
```

**Новый интерфейс:**

```php
interface InterceptorInterface
{
    public function intercept(CallContextInterface $context, HandlerInterface $handler): mixed;
}
```

Основные отличия:

- параметры `$controller` и `$action` заменены более гибким понятием `Target`;
- массив `$parameters` теперь является частью `CallContext`;
- вместо `CoreInterface` используется более гибкий `HandlerInterface`.

### Совместимость

Spiral 3.x поддерживает и устаревшие (`CoreInterceptorInterface`), и новые (`InterceptorInterface`) перехватчики. Однако
устаревшая реализация объявлена deprecated и будет исключена в Spiral 4.x.

Для совместного использования обеих реализаций применяйте `CompatiblePipelineBuilder`:

```php
use Spiral\Core\CompatiblePipelineBuilder;
use Spiral\Core\CoreInterceptorInterface; // Legacy interceptor
use Spiral\Interceptors\InterceptorInterface; // New interceptor

$pipeline = (new CompatiblePipelineBuilder())
    ->withInterceptors(
        new LegacyStyleInterceptor(), // Implements CoreInterceptorInterface
        new NewStyleInterceptor()     // Implements InterceptorInterface
    )
    ->build(new CallableHandler());
```

### Рекомендации по миграции

При переходе со старой реализации:

- замените `CoreInterceptorInterface` на `InterceptorInterface`;
- используйте `Target` вместо параметров `$controller` и `$action`;
- перенесите `$parameters` в `CallContext`;
- замените `$core->callAction()` на `$handler->handle()`.

## События

| Событие                                      | Описание                                |
|----------------------------------------------|-----------------------------------------|
| Spiral\Interceptors\Event\InterceptorCalling | Вызывается перед обращением к перехватчику |

> **Предупреждение**
> Событие `Spiral\Core\Event\InterceptorCalling` отправляется только устаревшим
> `\Spiral\Core\InterceptorPipeline`. Новая реализация фреймворка его не использует.

> **Примечание**
> Подробнее о диспетчеризации событий читайте в разделе [События](../advanced/events.md).
