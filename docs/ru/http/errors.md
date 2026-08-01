# HTTP — Страницы ошибок

HTTP-компонент Spiral предоставляет middleware обработки ошибок, которое преобразует исключения в корректные HTTP-ответы.
В режиме отладки оно может показывать подробную информацию, а в производственном окружении — безопасные страницы без
раскрытия внутреннего состояния приложения.

## Установка

Стандартный каркас приложения уже содержит необходимые загрузчики. В альтернативной сборке добавьте
`Spiral\Bootloader\Http\ErrorHandlerBootloader`.

:::: tabs

::: tab С помощью метода

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Bootloader\Http\ErrorHandlerBootloader::class,
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
    \Spiral\Bootloader\Http\ErrorHandlerBootloader::class,
    // ...
];
```

Подробнее о загрузчиках читайте в разделе [Фреймворк — Загрузчики](../framework/bootloaders.md).
:::

::::

Загрузчик регистрирует `Spiral\Http\Middleware\ErrorHandlerMiddleware`. Убедитесь, что middleware присутствует в
глобальном списке и выполняется раньше остальных компонентов, способных выбрасывать исключения.

```php app/src/Application/Bootloader/RoutesBootloader.php
use Spiral\Http\Middleware\ErrorHandlerMiddleware;

protected function globalMiddleware(): array
{
    return [
        ErrorHandlerMiddleware::class,
        // ...
    ];
}
```

## HTTP-исключения

Для возврата определённого HTTP-кода можно выбросить исключение, реализующее
`Spiral\Http\Exception\HttpExceptionInterface`.

Встроенные исключения находятся в пространстве имён `Spiral\Http\Exception\ClientException`.

```php
use Spiral\Http\Exception\ClientException\NotFoundException;

throw new NotFoundException('User not found');
```

Такое исключение будет преобразовано в ответ со статусом `404`.

Другие распространённые исключения:

| Исключение                                                         | Код |
|--------------------------------------------------------------------|-----|
| `Spiral\Http\Exception\ClientException\BadRequestException`       | 400 |
| `Spiral\Http\Exception\ClientException\UnauthorizedException`     | 401 |
| `Spiral\Http\Exception\ClientException\ForbiddenException`        | 403 |
| `Spiral\Http\Exception\ClientException\NotFoundException`         | 404 |
| `Spiral\Http\Exception\ClientException\MethodNotAllowedException` | 405 |
| `Spiral\Http\Exception\ClientException\ConflictException`         | 409 |
| `Spiral\Http\Exception\ClientException\UnprocessableEntityException` | 422 |

## Пользовательское HTTP-исключение

Создайте собственный класс и реализуйте `HttpExceptionInterface` либо унаследуйтесь от базового клиентского исключения.

```php
namespace App\Exception;

use Spiral\Http\Exception\ClientException;

final class SubscriptionRequiredException extends ClientException
{
    protected int $statusCode = 402;
}
```

Теперь его можно выбросить из контроллера или сервиса:

```php
throw new SubscriptionRequiredException('Subscription is required');
```

## Рендеринг страниц ошибок

Страницы ошибок создаются через интерфейс `Spiral\Http\ErrorHandler\RendererInterface`. Рендерер получает исключение,
код ответа и объект запроса, после чего формирует `ResponseInterface`.

Для пользовательского отображения можно реализовать собственный рендерер:

```php
namespace App\Application\Exception;

use Psr\Http\Message\ResponseFactoryInterface;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Spiral\Http\ErrorHandler\RendererInterface;

final class JsonErrorRenderer implements RendererInterface
{
    public function __construct(
        private readonly ResponseFactoryInterface $responseFactory,
    ) {
    }

    public function render(
        ServerRequestInterface $request,
        \Throwable $exception,
        int $status
    ): ResponseInterface {
        $response = $this->responseFactory
            ->createResponse($status)
            ->withHeader('Content-Type', 'application/json');

        $response->getBody()->write((string)\json_encode([
            'error' => $exception->getMessage(),
            'status' => $status,
        ], JSON_THROW_ON_ERROR));

        return $response;
    }
}
```

Зарегистрируйте рендерер в загрузчике обработки ошибок:

```php app/src/Application/Bootloader/AppBootloader.php
use App\Application\Exception\JsonErrorRenderer;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Bootloader\Http\ErrorHandlerBootloader;

final class AppBootloader extends Bootloader
{
    public function init(ErrorHandlerBootloader $errors): void
    {
        $errors->setRenderer(JsonErrorRenderer::class);
    }
}
```

## Представления ошибок

Для HTML-приложения можно создать отдельные представления для кодов ошибок, например:

```text
app/views/exception/404.dark.php
app/views/exception/500.dark.php
```

Рендерер выбирает наиболее подходящий шаблон для текущего статуса. Если отдельного шаблона нет, используется общее
представление ошибки.

Пример страницы `404`:

```php app/views/exception/404.dark.php
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <title>Page not found</title>
</head>
<body>
    <h1>Page not found</h1>
</body>
</html>
```

## Режим отладки

При `DEBUG=true` обработчик может отображать трассировку, исходный код и сведения о запросе. В производственном
окружении установите:

```dotenv .env
DEBUG=false
VERBOSITY_LEVEL=basic
```

Это предотвращает раскрытие конфиденциальных сведений пользователю.

> **Смотрите также**
> Общая настройка рендереров, репортёров, Sentry и уровней подробности описана в разделе
> [Основы — Обработка ошибок](../basics/errors.md).

## Ошибки маршрутизации

Если маршрутизатор не нашёл подходящий маршрут, возникает `Spiral\Router\Exception\RouteNotFoundException`. Обработчик
ошибок преобразует его в HTTP-ответ `404`.

Если маршрут существует, но не допускает используемый HTTP-метод, возвращается `405 Method Not Allowed`.

## Обработка исключений в middleware

Пользовательское middleware также может перехватывать исключения и преобразовывать их в ответы:

```php
use Psr\Http\Message\ResponseFactoryInterface;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;

final class ApiErrorMiddleware implements MiddlewareInterface
{
    public function __construct(
        private readonly ResponseFactoryInterface $responses,
    ) {
    }

    public function process(
        ServerRequestInterface $request,
        RequestHandlerInterface $handler
    ): ResponseInterface {
        try {
            return $handler->handle($request);
        } catch (\DomainException $e) {
            $response = $this->responses
                ->createResponse(422)
                ->withHeader('Content-Type', 'application/json');

            $response->getBody()->write((string)\json_encode([
                'error' => $e->getMessage(),
            ], JSON_THROW_ON_ERROR));

            return $response;
        }
    }
}
```

Размещайте специализированное middleware после общего `ErrorHandlerMiddleware`, если оно должно самостоятельно
обрабатывать определённый тип исключений.
