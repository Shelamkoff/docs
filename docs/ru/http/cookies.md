# HTTP — Cookie

Стандартный каркас приложения включает поддержку cookie по умолчанию.

Чтобы включить cookie в альтернативной сборке, установите пакет Composer `spiral/cookies` и добавьте загрузчик
`Spiral\Bootloader\Http\CookiesBootloader`.

## Менеджер cookie

Проще всего управлять cookie через экземпляр `Spiral\Cookies\CookieManager`. Его можно хранить в singleton-сервисах и
контроллерах: он предоставляет доступ к активной области запроса.

```php
public function index(CookieManager $cookies): void
{
    dump($cookies->getAll());
    $cookies->set('name', 'value'); // read about more options down below
}
```

Если используется расширение `spiral/prototype`, менеджер также доступен через прототипное свойство `cookies`:

```php
use PrototypeTrait;

public function index(): void
{
    dump($this->cookies->getAll());
    $this->cookies->set('name', 'value'); // read about more options down below
}
```

Ниже описано низкоуровневое управление cookie.

## Чтение cookie

По умолчанию фреймворк шифрует и расшифровывает значения всех cookie с помощью ключа окружения `ENCRYPTER_KEY`.
Изменение ключа автоматически сделает недействительными все cookie, ранее установленные пользователям.

> **Примечание**
> Шифрование можно отключить ради производительности либо использовать альтернативную подпись HMAC — смотрите ниже.

Компонент расшифровывает все значения и обновляет объект запроса. Получить cookie можно через стандартный PSR-7
`ServerRequestInterface`:

```php
use Psr\Http\Message\ServerRequestInterface;

// ...

public function index(ServerRequestInterface $request): void
{
    dump($request->getCookieParams());
}
```

В качестве альтернативы используйте `Spiral\Http\Request\InputManager`. Он автоматически разрешает область запроса и
может храниться как singleton:

```php
class HomeController
{
    private $input;

    public function __construct(InputManager $input)
    {
        $this->input = $input;
    }

    public function index(): void
    {
        dump($this->input->cookies->all());
    }
}
```

> **Примечание**
> Значения cookie также можно получать в фильтрах запросов.

Если значение cookie недействительно или его невозможно расшифровать, оно будет заменено на `NULL` и не попадёт в
приложение.

## Запись cookie

Поскольку значения должны быть зашифрованы или подписаны, для записи используйте контекстный объект
`Spiral\Cookies\CookieQuery`.

```php
public function index(CookieQuery $cookies): void
{
    $cookies->set('name', 'value');
}
```

Метод принимает следующие аргументы в указанном порядке:

| Параметр | Тип    | Описание |
|----------|--------|----------|
| Name     | string | Имя cookie |
| Value    | string | Значение cookie, сохраняемое на компьютере клиента. Не храните в нём конфиденциальные сведения |
| Lifetime | int    | Время жизни в секундах относительно текущего момента |
| Path     | string | Путь сервера, в пределах которого доступна cookie. `/` означает весь домен, `/foo/` — каталог `/foo/` и его подкаталоги |
| Domain   | string | Домен доступности. `.example.com` делает cookie доступной всем поддоменам, `www.example.com` — только поддомену `www` |
| Secure   | bool   | При `true` cookie передаётся только через защищённое HTTPS-соединение |
| HttpOnly | bool   | При `true` cookie доступна только через HTTP и недоступна JavaScript, что помогает снизить риск кражи через XSS |

> **Примечание**
> Те же аргументы принимает `Spiral\Cookies\CookieManager::set()`.

## Использование в singleton-сервисах

`CookieQuery` нельзя внедрять через `__construct`: очередь доступна только в IoC-контексте `CookieMiddleware`. Получайте
её непосредственно из контейнера или используйте внедрение в метод, как показано выше:

```php
$container->get(CookieQuery::class)->set($name, $value);
```

> **Примечание**
> Наиболее подходящее место для `CookieQuery` — методы контроллера.

При наличии `ServerRequestInterface` очередь также доступна через атрибут `cookieQueue`:

```php
use Psr\Http\Message\ServerRequestInterface;

// ...

public function index(ServerRequestInterface $request): void
{
    $request->getAttribute('cookieQueue')->set('name', 'value');
}
```

## Ручная установка cookie

Заголовок cookie всегда можно сформировать вручную методом `withAddedHeader` интерфейса
`Psr\Http\Message\ResponseInterface`:

```php
return $response->withAddedHeader('Set-Cookie', 'name=value');
```

> Добавьте такую cookie в белый список, иначе `CookieMiddleware` не пропустит её.

## Конфигурация

Поведение `CookieQueue` настраивается через `Spiral\Bootloader\Http\CookiesBootloader`.

Чтобы добавить cookie в белый список и отключить для неё защиту:

```php
public function boot(CookiesBootloader $cookies): void
{
    $cookies->whitelistCookie('CustomCookie');
}
```

Для более глубокой настройки создайте файл `app/config/cookies.php`:

```php app/config/cookies.php
use Spiral\Cookies\Config\CookiesConfig;

return [
    // by default all cookies will be set as .domain.com
    'domain'   => '.%s',

    // protection method
    'method'   => CookiesConfig::COOKIE_ENCRYPT,

    // whitelisted cookies (no encrypt/decrypt)
    'excluded' => ['PHPSESSID', 'csrf-token']
];
```
