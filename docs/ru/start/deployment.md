# Начало работы — Развёртывание

Развёртывание приложения Spiral может быть сложной задачей: необходимо выполнить несколько этапов, чтобы правильно
настроить приложение и подготовить его к работе в производственной среде. В этом руководстве рассмотрены различные
стратегии развёртывания: простая передача файлов, использование системы контроля версий, сценарии сборки и Docker.

## Подготовка приложения

Для корректной и безопасной работы приложения Spiral на производственном сервере необходимо выполнить несколько
основных настроек.

> **Предупреждение**
> Не храните файл `.env` в репозитории: он может содержать конфиденциальные данные, включая учётные данные базы данных,
> ключи API и другие секреты.

Ниже приведена пошаговая настройка приложения Spiral для производственного сервера.

### Отключите режим отладки

Установите переменной `DEBUG` значение `false` в конфигурации окружения приложения. Это предотвратит отображение
конфиденциальных данных при возникновении ошибки.

```dotenv .env
DEBUG=false
```

### Установите производственное окружение

Установите переменной `APP_ENV` значение `production`. Это помогает предотвратить случайный запуск действий,
предназначенных только для среды разработки.

```dotenv .env
APP_ENV=production
```

### Включите кеш токенизатора

Установите переменной `TOKENIZER_CACHE_TARGETS` значение `true`. Это отключит повторное сканирование и анализ исходного
кода приложения, что может ускорить начальную загрузку.

```dotenv .env
TOKENIZER_CACHE_TARGETS=true
```

> **Примечание**
> Подробнее о токенизаторе читайте в разделе
> [Компоненты — Статический анализ](../advanced/tokenizer.md).

### Установите уровень подробности

Установите переменной `VERBOSITY_LEVEL` значение `basic`, чтобы ошибки сервера не отображались публично.

```dotenv .env
VERBOSITY_LEVEL=basic
```

### Настройте журналирование

Укажите необходимый `MONOLOG_DEFAULT_CHANNEL` в конфигурации окружения приложения. Он определяет, куда будут
записываться журналы приложения. Также установите `MONOLOG_DEFAULT_LEVEL` в `error`, чтобы отладочные и информационные
сообщения не попадали в журнал и он оставался компактным и удобным для чтения.

```dotenv .env
MONOLOG_DEFAULT_CHANNEL=roadrunner
MONOLOG_DEFAULT_LEVEL=error
```

### Cycle ORM

При использовании Cycle ORM установите переменным `CYCLE_SCHEMA_CACHE` и `CYCLE_SCHEMA_WARMUP` значение `true`.

Переменная `CYCLE_SCHEMA_CACHE` определяет, должна ли ORM кешировать схему таблиц базы данных. При значении `true` схема
кешируется, что повышает производительность за счёт уменьшения количества запросов для получения сведений о ней.

Переменная `CYCLE_SCHEMA_WARMUP` определяет, должна ли ORM прогревать кеш схемы при запуске приложения. При значении
`true` кеш заранее заполняется сведениями о схеме, что дополнительно сокращает время их получения.

```dotenv .env
CYCLE_SCHEMA_CACHE=true
CYCLE_SCHEMA_WARMUP=true
```

### Composer

Для установки зависимостей приложения Spiral можно использовать следующую команду Composer:

```terminal
composer install --optimize-autoloader --no-dev --no-scripts
```

Она устанавливает зависимости приложения, создаёт файлы автозагрузчика и оптимизирует их для повышения
производительности.

- параметр `--no-dev` запрещает установку зависимостей для разработки;
- параметр `--no-scripts` запрещает выполнение сценариев `post-install`.

## Nginx

### Проксирование в RoadRunner

Чтобы использовать Spiral с RoadRunner за сервером Nginx, настройте Nginx как обратный прокси.

Пример конфигурации:

```nginx /etc/nginx/sites-enabled/roadrunner.conf
server {
    listen 80;

    server_name _;

    location / {
        proxy_pass http://127.0.0.1:8080;
    }
}
```

Эта конфигурация заставляет Nginx прослушивать порт `80` и перенаправлять все входящие запросы на
`127.0.0.1:8080`, где работает RoadRunner.

Конфигурационный файл можно поместить в каталог `/etc/nginx/sites-available/`, а затем создать на него символическую
ссылку в `/etc/nginx/sites-enabled/`.

Пример конфигурации HTTP-сервера RoadRunner:

```yaml .rr.yaml
http:
  address: 127.0.0.1:8080
```

> **Предупреждение**
> Не используйте адрес `0.0.0.0:8080` в конфигурации RoadRunner: доступ к HTTP-серверу должен осуществляться только
> через обратный прокси Nginx.

### PHP-FPM

Для использования Spiral с PHP-FPM выполните следующие действия.

1. Установите пакет `spiral/sapi-bridge`, предоставляющий диспетчер для обработки запросов через PHP-FPM.

```terminal
composer require spiral/sapi-bridge
```

После установки зарегистрируйте загрузчик `Spiral\Sapi\Bootloader\SapiBootloader` в списке загрузчиков приложения.

:::: tabs

::: tab С помощью метода

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Sapi\Bootloader\SapiBootloader::class,
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
    \Spiral\Sapi\Bootloader\SapiBootloader::class,
    // ...
];
```

Подробнее о загрузчиках читайте в разделе [Фреймворк — Загрузчики](../framework/bootloaders.md).
:::

::::

2. Настройте Nginx для работы с PHP-FPM.

Пример конфигурации Nginx:

```nginx /etc/nginx/sites-enabled/spiral.conf
server {
    listen 80;
    listen [::]:80;
    server_name example.com;
    root /srv/example.com/public;
 
    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";
 
    index app.php;
 
    charset utf-8;
 
    location / {
        try_files $uri $uri/ /app.php?$query_string;
    }
 
    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }
 
    error_page 404 /app.php;
 
    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }
 
    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

## Deployer

Средства автоматизации, такие как [Deployer](https://deployer.org/), позволяют автоматизировать развёртывание,
миграции базы данных, очистку кеша и другие операции. Они ускоряют развёртывание и откат, а также интегрируются с
балансировщиками нагрузки и системами мониторинга. Это делает процесс эффективнее, снижает риск ошибок и упрощает
отслеживание версии кода, установленной на производственном сервере.

Deployer — популярный инструмент автоматизации развёртывания приложений Spiral. Одно из его преимуществ — возможность
развёртывания без простоя.

Развёртывание без простоя позволяет обновить приложение, не прерывая обслуживание пользователей. Новая версия
развёртывается рядом со старой, а после готовности версии переключаются. Пользователи не замечают простоя во время
обновления.

Deployer реализует такой подход, создавая новый выпуск приложения в отдельном каталоге и переключая символическую ссылку
на него.

### Установка

Установите Deployer в каталоге проекта:

```terminal
composer require --dev deployer/deployer
```

### Конфигурация

Инициализируйте Deployer:

```terminal
./vendor/bin/dep init
```

Deployer задаст несколько вопросов и создаст файл `deploy.php` или `deploy.yaml`. Это сценарий развёртывания, содержащий
хосты, задачи и подключения других сценариев.

Пример файла `deploy.php`:

```php deploy.php
namespace Deployer;

require 'recipe/spiral.php';

set('repository', 'https://github.com/xxx/my-app');

add('shared_files', []);
add('shared_dirs', []);
add('writable_dirs', []);

host('example.org')
    ->set('remote_user', 'deployer')
    ->set('deploy_path', '/var/www/my-app');

after('deploy:failed', 'deploy:unlock');

desc('Deploys your project');
task('deploy', [
    'deploy:prepare',
    'deploy:environment',
    'deploy:vendors',
    'spiral:encrypt-key',
    'spiral:configure',
    'deploy:download-rr',
    'deploy:publish',
    'deploy:restart-rr'
]);
```

Для подключения к удалённому серверу необходимо указать идентификационный или закрытый ключ. Ключ можно добавить
непосредственно в описание хоста, но лучше поместить его в `~/.ssh/config`:

```bash ~/.ssh/config
Host example.org
  IdentityFile ~/.ssh/id_rsa
```

Теперь подготовим сервер. Поскольку на хосте ещё нет пользователя `deployer`, для подготовки переопределим
`remote_user` через параметр `-o remote_user=root`:

```terminal
dep provision -o remote_user=root
```

Во время подготовки Deployer задаст несколько вопросов: о версии PHP, типе базы данных и других параметрах. Затем он
настроит сервер и создаст пользователя `deployer`. Подготовка занимает около пяти минут и устанавливает всё необходимое
для работы сайта. Новый сайт будет настроен в каталоге `deploy_path`.

### Развёртывание

Для развёртывания приложения выполните:

```terminal
./vendor/bin/dep deploy
```

> **Примечание**
> Если развёртывание завершится ошибкой, Deployer выведет сообщение и укажет неудачно выполненную команду. Чаще всего
> требуется указать правильные учётные данные базы данных в `.env` или аналогичном файле.

#### CI/CD

Deployer можно использовать в конвейерах CI/CD.

> **Смотрите также**
> Подробнее читайте в разделе [Deployer CI/CD](https://deployer.org/docs/7.x/ci-cd).

<hr>

## Docker

Docker упрощает развёртывание, масштабирование и управление контейнерами приложения. Он также позволяет воспроизводить
производственное окружение локально, что облегчает поиск и исправление ошибок.

При этом способе создаётся Docker-образ приложения, который затем запускается на производственном сервере.

### Dockerfile

Пример `Dockerfile` для сборки образа приложения Spiral:

```dockerfile 
# This example will work with application root directory as docker context
FROM php:8.2-cli-alpine3.17 as backend

RUN  --mount=type=bind,from=mlocati/php-extension-installer:1.5,source=/usr/bin/install-php-extensions,target=/usr/local/bin/install-php-extensions \
      install-php-extensions opcache zip xsl dom exif intl pcntl bcmath sockets && \
     apk del --no-cache  ${PHPIZE_DEPS} ${BUILD_DEPENDS}

WORKDIR /app

ENV COMPOSER_ALLOW_SUPERUSER=1
COPY --from=composer:2.3 /usr/bin/composer /usr/bin/composer
COPY ./composer.* .
RUN composer config --no-plugins allow-plugins.spiral/composer-publish-plugin false && \
    composer install --optimize-autoloader --no-dev

COPY --from=spiralscout/roadrunner:latest /usr/bin/rr /app

EXPOSE 8080/tcp

COPY ./ .

CMD ./rr serve -c .rr.yaml
```

Соберите образ:

```terminal
docker build . -t my-application:latest
```

После сборки отправьте его в реестр Docker:

```terminal
docker push my-application:latest
```

Образу также можно присвоить тег версии, например `my-application:1.0`:

```terminal
docker build . -t my-application:latest -t my-application:1.0
docker push my-application:latest
docker push my-application:1.0
```

### Docker Compose

Одно из преимуществ Docker — возможность управлять переменными окружения через файл `docker-compose` и внешний файл
`.env`, не создавая `.env` внутри контейнера.

Пример `docker-compose.yml` для запуска приложения:

```yaml docker-compose.yaml
version: '3'
services:
  app:
    image: my-application:1.0
    ports:
      - "8080:8080"
    environment:
      - DEBUG=false
      - APP_ENV=production
      - ...
...
```

### Запуск и остановка приложения

Запустите приложение:

```terminal
docker-compose up -d
```

Остановите его:

```terminal
docker-compose down
```

<hr>

## Система контроля версий

Распространённый способ развёртывания — подключить производственный сервер к удалённому репозиторию Git, SVN или другой
системы контроля версий и получать из него последние изменения.

### Пример

1. Подключитесь к производственному серверу по SSH.
2. Перейдите в корневой каталог приложения на сервере.
3. Выполните `git pull origin master`, чтобы получить последние изменения из удалённого репозитория.
4. Выполните `composer install --optimize-autoloader --no-dev` для установки зависимостей.
5. Запустите миграции базы данных и другие необходимые консольные команды.
6. Перезапустите воркеры RoadRunner командой `./rr reset`.

> **Предупреждение**
> Без автоматизации такая стратегия требует больше времени и подвержена ошибкам. При каждом развёртывании разработчику
> приходится вручную получать код и выполнять консольные команды в определённом порядке. Команды можно перепутать или
> пропустить, что способно нарушить работу приложения.

<hr>

## Передача файлов

Один из самых простых способов развернуть приложение Spiral — передать файлы по FTP или SCP с локального компьютера на
производственный сервер.

### Пример

1. Подключитесь к производственному серверу через FTP-клиент, например FileZilla или WinSCP.
2. Перейдите в корневой каталог приложения на сервере.
3. Выберите все локальные файлы приложения и загрузите их на сервер.
4. Выполните `composer install --optimize-autoloader --no-dev` для установки зависимостей.
5. Запустите миграции базы данных и другие необходимые консольные команды.
6. Перезапустите воркеры RoadRunner командой `./rr reset`.

> **Предупреждение**
> Эта стратегия считается менее эффективной и безопасной, чем остальные. Ручная загрузка всех файлов может занять
> много времени, а прерывание передачи создаёт риск потери или повреждения данных. Кроме того, сложнее определить, какая
> версия кода находится на производственном сервере, и выполнить откат при ошибке.
