Апгрейд Ruby on Rails
=====================

В этом руководстве приведены шаги, которые необходимо выполнить, чтобы апгрейднуть приложение на новую версию Ruby on Rails. Эти шаги также доступны в отдельных руководствах по релизам.

Общий совет
-----------

Перед попыткой апгрейда существующего приложения, следует убедиться, что есть хорошая причина для апгрейда. Нужно соблюсти баланс между несколькими факторами: необходимостью в новых особенностях, увеличением сложности в поиске поддержки для старого кода, доступностью вашего времени и навыков - это только некоторые из многих.

### Тестовое покрытие

Лучшим способом убедиться, что приложение продолжит работать после апгрейда, это иметь хорошее тестовое покрытие до начала апгрейда. Если у вас нет автоматических тестов, проверяющих большую часть вашего приложения, тогда нужно потратить время, проверяя все части, которые изменились. В случае обновления Rails это означает каждый отдельный кусок функциональности приложения. Пожалейте себя и убедитесь в хорошем тестовом покрытии _до_ начала апгрейда.

### Версии Ruby

В основном Rails использует последние выпущенные версии Ruby:

* Rails 7 требует Ruby 2.7.0 или новее.
* Rails 6 требует Ruby 2.5.0 или новее.
* Rails 5 требует Ruby 2.2.2 или новее.

Хорошей идеей будет обновлять Ruby и Rails раздельно. Сначала обновитесь на последний Ruby, а потом обновляйте Rails.

### Процесс апгрейда

При изменении версий Rails лучше двигаться медленно, одна второстепенная версия за раз, чтобы результативно использовать предупреждения об устаревании. Версии Rails записываются в форме Major.Minor.Patch. В главной (Major) и второстепенной (Minor) версиях допустимо делать изменения в публичном API, и это может вызвать ошибки в вашем приложении. Версии Patch включают только исправления ошибок и не изменяют публичное API.

Процесс должен быть следующим:

1. Пишете тесты и убеждаетесь, что они проходят.
2. Переходите к последней версии патча, следующую после вашей текущей версии.
3. Чините тесты и устаревшие особенности.
4. Переходите к последней версии патча следующей второстепенной версии.

Повторяйте этот процесс, пока не достигнете целевой версии Rails.

#### Переход между версиями

Чтобы перейти к версии:

1. Измените номер версии Rails в `Gemfile` и запустите `bundle update`.
2. Измените версии для пакетов Rails JavaScript в `package.json` и запустите `yarn install`, если вы на Webpacker.
3. Запустите [задачу Update](#the-update-task).
4. Запустите свои тесты.

Полный список всех выпущенных версий Rails можно найти [тут](https://rubygems.org/gems/rails/versions).

### (the-update-task) Задача Update

Rails предоставляет команду `rails app:update`. После обновления версии Rails в `Gemfile`, запустите эту команду. Она поможет вам с созданием новых файлов и изменением старых файлов в интерактивной сессии.

```bash
$ bin/rails app:update
       exist  config
    conflict  config/application.rb
Overwrite /myapp/config/application.rb? (enter "h" for help) [Ynaqdh]
       force  config/application.rb
      create  config/initializers/new_framework_defaults_7_0.rb
...
```

Не забывайте просматривать разницу, чтобы увидеть какие-либо неожидаемые изменения.

### Настройка умолчаний фреймворка

Возможно, что новая версия Rails будет иметь другие настройки по умолчанию, чем в предыдущей версии. Однако, после следования шагам, описанным ниже, ваше приложение все еще будет запускаться с настройками по умолчанию из *предыдущей* версии Rails. Это так, потому что значения для `config.load_defaults` в `config/application.rb` не были пока изменены.

Чтобы позволить обновиться до новых значений по умолчанию один за другим, задача обновления создала файл `config/initializers/new_framework_defaults_X.Y.rb` (с желаемой версией Rails в имени файла). Следует включать новые конфигурационные значения по умолчанию, снимая комментарий с них в этом файле; это можно сделать постепенно на протяжение нескольких развертываний. Как только ваше приложение готово быть запущенным с новыми значениями по умолчанию, этот файл можно удалить и изменить значение `config.load_defaults`.

(Upgrading from Rails 6.1 to Rails 7.0) Апгрейд с Rails 6.1 на Rails 7.0
------------------------------------------------------------------------

### Spring

Если ваше приложение использует Spring, он должен быть обновлен до, как минимум, версии 3.0.0. В противном случае вы получите

```
undefined method `mechanism=' for ActiveSupport::Dependencies:Module
```

А также убедитесь, что `config.cache_classes` установлен `false` в `config/environments/test.rb`.

### Приложения должны запускаться в режиме `zeitwerk`

Приложения, все еще запущенные в режиме `classic`, должны быть переключены в режим `zeitwerk`. Пожалуйста, обратитесь к руководству [Classic to Zeitwerk HOWTO](https://guides.rubyonrails.org/classic_to_zeitwerk_howto.html).

### Метод назначения `config.autoloader=` был удален

В Rails 7 больше нет конфигурационной настройки для установки режима автоматической загрузки, `config.autoloader=` был удален. Если вам нужно было назначить `:zeitwerk` по какой-то причине, просто уберите ее.

### Приватный API `ActiveSupport::Dependencies` был удален

Приватный API `ActiveSupport::Dependencies` был удален. Он включал методы, такие как `hook!`, `unhook!`, `depend_on`, `require_or_load`, `mechanism` и многие другие.

Немного основных моментов:

* Если вы использовали `ActiveSupport::Dependencies.constantize` или `ActiveSupport::Dependencies.safe_constantize`, просто измените их на `String#constantize` или `String#safe_constantize`.

  ```ruby
  ActiveSupport::Dependencies.constantize("User") # БОЛЬШЕ НЕВОЗМОЖНО
  "User".constantize # 👍
  ```

* Любое использование `ActiveSupport::Dependencies.mechanism`, чтение или запись, должно быть заменено доступом к `config.cache_classes`, соответственно.

* Если хотите отследить активность автоматического загрузчика, `ActiveSupport::Dependencies.verbose=` больше не доступен, просто передайте в `Rails.autoloaders.log!` в `config/application.rb`.

Вспомогательные внутренние классы или модули тоже исчезли, такие как `ActiveSupport::Dependencies::Reference`, `ActiveSupport::Dependencies::Blamable` и прочие.

### Автоматическая загрузка во время инициализации

Приложения, которые автоматически загружают перезагружаемые константы во время инициализации вне блоков `to_prepare`, получали эти константы выгруженными, и получали это предупреждение, начиная с Rails 6.0:

```
DEPRECATION WARNING: Initialization autoloaded the constant ....

Being able to do this is deprecated. Autoloading during initialization is going
to be an error condition in future versions of Rails.

...
```

Если вы все еще получаете это предупреждение в логах, обратитесь к разделу об автоматической загрузки при запуске приложения в [руководстве по автозагрузке](/constant_autoloading_and_reloading#autoloading-when-the-application-boots). В противном случае вы получите `NameError` в Rails 7.

### Возможность настроить `config.autoload_once_paths`

Можно установить `config.autoload_once_paths` в классе приложения, определенном в `config/application.rb`, или в конфигурациях для сред в `config/environments/*`.

Схожим образом в engine можно настроить эту коллекцию в классе engine или в конфигурациях для сред.

После этого коллекция замораживается, и вы можете автоматически загружать из этих путей. В частности, оттуда можно загружать во время инициализации. Они управляются автоматическим загрузчиком `Rails.autoloaders.once`, который не перезагружает, а только автоматически/нетерпеливо загружает.

Если вы установили эту настройку после того, как конфигурации для сред были обработаны, и получили `FrozenError`, просто переместите этот код.

### `ActionDispatch::Request#content_type` теперь возвращает заголовок Content-Type как есть.

Раньше возвращаемое значение `ActionDispatch::Request#content_type` НЕ содержало часть charset. Это поведение изменилось, и возвращаемый заголовок Content-Type содержит часть charset как его часть.

Если вам нужен только тип MIME, используйте вместо этого `ActionDispatch::Request#media_type`.

До:

```ruby
request = ActionDispatch::Request.new("CONTENT_TYPE" => "text/csv; header=present; charset=utf-16", "REQUEST_METHOD" => "GET")
request.content_type #=> "text/csv"
```

После:

```ruby
request = ActionDispatch::Request.new("Content-Type" => "text/csv; header=present; charset=utf-16", "REQUEST_METHOD" => "GET")
request.content_type #=> "text/csv; header=present; charset=utf-16"
request.media_type   #=> "text/csv"
```

### Класс дайджеста для генерации ключей изменили на использование SHA256

Класс дайджеста по умолчанию для генерации ключей изменили с SHA1 на SHA256. Последствия этого в любых зашифрованных сообщениях, генерируемых Rails, включая зашифрованные куки.

Для возможности читать сообщения с помощью старого класса дайджеста необходимо зарегистрировать ротатор.

Вот пример ротатора для зашифрованных куки.

```ruby
Rails.application.config.action_dispatch.cookies_rotations.tap do |cookies|
  salt = Rails.application.config.action_dispatch.authenticated_encrypted_cookie_salt
  secret_key_base = Rails.application.secrets.secret_key_base

  key_generator = ActiveSupport::KeyGenerator.new(
    secret_key_base, iterations: 1000, hash_digest_class: OpenSSL::Digest::SHA1
  )
  key_len = ActiveSupport::MessageEncryptor.key_len
  secret = key_generator.generate_key(salt, key_len)

  cookies.rotate :encrypted, secret
end
```

### Класс дайджеста для ActiveSupport::Digest изменили на SHA256

Класс дайджеста по умолчанию для ActiveSupport::Digest изменили с SHA1 на SHA256. Последствия этого в том, что такие вещи как Etag или ключи хэша, изменяться. Изменение этих ключей влияет на обращение к хэшу, будьте осторожны и следите за этим при обновлении на новый хэш.

### Новый формат сериализации ActiveSupport::Cache

Был представлен более быстрый и компактный формат сериализации.

Чтобы его включить, вы должны установить `config.active_support.cache_format_version = 7.0`:

```ruby
# config/application.rb

config.load_defaults 6.1
config.active_support.cache_format_version = 7.0
```

Или просто:

```ruby
# config/application.rb

config.load_defaults 7.0
```

Однако, приложения Rails 6.1 не способны прочитать этот новый формат сериализации, поэтому, чтобы обеспечить бесшовный апгрейд, нужно сперва задеплоить ваш обновленный Rails 7.0 с `config.active_support.cache_format_version = 6.1`, и только после того, как все процессы Rails были обновлены, можно установить `config.active_support.cache_format_version = 7.0`.

Rails 7.0 может прочитать оба формата, поэтому не нужно инвалидировать кэш во время апгрейда.

### Генерация изображения предварительного просмотра видео в ActiveStorage

Генерация изображения предварительного просмотра видео теперь использует обнаружение смены сцен FFmpeg для генерации более значимых предварительны изображений. До этого использовался первый кадр видео, и это вызывало проблемы, если видео постепенно появлялось из черного экрана. Это изменение требует FFmpeg v3.4+.

### Обработчик варианта по умолчанию Active Storage изменился на `:vips`

Для новых приложений трансформация изображения будет использовать libvips вместо ImageMagick. Это уменьшит время генерации вариантов, а также потребление CPU и памяти, улучшит время отклика в приложениях, полагающихся на active storage для раздачи своих изображений.

Опция `:mini_magick` не стала устаревшей, это нормально продолжать ее использование.

Чтобы мигрировать существующее приложение на libvips, установите:

```ruby
Rails.application.config.active_storage.variant_processor = :vips
```

Затем вам нужно изменить существующий код преобразования на макрос `image_processing` и заменить опции ImageMagick на опции libvips.

#### Замените resize на resize_to_limit

```diff
- variant(resize: "100x")
+ variant(resize_to_limit: [100, nil])
```

Если вы не сделаете это, при переключении на vips увидите эту ошибку: `no implicit conversion to float from string`.

#### Используйте массив при обрезке

```diff
- variant(crop: "1920x1080+0+0")
+ variant(crop: [0, 0, 1920, 1080])
```

Если вы не сделаете это, при переключении на vips увидите эту ошибку: `unable to call crop: you supplied 2 arguments, but operation needs 5`.

#### Пересмотрите значения обрезки:

Vips более строгий, чем ImageMagick, в отношение обрезки:

1. Он не обрежет, если `x` и/или `y` имеют отрицательные значения. Например: `[-10, -10, 100, 100]`
2. Он не обрежет, если (`x` или `y`) плюс размерность (`width`, `height`) больше, чем изображение. Например: изображение 125x125 и обрезка `[50, 50, 100, 100]`

Если вы не сделаете это, при переходе на vips увидите эту ошибку: `extract_area: bad extract area`

#### Исправьте фоновый цвет, используемый для `resize_and_pad`

Vips использует черный в качестве фонового цвета `resize_and_pad` по умолчанию, вместо белого в ImageMagick. Исправьте это с помощью опции `background`:

```diff
- variant(resize_and_pad: [300, 300])
+ variant(resize_and_pad: [300, 300, background: [255]])
```

#### Уберите любые ротации, основанные на EXIF

Vips будет осуществлять автоматическую ротацию с помощью значения EXIF при обработке вариантов. Если вы хранили значения ротации от загруженных пользователем изображений, чтобы применить ротацию в ImageMagick, это нужно перестать делать:

```diff
- variant(format: :jpg, rotate: rotation_value)
+ variant(format: :jpg)
```

#### Замените monochrome на colourspace
Vips другую опцию для создания монохромных изображений:

```diff
- variant(monochrome: true)
+ variant(colourspace: "b-w")
```

#### Переключитесь на опции libvips при сжатии изображений

JPEG

```diff
- variant(strip: true, quality: 80, interlace: "JPEG", sampling_factor: "4:2:0", colorspace: "sRGB")
+ variant(saver: { strip: true, quality: 80, interlace: true })
```

PNG

```diff
- variant(strip: true, quality: 75)
+ variant(saver: { strip: true, compression: 9 })
```

WEBP

```diff
- variant(strip: true, quality: 75, define: { webp: { lossless: false, alpha_quality: 85, thread_level: 1 } })
+ variant(saver: { strip: true, quality: 75, lossless: false, alpha_q: 85, reduction_effort: 6, smart_subsample: true })
```

GIF

```diff
- variant(layers: "Optimize")
+ variant(saver: { optimize_gif_frames: true, optimize_gif_transparency: true })
```

#### Деплой на production

Active Storage кодирует в url изображения список трансформаций, которые нужно выполнить. Если ваше приложение кэширует эти url, ваши изображения сломаются после деплоя нового кода на production. Поэтому вам нужно вручную инвалидировать затронутые ключи кэширования.

Например, Если у вас есть что-то наподобие этого во вью:

```erb
<% @products.each do |product| %>
  <% cache product do %>
    <%= image_tag product.cover_photo.variant(resize: "200x") %>
  <% end %>
<% end %>
```

Можно инвалидировать кэш, либо обновив product, или изменив ключ кэширования:

```erb
<% @products.each do |product| %>
  <% cache ["v2", product] do %>
    <%= image_tag product.cover_photo.variant(resize_to_limit: [200, nil]) %>
  <% end %>
<% end %>
```

(Upgrading from Rails 6.0 to Rails 6.1) Апгрейд с Rails 6.0 на Rails 6.1
------------------------------------------------------------------------

Подробнее о внесенных изменениях в Rails 6.1 смотрите в [заметках о релизе](/6_1_release_notes).

### Возвращаемое значение `Rails.application.config_for` больше не поддерживает доступ с помощью строковых ключей.

Допустим, у нас есть такой конфигурационный файл:

```yaml
# config/example.yml
development:
  options:
    key: value
```

```ruby
Rails.application.config_for(:example).options
```

Раньше это возвращало хэш, значения которого можно было получить с помощью строковых ключей. Это устарело в 6.0, а теперь вообще не работает.

Можно вызвать `with_indifferent_access` на возвращаемом значении `config_for`, если все еще хотите получать значения по строковым ключам, например:

```ruby
Rails.application.config_for(:example).with_indifferent_access.dig('options', 'key')
```

### Content-Type отклика при использовании `respond_to#any`

Заголовок Content-Type, возвращаемый в отклике, может отличаться от возвращаемого в Rails 6.0, особенно если приложение использует `respond_to { |format| format.any }`.
Теперь Content-Type будет основан на предоставленном блоке, а не формате запроса.

Example:

```ruby
def my_action
  respond_to do |format|
    format.any { render(json: { foo: 'bar' }) }
    end
  end
end
```

```ruby
get('my_action.csv')
```

Прежним поведением был возврат `text/csv` в Content-Type отклика, что является неверным, так как рендерили отклик JSON. Текущее поведение правильно возвращает `application/json` в Content-Type отклика.

Если ваше приложение полагалось на прежнее некорректное поведение, следует указать, какие форматы принимает ваш экшн, т.е.

```ruby
format.any(:xml, :json) { render request.format.to_sym => @people }
```

### `ActiveSupport::Callbacks#halted_callback_hook` теперь принимает второй аргумент

Active Support позволяет переопределить `halted_callback_hook` для вызова всякий раз, когда колбэк прерывает цепочку. Теперь этот метод принимает второй аргумент, являющийся именем прерываемого колбэка. Если у вас есть классы, переопределяющие этот метод, убедитесь, что он принимает два аргумента. Отметьте, что это значимое изменение без предшествующего цикла устаревания (по причинам быстродействия).

Пример:

```ruby
class Book < ApplicationRecord
  before_save { throw(:abort) }
  before_create { throw(:abort) }

  def halted_callback_hook(filter, callback_name) # => Теперь этот метод принимает 2 аргумента вместо 1
    Rails.logger.info("Book couldn't be #{callback_name}d")
  end
end
```

### Метод класса `helper` в контроллерах использует `String#constantize`

Концептуально, до Rails 6.1

```ruby
helper "foo/bar"
```

приводил к

```ruby
require_dependency "foo/bar_helper"
module_name = "foo/bar_helper".camelize
module_name.constantize
```

Вместо этого, теперь он делает это:

```ruby
prefix = "foo/bar".camelize
"#{prefix}Helper".constantize
```

Это изменение обратно совместимо с большинством приложений, в этом случае ничего делать не нужно.

Технически, однако, контроллеры могут настроить `helpers_path` на директорию в `$LOAD_PATH`, которая не в путях автозагрузки. Такой случай не поддерживается из коробки. Если модуль хелпера не загружается автоматически, приложение ответственно за его загрузку до вызова `helper`.

### Перенаправление к HTTPS от HTTP теперь будет использовать  308

Код статуса HTTP по умолчанию, используемый в `ActionDispatch::SSL` при перенаправлении не-GET/HEAD запросов от HTTP к HTTPS был изменен на `308`, как определено в https://tools.ietf.org/html/rfc7538.

### Active Storage теперь требует Image Processing

При обработке вариантов в Active Storage, теперь нужно иметь [гем image_processing](https://github.com/janko-m/image_processing) вместо непосредственного использования `mini_magick`. Image Processing настроен использовать `mini_magick` по умолчанию, поэтому проще всего для апгрейда будет заменить гем `mini_magick` на гем `image_processing`, и убедиться, что убрали явное использование `combine_options`, так как это больше не нужно.

Для читаемости можно изменить необработанные вызовы `resize` на макросы `image_processing`. Например, вместо:

```ruby
video.preview(resize: "100x100")
video.preview(resize: "100x100>")
video.preview(resize: "100x100^")
```

можно, соответственно, сделать:

```ruby
video.preview(resize_to_fit: [100, 100])
video.preview(resize_to_limit: [100, 100])
video.preview(resize_to_fill: [100, 100])
```

(Upgrading from Rails 5.2 to Rails 6.0) Апгрейд с Rails 5.2 на Rails 6.0
------------------------------------------------------------------------

Подробнее о внесенных изменениях в Rails 6.0 смотрите в [заметках о релизе](/6_0_release_notes).

### Использование Webpacker

[Webpacker](https://github.com/rails/webpacker) это компилятор JavaScript по умолчанию для Rails 6. Но если вы обновляете приложение, он не активирован по умолчанию. Если хотите использовать Webpacker, включите его в Gemfile и установите:

```ruby
gem "webpacker"
```

```bash
$ bin/rails webpacker:install
```

### Навязывание SSL

Метод `force_ssl` для контроллеров устарел и будет убран в Rails 6.1. Рекомендуется включить `config.force_ssl` для обеспечения подключения HTTPS во всем приложении. Если необходимо освободить определенные конечные точки от перенаправления, можно использовать `config.ssl_options` для конфигурирования этого поведения.

### Для увеличения безопасности, метаданные о назначении и истечении теперь встроены в подписанные и зашифрованные куки

Чтобы улучшить безопасность, Rails встраивает метаданные о назначении и истечении внутри зашифрованного или подписанного значения куки.

Затем Rails может помешать исполнению атак, пытающихся скопировать подписанное/зашифрованное значение куки и использовать его как значение другого куки.

Эти новые встроенные метаданные делает куки несовместимыми с версиями Rails старше чем 6.0.

Если необходимо, чтобы куки читались Rails 5.2 и старше, или вы все еще проверяете деплой 6.0 и хотите возможность отката, установите `Rails.application.config.action_dispatch.use_cookies_with_metadata` в `false`.

### Все пакеты npm были перемещены в пространство имен `@rails`

Если вы раньше загружали любые из пакетов `actioncable`, `activestorage` или `rails-ujs` с помощью npm/yarn, вам нужно обновить имена этих зависимостей до их обновления до `6.0.0`:

```
actioncable   → @rails/actioncable
activestorage → @rails/activestorage
rails-ujs     → @rails/ujs
```

### Изменения Action Cable JavaScript API

Пакет Action Cable JavaScript был конвертирован из CoffeeScript в ES2015, и исходный код теперь опубликован в дистрибуции npm.

Этот релиз включает некоторые переломные изменения опциональных частей Action Cable JavaScript API:

- Настройки адаптера WebSocket и адаптера логгера были перемещены из свойств `ActionCable` в свойства `ActionCable.adapters`. Если вы настраивали эти адаптеры, необходимы следующие изменения:

    ```diff
    -    ActionCable.WebSocket = MyWebSocket
    +    ActionCable.adapters.WebSocket = MyWebSocket
    ```
    ```diff
    -    ActionCable.logger = myLogger
    +    ActionCable.adapters.logger = myLogger
    ```

- Методы `ActionCable.startDebugging()` и `ActionCable.stopDebugging()` были убраны и заменены свойством `ActionCable.logger.enabled`. Если вы использовали эти методы, необходимы следующие изменения:

    ```diff
    -    ActionCable.startDebugging()
    +    ActionCable.logger.enabled = true
    ```
    ```diff
    -    ActionCable.stopDebugging()
    +    ActionCable.logger.enabled = false
    ```

### `ActionDispatch::Response#content_type` теперь возвращает заголовок Content-Type без изменений

Ранее, возвращаемое значение `ActionDispatch::Response#content_type` НЕ содержало часть charset. Это поведение было изменено, чтобы также возвращать ранее опускаемую часть charset.

Если необходим только тип MIME, используйте вместо этого `ActionDispatch::Response#media_type`.

До:

```ruby
resp = ActionDispatch::Response.new(200, "Content-Type" => "text/csv; header=present; charset=utf-16")
resp.content_type #=> "text/csv; header=present"
```

После:

```ruby
resp = ActionDispatch::Response.new(200, "Content-Type" => "text/csv; header=present; charset=utf-16")
resp.content_type #=> "text/csv; header=present; charset=utf-16"
resp.media_type   #=> "text/csv"
```

### (autoloading) Автозагрузка

Конфигурация по умолчанию для Rails 6

```ruby
# config/application.rb

config.load_defaults 6.0
```

включает режим автозагрузки `zeitwerk` на CRuby. В этом режиме автозагрузка, перезагрузка и нетерпеливая загрузка управляются [Zeitwerk](https://github.com/fxn/zeitwerk).

Если вы используете умолчания из предыдущей версии Rails, можно включить zeitwerk так:

```ruby
# config/application.rb

config.autoloader = :zeitwerk
```

#### Публичный API

В целом, приложениям не нужно использовать API Zeitwerk напрямую. Rails настраивает его в соответствии с существующими контрактами: `config.autoload_paths`, `config.cache_classes` и т.д.

Хотя приложения должны придерживаться этого интерфейса, фактический объект загрузки Zeitwerk доступен как

```ruby
Rails.autoloaders.main
```

Это может быть удобным, к примеру, если необходимо предварительно загрузить наследование с единой таблицей (Single Table Inheritance, STI) или настроить пользовательский инфлектор.

#### Структура проекта

Если в приложение автозагрузки были обновлены корректно, структура проекта должна быть, в основном, совместимой.

Однако, режим `classic` производит имена файлов из имен отсутствующих констант (`underscore`), в то время как режим `zeitwerk` производит имена констант из имен файлов (`camelize`). Эти хелперы не всегда противоположны друг другу, в частности для сокращений. Например, `"FOO".underscore` это `"foo"`, но `"foo".camelize` это `"Foo"`, а не `"FOO"`.

Совместимость можно проверить с помощью задачи `zeitwerk:check`:

```bash
$ bin/rails zeitwerk:check
Hold on, I am eager loading the application.
All is good!
```

#### require_dependency

Все известные случаи применений `require_dependency` были устранены, следует найти и удалить их во всем проекте.

Если в вашем приложении есть Single Table Inheritance, посмотрите на [раздел о наследовании с единой таблицей](/constant_autoloading_and_reloading#single-table-inheritance) руководства Автозагрузка и перезагрузка констант (режим Zeitwerk).

#### Ограниченные имена в определениях класса и модуля

Теперь можно без проблем использовать пути констант в определениях класса и модуля:

```ruby
# Автозагрузка в теле этого класса теперь соответствует семантике Ruby.
class Admin::UsersController < ApplicationController
  # ...
end
```

Особенность, о которой нужно знать, в том, что классический автозагрузчик мог иногда автоматически загружать `Foo::Wadus` в

```ruby
class Foo::Bar
  Wadus
end
```

Это не соответствует семантике Ruby, так как `Foo` не во вложенности, и это вообще не будет работать в режиме `zeitwerk`. Если вы найдете такие частные случаи, можете использовать ограниченное имя `Foo::Wadus`:

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

#### Концерны

Можно автоматически загружать и нетерпеливо загружать их стандартной структуры, наподобие

```
app/models
app/models/concerns
```

В этом случае, `app/models/concerns` полагается корневой директорией (так как она принадлежит путям автозагрузки), и будет игнорироваться в качестве пространства имен. Поэтому, `app/models/concerns/foo.rb` должен определять `Foo`, а не `Concerns::Foo`.

Пространство имен `Concerns::` работало с классическим автозагрузчиком как побочный эффект реализации, но это никогда не было желаемым поведением. Приложение, использующее `Concerns::`, нуждается в переименовании этих классов и модулей, чтобы их можно было использовать в режиме `zeitwerk`.

#### `app` в путях автоматической загрузки

В некоторых проектах когда мы хотели что-то вроде `app/api/base.rb` для определения `API::Base`, то добавляли `app` в пути автозагрузки, для того, чтобы это работало в режиме `classic`. С тех пор, как Rails автоматически добавляет все поддиректории `app` в пути автозагрузки, у нас теперь другая ситуация, в которой есть вложенные корневые директории, поэтому такая настройка больше не работает. Похожий принцип мы объяснили выше про `concerns`.

Если хотите сохранить эту структуру, необходимо удалить поддиректорию из путей автозагрузки в инициализаторе:

```ruby
ActiveSupport::Dependencies.autoload_paths.delete("#{Rails.root}/app/api")
```

#### Автоматическая загрузка констант и явные пространства имен

Если пространство имен определено в файле, как `Hotel` тут:

```
app/models/hotel.rb         # Defines Hotel.
app/models/hotel/pricing.rb # Defines Hotel::Pricing.
```

константа `Hotel` должна быть установлена с помощью ключевых слов `class` или `module`. Например:

```ruby
class Hotel
end
```

это хорошо.

Альтернативы, наподобие

```ruby
Hotel = Class.new
```

или

```ruby
Hotel = Struct.new
```

не будут работать, дочерние объекты, такие как `Hotel::Pricing`, не будут найдены.

Это ограничение применяется только к явным пространствам имен. Классы и модули, не определяющие пространство имен, могут быть определены с помощью этих идиом.

#### Один файл, одна константа (на том же уровне)

В режиме `classic` технически было возможно определить несколько констант на том же уровне, и они все перезагружались. Например, для

```ruby
# app/models/foo.rb

class Foo
end

class Bar
end
```

хотя `Bar` не мог быть автоматически загружен, автозагрузка `Foo` также помечала `Bar` как автоматически загруженный. Это не так в режиме `zeitwerk`, необходимо переместить `Bar` в собственный файл `bar.rb`. Один файл, одна константа.

Это влияет только на константы на том же уровне, как в вышеописанном примере. Внутренние классы и модули — это нормально. Например, рассмотрим

```ruby
# app/models/foo.rb

class Foo
  class InnerClass
  end
end
```

Если приложение перезагрузит `Foo`, оно также перезагрузит `Foo::InnerClass`.

#### Spring и среда `test`

Spring перезагружает код приложения, если что-то меняется. В среде `test` нужно включить перезагрузку, чтобы это работало:

```ruby
# config/environments/test.rb

config.cache_classes = false
```

Иначе, вы получите такую ошибку:

```
reloading is disabled because config.cache_classes is true
```

#### Bootsnap

Bootsnap должен быть как минимум версии 1.4.2.

Помимо этого, для Bootsnap необходимо отключить кэш iseq из-за ошибки в интерпретаторе, если запускается Ruby 2.5. Убедитесь, что зависимы от минимум Bootsnap 1.4.4 в таком случае.

#### `config.add_autoload_paths_to_load_path`

Новый конфигурационный пункт

```ruby
config.add_autoload_paths_to_load_path
```

по умолчанию `true` для обратной совместимости, но позволяет уйти от добавления путей автозагрузки в `$LOAD_PATH`.

Это имеет смысл в большинстве приложений, так как никогда не следует требовать файл в `app/models`, к примеру, и Zeitwerk использует только абсолютные имена файлов.

Уходя от этого, вы оптимизируете поиск `$LOAD_PATH` (меньше директорий для проверки), и экономите работу Bootsnap и потребление памяти, поскольку ему не нужно строить индекс для этих директорий.

#### Тредобезопасность

В классическом режиме автоматическая загрузка констант не является тредобезопасной, хотя в Rails есть локальные блокировки, например, чтобы сделать веб запросы тредобезопасными, когда включена автоматическая загрузка, так как это принято в среде development.

Автоматическая загрузка констант является тредобезопасной в режиме `zeitwerk`. Например, теперь можно автоматически загружать в многотредовых скриптах, запускаемых с помощью команды `runner`.

#### Шаблоны поиска в config.autoload_paths

Остерегайтесь конфигураций, таких как

```ruby
config.autoload_paths += Dir["#{config.root}/lib/**/"]
```

Каждый элемент `config.autoload_paths` должен представлять пространство имен верхнего уровня (`Object`) и они не могут быть последовательно вложенными (с исключением директорий `concerns`, описанных выше).

Чтобы это починить, просто уберите подстановки:

```ruby
config.autoload_paths << "#{config.root}/lib"
```

#### Нетерпеливая загрузка и автоматическая загрузка согласуются

В режиме `classic`, если `app/models/foo.rb` определяет `Bar`, вы не сможете автоматически загрузить этот файл, но нетерпеливая загрузка будет работать, так как она загружает файлы вслепую. Это может быть источником ошибок, если вы сначала тестируете с помощью нетерпеливой загрузки, потом выполнение может сломаться при автоматической загрузке.

В режиме `zeitwerk` оба режима загрузки согласуются, они выдадут ошибки в тех же файлах.

#### Как использовать классический автозагрузчик в Rails 6

Приложения могут загружать умолчания Rails 6 и все еще использовать классический автозагрузчик, настроив `config.autoloader` следующим образом:

```ruby
# config/application.rb

config.load_defaults 6.0
config.autoloader = :classic
```

При использовании Classic Autoloader в приложении Rails 6 рекомендовано установить уровень concurrency на 1 в среде development, для веб-серверов и фоновых обработчиков, по причинам тредобезопасности.

### Изменение поведения присвоения Active Storage

С настройками по умолчанию для Rails 5.2, присвоение к коллекции вложений, объявленное с помощью `has_many_attached`, добавляет новые файлы:

```ruby
class User < ApplicationRecord
  has_many_attached :highlights
end

user.highlights.attach(filename: "funky.jpg", ...)
user.highlights.count # => 1

blob = ActiveStorage::Blob.create_after_upload!(filename: "town.jpg", ...)
user.update!(highlights: [ blob ])

user.highlights.count # => 2
user.highlights.first.filename # => "funky.jpg"
user.highlights.second.filename # => "town.jpg"
```

С настройками по умолчанию для Rails 6.0, присвоение к коллекции вложений заменяет существующие файлы вместо добавления к ним. Это соответствует поведению Active Record при присвоении к связанной коллекции:

```ruby
user.highlights.attach(filename: "funky.jpg", ...)
user.highlights.count # => 1

blob = ActiveStorage::Blob.create_after_upload!(filename: "town.jpg", ...)
user.update!(highlights: [ blob ])

user.highlights.count # => 1
user.highlights.first.filename # => "town.jpg"
```

Для добавления новых вложений вместо удаления существующих можно использовать `#attach`:

```ruby
blob = ActiveStorage::Blob.create_after_upload!(filename: "town.jpg", ...)
user.highlights.attach(blob)

user.highlights.count # => 2
user.highlights.first.filename # => "funky.jpg"
user.highlights.second.filename # => "town.jpg"
```

Существующие приложения могут включить это новое поведение установив `config.active_storage.replace_on_assign_to_many` в `true`. Старое поведение будет устаревшим в Rails 7.0 и убрано в Rails 7.1.

(Upgrading from Rails 5.1 to Rails 5.2) Апгрейд с Rails 5.1 на Rails 5.2
------------------------------------------------------------------------

Подробнее о внесенных изменениях в Rails 5.2 смотрите в [заметках о релизе](/5_2_release_notes).

### Bootsnap

Rails 5.2 добавляет гем bootsnap во [вновь сгенерированный Gemfile приложения](https://github.com/rails/rails/pull/29313).
Команда `app:update` устанавливает его в `boot.rb`. Если необходимо использовать его, добавьте в Gemfile, иначе измените `boot.rb`, чтобы не использовать bootsnap.

### Истечение срока действия в подписанных или зашифрованных куки теперь встроено в значения куки

Чтобы повысить безопасность, Rails теперь встраивает информацию об истечении срока действия также в зашифрованное или подписанное значение куки.

Эта новая встроенная информация делает эти куки несовместимыми с версиями Rails до 5.2.

Если необходимо, чтобы куки читались в версии 5.1 и раньше, или если все еще проверяется деплой 5.2 и необходимо оставить возможность сделать откат, установите `Rails.application.config.action_dispatch.use_authenticated_cookie_encryption` в `false`.

(Upgrading from Rails 5.0 to Rails 5.1) Апгрейд с Rails 5.0 на Rails 5.1
------------------------------------------------------------------------

Подробнее о внесенных изменениях в Rails 5.1 смотрите в [заметках о релизе](/5_1_release_notes).

### Верхнеуровневый `HashWithIndifferentAccess` в скором времени устареет.

Если ваше приложение использует верхнеуровневый класс `HashWithIndifferentAccess`, то вам следует вместо него постепенно переходить на `ActiveSupport::HashWithIndifferentAccess`.

Это всего лишь постепенное устаревание, которое означает, что ваш код не будет прерываться в данный момент и не будет отображаться предостережение об устаревании, но эта константа в будущем будет удалена.

Кроме того, если имеются довольно старые документы YAML, содержащие выгрузки таких объектов, может понадобиться загрузить и выгрузить их снова, чтобы убедиться, что они ссылаются на нужную константу и что их загрузка не будет прерываться в будущем.

### `application.secrets` теперь загружается со всеми ключами в качестве символов

Если ваше приложение хранит вложенную конфигурацию в `config/secrets.yml`, все ключи теперь загружаются как символы, поэтому доступ с использованием строк должен быть изменен.

С:

```ruby
Rails.application.secrets[:smtp_settings]["address"]
```

На:

```ruby
Rails.application.secrets[:smtp_settings][:address]
```

### Убрана устаревшая поддержка `:text` и `:nothing` в `render`

Если ваши контроллеры используют `render :text`, они перестанут работать. Новый способ рендерить вью с типом MIME `text/plain` — использовать `render :plain`.

Похожим образом, `render :nothing` также убран, и следует использовать метод `head` для отправки откликов, содержащих только заголовки. Например, `head :ok` посылает отклик 200 без тела.

### Убрана устаревшая поддержка `redirect_to :back`

В Rails 5.0, `redirect_to :back` устарел. В Rails 5.1 он был убран полностью.

В качестве альтернативы используйте `redirect_back`. Важно отметить, что `redirect_back` также принимает опцию `fallback_location`, которая будет использована в случае отсутствия `HTTP_REFERER`.

```
redirect_back(fallback_location: root_path)
```

(Upgrading from Rails 4.2 to Rails 5.0) Апгрейд с Rails 4.2 на Rails 5.0
------------------------------------------------------------------------

Подробнее о внесенных изменениях в Rails 5.0 смотрите в [заметках о релизе](/5_0_release_notes).

### Требуется Ruby 2.2.2+

Начиная с Ruby on Rails 5.0, Ruby 2.2.2+ являются единственными поддерживаемыми версиями Ruby. Перед тем, как продолжить, убедитесь, что вы на Ruby версии 2.2.2 или выше.

### Модели Active Record теперь по умолчанию наследуются от ApplicationRecord

В Rails 4.2 модель Active Record наследуется от `ActiveRecord::Base`. В Rails 5.0 все модели наследуются от `ApplicationRecord`.

`ApplicationRecord` - это новый суперкласс для всех моделей приложения, аналогично контроллерам, наследуемым от `ApplicationController` вместо `ActionController::Base`. Это дает приложению единое место для настройки специфичного для приложения поведения моделей.

При апгрейде с Rails 4.2 на Rails 5.0 необходимо создать файл `application_record.rb` в `app/models/` и добавить следующее содержимое:

```ruby
class ApplicationRecord < ActiveRecord::Base
  self.abstract_class = true
end
```

Затем убедитесь, что все ваши модели наследуются от него.

### Прерывание цепочек колбэков с помощью `throw(:abort)`

В Rails 4.2 в Active Record и Active Model, когда колбэк 'before' возвращает `false`, вся цепочка цепочка колбэков прерывалась. Другими словами, последующие колбэки 'before' не выполнялись, как и экшн, обернутый в колбэки.

В Rails 5.0, возврат
