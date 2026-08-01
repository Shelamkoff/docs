# Основы — Обработка ошибок

Во время разработки неизбежно возникают ошибки и исключения. Их отладка может быть сложной и занимать много времени,
но это необходимая часть разработки. Spiral предоставляет инструменты для обработки исключений, поиска причин ошибок
и настройки представления диагностической информации.

В этом разделе описаны возможности Spiral для обработки, отображения и настройки исключений.

## Обработчик исключений

Spiral предоставляет надёжный механизм обработки исключений в классе `Spiral\Exceptions\ExceptionHandler`.

Он обрабатывает как глобальные ошибки, так и ошибки времени выполнения, предоставляя приложению единый структурированный
подход.

#### Основные возможности

- **Отображение исключений:** представление исключений в HTML, JSON, обычном тексте и других форматах.
- **Отправка отчётов:** передача исключений во внешние сервисы, например [Sentry](#sentry-integration) или
  [хранилище S3](#cloud-storage-reporter).
- **Глобальная обработка ошибок:** обработка фатальных ошибок и ошибок завершения работы.
- **Расширяемость:** возможность подключать собственные рендереры и репортёры.

### Настройка обработчика исключений

Если приложению требуется особая стратегия обработки ошибок, стандартный обработчик можно заменить собственной
реализацией.

Сначала создайте класс, наследующий `Spiral\Exceptions\ExceptionHandler`:

```php app/src/Application/Exception/Handler.php
<?php

declare(strict_types=1);

namespace App\Application\Exception;

use Spiral\Exceptions\ExceptionHandler;
use Throwable;

final class Handler extends ExceptionHandler
{
    // ...
}
```

Затем укажите класс в файле `app.php`:

```php app.php
use App\Application\Kernel;
use App\Application\Exception\Handler;

// ...

$app = Kernel::create(
    directories: ['root' => __DIR__],
    exceptionHandler: Handler::class, // <--
)->run();

// ...
```

При инициализации обработчик вызывает метод `bootBasicHandlers`. Это одна из точек настройки, используемая для
регистрации основных рендереров и репортёров.

```php app/src/Application/Exception/Handler.php
final class Handler extends ExceptionHandler
{
    protected function bootBasicHandlers(): void
    {
        parent::bootBasicHandlers();
        
        // Register your renderers and reporters here
        // $this->addRenderer(new MyRenderer());
        // $this->addRenderer(new MyReporter());
    }
}
```

Обработчик также подходит для исключений, возникающих при начальной загрузке приложения. Например, чтобы не отправлять
отчёты об определённых исключениях, переопределите метод `report`.

```php app/src/Application/Exception/Handler.php
<?php

declare(strict_types=1);

namespace App\Application\Exception;

use Spiral\Exceptions\ExceptionHandler;
use Spiral\Http\Exception\ClientException;

final class Handler extends ExceptionHandler
{
    /**
     * @var class-string<\Throwable>[]
     */
    private array $nonReportableExceptions = [
        ClientException::class,
        // ...
    ];

    public function report(\Throwable $exception): void
    {
        foreach ($this->nonReportableExceptions as $nonReportableException) {
            if ($exception instanceof $nonReportableException) {
                return;
            }
        }

        parent::report($exception);
    }
}
```

## Отображение исключений

Spiral определяет нужный рендерер по формату. Формат может зависеть от окружения: например, `cli` для консольного
приложения или `http` для HTTP-запроса. Благодаря этому в разных контекстах можно регистрировать и использовать разные
рендереры.

### Как это работает

Иногда ошибку необходимо отобразить в специальном формате, например JSON для API. Для этого выполните следующие шаги.

1. **Создайте рендерер**

Пример реализации JSON-рендерера:

```php app/src/Application/Exception/Renderer/JsonRenderer.php
<?php

namespace Spiral\YiiErrorHandler;

use Spiral\Exceptions\ExceptionRendererInterface;
use Spiral\Exceptions\Verbosity;
use Yiisoft\ErrorHandler\Renderer\JsonRenderer as YiiJsonRenderer;
use Yiisoft\ErrorHandler\ThrowableRendererInterface;

final class JsonRenderer implements ExceptionRendererInterface
{
    public const FORMATS = ['application/json', 'json'];

    public function __construct(
        private readonly ?ThrowableRendererInterface $renderer = new YiiJsonRenderer()
    ) {
    }

    public function render(
        \Throwable $exception,
        ?Verbosity $verbosity = Verbosity::BASIC,
        string $format = null,
    ): string {
        if ($verbosity >= Verbosity::VERBOSE) {
            return (string)$this->renderer->renderVerbose($exception);
        }

        return (string)$this->renderer->render($exception);
    }

    public function canRender(string $format): bool
    {
        return \in_array($format, self::FORMATS, true);
    }
}
```

> **Примечание**
> `Spiral\YiiErrorHandler\JsonRenderer` входит в пакет `spiral-packages/yii-error-handler-bridge`.

2. **Зарегистрируйте рендерер**

Зарегистрируйте пользовательский рендерер через загрузчик:

```php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Exceptions\ExceptionHandler;
use Spiral\YiiErrorHandler\JsonRenderer;

final class ExceptionHandlerBootloader extends Bootloader
{
    public function init(ExceptionHandler $handler): void
    {
        $handler->addRenderer(new JsonRenderer());
    }
}
```

> **Предупреждение**
> Не забудьте добавить загрузчик в начало списка загрузчиков в `app/src/Application/Kernel.php`.

:::: tabs

::: tab С помощью метода

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \App\Application\Bootloader\ExceptionHandlerBootloader::class,
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
    \App\Application\Bootloader\ExceptionHandlerBootloader::class,
    // ...
];
```

Подробнее о загрузчиках читайте в разделе [Фреймворк — Загрузчики](../framework/bootloaders.md).
:::

::::

3. **Используйте рендерер**

Для обработки исключений в веб-приложении можно создать middleware, которое перехватывает исключения и использует этот
рендерер только при наличии в запросе определённого заголовка, например `Accept=application/json`. Это позволяет точно
управлять способом отображения ошибки в зависимости от формата, запрошенного клиентом.

```php
namespace App\Endpoint\Web\Middleware;

use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Message\ResponseInterface as Response;
use Psr\Http\Message\ServerRequestInterface as Request;
use Psr\Http\Server\RequestHandlerInterface as Handler;
use Psr\Http\Message\ResponseFactoryInterface;
use Spiral\Exceptions\ExceptionRendererInterface;
use Spiral\Http\Exception\ClientException;
use Spiral\Router\Exception\RouterException;

class ErrorHandlerMiddleware implements MiddlewareInterface
{
    public function __construct(
        private readonly ExceptionRendererInterface $renderer,
        private readonly ResponseFactoryInterface $responseFactory,
    ) {
    }

    public function process(Request $request, Handler $handler): Response
    {
        try {
            return $handler->handle($request);
        } catch (ClientException|RouterException $e) {
            $code = $e instanceof ClientException ? $e->getCode() : 404;
        } catch (\Throwable $e) {
            $code = 500;
        }
        
        $response = $this->responseFactory->createResponse($code);
        $response->getBody()->write(
            (string) $this->renderer->render(
                exception: $e,
                format: $request->getHeaderLine('Accept') ?? 'application/json'
            )
        );

        return $response;
    }
}
```

В разных окружениях можно использовать разные рендереры: консольный для приложений командной строки или JSON-рендерер
для ответов API. Кроме того, рендереры можно выбирать по формату.

### Встроенные рендереры

Spiral предоставляет следующие рендереры:

| Рендерер                                     | Форматы                                         |
|----------------------------------------------|-------------------------------------------------|
| `Spiral\Exceptions\Renderer\ConsoleRenderer` | `console`, `cli`                                |
| `Spiral\Exceptions\Renderer\JsonRenderer`    | `application/json`, `json`                      |
| `Spiral\Exceptions\Renderer\PlainRenderer`   | `text/plain`, `text`, `plain`, `cli`, `console` |

В некоторых случаях, например при `DEBUG=true`, удобнее отображать красивую страницу ошибки с подсветкой кода. Для
этого можно использовать [filp/whoops](https://github.com/filp/whoops) или
[yiisoft/error-handler](https://github.com/spiral-packages/yii-error-handler-bridge).

### Рендерер ошибок Yii

Yii Error Handler — пакет интеграции Spiral с обработчиками ошибок Yii.

![Снимок экрана](https://user-images.githubusercontent.com/773481/215085868-a7228f6c-1be0-460d-b910-85fa2cd1195b.png)

#### Установка

Установите компонент:

```terminal
composer require spiral-packages/yii-error-handler-bridge
```

После установки зарегистрируйте загрузчик пакета.

:::: tabs

::: tab С помощью метода

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\YiiErrorHandler\Bootloader\YiiErrorHandlerBootloader::class,
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
    \Spiral\YiiErrorHandler\Bootloader\YiiErrorHandlerBootloader::class,
    // ...
];
```

Подробнее о загрузчиках читайте в разделе [Фреймворк — Загрузчики](../framework/bootloaders.md).
:::

::::

`YiiErrorHandlerBootloader` регистрирует при инициализации все доступные рендереры. При необходимости можно
зарегистрировать только определённые рендереры.

#### Встроенные рендереры

Пакет интеграции предоставляет несколько рендереров:

- `HtmlRenderer` отображает страницу ошибки в HTML;
- `JsonRenderer` отображает ошибку в JSON и подходит для API-запросов;
- `PlainTextRenderer` отображает ошибку обычным текстом.

### Уровни подробности

Уровень подробности управляет количеством сведений, отображаемых при рендеринге исключения. Задайте
`VERBOSITY_LEVEL` в файле `.env`:

```dotenv .env
# Verbosity level
VERBOSITY_LEVEL=verbose # basic, verbose, debug
```

Возможные значения определены перечислением `Spiral\Exceptions\Verbosity`.

#### basic или 0

Отображаются только основные сведения об исключении:

```output
[Spiral\Router\Exception\RouteNotFoundException] 
Unable to route `http://127.0.0.1`. in vendor/spiral/framework/src/Router/src/Router.php:75
```

#### verbose или 1

Отображаются более подробные сведения:

```output

[Spiral\Router\Exception\RouteNotFoundException]
Unable to route `http://127.0.0.1`. in vendor/spiral/framework/src/Router/src/Router.php:75

 1. Spiral\Router\Router->Spiral\Router\{closure}() at vendor/spiral/framework/src/Router/src/Router.php:75
 2. Spiral\Router\Router->Spiral\Router\{closure}()
 3. ReflectionFunction->invokeArgs() at vendor/spiral/framework/src/Core/src/Internal/Invoker.php:73
 4. ...
```

#### debug или 2

Отображается максимально подробная информация:

```output
[Spiral\Router\Exception\RouteNotFoundException]
Unable to route `http://127.0.0.1`. in vendor/spiral/framework/src/Router/src/Router.php:75

 1. Spiral\Router\Router->Spiral\Router\{closure}() at vendor/spiral/framework/src/Router/src/Router.php:75
   73                 if ($route === null) {
   74                     $this->eventDispatcher?->dispatch(new RouteNotFound($request));
>  75                     throw new RouteNotFoundException($request->getUri());
   76                 }
   77 

 2. ...
```

<hr />

## Отправка отчётов об исключениях

Репортёры позволяют отслеживать ошибки приложения. Они выполняют две основные задачи:

- сохраняют сведения о проблеме в файл, чтобы их можно было проанализировать позднее;
- отправляют отчёты внешним сервисам, например [Sentry](https://sentry.io), которые предоставляют дополнительную
  информацию об ошибках.

Например, сайт может работать неправильно из-за отсутствующего файла или проблемы с базой данных. Репортёры помогают
собирать и обрабатывать сведения о таких ошибках.

### Как работают репортёры

#### 1. Реализация `ExceptionReporterInterface` или использование встроенного репортёра

Создайте класс, например `CustomReporter`, реализующий `Spiral\Exceptions\ExceptionReporterInterface`. Он определяет,
что необходимо сделать при возникновении исключения.

> **Примечание**
> Подробнее о встроенных репортёрах читайте в разделе [Доступные репортёры](#доступные-репортёры).

```php app/src/Application/Exception/Reporter/CustomReporter.php
namespace App\Application\Exception\Reporter;

final class CustomReporter implements ExceptionReporterInterface
{
    public function __construct(
        private readonly LoggerInterface $logger,
    ) {}

    public function report(\Throwable $exception): void
    {
        // Store exception information in a file or send it to an external service
        $this->logger->error($exception->getMessage(), ['exception' => $exception]);
    }
}
```

#### 2. Регистрация репортёра

Репортёр регистрируется в `ExceptionHandler` аналогично рендереру: вызовите `addReporter` и передайте экземпляр класса,
реализующего `Spiral\Exceptions\ExceptionReporterInterface`.

```php app/src/Application/Bootloader/ExceptionHandlerBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Exceptions\ExceptionHandler;
use App\Application\Exception\Reporter\CustomReporter;

final class ExceptionHandlerBootloader extends Bootloader
{
    public function init(ExceptionHandler $handler): void
    {
        $handler->addReporter(new CustomReporter());
    }
}
```

#### 3. Использование репортёра в коде

После регистрации репортёр можно использовать в местах, где ожидается исключение.

Например, если при выполнении `PingSiteJob` сайт недоступен, исключение перехватывается и передаётся репортёру.

```php
<?php

declare(strict_types=1);

namespace App\Job;

use Spiral\Exceptions\ExceptionReporterInterface;

final class PingSiteJob
{
    public function __construct(
        private PingClient $client,
        private ExceptionReporterInterface $reporter,
    ) {
    }

    public function handle(string $url): void
    {
        try {
            $this->client->ping($url);
        } catch (\Throwble $e) {
            $this->reporter->report($e);
        }
    }
}
```

> **Примечание**
> Репортёр передаст исключение всем зарегистрированным репортёрам.

### Доступные репортёры

Spiral предоставляет встроенные репортёры `Spiral\Exceptions\Reporter\LoggerReporter`,
`Spiral\Exceptions\Reporter\FileReporter` и `Spiral\Exceptions\Reporter\StorageReporter`.

#### LoggerReporter

`Spiral\Exceptions\Reporter\LoggerReporter` включён по умолчанию и записывает исключения через зарегистрированный в
приложении логгер. Это помогает отслеживать и анализировать ошибки во времени.

#### FileReporter

`Spiral\Exceptions\Reporter\FileReporter` также включён по умолчанию. Он сохраняет подробные сведения об исключении в
файл-снимок в каталоге `runtime/snapshots`.

#### Репортёр облачного хранилища

В приложениях без состояния локальное хранение снимков исключений может быть неудобно. Интеграция с компонентом
`spiral/storage` позволяет сохранять снимки непосредственно в облачное хранилище, например **S3**.

**Преимущества:**

1. **Простое хранение.** Снимки можно без сложной дополнительной инфраструктуры сохранять прямо в S3.
2. **Поддержка приложений без состояния.** Решение специально подходит для stateless-приложений и упрощает их
   развёртывание.
3. **Надёжность.** Снимки безопасно хранятся в S3 и доступны для последующего анализа.

`Spiral\Exceptions\Reporter\StorageReporter` позволяет сохранять подробные сведения об исключении в виде снимка.

**Для использования репортёра:**

1. Настройте компонент `spiral/storage`. Подробнее читайте в разделе
   [Компоненты — Хранилища и облачное распространение](../advanced/storage.md).
2. Зарегистрируйте `Spiral\Bootloader\StorageSnapshotsBootloader`.
3. Укажите нужный bucket переменной окружения `SNAPSHOTS_BUCKET`.
4. Зарегистрируйте `Spiral\Exceptions\Reporter\StorageReporter` в `Spiral\Exceptions\ExceptionHandler`.

## Интеграция с Sentry

Spiral предоставляет пакет интеграции с сервисом [Sentry](https://sentry.io). Ниже описаны его подключение и настройка.

### Установка

1. Установите компонент интеграции:

```terminal
composer require spiral/sentry-bridge
```

2. Зарегистрируйте загрузчик `Spiral\Sentry\Bootloader\SentryReporterBootloader`.

:::: tabs

::: tab С помощью метода

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Sentry\Bootloader\SentryReporterBootloader::class,
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
    \Spiral\Sentry\Bootloader\SentryReporterBootloader::class,
    // ...
];
```

Подробнее о загрузчиках читайте в разделе [Фреймворк — Загрузчики](../framework/bootloaders.md).
:::

::::

Загрузчик зарегистрирует `Spiral\Sentry\SentryReporter` в `Spiral\Exceptions\ExceptionHandler`.

### Конфигурация

Установите переменной окружения `SENTRY_DSN` значение DSN проекта Sentry.

```dotenv .env
SENTRY_DSN=https://...
```

Начиная с версии **2.2**, репортёр также поддерживает дополнительные переменные окружения:

- `SENTRY_DSN` — Data Source Name проекта Sentry;
- `SENTRY_SAMPLE_RATE` — доля событий для выборки, например `0.4`;
- `SENTRY_TRACES_SAMPLE_RATE` — доля трассировок для выборки, например `1.0`;
- `SENTRY_SEND_DEFAULT_PII` — отправка стандартных персональных данных: `true` или `false`;
- `SENTRY_ENVIRONMENT` — окружение, например `develop`; вместо него можно использовать `APP_ENV`;
- `SENTRY_RELEASE` — версия выпуска, например `1.0.0`; вместо неё можно использовать `APP_VERSION`.

Пример:

```dotenv .env
SENTRY_DSN=https://...
SENTRY_SAMPLE_RATE=0.4
SENTRY_TRACES_SAMPLE_RATE=1.0
SENTRY_SEND_DEFAULT_PII=false

SENTRY_ENVIRONMENT=develop
SENTRY_RELEASE=1.0.0
# or
APP_ENV=develop
APP_VERSION=1.0.0
```

Репортёр также можно настроить через файл `config/sentry.php`:

```php config/sentry.php
return [
  'dsn' => 'http://...',
  'environment' => 'develop',
  'release' => '1.0.0',
  'sample_rate' => 1.0,
  'traces_sample_rate' => null,
  'send_default_pii' => true,
];
```

### Интеграции Sentry — начиная с версии **2.2**

Начиная с версии **2.2**, поддерживаются
[интеграции Sentry](https://docs.sentry.io/platforms/php/integrations/).

Интеграции приложения можно регистрировать через `Spiral\Sentry\Bootloader\ClientBootloader`.

**Пример регистрации пользовательской интеграции:**

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Sentry\Bootloader\ClientBootloader;

use Spiral\Boot\Bootloader\Bootloader;

final class AppBootloader extends Bootloader
{
    public function init(ClientBootloader $sentry): void
    {
        $sentry->addIntegration(new ExceptionContextIntegration());
    }
}
```

#### Интеграция HTTP-запроса

Sentry автоматически собирает сведения о текущем запросе через встроенную интеграцию
`Sentry\Integration\RequestIntegration`. Также доступно middleware `Spiral\Sentry\Http\SetRequestIpMiddleware`,
необходимое для сбора IP-адресов пользователей при включённом `send_default_pii`. Это необязательная возможность для
сценариев, где требуются подробные пользовательские данные.

> **Примечание**
> Подробнее о middleware читайте в разделе [HTTP — Маршрутизация](../http/routing.md#add-middleware).

### Доступные связи контейнера — начиная с версии **2.2**

Для более глубокой интеграции и управления доступны следующие связи контейнера:

- `Sentry\Options` — контейнер конфигурации клиента Sentry;
- `Sentry\State\HubInterface` — доступ к состоянию и управлению контекстом Sentry;
- `Sentry\ClientInterface` — прямое взаимодействие с клиентом Sentry.

Эти связи предоставляют детальное управление клиентом для расширенных сценариев.

### Передача дополнительных данных

Чтобы передавать текущее состояние приложения, например журналы или данные PSR-7-запроса, включите коллекторы
отладочной информации. Они собирают сведения о текущем состоянии приложения.

При возникновении исключения репортёр Sentry запрашивает `Spiral\Debug\StateInterface` из IoC-контейнера. При создании
объект заполняется сведениями зарегистрированных коллекторов.

> **Предупреждение**
> Будьте осторожны при прямом запросе `Spiral\Debug\StateInterface` из контейнера. При каждом запросе создаётся новый
> объект, который нельзя заполнить вне коллекторов. Для добавления данных используйте коллекторы.

#### HTTP-коллектор

HTTP-коллектор позволяет передать Sentry сведения о текущем запросе.

> **Примечание**
> Начиная с версии **2.2**, HTTP-коллектор можно не использовать: репортёр лучше собирает сведения через встроенную
> интеграцию `Sentry\Integration\RequestIntegration`.

Передаются следующие сведения:

- `method`;
- `url`;
- `headers`;
- `query params`;
- `request body`.

Чтобы включить HTTP-коллектор, зарегистрируйте `Spiral\Bootloader\Debug\HttpCollectorBootloader` перед
`SentryReporterBootloader`.

:::: tabs

::: tab С помощью метода

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Bootloader\Debug\HttpCollectorBootloader::class,
        \Spiral\Sentry\Bootloader\SentryReporterBootloader::class,
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
    \Spiral\Bootloader\Debug\HttpCollectorBootloader::class,
    \Spiral\Sentry\Bootloader\SentryReporterBootloader::class,
    // ...
];
```

Подробнее о загрузчиках читайте в разделе [Фреймворк — Загрузчики](../framework/bootloaders.md).
:::

::::

Затем зарегистрируйте middleware `Spiral\Debug\StateCollector\HttpCollector` в приложении.

> **Смотрите также**
> О регистрации middleware читайте в разделе
> [HTTP — Маршрутизация](../http/routing.md#add-middleware).

#### Коллектор журналов

Коллектор журналов передаёт в Sentry все полученные записи.

Чтобы включить его, зарегистрируйте S`piral\Bootloader\Debug\LogCollectorBootloader` перед `SentryBootaloder`.

:::: tabs

::: tab С помощью метода

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Bootloader\Debug\LogCollectorBootloader::class,
        \Spiral\Sentry\Bootloader\SentryReporterBootloader::class,
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
    \Spiral\Bootloader\Debug\LogCollectorBootloader::class,
    \Spiral\Sentry\Bootloader\SentryReporterBootloader::class,
    // ...
];
```

Подробнее о загрузчиках читайте в разделе [Фреймворк — Загрузчики](../framework/bootloaders.md).
:::

::::

#### Создание пользовательских коллекторов

Для специализированного сбора данных можно создать собственный коллектор. Он должен реализовывать
`Spiral\Debug\StateCollectorInterface`.

Пример SQL-коллектора:

```php app/src/Application/Debug/Collector/SqlCollector.php
namespace App\Application\Debug\Collector;

use Spiral\Logger\Event\LogEvent;
use Spiral\Debug\StateCollectorInterface;

final class SqlCollector implements StateCollectorInterface
{
    public function __construct(
        private readonly Database $db
    ) {
    }

    public function collect(\Spiral\Debug\StateInterface $state): void
    {
       foreach($this->db->getQueries() as $query) {
            $state->addLogEvent(new LogEvent(
                time: $query->getTime(),
                channel: 'sql',
                level: 'info',
                message: $query->getQuery(),
                context: $query->getParameters()
            ));
       }
    }
}
```

> **Предупреждение**
> В примере используется несуществующий класс `Database`; его необходимо реализовать самостоятельно.

Полезные методы объекта `Spiral\Debug\StateInterface`:

**Добавление тега**

Метод добавляет теги, связанные с текущей областью.

```php
$state->addTag('IP address', $currentRequest->getIpAddress());
$state->addTag('Environment', $env->get('APP_ENV'));
```

**Добавление переменной**

Метод добавляет дополнительные данные, связанные с текущей областью.

```php
$state->setVariable('query', $currentRequest->getQueryParams());
```

**Добавление записи журнала**

Метод добавляет запись журнала как breadcrumb текущей области.

```php
$state->addLogEvent(new \Spiral\Logger\Event\LogEvent(
    time: new \DateTimeImmutable(),
    channel: 'default',
    level: 'info',
    message: 'Something went wrong',
    context: ['foo' => 'bar']
));
```

#### Регистрация пользовательского коллектора

Коллектор можно зарегистрировать в загрузчике.

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Bootloader\DebugBootloader;
use App\Application\Exception\Reporter\CustomReporter;

final class AppBootloader extends Bootloader
{
    public function init(DebugBootloader $debug, SqlCollector $sqlCollector): void
    {
        $debug->addStateCollector($sqlCollector);
    }
}
```
