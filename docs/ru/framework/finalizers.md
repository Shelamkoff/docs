# Фреймворк — Финализаторы

Большинство компонентов фреймворка не требуется сбрасывать после завершения запроса. Однако существуют случаи, когда
после обработки пользовательского запроса библиотеку необходимо вернуть в исходное состояние.

> **Примечание**
> По возможности используйте области IoC вместо финализаторов.

## FinalizerInterface

Используйте `Spiral\Boot\FinalizerInterface`:

```php
/**
 * Used to close resources and connections for long-running processes.
 */
interface FinalizerInterface
{
    /**
     * Finalizers are executed after every request and used for garbage collection
     * or to close open connections.
     *
     * @param callable $finalizer
     */
    public function addFinalizer(callable $finalizer);
    
    /**
     * Finalize execution.
     *
     * @param bool $terminate Set to true if finalization is caused on application termination.
     */
    public function finalize(bool $terminate = false);
}
```

Все диспетчеры приложения вызывают финализатор в следующих случаях:

* HTTP-запрос успешно завершён;
* HTTP-запрос завершён ошибкой;
* задание успешно завершено;
* задание завершено ошибкой;
* gRPC-вызов успешно завершён;
* gRPC-вызов завершён ошибкой;
* консольная команда завершена.

> **Предупреждение**
> Финализатор вызывается только при запуске конкретного диспетчера. Команды приложения и HTTP-методы можно вызывать
> напрямую, без диспетчера, но в таком случае сервисы после каждого запроса автоматически сбрасываться не будут.

Обработчик получает первым аргументом логическое значение, указывающее, завершит ли приложение работу после запроса.

> **Примечание**
> Не сбрасывайте настройки IoC в финализаторе: singleton-сервис может сохранить в кеше прежнюю версию зависимости.

## Пример финализатора

С помощью финализатора можно автоматически закрывать соединение с базой данных после каждого запроса. Это полезно при
большом количестве воркеров или лямбда-функций, когда не следует занимать все доступные сокеты базы данных.

```php
// in bootloader
use Spiral\Boot\FinalizerInterface;
use Psr\Container\ContainerInterface;
use Cycle\Database\DatabaseManager;

public function boot(FinalizerInterface $finalizer, ContainerInterface $container): void
{
    $finalizer->addFinalizer(function () use ($container) {
        /** @var DatabaseManager $dbal */
        $dbal = $container->get(DatabaseManager::class);
 
        foreach ($dbal->getDrivers() as $driver) {
            $driver->disconnect();
        }
    });
}
```

> **Примечание**
> Такой загрузчик уже включён в пакет
> [`spiral\cycle-bridge`](https://github.com/spiral/cycle-bridge/blob/master/src/Bootloader/DisconnectsBootloader.php)
> и доступен как `Spiral\Cycle\Bootloader\DisconnectsBootloader`.
