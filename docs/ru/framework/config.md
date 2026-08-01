# Фреймворк — Объекты конфигурации

Spiral использует объекты конфигурации, чтобы разделить этапы начальной загрузки и выполнения приложения и предоставить
удобный источник настроек.

Главное преимущество объектов конфигурации состоит в том, что во время начальной загрузки значения можно свободно
изменять, а во время выполнения они остаются неизменяемыми. Это предотвращает неожиданные изменения настроек и делает
поведение приложения более предсказуемым.

![Этапы управления приложением](https://user-images.githubusercontent.com/67324318/186413037-e60f89fd-9313-44c5-b4f8-eb2585a77230.png)

**Преимущества объектов конфигурации:**

- значения легко изменять во время начальной загрузки, не меняя код приложения;
- после запуска приложения значения фиксируются, что предотвращает неожиданные изменения во время выполнения;
- сведения о конфигурации централизованы, поэтому их проще находить, изменять и сопровождать;
- настройки отделены от кода приложения, что улучшает разделение ответственности и упрощает поддержку кодовой базы.

## Поставщик конфигурации

Spiral предоставляет интерфейс `Spiral\Config\ConfiguratorInterface`, через который можно получать значения из
конфигурационных файлов и проверять их наличие.

Создадим файл `app/config/github.php`:

```php app/config/github.php
<?php

return [
    'access_token' => 'xxx-xxxx',
    // ...
];
```

Имя может быть любым, но рекомендуется использовать название сервиса, который будет читать эту конфигурацию.

> **Примечание**
> Можно использовать формат `json` или расширить `ConfiguratorInterface`, добавив собственные средства чтения
> конфигурации.

### Использование

Получить значения конфигурации в сервисе можно через `ConfiguratorInterface`:

```php
use Spiral\Config\ConfiguratorInterface;

final class GithubClient
{
    private readonly string $accessToken;
    
    public function __construct(ConfiguratorInterface $configurator)
    {
        if (!$configurator->exists('github')) {
            throw new \RuntimeException('Github configuration is missing');
        }
        
        $config = $configurator->get('github');
        $this->accessToken = $config['access_token'] ?? throw new \RuntimeException('Missing access token');
    }

    // ...
}
```

## Объект конфигурации

Читать конфигурацию в виде массивов не всегда удобно. Фреймворк предоставляет объектно-ориентированную абстракцию для
доступа к значениям. Класс можно создать вручную или сгенерировать с помощью `spiral/scaffolder`:

```terminal
php app.php create:config github -r
``` 

> **Примечание**
> Параметр `-r` восстанавливает структуру конфигурации из файла `app/config/github.php`.

Сгенерированный класс будет находиться в `app/src/Config/GithubConfig.php`:

```php app/src/Config/GithubConfig.php
namespace App\Config;

use Spiral\Core\InjectableConfig;

class GithubConfig extends InjectableConfig
{
    public const CONFIG = 'github';

    protected array $config = [
        'access_token' => '',
        // ...
    ];

    public function getAccessToken(): string
    {
        return $this->config['access_token'];
    }
}
``` 

Базовый класс `Spiral\Core\InjectableConfig` позволяет сразу запрашивать объект в коде без дополнительной настройки
IoC-контейнера. Константа `CONFIG` содержит имя конфигурационного файла.

> **Примечание**
> Сгенерированный класс можно изменять и добавлять в него дополнительные методы.

```php
use App\Config\GithubConfig;

final class GithubClient
{
    private readonly string $accessToken;
    
    public function __construct(GithubConfig $config)
    {
        $this->accessToken = $config->getAccessToken();
    }

    // ...
}
```

> **Примечание**
> Объект конфигурации предоставляет API только для чтения. Изменение значений во время выполнения запрещено, чтобы
> избежать нежелательных побочных эффектов в долгоживущих приложениях.

Каждый компонент Spiral предоставляет собственный объект конфигурации, который можно использовать в приложении.

Классы конфигурации упрощают доступ к значениям и управление ими, одновременно предоставляя преимущества
объектно-ориентированной абстракции и более упорядоченной структуры кода.

## Конфигурация по умолчанию в загрузчике

Значения по умолчанию можно объявить в пользовательском загрузчике, чтобы не создавать лишние файлы. В качестве значений
по умолчанию можно использовать переменные окружения.

Это удобно, когда стандартная конфигурация подходит большинству приложений и отдельный конфигурационный файл не нужен.

```php
namespace App\Application\Bootloader;

use App\Config\GithubConfig;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Boot\EnvironmentInterface;
use Spiral\Config\ConfiguratorInterface;

final class GithubBootloader extends Bootloader
{
    public function init(ConfiguratorInterface $configurator, EnvironmentInterface $env): void
    {
        $configurator->setDefaults(GithubConfig::CONFIG, [
            'access_token' => $env->get('GITHUB_ACCESS_TOKEN')
            'authentication_type' => $env->get('GITHUB_AUTHENTICATION_TYPE', 'token')
        ]);
    }
}
```

Так можно задавать общие для всех окружений значения по умолчанию или резервные значения на случай, если соответствующая
переменная окружения отсутствует.

Чтобы использовать только конфигурацию по умолчанию, удалите файл `app/config/github.php`.

> **Примечание**
> Значения по умолчанию при необходимости легко переопределяются, но продолжают использоваться как резервные, если
> конфигурационного файла нет.

## Автоматическая конфигурация

Некоторые компоненты предоставляют API автоматической настройки, позволяющий изменять параметры во время начальной
загрузки приложения. Обычно такой API доступен через загрузчик компонента.

> **Примечание**
> Например, метод `HttpBootloader`->`addMiddleware`.

Собственный API автоматической настройки можно добавить в загрузчик с помощью `ConfiguratorInterface`->`modify`.
Объявим загрузчик singleton-объектом, чтобы немного ускорить обработку.

```php
namespace App\Application\Bootloader;

use App\Config\GithubConfig;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Config\ConfiguratorInterface;
use Spiral\Config\Patch\Set;
use Spiral\Core\Container\SingletonInterface;

class GithubBootloader extends Bootloader implements SingletonInterface
{
    public function __construct(
        private readonly ConfiguratorInterface $configurator
    ) {
    }

    public function init(): void
    {
        $configurator->setDefaults(GithubConfig::CONFIG, [
            'access_token' => $env->get('GITHUB_ACCESS_TOKEN')
            'authentication_type' => $env->get('GITHUB_AUTHENTICATION_TYPE', 'token')
        ]);
    }

    public function setAccessToken(string $token): void
    {
        $this->configurator->modify(
          GithubConfig::CONFIG, 
          new Set('access_token', $token)
        );
    }
}
```

Теперь конфигурацию можно изменять через строгий API из другого загрузчика:

```php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;

class SomeBootloader extends Bootloader
{
    public function init(GithubBootloader $github): void
    {
        $github->setAccessToken('xxx-xxxx');
    }
}
```

## Жизненный цикл конфигурации

Фреймворк не позволяет изменять значения после того, как объект конфигурации был запрошен одним из компонентов. В этот
момент конфигурация считается зафиксированной.

Для демонстрации внедрим `GithubConfig` в `SomeBootloader`:

```php
namespace App\Application\Bootloader;

use App\Config\GithubConfig;
use Spiral\Boot\Bootloader\Bootloader;

final class SomeBootloader extends Bootloader
{
    public function boot(GithubBootloader $github, GithubConfig $config): void
    {
        // forbidden
        $github->setAccessToken(800);
    }
}
```

Будет выброшено исключение `Spiral\Config\Exception\ConfigDeliveredException`: *Unable to patch config `github`,
config object has already been delivered.*

> **Предупреждение**
> Значения по умолчанию и изменения конфигурации рекомендуется задавать в методе `init`, а объект конфигурации
> запрашивать только в `boot`. Метод `init` выполняется раньше `boot`, поэтому к моменту вызова `boot` конфигурация уже
> будет находиться в правильном состоянии.
