# Основы — Файлы и каталоги

Фреймворк предоставляет простой компонент `spiral/files` для работы с файловой системой.

## Реестр каталогов

Большинство компонентов Spiral используют реестр каталогов вместо жёстко заданных путей. Реестр представлен
интерфейсом `Spiral\Boot\DirectoriesInterface`.

> **Смотрите также**
> Подробнее о структуре каталогов приложения читайте в разделе
> [Начало работы — Структура каталогов](../start/structure.md).

Каталоги конкретного приложения можно настроить в его точке входа `app.php`:

```php app.php
$app = \App\Application\Kernel::create(
    directories: [
        'root' => __DIR__,
        'uploadDir' => __DIR__ . '/upload'
    ]
)->run();
```

Или с помощью загрузчика:

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Boot\DirectoriesInterface;

final class AppBootloader extends Bootloader
{
    public function boot(DirectoriesInterface $dirs): void
    {
        $dirs->set(
            'uploadDir',
            $dirs->get('root') . '/upload'
        );
    }
}
```

Получение путей каталогов:

```php
use Spiral\Boot\DirectoriesInterface;

final class UploadService {
    public function __construct(
        private readonly DirectoriesInterface $dirs
    ) {}
    
    public function store(UploadedFile $file) {
        $filePath = $this->dirs->get('uploadDir') . $file->getFilename();
        // ...
    }
}
```

> **Примечание**
> В конфигурационных файлах также можно использовать короткую функцию `directory`. За пределами фреймворка она не
> работает, поскольку зависит от глобальной области контейнера.

Чтобы имена каталогов всегда получались через специализированные методы, можно создать вспомогательный класс, например
`AppDirectories`.

<details>
  <summary>Показать пример</summary>

```php app/src/Application/AppDirectories.php
namespace App\Application;

use Spiral\Boot\DirectoriesInterface;

final class AppDirectories
{
    public function __construct(
        private readonly DirectoriesInterface $directories
    ) {
    }

    /**
     * Application root directory.
     * @return non-empty-string
     */
    public function getRoot(?string $path = null): string
    {
        return $this->buildPath('root', $path);
    }

    /**
     * Application directory.
     * @return non-empty-string
     */
    public function getApp(?string $path = null): string
    {
        return $this->buildPath('app', $path);
    }

    /**
     * Public directory.
     * @return non-empty-string
     */
    public function getPublic(?string $path = null): string
    {
        return $this->buildPath('public', $path);
    }

    /**
     * Runtime directory.
     * @return non-empty-string
     */
    public function getRuntime(?string $path = null): string
    {
        return $this->buildPath('runtime', $path);
    }

    /**
     * Runtime cache directory.
     * @return non-empty-string
     */
    public function getCache(?string $path = null): string
    {
        return $this->buildPath('cache', $path);
    }

    /**
     * Vendor libraries directory.
     * @return non-empty-string
     */
    public function getVendor(?string $path = null): string
    {
        return $this->buildPath('vendor', $path);
    }

    /**
     * Config directory.
     * @return non-empty-string
     */
    public function getConfig(?string $path = null): string
    {
        return $this->buildPath('config', $path);
    }

    /**
     * Resources directory.
     * @return non-empty-string
     */
    public function getResources(?string $path = null): string
    {
        return $this->buildPath('resources', $path);
    }

    private function buildPath(string $key, ?string $path = null): string
    {
        return \rtrim($this->directories->get($key), '/') . ($path ? '/' . \ltrim($path, '/') : '');
    }
}
```
</details>

## Файлы

Для работы с файловой системой используйте компонент `Spiral\Files\FilesInterface`:

```php
use Spiral\Files\FilesInterface;

final class FileService
{   
    public function __construct(
        private readonly FilesInterface $files,
        private readonly AppDirectories $dirs
    ) {}

    public function getRootFiles()
    {
        // get all files from root directory recursively
        dump(
            $files->getFiles($dirs->getRoot())
        );
    }
}
```

Экземпляр также доступен через прототипное свойство `files`:

```php
use Spiral\Prototype\Traits\PrototypeTrait;

final class FileService
{
    use PrototypeTrait;
   
    public function __construct(
        private readonly AppDirectories $dirs
    ) {}
    
    public function store()
    {
        dump($this->files->exists(__FILE__)); // true
    }
}
```

> **Смотрите также**
> Подробнее о прототипных свойствах читайте в разделе
> [Основы — Прототипирование](../basics/prototype.md).

### Создание каталога

Чтобы убедиться, что каталог существует, используйте метод `ensureDirectory`. Второй аргумент задаёт права доступа:

```php
public function store()
{
    $this->files->ensureDirectory(
        $dirs->get('customDir'),
        FilesInterface::READONLY // or FilesInterface::RUNTIME for editable dirs and files
    );
}
```

Проверка существования каталога:

```php
dump($files->isDirectory(__DIR__));
```

### Удаление каталога

Удаление каталога вместе с содержимым:

```php
$files->deleteDirectory('custom');
```

Удаление только содержимого каталога:

```php
$files->deleteDirectory('custom', true);
```

### Сведения о файле

Проверка существования файла:

```php
dump($files->exists(__FILE__)); // bool
```

Получение времени создания или изменения файла:

```php
dump($files->time(__FILE__)); // unix timestamp
```

Получение MD5 файла:

```php
dump($files->md5('filename'));
```

Получение расширения имени файла:

```php
dump($files->extension(__FILE__)); // without leading "."
```

Проверка, является ли путь файлом:

```php
dump($files->isFile(__DIR__));
```

Получение размера файла:

```php
dump($files->size(__DIR__));
```

### Права доступа

Получение прав файла или каталога:

```php
dump($files->getPermissions(__FILE__)); // int
```

Установка прав файла:

```php
$files->setPermissions(__FILE__, 0777)
```

Для выбора режима файла используйте константы:

| Константа                | Значение |
|--------------------------|----------|
| FilesInterface::READONLY | 644      |
| FilesInterface::RUNTIME  | 666      |

### Перемещение и копирование

Копирование файла из одного пути в другой:

```php
$files->copy('old-path', 'new-path');
```

Перемещение файла:

```php
$files->move('old-path', 'new-path');
```

### Временные файлы

Получение временного имени файла:

```php
dump($files->tempFilename());
```

Получение временного имени с определённым расширением:

```php
dump($files->tempFilename('php'));
```

Получение временного имени в указанном каталоге:

```php
dump($files->tempFilename('php', __DIR__));
```

## Операции чтения и записи

Компонент предоставляет методы для атомарной работы с содержимым файла без самостоятельного получения файлового
ресурса.

### Запись и создание файла

Запись содержимого в файл с эксклюзивной блокировкой:

```php
$files->write('filename', 'data');
```

Запись или создание файла с установкой нужного режима доступа:

```php
$files->write('filename', 'data', 0777);
```

Автоматическое создание каталога файла при необходимости:

```php
$files->write('filename', 'data', 0777, true);
```

> **Примечание**
> Если файл недоступен для записи, обработайте исключение `Spiral\Files\Exception\WriteErrorException`.

### Добавление содержимого

Добавление данных в конец файла:

```php
$files->append('filename', 'data');
```

Добавление данных с установкой режима файла:

```php
$files->append('filename', 'data', 0777);
```

Добавление или создание файла с автоматическим созданием целевого каталога:

```php
$files->append('filename', 'data', 0777, true);
```

### Touch

Обновление времени файла и его создание при отсутствии:

```php
$files->touch('filename');
```

Обновление времени файла с установкой режима доступа:

```php
$files->touch('filename', 0777);
```

### Чтение файла

Чтение содержимого файла:

```php
dump($files->read('filename'));
```

> **Примечание**
> Если файл не найден, обработайте исключение `Spiral\Files\Exception\FileNotFoundException`.
