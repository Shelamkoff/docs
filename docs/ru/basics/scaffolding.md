# Основы — Генерация кода

Spiral предоставляет компонент `spiral/scaffolder`. С помощью набора консольных команд он позволяет быстро создавать
код различных классов приложения:

- загрузчиков приложения;
- консольных команд;
- конфигураций приложения;
- HTTP-контроллеров, middleware и фильтров запросов;
- обработчиков заданий очереди;
- и других классов.

## Установка

Чтобы включить компонент, добавьте `Spiral\Scaffolder\Bootloader\ScaffolderBootloader` в список загрузчиков.

:::: tabs

::: tab С помощью метода

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Scaffolder\Bootloader\ScaffolderBootloader::class,
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
    \Spiral\Scaffolder\Bootloader\ScaffolderBootloader::class,
    // ...
];
```

Подробнее о загрузчиках читайте в разделе [Фреймворк — Загрузчики](../framework/bootloaders.md).
:::

::::

## Конфигурация

После добавления загрузчика компонент можно настроить под требования приложения, заменив генераторы объявлений или их
параметры в конфигурационном файле `scaffolder`.

Конфигурация доступных объявлений по умолчанию:

```php 
use Spiral\Scaffolder\Declaration;

return [
    'declarations' => [
        Declaration\BootloaderDeclaration::TYPE => [
            'namespace' => 'Bootloader',
            'postfix' => 'Bootloader',
            'class' => Declaration\BootloaderDeclaration::class,
        ],
        Declaration\ConfigDeclaration::TYPE => [
            'namespace' => 'Config',
            'postfix' => 'Config',
            'class' => Declaration\ConfigDeclaration::class,
            'options' => [
                'directory' => directory('config'),
            ],
        ],
        Declaration\ControllerDeclaration::TYPE => [
            'namespace' => 'Controller',
            'postfix' => 'Controller',
            'class' => Declaration\ControllerDeclaration::class,
        ],
        Declaration\FilterDeclaration::TYPE => [
            'namespace' => 'Filter',
            'postfix' => 'Filter',
            'class' => Declaration\FilterDeclaration::class,
        ],
        Declaration\MiddlewareDeclaration::TYPE => [
            'namespace' => 'Middleware',
            'postfix' => '',
            'class' => Declaration\MiddlewareDeclaration::class,
        ],
        Declaration\CommandDeclaration::TYPE => [
            'namespace' => 'Command',
            'postfix' => 'Command',
            'class' => Declaration\CommandDeclaration::class,
        ],
        Declaration\JobHandlerDeclaration::TYPE => [
            'namespace' => 'Job',
            'postfix' => 'Job',
            'class' => Declaration\JobHandlerDeclaration::class,
        ],
    ],
];
```

Для каждого типа объявления можно изменить пространство имён, постфикс и класс генератора. Полностью переопределять
конфигурацию не требуется: достаточно указать только изменяемые типы.

Пример пользовательской конфигурации:

```php app/config/scaffolder.php
use Spiral\Scaffolder\Declaration;

return [
    // ...
    'declarations' => [
        Declaration\MiddlewareDeclaration::TYPE => [
            'class' => Declaration\MiddlewareDeclaration::class,
        ],
        Declaration\CommandDeclaration::TYPE => [
            'namespace' => 'Endpoint\Console',
        ],
        Declaration\JobHandlerDeclaration::TYPE => [
            'namespace' => 'Endpoint\Queue',
            'postfix' => 'Job',
        ],
    ],
];
```

> **Примечание**
> Такой подход особенно полезен в крупных приложениях с большим количеством типов объявлений. Настройка только
> необходимых параметров упрощает конфигурацию и снижает риск ошибок.

### Изменение каталога генерируемых классов

По умолчанию классы создаются в каталоге `app/src`. Изменить общий каталог можно параметром `directory`.

```php app/config/scaffolder.php
return [
    'directory' => directpry('app') . '/Generated' // <=============
];
```

Каталог также можно изменить для конкретного типа объявления. Например, консольные команды можно создавать в
`app/src/Endpoint/Console`:

```php app/config/scaffolder.php
return [
    'declarations' => [
        Declaration\CommandDeclaration::TYPE => [
            // ...
            'directory' => directpry('app') . '/Endpoint/Console' // <=============
        ],
    ],
];
```

## Добавление пользовательских объявлений через ScaffolderBootloader

Пользовательские объявления регистрируются через `ScaffolderBootloader`:

```php app/src/Application/Bootloader/ScaffolderBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Scaffolder\Bootloader\ScaffolderBootloader as BaseScaffolderBootloader;

final class ScaffolderBootloader extends Bootloader
{
    public function init(BaseScaffolderBootloader $scaffolder): void
    {
        $scaffolder->addDeclaration('Repository', [
            'namespace' => 'App\\Repository',
            'postfix'   => 'Repository',
            'class'     => RepositoryDeclaration::class,
            'options'   => [
                'orm' => 'cycle',
                // some custom options
            ],
        ]);
    }
}
```

Пользовательские объявления позволяют расширить компонент в соответствии с требованиями конкретного приложения.

## Доступные команды

| Команда           | Описание                              |
|-------------------|---------------------------------------|
| create:bootloader | Создать класс загрузчика              |
| create:command    | Создать консольную команду            |
| create:config     | Создать объект конфигурации           |
| create:controller | Создать контроллер                    |
| create:middleware | Создать middleware                    |
| create:filter     | Создать фильтр запроса                |
| create:jobHandler | Создать обработчик задания            |

Некоторые пакеты предоставляют собственные команды. Например, установленный пакет `Cycle Bridge` добавляет:

| Команда           | Описание                              |
|-------------------|---------------------------------------|
| create:migration  | Создать миграцию                      |
| create:repository | Создать репозиторий сущности          |
| create:entity     | Создать сущность                      |

> **Смотрите также**
> Подробнее о пакете `Cycle Bridge` и его консольных командах читайте
> [здесь](https://spiral.dev/docs/packages-cycle-bridge).

### Загрузчик

Команда создаёт класс загрузчика. Загрузчики инициализируют и настраивают компоненты при запуске приложения.
Сгенерированный класс можно дополнить необходимой логикой.

> **Смотрите также**
> Подробнее читайте в разделе [Фреймворк — Загрузчики](../framework/bootloaders.md).

```terminal
php app.php create:bootloader <name>
```

Будет создан класс `<Name>Bootloader`.

#### Конфигурация

В примере используется следующая конфигурация объявления:

```php
Spiral\Scaffolder\Declaration\BootloaderDeclaration::TYPE => [
    'namespace' => 'Application\Bootloader',
],
```

#### Пример

```terminal
php app.php create:bootloader App
```

Результат:

```php app/src/Application/Bootloader/AppBootloader.php
declare(strict_types=1);

namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;

final class AppBootloader extends Bootloader
{
    protected const BINDINGS = [];
    protected const SINGLETONS = [];
    protected const DEPENDENCIES = [];

    public function init(): void
    {
    }

    public function boot(): void
    {
    }
}
```

Параметр `-d` создаёт доменный загрузчик.

#### Пример

```terminal
php app.php create:bootloader App -d
```

Результат:

```php app/src/Application/Bootloader/AppBootloader.php
declare(strict_types=1);

namespace App\Application\Bootloader;

use Spiral\Bootloader\DomainBootloader;
use Spiral\Core\CoreInterface;

final class AppBootloader extends DomainBootloader
{
    protected const BINDINGS = [];
    protected const SINGLETONS = [
        CoreInterface::class => [self::class, 'domainCore']
    ];
    protected const DEPENDENCIES = [];
    protected const INTERCEPTORS = [
        // Put your interceptors here
    ];

    public function init(): void
    {
    }

    public function boot(): void
    {
    }
}
```

### Консольная команда

Команда создаёт класс консольной команды. Консольные команды позволяют запускать функции приложения из командной
строки. Сгенерированный класс можно дополнить нужной логикой.

> **Смотрите также**
> Подробнее читайте в разделе [Консоль — Начало работы](../console/configuration.md).

```terminal
php app.php create:command <name> [alias]
```

Будет создан класс `<Name>Command`. Имя команды будет равно `name` или `alias`, если он указан.

Если псевдоним не передан, он создаётся автоматически из имени. Например, для `CreateUser` будет создан псевдоним
`create:user`.

#### Конфигурация

В примере используется следующая конфигурация объявления:

```php
Spiral\Scaffolder\Declaration\CommandDeclaration::TYPE => [
    'namespace' => 'Endpoint\Console',
],
```

#### Пример без `alias`

```terminal
php app.php create:command UserRegister
```

Результат:

```php app/src/Endpoint/Console/UserRegisterCommand.php
declare(strict_types=1);

namespace App\Endpoint\Console;

use Spiral\Console\Attribute\Argument;
use Spiral\Console\Attribute\AsCommand;
use Spiral\Console\Attribute\Option;
use Spiral\Console\Attribute\Question;
use Spiral\Console\Command;

#[AsCommand(name: 'user:register')]
final class UserRegisterCommand extends Command
{
    public function __invoke(): int
    {
        // Put your command logic here
        $this->info('Command logic is not implemented yet');

        return self::SUCCESS;
    }
}
```

#### Пример с псевдонимом

```terminal
php app.php create:command UserRegister create:user
```

Результат:

```php app/src/Endpoint/Console/UserRegisterCommand.php
#[AsCommand(name: 'create:user')]
final class UserRegisterCommand extends Command
```

#### Параметры и аргументы

Параметры `-a` и `-o` добавляют аргументы и опции команды.

```terminal
php app.php create:command UserRegister -a username -a password -o isAdmin
```

Результат:

```php app/src/Endpoint/Console/UserRegisterCommand.php
declare(strict_types=1);

namespace App\Endpoint\Console;

use Spiral\Console\Attribute\Argument;
use Spiral\Console\Attribute\AsCommand;
use Spiral\Console\Attribute\Option;
use Spiral\Console\Attribute\Question;
use Spiral\Console\Command;

#[AsCommand(name: 'user:register')]
final class UserRegisterCommand extends Command
{
    #[Argument(description: 'Argument description')]
    #[Question(question: 'What would you like to name the username argument?')]
    private string $username;

    #[Argument(description: 'Argument description')]
    #[Question(question: 'What would you like to name the password argument?')]
    private string $password;

    #[Option(description: 'Argument description')]
    private bool $isAdmin;

    public function __invoke(): int
    {
        // Put your command logic here
        $this->info('Command logic is not implemented yet');

        return self::SUCCESS;
    }
}
```

#### Описание команды

Параметр `-d` добавляет описание команды.

```terminal
php app.php create:command UserRegister -d "Register a new user"
```

Результат:

```php app/src/Endpoint/Console/UserRegisterCommand.php
#[AsCommand(name: 'create:user', description: 'Register a new user')]
final class UserRegisterCommand extends Command
```

### Конфигурация приложения

Команда создаёт класс конфигурации. Он предоставляет типизированный способ управления настройками приложения.
Сгенерированный класс можно дополнить методами для необходимых параметров.

> **Смотрите также**
> Подробнее читайте в разделе [Фреймворк — Объекты конфигурации](../framework/config.md).

```terminal
php app.php create:config <name>
```

Будут созданы класс `<Name>Config` и файл `<app directory>/config/<name>.php`, если он ещё не существует.

#### Доступные параметры

`reverse (r)` — генератор найдёт файл `<app directory>/config/<name>.php` и создаст класс на основе его содержимого.
Класс будет содержать значения по умолчанию и методы получения. Для некоторых массивов также создаются методы получения
по ключу. Если массив содержит несколько элементов с одинаковыми типами ключей и значений, генератор попытается создать
такой метод. При конфликте имени с существующим методом он будет пропущен.

#### Пример с пустым конфигурационным файлом

```terminal
php app.php create:config app
```

Созданный конфигурационный файл:

```php app/config/app.php
return [];
```

Созданный класс:

```php app/src/Application/Config/AppConfig.php
declare(strict_types=1);

namespace App\Application\Config;

use Spiral\Core\InjectableConfig;

final class AppConfig extends InjectableConfig
{
    public const CONFIG = 'app';

    /**
     * Default values for the config.
     * Will be merged with application config in runtime.
     */
    protected array $config = [];
}
```

#### Пример обратной генерации

```php app/config/app.php
return [
    //will create "getParam()" by-key-getter (successfully singularized name)
    'params' => [
        'one' => 'param',
        'two' => 'another param',
    ],
    //will create "getParameterBy()" by-key-getter (unsuccessfully singularized name)
    'parameter' => [
        'one' => 'parameter',
        'two' => 'another parameter',
    ],
    //will create "getValueBy()" by-key-getter (because "getValue()" conflicts with the next "value" field)
    'values' => [
        1 => 'value',
        2 => 'another value',
    ],
    'value' => 'third value',
    //won't create by-key-getter due to only 1 sub-value
    'few' => [
        'one' => 'value',
    ],
    //won't create by-key-getter due to mixed values' types
    'mixedValues' => [
        'one' => 'value',
        'two' => 2,
    ],
    //won't create by-key-getter due to mixed keys' types
    'mixedKeys' => [
        'one' => 'value',
        2 => 'another value',
    ],
    //won't create by-key-getter to name conflicts
    //(because "getConflict()" and "getConflictBy()" conflicts with the next "conflict" and "conflictBy" fields)
    'conflicts' => [
        'one' => 'conflict',
        'two' => 'another conflict',
    ],
    'conflict' => 'third conflic',
    'conflictBy' => 'fourth conflic',
];
```

```terminal
php app.php create:config my -r
```

Результат:

```php app/src/Application/Config/AppConfig.php
declare(strict_types=1);

namespace App\Application\Config;

use Spiral\Core\InjectableConfig;

final class AppConfig extends InjectableConfig
{
    public const CONFIG = 'app';

    /**
     * Default values for the config.
     * Will be merged with application config in runtime.
     */
    protected array $config = [
        'params' => [],
        'parameter' => [],
        'values' => [],
        'value' => '',
        'few' => [],
        'mixedValues' => [],
        'mixedKeys' => [],
        'conflicts' => [],
        'conflict' => '',
        'conflictBy' => '',
    ];

    public function getParams(): array
    {
        return $this->config['params'];
    }

    public function getParameter(): array
    {
        return $this->config['parameter'];
    }

    public function getValues(): array
    {
        return $this->config['values'];
    }

    public function getValue(): string
    {
        return $this->config['value'];
    }

    public function getFew(): array
    {
        return $this->config['few'];
    }

    public function getMixedValues(): array
    {
        return $this->config['mixedValues'];
    }

    public function getMixedKeys(): array
    {
        return $this->config['mixedKeys'];
    }

    public function getConflicts(): array
    {
        return $this->config['conflicts'];
    }

    public function getConflict(): string
    {
        return $this->config['conflict'];
    }

    public function getConflictBy(): string
    {
        return $this->config['conflictBy'];
    }

    public function getParam(string $param): string
    {
        return $this->config['params'][$param];
    }

    public function getParameterBy(string $parameter): string
    {
        return $this->config['parameter'][$parameter];
    }

    public function getValueBy(int $value): string
    {
        return $this->config['values'][$value];
    }
}
```

### HTTP-контроллер

Команда создаёт класс контроллера. Контроллеры обрабатывают HTTP-запросы и ответы определённых конечных точек приложения.
Сгенерированный класс можно дополнить необходимыми действиями.

> **Смотрите также**
> Подробнее читайте в разделе [HTTP — Начало работы](../http/configuration.md).

```terminal
php app.php create:controller <name>
```

Будет создан класс `<Name>Controller`. Доступные параметры:

* `action (a)` — можно передать несколько раз для добавления действий;
* `prototype (p)` — добавляет `PrototypeTrait`.

#### Конфигурация

В примере используется следующая конфигурация объявления:

```php
Spiral\Scaffolder\Declaration\ControllerDeclaration::TYPE => [
    'namespace' => 'Endpoint\Web',
],
```

#### Пример без действий

```terminal
php app.php create:controller User
```

Результат:

```php app/src/Endpoint/Web/UserController.php
declare(strict_types=1);

namespace App\Endpoint\Web;

use Psr\Http\Message\ResponseInterface;
use Spiral\Router\Annotation\Route;

class UserController
{
}

```

#### Пример с параметром `prototype`

```terminal
php app.php create:controller User -p
```

Результат:

```php app/src/Endpoint/Web/UserController.php
declare(strict_types=1);

namespace App\Endpoint\Web;

use Psr\Http\Message\ResponseInterface;
use Spiral\Prototype\Traits\PrototypeTrait;
use Spiral\Router\Annotation\Route;

class UserController
{
    use PrototypeTrait;
}
```

#### Пример со списком действий

```bash
php app.php create:controller User \
      -a index \
      -a show \
      -a create \
      -a update \
      -a delete
```

Результат:

```php app/src/Endpoint/Web/UserController.php
declare(strict_types=1);

namespace App\Endpoint\Web;

use Psr\Http\Message\ResponseInterface;
use Spiral\Router\Annotation\Route;

class UserController
{
    /**
     * Please, don't forget to configure the Route attribute or remove it and register the route manually.
     */
    #[Route(route: 'path', name: 'name')]
    public function index(): ResponseInterface
    {
    }

    /**
     * Please, don't forget to configure the Route attribute or remove it and register the route manually.
     */
    #[Route(route: 'path', name: 'name')]
    public function show(): ResponseInterface
    {
    }

    /**
     * Please, don't forget to configure the Route attribute or remove it and register the route manually.
     */
    #[Route(route: 'path', name: 'name')]
    public function create(): ResponseInterface
    {
    }

    /**
     * Please, don't forget to configure the Route attribute or remove it and register the route manually.
     */
    #[Route(route: 'path', name: 'name')]
    public function update(): ResponseInterface
    {
    }

    /**
     * Please, don't forget to configure the Route attribute or remove it and register the route manually.
     */
    #[Route(route: 'path', name: 'name')]
    public function delete(): ResponseInterface
    {
    }
}
```

### Фильтр запроса

Команда создаёт класс фильтра запроса. Фильтры сопоставляют и проверяют данные HTTP-запроса до передачи контроллеру.
Сгенерированный класс можно дополнить необходимыми правилами.

> **Смотрите также**
> Подробнее читайте в разделе [Фильтры — Начало работы](../filters/configuration.md).

```terminal
php app.php create:filter <name>
```

Будет создан класс `<Name>Filter`.

#### Конфигурация

В примере используется следующая конфигурация объявления:

```php
Spiral\Scaffolder\Declaration\FilterDeclaration::TYPE => [
    'namespace' => 'Endpoint\Web\Filter',
],
```

#### Пример

```terminal
php app.php create:filter CreateUser
```

> **Предупреждение**
> Убедитесь, что в приложении включён загрузчик `Spiral\Validation\Bootloader\ValidationBootloader`.

Результат:

```php app/src/Endpoint/Web/Filter/CreateUserFilter.php
declare(strict_types=1);

namespace App\Endpoint\Web\Filter;

use Spiral\Filters\Model\Filter;

final class CreateUserFilter extends Filter
{
}
```

#### Создание фильтра со свойствами

Параметр `property (p)` определяет свойства класса фильтра. Каждое свойство задаётся в формате
`<name>:<source>:<type>`, где:

- `<name>` — имя свойства;
- `<source>` — источник входных данных: `post`, `get`, `header`, `cookie`, `server` и другие;
- `<type>` — тип свойства: `string`, `int`, `bool` или `array`.

```terminal
php app.php create:filter CreateUser -p username:post -p tags:post:array -p ip:ip -p token:header -p status:query:int
```

Результат:

```php app/src/Endpoint/Web/Filter/CreateUserFilter.php
declare(strict_types=1);

namespace App\Endpoint\Web\Filter;

use Spiral\Filters\Attribute\Input\Header;
use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Attribute\Input\Query;
use Spiral\Filters\Attribute\Input\RemoteAddress;
use Spiral\Filters\Model\Filter;

final class CreateUserFilter extends Filter
{
    #[Post(key: 'username')]
    public string $username;

    #[Post(key: 'tags')]
    public array $tags;

    #[RemoteAddress(key: 'ip')]
    public string $ip;

    #[Header(key: 'token')]
    public string $token;

    #[Query(key: 'status')]
    public int $status;
}
```

> **Примечание**
> Подробнее о доступных атрибутах читайте [здесь](../filters/filter.md).

#### Создание фильтра с правилами валидации

Чтобы сгенерировать фильтр с правилами валидации, добавьте параметр `-s`:

```terminal
php app.php create:filter CreateUser -p ... -s
```

Результат:

```php app/src/Endpoint/Web/Filter/CreateUserFilter.php
declare(strict_types=1);

namespace App\Endpoint\Web\Filter;

use Spiral\Filters\Attribute\Input\Header;
use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Attribute\Input\Query;
use Spiral\Filters\Attribute\Input\RemoteAddress;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

final class CreateUserFilter extends Filter implements HasFilterDefinition
{
    // ...

    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition(validationRules: [
            // Put your validation rules here
        ]);
    }
}
```

> **Предупреждение**
> В приложении должна быть установлена библиотека валидации. Подробнее о доступных библиотеках читайте
> [здесь](../validation/factory.md).

### HTTP Middleware

Команда создаёт класс middleware. Middleware изменяет HTTP-запросы и ответы по мере их прохождения через стек
приложения. Сгенерированный класс можно дополнить нужной логикой.

> **Смотрите также**
> Подробнее читайте в разделе [HTTP — Middleware](../http/middleware.md).

```terminal
php app.php create:middleware <name>
```

Будет создан класс `<Name>`.

#### Конфигурация

В примере используется следующая конфигурация объявления:

```php
Spiral\Scaffolder\Declaration\MiddlewareDeclaration::TYPE => [
    'namespace' => 'Endpoint\Web\Middleware',
    'postfix' => 'Middleware',
],
```

#### Пример

```terminal
php app.php create:middleware Logger
```

Результат:

```php app/src/Endpoint/Web/Middleware/LoggerMiddleware.php
declare(strict_types=1);

namespace App\Endpoint\Web\Middleware;

class LoggerMiddleware implements \Psr\Http\Server\MiddlewareInterface
{
    public function process(
        \Psr\Http\Message\ServerRequestInterface $request,
        \Psr\Http\Server\RequestHandlerInterface $handler,
    ): \Psr\Http\Message\ResponseInterface
    {
        return $handler->handle($request);
    }
}
```

### Обработчик задания

Команда создаёт класс обработчика задания очереди. Сгенерированный класс можно дополнить логикой обработки фоновой
задачи.

> **Смотрите также**
> Подробнее читайте в разделе [Очереди — Начало работы](../queue/configuration.md).

```terminal
php app.php create:jobHandler <name>
```

Будет создан класс `<Name>Job`.

#### Конфигурация

В примере используется следующая конфигурация объявления:

```php
Spiral\Scaffolder\Declaration\JobHandlerDeclaration::TYPE => [
    'namespace' => 'Endpoint\Job',
],
```

#### Пример

```terminal
php app.php create:jobHandler UserRegisteredNotification
```

Результат:

```php app/src/Endpoint/Job/UserRegisteredNotificationJob.php
declare(strict_types=1);

namespace App\Endpoint\Job;

use Spiral\Queue\JobHandler;

final class UserRegisteredNotificationJob extends JobHandler
{
    public function invoke(string $id, array $payload, array $headers): void
    {
    }
}
```
