# HTTP — Жизненный цикл запроса

В отличие от большинства PHP-фреймворков, обработка HTTP-запроса начинается за пределами приложения — в сервере
приложений [RoadRunner](https://roadrunner.dev).

![Жизненный цикл запроса](https://user-images.githubusercontent.com/773481/181182150-8cc2b6c4-2b50-4e85-afd7-e5c2d1c98b2c.png)

> **Примечание**
> Ответ проходит этот путь в обратном направлении.

## PSR

Spiral основан на наборе стандартов, обеспечивающих совместимость с другими фреймворками, маршрутизаторами, middleware
и компонентами. В HTTP-слое используются следующие стандарты:

- [PSR-7: интерфейсы HTTP-сообщений](https://www.php-fig.org/psr/psr-7/);
- [PSR-15: обработчики серверных HTTP-запросов](https://www.php-fig.org/psr/psr-15/);
- [PSR-17: HTTP-фабрики](https://www.php-fig.org/psr/psr-17/).

## Описание потока

Пользовательский запрос поступает серверу приложений RoadRunner. Сервер пропускает его через несколько слоёв
middleware, часть которых обслуживает статические файлы или реализует доменную логику.

После завершения обработки middleware запрос `net/http` преобразуется в формат `PSR-7` и передаётся первому доступному
PHP-воркеру.

Воркер обрабатывает запрос с помощью компонента `spiral/http` и ядра `Spiral\Http\Http`. Ядро пропускает объект
PSR-7-запроса (`Psr\Http\Message\ServerRequestInterface`) через набор middleware, совместимых с PSR-15.

После завершения обработки middleware фреймворк создаёт для объекта запроса [область IoC](../framework/scopes.md).
Благодаря этому PSR-7-запрос можно использовать как обычный глобальный объект, хотя технически он существует только во
время обработки конкретного пользовательского запроса.

Затем запрос передаётся выбранному обработчику PSR-15 — по умолчанию `spiral/router`. Обработчик должен сформировать
ответ, который будет отправлен пользователю через все слои middleware в обратном порядке.

> **Примечание**
> Маршрутизатор Spiral позволяет связать с каждым маршрутом отдельный набор middleware.

## Ручной вызов HTTP-ядра

HTTP-ядро можно вызвать непосредственно внутри приложения. Это полезно в тестах или при запуске Spiral из другого
фреймворка. Для этого получите экземпляр `Spiral\Http\Http`:

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use Nyholm\Psr7\Uri;
use Psr\Http\Message\ServerRequestInterface;
use Spiral\Http\Http;

class HomeController implements SingletonInterface
{
    public function __construct(
        private readonly Http $http
    ) {
    }

    public function index(ServerRequestInterface $request): string
    {
        $response = $this->http->handle(
            $request->withUri(new Uri('/home/other')) // modify Uri of current request
        );

        return (string) $response->getBody(); // "other"
    }

    public function other(): string
    {
        return 'other';
    }
}
```

> **Примечание**
> Области IoC могут быть вложенными, поэтому функциональность продолжит работать корректно. Однако не все расширения
> допускают вложенность: например, вложенные сессии пока создавать нельзя.

## События

| Событие                             | Описание                                             |
|-------------------------------------|------------------------------------------------------|
| Spiral\Http\Event\RequestReceived | Вызывается при получении запроса                     |
| Spiral\Http\Event\RequestHandled  | Вызывается после успешной обработки запроса          |

> **Примечание**
> Подробнее о диспетчеризации событий читайте в разделе [События](../advanced/events.md).
