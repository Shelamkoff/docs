# Основы — Отладка

При разработке долгоживущего приложения на Spiral и RoadRunner необходимо учитывать особенности отладки такого кода.

## Распространённые приёмы отладки

### Не используйте функции `die` и `exit`

В обычном PHP-приложении функции `die` и `exit` часто применяются для остановки выполнения скрипта, например вместе с
`var_dump`. В окружении Spiral они могут нарушить работу приложения: будет завершён весь воркер RoadRunner, а не только
текущий запрос.

Например, функция `dd` из пакета `symfony/var-dumper` выводит содержимое переменной, а затем вызывает `die`, из-за чего
воркер RoadRunner завершает работу.

### Учитывайте различие `PHP_SAPI`

RoadRunner не использует традиционный PHP SAPI. Некоторые средства вывода диагностических данных предполагают, что
работают в окружении CLI. Такое несоответствие может приводить к неожиданному поведению во время отладки.

## Spiral Dumper

Для решения этих проблем создан пакет `spiral/dumper`. Он служит оболочкой над библиотекой
[symfony/var-dumper](https://symfony.com/doc/current/components/var_dumper.html) и отправляет дампы переменных прямо в
браузер из HTTP-воркеров либо в `STDERR` в других окружениях. Пакет рассчитан на долгоживущую модель RoadRunner.

`spiral/dumper` позволяет удобно исследовать значения переменных во время разработки и подходит для отладки как
веб-приложений, так и консольных приложений.

### Установка

По умолчанию `spiral/dumper` уже включён в каркас `spiral/app`. При использовании другого каркаса установите пакет:

```terminal
composer require --dev spiral/dumper
```

После установки добавьте загрузчик пакета в приложение.

:::: tabs

::: tab С помощью метода

```php app/src/Application/Kernel.php
public function defineSystemBootloaders(): array
{
    return [
        // ...
        \Spiral\Debug\Bootloader\DumperBootloader::class,
    ];
}
```

Подробнее о загрузчиках читайте в разделе [Фреймворк — Загрузчики](../framework/bootloaders.md).
:::

::: tab С помощью константы

```php app/src/Application/Kernel.php
protected const SYSTEM = [
    // ...
    \Spiral\Debug\Bootloader\DumperBootloader::class,
];
```

Подробнее о загрузчиках читайте в разделе [Фреймворк — Загрузчики](../framework/bootloaders.md).
:::

::::

### Использование

Для вывода переменной используйте вспомогательную функцию `dump()`:

```php
dump($variable);
```

Пакет также позволяет использовать `dd` как в обычном PHP-приложении, но без риска завершить весь воркер RoadRunner.

```php
dd($variable);
```

<hr />

## Symfony VarDumper

Для более традиционной отладки можно использовать пакет `symfony/var-dumper`.

Он предоставляет отдельный сервер, собирающий все дампы. После запуска сервер принимает данные функции `dump()`, а
вывод отображается в отдельном окне консоли, не смешиваясь с основным выводом приложения.

Пример консольного вывода:

```terminal
./vendor/bin/var-dump-server

Symfony Var Dumper Server
=========================
 [OK] Server listening on tcp://127.0.0.1:9912
 // Quit the server with CONTROL-C.

$ app.php
---------
 -------- ---------------------------------------------------------
  date     Fri, 18 Aug 2023 11:54:44 +0000
  source   SimpleController.php on line 36
  file     app/src/Interfaces/Http/Controller/SimpleController.php
 -------- ---------------------------------------------------------
null

 -------- ---------------------------------------------------------
  date     Fri, 18 Aug 2023 11:54:44 +0000
  source   SimpleController.php on line 37
  file     app/src/Interfaces/Http/Controller/SimpleController.php
 -------- ---------------------------------------------------------
App\Service\Site\Site^ {#1260
  -theme: "default"
  -docs: App\Service\Site\Docs^ {#1269
    -defaultVersion: "3.5"
    -defaultLanguage: "en"
  }
  -host: "127.0.0.1"
}
```

### Установка

Установите пакет:

```terminal
composer require --dev symfony/var-dumper
```

### Использование

Запустите сервер:

```terminal
./vendor/bin/var-dump-server
```

Также задайте переменную окружения `VAR_DUMPER_FORMAT` в файле `.env`:

```dotenv .env
VAR_DUMPER_FORMAT=server
```

### Известные ограничения

Если объект содержит много свойств или значительные объёмы данных, консольный вывод становится слишком большим. В нём
сложно найти конкретные сведения.

Проблема особенно заметна при работе со сложными объектами — например, сущностями ORM с большим количеством связей или
крупными вложенными массивами. Линейный консольный вывод ограниченного размера плохо подходит для представления таких
структур в удобной для чтения и навигации форме.

<hr />

## Расширенная отладка с Buggregator

[Buggregator](https://github.com/buggregator/spiral-app) — мощное веб-приложение и сервер в Docker, улучшающий процесс
отладки PHP. Он прослушивает TCP- и HTTP-порты и принимает дампы переменных, исключения, журналы приложения, SMTP-письма
и другие данные.

![var-dumper](https://user-images.githubusercontent.com/773481/208727353-b8201775-c360-410b-b5c8-d83843d388ff.png)

### Основные возможности

1. **Перехват и отображение переменных PHP.** Buggregator интегрируется с такими инструментами, как
   [Symfony VarDumper](https://github.com/buggregator/spiral-app#2-symfony-vardumper-server), и показывает дампы в
   упорядоченном и читаемом виде.

2. **Обработка исключений.** Он принимает и отображает исключения, включая отчёты платформ отслеживания ошибок, таких
   как [Sentry](https://github.com/buggregator/spiral-app#4-compatible-with-sentry-reports), предоставляя единое место
   для анализа проблем.

3. **Перехват SMTP-писем.** Buggregator может работать как
   [тестовый SMTP-сервер](https://github.com/buggregator/spiral-app#3-fake-smtp-server-for-catching-mail), принимая и
   отображая письма приложения без их фактической отправки.

4. **Удобный интерфейс.** Веб-интерфейс организует диагностические данные и упрощает навигацию по ним.

5. **Простой запуск в Docker.** Buggregator распространяется как Docker-контейнер и легко запускается в любой среде
   разработки.

При работе со сложными объектами поиск данных в консоли или журналах может быть трудоёмким. Buggregator представляет
информацию в структурированном веб-интерфейсе с возможностью сворачивания и поиска, поэтому не приходится просматривать
сотни строк текста.

### Установка

Загрузите Docker-образ и запустите контейнер:

```bash Latest stable release
docker run --pull always ghcr.io/buggregator/server:latest
    -p 8000:8000 
    -p 1025:1025 
    -p 9912:9912 
    -p 9913:9913 
```

Для использования с Docker Compose добавьте сервис в `docker-compose.yaml`:

```yaml docker-compose.yaml
services:
  # ...
  buggregator:
    image: ghcr.io/buggregator/server:latest
    ports:
      - 8000:8000
      - 1025:1025
      - 9912:9912
      - 9913:9913
```

После запуска откройте http://127.0.0.1:8000 в браузере. В интерфейсе Buggregator можно наблюдать диагностические данные
приложения в реальном времени.

> **Примечание**
> Настройка отправки данных из приложения описана в
> [репозитории GitHub](https://github.com/buggregator/server#features).

### Интеграция с XHProf

Buggregator не только принимает диагностические данные, но и помогает профилировать приложение. Он отображает профили
[XHProf](https://github.com/buggregator/spiral-app#1-xhprof-profiler), позволяя анализировать производительность, находить
узкие места и утечки памяти в PHP-приложении.

> **Примечание**
> XHProf показывает, как выполняется PHP-код: сколько раз вызываются разные участки, сколько времени они занимают и
> сколько памяти используют. Пакет [spiral/profiler](https://github.com/spiral/profiler) упрощает подключение XHProf к
> приложению и помогает быстро находить и устранять проблемы производительности.

![xhprof](https://user-images.githubusercontent.com/773481/208724383-3790a3e1-9ebe-4616-8d4d-d1869f8f2b7c.png)

**Профилирование полезно для следующих задач:**

1. **Поиск узких мест.** Buggregator показывает данные XHProf в сортируемых таблицах и графиках, помогая определить части
   приложения, потребляющие больше всего времени и ресурсов.

2. **Поиск утечек памяти.** Подробное представление профиля помогает обнаруживать скачки использования памяти и
   устранять их причины.

#### Установка

Сначала установите расширение XHProf, например через PECL:

```terminal
pear channel-update pear.php.net
pecl install xhprof
```

Затем установите пакет профилировщика:

```terminal
composer require --dev spiral/profiler:^3.0
```

После установки добавьте загрузчик пакета в приложение.

:::: tabs

::: tab С помощью метода

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Profiler\ProfilerBootloader::class,
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
    \Spiral\Profiler\ProfilerBootloader::class,
    // ...
];
```

Подробнее о загрузчиках читайте в разделе [Фреймворк — Загрузчики](../framework/bootloaders.md).
:::

::::

#### Конфигурация

Следующие переменные окружения настраивают отправку данных профилировщика серверу
[Buggregator](https://github.com/buggregator/spiral-app):

```dotenv .env
PROFILER_ENDPOINT=http://127.0.0.1:8000/api/profiler/store
PROFILER_APP_NAME=My super app
```

#### Использование

Профилировщик можно использовать двумя способами:

- как перехватчик;
- как middleware.

#### Профилировщик как перехватчик

Перехватчик подходит для профилирования определённой части приложения, поддерживающей перехватчики:

- [контроллеров](../http/interceptors.md);
- [gRPC](../grpc/interceptors.md);
- [заданий очередей](../queue/interceptors.md);
- TCP;
- [событий](../advanced/events.md#interceptors).

> **Смотрите также**
> Подробнее о перехватчиках читайте в разделе
> [Фреймворк — Перехватчики](../framework/interceptors.md).

Для использования профилировщика как перехватчика зарегистрируйте класс `Spiral\Profiler\ProfilerInterceptor`.

Пример для HTTP-уровня:

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Bootloader\DomainBootloader;
use Spiral\Core\CoreInterface;

class AppBootloader extends DomainBootloader
{
    protected const SINGLETONS = [
        CoreInterface::class => [self::class, 'domainCore']
    ];

    protected const INTERCEPTORS = [
        \Spiral\Profiler\ProfilerInterceptor::class
    ];
}
```

#### Профилировщик как middleware

Middleware подходит для профилирования всех запросов к приложению. Добавьте его в маршрутизатор.

> **Смотрите также**
> Подробнее о middleware читайте в разделе [HTTP — Маршрутизация](../http/routing.md).

##### Глобальное middleware

```php app/src/Application/Bootloader/RoutesBootloader.php
namespace App\Application\Bootloader;

use Spiral\Bootloader\Http\RoutesBootloader as BaseRoutesBootloader;
use Spiral\Profiler\ProfilerMiddleware;

final class RoutesBootloader extends BaseRoutesBootloader
{
    protected function globalMiddleware(): array
    {
        return [
            ProfilerMiddleware::class,  // <================
            // ...
        ];
    }
    
    // ...
}
```

##### Middleware группы маршрутов

```php app/src/Application/Bootloader/RoutesBootloader.php
namespace App\Application\Bootloader;

use Spiral\Bootloader\Http\RoutesBootloader as BaseRoutesBootloader;
use Spiral\Profiler\ProfilerMiddleware;

final class RoutesBootloader extends BaseRoutesBootloader
{
    protected function middlewareGroups(): array
    {
        return [
            'web' => [
                // ...
            ],
            'profiler' => [                  // <================
                ProfilerMiddleware::class,
                'middleware:web',
            ],
        ];
    }
    
    // ...
}
```

##### Middleware отдельного маршрута

```php app/src/Application/Bootloader/RoutesBootloader.php
use Spiral\Router\Annotation\Route;

final class UserController
{
    #[Route(route: '/users', name: 'user.store', methods: ['POST'], middleware: \Spiral\Profiler\ProfilerMiddleware::class)]
    public function store(...): void 
    {
        // ...
    }
}
```

<hr />

## XDebug

При использовании расширения XDebug приложение Spiral можно отлаживать так же, как обычное PHP-приложение.

### Настройка IDE

Сначала настройте IDE для работы с XDebug.

> **Смотрите также**
> Подробнее о настройке IDE читайте в официальной
> [документации](https://roadrunner.dev/docs/php-debugging).

### Запуск по требованию

Удобнее запускать RoadRunner с XDebug только при необходимости. Добавьте в `.rr.yaml` следующие переменные окружения:

```yaml .rr.yaml
env:
  PHP_IDE_CONFIG: serverName=application.loc
  XDEBUG_CONFIG: remote_host=localhost max_nesting_level=250 remote_enable=1 remote_connect_back=0 var_display_max_depth=5 idekey='PHPSTORM'
```

> **Примечание**
> Измените значения в соответствии со своим окружением.

Чтобы включить XDebug, запустите сервер приложений с флагом переопределения `-o`:

```terminal
./rr serve -o "server.command=php -d zend_extension=xdebug app.php"
```

### В Docker

Для изменения конфигурации воркеров в Docker используйте следующую или аналогичную конфигурацию контейнера:

```yaml docker-compose.yaml
version: "2"
services:
  ...
  app:
    ...
    command:
      - /usr/local/bin/rr
      - serve
      - -o
      - server.command=php -d zend_extension=xdebug.so app.php
    environment:
      PHP_IDE_CONFIG: serverName=application.loc
      XDEBUG_CONFIG: remote_host=host.docker.internal max_nesting_level=250 remote_enable=1 remote_connect_back=0 var_display_max_depth=5 idekey='PHPSTORM'
```
