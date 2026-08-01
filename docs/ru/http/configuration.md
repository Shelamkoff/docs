# HTTP — Начало работы

Набор веб-приложения (`spiral/app`) поставляется с предварительно настроенным HTTP-компонентом. Чтобы включить его в
альтернативной сборке, потребуется установить несколько расширений.

## Установка

Установите расширение:

```terminal
composer require spiral/nyholm-bridge
```

Активируйте компонент, добавив загрузчики:

:::: tabs

::: tab С помощью метода

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        // Fast PSR-7 implementation
        \Spiral\Nyholm\Bootloader\NyholmBootloader::class,
    
        // HTTP core
        \Spiral\Bootloader\Http\HttpBootloader::class,
    
        // PSR-15 handler      
        \Spiral\Bootloader\Http\RouterBootloader::class,
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
    // Fast PSR-7 implementation
    \Spiral\Nyholm\Bootloader\NyholmBootloader::class,

    // HTTP core
    \Spiral\Bootloader\Http\HttpBootloader::class,

    // PSR-15 handler      
    \Spiral\Bootloader\Http\RouterBootloader::class,
    // ...
];
```

Подробнее о загрузчиках читайте в разделе [Фреймворк — Загрузчики](../framework/bootloaders.md).
:::

::::

> **Примечание**
> Пример использования собственного обработчика PSR-15 приведён [здесь](../cookbook/psr-15.md).

Убедитесь, что настроена [маршрутизация](../http/routing.md).

## Конфигурация

HTTP-расширение настраивается в файле `app/config/http.php`:

```php app/config/http.php
return [
    // default base path
    'basePath'   => '/',
    
    // default headers
    'headers'    => [
        'Content-Type' => 'text/html; charset=UTF-8'
    ],

    // application level middleware
    'middleware' => [
        // middleware class name
    ],
];
```

> **Примечание**
> Если конфигурационный файл отсутствует, будут использованы настройки по умолчанию.
