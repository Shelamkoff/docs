# HTTP — Документация OpenAPI

Spiral позволяет создавать документацию [OpenAPI](https://www.openapis.org/), используя атрибуты PHP. Это удобный
способ документировать HTTP API прямо в коде рядом с контроллерами и DTO.

Компонент основан на пакетах:

- [zircote/swagger-php](https://github.com/zircote/swagger-php) — генерация спецификации OpenAPI из атрибутов PHP;
- [swagger-api/swagger-ui](https://github.com/swagger-api/swagger-ui) — интерактивный веб-интерфейс документации.

## Установка

Установите пакеты:

```terminal
composer require zircote/swagger-php
composer require --dev spiral-packages/swagger-ui
```

> **Предупреждение**
> Не используйте версии `spiral-packages/swagger-ui` старше `^2.0` с Spiral 3.0.

После установки добавьте загрузчик `Spiral\OpenApi\Bootloader\OpenApiBootloader` в список загрузчиков приложения.

:::: tabs

::: tab С помощью метода

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\OpenApi\Bootloader\OpenApiBootloader::class,
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
    \Spiral\OpenApi\Bootloader\OpenApiBootloader::class,
    // ...
];
```

Подробнее о загрузчиках читайте в разделе [Фреймворк — Загрузчики](../framework/bootloaders.md).
:::

::::

Загрузчик автоматически обнаруживает классы с атрибутами OpenAPI и создаёт спецификацию.

## Конфигурация

Создайте файл `app/config/openapi.php`:

```php app/config/openapi.php
use OpenApi\Generator;

return [
    /**
     * -------------------------------------------------------------------------
     *  Documentation path
     * -------------------------------------------------------------------------
     */
    'path' => '/api/openapi.json',

    /**
     * -------------------------------------------------------------------------
     *  Generator config
     * -------------------------------------------------------------------------
     */
    'generator' => [
        'version' => '3.1.0',
        'scanOptions' => [
            'exclude' => [],
            'pattern' => null,
            'analysis' => null,
            'processors' => [],
            'validate' => true,
        ],
    ],
];
```

### Путь к спецификации

Параметр `path` задаёт URL, по которому будет доступна спецификация OpenAPI в формате JSON.

```php
'path' => '/api/openapi.json',
```

### Версия OpenAPI

```php
'generator' => [
    'version' => '3.1.0',
],
```

Поддерживаемые версии зависят от установленного пакета `zircote/swagger-php`.

### Параметры сканирования

Секция `scanOptions` передаётся генератору Swagger PHP:

- `exclude` — каталоги и файлы, исключаемые из сканирования;
- `pattern` — шаблон имён сканируемых файлов;
- `analysis` — пользовательский объект анализа;
- `processors` — дополнительные процессоры;
- `validate` — проверка созданной спецификации.

## Базовое описание API

Создайте класс с общей информацией о приложении:

```php app/src/Application/OpenApi/Documentation.php
namespace App\Application\OpenApi;

use OpenApi\Attributes as OA;

#[OA\Info(
    version: '1.0.0',
    title: 'My API',
    description: 'Application API documentation',
)]
#[OA\Server(
    url: 'https://api.example.com',
    description: 'Production server',
)]
final class Documentation
{
}
```

Атрибут `OA\Info` обязателен. Без него генератор не сможет сформировать корректную спецификацию.

## Документирование контроллеров

Атрибуты OpenAPI можно добавлять непосредственно к методам контроллера.

```php app/src/Endpoint/Web/UserController.php
namespace App\Endpoint\Web;

use OpenApi\Attributes as OA;
use Spiral\Router\Annotation\Route;

final class UserController
{
    #[Route(route: '/api/users/<id>', name: 'users.show', methods: 'GET')]
    #[OA\Get(
        path: '/api/users/{id}',
        operationId: 'getUser',
        summary: 'Get user by ID',
        tags: ['Users'],
        parameters: [
            new OA\Parameter(
                name: 'id',
                description: 'User ID',
                in: 'path',
                required: true,
                schema: new OA\Schema(type: 'integer'),
            ),
        ],
        responses: [
            new OA\Response(
                response: 200,
                description: 'User found',
                content: new OA\JsonContent(ref: '#/components/schemas/User'),
            ),
            new OA\Response(
                response: 404,
                description: 'User not found',
            ),
        ],
    )]
    public function show(int $id): array
    {
        // ...
    }
}
```

> **Примечание**
> Шаблон маршрута Spiral и путь OpenAPI имеют разный синтаксис: Spiral использует `<id>`, а OpenAPI — `{id}`.

## Схемы данных

DTO или сущности можно описывать атрибутом `OA\Schema`:

```php app/src/Endpoint/Web/Dto/User.php
namespace App\Endpoint\Web\Dto;

use OpenApi\Attributes as OA;

#[OA\Schema(
    schema: 'User',
    title: 'User',
    required: ['id', 'name', 'email'],
)]
final readonly class User
{
    public function __construct(
        #[OA\Property(example: 42)]
        public int $id,

        #[OA\Property(example: 'John Doe')]
        public string $name,

        #[OA\Property(format: 'email', example: 'john@example.com')]
        public string $email,
    ) {
    }
}
```

После этого схема доступна по ссылке `#/components/schemas/User`.

## Тело запроса

Пример документации JSON-тела запроса:

```php
#[OA\Post(
    path: '/api/users',
    operationId: 'createUser',
    summary: 'Create user',
    tags: ['Users'],
    requestBody: new OA\RequestBody(
        required: true,
        content: new OA\JsonContent(
            required: ['name', 'email'],
            properties: [
                new OA\Property(property: 'name', type: 'string'),
                new OA\Property(property: 'email', type: 'string', format: 'email'),
            ],
        ),
    ),
    responses: [
        new OA\Response(response: 201, description: 'User created'),
        new OA\Response(response: 422, description: 'Validation error'),
    ],
)]
public function create(): array
{
    // ...
}
```

## Безопасность

Опишите схему аутентификации в общем классе документации:

```php
#[OA\SecurityScheme(
    securityScheme: 'bearerAuth',
    type: 'http',
    scheme: 'bearer',
    bearerFormat: 'JWT',
)]
final class Documentation
{
}
```

Затем укажите её для защищённой операции:

```php
#[OA\Get(
    path: '/api/profile',
    security: [['bearerAuth' => []]],
    responses: [
        new OA\Response(response: 200, description: 'Profile'),
        new OA\Response(response: 401, description: 'Unauthorized'),
    ],
)]
public function profile(): array
{
    // ...
}
```

## Swagger UI

Пакет `spiral-packages/swagger-ui` предоставляет интерактивный интерфейс Swagger UI.

Добавьте загрузчик:

:::: tabs

::: tab С помощью метода

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\OpenApi\Bootloader\OpenApiBootloader::class,
        \Spiral\SwaggerUI\Bootloader\SwaggerUIBootloader::class,
        // ...
    ];
}
```

:::

::: tab С помощью константы

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\OpenApi\Bootloader\OpenApiBootloader::class,
    \Spiral\SwaggerUI\Bootloader\SwaggerUIBootloader::class,
    // ...
];
```

:::

::::

По умолчанию интерфейс будет доступен по адресу `/api/docs` и загрузит спецификацию из настроенного OpenAPI-маршрута.

Настройки Swagger UI можно переопределить в `app/config/swagger-ui.php`:

```php app/config/swagger-ui.php
return [
    'path' => '/api/docs',
    'title' => 'API Documentation',
    'openapiUrl' => '/api/openapi.json',
];
```

> **Предупреждение**
> В производственном окружении документацию можно ограничить аутентификацией или полностью отключить.

## Генерация спецификации

Спецификацию можно получить HTTP-запросом к настроенному маршруту:

```terminal
curl http://127.0.0.1:8080/api/openapi.json
```

Также её можно сформировать средствами `zircote/swagger-php` из командной строки:

```terminal
./vendor/bin/openapi app/src -o openapi.json
```

> **Смотрите также**
> Полный перечень атрибутов и примеров доступен в документации
> [Swagger PHP](https://zircote.github.io/swagger-php/).