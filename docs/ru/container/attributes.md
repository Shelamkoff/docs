# Контейнер — Атрибуты

Spiral предоставляет набор атрибутов для дополнительного управления разрешением зависимостей.

## Singleton

Атрибут `#[Singleton]` полезен, когда во всём приложении должен существовать только один экземпляр класса. Например, это
подходит для менеджеров конфигурации, подключений к базе данных и логгеров.

#### Пример

Предположим, что в приложении есть менеджер конфигурации, который читает файл и предоставляет доступ к его параметрам.
Повторно загружать и разбирать файл при каждом обращении неэффективно. Атрибут `#[Singleton]` гарантирует, что после
первой загрузки объект останется в памяти на протяжении жизненного цикла приложения.

```php
use Spiral\Core\Attribute\Singleton;

#[Singleton]
final class ConfigurationManager
{
    private readonly array $config;

    public function __construct()
    {
        $this->config = parse_ini_file('config.ini');
    }

    public function get(string $key)
    {
        return $this->config[$key] ?? null;
    }
}
```

## Scope

Атрибут позволяет указать область, в которой зависимость разрешено создавать. При попытке разрешить её в другой области
будет выброшено исключение о несоответствии областей. Это помогает строго соблюдать границы контекста и не создавать
зависимости там, где они не должны быть доступны.

#### Пример

Предположим, что некоторые ресурсы или операции приложения доступны только аутентифицированным пользователям. В таком
случае необходимо:

1. убедиться, что пользователь аутентифицирован;
2. сделать его данные доступными приложению на время текущего запроса.

Следующий класс хранит сведения об аутентифицированном пользователе. Эти данные должны разрешаться только внутри области
`auth`.

```php
use Spiral\Core\Attribute\Scope;

#[Scope('auth')]
final readonly class AuthenticatedUser
{
    public function __construct(
        private int $id,
        private string $name, 
        private string $email,
    ) {
    }
}
```

Middleware проверяет аутентификацию пользователя. Если проверка успешна, оно создаёт область `auth` в IoC-контейнере и
связывает данные пользователя с классом `AuthenticatedUser`.

```php;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Message\ResponseInterface as Response;
use Psr\Http\Message\ServerRequestInterface as Request;
use Psr\Http\Server\RequestHandlerInterface as Handler;

final class AuthMiddleware implements MiddlewareInterface
{
    public function __construct(
        private readonly ContainerInterface $container
    ) {
    }

    public function process(Request $request, Handler $handler): Response
    {
        // This is a simplified authentication check.
        // In a real application, this might involve checking session data, JWT tokens, etc.
        if ($request->hasHeader('Authorization')) {
            // Fetch user data based on the authorization header. 
            // For simplicity, we're hardcoding user data here.
            $user = new AuthenticatedUser(1, 'John Doe', 'john.doe@example.com');

            // Set up the auth scope
            return $this->container->runScoped(
                closure: function (ContainerInterface $container) use ($next, $request) {
                    // Now, within this scope, you can get the authenticated user instance.
                    $authenticatedUser = $container->get(AuthenticatedUser::class);
                    
                    // Continue processing the request.
                    return $handler($request);
                },
                bindings: [AuthenticatedUser::class => $user],
                name: 'auth'
            );

        } else {
            // No authentication header found.
            // Return an unauthorized response or simply continue processing.
            return $handler($request);
        }
    }
}
```

Если контроллеру или сервису нужны сведения об аутентифицированном пользователе, он может запросить
`AuthenticatedUser` из контейнера. Благодаря атрибуту `#[Scope('auth')]` объект будет получен только внутри правильной
области.

```php
final class UserProfileController
{
    public function getProfile(AuthenticatedUser $user)
    {
        // Use the $user data to fetch and return the profile.
    }
}
```

> **Примечание**
> Подробнее об областях контейнера читайте в разделе
> [Фреймворк — Области IoC](../framework/scopes.md).

## Finalize

Атрибут позволяет указать метод финализации класса. Если зависимость была разрешена внутри области, указанный метод будет
вызван при уничтожении этой области. Он предназначен для очистки ресурсов и других завершающих действий перед
уничтожением объекта.

#### Пример

Предположим, что класс управляет подключением к базе данных. После завершения работы, особенно внутри отдельной области,
подключение необходимо закрыть и освободить связанные ресурсы.

```php
use Spiral\Core\Attribute\Finalize;

#[Finalize(method: 'closeConnection')]
final class DatabaseConnection
{
    private $connection;

    public function __construct()
    {
        // Initialize the database connection
    }

    public function query($sql)
    {
        // Execute the query on the database
    }

    public function closeConnection(): void
    {
        // Close the connection
    }
}
```

При использовании класса внутри области метод `closeConnection` будет вызван после её завершения, что гарантирует
освобождение ресурсов:

```php
$container->runScoped(
    closure: function (DatabaseConnection $db) {
        // Execute some database operations
        $users = $db->query('SELECT * FROM users');
        // ...  
    },
    bindings: [DatabaseConnection::class => new DatabaseConnection()],
);

// Once the scope is destroyed, the connection is automatically closed.
```

> **Предупреждение**
> Объект может остаться доступным после вызова финализатора. Следует избегать таких ситуаций.

```php
$root = new Container();
$obj = $root->get(Foo::class);
unset($root); // The Foo finalizer will be called

// Here we have a leaked finalized object. It is `$obj`.
```

## Совместное использование атрибутов

Атрибуты `#[Finalize]`, `#[Singleton]` и `#[Scope]` полностью совместимы. Их можно одновременно применять к одному
классу, точно управляя поведением и жизненным циклом зависимости.

#### Пример

Предположим, что в приложении есть сервис кеширования со следующими требованиями:

1. На протяжении жизненного цикла приложения должен существовать только один экземпляр сервиса.
2. Сервис должен быть доступен только в определённой части приложения, например при обработке HTTP-запроса.
3. При завершении приложения или области необходимо закончить отложенные операции кеша и закрыть открытые файловые или
   сетевые ресурсы.

```php
namespace App\Services;

use Psr\Log\LoggerInterface;
use Spiral\Core\Attribute\Finalize;
use Spiral\Core\Attribute\Scope;
use Spiral\Core\Attribute\Singleton;

#[Singleton]
#[Scope('http')]
#[Finalize(method: 'shutdown')]
final class CacheService
{
    private array $cache = [];
    public function __construct(
        private readonly LoggerInterface $logger,
    ) {
    }

    public function get(string $key): ?string
    {
        return $this->cache[$key] ?? null;
    }

    public function set(string $key, string $value): void
    {
        $this->cache[$key] = $value;
    }

    public function shutdown(): void
    {
        $this->logger->info("CacheService is finalizing.");
        
        // Flush the cache to a persistent storage, close any resources, etc.
        $this->cache = [];
    }
}
```
