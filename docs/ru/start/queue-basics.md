# Начало работы — Первое фоновое задание

В этом руководстве показано, как создать и запустить фоновое задание с помощью Spiral и сервера приложений
[RoadRunner](https://roadrunner.dev/). Это позволяет выполнять задачи асинхронно, не останавливая основную работу
приложения.

Рассмотрим необходимые шаги.

## Создание задания

Чтобы быстро создать первый обработчик задания, используйте генератор:

```terminal
php app.php create:jobHandler PingSite
```

> **Примечание**
> Подробнее о генерации кода читайте в разделе
> [Основы — Генерация кода](../basics/scaffolding.md#job-handler).

После выполнения команды успешное создание будет подтверждено следующим выводом:

```output
Declaration of '[32mPingSiteJob[39m' has been successfully written into '[33mapp/src/Endpoint/Job/PingSiteJob.php[39m'.
```

Теперь добавим логику в созданный обработчик.

Пример задания, отправляющего запрос `GET` указанному сайту:

```php app/src/Endpoint/Job/PingSiteJob.php
namespace App\Endpoint\Job;

use Spiral\Queue\JobHandler;

final class PingSiteJob extends JobHandler
{
    public function invoke(HttpClientInterface $client, string $site): void
    {
        $response = $client->request('GET', $site);
        
        // do something with response ...
    }
}
```

## Конфигурация

Убедитесь, что плагин jobs включён в конфигурационном файле RoadRunner `.rr.yaml`:

```yaml .rr.yaml
rpc:
  listen: 'tcp://127.0.0.1:6001'

jobs:
  consume: { }

# ...
```

Затем настройте приложение для отправки заданий в RoadRunner. Откройте конфигурационный файл `app/config/queue.php` и
внесите следующие изменения:

```php app/config/queue.php
use Spiral\RoadRunner\Jobs\Queue\MemoryCreateInfo;

return [
    'default' => env('QUEUE_CONNECTION', 'roadrunner'),

    pipelines' => [
        'memory' => [
            'connector' => new MemoryCreateInfo('local'),
            'consume' => true,
        ]
    ],
            
    'connections' => [
        'roadrunner' => [
            'driver' => 'roadrunner',
            'default' => 'memory',
        ],
    ],
];
```

Эти настройки создают для RoadRunner новый конвейер `in-memory`. При отправке задания в этот конвейер оно добавляется
во внутрипроцессную очередь, после чего RoadRunner передаёт его потребителю для обработки.

## Запуск задания

После настройки задания и RoadRunner создадим консольную команду, которая отправит задание в очередь.

Создайте команду для отправки `PingSiteJob`:

```terminal
php app.php create:command PingSite
```

```php app/src/Endpoint/Console/PingSiteCommand.php
namespace App\Endpoint\Console;

use App\Endpoint\Job\PingSiteJob;
use Spiral\Console\Attribute\Argument;
use Spiral\Console\Attribute\AsCommand;
use Spiral\Console\Command;
use Spiral\Queue\QueueInterface;

#[AsCommand(name: 'ping:site', description: 'Ping site')]
final class PingSiteCommand extends Command
{
    #[Argument(description: 'Site to ping')]
    public string $site;

    public function __invoke(QueueInterface $queue): int
    {
        $id = $queue->push(PingSiteJob::class, [
            'site' => $this->site,
        ]);

        $this->writeln(\sprintf('Job %s pushed', $id));

        return self::SUCCESS;
    }
}
```

В этом примере `QueueInterface` внедряется в метод. Контейнер зависимостей автоматически разрешает интерфейс и
передаёт экземпляр подключения к очереди, указанного в конфигурации по умолчанию.

#### Запуск сервера RoadRunner

Сначала запустите RoadRunner:

```terminal
./rr serve
```

#### Запуск консольной команды

Теперь выполните консольную команду и отправьте задание в очередь:

```terminal
php app.php ping:site "https://google.com"
```

Должен появиться следующий вывод:

```output
Job [32m3332e595-9774-434c-908c-3c419f80c967[39m pushed
```

После отправки RoadRunner заберёт задание из очереди и передаст его потребителю для обработки.

Готово! Вы создали первое фоновое задание с помощью Spiral и RoadRunner. Теперь можно добавлять новые задания и
асинхронно выполнять задачи, не блокируя работу приложения.

<hr>

## Что дальше?

Для более глубокого знакомства с основами прочитайте следующие разделы:

* [Очереди и задания](../queue/configuration.md);
* [Перехватчики очередей](../queue/interceptors.md);
* [Создание консольной команды](../console/commands.md);
* [Генерация кода](../basics/scaffolding.md).
