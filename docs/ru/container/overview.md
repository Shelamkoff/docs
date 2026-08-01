# Контейнер — Обзор

Spiral предоставляет набор инструментов, упрощающих управление зависимостями и создание объектов. Один из основных
инструментов — контейнер: он обрабатывает зависимости классов и автоматически внедряет их. Вместо ручного создания
объектов и настройки зависимостей эту работу выполняет контейнер.

## Контейнер PSR-11

Контейнер Spiral соответствует стандарту [PSR-11](https://github.com/php-fig/container), что обеспечивает совместимость
с другими библиотеками.

Получить доступ к контейнеру в коде можно через `Psr\Container\ContainerInterface`.

```php app/src/Endpoint/Web/UserController.php
final class UserController
{
    public function __construct(
        private readonly \Psr\Container\ContainerInterface $container
    ) {}

    public function show(string $id): void
    {
       $repository = $this->container->get(UserRepository::class);
       // ...
    }
}
```

## Внедрение зависимостей

Контейнер Spiral поддерживает внедрение зависимостей через **конструктор** и **метод**. Зависимости автоматически
передаются классу при его создании либо при вызове определённого метода.

:::: tabs

::: tab Конструктор

В классе `UserController` зависимость `UserRepository` внедряется через метод `__construct()`. Контейнер автоматически
создаёт и передаёт объект `UserRepository` при создании контроллера.

```php app/src/Endpoint/Web/UserController.php
use Psr\Container\ContainerInterface;

final class UserController
{
    public function __construct(
        private readonly UserRepository $users
    ) {}

    public function show(string $id): void
    {
       $user = $this->users->findOrFail($id);
       // ...
    }
}
```

:::

::: tab Метод

В следующем примере зависимость `UserRepository` внедряется через метод `show()`. Контейнер автоматически создаёт и
передаёт репозиторий при вызове метода.

```php app/src/Endpoint/Web/UserController.php
final class UserController
{
    public function show(UserRepository $users, string $id): void
    {
       $user = $users->findOrFail($id);
       // ...
    }
}
```

:::
::::

> **Примечание**
> Внедрение в методы доступно, например, в контроллерах, консольных командах и заданиях очереди. Контейнер также
> поддерживает объединённые типы, вариативные и ссылочные аргументы, а также значения объектов по умолчанию при
> автоматическом разрешении зависимостей.

### Автоматическое разрешение зависимостей

Контейнер фреймворка автоматически разрешает зависимости конструктора или метода, создавая экземпляры конкретных
классов.

```php
class MyController
{
    public function __construct(
        OtherClass $class, 
        SomeInterface $some
    ) {
    }
}
```

В этом примере контейнер попытается автоматически создать экземпляр `OtherClass`. Интерфейс `SomeInterface` разрешить
не получится, пока в контейнере не будет зарегистрирована соответствующая связь.

```php
$container->bind(SomeInterface::class, SomeClass::class); 
```

Контейнер пытается разрешить все зависимости конструктора, кроме значений, переданных вручную. Поэтому каждая
зависимость должна быть доступна либо параметр должен быть необязательным.

```php
// will fail if `value` dependency not provided
__construct(OtherClass $class, $value)

// will use `null` as `value` if no other value provided
__construct(OtherClass $class, $value = null) 

// will fail if SomeInterface does not point to the concrete implementation
__construct(OtherClass $class, SomeInterface $some) 

// will use null as value of `some` if no concrete implementation is provided
__construct(OtherClass $class, SomeInterface $some = null) 
```

## Сервисы

### Binder

Фреймворк предоставляет `Spiral\Core\BinderInterface`, позволяющий связать класс с интерфейсом или псевдонимом.
Подробнее читайте в разделе [Контейнер — Конфигурация](../container/configuration.md).

### Фабрика

Интерфейс `Spiral\Core\FactoryInterface` позволяет создать класс, передав вручную часть зависимостей конструктора.
Остальные зависимости будут разрешены контейнером автоматически.

```php
public function makeClass(FactoryInterface $factory): MyClass
{
    return $factory->make(UserService::class, [
        'table' => 'users'
    ]); 
}
```

В этом примере метод `make()` создаёт экземпляр `UserService` и передаёт значение `table` соответствующему параметру
конструктора. Остальные зависимости разрешаются контейнером.

Фабрика предоставляет более точный контроль над созданием объектов. Она особенно полезна, когда необходимо создать
несколько экземпляров одного класса с разными значениями параметров конструктора.

### Resolver

Интерфейс `Spiral\Core\ResolverInterface` разрешает аргументы методов динамических целей, например действий
контроллера. Он полезен, когда метод вызывается во время выполнения и его зависимости необходимо получить из контейнера.

```php
abstract class Handler
{
    public function __construct(
        protected readonly ResolverInterface $resolver
    ) {
    }

    public function run(array $params): bool
    {
        $method = new \ReflectionMethod($this, 'do');

        return $method->invokeArgs(
            $this, 
            $this->resolver->resolveArguments($method, $params) // resolve missing arguments
        );
    }
}
```

Метод `run()` использует `ResolverInterface`, чтобы вызвать `do` с внедрением зависимостей в метод.

```php
class UserStoreHandler extends Handler
{
    public function do(UserService $service): bool
    {
        // Store user
    }
}
```

Таким способом можно разрешить зависимости метода во время выполнения и вызвать его с необходимыми аргументами
независимо от того, были ли они переданы вручную или должны быть получены из контейнера.

#### Поддерживаемые типы

##### Объединённые типы

Стандартная реализация `ResolverInterface` поддерживает объединённые типы. Будет передана одна из доступных зависимостей
подходящего типа.

```php
use Doctrine\Common\Annotations\Reader;
use Spiral\Attributes\ReaderInterface;

final class Entities
{
    public function __construct(
        private Reader|ReaderInterface $reader
    ) {
    }
}
```

##### Вариативные аргументы

```php
$resolver = $this->container->get(ResolverInterface::class);
$function = static fn(int ...$bar) => $bar;

// array passed by parameter name
$args = $resolver->resolveArguments(
    new \ReflectionFunction($function),
    ['bar' => [1, 2]]
);

dump($args); // [1, 2]

// array passed by parameter name with named arguments inside
$args = $resolver->resolveArguments(
    new \ReflectionFunction($function),
    ['bar' => ['ab' => 1, 'bc' => 2]]
);

dump($args); // ['ab' => 1 'bc' => 2]

// value passed by parameter name
$args = $resolver->resolveArguments(
    new \ReflectionFunction($function),
    ['bar' => 1]
);

dump($args); // [1]

```

##### Ссылочные аргументы

```php
$resolver = $this->container->get(ResolverInterface::class);
$function = static fn(int $bar) => $bar;

$bar = 1;

$args = $resolver->resolveArguments(
    new \ReflectionFunction($function),
    ['bar' => &$bar]
);

$bar = 42;
dump($args); // [42]
```

##### Объект по умолчанию

```php
$resolver = $this->container->get(ResolverInterface::class);
$function = static fn(stdClass $std = new \stdClass()) => $std;

$args = $resolver->resolveArguments(new \ReflectionFunction($function));

dump($args); 

// array(1) {
//   [0] =>
//   class stdClass#369 (0) {
//   }
// }
```

#### Проверка аргументов

Аргументы функции или метода можно проверить публичным методом `validateArguments`. Передайте ему
`ReflectionMethod` или `ReflectionFunction` и массив аргументов. Если аргументы были получены через
`resolveArguments` и параметру `$validate` не передано `false`, дополнительная проверка не требуется: она выполняется
автоматически.

При недопустимых аргументах выбрасывается `Spiral\Core\Exception\Resolver\InvalidArgumentException`.

```php
$resolver = $this->container->get(ResolverInterface::class);
$function = static fn(int $bar) => $bar;

$resolver->validateArguments(new \ReflectionFunction($function), [42]);
```

### Invoker

Интерфейс `Spiral\Core\InvokerInterface` вызывает методы и автоматически разрешает их зависимости. Это полезно, когда
метод объекта необходимо вызвать во время выполнения с аргументами из контейнера.

#### Вызов метода класса

Метод `invoke()` вызывает метод объекта и позволяет передать определённые значения его зависимостей.

```php
use Spiral\Core\InvokerInterface;

abstract class Handler
{
    public function __construct(
        protected readonly InvokerInterface $invoker
    ) {
    }

    public function run(array $params): bool
    {
        return $this->invoker->invoke([$this, 'do'], $params)
    }
}
```

Если первым элементом callable-массива передана строка, например `['user-service', 'store']`, объект с таким
псевдонимом будет запрошен из контейнера.

```php
$container->bind('user-service', UserService::class);
// ...
$invoker->invoke(
    ['user-service', 'store'], 
    $params
);
```

Контейнер разрешит сервис `user-service` и вызовет его метод `store`.

Так можно обращаться к классам, управляемым контейнером, не создавая их вручную. Это особенно удобно для сервисов,
методы которых вызываются из разных частей приложения.

> **Примечание**
> Вызываемый метод может иметь любую видимость: `public`, `protected` или `private`.

#### Вызов callable

`InvokerInterface` также вызывает замыкания и автоматически разрешает их зависимости.

```php
$invoker->invoke(
    function (MyClass $class, string $parameter) {
        return new MyClassService($class);
    },
    [
        'parameter' => 'value',
    ]
); 
```

Замыкание получает необходимые зависимости независимо от того, были ли они переданы в массиве параметров или разрешены
контейнером.

### Области

Важная часть разработки долгоживущих приложений — правильное управление контекстом. В демонизированных приложениях
нельзя считать пользовательский запрос глобальным singleton-объектом и хранить ссылки на него в сервисах.

На практике это означает, что контекст необходимо запрашивать явно во время обработки пользовательского ввода. Spiral
решает эту задачу с помощью глобального IoC-контейнера, который выступает переносчиком контекста. Через него можно
получать экземпляры, ограниченные определённой областью, как если бы они были глобальными объектами.

Подробнее об областях читайте в разделе [Контейнер — Области IoC](../container/scopes.md).

## Замена контейнера приложения

При необходимости контейнер приложения можно заменить во время создания экземпляра приложения.

```php app.php
use Spiral\Core\Container;
use App\Application\Kernel;

$container = new Container();
$container->bind(...);

$app = Kernel::create(
    directories: ['root' => __DIR__],
    container: $container
)
```
