# Основы — Кеширование

Кеширование — способ значительно повысить производительность приложения за счёт хранения часто используемых данных в
более быстром хранилище, например в памяти. Это сокращает необходимость повторно вычислять данные или получать их из
более медленного источника, например базы данных. Слой кеширования позволяет быстро возвращать сохранённые данные,
сокращая время ответа приложения и нагрузку на исходное хранилище.

Компонент `spiral/cache` предоставляет простой механизм хранения и получения данных. Он соответствует стандарту
[PSR-16](https://www.php-fig.org/psr/psr-16/), поэтому будет знаком разработчикам, использующим стандарты PHP.

## RoadRunner как основа кеширования

RoadRunner играет центральную роль в возможностях кеширования Spiral. Его основные особенности:

- **Написан на Go.** RoadRunner ориентирован на скорость и эффективность, а Go хорошо подходит для
  высокопроизводительных операций.

- **Рассчитан на высокую конкурентность.** Он выполняет операции параллельно и способен обрабатывать значительную
  конкурентную нагрузку.

- **Разные хранилища через плагин Key-Value.** Плагин
  [Key-Value](https://roadrunner.dev/docs/plugins-kv/) позволяет использовать Redis, Memcached или бессерверные варианты,
  например хранилище в памяти.

- **Не требует расширений PHP.** При работе через RoadRunner не нужны отдельные расширения PHP для Redis или Memcache:
  приложение взаимодействует с RoadRunner напрямую по RPC.

## Установка

Чтобы включить компонент, добавьте `Spiral\Cache\Bootloader\CacheBootloader` в список загрузчиков.

> **Предупреждение**
> Убедитесь, что установлен пакет
> [spiral/roadrunner-bridge](../start/server.md#roadrunner-bridge).

:::: tabs

::: tab С помощью метода

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Cache\Bootloader\CacheBootloader::class,
        \Spiral\RoadRunnerBridge\Bootloader\CacheBootloader::class,
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
    \Spiral\Cache\Bootloader\CacheBootloader::class,
    \Spiral\RoadRunnerBridge\Bootloader\CacheBootloader::class,
    // ...
];
```

Подробнее о загрузчиках читайте в разделе [Фреймворк — Загрузчики](../framework/bootloaders.md).
:::

::::

> **Примечание**
> Пакет `spiral/roadrunner-bridge` позволяет использовать в Spiral
> [плагин KV](https://roadrunner.dev/docs/plugins-kv/) RoadRunner. Пакет предоставляет RPC API для KV и загрузчик
> приложения.

## Конфигурация

Создайте конфигурационный файл `app/config/cache.php`. В нём задаются хранилища и хранилище по умолчанию.

Пример конфигурации `cache.php`:

```php app/config/cache.php
use Spiral\Cache\Storage\ArrayStorage;
use Spiral\Cache\Storage\FileStorage;

return [
    /**
     * -------------------------------------------------------------------------
     *  Default storage
     * -------------------------------------------------------------------------
     * 
     * The key of one of the registered cache storages to use by default.
     */
    'default' => env('CACHE_STORAGE', 'rr-redis'),
    
    /**
     * -------------------------------------------------------------------------
     *  Aliases
     * -------------------------------------------------------------------------
     * 
     * Aliases, if you want to use domain specific storages.
     */
    'aliases' => [
        'user-data' => 'rr-redis',
        'blog-data' => [
            'storage' => 'rr-redis',
            'prefix' => 'blog_'
        ],
    ],
    
    /**
     * -------------------------------------------------------------------------
     *  Storages
     * -------------------------------------------------------------------------
     * 
     * Here you may define all of the cache "storages" for your application as well as their types.
     */
    'storages' => [
        'rr-redis' => [
            'type' => 'roadrunner',
            'driver' => 'redis',
        ],

        'rr-local' => [
            'type' => 'roadrunner',
            'driver' => 'local',
        ],
        
        'local' => [
            'type' => 'array',
        ],
        
        'file' => [
            'type' => 'file',
            'path' => directory('runtime') . 'cache',
        ],
    ],
    
    /**
     * -------------------------------------------------------------------------
     *  Aliases for storage types
     * -------------------------------------------------------------------------
     */
    'typeAliases' => [
        'array' => ArrayStorage::class,
        'file' => FileStorage::class,
    ],
];
```

### Работа с псевдонимами

Псевдонимы — короткие имена для обращения к кеш-хранилищам. Они упрощают код и делают его понятнее.

Например, хранилище `rr-redis` можно использовать в приложении под именем `user-data`. Для этого предназначена секция
`aliases`.

Получить хранилище по имени псевдонима можно через `Spiral\Cache\CacheStorageProviderInterface`. Поставщик сопоставит
псевдоним с фактическим хранилищем.

### Префиксы ключей кеша

Префиксы особенно важны при использовании псевдонимов. Они сохраняют уникальность ключей и предотвращают случайные
пересечения. Если псевдониму назначен `prefix`, он автоматически добавляется ко всем ключам этого псевдонима.

Добавление префикса в `cache.php`:

```php app/config/cache.php
return [
    // ...

    'aliases' => [
        'user-data' => [
            'storage' => 'rr-redis',
            'prefix' => 'user_'
        ],
    ],
];
```

В этом примере все ключи псевдонима `user-data` получают префикс `user_`. Ключ `profile` будет сохранён как
`user_profile`.

### Конфигурация плагина RoadRunner KV

Конфигурационный файл плагина RoadRunner KV:

```yaml .rr.yaml
kv:
  local:
    driver: memory
    config:
      interval: 60
  redis:
    driver: redis
    config:
      addrs:
        - localhost:6379
...
```

> **Смотрите также**
> Подробнее о настройке плагина RoadRunner KV читайте в
> [документации RoadRunner](https://roadrunner.dev/docs/plugins-kv).

## Использование

Компонент кеширования Spiral позволяет работать с кешем через два интерфейса:
`Psr\SimpleCache\CacheInterface` и `Spiral\Cache\CacheStorageProviderInterface`.

> **Смотрите также**
> Можно использовать прототипные свойства `cache` и `cacheManager`. Подробнее о прототипах читайте в разделе
> [Основы — Прототипирование](../basics/prototype.md).

### Хранилище по умолчанию

Интерфейс `Psr\SimpleCache\CacheInterface` предоставляет доступ к хранилищу по умолчанию, заданному в конфигурации. Его
можно внедрить в класс как зависимость:

```php
namespace App\Service;

use Psr\SimpleCache\CacheInterface;

final class UserService
{
    public function __construct(
        private readonly CacheInterface $cache,
    ) {
    }

    public function find(int $id): User
    {
        // ...
    }
}
```

### Поставщик кеш-хранилищ

Через `Spiral\Cache\CacheStorageProviderInterface` можно получить конкретное хранилище по строковому ключу из
конфигурации. Это позволяет выделять отдельные кеши для разных доменов и сценариев.

```php
namespace App\Service;

use Spiral\Cache\CacheStorageProviderInterface;

class UserService
{
    private readonly CacheInterface $cache;
  
    public function __construct(CacheStorageProviderInterface $provider) 
    {
        $this->cache = $provider->storage('user-data');
    }

    public function find(int $id): User
    {
        //...
    }
}
```

В примере в `UserService` передаётся специализированное хранилище `user-data`. Его можно использовать во всём классе
для сохранения, получения и удаления кешированных пользовательских данных.

Такой подход упрощает управление кешем разных частей приложения и позволяет при необходимости менять используемые
хранилища.

### Получение элементов

Для получения элемента используйте метод `get`. Если элемента нет, метод вернёт `null`. Также можно указать значение по
умолчанию.

```php
$data = $this->cache->get('key');

$data = $this->cache->get('key', 'default');
```

Для одновременного получения нескольких элементов используйте `getMultiple` и передайте массив ключей.

```php
$data = $this->cache->getMultiple(['key', 'other']);
```

### Проверка существования элемента

Метод `has` возвращает `true`, если элемент существует, и `false`, если его нет.

```php
if ($this->cache->has('key')) {
    // ...
}
```

### Сохранение элементов

Для сохранения элемента используйте метод `set`.

```php
$this->cache->set(
    key: 'key', 
    value: ['some' => 'data'], 
    ttl: 3600
);
```

Срок хранения, или TTL, можно передать в секундах либо объектом `DateInterval` или `DateTimeInterface`.

```php
$this->cache->set(
    key: 'key', 
    value: ['some' => 'data'], 
    ttl: \Carbon\Carbon::now()->addHour()
);
```

Если TTL не указан, значение хранится бессрочно либо столько, сколько допускает используемый драйвер.

```php
$this->cache->set(
    key: 'key', 
    value: ['some' => 'data']
);
```

Несколько элементов можно сохранить одновременно методом `setMultiple`, передав массив пар ключ-значение.

```php
$this->cache->setMultiple(
    values: [
        'key' => ['some' => 'data'], 
        'other' => ['foo' => 'bar']
    ], 
    ttl: 3600
);
```

### Удаление элементов

Для удаления элемента используйте метод `delete`.

```php
$this->cache->delete('key');
```

Несколько элементов можно удалить методом `deleteMultiple`, передав массив ключей.

```php
$this->cache->deleteMultiple(['key', 'other']);
```

### Очистка кеша

Для удаления всех элементов используйте метод `clear`.

```php
$this->cache->clear();
```

## Пользовательское хранилище

Через конфигурацию компонент позволяет подключать
[пользовательские хранилища](https://packagist.org/providers/psr/simple-cache-implementation). Можно использовать любую
доступную реализацию PSR-16.

Чтобы подключить пользовательское хранилище, зарегистрируйте его в секции `storages` конфигурационного файла, укажите
тип ключом `type` и передайте необходимые параметры.

Класс хранилища также необходимо зарегистрировать в секции `typeAliases`, чтобы Spiral мог создать его экземпляр.

## События

| Событие                          | Описание                                                    |
|----------------------------------|-------------------------------------------------------------|
| `Spiral\Cache\Event\CacheHit`    | Вызывается после успешного получения данных из кеша         |
| `Spiral\Cache\Event\CacheMissed` | Вызывается, если запрошенные данные в кеше не найдены        |
| `Spiral\Cache\Event\KeyDeleted`  | Вызывается `после` удаления данных из кеша                  |
| `Spiral\Cache\Event\KeyWritten`  | Вызывается `после` сохранения данных в кеше                 |

> **Примечание**
> Подробнее о диспетчеризации событий читайте в разделе [События](../advanced/events.md).
