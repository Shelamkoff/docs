# Начало работы — Первый HTTP-контроллер

Если вы используете Spiral в PHP-проекте и готовы создать первый контроллер, выполните следующие основные шаги.

## Создание контроллера

Сначала необходимо создать контроллер. В Spiral контроллер — это класс, определяющий поведение приложения для
определённого набора маршрутов. Он принимает входящие запросы, совместимые с
[PSR-7](https://www.php-fig.org/psr/psr-7/), обрабатывает данные и возвращает ответ клиенту.

Чтобы быстро создать первый контроллер, используйте команду генератора:

```terminal
php app.php create:controller CurrentDate
```

> **Примечание**
> Подробнее о генерации кода читайте в разделе
> [Основы — Генерация кода](../basics/scaffolding.md#http-controller).

После выполнения команды успешное создание будет подтверждено следующим выводом:

```output
Declaration of '[32mCurrentDateController[39m' has been successfully written into '[33mapp/src/Endpoint/Web/CurrentDateController.php[39m'.
```

Теперь добавим логику в созданный контроллер.

Пример контроллера, возвращающего текущие дату и время:

```php app/src/Endpoint/Web/CurrentDateController.php
namespace App\Endpoint\Web;

final class CurrentDateController 
{
    public function show(): string
    {
        return \date('Y-m-d H:i:s');
    }
}
```

Следующий шаг — связать контроллер с маршрутом.

## Создание маршрута

:::: tabs

::: tab С помощью атрибутов

Spiral упрощает объявление маршрутов с помощью атрибутов PHP. Достаточно добавить атрибут `#[Route]` к методу
контроллера:

```php app/src/Endpoint/Web/CurrentDateController.php
use Spiral\Router\Annotation\Route;

// ...

#[Route(route: '/date', name: 'current-date', methods: 'GET')]
public function show(): string
{
    return \date('Y-m-d H:i:s');
}
```

:::

::: tab С помощью RoutingConfigurator

Для централизованного и упорядоченного объявления маршрутов Spiral предоставляет метод `defineRoutes` класса
`App\Application\Bootloader\RoutesBootloader`.

Пример маршрута, который будет обрабатываться нашим контроллером:

```php app/src/Application/Bootloader/RoutesBootloader.php
final class RoutesBootloader extends BaseRoutesBootloader
{
    // ...

    protected function defineRoutes(RoutingConfigurator $routes): void
    {
        $routes->add(name: 'current-date', pattern: '/date')
            ->action(controller: CurrentDateController::class, action: 'show');
    }
}
```

:::

::::

Чтобы просмотреть список маршрутов, выполните:

```terminal
php app.php route:list
```

В выведенном списке должен появиться маршрут `current-date`:

```output
+--------------+--------+----------+------------------------------------------------+--------+
|[32m Name:        [39m|[32m Verbs: [39m|[32m Pattern: [39m|[32m Target:                                        [39m|[32m Group: [39m|
+--------------+--------+----------+------------------------------------------------+--------+
| current-date | [32mGET[39m    | /date    | App\Endpoint\Web\CurrentDateController->show | web    |
+--------------+--------+----------+------------------------------------------------+--------+
```

## Проверка контроллера

После настройки контроллера запустите сервер RoadRunner:

```terminal
./rr serve
```

Теперь контроллер можно проверить в браузере. Откройте адрес: http://127.0.0.1/date

<br><br>

**Готово! Вы успешно создали первый контроллер в Spiral.**

<hr>

## Что дальше?

Для более глубокого знакомства с основами прочитайте следующие разделы:

* [Маршрутизация](../http/routing.md);
* [Маршрутизация с помощью атрибутов](../http/annotated-routes.md);
* [Middleware](../http/middleware.md);
* [Страницы ошибок](../http/errors.md);
* [Пользовательский HTTP-обработчик](../cookbook/psr-15.md);
* [Генерация кода](../basics/scaffolding.md).
