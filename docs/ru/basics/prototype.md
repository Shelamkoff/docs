# Основы — Прототипирование

Прототипирование — этап разработки, на котором определяется структура приложения и быстро проверяется базовый дизайн
классов и их взаимодействий. Spiral предоставляет мощное расширение, ускоряющее разработку сервисов, контроллеров,
middleware и других классов за счёт изменения AST — фактически оно пишет часть кода за разработчика. Расширение также
добавляет удобные для IDE подсказки для распространённых компонентов фреймворка и репозиториев Cycle.

## Установка

Добавьте `Spiral\Prototype\Bootloader\PrototypeBootloader` в класс приложения.

:::: tabs

::: tab С помощью метода

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Prototype\Bootloader\PrototypeBootloader::class,
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
    \Spiral\Prototype\Bootloader\PrototypeBootloader::class,
    // ...
];
```

Подробнее о загрузчиках читайте в разделе [Фреймворк — Загрузчики](../framework/bootloaders.md).
:::

::::

Выполните `php app.php configure`, чтобы создать вспомогательные файлы автодополнения IDE для приложения.

## Использование прототипных свойств

### Прототипирование

Чтобы использовать прототипирование, добавьте трейт `Spiral\Prototype\Traits\PrototypeTrait` в нужный класс. Он
позволяет IDE находить и предлагать другие части приложения без предварительного объявления всех зависимостей.

![Подсказки IDE](https://user-images.githubusercontent.com/796136/67619538-8f0c8c80-f805-11e9-9cd8-0597133bf33a.gif)

**Пример добавления трейта в контроллер:**

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use Spiral\Prototype\Traits\PrototypeTrait;

class HomeController
{
    use PrototypeTrait;

    public function index()
    {
        return $this->views->render('profile', [
            'user' => $this->users->findByName('Antony')
        ]);
    }
}
```

После подключения трейта IDE начинает предлагать доступные сервисы и компоненты. Например, при вводе `$this->users`
она может предложить методы пользовательского сервиса или репозитория.

Это удобно для быстрой проверки идей, поскольку на раннем этапе не требуется формально настраивать внедрение всех
зависимостей.

### Преобразование прототипа в обычный код

Магические свойства удобны для прототипирования, но в долгосрочной перспективе хуже с точки зрения производительности и
понятности кода. Когда прототип готов, Spiral может заменить магические свойства обычным внедрением зависимостей.

Выполните команду:

```terminal
php app.php prototype:inject -r
```

Spiral найдёт магические свойства и добавит необходимый код внедрения зависимостей.

> **Примечание**
> Параметр `-r` удаляет `PrototypeTrait` из класса.

**Код после выполнения команды:**

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use App\Database\Repository\UserRepository;
use Spiral\Views\ViewsInterface;

class HomeController
{
    public function __construct(
        private readonly ViewsInterface $views, 
        private readonly UserRepository $users
    ) {
    }

    public function index(): string
    {
        return $this->views->render('profile', [
            'user' => $this->users->findByName('Antony')
        ]);
    }
}
```

### Поиск классов с прототипными свойствами

Чтобы вывести все классы, использующие прототипные свойства, выполните:

```terminal
php app.php prototype:usage
```

Команда выведет список классов:

```terminal
+--------------------------------------------------------------------+----------------------+--------------------------------------------------------------+
| Class:                                                             | Property:            | Target:                                                      |
+--------------------------------------------------------------------+----------------------+--------------------------------------------------------------+
| App\Endpoint\Web\Controller\User\SetupPasswordAction               | userService          | App\Service\UserServiceInterface                             |
|                                                                    | response             | Spiral\Http\ResponseWrapper                                  |
|                                                                    | views                | Spiral\Views\ViewsInterface                                  |
| App\Endpoint\Web\Controller\User\SetupPasswordFormAction           | users                | App\Repository\UserRepositoryInterface                       |
|                                                                    | response             | Spiral\Http\ResponseWrapper                                  |
|                                                                    | views                | Spiral\Views\ViewsInterface                                  |
|                                                                    | request              | Spiral\Http\Request\InputManager                             |
|                                                                    | sessionErrors        | App\Application\HTTP\SessionErrorsInterface                  |
| App\Endpoint\Web\Controller\Auth\LoginFormAction                   | response             | Spiral\Http\ResponseWrapper                                  |
|                                                                    | views                | Spiral\Views\ViewsInterface                                  |
|                                                                    | request              | Spiral\Http\Request\InputManager                             |
|                                                                    | sessionErrors        | App\Application\HTTP\SessionErrorsInterface                  |
+--------------------------------------------------------------------+----------------------+--------------------------------------------------------------+
```

> **Примечание**
> После завершения всех внедрений расширение `spiral/prototype` можно удалить.

### Проверка изменений

После преобразования проверьте изменения и убедитесь, что зависимости настроены правильно. Инструмент автоматизирует
основную работу, но полученный код может потребовать ручной корректировки или оптимизации.

После проверки можно продолжать разработку на основе обычного типизированного кода.

## Пользовательские свойства

Через `Spiral\Prototype\Bootloader\PrototypeBootloader` в загрузчике можно зарегистрировать любое количество
прототипных свойств:

```php
use Spiral\Prototype\Bootloader\PrototypeBootloader;

public function boot(PrototypeBootloader $prototype): void
{
    $prototype->bindProperty('myService', MyService::class);
}
```

> **Примечание**
> Этот подход можно сочетать с автоматическим обнаружением классов для более глубокой интеграции доменного слоя в процесс
> разработки.

## Регистрация атрибутом

Прототипные классы и сервисы также можно регистрировать атрибутами. Добавьте
`Spiral\Prototype\Annotation\Prototyped` к внедряемому классу:

```php app/src/Domain/User/Service/UserService.php
namespace App\Domain\User\Service;

use Spiral\Prototype\Annotation\Prototyped;

#[Prototyped(property: 'userService')]
final class UserService
{
    // ...
}
```

Выполните `php app.php update` или `php app.php prototype:dump`, чтобы автоматически обнаружить сервис.

> **Предупреждение**
> Для использования атрибутов с интерфейсами необходимо включить поиск интерфейсов. Подробнее читайте в разделе
> [Настройка слушателей](../advanced/tokenizer.md#configuring-listeners).

## Доступные короткие имена

Чтобы вывести все зарегистрированные короткие имена приложения, выполните:

```terminal
php app.php prototype:list
```

### Короткие имена компонентов

| Свойство     | Компонент                                                                                    |
|--------------|----------------------------------------------------------------------------------------------|
| app          | App\App или класс, реализующий `Spiral\Boot\Kernel`                                         |
| classLocator | Spiral\Tokenizer\ClassesInterface                                                            |
| console      | Spiral\Console\Console                                                                       |
| container    | Psr\Container\ContainerInterface                                                             |
| db           | Cycle\Database\DatabaseInterface — требуется `spiral/cycle-bridge`                          |
| dbal         | Cycle\Database\DatabaseProviderInterface — требуется `spiral/cycle-bridge`                  |
| encrypter    | Spiral\Encrypter\EncrypterInterface                                                          |
| env          | Spiral\Boot\EnvironmentInterface                                                             |
| files        | Spiral\Files\FilesInterface                                                                  |
| guard        | Spiral\Security\GuardInterface                                                               |
| http         | Spiral\Http\Http                                                                             |
| i18n         | Spiral\Translator\TranslatorInterface                                                        |
| input        | Spiral\Http\Request\InputManager                                                             |
| session      | Spiral\Session\SessionScope                                                                  |
| cookies      | Spiral\Cookies\CookieManager                                                                 |
| logger       | Psr\Log\LoggerInterface                                                                      |
| logs         | Spiral\Logger\LogsInterface                                                                  |
| memory       | Spiral\Boot\MemoryInterface                                                                  |
| orm          | Cycle\ORM\ORMInterface — требуется `spiral/cycle-bridge`                                    |
| paginators   | Spiral\Pagination\PaginationProviderInterface                                                |
| queue        | Spiral\Queue\QueueInterface                                                                  |
| queueManager | Spiral\Queue\QueueConnectionProviderInterface                                                |
| request      | Spiral\Http\Request\InputManager                                                             |
| response     | Spiral\Http\ResponseWrapper                                                                  |
| router       | Spiral\Router\RouterInterface                                                                |
| server       | Spiral\Goridge\RPC — требуется `spiral/roadrunner-bridge`                                     |
| snapshots    | Spiral\Snapshots\SnapshotterInterface                                                        |
| storage      | Spiral\Storage\StorageInterface                                                              |
| validator    | Spiral\Validation\ValidationInterface                                                        |
| views        | Spiral\Views\ViewsInterface                                                                  |
| auth         | Spiral\Auth\AuthScope                                                                        |
| authTokens   | Spiral\Auth\TokenStorageInterface                                                            |
| cache        | Psr\SimpleCache\CacheInterface                                                               |
| cacheManager | Spiral\Cache\CacheStorageProviderInterface                                                   |
