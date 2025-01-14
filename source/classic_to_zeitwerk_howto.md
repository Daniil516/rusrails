Как перейти с Classic на Zeitwerk
=================================

Это руководство документирует, как мигрировать приложение Rails с режима `classic` на `zeitwerk`.

После прочтения этого руководства вы узнаете:

* Что такое режимы `classic` и `zeitwerk`
* Зачем переключаться из `classic` в `zeitwerk`
* Как активировать режим `zeitwerk`
* Как проверить, что ваше приложение запущено в режиме `zeitwerk`
* Как проверить, что ваше проект правильно загружается в командной строке
* Как проверить, что ваше проект правильно загружается в тестах
* Как разрешить возможные крайние случаи
* Новые особенности в Zeitwerk, которые можно использовать

--------------------------------------------------------------------------------

* Что такое режимы `classic` и `zeitwerk`?
------------------------------------------

С самого начала и до Rails 5, Rails использовал автоматический загрузчик, реализованный в Active Support. Этот автозагрузчик, известный как `classic`, все еще доступен в Rails 6.x. Rails 7 больше не включает этот автозагрузчик.

Начиная с Rails 6, Rails поставляется с новым и лучшим способом автозагрузки, делегирующим гему [Zeitwerk](https://github.com/fxn/zeitwerk). Это режим `zeitwerk`. По умолчанию, приложения, загружающие умолчания для фреймворка 6.0 and 6.1, запускаются в режиме `zeitwerk`, и в Rails 7 это единственный доступный режим.

Зачем переключаться из `classic` в `zeitwerk`?
----------------------------------------------

Автозагрузчик `classic` был чрезвычайно полезным, но имел ряд [проблем](https://github.com/morsbox/rusrails/blob/6.1/source/autoloading_and_reloading_constants_classic_mode.md#common-gotchas-распространенные-случаи), которые иногда делали автоматическую загрузку немного запутанной и непонятной. Zeitwerk был разработан, чтобы их решить, среди прочих [мотивов](https://github.com/fxn/zeitwerk#motivation).

При обновлении на Rails 6.x крайне рекомендуется переключиться на режим `zeitwerk`, так как этот автозагрузчик лучше, а режим `classic` устарел.

Rails 7 заканчивает переходный период и больше не включает режим `classic`.

Мне страшно
-----------

Не бойтесь :).

Zeitwerk был разработан, чтобы быть как можно более совместимым с классическим автозагрузчиком. Если у вас сейчас есть корректно работающая автозагрузка приложения, переключение, скорее всего, будет простым. Многие проекты, большие и малые, отчитались о реально гладком переходе.

Это руководство поможет вам уверенно изменить автоматический загрузчик.

Если, по какой-то причине, вы попали в ситуацию, которую не знаете как разрешить, не стесняйтесь [открыть проблему в `rails/rails`](https://github.com/rails/rails/issues/new) и поставить тег [`@fxn`](https://github.com/fxn).

Как активировать режим `zeitwerk`
---------------------------------

### Приложения на Rails 5.x и ниже

В приложениях на версиях Rails до 6.0, режим `zeitwerk` недоступен. Нужен как минимум Rails 6.0.

### Приложения на Rails 6.x

В приложениях на Rails 6.x есть два сценария.

Если приложение загружает умолчания фреймворка Rails 6.0 или 6.1, и оно запускается в режиме `classic`, это должно быть установлено вручную. Вам нужно что-то наподобие этого:

```ruby
# config/application.rb
config.load_defaults 6.0
config.autoloader = :classic # УДАЛИТЕ ЭТУ СТРОЧКУ
```

Как отмечено, просто удалите переопределение, режим `zeitwerk` установлен по умолчанию.

С другой стороны, если приложение загружает умолчания старого фреймворка, вам нужно включить режим `zeitwerk` явно:

```ruby
# config/application.rb
config.load_defaults 5.2
config.autoloader = :zeitwerk
```

### Приложения на Rails 7

В Rails 7 имеется только режим `zeitwerk`, вам не нужно ничего делать, чтобы его включить.

На самом деле, в Rails 7 метод `config.autoloader=` даже не существует. Если `config/application.rb` его использует, пожалуйста удалите эту строчку.

Как проверить, что ваше приложение запущено в режиме `zeitwerk`?
----------------------------------------------------------------

Чтобы проверить, что приложение запускается в режиме `zeitwerk`, выполните

```
bin/rails runner 'p Rails.autoloaders.zeitwerk_enabled?'
```

Если это выведет `true`, режим `zeitwerk` включен.


Соответствует ли мое приложение соглашениям Zeitwerk?
-----------------------------------------------------

### config.eager_load_paths

Тест на соответствие запускается для нетерпеливо загружаемых файлов. Следовательно, чтобы проверить на соответствие Zeitwerk, рекомендовано иметь все пути автозагрузки в пути нетерпеливой загрузки.

Это уже так по умолчанию, но если в проекте есть пользовательские пути автозагрузки, сконфигурированные наподобие:

```ruby
config.autoload_paths << "#{Rails.root}/extras"
```

то они не будут нетерпеливо загружены и не будут проверены. Добавить их в пути нетерпеливой загрузки просто:

```ruby
config.autoload_paths << "#{Rails.root}/extras"
config.eager_load_paths << "#{Rails.root}/extras"
```

### zeitwerk:check

Как только режим `zeitwerk` включен и конфигурация путей нетерпеливой загрузки дважды проверена, запустите:

```
bin/rails zeitwerk:check
```

Успешная проверка выглядит так:

```
% bin/rails zeitwerk:check
Hold on, I am eager loading the application.
All is good!
```

Может быть дополнительный вывод в зависимости от конфигурации приложения, но итоговый "All is good!" это то, что вы должны увидеть.

Если двойная проверка, описанная в предыдущем разделе, определила, что фактически есть некоторые пользовательские пути автозагрузки вне путей нетерпеливой загрузки, задача их обнаружит и предупредит. Однако, если тестовый набор загружает эти файлы успешно, у вас все хорошо.

Теперь, если есть какой-то файл, который не определяет ожидаемую константу, задача вам подскажет. Она выводит один файл за раз, так как, если бы она продолжила, ошибка загрузки одного файла могла бы вызвать другие ошибки, не относящиеся к проверке, которую мы запустили, и отчет об ошибки мог бы быть запутанным.

Если выведена одна константа, почините ее и запустите задачу заново. Повторяйте, пока не получите "All is good!".

Возьмем, к примеру:

```
% bin/rails zeitwerk:check
Hold on, I am eager loading the application.
expected file app/models/vat.rb to define constant Vat
```

VAT это Европейский налог. Файл `app/models/vat.rb` определяет `VAT`, но автоматический загрузчик ожидает `Vat`, почему?

### Аббревиатуры

Это наиболее распространенный тип несоответствия, нужно разобраться с аббревиатурами. Давайте поймем, почему мы получаем это сообщение об ошибке.

Классический автозагрузчик способен автоматически загрузить `VAT`, так как у него на входе имя отсутствующей константы, `VAT`, он вызывает `underscore` на нем, что приводит к `vat`, и ищет файл с именем `vat.rb`. Это работает.

На входе у нового автозагрузчика файловая система. Взяв файл `vat.rb`, Zeitwerk вызывает `camelize` на `vat`, что приводит к `Vat`, и ожидает, что этот файл определяет константу `Vat`. Вот о чем говорит сообщение об ошибке.

Это просто починить, нужно всего лишь сообщить преобразователю слов об этой аббревиатуре:

```ruby
# config/initializers/inflections.rb
ActiveSupport::Inflector.inflections(:en) do |inflect|
  inflect.acronym "VAT"
end
```

Это повлияет на то, как Active Support образует слова глобально. Это может быть нормальным, но если хотите, можно также переопределить преобразователи слов, используемые автозагрузчиком:

```ruby
# config/initializers/zeitwerk.rb
Rails.autoloaders.main.inflector.inflect("vat" => "VAT")
```

С этой опцией у вас есть больше контроля, поскольку только файлы, названные непосредственно `vat.rb`, или директории, непосредственно названные `vat`, будут приведены к `VAT`. Файл, названный `vat_rules.rb`, не будет затронут этим, и может определять `VatRules`. Это может быть удобным, если в проекте есть такой тип несоответствий именования.

После добавления проверка проходит!

```
% bin/rails zeitwerk:check
Hold on, I am eager loading the application.
All is good!
```

Как только All is good, рекомендуется оставить валидацию проекта в тестовом наборе. Раздел [_Проверка правильности Zeitwerk в тестах_](#check-zeitwerk-compliance-in-the-test-suite) объясняет, как это сделать.

### Концерны

Можно автоматически и нетерпеливо загружать из стандартной структуры с поддиректориями `concerns` наподобие

```
app/models
app/models/concerns
```

По умолчанию, `app/models/concerns` принадлежит к путям автозагрузки, следовательно, подразумевается корневой директорией. Таким образом, по умолчанию `app/models/concerns/foo.rb` должен определять `Foo`, а не `Concerns::Foo`.

Если ваше приложение использует `Concerns` в качестве пространства имен, есть два варианта:

1. Убрать пространство имен `Concerns` из этих классов и модулей, и обновить клиентский код.
2. Оставить все как есть, убрав `app/models/concerns` из путей автозагрузки:

  ```ruby
  # config/initializers/zeitwerk.rb
  ActiveSupport::Dependencies.
    autoload_paths.
    delete("#{Rails.root}/app/models/concerns")
  ```

### Добавление `app` в пути автозагрузки

Некоторым проектам нужно, что что-то наподобие `app/api/base.rb` определяло `API::Base`, и для этого добавляют `app` в пути автозагрузки.

Так как Rails автоматически добавляет все поддиректории `app` в пути автозагрузки (с небольшим исключением), тут у нас другая ситуация со вложенными корневыми директориями, подобная той, что случилась с `app/models/concerns`. Эта настройка больше не будет работать как есть.

Однако, можно сохранить эту структуру, просто удалите `app/api` из путей автозагрузки в инициализаторе:

```ruby
# config/initializers/zeitwerk.rb
ActiveSupport::Dependencies.
  autoload_paths.
  delete("#{Rails.root}/app/api")
```

Остерегайтесь поддиректорий, в которых нет файлов, которые будут автоматически / нетерпеливо загружены. Например, если в приложении есть `app/admin` с ресурсами для [ActiveAdmin](https://activeadmin.info/), их нужно игнорировать. То же самое для `assets` сотоварищи:

```ruby
# config/initializers/zeitwerk.rb
Rails.autoloaders.main.ignore(
  "app/admin",
  "app/assets",
  "app/javascripts",
  "app/views"
)
```

Без такой настройки, приложение будет нетерпеливо загружать эти деревья. Не вызовет ошибку на `app/admin` из-за того, что ее файлы не определяют константы, и не определит модуль `Views`, к примеру, в качестве нежелательного стороннего эффекта.

Как видите, иметь `app` в путях автозагрузки технически возможно, но но немного запутано.

### Автоматически загруженные константы и явные пространства имен

Если в файле определено пространство имен, как `Hotel` тут:

```
app/models/hotel.rb         # Определяет Hotel.
app/models/hotel/pricing.rb # Определяет Hotel::Pricing.
```

константа `Hotel` должна быть установлена с помощью ключевых слов `class` или `module`. Например:

```ruby
class Hotel
end
```

это правильно.

Альтернативы, такие как

```ruby
Hotel = Class.new
```

или

```ruby
Hotel = Struct.new
```

не будут работать, дочерние объекты, такие как `Hotel::Pricing` не будут найдены.

Это ограничение применяется только для явных пространств имен. Классы и модули, не определяющие пространство имен, могут быть определены с помощью этих идиом.

### Один файл - одна константа (на том же уровне)

В режиме `classic` технически вы могли определить несколько констант на том же уровне, и получить их перезагружаемыми. Например, в

```ruby
# app/models/foo.rb

class Foo
end

class Bar
end
```

хотя `Bar` не мог быть автоматически загружаемым, автозагрузка `Foo` также пометила бы `Bar` как автоматически загруженным.

Это не так в режиме `zeitwerk`, вам нужно переместить `Bar` в собственный файл `bar.rb`. Один файл, одна константа верхнего уровня.

Это влияет только на константы того же уровня, как в вышеприведенном примере. Вложенные классы и модули это нормально. Например, рассмотрим

```ruby
# app/models/foo.rb

class Foo
  class InnerClass
  end
end
```

Если приложение перезагружает `Foo`, оно также перезагрузит `Foo::InnerClass`.

### Шаблоны поиска в `config.autoload_paths`

Остерегайтесь конфигураций, в которых используются подстановочные знаки, например

```ruby
config.autoload_paths += Dir["#{config.root}/extras/**/"]
```

Каждый элемент в `config.autoload_paths` должен представлять пространство имен верхнего уровня (`Object`). Это не будет работать.

Чтобы починить, просто уберите подстановочные знаки:

```ruby
config.autoload_paths << "#{config.root}/extras"
```

### Декорирование классов и модулей из engine

Если ваше приложение декорирует классы или модули из engine, вероятно вы делаете где-то что-то вроде этого:

```ruby
config.to_prepare do
  Dir.glob("#{Rails.root}/app/overrides/**/*_override.rb").each do |override|
    require_dependency override
  end
end
```

Это нужно обновить: нужно сообщить автозагрузчику `main` игнорировать директорию с переопределениями, и вам нужно загрузить их с помощью `load`. Что-то вроде:

```ruby
overrides = "#{Rails.root}/app/overrides"
Rails.autoloaders.main.ignore(overrides)
config.to_prepare do
  Dir.glob("#{overrides}/**/*_override.rb").each do |override|
    load override
  end
end
```

### `before_remove_const`

Rails 3.1 добавил поддержку для колбэка с именем `before_remove_const`, который вызывался, если класс или модуль отвечают на этот метод, и сейчас будет перезагружен. Этот колбэк остается недокументированным, и ваш код вряд ли его использует.

Однако, если он использует, следует переписать что-то вроде

```ruby
class Country < ActiveRecord::Base
  def self.before_remove_const
    expire_redis_cache
  end
end
```

как

```ruby
# config/initializers/country.rb
if Rails.application.config.reloading_enabled?
  Rails.autoloaders.main.on_unload("Country") do |klass, _abspath|
    klass.expire_redis_cache
  end
end
```

### Spring и окружение `test`

Spring перезагружает код приложения, если что-то изменилось. В среде `test` нужно включить перезагрузку, чтобы это работало:

```ruby
# config/environments/test.rb
config.cache_classes = false
```

или, начиная с Rails 7.1:

```ruby
# config/environments/test.rb
config.enable_reloading = true
```

В противном случае вы получите

```
reloading is disabled because config.cache_classes is true
```

или

```
reloading is disabled because config.enable_reloading is false
```

В этом нет никакого ухудшения производительности.

### Bootsnap

Убедитесь, что зависите от как минимум Bootsnap 1.4.4.

(check-zeitwerk-compliance-in-the-test-suite) Проверка правильности Zeitwerk в тестах
-------------------------------------------------------------------------------------

Задача `zeitwerk:check` удобна при миграции. Как только проект соответствует, рекомендуется автоматизировать эту проверку. Для этого достаточно нетерпеливо загрузить приложение, и, на самом деле, это единственное, что делает `zeitwerk:check`.

### Непрерывная интеграция

Если ваш проект имеет непрерывную интеграцию, неплохо было бы нетерпеливо загрузить приложение при запуске тестов там. Если приложение не сможет быть нетерпеливо загружено по какой-то причине, лучше узнать это в CI, чем в production, не правда ли?

В CI обычно имеется некая установленная переменная среды для обозначения, что тесты выполняются там. К примеру, это может быть `CI`:

```ruby
# config/environments/test.rb
config.eager_load = ENV["CI"].present?
```

Начиная с Rails 7, новые приложения конфигурируются таким способом по умолчанию.

### Чистые тесты

Если в вашем проекте нет непрерывной интеграции, вы все еще можете нетерпеливо загружать в тестах, вызывая `Rails.application.eager_load!`:

#### minitest

```ruby
require "test_helper"

class ZeitwerkComplianceTest < ActiveSupport::TestCase
  test "eager loads all files without errors" do
    assert_nothing_raised { Rails.application.eager_load! }
  end
end
```

#### RSpec

```ruby
require "rails_helper"

RSpec.describe "Zeitwerk compliance" do
  it "eager loads all files without errors" do
    expect { Rails.application.eager_load! }.not_to raise_error
  end
end
```

Удаление любых вызовов `require`
--------------------------------

Проекты обычно так не делают. Но иногда так бывает.

В приложениях Rails `require` используется эксклюзивно для загрузки кода из `lib` или кода третьих сторон, например гемов или стандартной библиотеки. **Никогда не загружайте автоматически загружаемый код приложения с помощью `require`**. Посмотрите, почему это уже было плохой идеей в `classic`, [тут](https://github.com/morsbox/rusrails/blob/6.1/source/autoloading_and_reloading_constants_classic_mode.md#автозагрузка-и-require).

```ruby
require "nokogiri" # ХОРОШО
require "net/http" # ХОРОШО
require "user"     # ПЛОХО, УДАЛИТЕ ЭТО (подразумеваем app/models/user.rb)
```

Пожалуйста, удалите любые вызовы `require` этого типа.

Новые особенности, которые можно использовать
---------------------------------------------

### Удаление вызовов `require_dependency`

Все известные случаи использования `require_dependency` были устранены в Zeitwerk. Можно найти и удалить их в проекте.

Если ваше приложение использует наследование с единой таблицей, обратитесь к [разделу по Single Table Inheritance](/autoloading-and-reloading-constants/#single-table-inheritance) руководства по автозагрузке и перезагрузке констант (режим Zeitwerk).

### Теперь возможны полные имена в определениях класса и модуля

Теперь можно с уверенностью использовать пути констант в определениях модуля и класса:

```ruby
# Автозагрузка в теле этого класса теперь соответствует семантике Ruby.
class Admin::UsersController < ApplicationController
  # ...
end
```

Хитрость, о которой нужно было знать, в том, что, в зависимости от порядка выполнения, классический автозагрузчик иногда мог автоматически загрузить `Foo::Wadus` в

```ruby
class Foo::Bar
  Wadus
end
```

Это не соответствует семантике Ruby, так как `Foo` не во вложенности, и не будет работать в режиме `zeitwerk`. Если вы обнаружите такой случай, можно использовать полное имя `Foo::Wadus`:

```ruby
class Foo::Bar
  Foo::Wadus
end
```

или добавить `Foo` во вложенность:

```ruby
module Foo
  class Bar
    Wadus
  end
end
```

### Повсеместная тредобезопасность

В режиме `classic` автозагрузка констант не является тредобезопасной, хотя в самом Rails есть блокировки, например, чтобы сделать веб-запросы тредобезопасными.

Автозагрузка констант в режиме `zeitwerk` является тредобезопасной. Например, теперь можно автоматически загрузить в многотредовых скриптах, выполняемых с помощью команды `runner`.

### Нетерпеливая загрузка и автозагрузка согласованные

В режиме `classic` если `app/models/foo.rb` определяет `Bar`, вы не сможете автоматически загрузить этот файл, но нетерпеливая загрузка будет работать, так как она загружает файлы рекурсивно вслепую. Это может быть источником ошибок, если вы сначала тестируете что-то нетерпеливо загрузив, а потом выполнение выдаст ошибку при автозагрузке.

В режиме `zeitwerk` оба режима загрузки согласованы, они выдают ошибку в тех же самых файлах.
