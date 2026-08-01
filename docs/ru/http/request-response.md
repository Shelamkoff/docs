# HTTP — Запрос и ответ

Контроллерам и конечным точкам необходим доступ к активному PSR-7-запросу и возможность сформировать ответ. В этом
разделе рассматривается работа с запросами и ответами в MVC-приложении.

> **Примечание**
> Middleware и нативные обработчики PSR-15 могут получать PSR-7-объекты напрямую.

## Область запроса

Самый быстрый способ получить пользовательский запрос — внедрить `Psr\Http\Message\ServerRequestInterface` в метод.

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use Psr\Http\Message\ServerRequestInterface;
use Spiral\Core\Container\SingletonInterface;

class HomeController implements SingletonInterface
{
    public function index(ServerRequestInterface $request): void
    {
        dump($request->getHeaders());
    }
}
```

> **Предупреждение**
> Нельзя внедрять `Psr\Http\Message\ServerRequestInterface` через конструктор singleton-класса.

После получения запроса доступны все методы чтения, предусмотренные
[стандартом PSR-7](https://www.php-fig.org/psr/psr-7/).

## InputManager

В качестве альтернативы можно использовать менеджер контекста `Spiral\Http\Request\InputManager`. Его разрешено
хранить в singleton-сервисах и контроллерах: объект всегда указывает на текущий пользовательский запрос. Он предоставляет
набор удобных методов для чтения входящих данных.

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use Spiral\Core\Container\SingletonInterface;
use Spiral\Http\Request\InputManager;

class HomeController implements SingletonInterface
{
    private InputManager $input;

    public function __construct(InputManager $input)
    {
        $this->input = $input;
    }

    public function index(): void
    {
        dump($this->input->query->all());
    }
}
```

Получить доступ к `Spiral\Http\Request\InputManager` также можно через `Spiral\Prototype\Traits\PrototypeTrait`.

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use Spiral\Prototype\Traits\PrototypeTrait;

class HomeController
{
    use PrototypeTrait;

    public function index(): void
    {
        // $this->request is alias to $this->input
        dump($this->request->data->all());
    }
}
```

> **Предупреждение**
> Без необходимости не обращайтесь напрямую к `Psr\Http\Message\ServerRequestInterface` и
> `Spiral\Http\Request\InputManager`. Вместо этого рекомендуется использовать **фильтры запросов**.

Через `Spiral\Http\Request\InputManager` можно получить полный массив входных данных или отдельное поле по имени.
Для вложенных структур поддерживается точечная нотация. Каждый источник данных представлен объектом
`Spiral\Http\Request\InputBag` с общим набором методов. Рассмотрим работу с параметрами запроса:

```php
/** @var \Spiral\Http\Request\InputManager $input */

// Get instance of QueryBag associated with query data
dump($input->query);
 
// Get all query params as array
dump($input->query->all());
 
// Count of query params
dump($input->query->count());
 
// Check if parameter "name" is presented in query
dump($input->query->has('name'));
 
// Get value for parameter "name"
dump($input->query->get('name'));
 
// Both get and has methods support dot notation for nested structures
dump($input->query->has('name.subName'));
dump($input->query->get('name.subName'));
 
// Fetch only given query params (no dot notation allowed), only existed records will be returned
dump($input->query->fetch(["name", "nameB"]);

// Fetch only given query params (no dot notation allowed), non existed records will be filled with `null`
dump($input->query->fetch(['name', 'nameB'], true, null);

// In addition query get method has short alias in input manager
dump($input->query('name'));
```

> **Предупреждение**
> Если входной контейнер содержит ключ со значением `null`, например
> `new \Spiral\Http\Request\InputBag(['name' => null]);`, метод `$input->query->has('name')` вернёт `true`, а
> `isset($input->query['name])` — `false`.

### Заголовки запроса

Для доступа к заголовкам используются контейнер `headers` и метод `header` класса
`Spiral\Http\Request\InputManager`. Класс `Spiral\Http\Request\HeadersBag` имеет несколько особенностей:

- имя запрашиваемого заголовка нормализуется автоматически;
- метод `get` по умолчанию объединяет значения заголовка через запятую.

```php
/** @var \Spiral\Http\Request\InputManager $sinput */

// Get all headers as array
dump($input->headers->all());

// Will be normalized into "Accept"
dump($input->headers->get('accept')); 

// Return Accept header as array of values
dump($input->headers->get('accept', false));

dump($input->header('accept'));
```

### Cookie

```php
/** @var \Spiral\Http\Request\InputManager $input */

dump($input->cookies->all());

dump($input->cookie('name'));
```

### Серверные переменные

```php
/** @var \Spiral\Http\Request\InputManager $input */

dump($input->server->all());

dump($input->server('name'));
```

> **Примечание**
> `Spiral\Http\Request\ServerBag` автоматически нормализует имена запрашиваемых серверных переменных, поэтому их можно
> получать без написания имени только в верхнем регистре.

```php
/** @var \Spiral\Http\Request\InputManager $input */

dump($input->server('SERVER_PORT'));

dump($input->server('server-port'));
```

### Параметры POST/Data

```php
/** @var \Spiral\Http\Request\InputManager $input */

dump($input->data->all());

dump($input->data('name'));

// An alias
dump($input->post('name'));
```

### POST/Data с резервным чтением Query

Чтобы сначала получить значение из POST-данных, а при его отсутствии — из Query, используйте метод `input`.

```php
dump($input->input('name'));
```

### Атрибуты PSR-7-запроса

```php
dump($input->attributes->all());

dump($input->attribute('name'));
```

#### Загруженные файлы

Для получения списка загруженных файлов или отдельного файла используйте контейнер `files` и метод `file`. Каждый файл
представлен объектом `Psr\Http\Message\UploadedFileInterface`, являющимся частью PSR-7.

```php
/** @var \Spiral\Http\Request\InputManager $input */

dump($this->input->files->all());

dump($this->input->file('upload'));
```

> **Примечание**
> В соответствии с PSR файлы организованы в логическую иерархию, отличающуюся от стандартного формата PHP. Для доступа
> к вложенным файлам можно использовать точечную нотацию.

### Упрощённые методы

Помимо контейнеров данных, `Spiral\Http\Request\InputManager` предоставляет методы для чтения разных свойств активного
запроса.

```php
/** @var \Spiral\Http\Request\InputManager $input */

//Request Uri path, will always include leading /
dump($input->path());

//Active request Uri instance
dump($input->uri());

//GET, POST, PUT...
dump($input->method());

//Check if connection made over https
dump($input->isSecure());

//Check request headers to verify that request made over ajax
dump($input->isAjax());

//Check is request expects application/json as response (Accept: application/json)
dump($input->isJsonExpected());

//Receive client ip address (this method uses _SERVER value and may not be correct in some cases).
dump($input->remoteAddress());
```

Получить `Spiral\Http\Request\InputBag` без использования `__get` можно так:

```php
/** @var \Spiral\Http\Request\InputManager $input */

dump($input->bag('data')->all());
```

### Добавление собственного контейнера входных данных

Через `Spiral\Bootloader\Http\HttpBootloader` можно добавить собственный контейнер данных.

Например, необходимо получать загруженные файлы в виде объектов
`Symfony\Bridge\PsrHttpMessage\Factory\UploadedFile`. Создадим класс контейнера:

```php
namespace App\Http\Request;

use Spiral\Http\Request\InputBag;
use Symfony\Bridge\PsrHttpMessage\Factory\UploadedFile;

final class FilesBag extends InputBag
{
    public function __construct(array $data, string $prefix = '')
    {
        foreach ($data as $name => $file) {
            $data[$name] = new UploadedFile($file, fn(): string => $this->getTemporaryPath());
        }

        parent::__construct($data, $prefix);
    }

    protected function getTemporaryPath(): string
    {
        return \tempnam(\sys_get_temp_dir(), \uniqid('symfony', true));
    }
}
```

Затем добавьте созданный `FilesBag` методом `addInputBag` класса `HttpBootloader`:

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use App\Http\Request\FilesBag;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Bootloader\Http\HttpBootloader;

class AppBootloader extends Bootloader
{
    public function init(HttpBootloader $http): void
    {
        $http->addInputBag('symfonyFiles', [
            'class'  => FilesBag::class,
            'source' => 'getUploadedFiles',
            'alias' => 'symfony-file'
        ]);
    }
}
```

## InputInterface

У методов `Spiral\Http\Request\InputManager` отсутствует префикс `get`. Причина связана с внешним пакетом
`spiral/filters`, которому необходим поставщик данных `Spiral\Filters\InputInterface`:

```php
namespace Spiral\Filters;

// ...

interface InputInterface
{
    public function withPrefix(string $prefix, bool $add = true): InputInterface;

    public function getValue(string $source, string $name = null);
}
```

Методы `Spiral\Http\Request\InputManager` можно вызывать через короткую нотацию `Spiral\Filters\InputInterface`. Оба
подхода возвращают одинаковые данные.

```php app/src/Endpoint/Web/HomeController.php
use Spiral\Filters\InputInterface;
use Spiral\Http\Request\InputManager;

public function index(InputInterface $inputSource, InputManager $inputManager): void
{
    dump($inputManager->query('name'));
    dump($inputSource->getValue('query', 'name'));

    dump($inputManager->path());
    dump($inputSource->getValue('path'));
}
```

Этот механизм используется для сопоставления входящих данных с фильтром запроса.

> **Предупреждение**
> Для доступа к `Spiral\Filters\InputInterface` необходимо активировать
> `Spiral\Bootloader\Security\FiltersBootloader`.

## Формирование ответа

Контроллер может вернуть экземпляр `Psr\Http\Message\ResponseInterface`, который будет непосредственно отправлен
пользователю.

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use Nyholm\Psr7\Response;
use Psr\Http\Message\ResponseInterface;

class HomeController 
{
    public function index(): ResponseInterface
    {
        $response = new Response(200);
        $response->getBody()->write("hello world");

        return $response;
    }
}
```

Обработчик PSR-15, включённый по умолчанию, может автоматически сформировать ответ из возвращённой строки или содержимого
буфера вывода:

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

class HomeController
{
    public function index(): string
    {
        return "hello world";
    }
}
```

Этот код эквивалентен следующему:

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

class HomeController
{
    public function index(): void
    {
        echo "hello world";
    }
}
```

> **Примечание**
> Буфер вывода рекомендуется использовать только при разработке для отображения отладочных данных. В рабочем коде
> следует придерживаться строгих возвращаемых типов.

## JSON-ответы

Стандартный обработчик PSR-15 поддерживает массивы и объекты `JsonSerializable`, автоматически преобразуя их в JSON:

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

class HomeController
{
    public function index(): array
    {
        return [
            'status' => 200,
            'data' => ['some' => 'json']
        ];
    }
}
```

## Фабрика ответов

Для абстрагирования от ручного создания ответа используйте `Psr\Http\Message\ResponseFactoryInterface`:

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use Psr\Http\Message\ResponseFactoryInterface;
use Psr\Http\Message\ResponseInterface;

class HomeController
{
    public function index(ResponseFactoryInterface $responseFactory): ResponseInterface
    {
        $response = $responseFactory->createResponse(200);
        $response->getBody()->write("hello world");

        return $response;
    }
}
```

## ResponseWrapper

Для формирования более сложных ответов используйте оболочку над `ResponseFactoryInterface` —
`Spiral\Http\ResponseWrapper`. Она добавляет набор удобных методов:

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use Psr\Http\Message\ResponseInterface;
use Spiral\Http\ResponseWrapper;

class HomeController
{
    public function index(ResponseWrapper $response): ResponseInterface
    {
        return $response->attachment(
            __FILE__,
            'controller.php'
        )->withAddedHeader('Key', 'value');
    }
}
```

Оболочка также доступна через `PrototypeTrait` в свойстве `response`:

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use Psr\Http\Message\ResponseInterface;
use Spiral\Prototype\Traits\PrototypeTrait;

class HomeController
{
    Use PrototypeTrait;

    public function index(): ResponseInterface
    {
        // temporary redirect
        return $this->response->redirect('https://google.com', 307);
    }
}
```

Создание HTML-ответа:

```php app/Interface/Controllers/HomeController.php
public function index(): ResponseInterface
{
    return $this->response->html('hello world');
}
```

Создание ответа `application/json`:

```php app/Interface/Controllers/HomeController.php
public function index(): ResponseInterface
{
    return $this->response->json(
        ['something' => 123],
        200
    );
}
```

Отправка вложения:

```php app/Interface/Controllers/HomeController.php
public function index(): ResponseInterface
{
    return $this->response->attachment(__FILE__, 'name.php');
}
```

> **Примечание**
> Первым аргументом также можно передать `Psr\Http\Message\StreamInterface`, а третьим — указать MIME-тип.
