# Фреймворк — Диспетчеры

Одна из ключевых возможностей Spiral — поддержка нескольких диспетчеров ядра. Они направляют входящие запросы
подходящему обработчику в зависимости от текущего окружения.

Предположим, что в приложении есть два диспетчера: `console` и `http`.

Пример диспетчера `http`:

```php
use Spiral\Boot\EnvironmentInterface;
use Spiral\Boot\DispatcherInterface;

final class HttpDispatcher implements DispatcherInterface
{
    public function __construct(
        private readonly EnvironmentInterface $env
    ) {
    }
    
    public function canServe(): bool
    {
        return $this->env->get('RR_MODE') === 'http';
    }
    
    public function serve(): void
    {
        // Handle HTTP requests
    }
}
```

Пример диспетчера `console`:

```php
use Spiral\Boot\EnvironmentInterface;
use Spiral\Boot\DispatcherInterface;

final class ConsoleDispatcher implements DispatcherInterface
{
    public function __construct(
        private readonly EnvironmentInterface $env
    ) {
    }
    
    public function canServe(): bool
    { 
        return (PHP_SAPI === 'cli' && $this->env->get('RR_MODE') === null);
    }
    
    public function serve(InputInterface $input = null, OutputInterface $output = null): int
    {
        // Handle console command
    }
}
```

Зарегистрируем диспетчеры в приложении:

```php
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Boot\KernelInterface;

class AppBootloader extends Bootloader
{
    public function boot(
      KernelInterface $kernel, 
      HttpDispatcher $http,
      ConsoleDispatcher $console,
    ): void  {
        $kernel->addDispatcher($http)
        $kernel->addDispatcher($console);
    }
}
```

Точка входа приложения `app.php` будет выглядеть следующим образом:

```php app.php
use App\Application\Kernel;

\mb_internal_encoding('UTF-8');
\error_reporting(E_ALL | E_STRICT ^ E_DEPRECATED);
\ini_set('display_errors', 'stderr');

require __DIR__ . '/vendor/autoload.php';
$app = Kernel::create(
    directories: ['root' => __DIR__],
)->run();

$code = (int)$app->serve();  // <========== Will run the appropriate dispatcher based on the current environment
exit($code);
```

При запуске приложения будет выбран диспетчер, соответствующий текущему окружению. Например, выполним команду:

```terminal
php app.php db:migrate
```

Фреймворк переберёт зарегистрированные диспетчеры и вызовет у каждого метод `canServe`. Через него диспетчер сообщает,
может ли он обработать запрос в текущем окружении. Будет использован первый диспетчер, вернувший `true`.

`ConsoleDispatcher` вернёт `true`, если приложение работает в окружении `cli`. Когда RoadRunner запускает HTTP-плагин,
он создаёт процесс воркера и передаёт ему переменную окружения `RR_MODE=http`. В этом случае будет выбран
`HttpDispatcher`.

Если ни один диспетчер не вернёт `true`, фреймворк выбросит исключение.

## Доступные диспетчеры

Spiral включает несколько встроенных диспетчеров:

- [консольный диспетчер](https://github.com/spiral/framework/blob/master/src/Framework/Console/ConsoleDispatcher.php)
  обрабатывает консольные команды приложения и позволяет создавать собственные команды для запуска из командной строки;

- [HTTP-диспетчер RoadRunner](https://github.com/spiral/roadrunner-bridge/blob/4.x/src/Http/Internal/Dispatcher.php)
  обрабатывает входящие HTTP-запросы и направляет их соответствующему действию контроллера или функции. Он используется,
  когда приложение работает как HTTP-сервис;

- [gRPC-диспетчер RoadRunner](https://github.com/spiral/roadrunner-bridge/blob/4.x/src/GRPC/Internal/Dispatcher.php)
  обрабатывает входящие gRPC-запросы и направляет их соответствующим сервисам. Он используется, когда приложение работает
  как gRPC-сервис;

- [TCP-диспетчер RoadRunner](https://github.com/spiral/roadrunner-bridge/blob/4.x/src/Tcp/Internal/Dispatcher.php)
  обрабатывает входящие TCP-соединения и направляет их соответствующему обработчику. Он используется, когда приложение
  работает как TCP-сервис;

- [диспетчер Temporal](https://github.com/spiral/temporal-bridge/blob/2.0/src/Dispatcher.php) обрабатывает входящие
  действия рабочих процессов. Temporal — распределённый, масштабируемый и отказоустойчивый движок для построения и
  оркестрации долгоживущей бизнес-логики;

- [диспетчер очередей RoadRunner](https://github.com/spiral/roadrunner-bridge/blob/4.x/src/Queue/Internal/Dispatcher.php)
  получает сообщения из очереди и направляет их соответствующему обработчику. Он полезен для систем с событийной или
  управляемой сообщениями архитектурой и используется, когда приложение работает как потребитель очереди.

> **Примечание**
> О создании собственного диспетчера читайте [здесь](../cookbook/custom-dispatcher.md).

Диспетчеры ядра Spiral предоставляют гибкий и мощный механизм направления входящих запросов подходящим обработчикам.
Это важная часть общей архитектуры фреймворка.
