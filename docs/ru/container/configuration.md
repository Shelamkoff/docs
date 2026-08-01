# Контейнер — Конфигурация

Контейнер Spiral настраивается с помощью связей между интерфейсами или псевдонимами и конкретными реализациями.
Определять такие связи можно в [загрузчиках](../framework/bootloaders.md).

## Обзор

Контейнер можно настраивать через `Spiral\Core\BinderInterface` или непосредственно через
`Spiral\Core\Container`.

### Связывание интерфейса с реализацией

Чтобы связать интерфейс с конкретной реализацией, используйте следующий код:

:::: tabs

::: tab BinderInterface

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Core\BinderInterface;

public function boot(BinderInterface $binder): void
{
    $binder->bind(
        UserRepositoryInterface::class, 
        CycleUserRepository::class
    );
}
```

:::

::: tab Container

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Core\Container;

public function boot(Container $container): void
{
    $container->bind(
        UserRepositoryInterface::class, 
        CycleUserRepository::class
    );
}
```

:::
::::

### Связывание интерфейса с singleton-объектом

Чтобы зарегистрировать singleton-связь:

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Core\BinderInterface;

public function boot(BinderInterface $binder): void
{
    $binder->bindSingleton(
        UserRepositoryInterface::class, 
        CycleUserRepository::class
    );
}
```

Для передачи определённых параметров используйте класс `Autowire`:

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Core\BinderInterface;
use Spiral\Core\Container\Autowire;

public function boot(BinderInterface $binder): void
{
    $binder->bindSingleton(
        UserRepositoryInterface::class, 
        new Autowire(CycleUserRepository::class, ['table' => 'users'])
    );
}
```

Класс также можно автоматически настроить замыканием, переданным методу `bind` или `bindSingleton`:

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Core\BinderInterface;

public function boot(BinderInterface $binder): void
{
    $binder->bindSingleton(
        UserRepositoryInterface::class, 
        static fn() => new CycleUserRepository(table: users)
    );
}
```

Замыкания поддерживают внедрение зависимостей:

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Core\BinderInterface;
use Spiral\Core\Container\Autowire;

public function boot(BinderInterface $binder): void
{
    $binder->bindSingleton(
        UserRepositoryInterface::class, 
        static fn(UserConfig $config) => new CycleUserRepository(table: $config->getTable())
    );
}
```

При выполнении замыкания контейнер автоматически разрешит `UserConfig` и передаст его аргументом. Это позволяет
настраивать классы с зависимостями без ручного создания и управления ими.

### Проверка наличия связи

Чтобы проверить наличие связи в контейнере:

```php
use Spiral\Core\Container;

public function boot(Container $container): void
{
    $container->has(UserRepositoryInterface::class)
}
```

### Удаление связи

Чтобы удалить связь:

```php
use Spiral\Core\BinderInterface;

public function boot(BinderInterface $binder): void
{
    $binder->removeBinding(UserRepositoryInterface::class)
}
```

Контейнер поддерживает связи `WeakReference`.

:::: tabs

::: tab Строковый псевдоним

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Core\Container;
use WeakReference;

public function boot(Container $container): void
{
    $object = new stdClass();
    $hash = \spl_object_hash($object);
    $reference = WeakReference::create($object);

    $container->bind('test-alias', $reference);
    
    dump($hash === \spl_object_hash($container->get('test-alias'))); // true
    
    unset($object);
    // New object can't be created because classname has not been stored
    dump($container->get('test-alias')); // null
}
```

:::

::: tab Псевдоним имени класса

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Core\Container;
use WeakReference;

public function boot(Container $container): void
{
    $object = new stdClass();
    $hash = \spl_object_hash($object);
    $reference = WeakReference::create($object);

    $container->bind(stdClass::class, $reference);
    
    dump($hash === \spl_object_hash($container->get(stdClass::class))); // true
    
    unset($object);
    // new instance created using alias class
    dump($hash === \spl_object_hash($container->get(stdClass::class))); // false
}
```

:::
::::

## Расширенные связи

Начиная с версии **3.8.0 Spiral Framework**, прежняя структура массивов для хранения сведений о связях заменена
объектами передачи данных — DTO. Новый подход предоставляет более структурированный и упорядоченный способ хранения и
настройки связей контейнера.

Пример новой конфигурации:

```php
use Spiral\Core\Config\Factory;

$container->bind(LoggerInterface::class, new Factory(
    callable: static function() {
        return new Logger(....);
    }, 
    singleton: true
))
```

Доступны следующие DTO связей.

### Alias

Упрощённый DTO создаёт ссылку на другую связь контейнера.

```php
use Spiral\Core\Config\Alias;

$container->bind(
    \DatetimeImmutable::class, 
    static fn() => new \DatetimeImmutable()
);

$container->bind(
    'now', 
    new Alias(alias: \DatetimeImmutable::class)
);
```

Сначала класс `\DatetimeImmutable` связывается с фабрикой, создающей новый экземпляр при каждом запросе. Затем создаётся
связь-псевдоним `now`, указывающая на связь `\DatetimeImmutable`.

Теперь класс `\DatetimeImmutable` можно запросить из контейнера по псевдониму `$container->get('now')`.

`Alias` также принимает второй аргумент — `singleton`. При значении `true` объект, получаемый через псевдоним, становится
singleton-объектом. Каждый вызов `$container->get('now')` будет возвращать один экземпляр. При этом прямой вызов
`$container->get(\DatetimeImmutable::class)` по-прежнему создаёт новый объект с текущим временем.

### Autowire

Связь `Spiral\Core\Config\Autowire` служит оболочкой над классом `Spiral\Core\Container\Autowire` и автоматически
создаёт классы, внедряя их зависимости.

```php
use Spiral\Core\Config\Autowire;
use Spiral\Core\Container\Autowire as AutowireAlias;

$container->bind(MyClass::class, new Autowire(
    autowire: new AutowireAlias(MyClass::class, ['foo' => 'bar']),
    singleton: true
));
```

Аргумент `singleton`, равный `true`, указывает контейнеру использовать `MyClass` как singleton. Вызов
`$container->get(MyClass::class)` каждый раз будет возвращать один и тот же экземпляр.

### Factory

Связь `Spiral\Core\Config\Factory` представляет простую фабрику, создающую значения любого типа через замыкание.

```php
use Spiral\Core\Config\Factory;

$container->bind('time', new Factory(
    callable: static fn() => time(),
));
```

Каждый вызов `$container->get('time')` возвращает текущую временную метку.

Фабрику также можно сделать singleton-связью:

```php
$container->bind('time', new Factory(
    callable: static fn() => time(),
    singleton: true,
));
```

При `singleton: true` каждый вызов `$container->get('time')` будет возвращать одно и то же значение.

### DeferredFactory

Связь `Spiral\Core\Config\DeferredFactory` позволяет зарегистрировать отложенную фабрику специальным
callable-массивом.

```php
use Spiral\Core\Config\DeferredFactory;

$container->bind('some-binding', new DeferredFactory(
    factory: [MyClass::class, 'handle'],
));
```

Ключ `some-binding` связывается с `DeferredFactory`, фабрика которой задана как `[MyClass::class, 'handle']`. При запросе
`$container->get('some-binding')` контейнер разрешит экземпляр `MyClass`, а затем вызовет его метод `handle`, чтобы
получить требуемое значение.

Связь также можно сделать singleton:

```php
$container->bind('some-binding', new DeferredFactory(
    factory: ...,
    singleton: true
));
```

### Scalar

Связь `Scalar` предоставляет удобный способ хранения и получения статических скалярных значений. Она подходит для
путей, констант и других скалярных параметров приложения.

```php
use Spiral\Core\Config\Scalar;

$container->bind('app-path', new Scalar(value: '/var/www/my-app'));
```

### Shared

Связь `Shared` привязывает постоянный объект к ключу контейнера. Объект повторно используется при каждом запросе ключа,
а передать новые пользовательские аргументы при последующих запросах нельзя.

```php
use Spiral\Core\Config\Shared;

$container->bind(MyClass::class, new Shared(value: new MyClass(...)));
```

Объект создаётся с исходными аргументами и затем используется без изменений. Это полезно, когда во всём приложении
должен применяться один экземпляр и нельзя допустить создания нескольких объектов с разными параметрами.

### Inflector

Инфлектор изменяет объект после его создания контейнером. Он особенно полезен для применения общих модификаций или
внедрения зависимостей во все объекты определённого типа.

```php
use Spiral\Core\Config\Inflector;

$container->bind(LoggerAwareInterface::class, new Inflector(
    inflector: static function (LoggerAwareInterface $obj, LoggerInterface $logger): LoggerAwareInterface {
        $obj->setLogger($logger);

        return $obj;
    }
));
```

В этом примере каждому объекту, реализующему `LoggerAwareInterface`, будет назначен логгер из контейнера.

Связь `Inflector` позволяет единообразно изменять и настраивать объекты приложения после создания.

### WeakReference

Связь `WeakReference` позволяет использовать в контейнере слабые ссылки. Такая ссылка не препятствует удалению объекта
сборщиком мусора, если на него больше нет сильных ссылок.

```php
se Spiral\Core\Config\WeakReference;

$obj = new MyClass();

$container->bind(MyClass::class, new WeakReference(
    reference: new \WeakReference($obj)
));

$obj === $container->get(MyClass::class); // true

unset($obj);

$obj1 = $container->get(MyClass::class); // A new object will be created
$obj1 === $container->get(MyClass::class); // true
```

Пока исходный объект существует, `$container->get(MyClass::class)` возвращает именно его. После удаления `$obj` сильная
ссылка исчезает, объект может быть собран, а следующий запрос создаст новый экземпляр `MyClass`.

Слабые ссылки полезны, когда необходимо контролировать жизненный цикл объекта и разрешить его удаление при отсутствии
сильных ссылок.

## Ленивые singleton-объекты

Фреймворк также поддерживает «ленивые singleton-объекты»: контейнер автоматически считает такие классы singleton без
явной регистрации через `bindSingleton`.

:::: tabs

::: tab Атрибут

`Spiral\Core\Attribute\Singleton` помечает класс как singleton. Контейнер создаёт только один экземпляр и повторно
использует его во всём приложении. Атрибут служит альтернативой интерфейсу-маркеру.

```php
use Spiral\Core\Attribute\Singleton;

#[Singleton]
final class UserService
{
    public function store(User $user): void
    {
        //...
    }
}
```

> **Примечание**
> Подробнее читайте в разделе [Контейнер — Атрибуты](./attributes.md).

:::

::: tab SingletonInterface

Для использования интерфейса-маркера реализуйте `Spiral\Core\Container\SingletonInterface`:

```php
use Spiral\Core\Container\SingletonInterface;

final class UserService implements SingletonInterface
{
    public function store(User $user): void
    {
        //...
    }
}
```

:::
::::

После этого контейнер автоматически создаёт для всего приложения только один экземпляр класса независимо от количества
запросов.

```php
protected function index(UserService $service): void
{
    dump($this->container->get(UserService::class) === $service);
}
```
