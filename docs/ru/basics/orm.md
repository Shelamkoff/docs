# Основы — База данных и ORM

Для работы с ORM и базами данных Spiral предоставляет компонент
[spiral/cycle-bridge](https://github.com/spiral/cycle-bridge).

## Установка

Компонент автоматически включён в `spiral/app`. В существующий проект его можно установить через Composer:

```terminal
composer require spiral/cycle-bridge
```

После установки добавьте загрузчик `Spiral\Cycle\Bootloader\BridgeBootloader` в Kernel.

:::: tabs

::: tab С помощью метода

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Cycle\Bootloader\BridgeBootloader::class,
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
    \Spiral\Cycle\Bootloader\BridgeBootloader::class,
    // ...
];
```

Подробнее о загрузчиках читайте в разделе [Фреймворк — Загрузчики](../framework/bootloaders.md).
:::

::::

Для более точного управления можно не использовать `BridgeBootloader`, а подключить только необходимые загрузчики.

Пример конфигурации Kernel:

:::: tabs

::: tab С помощью метода

```php app/src/Application/Kernel.php
use Spiral\Cycle\Bootloader as CycleBridge;

public function defineBootloaders(): array
{
    return [
        // ...
    
        // Database
        CycleBridge\DatabaseBootloader::class,
        CycleBridge\MigrationsBootloader::class,
    
        // Close the database connection after every request automatically (Optional)
        // CycleBridge\DisconnectsBootloader::class,
    
        // ORM
        CycleBridge\SchemaBootloader::class,
        CycleBridge\CycleOrmBootloader::class,
        CycleBridge\AnnotatedBootloader::class,
        CycleBridge\CommandBootloader::class,
    
        // Validation (Optional)
        // CycleBridge\ValidationBootloader::class,
    
        // DataGrid (Optional)
        // CycleBridge\DataGridBootloader::class,
    
        // Database Token Storage (Optional)
        CycleBridge\AuthTokensBootloader::class,
    
        // Migrations and Cycle Scaffolders (Optional)
        CycleBridge\ScaffolderBootloader::class,
        
        // Prototyping (Optional)
        CycleBridge\PrototypeBootloader::class,
    ];
}
```

Подробнее о загрузчиках читайте в разделе [Фреймворк — Загрузчики](../framework/bootloaders.md).
:::

::: tab С помощью константы

```php app/src/Application/Kernel.php
use Spiral\Cycle\Bootloader as CycleBridge;

protected const LOAD = [
    // ...

    // Database
    CycleBridge\DatabaseBootloader::class,
    CycleBridge\MigrationsBootloader::class,

    // Close the database connection after every request automatically (Optional)
    // CycleBridge\DisconnectsBootloader::class,

    // ORM
    CycleBridge\SchemaBootloader::class,
    CycleBridge\CycleOrmBootloader::class,
    CycleBridge\AnnotatedBootloader::class,
    CycleBridge\CommandBootloader::class,

    // Validation (Optional)
    // CycleBridge\ValidationBootloader::class,

    // DataGrid (Optional)
    // CycleBridge\DataGridBootloader::class,

    // Database Token Storage (Optional)
    CycleBridge\AuthTokensBootloader::class,

    // Migrations and Cycle Scaffolders (Optional)
    CycleBridge\ScaffolderBootloader::class,
    
    // Prototyping (Optional)
    CycleBridge\PrototypeBootloader::class,
];
```

Подробнее о загрузчиках читайте в разделе [Фреймворк — Загрузчики](../framework/bootloaders.md).
:::

::::

#### Загрузчик Disconnects

Этот необязательный загрузчик автоматически закрывает соединение с базой данных после каждого запроса в долгоживущем
приложении. Подключайте его в зависимости от требований проекта.

## Конфигурация

### База данных

Настройки сервисов базы данных находятся в `app/config/database.php`. Здесь объявляются подключения и выбирается
подключение по умолчанию. Большинство параметров можно получать из переменных окружения приложения.

Пример конфигурации подключения:

```php app/config/database.php
use Cycle\Database\Config;

return [
    'logger' => [
        'default' => env('DB_LOGGER_DEFAULT'),
        'drivers' => [
            // 'runtime' => 'stdout'
        ],
    ],

    'default' => env('DB_DEFAULT', 'default'),

    /**
     * The Spiral/Database module provides support to manage multiple databases
     * in one application, use read/write connections and logically separate
     * multiple databases within one connection using prefixes.
     *
     * To register a new database simply add a new one into
     * "databases" section below.
     */
    'databases' => [
        'default' => [
            'driver' => 'runtime',
        ],
    ],

    /**
     * Each database instance must have an associated connection object.
     * Connections used to provide low-level functionality and wrap different
     * database drivers. To register a new connection you have to specify
     * the driver class and its connection options.
     */
    'drivers' => [
        'runtime' => new Config\MySQLDriverConfig(
            connection: new Config\MySQL\TcpConnectionConfig(
                database: 'homestead',
                host: '127.0.0.1',
                port: 3307,
                user: 'root',
                password: 'secret',
            ),
            queryCache: true
        ),
        // ...
    ],
];
```

> **Предупреждение**
> Использование SQLite с несколькими воркерами имеет существенные ограничения. Учитывайте их, чтобы обеспечить
> корректную работу и производительность приложения. Подробнее читайте в разделе
> [Ограничения SQLite](./#sqlite-limitations).

> **Смотрите также**
> Подробнее о конфигурации базы данных читайте в официальной документации
> [Database — Installation and Configuration](https://cycle-orm.dev/docs/database-configuration).

### ORM

Настройки ORM находятся в `app/config/cycle.php`.

```php app/config/cycle.php
use Cycle\ORM\SchemaInterface;

return [
    'schema' => [
        /**
         * true (Default) - Schema will be stored in a cache after compilation.
         * It won't be changed after entity modification. Use `php app.php cycle` to update schema.
         *
         * false - Schema won't be stored in a cache after compilation.
         * It will be automatically changed after entity modification. (Development mode)
         */
        'cache' => false,

        /**
         * The CycleORM provides the ability to manage default settings for
         * every schema with not defined segments
         */
        'defaults' => [
            SchemaInterface::MAPPER => \Cycle\ORM\Mapper\Mapper::class,
            SchemaInterface::REPOSITORY => \Cycle\ORM\Select\Repository::class,
            SchemaInterface::SCOPE => null,
            SchemaInterface::TYPECAST_HANDLER => [
                \Cycle\ORM\Parser\Typecast::class
            ],
        ],

        'collections' => [
            'default' => 'array',
            'factories' => [
                'array' => new \Cycle\ORM\Collection\ArrayCollectionFactory(),
                // 'doctrine' => new \Cycle\ORM\Collection\DoctrineCollectionFactory(),
                // 'illuminate' => new \Cycle\ORM\Collection\IlluminateCollectionFactory(),
            ],
        ],

        /**
         * Schema generators (Optional)
         * null (default) - Will be used schema generators defined in bootloaders
         */
        'generators' => null,

        // 'generators' => [
        //        \Cycle\Schema\Generator\ResetTables::class,
        //        \Cycle\Annotated\Embeddings::class,
        //        \Cycle\Annotated\Entities::class,
        //        \Cycle\Annotated\TableInheritance::class,
        //        \Cycle\Annotated\MergeColumns::class,
        //        \Cycle\Schema\Generator\GenerateRelations::class,
        //        \Cycle\Schema\Generator\GenerateModifiers::class,
        //        \Cycle\Schema\Generator\ValidateEntities::class,
        //        \Cycle\Schema\Generator\RenderTables::class,
        //        \Cycle\Schema\Generator\RenderRelations::class,
        //        \Cycle\Schema\Generator\RenderModifiers::class,
        //        \Cycle\Annotated\MergeIndexes::class,
        //        \Cycle\Schema\Generator\GenerateTypecast::class,
        // ],
    ],

    /**
     * Prepare all internal ORM services (mappers, repositories, typecasters...)
     */
    'warmup' => false,
];
```

## Cycle ORM

Cycle ORM — мощный и гибкий объектно-реляционный маппер для PHP, позволяющий работать с базой данных через объекты.
Он предоставляет гибкую конфигурацию, развитый построитель запросов и динамическое сопоставление схем.

Поддерживаются популярные реляционные СУБД: MySQL, MariaDB, PostgreSQL, SQL Server и SQLite.

> **Примечание**
> Полная документация доступна на официальном сайте [Cycle ORM](https://cycle-orm.dev/docs).

### Экземпляр ORM

Экземпляр ORM можно получить из контейнера через интерфейс `Cycle\ORM\ORMInterface`.

### Репозитории

Предположим, что в приложении есть сущность `User`:

```php
use Cycle\Annotated\Annotation as Cycle;

#[Cycle\Entity(repository: UserRepository::class)]
class User
{
    // ...
}
```

и репозиторий `UserRepository`:

```php 
class UserRepository extends \Cycle\ORM\Select\Repository
{
    public function findByEmail(string $email): ?User
    {
        return $this->findOne(['email' => $email]);
    }
}
```

Репозиторий можно получить непосредственно из ORM, передав имя сущности или роли.

```php
use Cycle\ORM\ORMInterface;
use Cycle\ORM\RepositoryInterface;

class UserService
{   
    private readonly RepositoryInterface $repository;

    public function __construct(
        Cycle\ORM\ORMInterface $orm
    ) {
        $this->repository = $orm->getRepository(User::class);
    }
    
    public function getProfile(string $email): User
    {
        $user = $this->repository->findOne(['email' => $email]);
        // ...
    }
}
```

Репозиторий также можно запросить из контейнера. Фреймворк использует
[инжекторы IoC](../container/injectors.md) для внедрения репозиториев, реализующих `Cycle\ORM\RepositoryInterface`.

```php
class UserService
{
    public function __construct(
        private readonly UserRepository $repository
    ) {
    }
    
    public function getProfile(string $email): User
    {
        $user = $this->repository->findByEmail($email);
        // ...
    }
}
```

При запросе репозитория из контейнера Spiral автоматически получает его из ORM и связывает с правильной сущностью.

### Транзакции

Для сохранения изменений сущностей сервисам и контроллерам требуется `Cycle\ORM\EntityManagerInterface`.

Фреймворк создаёт транзакцию по требованию при разрешении из контейнера. Поскольку после операции `run` транзакция
очищается, интерфейс можно безопасно внедрять через конструктор.

> **Смотрите также**
> Подробнее о транзакциях читайте в
> [документации Cycle ORM](https://cycle-orm.dev/docs/advanced-entity-manager).

Пример сервиса с `EntityManagerInterface`:

```php
use Cycle\ORM\EntityManagerInterface;

class UserService
{
    public function __construct(
        private readonly EntityManagerInterface $entityManager
    ) {
    }
    
    public function create(string $name, string $email): User
    {
        $user = new User($name, $email);
        
        $this->entityManager->persist($user);
        $this->entityManager->run();
        
        return $user;
    }
}
```

> **Примечание**
> При использовании транзакций уровня сервиса методы `persist/delete` и `run` должны вызываться в области одного метода.

### Валидация сущностей

Cycle Bridge предоставляет загрузчик `CycleBridge\ValidationBootloader`, регистрирующий дополнительные проверки для
пакета [spiral/validator](../validation/spiral.md). Он добавляет два правила валидации.

#### exists

Проверяет существование сущности с указанной ролью и первичным ключом.

По умолчанию правило ищет сущность по первичному ключу.

```php
use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Attribute\Setter;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;

final class StoreUser extends Filter implements HasFilterDefinition
{
    #[Post]
    #[Setter(filter: 'intval')]
    public int $id;
    
    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'username' => [
                [
                    'entity::exists', 
                    \App\Entity\User::class // Entity role
                ] 
            ]       
        ]);
    }
}
```

Также можно указать поле, значение которого используется для проверки существования сущности.

```php
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;

final class UpdateUser extends Filter implements HasFilterDefinition
{
    #[Post]
    public string $username;
    
    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'username' => [
                [
                    'entity::exists', 
                    \App\Entity\User::class, // Entity role
                    'username', // Field name
                ], 
            ],       
        ]);
    }
}
```

#### unique

Проверяет уникальность сущности указанной роли.

```php
use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Attribute\Setter;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;

final class StoreUser extends Filter implements HasFilterDefinition
{
    #[Post]
    public string $username;
    
    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'username' => [
                [
                    'entity::unique', 
                    \App\Entity\User::class, // Entity role
                    'username', // Field name
                ] 
            ]       
        ]);
    }
}
```

### Поведение сущностей

Чтобы использовать пакет [cycle/entity-behavior](https://cycle-orm.dev/docs/entity-behaviors-install/), сначала
установите его:

```bash
composer require cycle/entity-behavior
```

Затем свяжите `Cycle\ORM\Transaction\CommandGeneratorInterface` с
`\Cycle\ORM\Entity\Behavior\EventDrivenCommandGenerator` в контейнере приложения:

```php app/src/Application/Bootloader/EntityBehaviorBootloader.php
namespace App\Application\Bootloader;

use Cycle\ORM\Transaction\CommandGeneratorInterface;
use Cycle\ORM\Entity\Behavior\EventDrivenCommandGenerator;
use Spiral\Boot\Bootloader\Bootloader;

final class EntityBehaviorBootloader extends Bootloader
{
    protected const BINDINGS = [
        CommandGeneratorInterface::class => \Cycle\ORM\Entity\Behavior\EventDrivenCommandGenerator::class,
    ];
}
```

Зарегистрируйте `App\Application\Bootloader\EntityBehaviorBootloader` в ядре приложения.

:::: tabs

::: tab С помощью метода

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \App\Application\Bootloader\EntityBehaviorBootloader::class,
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
    \App\Application\Bootloader\EntityBehaviorBootloader::class,
    // ...
];
```

Подробнее о загрузчиках читайте в разделе [Фреймворк — Загрузчики](../framework/bootloaders.md).
:::

::::

После этого в приложении можно использовать поведения сущностей.

### Перехватчики

#### Разрешение сущностей Cycle

Интеграция Cycle ORM предоставляет перехватчик `Spiral\Cycle\Interceptor\CycleInterceptor`, автоматически разрешающий
сущности в методах контроллера по первичному ключу.

> **Примечание**
> Подробнее о перехватчиках читайте в разделе [HTTP — Перехватчики](../http/interceptors.md).

Активация перехватчика:

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Cycle\Interceptor\CycleInterceptor;
use Spiral\Bootloader\DomainBootloader;
use Spiral\Core\CoreInterface;

class AppBootloader extends DomainBootloader
{
    protected const SINGLETONS = [
        CoreInterface::class => [self::class, 'domainCore']
    ];

    protected const INTERCEPTORS = [
        CycleInterceptor::class,
        // ...
    ];
}
```

После этого сущность Cycle можно внедрить в метод контроллера:

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use App\Entity\User;
use Spiral\Router\Annotation\Route;

final class HomeController
{
    #[Route('/users/<user>')]
    public function index(User $user)
    {
        dump($user);
    }
}
```

> **Примечание**
> Если сущность не найдена, будет выброшено исключение 404.

### Долгоживущие приложения

Cycle ORM упрощает использование библиотеки в демонизированных приложениях, например PHP-воркерах RoadRunner или
Swoole. ORM предоставляет механизмы предотвращения утечек памяти, которые также подходят для пакетной обработки. Это
повышает стабильность и эффективность долгоживущих процессов.

Пакет автоматически очищает heap после каждого запроса. Для ручной очистки используйте:

```php app/src/Domain/User/Service/UserService.php
use Cycle\ORM\ORMInterface;

class UserService
{
    public function __construct(
        private readonly ORMInterface $orm
    ) {
    }
    
    public function create(string $name, string $email): User
    {
        // Create a new user
        
        $this->orm->getHeap()->clean();
    }
}
```

### Консольные команды

Интеграция Cycle ORM предоставляет несколько команд управления. Получить справку можно так:

```bash
php app.php help cycle...
```

> **Примечание**
> Чтобы включить вспомогательные команды, зарегистрируйте `Spiral\Cycle\Bootloader\CommandBootloader` после остальных
> загрузчиков Cycle.

#### База данных

| Команда            | Описание                                                                                                                   |
|--------------------|----------------------------------------------------------------------------------------------------------------------------|
| `db:list [db]`     | Выводит базы данных, их таблицы и количество записей.<br/>`db` — имя базы данных.                                          |
| `db:table <table>` | Описывает схему таблицы.<br/>`table` — обязательное имя таблицы.<br/>`--database` — исходная база данных.                  |

#### ORM и схема

| Команда         | Описание                                                                                       |
|-----------------|------------------------------------------------------------------------------------------------|
| `cycle`         | Обновляет или инициализирует схему Cycle из базы данных и аннотированных классов.              |
| `cycle:migrate` | Создаёт миграции схемы ORM.<br/>`--run` автоматически запускает созданную миграцию.             |
| `cycle:render`  | Отображает доступные схемы Cycle ORM.<br/>`--no-color` отключает цветной вывод.                 |

> **Примечание**
> Любую команду Cycle можно выполнить с флагом `-vv`, чтобы увидеть список изменённых таблиц.

<hr>

## База данных

### Доступ к базе данных

Получить доступ к базам данных в контроллерах и сервисах можно несколькими способами.

#### Через поставщика баз данных

```php app/src/Domain/User/Service/UserService.php
use Cycle\Database\DatabaseProviderInterface;

final class UserService 
{
    public function __construct(
        private readonly DatabaseProviderInterface $dbal
    ) {}
    
    public function store(): void
    {
        // Default database
        dump($this->dbal->database());
    
        // Using alias default which points to primary database
        dump($this->dbal->database('default'));
    
        // Secondary
        dump($this->dbal->database('slave'));
    }
}
```

#### Через внедрение в метод или конструктор

Компонент DBAL полностью поддерживает [инжекторы IoC](../container/injectors.md), использующие имена и псевдонимы баз
данных:

```php
use Cycle\Database\DatabaseInterface;

public function store(
    DatabaseInterface $database, 
    DatabaseInterface $primary,
    DatabaseInterface $slave
): void {
    // Database is an alias for "primary"
    dump($database === $primary);

    dump($primary);
    dump($slave);
}
```

#### Через прототип

Доступ к `Cycle\Database\DatabaseProviderInterface` и базе данных по умолчанию можно получить через `PrototypeTrait`:

```php app/src/Domain/User/Service/UserService.php
final class UserService 
{
    use PrototypeTrait;
    
    public function store(): void
    {
        dump($this->dbal);
        dump($this->db); // default db
    }
}
```

### Выполнение запросов

Для выполнения запроса используйте метод `query`:

```php
dump(
    $db->query('SELECT * FROM users WHERE id > ?', [
        1
    ])->fetchAll()
);
```

Для выполнения `UPDATE` или `DELETE` используйте метод `execute`:

```php
dump(
    $db->execute('DELETE FROM users WHERE id > ?', [
        1,
    ]) // number of affected rows 
);
```

> **Примечание**
> О построителях запросов читайте [здесь](https://cycle-orm.dev/docs/database-query-builders).

### Журналирование

Spiral позволяет записывать запросы к базе данных через компонент `spiral/logger`, использующий Monolog по умолчанию.

> **Смотрите также**
> Подробнее о логгере читайте в разделе [Основы — Журналирование](../basics/logging.md).

Драйверы журналирования баз данных настраиваются в секции `logger` файла `app/config/database.php`.

```php app/config/database.php
return [
    'logger' => [
        'default' => null,
        'drivers' => [],
    ],

    // ...
];
```

Если драйвер логгера не указан, используется канал с именем текущего драйвера базы данных. Например, при выполнении
запроса через SQLite канал Monolog автоматически получает имя `Cycle\Database\Driver\SQLite\SQLiteDriver`.

Чтобы записывать все запросы этой базы данных, настройте обработчик Monolog для соответствующего канала:

```php app/config/monolog.php
return [
    'handlers' => [
        // ...

        \Cycle\Database\Driver\SQLite\SQLiteDriver::class => [
            [
                'class' => 'log.rotate',
                'options' => [
                    'filename' => directory('runtime') . 'logs/db.log',
                    'level' => Logger::DEBUG,
                ],
            ],
        ],
    ],
    
    // ...
];
```

Секция `drivers` указывает, какой канал журнала должен использовать конкретный драйвер базы данных.

Рассмотрим конфигурацию:

```php app/config/database.php
return [
    'logger' => [
        'drivers' => [
            'runtime' => 'console'
        ],
    ],
    
    'databases' => [
        'default' => [
            'driver' => 'runtime',
        ],
    ],
    
    'drivers' => [
        'runtime' => new Config\SQLiteDriverConfig(...),
        // ...
    ],
];
```

Здесь драйвер `runtime` сопоставлен каналу `console`.

Конфигурация Monolog:

```php app/config/monolog.php
return [
    'handlers' => [
        //...
        'console' => [
            \Monolog\Handler\ErrorLogHandler::class,
        ],
    ],
];
```

При использовании драйвера `runtime` его записи будут отправляться в канал `console`.

Конкретные классы драйверов также можно сопоставить отдельным каналам. Например, журналы SQLite и MySQL можно хранить
раздельно, чтобы проще анализировать проблемы каждой базы данных.

```php app/config/database.php
return [
    'logger' => [
        'drivers' => [
            \Cycle\Database\Driver\MySQL\MySQLDriver::class => 'db_logs',
            \Cycle\Database\Driver\SQLite\SQLiteDriver::class => 'console'
        ],
    ],
];
```

В этом случае `SQLiteDriver` отправляет записи в канал `console`.

Ключ `default` задаёт канал по умолчанию для всех запросов драйверов, не перечисленных в секции `drivers`.

```php app/config/database.php
return [
    'logger' => [
        'default' => 'console',
    ],
];
```

Канал по умолчанию обеспечивает запись запросов даже тогда, когда для конкретного драйвера не задано отдельное
сопоставление.

### Консольные команды

Стандартные Web- и gRPC-сборки включают команды просмотра схемы базы данных.

Активируйте `Spiral\Cycle\Bootloader\CommandBootloader`:

:::: tabs

::: tab С помощью метода

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Cycle\Bootloader\CommandBootloader::class,
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
    \Spiral\Cycle\Bootloader\CommandBootloader::class,
    // ...
];
```

Подробнее о загрузчиках читайте в разделе [Фреймворк — Загрузчики](../framework/bootloaders.md).
:::

::::

#### Просмотр доступных драйверов и таблиц

```terminal
php app.php db:list
```

Результат:

```output
+------------+------------+---------+---------+-----------+---------+----------------+
| [32mName (ID):[39m | [32mDatabase:[39m  | [32mDriver:[39m | [32mPrefix:[39m | [32mStatus:[39m   | [32mTables:[39m | [32mCount Records:[39m |
+------------+------------+---------+---------+-----------+---------+----------------+
| [32mdefault[39m    | runtime.db | SQLite  | ---     | connected | users   | [33m0[39m              |
|            |            |         |         |           | posts   | [33m0[39m              |
+------------+------------+---------+---------+-----------+---------+----------------+
```

#### Просмотр схемы таблицы

```terminal
php app.php db:table posts
```

Результат:

```output
Columns of default.posts:
+---------+----------------+----------------+-----------+----------------+
| Column: | Database Type: | Abstract Type: | PHP Type: | Default Value: |
+---------+----------------+----------------+-----------+----------------+
| id      | int            | primary        | int       | ---            |
| title   | string (255)   | text           | string    | ---            |
| user_id | int            | integer        | int       | ---            |
+---------+----------------+----------------+-----------+----------------+

Indexes of default.posts:
+-----------------------------------+-------+----------+
| Name:                             | Type: | Columns: |
+-----------------------------------+-------+----------+
| posts_index_user_id_5e32b9642a0ff | INDEX | user_id  |
+-----------------------------------+-------+----------+

Foreign Keys of default.posts:
+------------------+---------+----------------+-----------------+------------+------------+
| Name:            | Column: | Foreign Table: | Foreign Column: | On Delete: | On Update: |
+------------------+---------+----------------+-----------------+------------+------------+
| posts_user_id_fk | user_id | users          | id              | CASCADE    | CASCADE    |
+------------------+---------+----------------+-----------------+------------+------------+
```

<hr>

## Миграции

Пакет `cycle/migrations` создаёт и управляет миграциями при изменении сущностей приложения. Он сравнивает текущую схему
базы данных с изменениями сущностей и формирует необходимые файлы миграций.

### Создание миграций

После изменения или создания сущностей выполните:

```terminal
php app.php cycle:migrate
```

Новые файлы появятся в каталоге `app/migrations`.

> **Предупреждение**
> Перед созданием новой миграции примените существующие командой `php app.php migrate`, чтобы текущая схема базы данных
> была актуальной.

### Применение миграций

```terminal
php app.php migrate
```

Команда применяет последние созданные миграции и изменяет схему базы данных в соответствии с их содержимым.

**Cycle ORM предоставляет несколько команд управления миграциями.**

#### Повторное выполнение

```terminal
php app.php migrate:replay
```

Команда откатывает миграции, а затем выполняет их заново. Это удобно для быстрой проверки изменений.

**Параметры:**

- `--all` — повторно выполнить все миграции, а не только последнюю.

#### Откат

```terminal
php app.php migrate:rollback
```

Команда отменяет изменения миграций. По умолчанию откатывается последняя миграция.

**Параметры:**

- `--all` — откатить все миграции.

#### Состояние

```terminal
php app.php migrate:status
```

Команда показывает все миграции и их состояние: выполнена миграция или ещё ожидает запуска.

**Пример вывода:**

```output
+-----------------------------------------------------------+---------------------+---------------------+
| Migration                                                 | Created at          | Executed at         |
+-----------------------------------------------------------+---------------------+---------------------+
| 0_default_create_auth_tokens                              | 2023-09-25 16:45:13 | 2023-10-04 10:46:11 |
| 0_default_create_user_role_create_user_roles_create_users | 2023-09-26 21:22:16 | 2023-10-04 10:46:11 |
| 0_default_change_user_roles_add_read_only                 | 2023-09-27 13:53:19 | 2023-10-04 10:46:11 |
+-----------------------------------------------------------+---------------------+---------------------+
```

#### Инициализация

```terminal
php app.php migrate:init
```

Компоненту миграций необходима таблица, в которой отмечаются выполненные миграции. Эта команда создаёт такую таблицу.

> **Примечание**
> Команда выполняется автоматически при первом запуске `php app.php migrate`.

### Конфигурация

Для изменения поведения миграций создайте файл `app/config/migration.php`. В нём можно настроить расположение файлов,
имя таблицы миграций и другие параметры.

**Доступные настройки:**

- **Directory** — каталог файлов миграций;
- **Table** — таблица состояния миграций;
- **Strategy** — стратегия создания файлов;
- **Name Generator** — генератор имён файлов;
- **Safe** — пропуск запросов подтверждения при запуске миграций в производственном окружении.

**Пример:**

```php app/config/migration.php
use Cycle\Schema\Generator\Migrations\Strategy\SingleFileStrategy;
use Cycle\Schema\Generator\Migrations\NameBasedOnChangesGenerator;

return [
    /**
     * Directory to store migration files
     */
    'directory' => directory('app').'migrations/',

    /**
     * Table name to store information about migrations status (per database)
     */
    'table' => 'migrations',
    
    /**
     * Migration file generator strategy
     */
    'strategy' => SingleFileStrategy::class,
    
    /**
     * Migration file name generator
     */
    'nameGenerator' => NameBasedOnChangesGenerator::class,

    /**
     * When set to true no confirmation will be requested on migration run.
     */
    'safe' => env('APP_ENV') === 'production',
];
```

### Стратегии файлов миграций

Начиная с версии **2.6.0** пакета `spiral/cycle-bridge`, можно выбирать способ создания файлов миграций.

#### 1. Один файл

Стратегия объединяет все изменения в один файл при каждом запуске команды.

```php
<?php  
  
declare(strict_types=1);  
  
namespace Migration;  
  
use Cycle\Migrations\Migration;  
  
class OrmDefaultA42b7e366d78543ca8c5a4b60d305083 extends Migration  
{  
    protected const DATABASE = 'default';  
  
    public function up(): void  
    {  
        $this->table('user_roles')  
	        ->addColumn('created_at', 'datetime', ['nullable' => false, 'default' => 'CURRENT_TIMESTAMP'])  
	        // ...
	        ->setPrimaryKeys(['uuid'])  
	        ->create();  
        
        $this->table('users')  
	        ->addColumn('created_at', 'datetime', ['nullable' => false, 'default' => 'CURRENT_TIMESTAMP'])  
	        //...
	        ->setPrimaryKeys(['uuid'])  
	        ->create();
    }  
  
    public function down(): void  
    {  
        $this->table('users')->drop();  
        $this->table('user_roles')->drop();
    }
}
```

#### 2. Несколько файлов

Изменения разделяются по таблицам: для каждой изменённой таблицы создаётся отдельный файл.

```php
<?php  
  
declare(strict_types=1);  
  
namespace Migration;  
  
use Cycle\Migrations\Migration;  
  
class OrmDefaultA42b7e366d78543ca8c5a4b60d305083 extends Migration  
{  
    protected const DATABASE = 'default';  
  
    public function up(): void  
    {  
        $this->table('user_roles')  
	        ->addColumn('created_at', 'datetime', ['nullable' => false, 'default' => 'CURRENT_TIMESTAMP'])  
	        // ...
	        ->setPrimaryKeys(['uuid'])  
	        ->create(); 
    }  
  
    public function down(): void  
    {  
        $this->table('user_roles')->drop();  
    }
}

```

```php
<?php  
  
declare(strict_types=1);  
  
namespace Migration;  
  
use Cycle\Migrations\Migration;  
  
class OrmDefaultA42b7e366d78543ca8c5a4b60d305043 extends Migration  
{  
    protected const DATABASE = 'default';  
  
    public function up(): void  
    {  
        $this->table('users')  
	        ->addColumn('created_at', 'datetime', ['nullable' => false, 'default' => 'CURRENT_TIMESTAMP'])  
	        //...
	        ->setPrimaryKeys(['uuid'])  
	        ->create();
    }  
  
    public function down(): void  
    {  
        $this->table('users')->drop();
    }
}
```

#### 3. Пользовательская стратегия

Собственную стратегию можно создать, реализовав
`Cycle\Schema\Generator\Migrations\Strategy\GeneratorStrategyInterface`.

### Генерация имён файлов миграций

Начиная с версии **2.6.0**, можно настроить стратегию именования файлов миграций.

По умолчанию Spiral использует `Cycle\Schema\Generator\Migrations\NameBasedOnChangesGenerator`, который учитывает все
изменения файла и создаёт уникальное имя. Пользовательский генератор должен реализовывать
`Cycle\Schema\Generator\Migrations\NameGeneratorInterface`.

---

## Ограничения SQLite

При использовании нескольких воркеров важно учитывать конкурентный доступ к базе данных. SQLite — надёжное и лёгкое
решение, но оно имеет ограничения при одновременной работе нескольких процессов.

### Блокировки файловой базы данных

База SQLite хранится в одном файле. Когда несколько воркеров одновременно обращаются к нему, возникают ограничения:
SQLite использует блокировки уровня файла и допускает только одного писателя в каждый момент времени. Одновременные
операции записи конкурируют за блокировку, что снижает производительность и может создавать проблемы с доступом к
данным.

### Невозможность совместного использования базы в памяти

SQLite не предоставляет встроенного способа совместного использования базы данных в памяти между несколькими
процессами или воркерами. Каждый воркер получает отдельную копию, поэтому изменения одного процесса не видны другим.
Это приводит к несогласованности и неверным результатам.

В приложениях Spiral с несколькими воркерами рекомендуется рассмотреть клиент-серверные СУБД, например MySQL или
PostgreSQL. Они рассчитаны на конкурентные соединения и лучше масштабируются в многоворкерном окружении.
