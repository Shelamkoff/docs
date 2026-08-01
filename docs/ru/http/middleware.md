# HTTP — Middleware

Spiral использует HTTP middleware, совместимые с [PSR-15](https://www.php-fig.org/psr/psr-15/).

Middleware отвечает за функциональность, связанную с запросом и ответом: аутентификацию, кеширование, журналирование и
другие сквозные задачи. Оно может изменить запрос и ответ до передачи маршрутизатору, но не должно решать, какие
маршруты обрабатывает приложение, обходить или переопределять решения маршрутизатора.

Для функциональности, тесно связанной с маршрутизатором приложения, лучше подходят
[перехватчики](./interceptors.md). Они выполняются после передачи запроса приложению и имеют более широкий доступ к его
внутреннему состоянию, включая маршрутизатор.

## Создание middleware

`Psr\Http\Server\MiddlewareInterface` — стандартный интерфейс PSR-15 для создания middleware. Пользовательский класс
должен реализовать его метод `process`.

```php app/src/Endpoint/Web/Middleware/MyMiddleware.php
namespace App\Endpoint\Web\Middleware;

use Psr\Http\Server\MiddlewareInterface;

class MyMiddleware implements MiddlewareInterface
{
    public function process(
        ServerRequestInterface $request, 
        RequestHandlerInterface $handler
    ): ResponseInterface {
        return $handler->handle($request)->withAddedHeader('My-Header', 'my-value');
    }
}
```

> **Примечание**
> В репозитории [middlewares/psr15-middlewares](https://github.com/middlewares/psr15-middlewares) доступно множество
> поддерживаемых сообществом middleware.

Spiral предоставляет несколько способов регистрации middleware.

### Глобальное middleware

Глобальные middleware применяются ко всем маршрутам и запросам. Обычно так подключают аутентификацию, журналирование и
другую общую функциональность.

Активировать глобальные middleware можно в `RoutesBootloader`:

```php app/src/Application/Bootloader/RoutesBootloader.php
namespace App\Application\Bootloader;

use App\Endpoint\Web\Middleware\LocaleSelector;
use Spiral\Auth\Middleware\AuthTransportMiddleware;
use Spiral\Bootloader\Http\RoutesBootloader as BaseRoutesBootloader;
use Spiral\Cookies\Middleware\CookiesMiddleware;
use Spiral\Core\Container\Autowire;
use Spiral\Csrf\Middleware\CsrfMiddleware;
use Spiral\Debug\StateCollector\HttpCollector;
use Spiral\Http\Middleware\ErrorHandlerMiddleware;
use Spiral\Http\Middleware\JsonPayloadMiddleware;
use Spiral\Session\Middleware\SessionMiddleware;
use App\Endpoint\Web\Middleware\MyMiddleware;

final class RoutesBootloader extends BaseRoutesBootloader
{
    protected function globalMiddleware(): array
    {
        return [
            LocaleSelector::class,
            ErrorHandlerMiddleware::class,
            JsonPayloadMiddleware::class,
            HttpCollector::class,
            MyMiddleware::class,
        ];
    }
    
    // ...
}
```

Для подключения middleware ко всем пользовательским запросам также можно использовать
`Spiral\Bootloader\Http\HttpBootloader`. Изменять его следует только в загрузчиках приложения.

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Bootloader\Http\HttpBootloader;
use Spiral\Core\Container\Autowire;
use Psr\Container\ContainerInterface;
use App\Endpoint\Web\Middleware\MyMiddleware;

class AppBootloader extends Bootloader
{
    public function boot(HttpBootloader $http, ContainerInterface $container): void
    {
        // automatically resolved by Container
        $http->addMiddleware(MyMiddleware::class);
        
        // automatically resolved by Container
        $container->bind('my:middleware', fn() => new MyMiddleware);
        $http->addMiddleware('my:middleware');
        
        // Autowire allows creating an object with dependency resolving from the container
        // and passing some parameters manually
        $http->addMiddleware(new Autowire(MyMiddleware::class, ['someParameter' => 'value']));
    }
}
```

Объект middleware создаётся по требованию.

Middleware также можно настроить в файле `app/config/http.php`:

```php app/config/http.php
use App\Endpoint\Web\Middleware\MyMiddleware;
use Spiral\Core\Container\Autowire;

return [
    // ...
    'middleware' => [
        // via fully qualified class name
        MyMiddleware::class,
        
        'my:middleware',
        
        // via Autowire 
        new Autowire(MyMiddleware::class, ['someParameter' => 'value']),
        
        // or manual instantiating object
        new MyMiddleware(),
    ],
];
```

## Группы middleware

Middleware группы применяются только к маршрутам соответствующей группы. Группы регистрируются в контейнере как
конвейеры с именем `middleware:{group}`, поэтому их можно назначить любому маршруту.

```php app/src/Application/Bootloader/RoutesBootloader.php
namespace App\Application\Bootloader;

use App\Middleware\LocaleSelector;
use Spiral\Auth\Middleware\AuthTransportMiddleware;
use Spiral\Bootloader\Http\RoutesBootloader as BaseRoutesBootloader;
use Spiral\Cookies\Middleware\CookiesMiddleware;
use Spiral\Core\Container\Autowire;
use Spiral\Csrf\Middleware\CsrfMiddleware;
use Spiral\Debug\StateCollector\HttpCollector;
use Spiral\Http\Middleware\ErrorHandlerMiddleware;
use Spiral\Http\Middleware\JsonPayloadMiddleware;
use Spiral\Session\Middleware\SessionMiddleware;
use App\Endpoint\Web\Middleware\MyMiddleware;

final class RoutesBootloader extends BaseRoutesBootloader
{
    // ...

    protected function middlewareGroups(): array
    {
        return [
            'web' => [
                CookiesMiddleware::class,
                SessionMiddleware::class,
                CsrfMiddleware::class,
                MyMiddleware::class,
                // new Autowire(AuthTransportMiddleware::class, ['transportName' => 'cookie'])
            ],
            'api' => [
                // new Autowire(AuthTransportMiddleware::class, ['transportName' => 'header'])
            ],
        ];
    }
    
    // ...
}
```

## Middleware отдельного маршрута

Такое middleware применяется только к определённому маршруту, например одной конечной точке API.

:::: tabs

::: tab Конфигуратор маршрутизации

Используйте метод `middleware` конфигуратора маршрута:

```php app/src/Application/Bootloader/RoutesBootloader.php
namespace App\Application\Bootloader;

use Spiral\Bootloader\Http\RoutesBootloader as BaseRoutesBootloader;
use Spiral\Router\Loader\Configurator\RoutingConfigurator;
use App\Endpoint\Web\Middleware\MyMiddleware;

final class RoutesBootloader extends BaseRoutesBootloader
{
    // ...
 
    protected function defineRoutes(RoutingConfigurator $routes): void
    {
        $routes->add(name: 'news.show', pattern: '/news/<id:int>')
            ->middleware(['middleware:web', MyMiddleware::class]);
        ...
    }
}
```

:::

::: tab Атрибуты маршрута

Укажите middleware в свойстве `middleware` атрибута `Spiral\Router\Annotation\Route`:

```php app/src/Application/Controller/HomeController.php
namespace App\Endpoint\Web;

use Spiral\Router\Annotation\Route;
use App\Endpoint\Web\Middleware\MyMiddleware;

class HomeController
{
    #[Route(route: '/', name: 'index', methods: 'GET', middleware: [MyMiddleware::class])] 
    public function index(): string
    {
        // ...
    }
}
```

:::

::: tab Router

Для объекта маршрута используйте метод `withMiddleware`. Он возвращает новый экземпляр маршрута:

```php app/src/Application/Bootloader/AppBootloader.php
use App\Controller\HomeController;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;
use Spiral\Router\Target\Action;
use App\Endpoint\Web\Middleware\MyMiddleware;

// ...

public function boot(RouterInterface $router): void
{
    $route = new Route('/index', new Action(HomeController::class, 'index'));
    $route = $route->withMiddleware(MyMiddleware::class);

    $router->addRoute('index', $route);
}
```

:::

::::

## Совместное использование с областями IoC

Middleware можно объединить с [областью IoC](../framework/scopes.md), чтобы создать контекст, относящийся только к
текущему запросу. Такой контекст доступен другим частям приложения и подходит, например, для журналирования или доступа
к данным пользователя.

```php
class UserContext
{
    public int $id;
    public string $name;

    public function __construct(int $id, string $name)
    {
        $this->id = $id;
        $this->name = $name;
    }
}
```

С помощью `Spiral\Core\ScopeInterface` middleware может открыть область для текущего запроса:

```php app/src/Endpoint/Web/Middleware/MyMiddleware.php
namespace App\Endpoint\Web\Middleware;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;
use Spiral\Core\ScopeInterface;

class MyMiddleware implements MiddlewareInterface
{
    public function __construct(
        private readonly ScopeInterface $scope
    ) {
    }

    public function process(
        ServerRequestInterface $request,
        RequestHandlerInterface $handler
    ): ResponseInterface {
        return $this->scope->runScope([
            UserContext::class => new UserContext(123, 'test')
        ], function () use ($handler, $request) {
            return $handler->handle($request);
        });
    }
}
```

После создания контекста его можно получить из контейнера или внедрить в метод контроллера:

```php
public function index(UserContext $ctx): void
{
    dump($ctx);
}
```

> **Примечание**
> Область, созданная middleware, действует только во время текущего запроса и не влияет на остальные запросы. Это
> сохраняет изоляцию и целостность контекста каждого запроса.

## Использование существующей области запроса

Существующую область запроса можно использовать для передачи пользовательских значений. Middleware добавляет значение
в атрибут запроса:

```php app/src/Endpoint/Web/Middleware/MyMiddleware.php
namespace App\Endpoint\Web\Middleware;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;

class MyMiddleware implements MiddlewareInterface
{
    public function process(
        ServerRequestInterface $request,
        RequestHandlerInterface $handler
    ): ResponseInterface {
        return $handler->handle($request->withAttribute('userContext', new UserContext(123, 'test')));
    }
}
```

Для получения значения из контейнера создайте загрузчик:

```php app/src/Application/Bootloader/UserContextBootloader.php
namespace App\Application\Bootloader;

use Psr\Http\Message\ServerRequestInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Core\Exception\ScopeException;

class UserContextBootloader extends Bootloader 
{
    protected const BINDINGS = [
        UserContext::class => [self::class, 'userContext']
    ];
    
    private function userContext(ServerRequestInterface $request): UserContext
    {
        $userContext = $request->getAttribute('userContext', null);
        if ($userContext === null) {
            throw new ScopeException('Unable to resolve UserContext, invalid request scope');
        }
        
        return $userContext;
    }
}
```

## Доступные middleware

HTTP-расширение включает несколько middleware, которые можно активировать в проекте:

| Загрузчик                                      | Middleware                                                                 |
|------------------------------------------------|----------------------------------------------------------------------------|
| Spiral\Bootloader\Http\ErrorHandlerBootloader | Скрывает исключения вне режима отладки и отображает страницы HTTP-ошибок   |
| Spiral\Bootloader\Http\JsonPayloadsBootloader | Разбирает тело запросов `application/json`                                 |
| Spiral\Bootloader\Http\PaginationBootloader   | Настраивает пагинаторы по параметрам строки запроса                        |
| Spiral\Bootloader\Http\DiactorosBootloader    | Использует Zend/Diactoros как реализацию PSR-7 — устаревший вариант         |

## События

| Событие                                  | Описание                                      |
|------------------------------------------|-----------------------------------------------|
| Spiral\Http\Event\MiddlewareProcessing | Вызывается `до` обращения к middleware        |

> **Примечание**
> Подробнее о диспетчеризации событий читайте в разделе [События](../advanced/events.md).
