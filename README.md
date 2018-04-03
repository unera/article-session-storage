# Апгрейд хранимок Tarantool: «все своё ношу с собой!»

![](https://habrastorage.org/webt/c5/eq/jz/c5eqjzyoyvcwubdqeg27b9h_qjy.png)

В мире баз данных существует сложная проблема рефакторинга и апгрейда хранимых процедур.

Проблема состоит в противоречии:

* С точки зрения эффективности работы с данными желательно максимум бизнес-логики реализовывать в хранимых процедурах.
* С точки зрения эффективности разработки ПО желательно, чтобы части одной программы находились в одном месте. Хранение кода работы с хранилищем прямо в хранилище создаёт много трудностей.


<cut/>

Большой проект традиционно состоит из множества относительно независимых друг от друга приложений, использующих одну и ту же базу данных.

Как правило, развитие проекта происходит неравномерно и итеративно: каждое новое изменение затрагивает лишь несколько составляющих проекта.

Предположим, у вас имеется хранимая процедура `user.profile`, которая возвращает профиль пользователя по его идентификатору. Развивая одну из частей проекта, вы решаете, что эта функция отныне должна возвращать меньшее число параметров. Производите рефакторинг функции и кода, ее использующего. При интеграционном тестировании выясняется, что хранимая процедура `user.profile` также используется в других подпроектах, а изменение привело к багам или неприятным последствиям.

Когда сроки поджимают (т. е. практически всегда), разработчик стоит на распутье:

* Или отказаться от рефакторинга процедуры, поскольку он вызывает проблемы в другом подпроекте.
* Или заняться рефакторингом и смежных проектов тоже.


Способы выбора пути варьируются от проекта к проекту.


### Версионность в названии хранимых процедур

Многие компании создают внутреннюю полиси (зачастую подкреплённую скриптами CI), которая требует называть хранимые процедуры по определённому правилу.
Например, указанную функцию `user.profile` мы переименовываем в `user.profile_0001`, где `0001` — номер её версии. Программист, которому потребовался рефакторинг этой функции, пишет новую — `user.profile_0002`, а `user.profile_0001` продолжает существовать в системе до тех пор, пока весь зависимый от неё код не перепишут.

Достоинства и недостатки такого подхода очевидны.


### Полный отказ от использования хранимых процедур

Я видел многие SQL- и не SQL-проекты, сознательно отказывающиеся от хранимых процедур («кроме тех случаев, когда совсем уж без них нельзя»).
Разрешение противоречия в пользу разработчиков.


### Перенос максимума бизнес-логики в хранимые процедуры

Довольно редкий подход, но тоже встречается. Полная противоположность предыдущего случая. Разные приложения при этом не имеют общих хранимых процедур.


### Сессионное хранилище

Многие БД имеют сессионное хранилище, но немногие предоставляют возможность хранить в нём функции/процедуры.

БД Tarantool имеет [сессионное хранилище](https://tarantool.io/en/doc/1.9/book/box/box_session.html#box-session-storage), позволяющее хранить в нём в том числе и функции.


Что такое сессионное хранилище? Это хранилище, которое будет полностью очищено сразу после того, как клиент отсоединится от базы данных.


Как можно использовать сессионное хранилище для решения поставленных проблем? Алгоритм приблизительно следующий:

1. Клиент коннектится к БД.
2. Клиент заполняет сессионное хранилище ссылками на хранимые процедуры, которые будет использовать.
3. Работает, делая вызовы к созданным процедурам.
4. После дисконнекта БД сама очищает хранилище.

#### Плюсы

1. Код хранимых процедур можно держать в том же Git-дереве, что и код, их использующий.
2. При необходимости обращения к одним и тем же процедурам в разных проектах можно применять Git-сабмодули.
3. Исправление/рефакторинг хранимки и связанного с ней кода упрощается: правки производятся в одном месте.

#### Минусы

1. Компиляция хранимок происходит при каждом коннекте клиента. Если число присоединённых клиентов огромно (тысячи) — может получаться значительный оверхед по использованию памяти.
2. Данный подход, увы, не решает вопрос с хранимками, продолжающими работать и после дисконнекта клиента. Работа с ними — тема для отдельной статьи.

## Пример

Некоторые драйверы к Tarantool позволяют сразу после коннекта автоматически выполнять набор Lua-файлов из заданной директории.
Например, используя Perl-коннектор [DR::Tnt](http://search.cpan.org/~unera/DR-Tnt/lib/DR/Tnt.pm), вы можете указать опцию `lua_dir`.

Размещаем в директории проекта каталог `lua`, в который можем положить несколько файлов. Например `user.lua` из этого каталога будет выглядеть так:

```lua

function box.session.storage.user.profile(id)
    local user = box.space.users:get{id}
    local profile = box.space.profiles:get{id}
    -- do something
    return { user, profile }
end

function box.session.storage.user.add(o)
    return box.space.users:insert{uuid(), o.name, o.surname }
end

function box.session.storage.user.get(id)
    return box.space.users:get{id}
end

```

В приложении коннектимся и работаем, используя процедуры `box.session.storage`:

```perl

my $tnt = tarantool
	host 		=> $host,
	port 		=> $port,
	user 		=> $user,
	password	=> $password,
	lua_dir		=> "lua"
;

# Создание пользователя с именем «Вася»
my $new_user = $tnt->call_lua('box.session.storage.user.add', { name => 'Вася' });

# Получение пользователя
my $user = $tnt->call_lua('box.session.storage.user.get', 123);

```

Относительно длинный неймспейс `box.session.storage` можно сократить до минимального размера, поместив в глобальном справочнике его алиас:

```lua
_G.ss = box.session.storage

```


### Размышления

В «больших» БД, в PostgreSQL, например, существует сессионное хранилище (аналог) для данных `CREATE TEMPORARY TABLE`. Было бы замечательно, если бы там появилась возможность создавать и временные функции `CREATE TEMPORARY FUNCTION`, которые были бы видны только приконнектившемуся клиенту и удалялись бы после отсоединения.


## Ссылки

1. [Эта статья на GitHub](https://github.com/unera/session-storage-article)
2. [Проект Tarantool](http://tarantool.org)
3. [box.session.storage в документации](https://tarantool.io/en/doc/1.9/book/box/box_session.html#box-session-storage)
