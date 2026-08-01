# Основы — Сессии

Чтобы включить сессии в альтернативной сборке приложения, установите пакет Composer `spiral/session` и добавьте в
приложение загрузчик `Spiral\Bootloader\Http\SessionBootloader`.

## SessionInterface

Доступ к пользовательской сессии предоставляет контекстный объект `Spiral\Session\SessionInterface`:

```php app/src/Endpoint/Web/HomeController.php
use Spiral\Session\SessionInterface;

// ...

public function index(SessionInterface $session): void
{
    $session->resume();
    dump($session->getID());
}
```

> **Примечание**
> Нельзя сохранять ссылку на сессию в singleton-объектах. Ниже показан подходящий обходной способ.

## Раздел сессии

По умолчанию не следует работать с сессией напрямую. Вместо этого выделите изолированный именованный раздел,
предоставляющий методы `set`, `get`, `delete` и другие операции. Для этого используйте метод `getSection` объекта
сессии:

```php app/src/Endpoint/Web/HomeController.php
public function index(SessionInterface $session): void
{
    $cart = $session->getSection('cart');

    $cart->set('items', ['my-items']);

    dump($cart->getAll());
}
```

## Область сессии

Чтобы упростить работу с сессией в singleton-сервисах и контроллерах, используйте `Spiral\Session\SessionScope`. Этот
компонент также доступен через прототипное свойство `session`. Его можно использовать внутри singleton-сервисов: он
всегда указывает на активный контекст сессии.

```php app/src/Endpoint/Web/HomeController.php
use Spiral\Prototype\Traits\PrototypeTrait;

class HomeController
{
    use PrototypeTrait;

    public function index(): void
    {
        dump($this->session->getSection('cart')->getAll());
    }
}
```

## Жизненный цикл сессии

Сессия автоматически запускается при первом обращении к данным и сохраняется, когда запрос покидает
`SessionMiddleware`. Для ручного управления используйте методы объекта `Spiral\Session\SessionInterface`.

> **Примечание**
> `SessionScope` полностью реализует `SessionInterface`.

### Возобновление сессии

Чтобы вручную возобновить или создать сессию:

```php
$this->session->resume();
```

### Сохранение

Чтобы вручную сохранить и закрыть сессию:

```php
$this->session->commit();
```

### Отмена

Чтобы отменить все изменения и закрыть сессию:

```php
$this->session->abort();
```

### Получение идентификатора сессии

Чтобы получить идентификатор сессии — только после её возобновления:

```php
dump($this->session->getID());
```

Чтобы проверить, запущена ли сессия:

```php
dump($this->session->isStarted());
```

### Уничтожение

Чтобы уничтожить сессию и всё её содержимое:

```php
$this->session->destroy();
``` 

### Обновление идентификатора

Чтобы выдать новый идентификатор сессии, не изменяя её содержимое:

```php
$this->session->regenerateID();
```

## Пользовательская конфигурация

Чтобы изменить настройки сессий, создайте файл `app/config/session.php` и переопределите нужные значения.

Компонент сессий основан на нативной реализации PHP. По умолчанию содержимое сессий хранится в файловой системе в
каталоге `runtime/session`. Если нагрузка приложения распределяется между несколькими веб-серверами, выберите
централизованное хранилище, доступное всем серверам, например Redis.

Параметр конфигурации `handler` определяет, где будут храниться данные сессии каждого запроса. Spiral поставляется с
несколькими готовыми обработчиками.

### Конфигурация **FileHandler**

Сессии хранятся в каталоге `runtime/session`.

```php app/config/session.php
use Spiral\Core\Container\Autowire;
use Spiral\Session\Handler\FileHandler;

return [
    'lifetime' => 86400,
    'cookie' => 'sid',
    'secure' => false,
    'handler' => new Autowire(
        FileHandler::class,
        [
            'directory' => directory('runtime') . 'session',
            'lifetime'  => 86400
        ]
    )
];
```

### Конфигурация **CacheHandler**

Сессии хранятся в одном из кеш-хранилищ, настроенных в компоненте Cache.

```php app/config/session.php
use Spiral\Core\Container\Autowire;
use Spiral\Session\Handler\CacheHandler;

$ttl = 86400;

return [
    'lifetime' => $ttl,
    'cookie' => 'sid',
    'secure' => false,
    'handler' => new Autowire(
        CacheHandler::class,
        [
            'storage' => 'my-storage', // (Optional)  Cache storage name. Default - current cache storage
            'ttl' => $ttl,
            'prefix' => 'foo:' // (Optional) By default, session:
        ]
    )
];
```

### Пользовательский обработчик сессий

Если ни один из встроенных обработчиков не соответствует требованиям приложения, можно написать собственный. Он должен
реализовывать встроенный интерфейс PHP
[`SessionHandlerInterface`](https://www.php.net/manual/en/class.sessionhandlerinterface.php).

```php app/config/session.php
return [
    'handler' => new Autowire(
        MemoryHandler::class,
        [
            'driver' => 'redis',
            'database' => 1,
            'lifetime' => 86400
        ]
    )
];
```

> **Примечание**
> Вместо имени класса можно использовать `Autowire`, чтобы передать дополнительные параметры.

### Настройка инициализации сессии

Сессия создаётся специальной фабрикой `Spiral\Session\SessionFactoryInterface`.

```php
namespace Spiral\Session;

interface SessionFactoryInterface
{
    /**
     * @param string $clientSignature User specific token, does not provide full security but
     *                                     hardens session transfer.
     * @param string|null $id When null - expect php to create session automatically.
     */
    public function initSession(string $clientSignature, string $id = null): SessionInterface;
}
```

Стандартную реализацию `Spiral\Session\SessionFactoryInterface` можно заменить в контейнере собственной:

```php
$container->bindSingleton(\Spiral\Session\SessionFactoryInterface::class, CustomSessionFactory::class);
```
