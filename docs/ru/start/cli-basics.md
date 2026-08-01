# Начало работы — Первая консольная команда

Spiral предлагает удобный способ создания консольных приложений. Фреймворк поддерживает консольные команды из
коробки, позволяя разрабатывать интерфейсы командной строки для приложения. С их помощью можно автоматизировать задачи,
выполнять обслуживание и взаимодействовать с приложением за пределами возможностей обычного веб-интерфейса.

Работать с консольными командами в Spiral очень просто. Фреймворк предоставляет удобный интерфейс на основе возможностей
пакета `symfony/console`.

Рассмотрим основные шаги создания консольной команды.

## Создание команды

Чтобы быстро создать первую команду, воспользуйтесь генератором:

```terminal
php app.php create:command CurrentDate
```

> **Примечание**
> Подробнее о генерации кода читайте в разделе
> [Основы — Генерация кода](../basics/scaffolding.md#console-command).

После выполнения команды успешное создание будет подтверждено следующим выводом:

```output
Declaration of '[32mCurrentDateCommand[39m' has been successfully written into '[33mapp/src/Endpoint/Console/CurrentDateCommand.php[39m'.
```

Теперь добавим логику в созданную команду.

Пример консольной команды, выводящей текущую дату:

```php app/src/App/Endpoint/Console/CurrentDateCommand.php
namespace App\Endpoint\Console;

use Spiral\Console\Attribute\Argument;
use Spiral\Console\Attribute\AsCommand;
use Spiral\Console\Attribute\Option;
use Spiral\Console\Attribute\Question;
use Spiral\Console\Command;

#[AsCommand(name: 'current:date')]
final class CurrentDateCommand extends Command
{
    #[Argument(description: 'Date format')]
    public string $format = 'Y-m-d';

    public function __invoke(): int
    {
        $this->writeln(\date($this->format));

        return self::SUCCESS;
    }
}
```

По умолчанию Spiral автоматически обнаруживает команды в каталоге `app/src` с помощью
[компонента статического анализа](../advanced/tokenizer.md). Поэтому команды не требуется регистрировать вручную или
добавлять для них отдельный конфигурационный файл.

## Запуск команды

Чтобы получить справку по команде, выполните в терминале:

```terminal
php app.php help current:date
```

Будут показаны сигнатура и описание команды, а также доступные аргументы и параметры.

```output
[33mDescription:[39m
  Get current date

[33mUsage:[39m
  current:date [<format>]

[33mArguments:[39m
  [32mformat[39m                Date format[33m [default: "Y-m-d"][39m

[33mOptions:[39m
  [32m-h, --help[39m            Display help for the given command. When no command is given display help for the [32mlist[39m command
  [32m-q, --quiet[39m           Do not output any message
  [32m-V, --version[39m         Display this application version
  [32m    --ansi|--no-ansi[39m  Force (or disable --no-ansi) ANSI output
  [32m-n, --no-interaction[39m  Do not ask any interactive question
  [32m-v|vv|vvv, --verbose[39m  Increase the verbosity of messages: 1 for normal output, 2 for more verbose output and 3 for debug
```

<br>

**Готово! Вы успешно создали первую консольную команду в Spiral.**

<hr>

## Что дальше?

Для более глубокого знакомства с основами прочитайте следующие разделы:

* [Конфигурация CLI](../console/configuration.md);
* [Создание команды](../console/commands.md);
* [Перехватчики](../console/interceptors.md);
* [Валидация входных данных команды](../cookbook/console-validation.md);
* [Генерация кода](../basics/scaffolding.md).
