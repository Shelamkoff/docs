# Начало работы — Конфигурация

Spiral упрощает настройку приложения с помощью конфигурационных файлов. Все они находятся в каталоге `app/config` и
позволяют настраивать подключение к базе данных, хранилища кеша, очереди и другие компоненты.

Создавать все конфигурационные файлы самостоятельно не требуется: Spiral поставляется с настройками по умолчанию.
Основные параметры приложения также можно изменять с помощью переменных окружения.

> **Примечание**
> Кешировать конфигурационные файлы не требуется: они загружаются один раз при начальной загрузке приложения и не
> перечитываются до его перезапуска.

## Переменные окружения

Переменные окружения позволяют отделить конфигурацию приложения от его кода. В них удобно хранить конфиденциальные
данные, например учётные данные базы данных, ключи API и другие параметры, которые не следует жёстко задавать в коде.

Spiral интегрируется с [Dotenv](https://github.com/vlucas/phpdotenv) через класс
`Spiral\DotEnv\Bootloader\DotenvBootloader`. Этот загрузчик читает переменные окружения из файла `.env` и делает их
доступными приложению.

Обычно в новый проект добавляют файл `.env.sample`, который служит примером настройки переменных окружения.

<details>
  <summary>Показать .env.sample</summary>

```dotenv .env
# Environment (prod or local)
APP_ENV=local

# Debug mode set to TRUE disables view caching and enables higher verbosity
DEBUG=true
VERBOSITY_LEVEL=verbose # basic, verbose, debug

# Set to an application specific value, used to encrypt/decrypt cookies etc
ENCRYPTER_KEY=...

# Monolog
MONOLOG_DEFAULT_CHANNEL=default
MONOLOG_DEFAULT_LEVEL=DEBUG # DEBUG, INFO, NOTICE, WARNING, ERROR, CRITICAL, ALERT, EMERGENCY

# Queue
QUEUE_CONNECTION=roadrunner

# Cache
CACHE_STORAGE=roadrunner

# Telemetry
TELEMETRY_DRIVER=null

# Serializer
DEFAULT_SERIALIZER_FORMAT=json # csv, xml, yaml

# Session
SESSION_LIFETIME=86400
SESSION_COOKIE=sid

# Authorization
AUTH_TOKEN_TRANSPORT=cookie
AUTH_TOKEN_STORAGE=session

# Mailer
MAILER_DSN=
MAILER_FROM="My site <no-reply@site.com>"
```

</details>

> **Предупреждение**
> Переменная, определённая в суперглобальном массиве `$_SERVER` или `$_ENV`, имеет приоритет над одноимённой переменной
> из файла `.env`.

Значения из файла `.env` копируются в окружение приложения и доступны через
`Spiral\Boot\EnvironmentInterface` или функцию `env`.

### Доступные переменные

| Переменная                  | Описание                                                                                         |
|-----------------------------|--------------------------------------------------------------------------------------------------|
| `APP_ENV`                   | Текущее окружение приложения: `prod`, `stage`, `testing`, `local`. По умолчанию: `local`         |
| `DEBUG`                     | Режим отладки. По умолчанию: `false`                                                             |
| `TOKENIZER_CACHE_TARGETS`   | Кеширование целей токенизатора. Логическое значение. По умолчанию: `false`                        |
| `ENCRYPTER_KEY`             | Ключ шифрования                                                                                  |
| `VERBOSITY_LEVEL`           | Уровень подробности: `basic`, `verbose`, `debug`. По умолчанию: `verbose`                         |
| `MONOLOG_DEFAULT_CHANNEL`   | Канал по умолчанию из `app/config/monolog.php`. По умолчанию: `default`                          |
| `QUEUE_CONNECTION`          | Имя подключения очереди из `app/config/queue.php`. По умолчанию: `sync`                          |
| `BROADCAST_CONNECTION`      | Подключение для трансляций: `log`, `null`, `centrifugo`. По умолчанию: `null`                    |
| `CACHE_STORAGE`             | Хранилище кеша из `app/config/cache.php`                                                         |
| `TELEMETRY_DRIVER`          | Драйвер телеметрии: `null`, `log`, `otel`. По умолчанию: `null`                                  |
| `LOCALE`                    | Локаль по умолчанию. По умолчанию: `en`                                                          |
| `DEFAULT_SERIALIZER_FORMAT` | Формат сериализатора по умолчанию: `json`, `serializer`. По умолчанию: `json`                    |
| `AUTH_TOKEN_TRANSPORT`      | Способ передачи токена авторизации: `cookie`, `header`. По умолчанию: `cookie`                   |
| `AUTH_TOKEN_STORAGE`        | Хранилище токена авторизации: `session`, `cycle`. По умолчанию: `session`                         |
| `SESSION_LIFETIME`          | Время жизни сессии. По умолчанию: `86400`                                                        |
| `SESSION_COOKIE`            | Имя cookie сессии. По умолчанию: `sid`                                                           |
| `VIEW_CACHE`                | Кеширование представлений. По умолчанию: `DEBUG !== true`                                        |
| `MAILER_DSN`                | DSN почтового компонента                                                                         |
| `MAILER_FROM`               | Отправитель почты. По умолчанию: `Spiral <sendit@local.host>`                                    |
| `MAILER_QUEUE_CONNECTION`   | Подключение очереди почтового компонента. По умолчанию берётся из `QUEUE_CONNECTION`             |

### Получение переменных окружения

Переменные окружения можно получать через `Spiral\Boot\EnvironmentInterface`:

```php
use Spiral\Boot\EnvironmentInterface;

final class GithubClient
{
    public function __construct(
        private readonly EnvironmentInterface $env
    ) {}
    
    public function getAccessToken(): ?string
    {
        return $this->env->get('GITHUB_ACCESS_TOKEN');
    }
}
```

или с помощью короткой функции `env()`:

```php 
return [
    'access_token' => env('GITHUB_ACCESS_TOKEN'),
    // ...
];
```

### Предварительная обработка

Значения из `.env` предварительно обрабатываются следующим образом:

| Значение | Значение PHP |
|----------|--------------|
| true     | true         |
| (true)   | true         |
| false    | false        |
| (false)  | false        |
| null     | null         |
| (null)   | null         |
| empty    | ''           |

> **Примечание**
> Кавычки вокруг строк удаляются автоматически.

## Конфигурация

Переменные окружения удобны для настройки отдельных параметров, однако сложные или детальные изменения не всегда
практично задавать только через них. В таких случаях измените конфигурационный файл соответствующего компонента.

Например, так можно изменить HTTP-заголовки по умолчанию:

```php app/config/http.php
return [
    'basePath'   => '/',
    'headers' => [
        'Server' => 'Spiral',
        'Content-Type' => 'text/html; charset=UTF-8'
    ],
    'middleware' => [],
];
```

### Получение значений конфигурации

### Объекты конфигурации

Все объекты конфигурации Spiral можно внедрять как зависимости, поэтому их удобно использовать в приложении.

```php
use Spiral\Http\Config\HttpConfig;

final class HttpClient 
{
    private readonly string $basePath;

    public function __construct(
        HttpConfig $config // <-- Container will automatically load values from app/config/http.php
    ) {
        $this->basePath = $this->config->getBasePath();
    }
}
```

Каждый внедряемый класс конфигурации Spiral содержит
[константу CONFIG](https://github.com/spiral/http/blob/master/src/Config/HttpConfig.php#L19), определяющую имя
соответствующего конфигурационного файла. Когда контейнер разрешает объект конфигурации, он автоматически загружает
значения из файла и помещает их в свойство `$config` этого объекта.

> **Примечание**
> При загрузке связанного конфигурационного файла объект автоматически объединяет его значения с настройками по
> умолчанию, объявленными в классе конфигурации. Поэтому в файл достаточно добавить только изменяемые параметры. Для
> отсутствующих значений будут использованы настройки по умолчанию.

Например, конфигурация **HTTP** выглядит так:

```php spiral/framework/src/Http/src/Config/HttpConfig.php
final class HttpConfig extends InjectableConfig
{
    const CONFIG = 'http';
    
    // ...
}
```

> **Примечание**
> Описание конфигурации каждого компонента приведено в соответствующем разделе документации.

### Определение окружения приложения

Текущее окружение приложения определяется переменной `APP_ENV`. Получить её типизированное значение можно через
внедряемое перечисление `Spiral\Boot\Environment\AppEnvironment`.

> **Смотрите также**
> Подробнее о внедряемых перечислениях читайте в разделе
> [Расширенные возможности — Инжекторы контейнера](../container/injectors.md#enum-injectors).

При запросе `AppEnvironment` из контейнера будет автоматически внедрён элемент перечисления с правильным значением.

```php
use Spiral\Boot\Environment\AppEnvironment;
use Psr\Http\Server\MiddlewareInterface;

final class ErrorHandlerMiddleware implements MiddlewareInterface
{
    public function __construct(
        private readonly AppEnvironment $env
    ) {
    }

    public function process(
        ServerRequestInterface $request, 
        RequestHandlerInterface $handler
    ): ResponseInterface {
        try {
            return $handler->handle($request);
        } catch (Throwable $e) {
            if ($this->env->isProduction()) {
                // ...
            }
            
            // ...
        }
    }
}
```

### Определение режима отладки

Текущий режим отладки определяется переменной `DEBUG`. Получить его типизированное значение можно через внедряемое
перечисление `Spiral\Boot\Environment\DebugMode`.

```php
use Spiral\Boot\Environment\DebugMode;
use Psr\Http\Server\MiddlewareInterface;

final class ErrorHandlerMiddleware implements MiddlewareInterface
{
    public function __construct(
        private readonly DebugMode $debug
    ) {
    }

    public function process(
        ServerRequestInterface $request, 
        RequestHandlerInterface $handler
    ): ResponseInterface {
        try {
            return $handler->handle($request);
        } catch (Throwable $e) {
            if ($this->debug->isEnabled()) {
                // ...
            }
            
            // ...
        }
    }
}
```

### Определение уровня подробности

Текущий уровень подробности определяется переменной `VERBOSITY_LEVEL`. Получить его типизированное значение можно через
внедряемое перечисление `Spiral\Exceptions\Verbosity`.

```php
use Spiral\Exceptions\Verbosity;
use Psr\Http\Server\MiddlewareInterface;

final class ErrorHandlerMiddleware implements MiddlewareInterface
{
    public function __construct(
        private readonly Verbosity $verbosity
    ) {
    }

    public function process(
        ServerRequestInterface $request, 
        RequestHandlerInterface $handler
    ): ResponseInterface {
        try {
            return $handler->handle($request);
        } catch (Throwable $e) {
            if ($this->verbosity === Verbosity::BASIC)
                // ...
            }
            
            if ($this->verbosity === Verbosity::VERBOSE)
                // ...
            }
            
            // ...
        }
    }
}
```

<hr>

## Что дальше?

Для более глубокого знакомства с основами прочитайте следующие разделы:

* [Объекты конфигурации](../framework/config.md);
* [Расширенные возможности — Инжекторы контейнера](../container/injectors.md);
* [Ядро и окружение](../framework/kernel.md).
