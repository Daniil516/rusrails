Кэширование с Rails: Обзор
==========================

Это руководство является введением в ускорение вашего приложения Rails с помощью кэширования.

Кэширование означает хранение контента, генерируемого в цикле запрос-отклик, и повторное использование его при ответе на подобные запросы.

Кэширование часто является самым эффективным способом повысить производительность приложения. При помощи кэширования, веб-сайты, работающие на одном сервере с одной базой данных, могут выдержать нагрузку в несколько десятков тысяч конкурентных пользователей.

Rails предоставляет набор функций кэширования из коробки. Это руководство научит вас областям кэширования и целям каждой области. Освойте эти приемы и ваши Rails приложения смогут обслужить миллионы просмотров без запредельного времени отклика сервера или счетов за сервер.

После прочтения этого руководства, вы узнаете:

* О кэшировании фрагмента и кэшировании матрешкой (Russian doll caching).
* Как управлять зависимостями кэширования.
* Об альтернативных хранилищах кэша.
* Об условной поддержке GET.

Основы кэширования
------------------

Это введение в три типа техники кэширования: кэширование страницы, экшна и фрагмента. По умолчанию Rails предоставляет кэширование фрагмента. Чтобы использовать кэширование страницы и экшна, нужно добавить `actionpack-page_caching` и `actionpack-action_caching` в свой `Gemfile`.

По умолчанию кэширование включено только в среде production. Можно поиграть с кэшированием локально, запустив `rails dev:cache`, или установив [`config.action_controller.perform_caching`][] `true` в `config/environments/development.rb`.

NOTE: Изменение значения `config.action_controller.perform_caching` повлияет только на кэширование, предоставленное Action Controller. Например, это не повлияет на низкоуровневое кэширование, которое мы рассмотрим [ниже](#low-level-caching).

[`config.action_controller.perform_caching`]: /configuring#config-action-controller-perform-caching

### Кэширование страницы

Кэширование страницы это механизм Rails, позволяющий запросу на сгенерированную страницу быть полностью обслуженным веб сервером (т.е. Apache или NGINX) в принципе, без прохождения через весь стек Rails. Хотя это и очень быстро, но не может быть применено к каждой ситуации (например, к страницам, требующим аутентификации). А также, раз веб сервер получает файл напрямую из файловой системы, необходимо реализовать прекращение кэша.

INFO: Кэширование страниц было убрано из Rails 4. Обратитесь к [гему actionpack-page_caching](https://github.com/rails/actionpack-page_caching).

### Кэширование экшна

Кэширование страниц нельзя использовать для экшнов, имеющих предварительные фильтры, - например, для страниц, требующих аутентификации. И тут на помощь приходит кэширование экшна. Кэширование экшна работает как кэширование страницы, за исключением того, что входящий веб-запрос затрагивает стек Rails, таким образом, до обслуживания кэша могут быть запущены предварительные (before) фильтры. Это позволит использовать аутентификацию и другие ограничения, и в то же время выводит результат из кэшированной копии.

INFO: Кэширование экшна было убрано из Rails 4. Обратитесь к [гему actionpack-action_caching](https://github.com/rails/actionpack-action_caching). Также взгляните на статью [DHH по прекращению кэша, основанного на ключе](https://signalvnoise.com/posts/3113-how-key-based-cache-expiration-works), как более предпочтительного способа.

### Кэширование фрагмента

Динамические веб-приложения обычно создают страницы с рядом компонентов, не все из которых имеют сходные характеристики кэширования. Когда различные части страниц нуждаются в кэшировании и прекращаются по-разному, вы можете использовать Кэширование фрагмента.

Кэширование фрагмента позволяет фрагменту логики вью быть обернутым в блок кэша и обслуженным из хранилища кэша для последующего запроса.

Например, если хотите кэшировать каждый продукт на странице, можно использовать этот код:

```erb
<% @products.each do |product| %>
  <% cache product do %>
    <%= render product %>
  <% end %>
<% end %>
```

Когда приложение получит самый первый запрос на эту страницу, Rails запишет новую закэшированную запись с уникальным ключом. Ключ может выглядеть так:

```
views/products/index:bea67108094918eeba42cd4a6e786901/products/1
```

Строка символов в конце ключа является дайджестом дерева шаблона. Это дайджест хэша (hash digest), вычисленного на основе содержимого фрагмента вью, которую вы кэшируете. Если вы измените фрагмент вью (например, поменяете HTML), дайджест хэша изменится, прекращая существующий кэш.

Версия кэша, произведенная от версии product, хранится в записи кэша. Когда product обновляется, версия кэша меняется, и любые закэшированные фрагменты, содержащие предыдущую версию, игнорируются.

TIP: Хранилища кэша, такие как Memcached, автоматически удалят старые файлы с кэшем.

Если хотите кэшировать фрагмент по определенным условиям, можно использовать `cache_if` or `cache_unless`:

```erb
<% cache_if admin?, product do %>
  <%= render product %>
<% end %>
```

#### Кэширование коллекции

Хелпер `render` может также кэшировать отдельные шаблоны, отображающие коллекцию. В рассмотренном ранее примере с `each` можно считать все кэши шаблонов за один раз, а не по одному. Это делается передавая `cached: true` при рендеринге коллекции:

```html+erb
<%= render partial: 'products/product', collection: @products, cached: true %>
```

Все закэшированные шаблоны из предыдущих отображений будут считаны за один раз с гораздо большей скоростью. Помимо этого, те шаблоны, что еще не были закэшированы, будут записаны в кэш и извлечены при следующем рендеринге.

### Кэширование матрешкой

Можно вкладывать кэшированные фрагменты в другие кэшированные фрагменты. Это называется кэшированием матрешкой.

Преимуществом кэширования матрешкой является то, что если обновляется отдельный продукт, другие внутренние фрагменты могут быть повторно использованы при регенерации внешнего фрагмента.

Как объяснялось в предыдущем разделе, кэш будет прекращен, если изменится значение `updated_at` для записи, от которой напрямую зависит этот кэш. Однако, это не прекратит любой кэш, в который вложен этот фрагмент.

Например, возьмем следующую вью:

```erb
<% cache product do %>
  <%= render product.games %>
<% end %>
```

Которая, в свою очередь, рендерит эту вью:

```erb
<% cache game do %>
  <%= render game %>
<% end %>
```

Если изменится любой атрибут game, у значения `updated_at` будет установлено текущее время, тем самым прекращая. Однако, так как `updated_at` не изменится для объекта product, этот кэш не будет прекращен и ваше приложение отдаст устаревшие данные. Чтобы это починить, мы свяжем модели вместе с помощью метода `touch`:

```ruby
class Product < ApplicationRecord
  has_many :games
end

class Game < ApplicationRecord
  belongs_to :product, touch: true
end
```

С помощью `touch`, установленного в `true`, любой экшн, изменяющий `updated_at` для записи game, будет также изменять его для связанного product, тем самым прекращая кэш.

### Кэширование общих партиалов

Существует возможность делиться партиалами и связанным кэшированием между файлами с разными типами MIME. Например, кэширование общих партиалов позволяет разработчикам шаблонов делить партиал между файлами HTML и JavaScript. Когда шаблоны собираются в шаблонном распознавателе путей файла, они включают только расширение языка шаблона и не включают тип MIME. Из-за этого шаблоны можно использовать для нескольких типов MIME. Оба запроса, HTML и JavaScript, будут отвечать на следующий код:

```ruby
render(partial: 'hotels/hotel', collection: @hotels, cached: true)
```

Будет загружен файл с именем `hotels/hotel.erb`.

Другим вариантом является включение полного имени файла партиала для рендеринга.

```ruby
render(partial: 'hotels/hotel.html.erb', collection: @hotels, cached: true)
```

Будет загружен файл с именем `hotels/hotel.html.erb` в любом типе файла MIME, например, можно включить этот партиал в файл JavaScript.

### Управление зависимостями

Для того, чтобы правильно инвалидировать кэш, вам необходимо правильно определить зависимости кэширования. Rails достаточно умен, чтобы справиться с общими случаями так, что вы не должны будете ничего указывать. Однако, иногда, когда вы имеете дело с нестандартными хелперами например, вы должны будете явно определить их.

### Неявные зависимости

Большинство зависимостей шаблонов могут быть вычислены из вызовов `render` в самом шаблоне. Вот несколько примеров вызовов `render`, которые `ActionView::Digestor` знает как понять:

```ruby
render partial: "comments/comment", collection: commentable.comments
render "comments/comments"
render 'comments/comments'
render('comments/comments')

render "header" переводится в render("comments/header")

render(@topic)         переводится в render("topics/topic")
render(topics)         переводится в render("topics/topic")
render(message.topics) переводится в render("topics/topic")
```

С другой стороны, некоторые вызовы нужно изменить, чтобы кэширование работало верно. Например, если вы передаете нестандартную коллекцию, вам нужно изменить:

```ruby
render @project.documents.where(published: true)
```

на:

```ruby
render partial: "documents/document", collection: @project.documents.where(published: true)
```

### Явные зависимости

Иногда у вас будут зависимости шаблонов, которые не получается определить совсем. Это обычно для ситуаций, когда отображение происходит в хелперах. Вот пример:

```html+erb
<%= render_sortable_todolists @project.todolists %>
```

Вам необходимо использовать специальный формат комментариев для вызова их извне:

```html+erb
<%# Template Dependency: todolists/todolist %>
<%= render_sortable_todolists @project.todolists %>
```

В некоторых случаях, например, при установке наследования с единой таблицей (STI), вы можете иметь кучу явных зависимостей. Вместо написания каждого шаблона, вы можете использовать знак звездочку, чтобы подходил любой шаблон из каталога:

```html+erb
<%# Template Dependency: events/* %>
<%= render_categorizable_events @person.events %>
```

Как и для кэширования коллекций, если партиал начинается не с явного вызова кэша, вы все-таки можете извлечь выгоду кэширования коллекций, добавив специальный формат комментария в любом месте шаблона, наподобие:

```html+erb
<%# Template Collection: notification %>
<% my_helper_that_calls_cache(some_arg, notification) do %>
  <%= notification.name %>
<% end %>
```

### Внешние зависимости

Если вы используете метод хелпера, например, внутри кэшируемого блока, и затем обновляете хелпер, вам также нужно будет удалить кэш. Неважно как вы сделаете это, но MD5 файла шаблона должен измениться. Одна из рекомендаций, явно указать в комментарии, наподобие:

```html+erb
<%# Helper Dependency Updated: Jul 28, 2015 at 7pm %>
<%= some_helper_method(person) %>
```

### (low-level-caching) Низкоуровневое кэширование

Иногда хочется закэшировать определенное значение или результат запроса вместо кэширования фрагментов вью. Механизм кэширования Rails отлично работает для хранения информации любого рода.

Наиболее эффективным способом реализации низкоуровневого кэширования является использование метода `Rails.cache.fetch`. Этот метод и читает, и пишет в кэш. Если передать только один аргумент, этот ключ извлекается и возвращается значение из кэша. Если передан блок, этот блок будет выполнен в случае отсутствия кэша. Возвращаемое значение блока будет записано в кэш под заданным ключом кэша, и это возвращаемое значение будет возвращено. В случае наличия кэша, будет возвращено закэшированное значение без выполнения блока.

Рассмотрим следующий пример. В приложении есть модель `Product` с методом экземпляра, ищущим цену продукта на конкурирующем сайте. Данные, возвращаемые этим методом отлично подходят для низкоуровневого кэширования:

```ruby
class Product < ApplicationRecord
  def competing_price
    Rails.cache.fetch("#{cache_key_with_version}/competing_price", expires_in: 12.hours) do
      Competitor::API.find_price(id)
    end
  end
end
```

NOTE: Отметьте, что в этом пример мы использовали метод `cache_key_with_version`, таким образом результирующий ключ кэша будет выглядеть наподобие `products/233-20140225082222765838000/competing_price`. `cache_key_with_version` генерирует строку на основе имени класса модели и атрибутов `id` и `updated_at`. Это обычное соглашение, имеющее преимущество невалидности кэша, когда изменяется продукт. В основном при использовании низкоуровневого кэширования необходимо генерировать ключ кэша.

#### Избегайте кэширование экземпляров объектов Active Record

Рассмотрим пример, хранящий в кэше список объектов Active Record, представляющий супер-пользователям:

```ruby
# super_admins is an expensive SQL query, so don't run it too often
Rails.cache.fetch("super_admin_users", expires_in: 12.hours) do
  User.super_admins.to_a
end
```

Следует __избегать__ этот паттерн. Почему? Потому, что экземпляры могут измениться. В production его атрибуты могут отличаться, или запись быть удалена. И в development он работает ненадежно с хранилищами кэша, перезагружающих код при изменении кода вами.

Вместо этого кэшируйте ID или некоторые другие примитивные типы данных. Например:

```ruby
# super_admins is an expensive SQL query, so don't run it too often
ids = Rails.cache.fetch("super_admin_user_ids", expires_in: 12.hours) do
  User.super_admins.pluck(:id)
end
User.where(id: ids).to_a
```

### Кэширование SQL

Кэширование запроса (query) - это особенность Rails, кэширующая результат выборки по каждому запросу (query). Если Rails встречает тот же запрос (query) на протяжении текущего запроса (request), он использует кэшированный результат, вместо того, чтобы снова сделать запрос (query) к базе данных.

Например:

```ruby
class ProductsController < ApplicationController

  def index
    # Запускаем поисковый запрос
    @products = Product.all

    # ...

    # Снова запускаем тот же запрос
    @products = Product.all
  end

end
```

Когда тот же запрос будет сделан, фактически он не дойдет до базы данных. В первый раз возвращенный результат запроса сохраняется в кэше запроса (в памяти), а во второй раз он извлекается из памяти.

Однако, важно отметить, что кэши запросов создаются в начале экшна и уничтожаются в конце того же экшна, тем самым являются персистентными только на протяжении этого экшна. Если необходимо хранить результаты запроса в более персистентной форме, можно использовать низкоуровневое кэширование.

(cache-stores) Хранилища кэша
-----------------------------

Rails предоставляет различные хранилища для кэшированных данных (кроме SQL кэширования и кэширования страниц).

### Конфигурация

Можно настроить хранилище кэша по умолчанию своего приложения, установив конфигурационную опцию `config.cache_store`. Другие параметры могут будут переданы как аргументы в конструктор хранилища кэша.

```ruby
config.cache_store = :memory_store, { size: 64.megabytes }
```

Альтернативно можно вызвать `ActionController::Base.cache_store` вне конфигурационного блока.

К кэшу можно получить доступ, вызвав `Rails.cache`.

#### Опции пула соединений

По умолчанию [`:mem_cache_store`](#activesupport-cache-memcachestore) и [`:redis_cache_store`](#activesupport-cache-rediscachestore) настроены использовать пул соединений. Это означает, что при использовании Puma или другого сервера на тредах, можно использовать несколько тредов, выполняющих запросы к хранилищу кэша в то же самое время.

Если хотите отключить пул соединений, установите опции `:pool` `false` при конфигурировании хранилища кэша:

```ruby
config.cache_store = :mem_cache_store, "cache.example.com", pool: false
```

Также можно переопределить настройки пула по умолчанию, предоставив индивидуальные опции к опции `:pool`:

```ruby
config.cache_store = :mem_cache_store, "cache.example.com", pool: { size: 32, timeout: 1 }
```

* `:size` - Эта опция устанавливает количество соединений на процесс (по умолчанию 5).

* `:timeout` - Эта опция устанавливает количество секунд ожидания соединения (по умолчанию 5). Если не было доступного соединения в течение таймаута, будет вызвана `Timeout::Error`.

### ActiveSupport::Cache::Store

[`ActiveSupport::Cache::Store`][] представляет основу для взаимодействия с кэшем в Rails. Это абстрактный класс, и он сам не может быть использован. Вместо этого нужно использовать конкретную реализацию класса, связанного с engine-ом хранилища. Rails поставляется с несколькими реализациями, описанными ниже.

Главные вызываемые методы это [`read`][ActiveSupport::Cache::Store#read], [`write`][ActiveSupport::Cache::Store#write], [`delete`][ActiveSupport::Cache::Store#delete], [`exist?`][ActiveSupport::Cache::Store#exist?] и [`fetch`][ActiveSupport::Cache::Store#fetch].

Опции, переданные в конструктор хранилища кэша, будут трактованы как опции по умолчанию для соответствующих методов API.

[`ActiveSupport::Cache::Store`]: https://api.rubyonrails.org/classes/ActiveSupport/Cache/Store.html
[ActiveSupport::Cache::Store#delete]: https://api.rubyonrails.org/classes/ActiveSupport/Cache/Store.html#method-i-delete
[ActiveSupport::Cache::Store#exist?]: https://api.rubyonrails.org/classes/ActiveSupport/Cache/Store.html#method-i-exist-3F
[ActiveSupport::Cache::Store#fetch]: https://api.rubyonrails.org/classes/ActiveSupport/Cache/Store.html#method-i-fetch
[ActiveSupport::Cache::Store#read]: https://api.rubyonrails.org/classes/ActiveSupport/Cache/Store.html#method-i-read
[ActiveSupport::Cache::Store#write]: https://api.rubyonrails.org/classes/ActiveSupport/Cache/Store.html#method-i-write

### `ActiveSupport::Cache::MemoryStore`

[`ActiveSupport::Cache::MemoryStore`][] хранит записи в памяти в том же процессе Ruby. У хранилища кэша ограниченный размер, определенный опцией `:size`, указанной в инициализаторе (по умолчанию 32Mb). Когда кэш превышает выделенный размер, происходит очистка и самые ранние используемые записи будут убраны.

```ruby
config.cache_store = :memory_store, { size: 64.megabytes }
```

Если запущено несколько серверных процессов Ruby on Rails (что бывает в случае использования Phusion Passenger или puma в кластерном режиме), то экземпляры ваших серверов Rails не смогут разделять данные кэша друг с другом. Это хранилище кэша не подходит для больших приложений. Однако, оно замечательно работает с небольшими сайтами с низким трафиком, с несколькими серверными процессами, или для сред development и test.

Новые проекты Rails настроены для использования этой реализации в development среде по умолчанию.

NOTE: Поскольку процессы не делятся данными кэша при использовании `:memory_store`, то невозможно вручную считывать, записывать или очищать кэш через консоль Rails.

[`ActiveSupport::Cache::MemoryStore`]: https://api.rubyonrails.org/classes/ActiveSupport/Cache/MemoryStore.html

### `ActiveSupport::Cache::FileStore`

[`ActiveSupport::Cache::FileStore`][] использует файловую систему для хранения записей. Путь к директории, в которой будут храниться файлы, должен быть определен при инициализации кэша.

```ruby
config.cache_store = :file_store, "/path/to/cache/directory"
```

С этим хранилищем кэша несколько серверных процессов на одном хосте могут делиться кэшем. Это хранилище кэша подходит для сайтов с трафиком от низкого до среднего, обслуживающихся на одном или двух хостах. Серверные процессы, запущенные на разных хостах, могут делиться кэшем при использовании общей файловой системы, но эта настройка не рекомендована.

Так как кэш будет расти, пока не заполнится диск, рекомендуется периодически чистить старые записи.

Это реализация хранилища кэша по умолчанию (хранится в `"#{root}/tmp/cache/"`), если `config.cache_store` явно не указан.

[`ActiveSupport::Cache::FileStore`]: https://api.rubyonrails.org/classes/ActiveSupport/Cache/FileStore.html

### `ActiveSupport::Cache::MemCacheStore`

[`ActiveSupport::Cache::MemCacheStore`][] использует сервер Danga's `memcached` для предоставления централизованного кэша вашему приложению. Rails по умолчанию использует встроенный гем `dalli`. Сейчас это наиболее популярное хранилище кэша для работающих веб-сайтов. Оно представляет отдельный общий кластер кэша с очень высокими производительностью и резервированием.

При инициализации кэша необходимо указать адреса для всех серверов memcached в вашем кластере или убедиться, что переменная среды `MEMCACHE_SERVERS` установлена должным образом.

```ruby
config.cache_store = :mem_cache_store, "cache-1.example.com", "cache-2.example.com"
```

Если ничто из этого не определено, предполагается, что memcached запущен на localhost на порте по умолчанию (`127.0.0.1:11211`), но это не идеальная настройка для больших сайтов.

```ruby
config.cache_store = :mem_cache_store # Обратится к $MEMCACHE_SERVERS, затем к 127.0.0.1:11211
```

Поддерживаемые типы адресов смотрите в [документации `Dalli::Client`](https://www.rubydoc.info/gems/dalli/Dalli/Client#initialize-instance_method).

Метод [`write`][ActiveSupport::Cache::MemCacheStore#write] (и `fetch`) на кэше принимают дополнительных опции, дающие преимущества особенностей memcached.

[`ActiveSupport::Cache::MemCacheStore`]: https://api.rubyonrails.org/classes/ActiveSupport/Cache/MemCacheStore.html
[ActiveSupport::Cache::MemCacheStore#write]: https://api.rubyonrails.org/classes/ActiveSupport/Cache/MemCacheStore.html#method-i-write

### `ActiveSupport::Cache::RedisCacheStore`

В [`ActiveSupport::Cache::RedisCacheStore`][] используется поддержка Redis для автоматического вытеснения при достижении максимальной памяти, что позволяет вести себя так же, как сервер кэша Memcached.

Примечание по развертыванию: ключи в Redis не истекают по умолчанию, поэтому будьте осторожны при использовании выделенного сервера кэша Redis. Не заполняйте свой персистентный сервер Redis данными волатильного кэша! Подробнее читайте в [руководстве по настройке сервера кэша в Redis](https://redis.io/topics/lru-cache).

Для сервера Redis с использованием исключительно кэша установите `maxmemory-policy` в один из вариантов allkeys. Redis 4+ поддерживает наименее часто используемое вытеснения (`allkeys-lfu`), отличный выбор по умолчанию. Redis 3 и более ранние должны использовать давно неиспользуемое вытеснение (`allkeys-lru`).

Установите тайм-ауты чтения и записи кэша относительно небольшими. Заново сгенерировать кэшированное значение зачастую быстрее, чем ожидать более секунды для его получения. Тайм-ауты чтения и записи по умолчанию равны 1 секунде, но могут быть установлены меньше, если сеть будет с постоянно низкой задержкой.

По умолчанию хранилище кэша не будет пытаться повторно подключиться к Redis, если произошел сбой соединения во время запроса. Если возникают частые отключения, можно включить автоматическое переподключение.

Кэш читает и записывает никогда не вызывая исключений; вместо этого он просто возвращает `nil`, ведет себя так, будто в кэше ничего не хранится. Чтобы определить наличие в кэше исключений, можно предоставить `error_handler` для сообщения в службу сбора исключений. Он должен принимать три аргумента из ключевых слов: `method`, метод хранения кэша, который изначально был вызван; `returning`, значение, которое было возвращено пользователю, обычно `nil`; и `exception`, исключение, которое было обработано.

Чтобы начать работу, добавьте гем redis в Gemfile:

```ruby
gem 'redis'
```

Наконец, добавьте конфигурацию в соответствующий файл `config/environments/*.rb`:

```ruby
config.cache_store = :redis_cache_store, { url: ENV['REDIS_URL'] }
```

Более сложный пример, хранилище кэша Redis в production может выглядеть примерно так:

```ruby
cache_servers = %w(redis://cache-01:6379/0 redis://cache-02:6379/0)
config.cache_store = :redis_cache_store, { url: cache_servers,

  connect_timeout: 30,     # По умолчанию 20 секунд
  read_timeout:    0.2,    # По умолчанию 1 секунда
  write_timeout:   0.2,    # По умолчанию 1 секунда
  reconnect_attempts: 1,   # По умолчанию 0

  error_handler: -> (method:, returning:, exception:) {
    # Сообщать об ошибках Sentry как предупреждений
    Raven.capture_exception exception, level: 'warning',
      tags: { method: method, returning: returning }
  }
}
```

[`ActiveSupport::Cache::RedisCacheStore`]: https://api.rubyonrails.org/classes/ActiveSupport/Cache/RedisCacheStore.html

### `ActiveSupport::Cache::NullStore`

[`ActiveSupport::Cache::NullStore`][] ограничено каждым веб-запросом, и очищает хранимые значения в конце запроса. Оно предназначается для использования в средах development и test. Это может быть полезным, когда у вас имеется код, взаимодействующий непосредственно с `Rails.cache`, но кэширование может препятствовать способности видеть результат изменений в коде.

```ruby
config.cache_store = :null_store
```

[`ActiveSupport::Cache::NullStore`]: https://api.rubyonrails.org/classes/ActiveSupport/Cache/NullStore.html

### Произвольные хранилища кэша

Можно создать свое собственно хранилище кэша, просто расширив `ActiveSupport::Cache::Store` и реализовав соответствующие методы. Таким образом, можно применить несколько кэширующих технологий в вашем приложении Rails.

Для использования произвольного хранилища кэша просто присвойте хранилищу кэша новый экземпляр класса.

```ruby
config.cache_store = MyCacheStore.new
```

Ключи кэша
----------

Ключи, используемые в кэше могут быть любым объектом, отвечающим либо на `cache_key`, либо на `to_param`. Можно реализовать метод `cache_key` в своем классе, если необходимо сгенерировать обычные ключи. Active Record генерирует ключи, основанные на имени класса и id записи.

Как ключи хэша можно использовать хэши и массивы.

```ruby
# Это правильный ключ кэша
Rails.cache.read(site: "mysite", owners: [owner_1, owner_2])
```

Ключи, используемые на `Rails.cache` не те же самые, что фактически используются движком хранения. Они могут быть модифицированы пространством имен, или изменены в соответствии с ограничениями технологии. Это значит, к примеру, что нельзя сохранить значения с помощью `Rails.cache`, а затем попытаться вытащить их с помощью гема `memcache-client`. Однако, также не стоит беспокоиться о превышения лимита memcached или несоблюдении правил синтаксиса.

(conditional-get) Поддержка GET с условием (Conditional GET)
------------------------------------------------------------

GET с условием - это особенность спецификации HTTP, предоставляющая способ веб-серверам сказать браузерам, что отклик на запрос GET не изменился с последнего запроса и может быть спокойно извлечен из кэша браузера.

Это работает с использованием заголовков `HTTP_IF_NONE_MATCH` и `HTTP_IF_MODIFIED_SINCE` для передачи туда-обратно уникального идентификатора контента и временной метки, когда содержимое было последний раз изменено. Если браузер делает запрос, в котором идентификатор контента (ETag) или временная метка последнего модифицирования соответствует версии сервера, то серверу всего лишь нужно вернуть пустой отклик со статусом not modified.

Это обязанность сервера (т.е. наша) искать временную метку последнего модифицирования и заголовок if-none-match, и определять, нужно ли отсылать полный отклик. С поддержкой conditional-get в Rails это очень простая задача:

```ruby
class ProductsController < ApplicationController

  def show
    @product = Product.find(params[:id])

    # Если запрос устарел в соответствии с заданной временной меткой или значением
    # etag (т.е. нуждается в обработке снова), тогда выполняем этот блок
    if stale?(last_modified: @product.updated_at.utc, etag: @product.cache_key_with_version)
      respond_to do |wants|
        # ... обычное создание отклика
      end
    end

    # Если запрос свежий (т.е. не модифицирован), то не нужно ничего делать
    # Рендер по умолчанию проверит это, с помощью параметров,
    # использованных в предыдущем вызове stale?, и автоматически пошлет
    # :not_modified. И на этом все.
  end
end
```

Вместо хэша опций можно просто передать модель, Rails будет использовать методы `updated_at` и `cache_key_with_version` для настройки `last_modified` и `etag`:

```ruby
class ProductsController < ApplicationController
  def show
    @product = Product.find(params[:id])

    if stale?(@product)
      respond_to do |wants|
        # ... обычное создание отклика
      end
    end
  end
end
```

Если отсутствует специальная обработка отклика и используется дефолтный механизм рендеринга (т.е. вы не используете `respond_to` или вызываете сам `render`), то можете использовать простой хелпер `fresh_when`:

```ruby
class ProductsController < ApplicationController

  # Это автоматически отошлет :not_modified, если запрос свежий,
  # и отрендерит дефолтный шаблон (product.*), если он устарел.

  def show
    @product = Product.find(params[:id])
    fresh_when last_modified: @product.published_at.utc, etag: @product
  end
end
```

Иногда мы хотим кэшировать отклик, например, статичной страницы, которая никогда не устаревает. Чтобы достичь этого, можно использовать хелпер `http_cache_forever`, и в этом случае браузер и прокси закэшируют его на неопределенное время.

По умолчанию кэшированные отклики будут приватными, закэшированными только для браузера пользователя. Чтобы разрешить прокси кэшировать отклик, установите `public: true` чтобы обозначить, что они могут отдавать кэшированный отклик всем браузерам.

При использовании этого хелпера, заголовок `last_modified` устанавливается `Time.new(2011, 1, 1).utc`, и заголовок `expires` устанавливается 100 лет.

WARNING: Используйте этот метод осторожно, так как браузер/прокси будут не в состоянии инвалидировать кэшированный отклик до тех пор, пока кэш браузера будет принудительно очищен.

```ruby
class HomeController < ApplicationController
  def index
    http_cache_forever(public: true) do
      render
    end
  end
end
```

### Сильные против слабых ETag

Rails по умолчанию генерирует слабые ETag. Слабые ETag позволяют семантически эквивалентным откликам иметь одинаковые ETag, даже если их тела не имеют точного совпадения. Это полезно, когда мы не хотим, чтобы страница регенерировалась для второстепенных изменений в теле отклика.

У слабых ETags имеется начальный `W/`, чтобы отличить их от сильных ETag.

```
W/"618bbc92e2d35ea1945008b42799b0e7" → Слабый ETag
"618bbc92e2d35ea1945008b42799b0e7" → Сильный ETag
```

В отличие от слабых ETag, сильные ETag подразумевают, что отклик должен быть в точности идентичным, каждый байт. Полезно, когда делается ряд запросов для больших файлов видео или PDF. Некоторые CDNs поддерживают только сильные ETag, такие как Akamai. Если вам абсолютно необходимо генерировать сильные ETag, это можно сделать следующим образом.

```ruby
class ProductsController < ApplicationController
  def show
    @product = Product.find(params[:id])
    fresh_when last_modified: @product.published_at.utc, strong_etag: @product
  end
end
```

Также можно установить сильный ETag непосредственно на отклике.

```ruby
response.strong_etag = response.body # => "618bbc92e2d35ea1945008b42799b0e7"
```

Кэширование в development
-------------------------

Обычно требуется протестировать стратегию кэширования вашего приложения в development режиме. Rails предоставляет команду rake `dev:cache`, чтобы легко включать и выключать кэширование.

```bash
$ bin/rails dev:cache
Development mode is now being cached.
$ bin/rails dev:cache
Development mode is no longer being cached.
```

По умолчанию, когда кэширование в режиме development *отключено*, Rails использует [`:null_store`](#activesupport-cache-nullstore).

Ссылки
------

* [Статья DHH по прекращению, основанному на ключе](https://signalvnoise.com/posts/3113-how-key-based-cache-expiration-works)
* [Ryan Bates' Railscast по дайджестам кэша](http://railscasts.com/episodes/387-cache-digests)
