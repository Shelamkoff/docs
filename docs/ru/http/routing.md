# HTTP — Маршрутизация

## Установка

Компонент маршрутизации установлен в Spiral по умолчанию. Для использования в собственной сборке установите его через
Composer:

```terminal
composer require spiral/router
```

## Маршрутизация с помощью атрибутов

Самый простой способ определить маршруты — добавить атрибуты непосредственно к методам контроллера. Такой подход делает
код компактнее и понятнее, улучшает разделение ответственности и помогает разработчикам быстро находить доступные
маршруты.

> **Примечание**
> Для [статического анализа](../advanced/tokenizer.md) и поиска атрибутов маршрутов используется компонент Tokenizer.
> Каталоги поиска контроллеров настраиваются в разделе
> [Настройка каталогов поиска](../advanced/tokenizer.md#customizing-search-directories).

> **Предупреждение**
> Tokenizer игнорирует файлы контроллеров, содержащие операторы `include` или `require`.

Активируйте загрузчик `Spiral\Router\Bootloader\AnnotatedRoutesBootloader`:

:::: tabs

::: tab С помощью метода

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Router\Bootloader\AnnotatedRoutesBootloader::class,
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
    \Spiral\Router\Bootloader\AnnotatedRoutesBootloader::class,
    // ...
];
```

Подробнее о загрузчиках читайте в разделе [Фреймворк — Загрузчики](../framework/bootloaders.md).
:::

::::

После этого компонент готов к использованию.

### Определение маршрутов

Атрибут `Spiral\Router\Annotation\Route` позволяет определить маршрут для метода контроллера:

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use Spiral\Router\Annotation\Route;

class HomeController
{
    #[Route(route: '/', name: 'index', methods: 'GET')] 
    public function index(): string
    {
        return 'hello world';
    }
}
```

Параметры атрибута:

| Свойство   | Тип          | Описание                                                                                                                                                        |
|------------|--------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| route      | string       | Шаблон URL маршрута. [Маршрутизатор](/http/routing.md). **Обязательное**                                                                                         |
| name       | string       | Имя маршрута. **Необязательное**                                                                                                                                |
| methods    | array/string | HTTP-методы маршрута: `GET`, `POST`, `PUT` и другие. По умолчанию разрешены все методы                                                                           |
| defaults   | array        | Значения параметров маршрута по умолчанию                                                                                                                       |
| group      | string       | Группа маршрута. По умолчанию `default`                                                                                                                         |
| middleware | array        | Классы middleware конкретного маршрута                                                                                                                          |
| priority   | int          | Позиция в списке маршрутов. Чем меньше число, тем выше приоритет. Используется, когда запрос соответствует нескольким маршрутам. По умолчанию `0`                 |

### Имя маршрута

Маршрутам рекомендуется назначать имена, чтобы ссылаться на них из других частей приложения. Если имя не указано,
Spiral сформирует его автоматически из шаблона и HTTP-методов.

```php
#Route(route: '/api/news', methods: ["POST", "PATCH"]) // => post,patch:/api/news
````

## Объявление маршрутов

:::: tabs

::: tab Конфигуратор маршрутизации

Метод `defineRoutes` класса `App\Application\Bootloader\RoutesBootloader` предоставляет экземпляр
`Spiral\Router\Loader\Configurator\RoutingConfigurator` для создания и настройки маршрутов.

> **Предупреждение**
> `App\Application\Bootloader\RoutesBootloader` должен находиться в секции `LOAD` списка загрузчиков.

Через `RoutingConfigurator` можно задавать middleware, префиксы, HTTP-методы и автоматически регистрировать маршруты.

```php app/src/Application/Bootloader/RoutesBootloader.php
namespace App\Application\Bootloader;

use Spiral\Bootloader\Http\RoutesBootloader as BaseRoutesBootloader;
use Spiral\Router\Loader\Configurator\RoutingConfigurator;

final class RoutesBootloader extends BaseRoutesBootloader
{
    // ...
 
    protected function defineRoutes(RoutingConfigurator $routes): void
    {
        $routes->add(name: 'news.show', pattern: '/news/<id:int>')
            ->group('web')
            ->methods(methods: ['GET'])
            ->action(NewsController::class, 'show');
            
        ...
    }
}
```

:::

::: tab Импорт маршрутов

`RoutingConfigurator` позволяет импортировать маршруты из отдельных файлов.

```php app/src/Application/Bootloader/RoutesBootloader.php
use Spiral\Router\Loader\Configurator\RoutingConfigurator;

namespace App\Application\Bootloader;

use Spiral\Boot\DirectoriesInterface;
use Spiral\Bootloader\Http\RoutesBootloader as BaseRoutesBootloader;
use Spiral\Router\Loader\Configurator\RoutingConfigurator;

final class RoutesBootloader extends BaseRoutesBootloader
{
    public function __construct(
        private readonly DirectoriesInterface $dirs
    ) {
    }
    
    // ...

    protected function defineRoutes(RoutingConfigurator $routes): void
    {
        $routes->import($this->dirs->get('app') . '/routes/web.php')
            ->group('web');
            
        $routes->import($this->dirs->get('app') . '/routes/api.php')
            ->prefix('/api');
            ->group('api');
    }
}
```

В файле `web.php` добавьте маршруты методом `add`:

```php app/routes/web.php
use App\Controller\HomeController;
use Spiral\Router\Loader\Configurator\RoutingConfigurator;

return function (RoutingConfigurator $routes): void {
    $routes->add(name: 'news.show', pattern: '/news/<id:int>')
        ->methods(methods: ['GET'])
        ->action(NewsController::class, 'show');
};
```

:::

::::

## Конфигуратор маршрута

Конфигуратор предоставляет набор методов для определения и настройки маршрута.

### Цель маршрута

Цель определяет контроллер или другой обработчик маршрута.

:::: tabs

::: tab Пространство имён

Метод `namespaced` направляет запрос к набору контроллеров в одном пространстве имён. Шаблон должен содержать параметры
`<controller>` и `<action>`, если для них не заданы значения по умолчанию.

```php
$routes->add(name: 'admin', pattern: '/admin/<controller>/<action>')
    ->namespaced(
        namespace: 'App\Controllers\Admin', // required
    );
```

**Пример запроса**

```bash
GET /admin/users/index
```

Параметр `<controller>` равен `users`, а `<action>` — `index`. Запрос будет направлен методу `index` класса
`UsersController` из пространства имён `App\Controllers\Admin`.

По умолчанию предполагается, что имена контроллеров заканчиваются на `Controller`. Изменить постфикс можно аргументом
`postfix`:

```php
$routes->add(name: 'admin', pattern: '/admin/<controller>/<action>')
    ->namespaced(
        namespace: 'App\Controllers\Admin',
        postfix: 'Handler'
    );
```

:::

::: tab Контроллер

Метод `controller` направляет маршрут ко всем действиям одного контроллера. Для него требуется параметр `<action>`, если
не задано значение по умолчанию.

```php
$routes->add(name: 'user', pattern: '/user/<action>')
    ->controller(controller: UserController::class);
```

**Пример запроса**

```bash
GET /user/list
```

Запрос будет направлен методу `list` класса `UserController`.

Если шаблон не содержит сегмент `<action>`, по умолчанию используется действие `index`. Другое действие по умолчанию
можно указать методом `defaults`:

```php
$routes->add(name: 'user', pattern: '/user/list')
    ->controller(controller: UserController::class)
    ->defaults(['action' => 'list'])
```

:::

::: tab Действие

Метод `action` направляет маршрут определённому действию контроллера.

```php
$routes->add(name: 'post.show', pattern: '/post/<id:int>')
    ->action(PostController::class, 'show');
```

**Пример запроса**

```bash
GET /post/42
```

Запрос будет направлен методу `show` класса `PostController`, а значение `42` будет передано аргументом.

:::

::: tab Callable

Маршрут можно направить непосредственно функции, замыканию или массиву из объекта и имени метода. Это удобно для
небольшого обработчика, которому не требуется отдельный контроллер.

```php
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;

$routes->add(name: 'greeting', pattern: '/greeting')
    ->callable(function (ServerRequestInterface $r, ResponseInterface $rsp) {
        return 'Hello, world!';
    });
```

```


:::

::::

### Параметры маршрута по умолчанию

```php
$routes
    ->add(name: 'html', pattern: '/<action>.html')
    ->defaults(['action' => 'default'])
    ->...;
```

### Пользовательское доменное ядро

```php
$core = new \Spiral\Core\InterceptableCore(...);
$core->addInterceptor(...);

$routes
    ->add(name: 'html', pattern: '/<action>.html')
    ->core($core)
    ->...;
```

### Префикс маршрута

```php
$routes->add(name: 'news', pattern: '/news/<id:int>')
    ->prefix('/api')
    ->...;
```

### HTTP-методы

```php
$routes->add(name: 'html', pattern: '/<action>.html')
    ->methods('GET')
    ->...;

// or

$routes->add(name: 'news', pattern: '/news/<id:int>')
    ->methods(['GET', 'POST'])
    ->...;
```

### Добавление middleware

```php
$routes->add(name: 'news', pattern: '/news/<id:int>')
    ->middleware(LocaleSelector::class)
    ->...;
```

### Резервный маршрут

Если URL не соответствует ни одному определённому маршруту, резервный маршрут позволяет вернуть осмысленный ответ.

```php
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;

$routes->default('/<path:.*>')
    ->callable(function (ServerRequestInterface $r, ResponseInterface $rsp) {
        return 'Page not found!';
    });
```

> **Примечание**
> Для резервного маршрута можно использовать не только callable, но и любую другую цель: контроллер, действие,
> пространство имён и так далее.

## Конфигуратор групп маршрутов

Маршруты можно объединять в логические группы и применять ко всей группе middleware, префиксы и другие настройки.
Это упрощает сопровождение и масштабирование крупных приложений.

Группы настраиваются через `App\Application\Bootloader\RoutesBootloader`. Метод `configureRouteGroups` получает
`Spiral\Router\GroupRegistry`.

```php app/src/Application/Bootloader/RoutesBootloader.php
namespace App\Application\Bootloader;

use Spiral\Router\GroupRegistry;
use Spiral\Bootloader\Http\RoutesBootloader as BaseRoutesBootloader;

final class RoutesBootloader extends BaseRoutesBootloader
{
    // ...

    protected function configureRouteGroups(GroupRegistry $groups): void
    {
        $groups->getGroup('api')
            ->setNamePrefix('api.')
            ->setPrefix('/api');
            
        $groups->getGroup('web')
            ->addMiddleware(MyMiddelware::class)
            ->setPrefix('/api');
    }
}
```

Способы назначения маршрутов группе:

:::: tabs

::: tab Route

Укажите имя группы в конфигураторе маршрута.

```php app/src/Application/Bootloader/RoutesBootloader.php
$routes->add(name: 'news', pattern: '/news/<id:int>')
    ->action(NewsController::class, 'show')
    ->group('auth');
    ->methods('GET');
```

:::

::: tab Атрибуты

Укажите имя группы в параметре `group` атрибута.

> **Предупреждение**
> Загрузчик настройки группы должен быть зарегистрирован после `AnnotatedRoutesBootloader`.

```php app/src/Endpoint/Web/HomeController.php
#[Route(route: '/', name: 'index', methods: 'GET', group: 'api')]  
public function index(): ResponseInterface
{
    // ...    
}
```

Маршрут будет добавлен в группу `api`, получит префикс группы и её middleware.

:::

::: tab Глобально

Можно назначить группу по умолчанию для всех маршрутов:

```php app/src/Application/Bootloader/RoutesBootloader.php
protected function configureRouteGroups(GroupRegistry $groups): void
{
    // ...
    
    $groups->getGroup('api')
        ->setNamePrefix('api.')
        ->setPrefix('/api');
    
    $groups->setDefaultGroup('api');
}
```

:::

::::

## Router

Новые маршруты можно создавать непосредственно через `Spiral\Router\RouterInterface`.

Простой обработчик `/`:

```php app/src/Application/Bootloader/RoutesBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router): void
    {
        $router->setRoute(
            'home',                    // route name 
            new Route(
                '/',                   // pattern
                fn () => 'hello world' // handler
            )
        );
    }
}
```

> **Примечание**
> Класс `Route` принимает обработчик типа `Psr\Http\Server\RequestHandlerInterface`, замыкание, вызываемый класс или
> `Spiral\Router\TargetInterface`. Чтобы объект создавался по требованию, можно передать имя класса или связи вместо
> готового экземпляра.

### Обработчик-замыкание

В качестве обработчика маршрута можно передать замыкание. Оно получает два аргумента:
`Psr\Http\Message\ServerRequestInterface` и `Psr\Http\Message\ResponseInterface`.

```php
$router->setRoute('home', new Route(
    '/<name>',
    function (ServerRequestInterface $request, ResponseInterface $response): ResponseInterface {
        $response->getBody()->write('hello world');

        return $response;
    }
));
```

### Шаблон и параметры маршрута

Шаблон маршрута может содержать любое количество обязательных и необязательных параметров. Они передаются обработчику
через атрибут `matches` объекта `ServerRequestInterface`.

> **Примечание**
> В фильтрах запросов используйте `attribute:matches.id` для доступа к этим значениям.

Параметр задаётся в формате `<parameter_name:pattern>`, где `pattern` — выражение, совместимое с регулярными
выражениями. Шаблон можно опустить и использовать `<parameter_name>`: тогда будет применено выражение `[^\/]+`.

Простой параметр `name`:

```php
namespace App\Bootloader;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router): void
    {
        $router->setRoute('home', new Route(
            '/<name>',
            function (ServerRequestInterface $request, ResponseInterface $response): array {
                return $request->getAttribute('route')->getMatches(); // returns JSON ['name' => '']
            }
        ));
    }
}
```

Квадратные скобки `[]` делают часть маршрута, включая параметры, необязательной:

```php
$router->setRoute('home', new Route(
    '/[<name>]',
    function (ServerRequestInterface $request, ResponseInterface $response): array {
        return $request->getAttribute('route')->getMatches();
    }
));
```

> **Примечание**
> Маршрут соответствует `/`, а параметр `name` будет равен `null`.

Можно комбинировать обязательные и необязательные параметры. Например, URL `/group/user`, где `user` необязателен:

```php
$router->setRoute('home', new Route(
    '/<group>[/<user>]',
    function (ServerRequestInterface $request, ResponseInterface $response): array {
        return $request->getAttribute('route')->getMatches();
    }
));
```

Значения по умолчанию передаются третьим аргументом:

```php
$router->setRoute('home', new Route(
    '/<group>[/<user>]',
    function (ServerRequestInterface $request, ResponseInterface $response): array {
        return $request->getAttribute('route')->getMatches();
    },
    [
        'user' => 'default'
    ]
));
```

Пользовательский шаблон параметра:

```php
$router->setRoute('home', new Route(
    '/user/<id:\d+>',
    function (ServerRequestInterface $request, ResponseInterface $response): array {
        return $request->getAttribute('route')->getMatches();
    }
));
```

> **Примечание**
> Маршрут соответствует только числовому `id`, но значение атрибута всё равно будет строкой.

#### Предопределённые варианты

Для параметра можно перечислить несколько допустимых вариантов:

```php
$router->setRoute('home', new Route(
    '/do/<action:login|logout>',
    function (ServerRequestInterface $request, ResponseInterface $response): array {
        return $request->getAttribute('route')->getMatches();
    }
));
``` 

> **Примечание**
> Такой маршрут соответствует только `/do/login` и `/do/logout`.

#### Сопоставление домена

Чтобы сопоставить имя домена или поддомена, начните шаблон с `//`:

```php
$router->setRoute('home', new Route(
    '//<host>/',
    function (ServerRequestInterface $request, ResponseInterface $response): array {
        return $request->getAttribute('route')->getMatches();
    }
));
```

Сопоставление поддомена:

```php
$router->setRoute('home', new Route(
    '//<sub>.domain.com/',
    function (ServerRequestInterface $request, ResponseInterface $response): array {
        return $request->getAttribute('route')->getMatches();
    }
));
```

Домен и путь можно объединить:

```php
$router->setRoute('home', new Route(
    '//<sub>.domain.com/[<action>]',
    function (ServerRequestInterface $request, ResponseInterface $response): array {
        return $request->getAttribute('route')->getMatches();
    }
));
```

#### Неизменяемость

Объекты маршрутов неизменяемы. Изменить существующий экземпляр нельзя, но можно получить его копию с новыми значениями.

```php
namespace App\Bootloader;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router): void
    {
        $route = new Route('/[<action>]', function (ServerRequestInterface $request, ResponseInterface $response): array {
            return $request->getAttribute('route')->getMatches();
        });

        $router->setRoute('home', $route->withDefaults([
            'action' => 'default'
        ]));
    }
}
```

#### HTTP-методы

Метод `withVerbs` ограничивает маршрут определёнными HTTP-методами:

```php
namespace App\Bootloader;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router): void
    {
        $route = new Route('/[<action>]', function (ServerRequestInterface $request, ResponseInterface $response): array {
            return $request->getAttribute('route')->getMatches();
        });

        $router->setRoute('get.route',
            $route->withVerbs('GET')->withDefaults(['action' => 'GET'])
        );

        $router->setRoute(
            'post.route',
            $route->withVerbs('POST', 'PUT')->withDefaults(['action' => 'POST'])
        );
    }
}
```

#### Middleware

Метод `withMiddleware` назначает маршруту middleware. Параметры маршрута доступны через атрибут `route` запроса:

```php
namespace App\Bootloader;

use App\Middleware\ParamWatcher;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router): void
    {
        $route = new Route('/<param>', function (ServerRequestInterface $request, ResponseInterface $response): array {
            return $request->getAttribute('route')->getMatches();
        });

        $router->setRoute('home', $route->withMiddleware(
            ParamWatcher::class
        ));
    }
}
```

Класс `ParamWatcher`:

```php
namespace App\Middleware;

use Psr\Http\Message\ResponseInterface as Response;
use Psr\Http\Message\ServerRequestInterface as Request;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;
use Spiral\Http\Exception\ClientException\UnauthorizedException;
use Spiral\Router\RouteInterface;

class ParamWatcher implements MiddlewareInterface
{
    public function process(Request $request, RequestHandlerInterface $handler): Response
    {
        /** @var RouteInterface $route */
        $route = $request->getAttribute('route');

        if ($route->getMatches()['param'] === 'forbidden') {
           throw new UnauthorizedException();
        }

        return $handler->handle($request);
    }
}
```

При запросе `/forbidden` будет выброшено исключение Unauthorized.

> **Примечание**
> Маршруту можно назначить любое количество middleware.

### Несколько маршрутов

Маршрутизатор проверяет маршруты в порядке регистрации. Избегайте ситуаций, когда предыдущий маршрут перехватывает
запросы, предназначенные следующим маршрутам.

```php
$router->setRoute(
    'home',
    new Route('/<param>',
        function (ServerRequestInterface $request, ResponseInterface $response): array {
            return $request->getAttribute('route')->getMatches();
        }
    )
);

// this route will never trigger
$router->setRoute(
    'hello',
    new Route('/hello',
        function (ServerRequestInterface $request, ResponseInterface $response): array {
            return $request->getAttribute('route')->getMatches();
        }
    )
);
```

### Маршрут по умолчанию

Маршрут по умолчанию проверяется после всех остальных маршрутов.

```php
$router->setRoute(
    'home',
    new Route('/<param>',
        function (ServerRequestInterface $request, ResponseInterface $response): array {
            return $request->getAttribute('route')->getMatches();
        }
    )
);

$router->setDefault(new Route('/', fn (): string => 'default'));
``` 

Такой маршрут позволяет быстро организовать пути приложения без отдельного объявления каждого контроллера и действия.

### Цели маршрутов: контроллеры и действия

Для демонстрации целей маршрута создадим несколько контроллеров в пространстве имён `App\Controller`:

```php
namespace App\Controller;

class HomeController
{
    public function index(): string
    {
        return 'index';
    }

    public function other(): string
    {
        return 'other';
    }

    public function user(int $id): string
    {
        return "hello {$id}";
    }
}
```

Создадим второй контроллер командой `php ./app.php create:controller demo -a test`:

```php
namespace App\Controller;

class DemoController
{
    public function test(): string
    {
        return 'demo test';
    }
}
```

#### Маршрут к действию

Для направления маршрута действию контроллера используйте `Spiral\Router\Target\Action`:

```php
namespace App\Bootloader;

use App\Controller\HomeController;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;
use Spiral\Router\Target\Action;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router): void
    {
        $router->setRoute(
            'index',
            new Route('/index', new Action(HomeController::class, 'index'))
        );
    }
}
```

Цель можно объединить с обязательным или необязательным параметром. Параметр будет внедрён в метод:

```php
$router->setRoute(
    'user',
    new Route('/user/<id:\d+>', new Action(HomeController::class, 'user'))
);
```

#### Динамические действия

Один маршрут можно направить нескольким действиям. Для этого добавьте параметр `<action>` в шаблон. Параметр `<id>`
сделаем необязательным:

```php
$router->setRoute(
    'home',
    new Route('/<action>[/<id>]', new Action(HomeController::class, ['index', 'user']))
);
```

> **Примечание**
> Маршрут соответствует путям `/index` и `/user/1`.

Маршрут компилируется в выражение с ограничением действий:
`/^(?P<action>index|user)(?:\/(?P<id>[^\/]+))?$/iu`. Это повышает производительность и позволяет повторно использовать
один шаблон с разными наборами действий.

```php
// match "/index"
$router->setRoute(
    'home',
    new Route('/<action>', new Action(HomeController::class, 'index'))
);

// match "/other"
$router->setRoute(
    'home',
    new Route('/<action>', new Action(HomeController::class, 'other'))
);

// match "/test"
$router->setRoute(
    'demo',
    new Route('/<action>', new Action(DemoController::class, 'test'))
);
```

#### Маршрут к контроллеру

`Spiral\Router\Target\Controller` направляет маршрут всем действиям контроллера. Для цели требуется параметр `<action>`,
если не задано значение по умолчанию.

```php
namespace App\Bootloader;

use App\Controller\HomeController;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;
use Spiral\Router\Target\Controller;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router): void
    {
        $router->setRoute(
            'home',
            new Route('/home/<action>[/<id>]', new Controller(HomeController::class))
        );
    }
}
```

> **Примечание**
> Маршрут соответствует `/home/index`, `/home/other` и `/home/user/1`.

Значения по умолчанию позволяют сделать URL короче:

```php
$router->setRoute(
    'home',
    (new Route('/home[/<action>[/<id>]]', new Controller(HomeController::class)))
        ->withDefaults(['action' => 'index'])
);
```

> **Примечание**
> Маршрут соответствует `/home` с `action=index`. Необязательные сегменты `[]` должны продолжаться до конца шаблона.

#### Маршрут к пространству имён

`Spiral\Router\Target\Namespaced` направляет запросы набору контроллеров одного пространства имён. Требуются параметры
`<controller>` и `<action>`, если не заданы значения по умолчанию.

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;
use Spiral\Router\Target\Namespaced;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router): void
    {
        $router->setRoute('app', new Route(
            '/<controller>/<action>',
            new Namespaced('App\Controller', 'Controller')
        ));
    }
}
```

> **Примечание**
> Маршрут соответствует `/home/index`, `/home/other` и `/demo/test`.

Параметры можно сделать необязательными и назначить значения по умолчанию:

```php
$router->setRoute('app',
    (new Route(
        '[/<controller>[/<action>]]',
        new Namespaced('App\Controller', 'Controller')
    ))->withDefaults([
        'controller' => 'home',
        'action'     => 'index'
    ])
);
```

> **Примечание**
> Маршрут соответствует `/`, `/home`, `/home/index`, `/home/other` и `/demo/test`. Путь `/demo` вызовет ошибку, поскольку
> `DemoController` не содержит метод `index`.

Стандартная сборка веб-приложения назначает такой маршрут
[маршрутом по умолчанию](https://github.com/spiral/app/blob/2.x/app/src/Bootloader/RoutesBootloader.php#L42). Для
контроллеров пространства имён `App\Controller` не требуется создавать отдельные маршруты: используйте URL
`/controller/action`. Если действие не указано, по умолчанию используется `index`. Маршрутизатор обращается только к
публичным методам.

> **Примечание**
> После завершения разработки маршрут по умолчанию можно отключить.

#### Маршрут к группе контроллеров

`Spiral\Router\Target\Group` позволяет явно перечислить контроллеры без общего пространства имён. Для цели нужны
параметры `<controller>` и `<action>`, если не заданы значения по умолчанию.

```php
namespace App\Bootloader;

use App\Controller\DemoController;
use App\Controller\HomeController;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;
use Spiral\Router\Target\Group;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router): void
    {
        $router->setRoute('app', new Route('/<controller>/<action>', new Group([
            'home' => HomeController::class,
            'demo' => DemoController::class
        ])));
    }
}
```

> **Примечание**
> Такой подход полезен при объединении нескольких модулей под одним путём, например для административной панели.

## Именованные шаблоны параметров

Если параметр маршрута всегда должен соответствовать определённому регулярному выражению, зарегистрируйте именованный
шаблон через `Spiral\Router\Registry\RoutePatternRegistryInterface` в загрузчике:

```php
use Spiral\Router\Registry\RoutePatternRegistryInterface;

class AppBootloader extends Bootloader
{
   public function boot(RoutePatternRegistryInterface $patternRegistry): void
   {
      $patternRegistry->register(
          'uuid', 
          '[0-9a-fA-F]{8}\b-[0-9a-fA-F]{4}\b-[0-9a-fA-F]{4}\b-[0-9a-fA-F]{4}\b-[0-9a-fA-F]{12}'
      );
      $patternRegistry->register(
          'names', 
          new InArrayPattern(['tom', 'jerry'])
      );
   }
}
```

После регистрации шаблон автоматически применяется к параметрам с соответствующим именем.

#### Примеры

```php
#Route(uri: 'blog/post/<post:uuid>')  // <===== Will match: /blog/post/f403554a-e70f-479a-969b-3edc047912a3
public function show(string $post)
{ 
    \var_dump($post); // f403554a-e70f-479a-969b-3edc047912a3
}
```

```php
#Route(uri: 'user/<name:names>') // <===== Will match: /user/tom || /user/jerry
public function show(string $name)
{ 
    \var_dump($name); // tom
}
```

## RESTful

Все перечисленные цели маршрутов поддерживают третий аргумент, определяющий способ выбора метода. Значение
`AbstractTarget::RESTFUL` автоматически добавляет к имени метода префикс HTTP-глагола.

Контроллер:

```php app/src/Endpoint/Web/UserController.php
namespace App\Endpoint\Web;

class UserController
{
    public function getUser($id): string
    {
        return "get {$id}";
    }

    public function postUser($id): string
    {
        return "post {$id}";
    }

    public function deleteUser($id): string
    {
        return "delete {$id}";
    }
}
```

Маршрут:

```php
$router->setRoute('user', new Route(
    '/user/<id:\d+>',
    new Controller(UserController::class, Controller::RESTFUL),
    ['action' => 'user']
));
```

> **Примечание**
> Запросы `/user/1` с разными HTTP-методами вызывают разные методы контроллера. Имя действия всё равно необходимо
> указать.

### Повторное использование цели

Общую цель можно использовать в нескольких маршрутах и направлять разные HTTP-глаголы разным методам контроллера.

```php app/src/Endpoint/Web/UserController.php
namespace App\Endpoint\Web;

class UserController
{
    public function load($id): string
    {
        return "get {$id}";
    }

    public function store($id): string
    {
        return "post {$id}";
    }

    public function delete($id): string
    {
        return "delete {$id}";
    }
}
```

Создадим API вида `GET|POST|DELETE /v1/<controller>`.

Базовый маршрут:

```php
$resource = new Route('/v1/<controller>', new Group([
    'user' => UserController::class,
]));
```

Регистрация с разными HTTP-глаголами и действиями:

```php app/src/Application/Bootloader/RoutesBootloader.php
namespace App\Application\Bootloader;

use App\Controller\UserController;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;
use Spiral\Router\Target\Group;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router): void
    {
        $resource = new Route('/v1/<controller>/<id>', new Group([
            'user' => UserController::class,
        ]));

        $router->setRoute(
            'resource.get',
            $resource->withVerbs('GET')->withDefaults(['action' => 'load'])
        );

        $router->setRoute(
            'resource.store',
            $resource->withVerbs('POST')->withDefaults(['action' => 'store'])
        );

        $router->setRoute(
            'resource.delete',
            $resource->withVerbs('DELETE')->withDefaults(['action' => 'delete'])
        );
    }
}
```

Один набор маршрутов можно использовать для нескольких ресурсных контроллеров.

## Генерация URL

Маршрутизатор может сформировать URI по имени маршрута и его параметрам.

```php
$router->setRoute(
    'home',
    new Route('/home/<action>', new Controller(HomeController::class))
);
```

Используйте метод `uri` интерфейса `RouterInterface`:

```php app/src/Endpoint/Web/UserController.php
use Spiral\Router\RouterInterface;

// ...

public function index(RouterInterface $router)
{
    $uri = $router->uri('home', ['action' => 'index']);

    dump((string)$uri); // /home/index
}
```

Дополнительные параметры добавляются в строку запроса:

```php app/src/Endpoint/Web/UserController.php
use Spiral\Router\RouterInterface;

// ...

public function index(RouterInterface $router)
{
    $uri = $router->uri('home', [
        'action' => 'index',
        'page'   => 123
    ]);

    dump((string)$uri); // /home/index?page=123
}
```

Метод `uri` возвращает `Psr\Http\Message\UriInterface`:

```php app/src/Endpoint/Web/UserController.php
use Spiral\Router\RouterInterface;

// ...

public function index(RouterInterface $router)
{
    $uri = $router->uri('home', [
        'action' => 'index',
        'page'   => 123
    ]);

    dump((string)$uri->withFragment('hello')); // /home/index?page=123#hello
}
```

Параметры, помещаемые в шаблон URL, преобразуются в slug:

```php app/src/Endpoint/Web/UserController.php
use Spiral\Router\RouterInterface;

// ...

public function index(RouterInterface $router)
{
    $uri = $router->uri('home', [
        'action' => 'hello World',
    ]);

    dump((string)$uri); // /home/hello-world
}
```

> **Примечание**
> В представлениях Stempler можно использовать директиву `@route(name, params)`.

#### Нелатинские символы в URI

По умолчанию маршрутизатор Spiral транслитерирует нелатинские символы при формировании URI. Это может быть нежелательно
для SEO или многоязычных приложений, где важно сохранить исходный набор символов.

```php
$router->setRoute(
  'page',
  new Route('/page/<path>', ....),
);

$uri = $router->uri('page', ['path' => 'some-path']);
// Generates: /page/some-path

$uri = $router->uri('page', ['path' => 'некоторый-путь']); 
// Default behavior generates: /page/nekotoriy-put
```

Изменить поведение можно, заменив обработчик URI маршрута пользовательской функцией кодирования через
`withPathSegmentEncoder`.

```php
$router->setRoute(
  'page',
  new Route('/page/<path>', ....),
);

// Get the route by name
$route = $router->getRoute('page');

// Replace the default URI handler with a custom encoding function for path segments
$route = $route->withUriHandler(
    $route->getUriHandler()->withPathSegmentEncoder(
      static fn(string $segment): string => \rawurlencode($segment),
    ),
);

// Generate the URI
$uri = $route->uri(['path' => 'некоторый-путь']);

// Generates: /page/%D0%BD%D0%B5%D0%BA%D0%BE%D1%82%D0%BE%D1%80%D1%8B%D0%B9-%D0%BF%D1%83%D1%82%D1%8C
```

Пользовательскую фабрику `Spiral\Router\UriHandler` можно настроить в контейнере через загрузчик:

```php app/src/Application/Bootloader/AppBootloader.php
<?php

declare(strict_types=1);

namespace App\Application\Bootloader;

use Psr\Http\Message\UriFactoryInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\UriHandler;

final class AppBootloader extends Bootloader
{
    public function defineSingletons(): array
    {
        return [
            UriHandler::class => static function (UriFactoryInterface $uriFactory) {
                return (new UriHandler($uriFactory))->withPathSegmentEncoder(
                    static fn(string $segment): string => \rawurlencode($segment)
                );
            },
        ];
    }
}
```

## События

| Событие                             | Описание                                               |
|-------------------------------------|--------------------------------------------------------|
| Spiral\Router\Event\Routing       | Вызывается `до` сопоставления маршрута                 |
| Spiral\Router\Event\RouteMatched  | Вызывается после успешного сопоставления маршрута      |
| Spiral\Router\Event\RouteNotFound | Вызывается, если маршрут не найден                     |

> **Примечание**
> Подробнее о диспетчеризации событий читайте в разделе [События](../advanced/events.md).
