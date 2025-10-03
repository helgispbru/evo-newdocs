Создаём модуль для админки Evolution CMS 3.x
 

## Что такое модуль?
 
Модуль

* Мини-приложение внутри Эво: роуты, контроллеры, вьюшки, миграции, ассеты.
* Выполняется в контексте админки (manager), даёт полноценный UI/CRUD.
* Подключается в меню админки.
* Подходит для сложной логики, инструментов админки, конфигураций и экспорта/импорта данных.

Сниппет
* PHP-функция/скрипт, вызываемая зачастую прямо в шаблоне страницы (иногда в контроллерах)
* Обычно  с простой логикой: вывод блока, запрос к БД, умножить $var на ежа.
* Не имеет своих маршрутов/админки по умолчанию.
* Отлично для рендеринга отдельных частей и для быстрого повторного использования кода.

*Из любого правила есть исключения. Сниппеты типа Доклистер -- очень объёмные по функционалу и по коду.*


Когда что выбирать:
* Нужен инструмент с интерфейсом и множеством действий — делай модуль. 
* Нужен лёгкий повторяемый кусок логики для страниц — пиши сниппет.

## Подготовка

Модуль будем делать в отдельном пакете. 
Идём в консоль и из папки `core` запускаем команду

```
php artisan package:create polls
```

Пакет будет называться polls. Вы можете использовать пакет `main`, который зачастую уже в проекте присутствует. Тогда пропустите этот шаг.

 ![Когда-нибудь будут скриншоты из линёвой консоли](https://community.evocms.ru/assets/images/uploads/215/AtX3T7WPOx3guNX.png)

## Подключаем view  

Откройте файл сервис-провайдера пакета. У меня это
```
core\custom\packages\polls\src\PollsServiceProvider.php
```

Проверьте, какой у вас указан `namespace` - дальше он потребуется.

В моём случае:
```php
protected $namespace = 'polls';
```

Измените метод `boot`:

```php
public function boot()
{
	  $this->loadViewsFrom(__DIR__ . '/../views', $this->namespace);
}
```

Что это и зачем? Указываем, что для этого пакета файлы вьюшек теперь будут лежать в папке `/views` в корне пакета, и иметь неймспейс  `polls`. 

Если ваш пакет называется по-другому (`main`, скажем), скорректируйте под себя дальнейший код, мы это будем  довольно много где использовать.

** Создайте  пустую папку views/ в корне пакета **


## Регистрируем маршруты (routes)

Их будет два типа: для админки цмс (в самом модуле) и для внешней части сайта.

Измените или добавьте метод `register`:

```php

public function register()
{
    /**
        * Роуты для модуля в админке
        */
    $this->app->registerRoutingModule(
        'Опросы',
        __DIR__ . '/../routes/inner_routes.php',
        'fa fa-rss',
    );

    /**
        * Внешние роуты
        */
    Route::group(['middleware' => 'bindings'], function () {
        $this->loadRoutesFrom(
            __DIR__ . '/../routes/outer_routes.php'
        );
    });
}
```

Создайте папку routes/ в корне пакета.
В ней два файла: inner_routes.php, outer_routes.php.

Содержимое `outer_routes.php` пока оставляем в виде заглушки:
```php
<?php

use Illuminate\Support\Facades\Route;

/**
 * Роуты для публичной части
 */
```

А вот содержимое роутов для админки `inner_routes.php` уже можно протестировать:

```php
<?php
 
use Illuminate\Support\Facades\Route;

/**
 * Роуты для админ-части
 */
Route::get('/', function() {
    echo 'hello';
})->name('polls.module.index');
```


На данном этапе проект выглядит вот так:

 ![](https://community.evocms.ru/assets/images/uploads/215/7EgBz7uKy0F2XcV.png)

А модуль уже можно запустить внутри админки:

 ![](https://community.evocms.ru/assets/images/uploads/215/0uA5VBy3QivTNhW.png)


## Связывание маршрутов, контроллеров и вьюшек


### Контроллер
Создаём контроллер `core\custom\packages\polls\src\Controllers\Module.php` отвечающий логику модуля.

(Если вы в Main -- меняйте под себя)
```php
<?php

namespace EvolutionCMS\Polls\Controllers;
use Illuminate\Http\Request;

/**
 * Модуль для админки 
 */

class Module
{
    public $data;
    public function __construct()
    {
        // тестовые данные
        $this->data = [
            'name' => 'My 1st module',
            'content' => 'Content 1st module'
        ];
    }

    public function index()
    {
        return response()->view('polls::index', [
            'data' => $this->data
        ]);
    }
}


```

Как видите, `response()->view` использует префикс `polls` и имя `index` для того, чтобы решить, какую вьюшку "подцепить" для отображения. Т.е. это будет файл `core\custom\packages\polls\views\index.blade.php`.

Если вдруг у вас есть необходимость показать что-то из другой папки (странное желание), то без указания префикса Эво найдёт вам вьюшку в корневой папке цмс. Скажем, просто `pages.text` отправит вас в папку `views\pages\text.blade.php`  из корня сайта.



### Маршрут

А теперь расскажем Эво о том, что этот контроллер и его метод index должен отдавать вьюшку при открытии модуля.
Снова идём в `core\custom\packages\polls\src\Controllers\Module.php`:

```php
<?php

use EvolutionCMS\Polls\Controllers\Module;
use Illuminate\Support\Facades\Route;

/**
 * Роуты для админ-части
 */
Route::get('/', [Module::class,'index'])->name('polls.module.index');
```

Если открыть модуль сейчас, будет ошибка -- нет файла вью.


### View

Создаём шаблоны по классике: один базовый, один для отображения.

Файл `app.blade.php`

```
{{-- Базовый шаблон --}}
<?php include_once MODX_MANAGER_PATH . 'includes/header.inc.php' ?>
<h1><i class="fa fa-rss"></i> Мой модуль</h1>

@yield('buttons')

<div class="sectionBody">
    <div class="tab-pane" id="Panel">
        sanitized_by_modx<s cript type="text/javascript">
            var tpModule = new WebFXTabPane(document.getElementById('Panel'), true);
        </script>
        @yield('content')
    </div>
</div>

@stack('scripts')
<?php include_once MODX_MANAGER_PATH . 'includes/footer.inc.php' ?>
```

Я собрал некую заготовку, которую вы сможете в дальнейшем модифицировать под себя. Здесь подключение каких-то базовых вещей из Эво типа стилей или скриптов. Можно и без них -- но так модуль выглядит частью цмс, а не левым ифреймом.

На что обратить внимание:
`@yield('content')` - область вставки контента
`@yield('buttons')` - область вставки кнопок (опционально)
`@stack('scripts')` - область вставки каких-то своих скриптов (опционально)

С базовым шаблоном закончили, переходим к главной странице модуля -- `index.blade.php`

```
@extends('polls::app')

@section('content')
<div class="tab-page" id="monitor_tab_main">
  <h2 class="tab">{{$data['name']}}</h2>
  sanitized_by_modx<s cript type="text/javascript">
    tpModule.addTabPage(document.getElementById('monitor_tab_main'));
  </script>

  <div class="row col-12">
    {{$data['content']}}

  </div>
</div>
@endsection

@section('buttons')
<div id="actions">
  <div class="btn-group">
    <a href="javascript:;" class="btn btn-secondary" onclick="location.reload();">
      <i class="fa fa-refresh"></i><span> Обновить</span>
    </a>
  </div>
</div>
@endsection
 
```


Директива `@extends('polls::app')` указывает нам, откуда наследоваться. Это как раз наш свежесозданный `app.blade.php`. Если эта часть непонятна, рекомендую документацию и раздел "[Работа с шаблонами](https://github.com/0test/evo-newdocs/blob/main/v3/02_Создание%20сайта/003_Работа%20с%20шаблонами%20blade.md "Работа с шаблонами")" 

Переменные `{{$data['name']}}` и ` {{$data['content']}}` вставляют в соответствующие блоки тестовые данные из контроллера.

![](https://community.evocms.ru/assets/images/uploads/215/e5CFocZXfL9q4Gl.png)

Базовый функционал модуля готов. Дальше можете его расширять под ваши нужды: увеличить количество роутов, добавить модели, контроллеры, стили и скрипты.

### Частности

#### Зачем наружные роуты в outer_routes.php?

Модуль -- приватная история для админки. 
Все роуты модуля защищены от внешних запросов. Но иногда модуль должен отдавать наружу какие-то данные.
Вот для них мы и сделали второй файл. Всё, что мы пишем тут, будет открыто всему миру.

Пример.

Измените `outer_routes.php`  :
```php
<?php

use Illuminate\Support\Facades\Route;

/**
 * Роуты для публичной части
 */
Route::get('/polls-any-name-url', function(){
    echo 'im free access page';
})->name('polls.frontend.page');
```

И перейдите на адрес `/polls-any-name-url` -- должна открыться пустая страница с   текстом `im free access page`.

Таким образом модуль может отдавать (и принимать) какие-то данные извне.


#### А свои скрипты или стили как подключить?


Если история глобальная (скажем, стиль нужен везде), то подключаем его в `app.blade.php`
```
<link rel="stylesheet" href="{{ MODX_BASE_URL }}assets/modules/polls/css/poll.css">
```

А если какой-то скрипт нужен для одной страницы, то используем   директиву `@stack('scripts')` (мы её задавали в шаблоне)
```
@push('scripts')
 sanitized_by_modx<s cript src="{{MODX_BASE_URL}}assets/modules/polls/js/poll.js"></script>
@endpush
```

Обратите внимание, что файлы лежат вне пакета, в папке ассетов. Да, неудобно, но такое вот ограничение. С ним можно бороться, если пакет будет опубликован, как полноценный composer-пакет. Скажем, сделать "publish"-метод, как в Ларавел. Но это уже другая история.


#### Миграции как в Ларавел можно?

Да. По умолчанию миграции в Evolution CMS лежат в `core/database/migrations`. Но ничто не мешает пакету регистрировать свои. Для этого в метод `boot` сервис-провайдера добавьте:

```
$this->loadMigrationsFrom(__DIR__ . '/../migrations');
```
Теперь можно создавать миграции прямо внутри пакета. Запустите из core/

```
php artisan make:migration create_polls_table --path=custom/packages/polls/migrations
```

Всё, у вас готовая миграция. Вводите `php artisan migrate` и вперёд.
