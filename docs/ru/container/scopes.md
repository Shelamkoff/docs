# Контейнер — Области IoC

Для создания долгоживущих приложений необходимо правильно управлять контекстом. В демонизированных приложениях нельзя
рассматривать пользовательские запросы как глобальные singleton-объекты, ссылки на которые хранятся в сервисах.

Это означает, что при обработке пользовательского ввода контекст необходимо запрашивать явно. Spiral предоставляет для
этого области IoC-контейнера.

Области позволяют создавать изолированные контексты, переопределять в них сервисы и управлять их жизненным циклом.

## Создание изолированных областей

Чтобы создать изолированный контекст, используйте метод `$container->runScope()`. Первый аргумент — объект `Scope` с
параметрами области, второй — функция, которая будет выполнена внутри неё. Метод `runScope()` возвращает результат этой
функции.

```php
$result = $container->runScope(
    new Scope(bindings: [
        LoggerInterface::class => FileLogger::class,
    ]),
    function () {
        // Your code here
    },
);
```

В этом примере внутри области `LoggerInterface` будет разрешаться как `FileLogger`.

## Принцип работы

При вызове `$container->runScope(new Scope(...), fn() => ...)` создаётся новый контейнер со своими связями. Существующий
контейнер становится родительским для нового.

Новый контейнер используется внутри переданной функции и уничтожается после её завершения.

Важные особенности:

- **Видимость.** Родительские контейнеры не знают о дочерних. При этом сервисы родительского контейнера доступны внутри
  дочерних.
- **Имена областей:**
  - основная глобальная область всегда называется `root`;
  - именованные области должны иметь уникальные имена внутри одной иерархии, чтобы избежать конфликтов;
    ![конфликт областей](https://gist.github.com/user-attachments/assets/32f1ae89-9e35-4e7a-9e53-b3db15fee0ea)
  - параллельные области с одинаковыми именами, например в корутинах, допустимы и имеют собственные иерархии.
- При выходе из области связанный с ней контейнер уничтожается.

### Порядок разрешения зависимостей

При разрешении зависимости внутри изолированной области:

1. контейнер ищет связь в текущей области;
2. если связь не найдена, поиск продолжается в родительской области и далее до корневого контейнера;
3. экземпляр создаётся в той области, где была найдена связь. Его собственные зависимости также разрешаются в этой
   области.

## Предопределённые области

Spiral предоставляет несколько предопределённых областей:

![области Spiral](https://gist.github.com/user-attachments/assets/aa12be0a-bea1-439c-a676-ef8d6158bda9)

1. `root` — основная глобальная область. Все остальные области являются её дочерними.
2. **Область диспетчера** — открывается при запуске соответствующего
   [диспетчера](../framework/dispatcher.md): `http`, `console`, `grpc`, `centrifugo`, `tcp`, `queue` или `temporal`.
3. **Область запроса** — открывается перед выполнением контроллера, когда объект запроса полностью сформирован и готов к
   обработке. Для HTTP-диспетчера middleware выполняются в области `http`, а перехватчики — в `http-request`.

Если сервис используется только определённым диспетчером, его имеет смысл связать в соответствующей области. Например,
HTTP middleware следует связывать на уровне области `http`.

Также можно создавать собственные области, чтобы изолировать контекст и предоставлять только определённые сервисы.

![HTTP-области](https://gist.github.com/user-attachments/assets/a402f166-4396-40ec-a376-d2136fb25824)

## Настройка связей именованных областей

Связи для именованных областей можно заранее настроить методом `BinderInterface::getBinder()`. Это позволяет определить
стандартные связи конкретной области.

```php
$container->bindSingleton(Interface::class, Implementation::class);

// Configure default bindings for 'request' scope
$binder = $container->getBinder('request');
$binder->bindSingleton(Interface::class, Implementation::class);
$binder->bind(Interface::class, factory(...));
```

> **Примечание**
> Изменение связей области не влияет на уже существующие контейнеры этой области, кроме `root`.

## Переопределение стандартных связей

При вызове `Container::runScope()` можно передать связи, которые переопределят стандартные значения для конкретного
запуска области.

```php
$container->bindSingleton(SomeInterface::class, SomeImplementation::class);

$container->runScope(
    new Scope(
        name: 'request',
        bindings: [SomeInterface::class => AnotherImplementation::class],
    ),
    function () {
        // Your code here
    }
);
```

Даже если для области `request` по умолчанию зарегистрирован `SomeInterface`, в этом запуске будет использован
`AnotherImplementation`.

## Ограничение областей

Атрибут `#[Scope('name')]` ограничивает области, в которых разрешено создавать зависимость.

```php
use Spiral\Boot\Environment\DebugMode;
use Spiral\Core\Attribute\Scope;
use Spiral\Core\Attribute\Singleton;

#[Singleton]
#[Scope('http')]
final readonly class DebugMiddleware implements \Psr\Http\Server\MiddlewareInterface
{
    public function process(ServerRequestInterface $request, RequestHandlerInterface $handler): ResponseInterface
    {
        // ...
    }
}
```

В этом примере `DebugMiddleware` можно создать только при наличии области `http` в текущей иерархии. В противном случае
будет выброшено исключение.

## Уничтожение областей и финализация

При выходе из области связанный контейнер уничтожается. Singleton-объекты, созданные внутри области, должны быть собраны
сборщиком мусора, поэтому следует избегать циклических ссылок.

Если при уничтожении области требуется очистить разрешённые зависимости, укажите вызываемый метод атрибутом
`#[Finalize('methodName')]`.

```php
#[Finalize('destroy')]
class MyService
{
    /**
     * This method will be called before the scope is destroyed in case the service was resolved in this scope.
     * Arguments will be resolved using the container.
     */
    public function destroy(LoggerInterface $logger): void
    {
        // Clean up...
    }
}
```

## Прокси-объекты

Области похожи на вложенные контейнеры, но не ограничиваются простой передачей разрешения родителю.

Предположим, что в родительской области `root` или `http` необходимо создать сервис без состояния, работающий с
`ServerRequestInterface` из области `http-request`. При обычных вложенных контейнерах это невозможно: объект запроса
доступен только внутри `http-request` и различается для каждого запроса.

Spiral предоставляет прокси-объекты, откладывающие разрешение зависимости до момента фактического обращения к ней.

Для создания прокси интерфейса используйте атрибут `#[Proxy]`:

```php
use Psr\Http\Message\ServerRequestInterface;
use Spiral\Core\Attribute\Proxy;
use Spiral\Core\Attribute\Singleton;

#[Singleton]
final readonly class DebugService
{
    public function __construct(
        #[Proxy] private ServerRequestInterface $request,
    ) {}

    public function hasDebugInfo(): bool
    {
        return $this->request->hasHeader('X-Debug');
    }
}
```

Важные особенности:

- прокси настраиваются только для интерфейсов;
- при каждом вызове метода прокси получает реальный объект из контейнера;
- вызов методов, не объявленных в интерфейсе, запрещён.

С помощью `Binder` можно настроить прокси для сервисов, доступных только в определённых областях. Например, если
`AuthInterface` доступен только в области `http`, для области `root` можно зарегистрировать прокси:

```php
// Configure a proxy for `AuthInterface` in the `root` scope
$rootBinder = $container->getBinder('root');
$rootBinder->bindSingleton(new \Spiral\Core\Config\Proxy(
    AuthInterface::class,
    singleton: true,
    fallbackFactory: static fn() => throw new \LogicException(
        'Unable to receive AuthInterface instance outside of `http` scope.'
    ),
));

// Bind `AuthInterface` in the `http` scope
$container->getBinder('http')
    ->bindSingleton(AuthInterface::class, Auth::class);
```

Если прокси используется вне области `http`, зависимость будет разрешена через `fallbackFactory`. Если резервная фабрика
не задана, будет выброшено `RecursiveProxyException`.
