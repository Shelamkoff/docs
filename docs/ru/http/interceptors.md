# HTTP — Перехватчики

В Spiral перехватчики позволяют выполнять код до или после действия контроллера и изменять аргументы или результат
вызова. В отличие от middleware, они работают на уровне приложения после маршрутизации запроса и имеют доступ к цели
вызова, её аргументам и дополнительному контексту.

Перехватчики подходят для:

- авторизации и проверки разрешений;
- журналирования действий контроллера;
- кеширования результатов;
- преобразования входных и выходных данных;
- сбора метрик;
- реализации транзакций и другой сквозной логики.

> **Смотрите также**
> Общие принципы работы новой системы перехватчиков описаны в разделе
> [Фреймворк — Перехватчики](../framework/interceptors.md).

## Создание перехватчика

Перехватчик должен реализовывать `Spiral\Interceptors\InterceptorInterface`:

```php app/src/Endpoint/Web/Interceptor/LogInterceptor.php
namespace App\Endpoint\Web\Interceptor;

use Psr\Log\LoggerInterface;
use Spiral\Interceptors\Context\CallContextInterface;
use Spiral\Interceptors\HandlerInterface;
use Spiral\Interceptors\InterceptorInterface;

final readonly class LogInterceptor implements InterceptorInterface
{
    public function __construct(
        private LoggerInterface $logger,
    ) {
    }

    public function intercept(
        CallContextInterface $context,
        HandlerInterface $handler
    ): mixed {
        $this->logger->info('Controller action started', [
            'target' => (string)$context->getTarget(),
            'arguments' => $context->getArguments(),
        ]);

        try {
            return $handler->handle($context);
        } finally {
            $this->logger->info('Controller action finished', [
                'target' => (string)$context->getTarget(),
            ]);
        }
    }
}
```

Вызов `$handler->handle($context)` передаёт управление следующему перехватчику или целевому действию. Если его не
вызвать, цепочка будет остановлена.

## Регистрация перехватчиков

HTTP-перехватчики можно зарегистрировать в доменном ядре приложения. Стандартный подход — создать загрузчик,
наследующий `Spiral\Bootloader\DomainBootloader`.

```php app/src/Application/Bootloader/DomainBootloader.php
namespace App\Application\Bootloader;

use App\Endpoint\Web\Interceptor\LogInterceptor;
use Spiral\Bootloader\DomainBootloader as BaseDomainBootloader;
use Spiral\Core\CoreInterface;

final class DomainBootloader extends BaseDomainBootloader
{
    protected const SINGLETONS = [
        CoreInterface::class => [self::class, 'domainCore'],
    ];

    protected const INTERCEPTORS = [
        LogInterceptor::class,
    ];
}
```

Затем зарегистрируйте загрузчик в Kernel.

:::: tabs

::: tab С помощью метода

```php app/src/Application/Kernel.php
public function defineAppBootloaders(): array
{
    return [
        // ...
        \App\Application\Bootloader\DomainBootloader::class,
        // ...
    ];
}
```

:::

::: tab С помощью константы

```php app/src/Application/Kernel.php
protected const APP = [
    // ...
    \App\Application\Bootloader\DomainBootloader::class,
    // ...
];
```

:::

::::

Перехватчики выполняются в порядке регистрации. Первый элемент массива становится внешним слоем цепочки и первым
получает вызов.

## Регистрация через загрузчик домена

Перехватчики также можно добавлять программно:

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use App\Endpoint\Web\Interceptor\LogInterceptor;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Bootloader\DomainBootloader;

final class AppBootloader extends Bootloader
{
    public function init(DomainBootloader $domain): void
    {
        $domain->addInterceptor(LogInterceptor::class);
    }
}
```

## Контекст вызова

`CallContextInterface` предоставляет сведения о вызове контроллера.

### Цель

```php
$target = $context->getTarget();

dump((string)$target);
dump($target->getPath());
dump($target->getReflection());
dump($target->getObject());
```

Цель описывает контроллер и действие, которые будут вызваны.

### Аргументы

```php
$arguments = $context->getArguments();
```

Чтобы передать изменённые аргументы следующему обработчику, создайте новый контекст:

```php
$arguments['id'] = (int)$arguments['id'];

return $handler->handle(
    $context->withArguments($arguments)
);
```

### Атрибуты контекста

Атрибуты позволяют передавать дополнительные значения между перехватчиками:

```php
$context = $context->withAttribute('authorized', true);

return $handler->handle($context);
```

Получение значения:

```php
$authorized = $context->getAttribute('authorized', false);
```

## Изменение результата

Перехватчик может изменить результат действия контроллера:

```php
public function intercept(
    CallContextInterface $context,
    HandlerInterface $handler
): mixed {
    $result = $handler->handle($context);

    return [
        'data' => $result,
        'meta' => [
            'generatedAt' => new \DateTimeImmutable(),
        ],
    ];
}
```

## Прерывание вызова

Перехватчик может вернуть результат без обращения к следующему обработчику. Например, при наличии данных в кеше:

```php
use Psr\SimpleCache\CacheInterface;

final readonly class CacheInterceptor implements InterceptorInterface
{
    public function __construct(
        private CacheInterface $cache,
    ) {
    }

    public function intercept(
        CallContextInterface $context,
        HandlerInterface $handler
    ): mixed {
        $key = \sha1((string)$context->getTarget() . \serialize($context->getArguments()));

        if ($this->cache->has($key)) {
            return $this->cache->get($key);
        }

        $result = $handler->handle($context);
        $this->cache->set($key, $result, 300);

        return $result;
    }
}
```

## Авторизация

Перехватчики удобны для централизованной проверки доступа к действиям контроллеров.

```php
final readonly class AuthorizationInterceptor implements InterceptorInterface
{
    public function __construct(
        private AuthorizationInterface $authorization,
    ) {
    }

    public function intercept(
        CallContextInterface $context,
        HandlerInterface $handler
    ): mixed {
        if (!$this->authorization->allows((string)$context->getTarget())) {
            throw new \Spiral\Http\Exception\ClientException\ForbiddenException();
        }

        return $handler->handle($context);
    }
}
```

## Работа с PSR-7-запросом

HTTP-запрос доступен в области запроса и может быть внедрён в перехватчик через метод или получен из контейнера.

```php
use Psr\Http\Message\ServerRequestInterface;

final readonly class RequestLogInterceptor implements InterceptorInterface
{
    public function __construct(
        #[\Spiral\Core\Attribute\Proxy]
        private ServerRequestInterface $request,
        private LoggerInterface $logger,
    ) {
    }

    public function intercept(
        CallContextInterface $context,
        HandlerInterface $handler
    ): mixed {
        $this->logger->info('HTTP action', [
            'method' => $this->request->getMethod(),
            'uri' => (string)$this->request->getUri(),
            'target' => (string)$context->getTarget(),
        ]);

        return $handler->handle($context);
    }
}
```

> **Предупреждение**
> Не сохраняйте конкретный объект запроса в глобальном singleton-сервисе. Используйте внедрение в области запроса или
> прокси-объект.

## Перехватчики отдельных маршрутов

Глобальные перехватчики применяются ко всем действиям доменного ядра. Для более точного управления можно создать
отдельное ядро с определённым набором перехватчиков и назначить его маршруту.

```php
use Spiral\Core\InterceptableCore;

$core = new InterceptableCore($container, $handler);
$core->addInterceptor(AuthorizationInterceptor::class);
$core->addInterceptor(LogInterceptor::class);

$routes->add(name: 'admin.users', pattern: '/admin/users')
    ->core($core)
    ->action(AdminController::class, 'users');
```

Такой подход позволяет использовать разные цепочки перехватчиков для публичных, административных и API-маршрутов.

## Совместимость с устаревшей системой

Spiral 3.x продолжает поддерживать старые перехватчики `Spiral\Core\CoreInterceptorInterface`, но их использование не
рекомендуется. Для совместной работы старых и новых перехватчиков применяется `CompatiblePipelineBuilder`.

```php
use Spiral\Core\CompatiblePipelineBuilder;

$pipeline = (new CompatiblePipelineBuilder())
    ->withInterceptors(
        new LegacyInterceptor(),
        new NewInterceptor(),
    )
    ->build($handler);
```

> **Предупреждение**
> Устаревшая система будет удалена в Spiral 4.x. Новый код следует писать с использованием
> `Spiral\Interceptors\InterceptorInterface`.

## События

| Событие                                      | Описание                                      |
|----------------------------------------------|-----------------------------------------------|
| `Spiral\Interceptors\Event\InterceptorCalling` | Вызывается перед обращением к перехватчику |

> **Примечание**
> Подробнее о событиях читайте в разделе [События](../advanced/events.md).
