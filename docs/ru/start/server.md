# Начало работы — Долгоживущие приложения

Spiral спроектирован для разработки долгоживущих приложений с эффективным управлением памятью и защитой от её
утечек. Это достигается за счёт современных [механизмов управления памятью](../container/scopes.md). Дополнительное
использование RoadRunner повышает общую производительность и масштабируемость приложения.

RoadRunner — высокопроизводительный сервер PHP-приложений и менеджер процессов, позволяющий PHP-приложениям работать
в долгоживущем режиме. Он эффективно управляет ресурсами, включая процессорное время и память, благодаря чему
приложение может стабильно и эффективно работать длительное время.

RoadRunner обрабатывает разные типы запросов, включая [HTTP](../http/lifecycle.md),
[gRPC](../grpc/configuration.md), TCP, [задания очередей](../queue/roadrunner.md) и Temporal. При запуске он создаёт
воркеры один раз, а затем направляет запросы в [диспетчер](../framework/dispatcher.md) в зависимости от их типа. Каждый
воркер изолирован и работает независимо в соответствии с подходом «ничем не делиться»: ресурсы между воркерами не
разделяются.

Использование RoadRunner позволяет значительно повысить скорость и эффективность приложения, поскольку устраняет
необходимость повторять процесс начальной загрузки при каждом запросе. Это снижает расход процессорных ресурсов и
памяти, а также сокращает время ответа.

> **Смотрите также**
> Подробнее о взаимодействии фреймворка и сервера приложений читайте в разделе
> [Фреймворк — Жизненный цикл приложения](../framework/lifecycle.md).

## Установка

Использовать RoadRunner достаточно просто. После загрузки бинарного файла его можно применять для запуска
PHP-приложения.

Скачать RoadRunner можно несколькими способами.

:::: tabs

::: tab Composer

Рекомендуемый способ — пакет Composer `spiral/roadrunner-cli`, который позволяет автоматически загрузить сервер.

Установите пакет в проект:

```terminal
composer require spiral/roadrunner-cli
```

Затем выполните команду для загрузки последней версии RoadRunner:

```terminal
./vendor/bin/rr get
```

> **Предупреждение**
> Для автоматической загрузки RoadRunner необходимы расширения PHP `php-curl` и `php-zip`.
:::

::: tab cURL

Загрузите последнюю стабильную версию RoadRunner с помощью cURL:

```bash
curl --proto '=https' --tlsv1.2 -sSf  https://raw.githubusercontent.com/roadrunner-server/roadrunner/master/download-latest.sh | sh
```
:::

::: tab Docker

RoadRunner предоставляет предварительно скомпилированные бинарные файлы сервера в Docker-образе.

```docker Dockerfile
FROM spiralscout/roadrunner as roadrunner
# OR
# FROM ghcr.io/roadrunner-server/roadrunner as roadrunner

FROM php:8.1-cli

# Copy the RoadRunner binary from the roadrunner image to the local bin directory
COPY --from=roadrunner /usr/bin/rr /usr/local/bin/rr

# Run the RoadRunner server command
CMD ["rr", "serve"]
```

**Образы доступны в следующих реестрах:**

- **GitHub** — [ghcr.io/roadrunner-server/roadrunner](https://github.com/roadrunner-server/roadrunner/pkgs/container/roadrunner);
- **Docker Hub** — [spiralscout/roadrunner](https://hub.docker.com/r/spiralscout/roadrunner).

:::

::: tab Linux

Вариант установки для дистрибутивов на основе Debian — Ubuntu, Mint, MX и других:

```bash
wget https://github.com/roadrunner-server/roadrunner/releases/download/v2.X.X/roadrunner-2.X.X-linux-amd64.deb
sudo dpkg -i roadrunner-2.X.X-linux-amd64.deb
```
:::

::: tab GitHub

Если другие способы установки не подходят, бинарный файл RoadRunner всегда можно загрузить непосредственно с GitHub.

Откройте [последний выпуск RoadRunner](https://github.com/roadrunner-server/roadrunner/releases/latest), прокрутите
страницу до раздела «Assets» и выберите бинарный файл для своей операционной системы.

:::

::::

## Конфигурация

Количество воркеров, ограничения памяти и другие плагины настраиваются в файле `.rr.yaml`:

```yaml .rr.yaml
rpc:
  listen: tcp://127.0.0.1:6001

server:
  command: "php app.php"
  relay: pipes

# HTTP plugin settings
http:
  address: 0.0.0.0:8080
  middleware: [ "gzip", "static" ]
  static:
    dir: "public"
    forbid: [ ".php", ".htaccess" ]
  pool:
    num_workers: 2
    supervisor:
      max_worker_memory: 100
```

Чтобы установить количество HTTP-воркеров:

```yaml .rr.yaml
http:
  pool:
    num_workers: 4
```

> **Смотрите также**
> Подробнее о конфигурации сервера приложений читайте в официальной
> [документации](https://roadrunner.dev/docs).

## Запуск сервера

:::: tabs

::: tab Linux

Для запуска сервера приложений в **Linux** используйте:

```terminal
./rr serve
```

> **Предупреждение**
> Убедитесь, что бинарному файлу `rr` разрешено выполнение.

:::

::: tab Windows

Для запуска сервера приложений в **Windows** используйте:

```terminal
./rr.exe serve
```

:::

::::

> **Смотрите также**
> Подробнее о командах сервера читайте в [документации RoadRunner](https://roadrunner.dev/docs/app-server-cli).

## Интеграция RoadRunner

Пакет [spiral/roadrunner-bridge](https://github.com/spiral/roadrunner-bridge) обеспечивает полную интеграцию Spiral и
RoadRunner. С его помощью можно использовать различные плагины RoadRunner, включая `http`, `grpc`, `jobs`, `tcp`, `kv`,
`locks`, `centrifugo`, `app-logger` и `metrics`.

> **Примечание**
> Компонент по умолчанию входит в [набор приложения](https://github.com/spiral/app).

### Установка

Для установки пакета выполните:

```terminal
composer require spiral/roadrunner-bridge
```

После установки добавьте загрузчики пакета в `Kernel`, выбрав загрузчики, соответствующие необходимым плагинам:

:::: tabs

::: tab С помощью метода

```php app/src/Application/Kernel.php
use Spiral\RoadRunnerBridge\Bootloader as RoadRunnerBridge;

public function defineBootloaders(): array
{
    return [
        RoadRunnerBridge\HttpBootloader::class, // Optional, if it needs to work with http plugin
        RoadRunnerBridge\QueueBootloader::class, // Optional, if it needs to work with jobs plugin
        RoadRunnerBridge\CacheBootloader::class, // Optional, if it needs to work with KV plugin
        RoadRunnerBridge\GRPCBootloader::class, // Optional, if it needs to work with GRPC plugin
        RoadRunnerBridge\CommandBootloader::class,
        RoadRunnerBridge\TcpBootloader::class, // Optional, if it needs to work with TCP plugin
        RoadRunnerBridge\MetricsBootloader::class, // Optional, if it needs to work with metrics plugin
        RoadRunnerBridge\LoggerBootloader::class, // Optional, if it needs to work with app-logger plugin
        // ...
    ];
}
```

Подробнее о загрузчиках читайте в разделе [Фреймворк — Загрузчики](../framework/bootloaders.md).
:::

::: tab С помощью константы

```php app/src/Application/Kernel.php
use Spiral\RoadRunnerBridge\Bootloader as RoadRunnerBridge;

protected const LOAD = [
    RoadRunnerBridge\HttpBootloader::class, // Optional, if it needs to work with http plugin
    RoadRunnerBridge\QueueBootloader::class, // Optional, if it needs to work with jobs plugin
    RoadRunnerBridge\CacheBootloader::class, // Optional, if it needs to work with KV plugin
    RoadRunnerBridge\GRPCBootloader::class, // Optional, if it needs to work with GRPC plugin
    RoadRunnerBridge\CommandBootloader::class,
    RoadRunnerBridge\TcpBootloader::class, // Optional, if it needs to work with TCP plugin
    RoadRunnerBridge\MetricsBootloader::class, // Optional, if it needs to work with metrics plugin
    RoadRunnerBridge\LoggerBootloader::class, // Optional, if it needs to work with app-logger plugin
    // ...
];
```

Подробнее о загрузчиках читайте в разделе [Фреймворк — Загрузчики](../framework/bootloaders.md).
:::

::::

## Подводные камни

При использовании долгоживущих приложений необходимо учитывать несколько ограничений.

### Состояние приложения

Когда приложение запущено через RoadRunner, изменения файлов не влияют на него, поскольку после запуска код находится
в памяти. Чтобы применить изменения, сервер необходимо перезапустить. Во время разработки это может быть неудобно.

Чтобы принудительно перезагружать воркер после каждого запроса — полный режим отладки — и обрабатывать запросы одним
воркером, добавьте параметр `debug`:

```yaml .rr.yaml
http:
  pool:
    debug: true
```

> **Предупреждение**
> Эта возможность влияет на производительность сервера приложений, поэтому её следует использовать только в режиме
> разработки.

### Утечки памяти

Поскольку приложение долго остаётся в памяти, даже небольшая утечка может привести к перезапуску процесса. RoadRunner
отслеживает потребление памяти и выполняет мягкий сброс, однако утечек памяти в коде приложения всё равно следует
избегать.

> **Примечание**
> Фреймворк включает набор инструментов, упрощающих разработку и помогающих избегать утечек памяти и состояния:
> IoC-области, Cycle ORM, неизменяемые конфигурации, ядра доменов, маршруты и middleware.

<hr>

## Что дальше?

Для более глубокого знакомства с основами прочитайте следующие разделы:

* [Жизненный цикл приложения](../framework/lifecycle.md);
* [Диспетчеры](../framework/dispatcher.md);
* [Финализаторы](../framework/finalizers.md);
* [Статическая память](../advanced/memory.md);
* [Пользовательский диспетчер](../cookbook/custom-dispatcher.md).
