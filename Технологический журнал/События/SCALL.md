


Событие **`SCALL` (Server Call / Исходящий вызов)** — это твой главный секундомер. Но с этим событием связана **самая большая ловушка для сисадминов**, на которой валятся 90% кандидатов на собеседованиях.

Давай разберем эту ловушку, поймем физику процесса и выясним, как использовать `SCALL` для поиска "тормозов".

---

### 1. ГЛАВНАЯ ЛОВУШКА: `SCALL` vs `CALL` (Кто держит секундомер?)

Переведи слово `SCALL` как **«Исходящий звонок»**, а `CALL` как **«Входящий звонок»**.

*   Когда бухгалтер (на своем Windows-ПК) нажимает кнопку «Провести», его Тонкий клиент делает *исходящий звонок* на твой сервер.
*   Твой сервер Linux (процесс `rphost`) принимает этот *входящий звонок*.

**Внимание, вопрос:** Если ты включишь сбор `SCALL` на своем Linux-сервере, поймаешь ли ты тормоза интерфейса у бухгалтера?
**ОТВЕТ: НЕТ!** 
Событие `SCALL` бухгалтера запишется **на компьютере бухгалтера** (в его локальный лог), потому что именно он был инициатором звонка. На Linux-сервере это действие запишется как **`CALL`**.

---

### 2. Зачем тогда нам нужен `SCALL` на Linux-сервере?

Если на Linux-сервере мы видим `SCALL`, это значит, что **твой сервер сам кому-то позвонил и ждет ответа!** (Он выступил в роли клиента).

Вот три реальных жизненных сценария, когда `SCALL` на сервере спасет тебе жизнь:

#### 📞 Сценарий А: Интеграции и Web-сервисы (Внешний мир)
*   **Ситуация:** Твоя 1С:ERP при проведении документа лезет на сайт ФНС, в Честный Знак или в банк, чтобы проверить контрагента. 1С зависает на 20 секунд. Программисты кричат: *"Linux тупит, Postgres висит!"*.
*   **Как спасает `SCALL`:** В логе ты видишь, что `rphost` сделал `SCALL` (исходящий вызов) на внешний HTTP-адрес Честного Знака. Свойство `Duration` равно 20 секундам. 
*   **Твой вердикт:** *"Сервер работает идеально. Ваш код ждет ответа от сайта Честного Знака 20 секунд, потому что их сайт "лежит" или у нас забит шлюз провайдера"*.

#### 📞 Сценарий Б: Общение внутри кластера (`rphost` ➔ `rmngr`)
*   **Ситуация:** Сервер 1С внезапно начал тормозить при выдаче лицензий или установке управляемых блокировок.
*   **Как спасает `SCALL`:** Ты видишь, что рабочий процесс (`rphost`) звонит менеджеру кластера (`rmngr`), чтобы попросить лицензию, и этот `SCALL` длится 5 секунд (а должен 0.01 сек).
*   **Твой вердикт:** *"Проблема не в базе данных. У нас перегружен или завис процесс `rmngr` (например, ему не хватает оперативной памяти)"*.

#### 📞 Сценарий В: COM-соединения (если бы был Windows)
Если база 1С лезет в другую базу 1С через COM-соединение, это тоже будет зафиксировано как `SCALL`.

---

### Шаг 3. Поиск свободного кассира (Балансировка нагрузки)
> **`IName = ISelectSrvrProcess`** (Интерфейс выбора серверного процесса)

Прежде чем начать работу, Тонкому/Толстому клиенту нужно понять, к какому именно `rphost` ему подключиться (у тебя же их может быть несколько). Клиент обращается к Менеджеру кластера:
*   **`MName = methodsCount`** (Посчитать методы): Служебный микро-запрос. Клиент проверяет, жив ли интерфейс и какие команды он поддерживает, либо совместимость версий клиента и сервера.
*   **`MName = selectProcess`** (Выбрать процесс): Самая главная команда. Клиент говорит: *"Шеф, дай мне наименее загруженный `rphost`"*. Сервер отвечает ему: *"Иди на порт 1560"*.

### Шаг 4. Подготовка рабочего стола (Выделение памяти)
> **`IName = IRemoteCreatorService`** (Служба удаленного создания)

Теперь клиент знает, с каким `rphost` он работает. Но чтобы начать обмениваться данными, клиенту нужно, чтобы сервер выделил для него кусочек оперативной памяти и создал там нужные объекты.
*   **`MName = createRemoteInstance`** (Создать удаленный экземпляр): Клиент просит сервер: *"Создай для меня в своей памяти вот этот системный объект, я сейчас буду с ним работать"*.

### Шаг 5. Перекачка картинок и формочек (Тот самый VRS)
> **`IName = IVResourceRemoteConnection`** (Удаленное подключение к виртуальным ресурсам)

Помнишь, мы обсуждали кэш (`VRSCACHE`)? Клиенту нужно отрисовать на экране красивую форму с кнопочками, но у него нет данных.
*   **`MName = methodsCount`**: Снова служебная проверка готовности интерфейса.
*   **`MName = send`** (Отправить): Клиент отправляет на сервер запрос: *"Дай мне описание структуры вот этого документа, чтобы я нарисовал его на мониторе"*.
    *   *Обрати внимание на длительность (первая цифра в строке):* В самой нижней строке `send` длился **26187992** микросекунд (почти **26 секунд**!). 

### Шаг 6. Уборка мусора за собой
*В логе: Строка 4*
> **`MName = Release`** (Освободить)

Заметил, что тут нет `IName`? Это универсальная команда.
Клиент говорит серверу: *"Всё, тот временный объект, который ты для меня создавал (в шаге 1 или 2), мне больше не нужен. Можешь удалить его из оперативки"*. Это нормальная работа сборщика мусора.

---

### Итог: Как это использовать тебе, как Админу?

Тебе **не нужно** зубрить все эти интерфейсы наизусть (их в 1С сотни). Тебе нужно понимать общую логику:

1.  Если ты видишь **`ISelectSrvrProcess`** — значит, клиент только-только подключается к кластеру.
2.  Если ты видишь **`IVResourceRemoteConnection`** — значит, идет передача «внешнего вида» программы (картинки, кэш, интерфейс).
3.  Если ты видишь вызовы, у которых `Duration` (цифра после дефиса) **1, 2 или 4 микросекунды** (как в верхних строках твоего лога) — значит, сервер отвечает **МГНОВЕННО**. Сервер не виноват.
4.  А вот если (как в последней строке) вызов длится **26 000 000 микросекунд**, ты смотришь на `IName` и понимаешь: *"Ага, это не база данных тормозит, это у нас тяжелые формы по узкому интернет-каналу долго пролезают"*.


   
### 7. Как правильно настроить в `logcfg.xml`

Любые события вызовов (`CALL` и `SCALL`) генерируются тысячами в секунду. Их **КАТЕГОРИЧЕСКИ ЗАПРЕЩЕНО** включать без фильтра по времени, иначе диск умрет.

```xml
<!-- Ловим исходящие звонки от сервера (интеграции, внутренние тупняки) -->
<log location="/var/log/1c/server_outgoing_calls" history="24">
    <event>
        <eq property="Name" value="SCALL"/>
        <!-- Ловим только те звонки, где сервер прождал на трубке дольше 3 секунд -->
        <ge property="Duration" value="3000000"/>
    </event>
    <!-- Пишем все колонки, чтобы получить Context и Interface -->
    <property name="all"/>
</log>
```

---

1.  Событие имеет длительность, которая определяет время обработки исходящего
вызова.
2.  Событие фиксируется в момент успешного окончания вызова.
3.  Важное свойство события: `CalIID` - это идентификатор входящего вызова по которому можно
найти парный исходящий вызов (событие CALL).
4.  Разница времени между SCALL и парным ему CALL показывает потери времени на передачу
информации между источником и приемником вызова.

`26:54.148000-9968994,SCALL,3,process=1cv8c,OSThread=13876,ClientID=3,Interface=bc15bd01-10bf-413c-a856-
ddc907fcd123,IName=IVResourceRemoteConnection,Method=0,
CallID=12282,MName=send,
DstClientID=5,
Context='
Обработка.ПримерУправляемойФормы.Форма.Форма.Форма : 4 :
ВыполнитьСерверныйВызовНаСервере(КоличествоЭлементов, ПодстрокаПоиска);"`

---

Ссылка: https://infostart.ru/1c/articles/1407627/

Чтобы пойти дальше по событиям, нужно понять - что происходит, когда клиент подключается к кластеру. Первое – клиент первоначально устанавливает соединение с rmng.
Второе – rmng создает сеанс. Сеанс – это некая абстрактная сущность, которая представляет пользователя в системе:
rmngr:
`01:05.213000-14996,SCALL,2,level=INFO,process=1cv8c,OSThread=12188,ClientID=1,Interface=7f58f27d-5ad8-43a1-aa1e-c982f41bed5c,IName=IRemoteCreatorService,Method=0,CallID=12804,MName=createRemoteInstance,DstClientID=0
01:05.229000-15999,SCALL,2,level=INFO,process=1cv8c,OSThread=12188,ClientID=1,Interface=73b7d3a3-fe0b-4fdf-ba70-b74b3589ffc3,IName=ISelectSrvrProcess,CallID=12805,MName=methodsCount,DstClientID=248048
01:05.229002-1,SCALL,2,level=INFO,process=1cv8c,OSThread=12188,ClientID=1,Interface=73b7d3a3-fe0b-4fdf-ba70-b74b3589ffc3,IName=ISelectSrvrProcess,Method=0,CallID=12806,MName=selectProcess,DstClientID=248048
01:05.229004-1,SCALL,2,level=INFO,process=1cv8c,OSThread=12188,ClientID=1,CallID=12807,MName=Release,DstClientID=248048`

Что-то делает
rphost:
`01:05.244002-1,SCALL,2,level=INFO,process=1cv8c,OSThread=12188,ClientID=2,Interface=7f58f27d-5ad8-43a1-aa1e-c982f41bed5c,IName=IRemoteCreatorService,Method=0,CallID=12808,MName=createRemoteInstance,DstClientID=0
01:05.479000-218999,SCALL,2,level=INFO,process=1cv8c,OSThread=12188,ClientID=2,Interface=7f58f27d-5ad8-43a1-aa1e-c982f41bed5c,IName=IRemoteCreatorService,Method=0,CallID=12809,MName=createRemoteInstance,DstClientID=0
01:05.479003-1,SCALL,2,level=INFO,process=1cv8c,OSThread=12188,ClientID=2,Interface=bc15bd01-10bf-413c-a856-ddc907fcd123,IName=IVResourceRemoteConnection,CallID=12810,MName=methodsCount,DstClientID=81104`

Третье – rmng выделяет под сеанс определенные место в сеансовых данных:
rmngr:
`01:05.510001-30992,SCALL,3,level=INFO,process=1cv8c,OSThread=12188,ClientID=3,Interface=7f58f27d-5ad8-43a1-aa1e-c982f41bed5c,IName=IRemoteCreatorService,Method=0,CallID=12811,MName=createRemoteInstance,DstClientID=0
01:05.510003-1,SCALL,3,level=INFO,process=1cv8c,OSThread=12188,ClientID=3,Interface=73b7d3a3-fe0b-4fdf-ba70-b74b3589ffc3,IName=ISelectSrvrProcess,CallID=12812,MName=methodsCount,DstClientID=248049
01:05.510005-1,SCALL,3,level=INFO,process=1cv8c,OSThread=12188,ClientID=3,Interface=73b7d3a3-fe0b-4fdf-ba70-b74b3589ffc3,IName=ISelectSrvrProcess,Method=0,CallID=12813,MName=selectProcess,DstClientID=248049
01:05.510007-1,SCALL,3,level=INFO,process=1cv8c,OSThread=12188,ClientID=3,CallID=12814,MName=Release,DstClientID=248049`

Что-то делает
rphost:
`01:05.557000-15999,SCALL,3,level=INFO,process=1cv8c,OSThread=12188,ClientID=4,Interface=7f58f27d-5ad8-43a1-aa1e-c982f41bed5c,IName=IRemoteCreatorService,Method=0,CallID=12815,MName=createRemoteInstance,DstClientID=0
01:05.557004-1,SCALL,3,level=INFO,process=1cv8c,OSThread=12188,ClientID=4,Interface=7f58f27d-5ad8-43a1-aa1e-c982f41bed5c,IName=IRemoteCreatorService,Method=0,CallID=12816,MName=createRemoteInstance,DstClientID=0
01:05.557007-1,SCALL,3,level=INFO,process=1cv8c,OSThread=12188,ClientID=4,Interface=bc15bd01-10bf-413c-a856-ddc907fcd123,IName=IVResourceRemoteConnection,CallID=12817,MName=methodsCount,DstClientID=81105`

Четвертое – rmng назначает сеанс на какой-то рабочий процесс rphost и пятое – клиент устанавливает с этим rphost TCP соединение:
rmngr:
`01:05.869000-45996,SCALL,4,level=INFO,process=1cv8c,OSThread=12188,ClientID=5,Interface=7f58f27d-5ad8-43a1-aa1e-c982f41bed5c,IName=IRemoteCreatorService,Method=0,CallID=12818,MName=createRemoteInstance,DstClientID=0
01:05.869002-1,SCALL,4,level=INFO,process=1cv8c,OSThread=12188,ClientID=5,Interface=73b7d3a3-fe0b-4fdf-ba70-b74b3589ffc3,IName=ISelectSrvrProcess,CallID=12819,MName=methodsCount,DstClientID=248051
01:05.869004-1,SCALL,4,level=INFO,process=1cv8c,OSThread=12188,ClientID=5,Interface=73b7d3a3-fe0b-4fdf-ba70-b74b3589ffc3,IName=ISelectSrvrProcess,Method=0,CallID=12820,MName=selectProcess,DstClientID=248051
01:05.869006-1,SCALL,4,level=INFO,process=1cv8c,OSThread=12188,ClientID=5,CallID=12821,MName=Release,DstClientID=248051`

Пошла работа
rphost:
`01:05.901000-15997,SCALL,4,level=INFO,process=1cv8c,OSThread=12188,ClientID=6,Interface=7f58f27d-5ad8-43a1-aa1e-c982f41bed5c,IName=IRemoteCreatorService,Method=0,CallID=12822,MName=createRemoteInstance,DstClientID=0
01:05.916000-14997,SCALL,4,level=INFO,process=1cv8c,OSThread=12188,ClientID=6,Interface=7f58f27d-5ad8-43a1-aa1e-c982f41bed5c,IName=IRemoteCreatorService,Method=0,CallID=12823,MName=createRemoteInstance,DstClientID=0
01:05.916003-1,SCALL,4,level=INFO,process=1cv8c,OSThread=12188,ClientID=6,Interface=bc15bd01-10bf-413c-a856-ddc907fcd123,IName=IVResourceRemoteConnection,CallID=12824,MName=methodsCount,DstClientID=81107
01:31.604001-25687996,SCALL,0,level=INFO,process=1cv8c,OSThread=15860,ClientID=6,Interface=bc15bd01-10bf-413c-a856-ddc907fcd123,IName=IVResourceRemoteConnection,Method=0,CallID=12825,MName=send,DstClientID=81107
01:32.229000-265999,SCALL,0,level=INFO,process=1cv8c,OSThread=10024,ClientID=6,Interface=bc15bd01-10bf-413c-a856-ddc907fcd123,IName=IVResourceRemoteConnection,Method=0,CallID=12826,MName=send,DstClientID=81107
01:39.682001-7312993,SCALL,0,level=INFO,process=1cv8c,OSThread=10024,ClientID=6,Interface=bc15bd01-10bf-413c-a856-ddc907fcd123,IName=IVResourceRemoteConnection,Method=0,CallID=12827,MName=send,DstClientID=81107
01:39.776000-15985,SCALL,0,level=INFO,process=1cv8c,OSThread=10024,ClientID=6,Interface=bc15bd01-10bf-413c-a856-ddc907fcd123,IName=IVResourceRemoteConnection,Method=0,CallID=12828,MName=send,DstClientID=81107
01:39.823021-1,SCALL,0,level=INFO,process=1cv8c,OSThread=10024,ClientID=6,Interface=bc15bd01-10bf-413c-a856-ddc907fcd123,IName=IVResourceRemoteConnection,Method=0,CallID=12829,MName=send,DstClientID=81107
01:48.057000-8124985,SCALL,0,level=INFO,process=1cv8c,OSThread=16808,ClientID=6,Interface=bc15bd01-10bf-413c-a856-ddc907fcd123,IName=IVResourceRemoteConnection,Method=0,CallID=12830,MName=send,DstClientID=81107`
