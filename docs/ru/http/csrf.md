# HTTP — Защита от CSRF

Spiral предоставляет встроенную защиту от CSRF (Cross-Site Request Forgery — подделки межсайтовых запросов). Она
помогает гарантировать, что действия на сайте действительно инициированы пользователем, а не вредоносной страницей.

Spiral хранит CSRF-токен в cookie и не зависит от серверных сессий. Такой подход проще и эффективнее: он не требует
дополнительного серверного хранилища и снижает накладные расходы.

## Описание уязвимости

Предположим, в приложении есть страница смены пароля, принимающая `POST`-запрос с новым значением. Если приложение не
проверяет подлинность запроса, злоумышленник может заставить браузер пользователя отправить этот запрос с вредоносной
страницы.

**Пример вредоносной страницы:**

```html Malicious page
<form action="https://your-application.com/user/password" method="POST">
    <input name="password" type="password" value="secret">
</form>

<script>
    document.forms[0].submit();
</script>
```

Без защиты от CSRF пароль будет изменён, когда пользователь откроет такую страницу.

Для предотвращения атаки каждый входящий запрос `POST`, `PUT`, `PATCH` или `DELETE` необходимо проверять на наличие
CSRF-токена, недоступного стороннему сайту. Токен создаётся сервером и обычно помещается в скрытое поле формы.

```html
<form action="https://your-application.com/user/password" method="POST">
    <input type="hidden" name="csrf-token" value="{csrfToken}"/>
    <input name="password" type="password">
    // ...
    <button type="submit">Change password</button>
</form>
```

При отправке формы сервер сравнивает токен из запроса со значением cookie пользователя. Если значения не совпадают,
запрос отклоняется.

## Конфигурация

Стандартный `spiral/app` уже содержит middleware защиты от CSRF.

Для подключения к альтернативной сборке добавьте `Spiral\Bootloader\Http\CsrfBootloader` в список загрузчиков.

:::: tabs

::: tab С помощью метода

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Bootloader\Http\CsrfBootloader::class,
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
    \Spiral\Bootloader\Http\CsrfBootloader::class,
    // ...
];
```

Подробнее о загрузчиках читайте в разделе [Фреймворк — Загрузчики](../framework/bootloaders.md).
:::

::::

После добавления загрузчика включите `Spiral\Csrf\Middleware\CsrfMiddleware`, создающее уникальный токен для каждого
пользователя.

Добавьте middleware в группу маршрутов, которую необходимо защитить:

```php app/src/Application/Bootloader/RoutesBootloader.php
namespace App\Application\Bootloader;

use Spiral\Cookies\Middleware\CookiesMiddleware;
use Spiral\Csrf\Middleware\CsrfMiddleware;

final class RoutesBootloader extends BaseRoutesBootloader
{
    protected function middlewareGroups(): array
    {
        return [
            'web' => [
                CookiesMiddleware::class,
                CsrfMiddleware::class,
                // ...
            ],
            // ...
        ];
    }

    // ...
}
```

> **Смотрите также**
> Подробнее о глобальных middleware читайте в разделе
> [HTTP — Middleware](middleware.md#global-middleware).

Параметры компонента настраиваются в `app/config/csrf.php`.

Конфигурация по умолчанию:

```php app/config/csrf.php
return [
    'cookie'   => 'csrf-token',
    'length'   => 16,
    'lifetime' => 86400,
    'secure'   => true,
    'sameSite' => null,
];
```

> **Предупреждение**
> При изменении параметра `cookie` новое имя необходимо добавить в белый список cookie. Подробнее читайте в разделе
> [HTTP — Cookie](cookies.md#configuration).

## Включение firewall

Компонент предоставляет два middleware для включения защиты маршрутов.

Чтобы защитить все запросы, кроме `GET`, `HEAD` и `OPTIONS`, используйте `Spiral\Csrf\Middleware\CsrfFirewall`:

```php app/src/Application/Bootloader/RoutesBootloader.php
use Spiral\Csrf\Middleware\CsrfFirewall;

'web' => [
    CookiesMiddleware::class,
    CsrfMiddleware::class,
    CsrfFirewall::class,
    // ...
],
```

> **Примечание**
> Для защиты запросов со всеми HTTP-методами используйте `Spiral\Csrf\Middleware\StrictCsrfFirewall`.

## Использование

После включения firewall все нужные формы должны содержать токен из PSR-7-атрибута `csrfToken`.

> **Примечание**
> Атрибут `csrfToken` создаётся middleware `Spiral\Csrf\Middleware\CsrfMiddleware` при каждом запросе.

Получить токен в контроллере или представлении можно методом `getAttribute`:

```php
public function index(ServerRequestInterface $request): void
{
    $csrfToken = $request->getAttribute('csrfToken');
}
```

Каждый запрос `POST`, `PUT` или `DELETE` должен передавать токен в POST-параметре `csrf-token` или заголовке
`X-CSRF-Token`. При отсутствии или неверном значении пользователь получит ответ `412 Bad CSRF Token`.

```php
use Psr\Http\Message\ServerRequestInterface;

// ...

public function changePasswordForm(ServerRequestInterface $request): string
{
    $form = <<<FORM
<form action="https://your-application.com/user/password" method="POST">
    <input type="hidden" name="csrf-token" value="{csrfToken}"/>
    <input name="password" type="password">
    // ...
    <button type="submit">Change password</button>
</form>
FORM;

    return \str_replace(
        '{csrfToken}',
        $request->getAttribute('csrfToken'),
        $form
    );
}
```

Токен также можно зарегистрировать как глобальную переменную представлений.

### Регистрация глобальной переменной представления

Пример через middleware:

```php
use Psr\Http\Server\MiddlewareInterface;
use Spiral\Views\GlobalVariablesInterface ;

class ViewCsrfTokenMiddleware implements MiddlewareInterface
{
    public function __construct(
        private readonly GlobalVariablesInterface $globalVariables
    ) {}
    
    public function process(
        ServerRequestInterface $request, 
        RequestHandlerInterface $handler
    ): ResponseInterface {
        $this->globalVariables->set('csrfToken', $request->getAttribute('csrfToken'));
        
        return $handler->handle($request)->withAddedHeader('My-Header', 'my-value');
    }
}
```

> **Смотрите также**
> Подробнее о глобальных переменных читайте в разделе
> [Представления — Основы](../views/basics.md#global-variables).

Добавьте middleware в список:

```php app/src/Application/Bootloader/RoutesBootloader.php
'web' => [
    CookiesMiddleware::class,
    CsrfMiddleware::class,
    ViewCsrfTokenMiddleware::class,
    CsrfFirewall::class,
    // ...
],
```

После этого переменная `csrfToken` доступна в представлениях:

```html app/views/user/password.dark.php
<form action="https://your-application.com/user/password" method="POST">
    <input type="hidden" name="csrf-token" value="{csrfToken}"/>
    <input name="password" type="password">
    // ...
    <button type="submit">Change password</button>
</form>
```