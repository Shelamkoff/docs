# Начало работы — Структура каталогов

Spiral не навязывает приложению определённую структуру каталогов, поэтому вы можете организовать файлы и каталоги
удобным для проекта способом. При этом фреймворк предлагает рекомендуемую структуру, которую можно использовать как
отправную точку и при необходимости легко изменить.

## Каталоги

Структура каталогов по умолчанию управляется методом `mapDirectories` класса Kernel. Все каталоги приложения
вычисляются относительно `root` по следующей схеме:

| Каталог   | Значение               |
|-----------|------------------------|
| root      | **задаётся пользователем** |
| app       | **root**/app           |
| config    | **app**/config         |
| resources | **app**/resources      |
| runtime   | **root**/runtime       |
| cache     | **root**/runtime/cache |
| public    | **root**/public        |
| vendor    | **root**/vendor        |

Некоторые компоненты объявляют собственные каталоги:

| Компонент         | Каталог    | Значение           |
|-------------------|------------|--------------------|
| spiral/views      | views      | **app**/views      |
| spiral/translator | locale     | **app**/locale     |
| spiral/migrations | migrations | **app**/migrations |

## Инициализация каталогов

Значение каталога `root` и любого другого каталога можно задать в файле `app.php`.

```php
$app = \App\Application\Kernel::create(
    directories: ['root' => __DIR__]
)->run();
```

Например, чтобы перенести каталог `runtime` во временный каталог операционной системы:

```php
$app = \App\Application\Kernel::create(
    directories: [
        'root' => __DIR__, 
        'runtime' => \sys_get_temp_dir()
    ]
)->run();
```

Получить пути к каталогам приложения можно через интерфейс `Spiral\Boot\DirectoriesInterface`. Он предоставляет доступ
к каталогам, объявленным методом `mapDirectories`.

Пример получения пути к каталогу `runtime`:

```php
use Spiral\Boot\DirectoriesInterface;

final class UploadService {
    public function __construct(
        private readonly DirectoriesInterface $dirs
    ) {}
    
    public function store(UploadedFile $file) {
        $filePath = $this->dirs->get('runtime') . 'uploads/' . $file->getFilename();
        // ...
    }
}
```

В глобальной области IoC — конфигурационных файлах, контроллерах и сервисном коде — также можно использовать функцию
`directory`.

```php app/config/cache.php
return [
    'storages' => [
        'file' => [
            'path' => directory('runtime') . 'cache',
        ],   
    ],
];
```

## Пространства имён

По умолчанию каркасы приложений используют корневое пространство имён `App`, указывающее на каталог `app/src`. Базовое
пространство имён можно изменить в `composer.json`:

```json composer.json
{
  "autoload": {
    "psr-4": {
      "App\\": "app/src/"
    }
  }
}
```

## Структура приложения

Ниже приведён распространённый вариант структуры PHP-приложения, который можно использовать как отправную точку.
Он помогает логично организовать код, упрощает сопровождение и дальнейшее масштабирование. Под конкретные требования
проекта структуру можно изменять, но для большинства приложений она служит надёжной основой.

```
- Endpoint
    - Web
        - ...
        - Filter
            - ...
        - Middleware
            - ...
        - Interceptor
            - ...
        - DataGrid
            - ...
        - routes.php
    - Console
        - Interceptor
            - ...
        - ...
    - RPC
        - Interceptor
            - ...
        - ...
    - Temporal
        - Workflow
            - ...
        - Activity
            - ...
    - Centrifugo
        - Interceptor
        - ...
- Application
    - Bootloader
        - ...
    - Exception
        - SomeException.php
        - Renderer
            - ViewRenderer.php
    - Kernel.php
- Domain
    - User
        - Entity
            - User.php
        - Service
            - StoreUserService.php
        - Repository
            - UserRepositoryInterface.php
        - Exception
            - UserNotFoundException.php
- Infrastructure
    - Persistence
        - CycleUserRepository.php
    - CycleORM
        - Typecaster
            - UuidTypecast.php
    - Interceptor
        - LogInterceptor.php
```

#### Краткое описание каталогов и файлов

- **Endpoint** — точки входа приложения: HTTP-обработчики в подкаталоге Web, интерфейсы командной строки в Console и
  gRPC-сервисы в RPC.

- **Application** — ядро приложения: класс Kernel, который запускает приложение, классы Bootloader, регистрирующие
  сервисы в контейнере, и каталог Exception с логикой обработки исключений.

- **Domain** — доменная логика, разделённая по поддоменам. Например, Entity модели пользователя, Service создания
  пользователей, Repository получения пользователей из базы данных и Exception для ошибок, связанных с пользователями.

- **Infrastructure** — инфраструктурный код приложения: Persistence для работы с базой данных, CycleORM для кода,
  связанного с ORM, и Interceptor для глобальных перехватчиков.

<hr>

## Что дальше?

Для более глубокого знакомства с основами прочитайте следующие разделы:

* [Ядро и окружение](../framework/kernel.md);
* [Файлы и каталоги](../basics/files.md).
