# Обслуживания базы данных 

## Об этом много сказано

Тема обслуживания баз данных (индексов и статистик) поднимается достаточно часто в разных статьях, официальной документации, на форумах и так далее. Обычно дается описание процессов обслуживания, зачем они нужны и какие бывают, на что влияют. И, конечно же, как сделать настройку обслуживания. Последнее обычно преподносится на базовом уровне, но, конечно, не всегда.

Сегодня мы снова коснемся этой темы, но несколько в ином ключе. Мы на практических примерах посмотрим на настройку обслуживания от простого случая до более продвинутого. Так можно будет проследить как меняется обслуживание при изменении требований к работе информационной системы.

Мы сосредоточимся на решении именно практических задач. Теорию Вы можете найти по ссылкам в конце статьи.

## Общие слова

Прежде чем мы перейдем к примерам, определимся с тем, что будем делать.

Обслуживание подразумевает две больших части:

1. Поддержание фрагментации индексов на приемлемом уровне. В идеале процент фрагментации у каждого индекса должен быть равен 0 или близок к этому значению.
2. Состояние статистики должно быть в максимально в актуальном состоянии. При изменении строк в таблице, объекты статистики должны иметь гистограмму распределения значений, которая отражает состояние объектов базы наиболее актуальным образом.

Все это влияет на формируемые планы запросов. В самых общих чертах, если индекс имеет высокую фрагментацию или статистика знатно устарела, то оптимизатор SQL Server даже не будет пытаться использовать индекс и выполнит полное сканирование таблицы. Последнее, как Вы понимаете, не самая быстрая и оптимальная ситуация. Есть и другие последствия отсутствия обслуживания, но это совсем другая история.

Также сразу отметим, что никаких сторонних программ для настройки обслуживания мы использовать не будем. Чистый TSQL, возможно, в готовых скриптах и/или хранимых процедурах. Но никакого стороннего софта.

Итак, настало время первого примера!

## Примеры, примеры, примеры

На старт, внимание, марш!

### Типичное обслуживание

Когда речь заходит об обслуживании, то обычно начинают с создания плана обслуживания, где используют готовые компоненты. Для этого создаем план обслуживания через SQL Server Managment Studio в разделе "Managment -> Maintenance Plans" с субпланом "FullMaintenance".

![](img/d4cef60ac717f45a738e5ce7aeabd20b.png)Механизм планов обслуживания базируется на "обрезанной" версии [**SSIS**](https://learn.microsoft.com/ru-ru/sql/integration-services/sql-server-integration-services?view=sql-server-ver15), если так можно выразиться. Для использования готовых компонентов нужно использовать именно его.

![](img/8725438f0fe48fe5e7cf5e07373431f1.png)

Альтернативным вариантом является использование заданий (job'ов) агента SQL Server, просто указывая явно скрипты TSQL для выполнения.

В созданный субплан добавляем три компонента. Выше Вы уже могли видеть схему субплана обслуживания.

Rebuild Index Task

Компонент перестроения индексов **"Rebuild Index Task"** имеет в себе большинство необходимых настроек для запуска простого обслуживания. [**В официальной документации он достаточно подробно описан.**](https://learn.microsoft.com/ru-ru/sql/relational-databases/maintenance-plans/rebuild-index-task-maintenance-plan?view=sql-server-ver15)

![](img/aea12746d23e4c58ec7466be475440b6.png)

* **Connection** - указываем параметры соединения с базой данных, сервером СУБД. Обычно здесь остается стандартная настройка, останавливаться подробней не будем.
* **Database(s)** - указываем для какой базы (или списка баз) нужно выполнять обслуживание. Обычный фильтр.
* **Object** - здесь мы можем выбрать для каких объектов базы данных нужно выполнять обслуживание.
  + **Tables and Views** - все объекты базы.
  + **Tables** - таблицы базы, можно указать все или выбрать конкретные.
  + **Views** - представления базы, можно указать все или выбрать конкретные.
* **Free space options** - параметр отвечает за то, сколько процентов свободного места нужно оставлять в каждой странице. По умолчанию используются стандартные параметры сервера, но это значение можно переопределить. **[Подробнее об этом параметре читайте здесь.](https://learn.microsoft.com/ru-ru/sql/relational-databases/indexes/specify-fill-factor-for-an-index?view=sql-server-ver15)** Если оставлять некоторое место на страницах свободным, то можно уменьшить рост фрагментации индексов, т.к. изменения будут дописываться в существующие страницы и связанные данные меньше будут "раскидываться" между отдельными страницами. Подробнее читайте в документации, рассматривать это сегодня более детально не будем.
* **Advaned options** - это различные дополнительные настройки для более тонкого управления обслуживанием индексов.
  + **Sort results in tempdb** - этот флаг включает хранение промежуточных результатов сортировки при формировании индекса. По умолчанию они хранятся в памяти или целевой файловой группе. **[Подробнее читать здесь.](https://learn.microsoft.com/ru-ru/sql/relational-databases/indexes/sort-in-tempdb-option-for-indexes?view=sql-server-ver15)** По умолчанию он установлен в ЛОЖЬ.
  + **Keep index online** - SQL Server поддерживает [**перестроение индексов в онлайн режиме**](https://learn.microsoft.com/ru-ru/sql/relational-databases/indexes/perform-index-operations-online?view=sql-server-ver15), при котором работа с таблицей и индексом не блокируется во время выполнения обслуживания. По завершению перестроения выполняется переключение старого индекса на новый, что позволяет исключить блокирование работы запросов и выполнять обслуживание при минимальном размере технологического окна. Или при его полном отсутствии. Дополнительно к этому режиму указывают варианты действия для тех индексов, которые не поддерживают онлайн-обслуживание (например, если содержат типы text, ntext, image, filestream. Первые 3 считаются устаревшими, но все еще поддерживаются для совместимости). В момент переключения, конечно же, кратковременно создается блокировка на уровне схемы данных и есть различные сценарии обработки этой блокировки. Об этом подробней мы поговорим ниже.
    - **Do not rebuild** - индексы без поддержки онлайн перестроения будут пропущены.
    - **Rebuild indexes offline** - индекс без поддержки онлайн перестроения будет обслужен в обычном режиме с блокировкой работы с ним.
  + **Pad index** - указать заполнение индекса. Определяет разреженность индекса. По умолчанию выключен. При включении настройка fillfactor применяется к страницам промежуточного индекса.
  + **MAXDOP** - указываем степень параллелизма для операций обслуживания. Таким образом можно указать параметр, отличный от значения параметра всего сервера.
* **Index Stats Options** - параметры анализа индексов перед обслуживанием. Фактически это режимы просмотра для системной DMV [**sys.dm\_db\_index\_physical\_stats**](https://learn.microsoft.com/ru-ru/sql/relational-databases/system-dynamic-management-views/sys-dm-db-index-physical-stats-transact-sql?view=sql-server-ver15), которые влияют на работу некоторых фильтров и точность результатов анализа. Но зато поверхностный анализ позволяет ускорить запросы перед началом обслуживания. Детальный анализ потребует больше времени и создаст дополнительную нагрузку на сервер.
  + Fast
  + Sampled
  + Detailed
* **Oprimize index only if** - дополнительные фильтры для обслуживаемых индексов.
  + **Fragmnetation >** - фрагментация должна быть больше определенного процента. Общепринятым правилом считается, что перестроение нужно выполнять, если фрагментация выше 30%.
  + **Page Count** - фильтр по размеру индекса. Указывается в количестве страниц. Каждая страница = 8 КБ.

Базовые настройки именно такие.

Reorganize Index Task

[**Компонент реорганизации индексов "Reorganize Index Task"**](https://learn.microsoft.com/ru-ru/sql/relational-databases/maintenance-plans/reorganize-index-task-maintenance-plan?view=sql-server-ver15). Позволяет выполнять операции реорганизации, которые требуют меньше ресурсов, чем перестроение. Также этот процесс выполняется без долгосрочных блокировок объекта и минимально влияет на текущую работу запросов. Обычно дефрагментация при таком способе выполняется для страниц конечного уровня, что делает такую операцию не всегда оптимальным вариантом.

![](img/8977403e279523ece781ab05881a201a.png)

* **Connection** - указываем параметры соединения с базой данных, сервером СУБД. Обычно здесь остается стандартная настройка, останавливаться подробней не будем.
* **Database(s)** - указываем для какой базы (или списка баз) нужно выполнять обслуживание. Обычный фильтр.
* **Object** - здесь мы можем выбрать для каких объектов базы данных нужно выполнять обслуживание.
  + **Tables and Views** - все объекты базы.
  + **Tables** - таблицы базы, можно указать все или выбрать конкретные.
  + **Views** - представления базы, можно указать все или выбрать конкретные.
* **Compact large objects** - освобождение пространства для таблиц и индексов, если возможно.
* **Index Stats Options** - параметры анализа индексов перед обслуживанием. Фактически это режимы просмотра для системной DMV [**sys.dm\_db\_index\_physical\_stats**](https://learn.microsoft.com/ru-ru/sql/relational-databases/system-dynamic-management-views/sys-dm-db-index-physical-stats-transact-sql?view=sql-server-ver15), которые влияют на работу некоторых фильтров и точность результатов анализа. Но зато поверхностный анализ позволяет ускорить запросы перед началом обслуживания. Детальный анализ потребует больше времени и создаст дополнительную нагрузку на сервер.
  + Fast
  + Sampled
  + Detailed
* **Oprimize index only if** - дополнительные фильтры для обслуживаемых индексов.
  + **Framnetation >** - фрагментация должна быть больше определенного процента. Общепринятым правилом считается, что реорганизацию нужно выполнять, если процент фрагментации находится между 15 и 30%.
  + **Page Count** - фильтр по размеру индекса. Указывается в количестве страниц. Каждая страница = 8 КБ.
  + **Used in last** - фильтр позволяет указать, что индекс должен был использоваться за последние дни (кол. дней).

Настройки похожи на перестроение. Убраны параметры, которые для реорганизации просто не используются. Например MAXDOP, ведь операция реорганизации не распараллеливается как перестроение.

Update Statistics Task

[**Компонент обслуживания статистики "Update Statistics task".**](https://learn.microsoft.com/ru-ru/sql/relational-databases/maintenance-plans/update-statistics-task-maintenance-plan?view=sql-server-ver15) Позволяет настроить обслуживание статистики в базе данных.

![](img/f75ddf640e0f6bc55074339dc3a7d67e.png)

* **Connection** - указываем параметры соединения с базой данных, сервером СУБД. Обычно здесь остается стандартная настройка, останавливаться подробней не будем.
* **Database(s)** - указываем для какой базы (или списка баз) нужно выполнять обслуживание. Обычный фильтр.
* **Object** - здесь мы можем выбрать для каких объектов базы данных нужно выполнять обслуживание.
  + **Tables and Views** - все объекты базы.
  + **Tables** - таблицы базы, можно указать все или выбрать конкретные.
  + **Views** - представления базы, можно указать все или выбрать конкретные.
* **Update** - режим обновления объектов статистики.
  + **All existing statistics** - будут обновлены все объекты статистики.
  + **Column statistics only** - обновление только статистики столбцов.
  + **Index statictis only** - только статистику индексов.
* **Scan type** - тип сканирования данных таблицы для актуализации гистограммы распределения значений статистики.
  + **Full scan** - полное сканирование всех значений.
  + **Sample by** - для анализа будет использован только некоторый процент данных из таблицы, что позволит ускорить процесс обновления, но снизит точность.

На скриншоте ниже можно увидеть объекты статистики разных типов. Зеленым обведены объекты статистики, которые связаны с индексами. А объекты статистики с именем "\_WA\_Sys\_\*", обведенные красным, это как раз служебные статистики столбцов, которые СУБД создает автоматически. Конечно, никто не мешает создать свой объект статистики, если в этом есть необходимость.

![](img/c982ce19838735d2e079e84b269d7a6f.png)

Чаще всего оставляют обновление всех объектов статистики, а не только тех, которые относятся к индексам. Также в абсолютном большинстве случаев достаточно оставить обновление статистики через анализ ограниченной выборки данных. Полное сканирование конечно хорошо, но на больших базах оно может выполняться много часов. Если база небольшая, то можно оставлять и полное сканирование, но нужно учитывать, что обновление статистики таким образом в некоторых случаях может [**создавать проблемы ввода-вывода**](https://www.brentozar.com/archive/2014/01/update-statistics-the-secret-io-explosion/), но опять же для больших баз.

Мы пробежались по основным настройкам каждого доступного компонента.

Предположим, что у нас типичная задача - небольшая база на SQL Server, работ ночью не ведется и в качестве технологического окна у нас время с 21:00 до 06:00. Идеально! Тогда настроим запуск обслуживания как на схеме выше, чередуя операции следующим образом:

1. Перестроение индексов с указанием нужной базы и выбрав все таблицы и представления. При этом не используем онлайн-перестроение, оставляем отключенной сортировку в TempDB и так далее. В общем все настройки стандартные как на скрине выше, разве что MAXDOP можно поставить максимальный, пусть все ресурсы сервера будут выделены под перестроение.
2. Вторым шагом реорганизация индексов. Также оставляем все настройки по умолчанию. Реорганизацию оставляем именно вторым шагом, т.к. в первую очередь нужно выполнить обслуживание индексов, у которых фрагментация выше 30%.
3. И последней операцией устанавливаем обновление статистики. Пусть выполняется для всех объектов статистики и методом полного сканирования.

Также рекомендую добавить еще один субплан обслуживания для обновления статистики в дневное время, чтобы сгладить изменения в базе и помочь оптимизатору запросов актуальной статистикой. Например, установить запуск операции ежедневно в 13:00 (например, сориентироваться на обеденное время сотрудников). Настройки такие же, как и у обновления статистики в ночное время.

![](img/6b0fc5ee422249f0e6766387110ee7b5.png)

Так как база небольшая, то, скорее всего, весь процесс обслуживания завершится за 30-60 минут ночью и за 5-15 минут днем, а может и еще быстрее.

Подведем небольшой итог.

Плюсы:

* Простота настройки

Минусы:

* Отсутствие контроля выполнения
* Блокирование работы запросов в момент обслуживания
* Нет возможности проанализировать результаты выполнения обслуживания в динамике, нет истории работы обслуживания

Но если база небольшая, то все перечисленные минусы несущественны. Что такое небольшая база? Понятие относительное :). Главное, что нужно понять, так это если такое обслуживание Вас полностью устраивает, то делать что-то более сложное нет смысла.

### Первые проблемы

Какое-то время все шло хорошо, но от бизнеса появилось новое требование: в ночное время обслуживание не должно мешать работе информационной системы. Например, могли появиться важные регламентные задания по интеграции, расчетам и так далее.

Выше мы уже говорили, что SQL Server поддерживает онлайн-перестроение индексов и мы можем его использовать для тех объектов, которые это поддерживают. Те объекты, для которых онлайн-перестроение нельзя выполнить из-за использования legacy-типов в полях (text, ntext, image), будут обслужены обычным образом с блокировкой работы с ними. Так что от всех проблем мы не избавимся, но других настроек в стандартном компоненте нет. Поэтому следует выполнить следующие изменения:

* В шаге перестроения индексов, который выполняется в ночное время, установить использование онлайн перестроения. По сравнению с обычной операцией обслуживания онлайн перестроение требует больше ресурсов по CPU и по использованному месту, да и лог транзакций при полной модели восстановления будет заполнен больше. Такова цена беспрерывной работы запросов.

![](img/2b0a72b7940dc0f0f2279f3c4a4e62ff.png)

* Степень параллелизма желательно установить ограниченным значением, чтобы CPU не был загружен на 100% во время обслуживания. Ведь блокировки не единственная проблема, которая может помешать работе информационной системе. Высокая нагрузка от обслуживания тоже может остановить работу. Общая рекомендация поставить MAXDOP = 30% от общего количества ядер. Например, если на сервере 24 ядра, то под перестроение индексов выделить только 8.
* Остальные настройки оставляем как есть.

Тут также подведем небольшой итог.

Плюсы:

1. Простота настройки
2. Минимальное влияние на работу системы во время обслуживания

Минусы:

1. Отсутствие контроля выполнения
2. Блокирование работы запросов в момент обслуживания для тех объектов, в которых онлайн перестроение невозможно
3. Нет возможности проанализировать результаты выполнения обслуживания в динамике, нет истории работы обслуживания

Опять же, если последние 3 причины не критичны, то все отлично!

### Большой шаг

Компания развивается, и информационная база растет. У нас появились большие таблицы и много, а технологическое окно уменьшилось: с 21:00 до 23:00. Появились склады, которые работают 24/7. Кроме этого остается старая проблема с объектами, которые не поддерживают онлайн перестроение. Мы должны их обслуживать так, чтобы они минимально создавали блокировки. Кроме этого, нужно гарантировать, что обслуживание не выйдет за пределы установленного технологического окна с 21:00 до 23:00.

К сожалению, стандартными компонентами обслуживания, которыми пользовались до этого момента, данную задачу решить уже нельзя. Поэтому мы пойдем другим путем. Мы создадим служебную базу обслуживания и мониторинга с именем **"SQLServerMaintenance"**, а далее наполним ее объектами следующим скриптом.

Скрипт создания служебной базы мониторинга и обслуживания

Этот скрипт [**взят отсюда**](https://github.com/YPermitin/SQLServerTools/tree/master/SQL-Server-Maintenance/Service-Database). Там же есть примеры использования, описания процедур и их параметров, а также другая связанная информация.
???- info "скрипт"
    ``` sql


    USE [SQLServerMaintenance]

    GO



    CREATE TABLE [dbo].[MaintenanceActionsLog](

        [Id] [bigint] IDENTITY(1,1) NOT NULL,

        [Period] [datetime2](0) NOT NULL,

        [TableName] [nvarchar](255) NOT NULL,

        [IndexName] [nvarchar](255) NOT NULL,

        [Operation] [nvarchar](100) NOT NULL,

        [RunDate] [datetime2](0) NOT NULL,

        [StartDate] [datetime2](0) NOT NULL,

        [FinishDate] [datetime2](0) NULL,

        [DatabaseName] [nvarchar](500) NOT NULL,

        [UseOnlineRebuild] [bit] NOT NULL,

        [Comment] [nvarchar](255) NOT NULL,

        [IndexFragmentation] [float] NOT NULL,

        [RowModCtr] [bigint] NOT NULL,

        [SQLCommand] [nvarchar](max) NOT NULL,

    CONSTRAINT [PK__Maintena__3214EC074E078F4E] PRIMARY KEY CLUSTERED 

    (

        [Id] ASC

    )WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, FILLFACTOR = 90) ON [PRIMARY]

    ) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

    GO



    CREATE VIEW [dbo].[v_CommonStatsByDay]

    AS

    SELECT 

        CAST([RunDate] AS DATE) AS "День",

        COUNT(DISTINCT [TableName]) AS "Кол-во таблиц, для объектов которых выполнено обслуживание",

        COUNT(DISTINCT [IndexName]) AS "Количество индексов, для объектов которых выполнено обслуживание",

        SUM(CASE 

            WHEN [Operation] LIKE '%STAT%'

            THEN 1

            ELSE 0

        END) AS "Обновлено статистик",

        SUM(CASE 

            WHEN [Operation] LIKE '%INDEX%'

            THEN 1

            ELSE 0

        END) AS "Обслужено индексов"      

    FROM [SQLServerMaintenance].[dbo].[MaintenanceActionsLog]

    GROUP BY CAST([RunDate] AS DATE)

    GO



    CREATE TABLE [dbo].[ConnectionsStatistic](

        [id] [int] IDENTITY(1,1) NOT NULL,

        [Period] [datetime2](7) NOT NULL,

        [InstanceName] [nvarchar](255) NULL,

        [QueryText] [nvarchar](max) NULL,

        [RowCountSize] [bigint] NULL,

        [SessionId] [bigint] NULL,

        [Status] [nvarchar](255) NULL,

        [Command] [nvarchar](255) NULL,

        [CPU] [bigint] NULL,

        [TotalElapsedTime] [bigint] NULL,

        [StartTime] [datetime2](7) NULL,

        [DatabaseName] [nvarchar](255) NULL,

        [BlockingSessionId] [bigint] NULL,

        [WaitType] [nvarchar](255) NULL,

        [WaitTime] [bigint] NULL,

        [WaitResource] [nvarchar](255) NULL,

        [OpenTransactionCount] [bigint] NULL,

        [Reads] [bigint] NULL,

        [Writes] [bigint] NULL,

        [LogicalReads] [bigint] NULL,

        [GrantedQueryMemory] [bigint] NULL,

        [UserName] [nvarchar](255) NULL,

    CONSTRAINT [PK_ConnectionsStatistic] PRIMARY KEY CLUSTERED 

    (

        [id] ASC,

        [Period] ASC

    )WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, FILLFACTOR = 90) ON [PRIMARY]

    ) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

    GO



    CREATE TABLE [dbo].[DatabaseObjectsState](

        [Id] [bigint] IDENTITY(1,1) NOT NULL,

        [Period] [datetime2](0) NOT NULL,

        [DatabaseName] [nvarchar](150) NOT NULL,

        [TableName] [nvarchar](250) NOT NULL,

        [Object] [nvarchar](250) NOT NULL,

        [PageCount] [bigint] NOT NULL,

        [Rowmodctr] [bigint] NOT NULL,

        [AvgFragmentationPercent] [int] NOT NULL,

        [OnlineRebuildSupport] [int] NOT NULL,

        [Compression] [nvarchar](10) NULL,

        [PartitionCount] [bigint] NULL,

    CONSTRAINT [PK__DatabaseObjectsState__3214EC074E078F4E] PRIMARY KEY CLUSTERED 

    (

        [Id] ASC

    )WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, FILLFACTOR = 90) ON [PRIMARY]

    ) ON [PRIMARY]

    GO



    CREATE TABLE [dbo].[DatabasesTablesStatistic](

        [Period] [datetime2](7) NOT NULL,

        [DatabaseName] [nvarchar](255) NOT NULL,

        [SchemaName] [nvarchar](5) NOT NULL,

        [TableName] [nvarchar](255) NOT NULL,

        [RowCnt] [bigint] NOT NULL,

        [Reserved] [bigint] NOT NULL,

        [Data] [bigint] NOT NULL,

        [IndexSize] [bigint] NOT NULL,

        [Unused] [bigint] NOT NULL

    ) ON [PRIMARY]

    GO



    CREATE UNIQUE CLUSTERED INDEX [UK_DatabasesTablesStatistic_Period_DatabaseName_TableName] ON [dbo].[DatabasesTablesStatistic]

    (

        [Period] ASC,

        [DatabaseName] ASC,

        [SchemaName] ASC,

        [TableName] ASC

    )WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, SORT_IN_TEMPDB = OFF, IGNORE_DUP_KEY = OFF, DROP_EXISTING = OFF, ONLINE = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, FILLFACTOR = 90) ON [PRIMARY]

    GO



    CREATE TABLE [dbo].[MaintenanceIndexPriority](

        [ID] [int] IDENTITY(1,1) NOT NULL,

        [DatabaseName] [nvarchar](255) NOT NULL,

        [TableName] [nvarchar](255) NOT NULL,

        [IndexName] [nvarchar](255) NOT NULL,

        [Priority] [int] NOT NULL,

        [Exclude] [bit] NULL,

    CONSTRAINT [PK_MaintenanceIndexPriority] PRIMARY KEY CLUSTERED 

    (

        [ID] ASC

    )WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, FILLFACTOR = 90) ON [PRIMARY]

    ) ON [PRIMARY]

    GO



    CREATE NONCLUSTERED INDEX [UK_Table_Object_Period] ON [dbo].[DatabaseObjectsState]

    (

        [DatabaseName] ASC,

        [TableName] ASC,

        [Object] ASC,

        [Period] ASC

    )WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, SORT_IN_TEMPDB = OFF, DROP_EXISTING = OFF, ONLINE = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, FILLFACTOR = 90) ON [PRIMARY]

    GO



    CREATE NONCLUSTERED INDEX [UK_RunDate_Table_Index_Period_Operation] ON [dbo].[MaintenanceActionsLog]

    (

        [RunDate] ASC,

        [DatabaseName] ASC,

        [TableName] ASC,

        [IndexName] ASC

    )WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, SORT_IN_TEMPDB = OFF, DROP_EXISTING = OFF, ONLINE = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, FILLFACTOR = 90) ON [PRIMARY]

    GO



    CREATE FUNCTION [dbo].[fn_ResumableIndexMaintenanceAvailiable]()

    RETURNS bit

    AS

    BEGIN

        DECLARE @checkResult bit;



        SELECT

        -- Возобновляемые операции обслуживания индексов доступны со SQL Server 2017

        @checkResult = CASE

                            WHEN CAST(SUBSTRING(CONVERT(VARCHAR(128), SERVERPROPERTY('productversion')), 0,  3) AS INT) > 13

                            THEN 1

                            ELSE 0

                        END

        

        RETURN @checkResult



    END

    GO



    CREATE PROCEDURE [dbo].[sp_AdvancedPrint]

        @sql varchar(max)

    AS

    BEGIN

        declare

            @n int,

            @i int = 0,

            @s int = 0,

            @l int;



        set @n = ceiling(len(@sql) / 8000.0);



        while @i < @n

        begin

            set @l = 8000 - charindex(char(13), reverse(substring(@sql, @s, 8000)));

            print substring(@sql, @s, @l);

            set @i = @i + 1;

            set @s = @s + @l + 2;

        end



        return 0

    END

    GO



    CREATE PROCEDURE [dbo].[sp_add_maintenance_action_log]

        @TableName sysname,

        @IndexName sysname,

        @Operation nvarchar(100),

        @RunDate datetime2(0),

        @StartDate datetime2(0),

        @FinishDate datetime2(0),

        @DatabaseName sysname,

        @UseOnlineRebuild bit,

        @Comment nvarchar(255),

        @IndexFragmentation float,

        @RowModCtr bigint,

        @SQLCommand nvarchar(max),

        @MaintenanceActionLogId bigint OUTPUT

    AS

    BEGIN

        SET NOCOUNT ON;



        DECLARE @IdentityOutput TABLE ( Id bigint )



        SET @TableName = REPLACE(@TableName, '[', '')

        SET @TableName = REPLACE(@TableName, ']', '')

        SET @IndexName = REPLACE(@IndexName, '[', '')

        SET @IndexName = REPLACE(@IndexName, ']', '')

        SET @DatabaseName = REPLACE(@DatabaseName, '[', '')

        SET @DatabaseName = REPLACE(@DatabaseName, ']', '')



        SET @RowModCtr = ISNULL(@RowModCtr,0);

        

        SET @SQLCommand = LTRIM(RTRIM((REPLACE(REPLACE(@SQLCommand, CHAR(13), ''), CHAR(10), ''))));



        INSERT INTO [dbo].[MaintenanceActionsLog]

        (

            [Period]

            ,[TableName]

            ,[IndexName]

            ,[Operation]

            ,[RunDate]

            ,[StartDate]

            ,[FinishDate]

            ,[DatabaseName]

            ,[UseOnlineRebuild]

            ,[Comment]

            ,[IndexFragmentation]

            ,[RowModCtr]

            ,[SQLCommand]

        )

        OUTPUT inserted.Id into @IdentityOutput

        VALUES

        (

            GETDATE()

            ,@TableName

            ,@IndexName

            ,@Operation

            ,@RunDate

            ,@StartDate

            ,@FinishDate

            ,@DatabaseName

            ,@UseOnlineRebuild

            ,@Comment

            ,@IndexFragmentation

            ,@RowModCtr

            ,@SQLCommand

        )



        SET @MaintenanceActionLogId = (SELECT MAX(Id) FROM @IdentityOutput)



        RETURN 0

    END

    GO



    CREATE PROCEDURE [dbo].[sp_FillConnectionsStatistic]

        @monitoringDatabaseName sysname = 'SQLServerMaintenance'

    AS

    BEGIN

        SET NOCOUNT ON;



        DECLARE @cmd nvarchar(max);

        SET @cmd = 

    CAST('

    SET NOCOUNT ON;

    INSERT INTO [' AS nvarchar(max)) + CAST(@monitoringDatabaseName AS nvarchar(max)) + CAST('].[dbo].[ConnectionsStatistic]

            ([Period]

            ,[InstanceName]

            ,[QueryText]

            ,[RowCountSize]

            ,[SessionId]

            ,[Status]

            ,[Command]

            ,[CPU]

            ,[TotalElapsedTime]

            ,[StartTime]

            ,[DatabaseName]

            ,[BlockingSessionId]

            ,[WaitType]

            ,[WaitTime]

            ,[WaitResource]

            ,[OpenTransactionCount]

            ,[Reads]

            ,[Writes]

            ,[LogicalReads]

            ,[GrantedQueryMemory]

            ,[UserName]

    )

    SELECT 

        GetDate() AS [Period],

        @@servername AS [HostName],

        sqltext.TEXT AS [QueryText],

        req.row_count AS [RowCountSize],

        req.session_id AS [SessionId],

        req.status AS [Status],

        req.command AS [Command],

        req.cpu_time AS [CPU],

        req.total_elapsed_time AS [TotalElapsedTime],

        req.start_time AS [StartTime],

        DB_NAME(req.database_id) AS [DatabaseName],

        req.blocking_session_id AS [BlockingSessionId],

        req.wait_type AS [WaitType],

        req.wait_time AS [WaitTime],

        req.wait_resource AS [WaitResource],

        req.open_transaction_count AS [OpenTransactionCount],

        req.reads as [Reads],

        req.reads as [Writes],

        req.logical_reads as [LogicalReads],

        req.granted_query_memory as [GrantedQueryMemory],

        SUSER_NAME(user_id) AS [UserName]

    FROM sys.dm_exec_requests req

        OUTER APPLY sys.dm_exec_sql_text(sql_handle) AS sqltext

    ' AS nvarchar(max));



        EXECUTE sp_executesql @cmd;



        RETURN 0

    END

    GO



    CREATE PROCEDURE [dbo].[sp_FillDatabaseObjectsState]

        @databaseName sysname,

        @monitoringDatabaseName sysname = 'SQLServerMaintenance'

    AS

    BEGIN

        SET NOCOUNT ON;



        DECLARE @msg nvarchar(max);



        IF DB_ID(@databaseName) IS NULL

        BEGIN

            SET @msg = 'Database ' + @databaseName + ' is not exists.';

            THROW 51000, @msg, 1;

            RETURN -1;

        END



        DECLARE @cmd nvarchar(max);

        SET @cmd = 

    CAST('USE [' AS nvarchar(max)) + CAST(@databasename AS nvarchar(max)) + CAST(']

    SET NOCOUNT ON;

    INSERT INTO [' AS nvarchar(max)) + CAST(@monitoringDatabaseName AS nvarchar(max)) + CAST('].[dbo].[DatabaseObjectsState](

        [Period]

        ,[DatabaseName]

        ,[TableName]

        ,[Object]

        ,[PageCount]

        ,[Rowmodctr]

        ,[AvgFragmentationPercent]

        ,[OnlineRebuildSupport]

        ,[Compression]

        ,[PartitionCount]

    )

    SELECT

    GETDATE() AS [Period],

    ''' AS nvarchar(max)) + CAST(@databasename AS nvarchar(max)) + CAST(''' AS [DatabaseName],

    OBJECT_NAME(dt.[object_id]) AS [Table], 

    ind.name AS [Object],

    MAX(CAST([page_count] AS BIGINT)) AS [page_count], 

    SUM(CAST([si].[rowmodctr] AS BIGINT)) AS [rowmodctr],

    MAX([avg_fragmentation_in_percent]) AS [frag], 

    MIN(CASE WHEN objBadTypes.IndexObjectId IS NULL THEN 1 ELSE 0 END) AS [OnlineRebuildSupport],

    MAX(p.data_compression_desc) AS [Compression],

    MAX(p_count.[PartitionCount]) AS [PartitionCount]

    FROM 

    sys.dm_db_index_physical_stats (

        DB_ID(), 

        NULL, 

        NULL, 

        NULL, 

        N''LIMITED''

    ) dt 

    LEFT JOIN sys.partitions p

        ON dt.object_id = p.object_id and p.partition_number = 1

    LEFT JOIN sys.sysindexes si ON dt.object_id = si.id 

    LEFT JOIN (

            SELECT 

            t.object_id AS [TableObjectId], 

            ind.index_id AS [IndexObjectId]

            FROM 

            sys.indexes ind 

            INNER JOIN sys.index_columns ic ON ind.object_id = ic.object_id 

            and ind.index_id = ic.index_id 

            INNER JOIN sys.columns col ON ic.object_id = col.object_id 

            and ic.column_id = col.column_id 

            INNER JOIN sys.tables t ON ind.object_id = t.object_id 

            LEFT JOIN INFORMATION_SCHEMA.COLUMNS tbsc ON t.schema_id = SCHEMA_ID(tbsc.TABLE_SCHEMA) 

            AND t.name = tbsc.TABLE_NAME 

            LEFT JOIN sys.types tps ON col.system_type_id = tps.system_type_id 

            AND col.user_type_id = tps.user_type_id 

            WHERE 

            t.is_ms_shipped = 0 

            AND CASE WHEN ind.type_desc = ''CLUSTERED'' THEN CASE WHEN tbsc.DATA_TYPE IN (

                ''text'', ''ntext'', ''image'', ''FILESTREAM''

            ) THEN 1 ELSE 0 END ELSE CASE WHEN tps.[name] IN (

                ''text'', ''ntext'', ''image'', ''FILESTREAM''

            ) THEN 1 ELSE 0 END END > 0 

            GROUP BY 

            t.object_id, 

            ind.index_id

        ) AS objBadTypes ON objBadTypes.TableObjectId = dt.object_id 

        AND objBadTypes.IndexObjectId = dt.index_id

        LEFT JOIN sys.indexes AS [ind]

            ON dt.object_id = [ind].object_id AND dt.index_id = [ind].[index_id]

        LEFT JOIN (

            SELECT

                object_id,

                index_id,

                COUNT(DISTINCT partition_number) AS [PartitionCount]

            FROM sys.partitions p

            GROUP BY object_id, index_id

        ) p_count

        ON dt.object_id = p_count.object_id AND dt.index_id = p_count.index_id

    WHERE 

    [rowmodctr] IS NOT NULL -- Исключаем служебные объекты, по которым нет изменений

    AND dt.[index_id] > 0 -- игнорируем кучи (heap)

    GROUP BY

        dt.[object_id], 

        dt.[index_id],

        ind.[name],

        dt.[partition_number]

    ' AS nvarchar(max));



        EXECUTE sp_executesql @cmd;



        RETURN 0

    END

    GO







    CREATE PROCEDURE [dbo].[sp_GetCurrentResumableIndexRebuilds] 

        @databaseName sysname

    AS

    BEGIN

        SET NOCOUNT ON;



        DECLARE @msg nvarchar(max);



        IF DB_ID(@databaseName) IS NULL

        BEGIN

            SET @msg = 'Database ' + @databaseName + ' is not exists.';

            THROW 51000, @msg, 1;

            RETURN -1;

        END



        DECLARE @LOCAL_ResumableIndexRebuilds TABLE

        (

            [object_id] int, 

            [index_id] int, 

            [name] sysname, 

            [sql_text] nvarchar(max), 

            [partition_number] int,

            [state] tinyint, 

            [state_desc] nvarchar(60),

            [start_time] datetime,

            [last_pause_time] datetime,

            [total_execution_time] int,

            [percent_complete] real,

            [page_count] bigint

        );



        IF([dbo].[fn_ResumableIndexMaintenanceAvailiable]() > 0)

        BEGIN

            DECLARE @cmd nvarchar(max);

            SET @cmd = CAST('

            USE [' AS nvarchar(max)) + CAST(@databaseName AS nvarchar(max)) + CAST(']

            SET NOCOUNT ON;

            SELECT

                [object_id], 

                [index_id], 

                [name], 

                [sql_text], 

                [partition_number],

                [state], 

                [state_desc],

                [start_time],

                [last_pause_time],

                [total_execution_time],

                [percent_complete],

                [page_count]

            FROM sys.index_resumable_operations;

            ' AS nvarchar(max));

            INSERT @LOCAL_ResumableIndexRebuilds

            EXECUTE sp_executesql @cmd;

        END



        SELECT

            [object_id], 

            [index_id], 

            [name], 

            [sql_text], 

            [partition_number],

            [state], 

            [state_desc],

            [start_time],

            [last_pause_time],

            [total_execution_time],

            [percent_complete],

            [page_count]

        FROM @LOCAL_ResumableIndexRebuilds

    END

    GO



    CREATE PROCEDURE [dbo].[sp_IndexMaintenance]

        @databaseName sysname,

        @timeFrom TIME = '00:00:00',

        @timeTo TIME = '23:59:59',

        @fragmentationPercentMinForMaintenance FLOAT = 10.0,

        @fragmentationPercentForRebuild FLOAT = 30.0,

        @maxDop int = 8,

        @minIndexSizePages int = 0,

        @maxIndexSizePages int = 0,

        @useOnlineIndexRebuild int = 0,

        @useResumableIndexRebuildIfAvailable int = 0,

        @maxIndexSizeForReorganizingPages int = 6553600,

        @useMonitoringDatabase bit = 1,

        @monitoringDatabaseName sysname = 'SQLServerMaintenance',

        @usePreparedInformationAboutObjectsStateIfExists bit = 0,

        @ConditionTableName nvarchar(max) = 'LIKE ''%''',

        @ConditionIndexName nvarchar(max) = 'LIKE ''%''',

        @onlineRebuildAbortAfterWaitMode int = 1,

        @onlineRebuildWaitMinutes int = 5,

        @maxTransactionLogSizeUsagePercent int = 100,  

        @maxTransactionLogSizeMB bigint = 0

    AS

    BEGIN

        SET NOCOUNT ON;

    

        DECLARE @msg nvarchar(max),

                @abortAfterWaitOnlineRebuil nvarchar(25),

                @currentTransactionLogSizeUsagePercent int,

                @currentTransactionLogSizeMB int,

                @timeNow TIME = CAST(GETDATE() AS TIME),

                @useResumableIndexRebuild bit,

                @RunDate datetime = GETDATE(),

                @StartDate datetime,

                @FinishDate datetime,

                @MaintenanceActionLogId bigint,

                -- Список исключенных из обслуживания индексов.

                -- Например, если они были обслужены через механизм возобновляемых перестроений,

                -- еще до запуска основного обслуживания

                @excludeIndexes XML;

    

        IF(@onlineRebuildAbortAfterWaitMode = 0)

        BEGIN

            SET @abortAfterWaitOnlineRebuil = 'NONE'

        END ELSE IF(@onlineRebuildAbortAfterWaitMode = 1)

        BEGIN

            SET @abortAfterWaitOnlineRebuil = 'SELF'

        END ELSE IF(@onlineRebuildAbortAfterWaitMode = 2)

        BEGIN

            SET @abortAfterWaitOnlineRebuil = 'BLOCKERS'

        END ELSE

        BEGIN

            SET @abortAfterWaitOnlineRebuil = 'NONE'

        END

    

        IF DB_ID(@databaseName) IS NULL

        BEGIN

            SET @msg = 'Database ' + @databaseName + ' is not exists.';

            THROW 51000, @msg, 1;

            RETURN -1;

        END

    

        -- Информация о размере лога транзакций

        IF OBJECT_ID('tempdb..#tranLogInfo') IS NOT NULL

            DROP TABLE #tranLogInfo;

        CREATE TABLE #tranLogInfo

        (

            servername varchar(250) not null default @@servername,

            dbname varchar(250),

            logsize real,

            logspace real,

            stat int

        )

    

        -- Проверка процента занятого места в логе транзакций

        TRUNCATE TABLE #tranLogInfo;

        INSERT INTO #tranLogInfo (dbname,logsize,logspace,stat) exec('dbcc sqlperf(logspace)')

        SELECT

            @currentTransactionLogSizeUsagePercent = logspace,

            @currentTransactionLogSizeMB = logsize * (logspace / 100)

        FROM #tranLogInfo WHERE dbname = @databaseName

        IF(@currentTransactionLogSizeUsagePercent >= @maxTransactionLogSizeUsagePercent)

        BEGIN

            -- Процент занятого места в файлах лога транзакций превышает указанный порог

            RETURN 0;

        END

        IF(@maxTransactionLogSizeMB > 0 AND @currentTransactionLogSizeMB > @maxTransactionLogSizeMB)

        BEGIN

            -- Размер занятого места в файлах лога транзакций превышает указанный порог в МБ

            RETURN 0;

        END

    

        -- Возобновляемое перестроение индексов

        DECLARE @LOCAL_ResumableIndexRebuilds TABLE

        (

            [object_id] int, 

            [object_name] nvarchar(255), 

            [index_id] int, 

            [name] sysname, 

            [sql_text] nvarchar(max), 

            [partition_number] int,

            [state] tinyint, 

            [state_desc] nvarchar(60),

            [start_time] datetime,

            [last_pause_time] datetime,

            [total_execution_time] int,

            [percent_complete] real,

            [page_count] bigint,

            [ResumeCmd] nvarchar(max)

        );

        -- Флаг использования возобновляемого перестроения индексов

        SET @useResumableIndexRebuild = 

            CASE

                WHEN (@useResumableIndexRebuildIfAvailable > 0)	-- Передан флаг использования возобновляемого перестроения

                -- Возобновляемое перестроение доступно для версии SQL Server

                AND [dbo].[fn_ResumableIndexMaintenanceAvailiable]() > 0

                -- Включено использование онлайн-перестроения для скрипта

                AND (@useOnlineIndexRebuild = 1 -- Только онлайн-перестроение

                    OR @useOnlineIndexRebuild = 3) -- Для объектов где оно возможно

                THEN 1

                ELSE 0

            END;

        IF(@useResumableIndexRebuild > 0)

        BEGIN

            DECLARE @cmdResumableIndexRebuild nvarchar(max);

            SET @cmdResumableIndexRebuild = CAST('

            USE [' AS nvarchar(max)) + CAST(@databaseName AS nvarchar(max)) + CAST(']

            SET NOCOUNT ON;

            SELECT

                [object_id],

                OBJECT_NAME([object_id]) AS [TableName],

                [index_id], 

                [name], 

                [sql_text], 

                [partition_number],

                [state], 

                [state_desc],

                [start_time],

                [last_pause_time],

                [total_execution_time],

                [percent_complete],

                [page_count],

                ''ALTER INDEX ['' + [name] + ''] ON ['' + OBJECT_SCHEMA_NAME([object_id]) + ''].['' + OBJECT_NAME([object_id]) + ''] RESUME'' AS [ResumeCmd]

            FROM sys.index_resumable_operations

            WHERE OBJECT_NAME([object_id]) ' AS nvarchar(max)) + CAST(@ConditionTableName  AS nvarchar(max)) + CAST('

                AND [name] ' AS nvarchar(max)) + CAST(@ConditionIndexName  AS nvarchar(max)) + CAST(';

            ' AS nvarchar(max));

            INSERT @LOCAL_ResumableIndexRebuilds

            EXECUTE sp_executesql @cmdResumableIndexRebuild;



            DECLARE 

                @objectNameResumeRebuildForIndex nvarchar(255), 

                @indexNameResumeRebuildForIndex nvarchar(255), 

                @cmdResumeRebuildForIndex nvarchar(max);

            DECLARE resumableIndexRebuild_cursor CURSOR FOR				

            SELECT 

                [object_name],

                [name],

                [ResumeCmd]

            FROM @LOCAL_ResumableIndexRebuilds

            ORDER BY start_time;

            OPEN resumableIndexRebuild_cursor;		

            FETCH NEXT FROM resumableIndexRebuild_cursor 

            INTO @objectNameResumeRebuildForIndex, @indexNameResumeRebuildForIndex, @cmdResumeRebuildForIndex;

            WHILE @@FETCH_STATUS = 0  

            BEGIN

                -- Проверка доступен ли запуск обслуживания в текущее время

                SET @timeNow = CAST(GETDATE() AS TIME);

                IF (@timeTo >= @timeFrom) BEGIN

                    IF(NOT (@timeFrom <= @timeNow AND @timeTo >= @timeNow))

                        RETURN;

                    END ELSE BEGIN

                        IF(NOT ((@timeFrom <= @timeNow AND '23:59:59' >= @timeNow)

                            OR (@timeTo >= @timeNow AND '00:00:00' <= @timeNow))) 

                                RETURN;

                END



                -- Проверки использования лога транзакций

                -- Проверка процента занятого места в логе транзакций

                TRUNCATE TABLE #tranLogInfo;

                INSERT INTO #tranLogInfo (dbname,logsize,logspace,stat) exec('dbcc sqlperf(logspace)')

                SELECT

                    @currentTransactionLogSizeUsagePercent = logspace,

                    @currentTransactionLogSizeMB = logsize * (logspace / 100)

                FROM #tranLogInfo WHERE dbname = @databaseName

                IF(@currentTransactionLogSizeUsagePercent >= @maxTransactionLogSizeUsagePercent)

                BEGIN

                    -- Процент занятого места в файлах лога транзакций превышает указанный порог

                    RETURN 0;

                END

                IF(@maxTransactionLogSizeMB > 0 AND @currentTransactionLogSizeMB > @maxTransactionLogSizeMB)

                BEGIN

                    -- Размер занятого места в файлах лога транзакций превышает указанный порог в МБ

                    RETURN 0;

                END

                

                BEGIN TRY

                    -- Сохраняем предварительную информацию об операции обслуживания без даты завершения				

                    IF(@useMonitoringDatabase = 1)

                    BEGIN

                        SET @StartDate = GETDATE();

                        EXECUTE [dbo].[sp_add_maintenance_action_log]

                            @objectNameResumeRebuildForIndex,

                            @indexNameResumeRebuildForIndex,

                            'REBUILD INDEX RESUME',

                            @RunDate,

                            @StartDate,

                            null,

                            @databaseName,

                            1, -- @UseOnlineRebuild

                            '',

                            0, -- @AvgFragmentationPercent

                            0, -- @RowModCtr

                            @cmdResumeRebuildForIndex,

                            @MaintenanceActionLogId OUTPUT;

                    END



                    SET @cmdResumeRebuildForIndex = CAST('

                        USE [' AS nvarchar(max)) + CAST(@databaseName AS nvarchar(max)) + CAST(']

                        SET NOCOUNT ON;

                        ' + CAST(@cmdResumeRebuildForIndex as nvarchar(max)) + '

                    ' AS nvarchar(max));

                    EXECUTE sp_executesql @cmdResumeRebuildForIndex;

                    SET @FinishDate = GetDate();



                    -- Устанавливаем фактическую дату завершения операции

                    IF(@useMonitoringDatabase = 1)

                    BEGIN

                        EXECUTE [dbo].[sp_set_maintenance_action_log_finish_date]

                            @MaintenanceActionLogId,

                            @FinishDate;

                    END				

                END TRY

                BEGIN CATCH		

                    IF(@MaintenanceActionLogId <> 0)

                    BEGIN

                        SET @msg = 'Error: ' + CAST(Error_message() AS NVARCHAR(500)) + ', Code: ' + CAST(Error_Number() AS NVARCHAR(500)) + ', Line: ' + CAST(Error_Line() AS NVARCHAR(500))

                        -- Устанавливаем текст ошибки при обслуживании индекса

                        -- Дата завершения при этом остается незаполненной

                        EXECUTE [dbo].[sp_set_maintenance_action_log_finish_date]

                            @MaintenanceActionLogId,

                            @FinishDate,

                            @msg;          

                    END

                END CATCH



                FETCH NEXT FROM resumableIndexRebuild_cursor 

                INTO @objectNameResumeRebuildForIndex, @indexNameResumeRebuildForIndex, @cmdResumeRebuildForIndex;

            END

            CLOSE resumableIndexRebuild_cursor;  

            DEALLOCATE resumableIndexRebuild_cursor;

        END

        -- Сохраняем список индексов, для которых имеются ожидающие операции перестроения

        -- Они будут исключены из основного обслуживания

        SET @excludeIndexes = (SELECT

            [name]

        FROM @LOCAL_ResumableIndexRebuilds

        FOR XML RAW, ROOT('root'));

        

        --PRINT 'Прервано для отладки'

        --RETURN 0;

    

        DECLARE @cmd nvarchar(max);

        SET @cmd =

    CAST('USE [' AS nvarchar(max)) + CAST(@databasename AS nvarchar(max)) + CAST(']

    SET NOCOUNT ON;

    DECLARE

        -- Текущее время

        @timeNow TIME = CAST(GETDATE() AS TIME),

        -- Текущий процент использования файла лога транзакций

        @currentTransactionLogSizeUsagePercent int,

        @currentTransactionLogSizeMB bigint;

    

    -- Проверка доступен ли запуск обслуживания в текущее время

    IF (@timeTo >= @timeFrom) BEGIN

        IF(NOT (@timeFrom <= @timeNow AND @timeTo >= @timeNow))

            RETURN;

        END ELSE BEGIN

            IF(NOT ((@timeFrom <= @timeNow AND ''23:59:59'' >= @timeNow)

                OR (@timeTo >= @timeNow AND ''00:00:00'' <= @timeNow))) 

                    RETURN;

    END

    

    -- Служебные переменные

    DECLARE

        @DBID SMALLINT = DB_ID()

        ,@DBNAME sysname = DB_NAME()

        ,@SchemaName SYSNAME

        ,@ObjectName SYSNAME

        ,@ObjectID INT

        ,@Priority INT

        ,@IndexID INT

        ,@IndexName SYSNAME

        ,@PartitionNum BIGINT

        ,@PartitionCount BIGINT

        ,@frag FLOAT

        ,@Command NVARCHAR(max)

        ,@CommandSpecial NVARCHAR(max)

        ,@Operation NVARCHAR(128)

        ,@RowModCtr BIGINT

        ,@AvgFragmentationPercent float

        ,@PageCount BIGINT

        ,@SQL nvarchar(max)

        ,@SQLSpecial nvarchar(max)

        ,@OnlineRebuildSupport int

        ,@UseOnlineRebuild int

        ,@StartDate datetime

        ,@FinishDate datetime

        ,@RunDate datetime = GETDATE()

        ,@MaintenanceActionLogId bigint;

    

    IF OBJECT_ID(''tempdb..#MaintenanceCommands'') IS NOT NULL

        DROP TABLE #MaintenanceCommands;

    IF OBJECT_ID(''tempdb..#MaintenanceCommandsTemp'') IS NOT NULL

        DROP TABLE #MaintenanceCommandsTemp;

    

    CREATE TABLE #MaintenanceCommands

    (

        [Command] nvarchar(max),

        [CommandSpecial] nvarchar(max),

        [Table] nvarchar(250),

        [Object] nvarchar(250),

        [page_count] BIGINT,

        [Rowmodctr] BIGINT,

        [Avg_fragmentation_in_percent] INT,

        [Operation] nvarchar(max),

        [Priority] INT,

        [OnlineRebuildSupport] INT,

        [UseOnlineRebuild] INT,

        [PartitionCount] BIGINT

    )

    

    IF OBJECT_ID(''tempdb..#tranLogInfo'') IS NOT NULL

        DROP TABLE #tranLogInfo;

    CREATE TABLE #tranLogInfo

    (

        servername varchar(250) not null default @@servername,

        dbname varchar(250),

        logsize real,

        logspace real,

        stat int

    )

    

    DECLARE @usedCacheAboutObjectsState bit = 0;

    

    IF @usePreparedInformationAboutObjectsStateIfExists = 1

        AND EXISTS(SELECT *

            FROM [' AS nvarchar(max)) + CAST(@monitoringDatabaseName  AS nvarchar(max)) + CAST('].[dbo].[DatabaseObjectsState]

            WHERE [DatabaseName] = @databaseName

                -- Информация должна быть собрана в рамках 12 часов от текущего запуска

                AND [Period] BETWEEN DATEADD(hour, -12, @RunDate) AND DATEADD(hour, 12, @RunDate))

    BEGIN  

        -- Получаем информацию через подготовленный сбор

        SET @usedCacheAboutObjectsState = 1;

    

        SELECT

        OBJECT_ID(dt.[TableName]) AS [objectid]

        ,ind.index_id as [indexid]

        ,1 AS [partitionnum]

        ,[AvgFragmentationPercent] AS [frag]

        ,[PageCount] AS [page_count]

        ,[Rowmodctr] AS [rowmodctr]

        ,ISNULL(prt.[Priority], 999) AS [Priority]

        ,ISNULL(prt.[Exclude], 0) AS Exclude

        ,dt.[OnlineRebuildSupport] AS [OnlineRebuildSupport]

        ,dt.[PartitionCount] AS [PartitionCount]

        INTO #MaintenanceCommandsTempCached

        FROM [' AS nvarchar(max)) + CAST(@monitoringDatabaseName  AS nvarchar(max)) + CAST('].[dbo].[DatabaseObjectsState] dt

            LEFT JOIN sys.indexes ind

                ON OBJECT_ID(dt.[TableName]) = ind.object_id

                    AND dt.[Object] = ind.[name]

            LEFT JOIN [' AS nvarchar(max)) + CAST(@monitoringDatabaseName  AS nvarchar(max)) + CAST('].[dbo].[MaintenanceIndexPriority] AS [prt]

                ON dt.DatabaseName = prt.[DatabaseName]

                    AND dt.TableName = prt.TableName

                    AND dt.[Object] = prt.Indexname

        WHERE dt.[DatabaseName] = @databaseName

            AND [Period] BETWEEN DATEADD(hour, -12, @RunDate) AND DATEADD(hour, 12, @RunDate)

            -- Записи от последнего получения данных за прошедшие 12 часов

            AND [Period] IN (

                SELECT MAX([Period])

                FROM [' AS nvarchar(max)) + CAST(@monitoringDatabaseName  AS nvarchar(max)) + CAST('].[dbo].[DatabaseObjectsState]

                WHERE [DatabaseName] = @databaseName

                    AND dt.[Period] BETWEEN DATEADD(hour, -12, @RunDate) AND DATEADD(hour, 12, @RunDate))

            AND [AvgFragmentationPercent] > @fragmentationPercentMinForMaintenance

            AND [PageCount] > 25 -- игнорируем небольшие таблицы

            -- Фильтр по мин. размеру индекса

            AND (@minIndexSizePages = 0 OR [PageCount] >= @minIndexSizePages)

            -- Фильтр по макс. размеру индекса

            AND (@maxIndexSizePages = 0 OR [PageCount] <= @maxIndexSizePages)

            -- Убираем обработку индексов, исключенных из обслуживания

            AND ISNULL(prt.[Exclude], 0) = 0

            -- Отбор по имени таблцы

            AND dt.[TableName] ' AS nvarchar(max)) + CAST(@ConditionTableName  AS nvarchar(max)) + CAST('

            AND dt.[Object] ' AS nvarchar(max)) + CAST(@ConditionIndexName  AS nvarchar(max)) + CAST('

            AND NOT dt.[Object] IN (

                SELECT 

                    XC.value(''@name'', ''nvarchar(255)'') AS [IndexName]

                FROM @excludeIndexes.nodes(''/root/row'') AS XT(XC)

            )

    END ELSE

    BEGIN

        -- Получаем информацию через анализ базы данных

        SELECT

            dt.[object_id] AS [objectid],

            dt.index_id AS [indexid],

            [partition_number] AS [partitionnum],

            MAX([avg_fragmentation_in_percent]) AS [frag],

            MAX(CAST([page_count] AS BIGINT)) AS [page_count],

            SUM(CAST([si].[rowmodctr] AS BIGINT)) AS [rowmodctr],

            MAX(

                ISNULL(prt.[Priority], 999)

            ) AS [Priority],

            MAX(

                CAST(ISNULL(prt.[Exclude], 0) AS INT)

            ) AS [Exclude],

            MIN(CASE WHEN objBadTypes.IndexObjectId IS NULL THEN 1 ELSE 0 END) AS [OnlineRebuildSupport],

            MAX(p_count.[PartitionCount]) AS [PartitionCount]

        INTO #MaintenanceCommandsTemp

        FROM

            sys.dm_db_index_physical_stats (

                DB_ID(),

                NULL,

                NULL,

                NULL,

                N''LIMITED''

            ) dt

            LEFT JOIN sys.sysindexes si ON dt.object_id = si.id

            LEFT JOIN (

                SELECT

                t.object_id AS [TableObjectId],

                ind.index_id AS [IndexObjectId]

                FROM

                sys.indexes ind

                INNER JOIN sys.index_columns ic ON ind.object_id = ic.object_id

                and ind.index_id = ic.index_id

                INNER JOIN sys.columns col ON ic.object_id = col.object_id

                and ic.column_id = col.column_id

                INNER JOIN sys.tables t ON ind.object_id = t.object_id

                LEFT JOIN INFORMATION_SCHEMA.COLUMNS tbsc ON t.schema_id = SCHEMA_ID(tbsc.TABLE_SCHEMA)

                AND t.name = tbsc.TABLE_NAME

                LEFT JOIN sys.types tps ON col.system_type_id = tps.system_type_id

                AND col.user_type_id = tps.user_type_id

                WHERE

                t.is_ms_shipped = 0

                AND CASE WHEN ind.type_desc = ''CLUSTERED'' THEN CASE WHEN tbsc.DATA_TYPE IN (

                    ''text'', ''ntext'', ''image'', ''FILESTREAM''

                ) THEN 1 ELSE 0 END ELSE CASE WHEN tps.[name] IN (

                    ''text'', ''ntext'', ''image'', ''FILESTREAM''

                ) THEN 1 ELSE 0 END END > 0

                GROUP BY

                t.object_id,

                ind.index_id

            ) AS objBadTypes ON objBadTypes.TableObjectId = dt.object_id

            AND objBadTypes.IndexObjectId = dt.index_id

            LEFT JOIN (

                SELECT

                i.[object_id],

                i.[index_id],

                os.[Priority] AS [Priority],

                os.[Exclude] AS [Exclude]

                FROM sys.indexes i

                    left join [' AS nvarchar(max)) + CAST(@monitoringDatabaseName  AS nvarchar(max)) + CAST('].[dbo].[MaintenanceIndexPriority] os

                    ON i.object_id = OBJECT_ID(os.TableName)

                        AND i.Name = os.IndexName

                WHERE os.Id IS NOT NULL

                    and os.DatabaseName = ''' AS nvarchar(max)) + CAST(@databaseName AS nvarchar(max)) + CAST('''

                ) prt ON si.id = prt.[object_id]

            AND dt.[index_id] = prt.[index_id]

            LEFT JOIN (

                SELECT

                    object_id,

                    index_id,

                    COUNT(DISTINCT partition_number) AS [PartitionCount]

                FROM sys.partitions p

                GROUP BY object_id, index_id

            ) p_count

            ON dt.object_id = p_count.object_id AND dt.index_id = p_count.index_id

        WHERE

            [rowmodctr] IS NOT NULL -- Исключаем служебные объекты, по которым нет изменений

            AND [avg_fragmentation_in_percent] > @fragmentationPercentMinForMaintenance

            AND dt.[index_id] > 0 -- игнорируем кучи (heap)

            AND [page_count] > 25 -- игнорируем небольшие таблицы

            -- Фильтр по мин. размеру индекса

            AND (@minIndexSizePages = 0 OR [page_count] >= @minIndexSizePages)

            -- Фильтр по макс. размеру индекса

            AND (@maxIndexSizePages = 0 OR [page_count] <= @maxIndexSizePages)

            -- Убираем обработку индексов, исключенных из обслуживания

            AND ISNULL(prt.[Exclude], 0) = 0

            -- Отбор по имени таблцы

            AND OBJECT_NAME(dt.[object_id]) ' AS nvarchar(max)) + CAST(@ConditionTableName  AS nvarchar(max)) + CAST('

            AND si.[name] ' AS nvarchar(max)) + CAST(@ConditionIndexName  AS nvarchar(max)) + CAST('

            AND NOT si.[name] IN (

                SELECT 

                    XC.value(''@name'', ''nvarchar(255)'') AS [IndexName]

                FROM @excludeIndexes.nodes(''/root/row'') AS XT(XC)

            )

        GROUP BY

            dt.[object_id],

            dt.[index_id],

            [partition_number];

    END

    

    IF(@usedCacheAboutObjectsState = 1)

    BEGIN

        DECLARE partitions CURSOR FOR

        SELECT [objectid], [indexid], [partitionnum], [frag], [page_count], [rowmodctr], [Priority], [OnlineRebuildSupport], [PartitionCount]

        FROM #MaintenanceCommandsTempCached;

    END ELSE

    BEGIN

        DECLARE partitions CURSOR FOR

        SELECT [objectid], [indexid], [partitionnum], [frag], [page_count], [rowmodctr], [Priority], [OnlineRebuildSupport], [PartitionCount]

        FROM #MaintenanceCommandsTemp;

    END

    

    OPEN partitions;

    WHILE (1=1)

    BEGIN

        FETCH NEXT FROM partitions INTO @ObjectID, @IndexID, @PartitionNum, @frag, @PageCount, @RowModCtr, @Priority, @OnlineRebuildSupport, @PartitionCount;

        IF @@FETCH_STATUS < 0 BREAK;

        

        SELECT @ObjectName = QUOTENAME([o].[name]), @SchemaName = QUOTENAME([s].[name])

        FROM sys.objects AS o

            JOIN sys.schemas AS s ON [s].[schema_id] = [o].[schema_id]

        WHERE [o].[object_id] = @ObjectID;

        SELECT @IndexName = QUOTENAME(name)

        FROM sys.indexes

        WHERE [object_id] = @ObjectID AND [index_id] = @IndexID;

        

        SET @CommandSpecial = '''';

        SET @Command = '''';

        -- Реорганизация индекса

        IF @Priority > 10 -- Приоритет обслуживания не большой

            AND @frag <= @fragmentationPercentForRebuild -- Процент фрагментации небольшой

            AND (@maxIndexSizeForReorganizingPages = 0 OR @PageCount <= @maxIndexSizeForReorganizingPages) BEGIN -- Таблица меньше 50 ГБ

            SET @Command = N''ALTER INDEX '' + @IndexName + N'' ON '' + @SchemaName + N''.'' + @ObjectName + N'' REORGANIZE'';

            SET @Operation = ''REORGANIZE INDEX''

        END ELSE IF(@useOnlineIndexRebuild = 0) -- Не использовать онлайн-перестроение

        BEGIN

            SET @Command = N''ALTER INDEX '' + @IndexName + N'' ON '' + @SchemaName + N''.'' + @ObjectName

                + N'' REBUILD WITH (MAXDOP='' + CAST(@MaxDop AS nvarchar(10)) + '')'';

            SET @Operation = ''REBUILD INDEX''

        END ELSE IF (@useOnlineIndexRebuild = 1 AND @OnlineRebuildSupport = 1) -- Только с поддержкой онлайн перестроения

        BEGIN

            SET @CommandSpecial = N''ALTER INDEX '' + @IndexName + N'' ON '' + @SchemaName + N''.'' + @ObjectName

                + N'' REBUILD WITH (MAXDOP='' + CAST(@MaxDop AS nvarchar(10)) + '','' 

                + (CASE WHEN @useResumableIndexRebuild > 0 THEN '' RESUMABLE = ON, '' ELSE '''' END) 

                + '' ONLINE = ON (WAIT_AT_LOW_PRIORITY ( MAX_DURATION = ' AS nvarchar(max)) + CAST(@onlineRebuildWaitMinutes  AS nvarchar(max)) + CAST(' MINUTES, ABORT_AFTER_WAIT = ' AS nvarchar(max)) + CAST(@abortAfterWaitOnlineRebuil  AS nvarchar(max)) + CAST(')))'';

            SET @Operation = ''REBUILD INDEX''

        END ELSE IF(@useOnlineIndexRebuild = 2 AND @OnlineRebuildSupport = 0) -- Только без поддержки

        BEGIN

            SET @Command = N''ALTER INDEX '' + @IndexName + N'' ON '' + @SchemaName + N''.'' + @ObjectName

                + N'' REBUILD WITH (MAXDOP='' + CAST(@MaxDop AS nvarchar(10)) + '')'';

            SET @Operation = ''REBUILD INDEX''

        END ELSE IF(@useOnlineIndexRebuild = 3) -- Использовать онлайн перестроение где возможно

        BEGIN

            if(@OnlineRebuildSupport = 1)

            BEGIN

                SET @CommandSpecial = N''ALTER INDEX '' + @IndexName + N'' ON '' + @SchemaName + N''.'' + @ObjectName

                    + N'' REBUILD WITH (MAXDOP='' + CAST(@MaxDop AS nvarchar(10)) + '',ONLINE = ON (WAIT_AT_LOW_PRIORITY ( MAX_DURATION = ' AS nvarchar(max)) + CAST(@onlineRebuildWaitMinutes  AS nvarchar(max)) + CAST(' MINUTES, ABORT_AFTER_WAIT = ' AS nvarchar(max)) + CAST(@abortAfterWaitOnlineRebuil  AS nvarchar(max)) + CAST(')))'';        

            END ELSE

            BEGIN

                SET @Command = N''ALTER INDEX '' + @IndexName + N'' ON '' + @SchemaName + N''.'' + @ObjectName

                    + N'' REBUILD WITH (MAXDOP='' + CAST(@MaxDop AS nvarchar(10)) + '')'';

            END

            SET @Operation = ''REBUILD INDEX''

        END

        IF (@PartitionCount > 1 AND @Command <> '''')

            SET @Command = @Command + N'' PARTITION='' + CAST(@PartitionNum AS nvarchar(10));

    

        SET @Command = LTRIM(RTRIM(@Command));

        SET @CommandSpecial = LTRIM(RTRIM(@CommandSpecial));

        IF(LEN(@Command) > 0 OR LEN(@CommandSpecial) > 0)

        BEGIN      

            INSERT #MaintenanceCommands

                ([Command], [CommandSpecial], [Table], [Object], [Rowmodctr], [Avg_fragmentation_in_percent], [Operation], [Priority], [OnlineRebuildSupport])

            VALUES

                (@Command, @CommandSpecial, @ObjectName, @IndexName, @RowModCtr, @frag, @Operation, @Priority, @OnlineRebuildSupport);

        END

    END

    CLOSE partitions;

    DEALLOCATE partitions;

    DECLARE todo CURSOR FOR

    SELECT

        [Command],

        [CommandSpecial],

        [Table],

        [Object],

        [Operation],

        [OnlineRebuildSupport],

        [Rowmodctr],

        [Avg_fragmentation_in_percent]

    FROM #MaintenanceCommands

    ORDER BY

        [Priority],

        [Rowmodctr] DESC,

        [Avg_fragmentation_in_percent] DESC

    OPEN todo;

    WHILE 1=1

    BEGIN

        FETCH NEXT FROM todo INTO @SQL, @SQLSpecial, @ObjectName, @IndexName, @Operation, @OnlineRebuildSupport, @RowModCtr, @AvgFragmentationPercent;

            

        IF @@FETCH_STATUS != 0    

            BREAK;

        -- Проверка доступен ли запуск обслуживания в текущее время

        SET @timeNow = CAST(GETDATE() AS TIME);

        IF (@timeTo >= @timeFrom) BEGIN

            IF(NOT (@timeFrom <= @timeNow AND @timeTo >= @timeNow))

                RETURN;

        END ELSE BEGIN

            IF(NOT ((@timeFrom <= @timeNow AND ''23:59:59'' >= @timeNow)

                OR (@timeTo >= @timeNow AND ''00:00:00'' <= @timeNow))) 

            RETURN;

        END

    

        -- Проверка процента занятого места в логе транзакций

        TRUNCATE TABLE #tranLogInfo;

        INSERT INTO #tranLogInfo (dbname,logsize,logspace,stat) exec(''dbcc sqlperf(logspace)'');

        SELECT

            @currentTransactionLogSizeUsagePercent = logspace,

            @currentTransactionLogSizeMB = logsize * (logspace / 100)

        FROM #tranLogInfo WHERE dbname = @databaseName

        IF(@currentTransactionLogSizeUsagePercent >= @maxTransactionLogSizeUsagePercent)

        BEGIN

            -- Процент занятого места в файлах лога транзакций превышает указанный порог

            RETURN;

        END

        IF(@maxTransactionLogSizeMB > 0 AND @currentTransactionLogSizeMB > @maxTransactionLogSizeMB)

        BEGIN

            -- Размер занятого места в файлах лога транзакций превышает указанный порог в МБ

            RETURN;

        END

    

        SET @StartDate = GetDate();

        BEGIN TRY

            DECLARE @currentSQL nvarchar(max) = ''''

            SET @MaintenanceActionLogId = 0

            IF(@SQLSpecial = '''')

            BEGIN

                SET @currentSQL = @SQL

                SET @UseOnlineRebuild = 0;

            END ELSE

            BEGIN

                SET @UseOnlineRebuild = 1;

                SET @currentSQL = @SQLSpecial

            END

            -- Сохраняем предварительную информацию об операции обслуживания без даты завершения

            IF(@useMonitoringDatabase = 1)

            BEGIN

                EXECUTE [' AS nvarchar(max)) + CAST(@monitoringDatabaseName  AS nvarchar(max)) + CAST('].[dbo].[sp_add_maintenance_action_log]

                @ObjectName

                ,@IndexName

                ,@Operation

                ,@RunDate

                ,@StartDate

                ,null

                ,@DBNAME

                ,@UseOnlineRebuild

                ,''''

                ,@AvgFragmentationPercent

                ,@RowModCtr

                ,@currentSQL

                ,@MaintenanceActionLogId OUTPUT;

            END

            EXEC sp_executesql @currentSQL;

            SET @FinishDate = GetDate();

            

            -- Устанавливаем фактическую дату завершения операции

            IF(@useMonitoringDatabase = 1)

            BEGIN

                EXECUTE [' AS nvarchar(max)) + CAST(@monitoringDatabaseName AS nvarchar(max)) + CAST('].[dbo].[sp_set_maintenance_action_log_finish_date]

                    @MaintenanceActionLogId,

                    @FinishDate;

            END 

        END  TRY   

        BEGIN CATCH

            IF(@MaintenanceActionLogId <> 0)

            BEGIN

                DECLARE @msg nvarchar(500) = ''Error: '' + CAST(Error_message() AS NVARCHAR(500)) + '', Code: '' + CAST(Error_Number() AS NVARCHAR(500)) + '', Line: '' + CAST(Error_Line() AS NVARCHAR(500))

                -- Устанавливаем текст ошибки при обслуживании индекса

                -- Дата завершения при этом остается незаполненной

                EXECUTE [' AS nvarchar(max)) + CAST(@monitoringDatabaseName AS nvarchar(max)) + CAST('].[dbo].[sp_set_maintenance_action_log_finish_date]

                    @MaintenanceActionLogId,

                    @FinishDate,

                    @msg;          

            END            

        END CATCH

    END

        

    CLOSE todo;

    DEALLOCATE todo;

    IF OBJECT_ID(''tempdb..#MaintenanceCommands'') IS NOT NULL

        DROP TABLE #MaintenanceCommands;

    IF OBJECT_ID(''tempdb..#MaintenanceCommandsTemp'') IS NOT NULL

        DROP TABLE #MaintenanceCommandsTemp;

    ' AS nvarchar(max))

    

        -- Для отладки. Выводит в SSMS весь текст сформированной команды

        --exec [dbo].[sp_AdvancedPrint] @sql = @cmd



        EXECUTE sp_executesql

            @cmd,

            N'@timeFrom TIME, @timeTo TIME, @fragmentationPercentForRebuild FLOAT,

            @fragmentationPercentMinForMaintenance FLOAT, @maxDop int,

            @minIndexSizePages int, @maxIndexSizePages int, @useOnlineIndexRebuild int,

            @maxIndexSizeForReorganizingPages int,

            @useMonitoringDatabase bit, @monitoringDatabaseName sysname, @usePreparedInformationAboutObjectsStateIfExists bit,

            @databaseName sysname, @maxTransactionLogSizeUsagePercent int, @maxTransactionLogSizeMB bigint, @useResumableIndexRebuild bit,

            @excludeIndexes XML',

            @timeFrom, @timeTo, @fragmentationPercentForRebuild,

            @fragmentationPercentMinForMaintenance, @maxDop,

            @minIndexSizePages, @maxIndexSizePages, @useOnlineIndexRebuild,

            @maxIndexSizeForReorganizingPages,

            @useMonitoringDatabase, @monitoringDatabaseName, @usePreparedInformationAboutObjectsStateIfExists,

            @databaseName, @maxTransactionLogSizeUsagePercent, @maxTransactionLogSizeMB, @useResumableIndexRebuild,

            @excludeIndexes;



        RETURN 0

    END

    GO



    CREATE PROCEDURE [dbo].[sp_SaveDatabasesTablesStatistic]

    AS

    BEGIN

        SET NOCOUNT ON;

        SET QUOTED_IDENTIFIER ON;



        IF OBJECT_ID('tempdb..#tableSizeResult') IS NOT NULL

            DROP TABLE #tableSizeResult;



        DECLARE @sql nvarchar(max);



        SET @sql = '

        SELECT

            DB_NAME() AS [databaseName],

            a3.name AS [schemaname],

            a2.name AS [tablename],

            a1.rows as row_count,

            (a1.reserved + ISNULL(a4.reserved,0))* 8 AS [reserved], 

            a1.data * 8 AS [data],

            (CASE WHEN (a1.used + ISNULL(a4.used,0)) > a1.data THEN (a1.used + ISNULL(a4.used,0)) - a1.data ELSE 0 END) * 8 AS [index_size],

            (CASE WHEN (a1.reserved + ISNULL(a4.reserved,0)) > a1.used THEN (a1.reserved + ISNULL(a4.reserved,0)) - a1.used ELSE 0 END) * 8 AS [unused]

        FROM

            (SELECT 

                ps.object_id,

                SUM (

                    CASE

                        WHEN (ps.index_id < 2) THEN row_count

                        ELSE 0

                    END

                    ) AS [rows],

                SUM (ps.reserved_page_count) AS reserved,

                SUM (

                    CASE

                        WHEN (ps.index_id < 2) THEN (ps.in_row_data_page_count + ps.lob_used_page_count + ps.row_overflow_used_page_count)

                        ELSE (ps.lob_used_page_count + ps.row_overflow_used_page_count)

                    END

                    ) AS data,

                SUM (ps.used_page_count) AS used

            FROM sys.dm_db_partition_stats ps

            GROUP BY ps.object_id) AS a1

        LEFT OUTER JOIN 

            (SELECT 

                it.parent_id,

                SUM(ps.reserved_page_count) AS reserved,

                SUM(ps.used_page_count) AS used

            FROM sys.dm_db_partition_stats ps

            INNER JOIN sys.internal_tables it ON (it.object_id = ps.object_id)

            WHERE it.internal_type IN (202,204)

            GROUP BY it.parent_id) AS a4 ON (a4.parent_id = a1.object_id)

        INNER JOIN sys.all_objects a2  ON ( a1.object_id = a2.object_id ) 

        INNER JOIN sys.schemas a3 ON (a2.schema_id = a3.schema_id)

        WHERE a2.type <> N''S'' and a2.type <> N''IT''

        ORDER BY reserved DESC

        ';



        CREATE TABLE #tableSizeResult (

            [DatabaseName] [nvarchar](255),

            [SchemaName] [nvarchar](255),

            [TableName] [nvarchar](255),

            [RowCnt] bigint,

            [Reserved] bigint,

            [Data] bigint,

            [IndexSize] bigint,

            [Unused] bigint

        );





        DECLARE @statement nvarchar(max);



        SET @statement = (

        SELECT 'EXEC ' + QUOTENAME(name) + '.sys.sp_executesql @sql; '

        FROM sys.databases

        WHERE NOT DATABASEPROPERTYEX(name, 'UserAccess') = 'SINGLE_USER' 

            AND HAS_DBACCESS(name) = 1

            AND state_desc = 'ONLINE'

            AND NOT database_id IN (

                DB_ID('tempdb'),

                DB_ID('master'),

                DB_ID('model'),

                DB_ID('msdb')

            )

        FOR XML PATH(''), TYPE

        ).value('.','nvarchar(max)');



        PRINT @statement



        INSERT #tableSizeResult

        EXEC sp_executesql @statement, N'@sql nvarchar(max)', @sql;



        DECLARE todo CURSOR FOR

        SELECT 

            [DatabaseName],

            [SchemaName],

            [TableName],

            [RowCnt],

            [Reserved],

            [Data],

            [IndexSize],

            [Unused]

        FROM #tableSizeResult;



        DECLARE

            @DatabaseName nvarchar(255),

            @SchemaName nvarchar(5),

            @TableName nvarchar(255),

            @RowCnt bigint,

            @Reserved bigint,

            @Data bigint,

            @IndexSize bigint,

            @Unused bigint,

            @currentDate datetime2(7);

        OPEN todo;



        WHILE 1=1

        BEGIN

            FETCH NEXT FROM todo INTO @DatabaseName, @SchemaName, @TableName, @RowCnt, @Reserved, @Data, @IndexSize, @Unused;

            IF @@FETCH_STATUS != 0

                BREAK;



            SET @currentDate = GETDATE();



            INSERT INTO [dbo].[DatabasesTablesStatistic]

            (

                [Period],

                [DatabaseName],

                [SchemaName],

                [TableName],

                [RowCnt],

                [Reserved],

                [Data],

                [IndexSize],

                [Unused]

            ) VALUES

            (

                @currentDate,

                @DatabaseName, 

                @SchemaName, 

                @TableName, 

                @RowCnt, 

                @Reserved, 

                @Data,

                @IndexSize, 

                @Unused

            );

        END



        CLOSE todo;

        DEALLOCATE todo;



        IF OBJECT_ID('tempdb..#tableSizeResult') IS NOT NULL

            DROP TABLE #tableSizeResult;

    END

    GO



    CREATE PROCEDURE [dbo].[sp_set_maintenance_action_log_finish_date]

        @MaintenanceActionLogId bigint,

        @FinishDate datetime2(0),

        @Comment nvarchar(255) = ''

    AS

    BEGIN

        SET NOCOUNT ON;



        UPDATE [dbo].[MaintenanceActionsLog]

        SET FinishDate = @FinishDate, 

            Comment = @Comment

        WHERE Id = @MaintenanceActionLogId

        RETURN 0

    END

    GO



    CREATE PROCEDURE [dbo].[sp_StatisticMaintenance]

        @databaseName sysname,

        @timeFrom TIME = '00:00:00',

        @timeTo TIME = '23:59:59',

        @mode int = 0,

        @ConditionTableName nvarchar(max) = 'LIKE ''%''',

        @useMonitoringDatabase bit = 1,

        @monitoringDatabaseName sysname = 'SQLServerMaintenance'

    AS

    BEGIN

        SET NOCOUNT ON;



        IF(@mode = 0)

        BEGIN

            EXECUTE [dbo].[sp_StatisticMaintenance_Sampled] 

            @databaseName

            ,@timeFrom

            ,@timeTo

            ,@ConditionTableName

            ,@useMonitoringDatabase

            ,@monitoringDatabaseName

        END ELSE IF(@mode = 1)

        BEGIN

            EXECUTE [dbo].[sp_StatisticMaintenance_Detailed] 

            @databaseName

            ,@timeFrom

            ,@timeTo

            ,@ConditionTableName

            ,@useMonitoringDatabase

            ,@monitoringDatabaseName

        END



        RETURN 0

    END

    GO



    CREATE PROCEDURE [dbo].[sp_StatisticMaintenance_Detailed]

        @databaseName sysname,

        @timeFrom TIME = '00:00:00',

        @timeTo TIME = '23:59:59',	

        @ConditionTableName nvarchar(max) = 'LIKE ''%''',

        @useMonitoringDatabase bit = 1,

        @monitoringDatabaseName sysname = 'SQLServerMaintenance'

    AS

    BEGIN

        SET NOCOUNT ON;



        DECLARE @msg nvarchar(max);



        IF DB_ID(@databaseName) IS NULL

        BEGIN

            SET @msg = 'Database ' + @databaseName + ' is not exists.';

            THROW 51000, @msg, 1;

            RETURN -1;

        END



        DECLARE @cmd nvarchar(max);

        SET @cmd = 

    CAST('USE [' AS nvarchar(max)) + CAST(@databasename AS nvarchar(max)) + CAST(']

    SET NOCOUNT ON;

    DECLARE

        -- Текущее время

        @timeNow TIME = CAST(GETDATE() AS TIME)

        -- Начало доступного интервала времени обслуживания

        -- @timeFrom TIME

        -- Окончание доступного интервала времени обслуживания

        -- @timeTo TIME

    -- Проверка доступен ли запуск обслуживания в текущее время

    IF (@timeTo >= @timeFrom) BEGIN

        IF(NOT (@timeFrom <= @timeNow AND @timeTo >= @timeNow))

            RETURN;

        END ELSE BEGIN

            IF(NOT ((@timeFrom <= @timeNow AND ''23:59:59'' >= @timeNow)

                OR (@timeTo >= @timeNow AND ''00:00:00'' <= @timeNow)))  

                    RETURN;

    END

    -- Служебные переменные

    DECLARE

        @DBID SMALLINT = DB_ID()

        ,@DBNAME sysname = DB_NAME()

        ,@TableName SYSNAME

        ,@IndexName SYSNAME

        ,@Operation NVARCHAR(128) = ''UPDATE STATISTICS''

        ,@RunDate DATETIME = GETDATE()

        ,@StartDate DATETIME

        ,@FinishDate DATETIME

        ,@SQL NVARCHAR(500)	

        ,@RowModCtr BIGINT

        ,@MaintenanceActionLogId bigint;

    DECLARE todo CURSOR FOR

    SELECT

        ''

        UPDATE STATISTICS ['' + SCHEMA_NAME([o].[schema_id]) + ''].['' + [o].[name] + ''] ['' + [s].[name] + '']

            WITH FULLSCAN'' + CASE WHEN [s].[no_recompute] = 1 THEN '', NORECOMPUTE'' ELSE '''' END + '';''

        , [o].[name]

        , [s].[name] AS [stat_name],

        [rowmodctr]

    FROM (

        SELECT

            [object_id]

            ,[name]

            ,[stats_id]

            ,[no_recompute]

            ,[last_update] = STATS_DATE([object_id], [stats_id])

            ,[auto_created]

        FROM sys.stats WITH(NOLOCK)

        WHERE [is_temporary] = 0) s

            LEFT JOIN sys.objects o WITH(NOLOCK) 

                ON [s].[object_id] = [o].[object_id]

            LEFT JOIN (

                SELECT

                    [p].[object_id]

                    ,[p].[index_id]

                    ,[total_pages] = SUM([a].[total_pages])

                FROM sys.partitions p WITH(NOLOCK)

                    JOIN sys.allocation_units a WITH(NOLOCK) ON [p].[partition_id] = [a].[container_id]

                GROUP BY 

                    [p].[object_id]

                    ,[p].[index_id]) p 

                ON [o].[object_id] = [p].[object_id] AND [p].[index_id] = [s].[stats_id]

            LEFT JOIN sys.sysindexes si

        ON [si].[id] = [s].[object_id] AND [si].[indid] = [s].[stats_id]

    WHERE [o].[type] IN (''U'', ''V'')

        AND [o].[is_ms_shipped] = 0

        AND [rowmodctr] > 0

        AND [o].[name] ' AS nvarchar(max)) + CAST(@ConditionTableName AS nvarchar(max)) + CAST('

    ORDER BY [rowmodctr] DESC;

    OPEN todo;

    WHILE 1=1

    BEGIN

        FETCH NEXT FROM todo INTO @SQL, @TableName, @IndexName, @RowModCtr;

        IF @@FETCH_STATUS != 0

            BREAK;

        -- Проверка доступен ли запуск обслуживания в текущее время

        SET @timeNow = CAST(GETDATE() AS TIME);

        IF (@timeTo >= @timeFrom) BEGIN

            IF(NOT (@timeFrom <= @timeNow AND @timeTo >= @timeNow))

                RETURN;

        END ELSE BEGIN

            IF(NOT ((@timeFrom <= @timeNow AND ''23:59:59'' >= @timeNow)

                OR (@timeTo >= @timeNow AND ''00:00:00'' <= @timeNow)))  

            RETURN;

        END

        SET @StartDate = GetDate();

        BEGIN TRY

            -- Сохраняем предварительную информацию об операции обслуживания без даты завершения

            IF(@useMonitoringDatabase = 1)

            BEGIN

                EXECUTE [' AS nvarchar(max)) + CAST(@monitoringDatabaseName  AS nvarchar(max)) + CAST('].[dbo].[sp_add_maintenance_action_log] 

                @TableName

                ,@IndexName

                ,@Operation

                ,@RunDate

                ,@StartDate

                ,null

                ,@DBNAME

                ,0

                ,''''

                ,0

                ,@RowModCtr

                ,@SQL

                ,@MaintenanceActionLogId OUTPUT;

            END

            EXEC sp_executesql @SQL;

            SET @FinishDate = GetDate();

            -- Устанавливаем фактическую дату завершения операции

            IF(@useMonitoringDatabase = 1)

            BEGIN

                EXECUTE [' AS nvarchar(max)) + CAST(@monitoringDatabaseName AS nvarchar(max)) + CAST('].[dbo].[sp_set_maintenance_action_log_finish_date]

                    @MaintenanceActionLogId,

                    @FinishDate;

            END

        END TRY

        BEGIN CATCH

            IF(@MaintenanceActionLogId <> 0)

            BEGIN

                DECLARE @msg nvarchar(500) = ''Error: '' + CAST(Error_message() AS NVARCHAR(500)) + '', Code: '' + CAST(Error_Number() AS NVARCHAR(500)) + '', Line: '' + CAST(Error_Line() AS NVARCHAR(500))

                -- Устанавливаем текст ошибки при обслуживании объекта статистики

                -- Дата завершения при этом остается незаполненной

                EXECUTE [' AS nvarchar(max)) + CAST(@monitoringDatabaseName AS nvarchar(max)) + CAST('].[dbo].[sp_set_maintenance_action_log_finish_date]

                    @MaintenanceActionLogId,

                    @FinishDate,

                    @msg;			

            END

        END CATCH

    END

    CLOSE todo;

    DEALLOCATE todo;

    ' AS nvarchar(max))



        EXECUTE sp_executesql 

            @cmd,

            N'@timeFrom TIME, @timeTo TIME,

            @useMonitoringDatabase bit, @monitoringDatabaseName sysname',

            @timeFrom, @timeTo,

            @useMonitoringDatabase, @monitoringDatabaseName;



        RETURN 0

    END

    GO



    CREATE PROCEDURE [dbo].[sp_StatisticMaintenance_Sampled]

        @databaseName sysname,

        @timeFrom TIME = '00:00:00',

        @timeTo TIME = '23:59:59',

        @ConditionTableName nvarchar(max) = 'LIKE ''%''',

        @useMonitoringDatabase bit = 1,

        @monitoringDatabaseName sysname = 'SQLServerMaintenance'

    AS

    BEGIN

        SET NOCOUNT ON;



        DECLARE @msg nvarchar(max);



        IF DB_ID(@databaseName) IS NULL

        BEGIN

            SET @msg = 'Database ' + @databaseName + ' is not exists.';

            THROW 51000, @msg, 1;

            RETURN -1;

        END



        DECLARE @cmd nvarchar(max);

        SET @cmd = 

    CAST('USE [' AS nvarchar(max)) + CAST(@databasename AS nvarchar(max)) + CAST(']

    SET NOCOUNT ON;

    DECLARE

        -- Текущее время

        @timeNow TIME = CAST(GETDATE() AS TIME)

        -- Начало доступного интервала времени обслуживания

        -- @timeFrom TIME

        -- Окончание доступного интервала времени обслуживания

        -- @timeTo TIME

    -- Проверка доступен ли запуск обслуживания в текущее время

    IF (@timeTo >= @timeFrom) BEGIN

        IF(NOT (@timeFrom <= @timeNow AND @timeTo >= @timeNow))

            RETURN;

        END ELSE BEGIN

            IF(NOT ((@timeFrom <= @timeNow AND ''23:59:59'' >= @timeNow)

                OR (@timeTo >= @timeNow AND ''00:00:00'' <= @timeNow)))  

                    RETURN;

    END

    -- Служебные переменные

    DECLARE

        @DBID SMALLINT = DB_ID()

        ,@DBNAME sysname = DB_NAME()

        ,@TableName SYSNAME

        ,@IndexName SYSNAME

        ,@Operation NVARCHAR(128) = ''UPDATE STATISTICS''

        ,@RunDate DATETIME = GETDATE()

        ,@StartDate DATETIME

        ,@FinishDate DATETIME

        ,@SQL NVARCHAR(500)	

        ,@RowModCtr BIGINT

        ,@MaintenanceActionLogId bigint;

    DECLARE @resample CHAR(8)=''NO'' -- Для включения установить значение RESAMPLE

    DECLARE @dbsid VARBINARY(85)

    SELECT @dbsid = owner_sid

    FROM sys.databases

    WHERE name = db_name()

    DECLARE @exec_stmt NVARCHAR(4000)

    -- "UPDATE STATISTICS [SYSNAME].[SYSNAME] [SYSNAME] WITH RESAMPLE NORECOMPUTE"

    DECLARE @exec_stmt_head NVARCHAR(4000)

    -- "UPDATE STATISTICS [SYSNAME].[SYSNAME] "

    DECLARE @options NVARCHAR(100)

    -- "RESAMPLE NORECOMPUTE"

    DECLARE @index_names CURSOR

    DECLARE @ind_name SYSNAME

    DECLARE @ind_id INT

    DECLARE @ind_rowmodctr INT

    DECLARE @updated_count INT

    DECLARE @skipped_count INT

    DECLARE @sch_id INT

    DECLARE @schema_name SYSNAME

    DECLARE @table_name SYSNAME

    DECLARE @table_id INT

    DECLARE @table_type CHAR(2)

    DECLARE @schema_table_name NVARCHAR(640)

    DECLARE @compatlvl tinyINT

    -- Получаем список объектов, для которых нужно обслуживание статистики

    DECLARE ms_crs_tnames CURSOR LOCAL FAST_FORWARD READ_ONLY for

    SELECT

        name, -- Имя объекта

        object_id, -- Идентификатор объекта

        schema_id, -- Идентификатор схемы

        type

    -- Тип объекта

    FROM sys.objects o

    WHERE (o.type = ''U'' OR o.type = ''IT'')

        AND [name] ' AS nvarchar(max)) + CAST(@ConditionTableName AS nvarchar(max)) + CAST('

    -- внутренняя таблица

    OPEN ms_crs_tnames

    FETCH NEXT FROM ms_crs_tnames INTO @table_name, @table_id, @sch_id, @table_type

    -- Определяем уровень совместимости для базы данных

    SELECT @compatlvl = cmptlevel

    FROM sys.sysdatabases

    WHERE name = db_name()

    WHILE (@@fetch_status <> -1)

    BEGIN

        -- Формируем полное имя объекта (схема + имя)

        SELECT @schema_name = schema_name(@sch_id)

        SELECT @schema_table_name = quotename(@schema_name, ''['') +''.''+ quotename(rtrim(@table_name), ''['')

        -- Пропускаем таблицы, для которых отключен кластерный индекс

        IF (1 = isnull((SELECT is_disabled

            FROM sys.indexes

            WHERE object_id = @table_id AND index_id = 1), 0))

        BEGIN

            FETCH NEXT FROM ms_crs_tnames INTO @table_name, @table_id, @sch_id, @table_type

            CONTINUE;

        END

        ELSE BEGIN

            -- Пропускаем локальные временные таблицы

            IF ((@@fetch_status <> -2) AND (substring(@table_name, 1, 1) <> ''#''))

            BEGIN

                SELECT @updated_count = 0

                SELECT @skipped_count = 0

                -- Подготавливаем начало команды: UPDATE STATISTICS [schema].[name]

                SELECT @exec_stmt_head = ''UPDATE STATISTICS '' + @schema_table_name + '' ''

                -- Обходим индексы и объекты статистики для текущего объекта

                -- Объекты статистики как пользовательские, так и созданные автоматически.				

                IF ((@table_type = ''U'') AND (1 = OBJECTPROPERTY(@table_id, ''TableIsMemoryOptimized'')))	-- In-Memory OLTP

                BEGIN

                    -- Hekaton-индексы (функциональность In-Memory OLTP) не отображаются в системном представлении sys.sysindexes,

                    -- Поэтому нужно использовать sys.stats для их обработки.

                    -- Примечание: OBJECTPROPERTY возвращает NULL для типа объекта "IT" (внутренние таблицы), 

                    -- поэтому можно использовать это только для типа ''U'' (пользовательские таблицы)

                    -- Для Hekaton-индексов (функциональность In-Memory OLTP) 

                    SET @index_names = CURSOR LOCAL FAST_FORWARD READ_ONLY for

                            SELECT name, stat.stats_id, modification_counter AS rowmodctr

                    FROM sys.stats AS stat

                            CROSS APPLY sys.dm_db_stats_properties(stat.object_id, stat.stats_id)

                    WHERE stat.object_id = @table_id AND indexproperty(stat.object_id, name, ''ishypothetical'') = 0

                        AND indexproperty(stat.object_id, name, ''iscolumnstore'') = 0

                    -- Для колоночных индексов статистика не обновляется

                    ORDER BY stat.stats_id

                END ELSE 

                BEGIN

                    -- Для обычных таблиц

                    SET @index_names = CURSOR LOCAL FAST_FORWARD READ_ONLY for

                            SELECT name, indid, rowmodctr

                    FROM sys.sysindexes

                    WHERE id = @table_id AND indid > 0 AND indexproperty(id, name, ''ishypothetical'') = 0

                        AND indexproperty(id, name, ''iscolumnstore'') = 0

                    ORDER BY indid

                END

                OPEN @index_names

                FETCH @index_names INTO @ind_name, @ind_id, @ind_rowmodctr

                -- Если объектов статистик нет, то пропускаем

                IF @@fetch_status < 0

                BEGIN

                    FETCH NEXT FROM ms_crs_tnames INTO @table_name, @table_id, @sch_id, @table_type

                    CONTINUE;

                END ELSE 

                    BEGIN

                    WHILE @@fetch_status >= 0

                        BEGIN

                        -- Формируем имя индекса

                        DECLARE @ind_name_quoted NVARCHAR(258)

                        SELECT @ind_name_quoted = quotename(@ind_name, ''['')

                        SELECT @options = ''''

                        -- Если нет данных о накопленных изменениях или они больше 0 (количество измененных строк)

                        IF ((@ind_rowmodctr is null) OR (@ind_rowmodctr <> 0))

                            BEGIN

                            SELECT @exec_stmt = @exec_stmt_head + @ind_name_quoted

                            -- Добавляем полное сканирование (FULLSCAN) для оптимизированных в памяти таблиц, если уровень совместимости < 130

                            IF ((@compatlvl < 130) AND (@table_type = ''U'') AND (1 = OBJECTPROPERTY(@table_id, ''TableIsMemoryOptimized''))) -- In-Memory OLTP

                                    SELECT @options = ''FULLSCAN''

                                -- add resample IF needed

                                ELSE IF (upper(@resample)=''RESAMPLE'')

                                    SELECT @options = ''RESAMPLE ''

                            -- Для уровнея совместимости больше 90 определяем доп. параметры

                            IF (@compatlvl >= 90)

                                    -- Устанавливаем параметр NORECOMPUTE, если свойство AUTOSTATS для него было установлено в OFF

                                    IF ((SELECT no_recompute

                            FROM sys.stats

                            WHERE object_id = @table_id AND name = @ind_name) = 1)

                                    BEGIN

                                IF (len(@options) > 0) SELECT @options = @options + '', NORECOMPUTE''

                                        ELSE SELECT @options = ''NORECOMPUTE''

                            END

                            -- Добавляем сформированные параметры в команду обновления статистики

                            IF (len(@options) > 0)

                                    SELECT @exec_stmt = @exec_stmt + '' WITH '' + @options

                            

                            SET @StartDate = GetDate();

                            

                            -- Проверка доступен ли запуск обслуживания в текущее время

                            SET @timeNow = CAST(GETDATE() AS TIME);

                            IF (@timeTo >= @timeFrom) BEGIN

                                IF(NOT (@timeFrom <= @timeNow AND @timeTo >= @timeNow))

                                    RETURN;

                            END ELSE BEGIN

                                IF(NOT ((@timeFrom <= @timeNow AND ''23:59:59'' >= @timeNow)

                                    OR (@timeTo >= @timeNow AND ''00:00:00'' <= @timeNow)))		

                                RETURN;

                            END

                            BEGIN TRY                            

                                -- Сохраняем предварительную информацию об операции обслуживания без даты завершения

                                IF(@useMonitoringDatabase = 1)

                                BEGIN

                                    EXECUTE [' AS nvarchar(max)) + CAST(@monitoringDatabaseName  AS nvarchar(max)) + CAST('].[dbo].[sp_add_maintenance_action_log] 

                                    @table_name

                                    ,@ind_name

                                    ,@Operation

                                    ,@RunDate

                                    ,@StartDate

                                    ,null

                                    ,@DBNAME

                                    ,0

                                    ,''''

                                    ,0

                                    ,@ind_rowmodctr

                                    ,@exec_stmt

                                    ,@MaintenanceActionLogId OUTPUT;

                                END

                                EXEC sp_executesql @exec_stmt;

                                SET @FinishDate = GetDate();

                                -- Устанавливаем фактическую дату завершения операции

                                IF(@useMonitoringDatabase = 1)

                                BEGIN

                                    EXECUTE [' AS nvarchar(max)) + CAST(@monitoringDatabaseName AS nvarchar(max)) + CAST('].[dbo].[sp_set_maintenance_action_log_finish_date]

                                        @MaintenanceActionLogId,

                                        @FinishDate;

                                END

                            END TRY

                            BEGIN CATCH

                                IF(@MaintenanceActionLogId <> 0)

                                BEGIN

                                    DECLARE @msg nvarchar(500) = ''Error: '' + CAST(Error_message() AS NVARCHAR(500)) + '', Code: '' + CAST(Error_Number() AS NVARCHAR(500)) + '', Line: '' + CAST(Error_Line() AS NVARCHAR(500))

                                    -- Устанавливаем текст ошибки при обслуживании объекта статистики

                                    -- Дата завершения при этом остается незаполненной

                                    EXECUTE [' AS nvarchar(max)) + CAST(@monitoringDatabaseName AS nvarchar(max)) + CAST('].[dbo].[sp_set_maintenance_action_log_finish_date]

                                        @MaintenanceActionLogId,

                                        @FinishDate,

                                        @msg;			

                                END

                            END CATCH

                            

                            SELECT @updated_count = @updated_count + 1

                        END ELSE

                        BEGIN

                            SELECT @skipped_count = @skipped_count + 1

                        END

                        FETCH @index_names INTO @ind_name, @ind_id, @ind_rowmodctr

                    END

                END

                DEALLOCATE @index_names

            END

        END

        FETCH NEXT FROM ms_crs_tnames INTO @table_name, @table_id, @sch_id, @table_type

    END

    DEALLOCATE ms_crs_tnames

    ' AS nvarchar(max))



        EXECUTE sp_executesql 

            @cmd,

            N'@timeFrom TIME, @timeTo TIME,

            @useMonitoringDatabase bit, @monitoringDatabaseName sysname',

            @timeFrom, @timeTo,

            @useMonitoringDatabase, @monitoringDatabaseName;



        RETURN 0

    END

    GO
    ```

Для актуальной версии следите за обновлениями. Также никто не мешает Вам дорабатывать эти скрипты для себя.

Инструмент готов, а теперь нужно заменить компоненты обслуживания на свои скрипты. И вот как это будет выглядеть.

![](img/ee8551c9c32f4d9292ee15deeb812265.png)Вместо трех шагов теперь только 2, в каждом свои скрипты для выполнения операций.

Шаг обслуживания индексов

На первом шаге используется такой скрипт.
???- info "скрипт"
    ``` sql


    EXECUTE [SQLServerMaintenance].[dbo].[sp_IndexMaintenance] 

    @databaseName = 'BSL-ORIG'

    -- Разрешаем запуск скрипта с 21:00:00 до 22:30:00

    ,@timeFrom = '21:00:00'

    ,@timeTo = '22:30:00'

    -- 1 - обслуживать только те объекты, которые поддерживают онлайн-перестроение

    ,@useOnlineIndexRebuild = 1

    -- Процент фрагментации, с которого будет выполняться обслуживание

    ,@fragmentationPercentMinForMaintenance = 10

    -- Процент фрагментации с которого начинается полное перестроение.

    -- Все, что между 10 и 30 будет обслуживаться операцией реорганизации

    ,@fragmentationPercentForRebuild = 30

    -- Степень параллелизма для операций перестроения оставляем равной 8

    ,@maxDop = 8

    -- Настраиваем поведение онлайн-перестроения в ситуациях, когда переключение индекса на новый блокируется

    -- другими запросами

    -- 1 - операция обслуживания завершит себя по истечении таймаута ожидания.

    ,@onlineRebuildAbortAfterWaitMode = 1

    -- Время ожидания операции онлайн перестроения перед прерыванием работы. Ставим 15 минут

    ,@onlineRebuildWaitMinutes = 15
    ```

В скрипте мы указываем имя базы, для которой устанавливаем обслуживание и диапазон времени, в который можно выполнять операции обслуживания.

Кроме этого указываем, что нужно обслуживать только те объекты, которые поддерживают онлайн перестроение (параметр @useOnlineIndexRebuild). Также устанавливаем, что обслуживание нужно начинать только с 10% фрагментацией (@fragmentationPercentMinForMaintenance), а полное перестроение только при 30% значении фрагментации индекса (@fragmentationPercentForRebuild). Таким образом, реорганизация индекса будет применяться, если фрагментация находится в диапазоне между 10% и 30%.

Дополнительно к этому установим особые параметры онлайн обслуживания:

* При блокировании обслуживания другими запросами отдаем приоритет именно им. То есть по истечении таймаута операция обслуживания завершит сама себя (@onlineRebuildAbortAfterWaitMode).
* Время ожидания при этом установим в 15 минут (@onlineRebuildAbortAfterWaitMode).

Это должно решить все перечисленные выше задачи.

Шаг обслуживания статистик

Для обслуживания статистики будем использовать следующий скрипт.
???- info "скрипт"
    ``` sql


    EXECUTE [SQLServerMaintenance].[dbo].[sp_StatisticMaintenance] 

    @databaseName = 'BSL-ORIG'

    ,@timeFrom = '21:00:00'

    ,@timeTo = '22:30:00'

    -- 0 устанавливаем режим анализа выборки данных, 

    -- 1 - режим полного сканирования

    ,@mode = 1
    ```

Также указываем время работы скрипта и режим полного сканирования.

На первом шаге мы обязательно установим таймаут выполнения операции в 2 часа. Это соотносится с указанным в настройках диапазоном времени с 21:00 до 22:30 (1.5 часа). Таймаут в 2 часа это последний рубеж защиты, чтобы процедуры обслуживания не вышли за 23:00. Такое может произойти, если индекс начал операцию перестроения или реорганизации в 22:20 и к 23:00 не завершился, то таймаут выполнения команды ее прервет "насильно".

При этом для операции обновления статистики таймаут выполнения мы не ставим, т.к. она не мешает работе других запросов. При этом если обслуживание индексов работало до 23:00 (с таймаутом или без), то обслуживание статистик не будет запущено. Считаем это проблемными ситуациями и они не должны возникать часто.

Также добавим дополнительный субплан для обслуживания индексов, которые не поддерживают онлайн перестроение. Считаем, что таких объектов не много и их можно обслуживать раз в неделю. Для этого добавим субплан с запуском раз в неделю, например в субботу в 23:00 и отдаем на работу скрипта 30 минут. Скрипт будет таким.

Еженедельное обслуживание индексов без поддержки онлайн перестроения

Заменяем параметр "@useOnlineIndexRebuild" и убираем настройки, связанные с онлайн перестроением.
???- info "скрипт"
    ``` sql


    EXECUTE [SQLServerMaintenance].[dbo].[sp_IndexMaintenance] 

    @databaseName = 'BSL-ORIG'

    -- Разрешаем запуск скрипта с 23:00:00 до 23:30:00

    ,@timeFrom = '23:00:00'

    ,@timeTo = '23:30:00'

    -- (2) - обслуживание только тех объектов, в которых онлайн-перестроение не поддерживается.

    ,@useOnlineIndexRebuild = 2

    -- Процент фрагментации, с корого будет выполняться обслуживание

    ,@fragmentationPercentMinForMaintenance = 10

    -- Процент фрагментации с которого начинается полное перестроение.

    -- Все, что между 10 и 30 будет обслуживаться операцией реорганизации

    ,@fragmentationPercentForRebuild = 30

    -- Степень параллелизма для операций перестроения оставляем равной 8

    ,@maxDop = 8
    ```

Остальное все как обычно.

Таймаут для операции установим в 1 час, то есть в 00:00 операция будет завершена в любом случае.

И последнее изменение - это дополнительные шаги обслуживания статистики. Ранее мы запускали в дневное время отдельное обновление статистики, а теперь будем выполнять это 3 раза (не считая основного ночного обслуживания): в 06:00, 13:00 и в 18:00.

Регулярное обслуживание статистики

Отключаем ограничения времени выполнения и режим обновления теперь выполняем через выборку данных, а не полным сканированием.
???- info "скрипт"
    ``` sql


    EXECUTE [SQLServerMaintenance].[dbo].[sp_StatisticMaintenance] 

    @databaseName = 'BSL-ORIG'

    -- 0 устанавливаем режим анализа выборки данных, 

    -- 1 - режим полного сканирования

    ,@mode = 0
    ```

 Так как полное сканирование не используется, то даже для многотеррабайтных баз эта операция будет выполняться за 5-30 минут. Поэтому ограничения выполнения или таймаутов выполнения устанавливать не будем.

Готово! Небольшие итоги:

Плюсы:

1. Минимальное влияние на работу запросов во время обслуживания (как по блокировкам, так и по нагрузке на базу).
2. Максимально актуальная статистика во время всего рабочего дня.
3. Полный контроль над выполняемыми операциями. Приоритет отдается выполняемым запросам, а не обслуживанию. Работа информационной системы не будет нарушена.
4. Практически полное покрытие задач обслуживания баз данных в большинстве мелких, средних и крупных баз данных.

Минусы:

1. Более сложная схема настройки, требующая понимания работы процессов обслуживания и их сопровождения.
2. Необходимость прочитать инструкции и документацию по SQL Server в нештатных ситуациях.
3. Желательны навыки работы с TSQL.

Может ли потребоваться изменять обслуживание еще как-то?

### А вот и полная модель

Настает момент, когда для задач отказоустойчивости и надежности базу данных переключают в [**полную модель восстановления**](https://learn.microsoft.com/ru-ru/sql/relational-databases/backup-restore/recovery-models-sql-server?view=sql-server-ver15). Это позволяет восстановить базу данных из бэкапа на любой момент времен и не потерять данные в случае внезапного падения. Другие полезные возможности при полной модели:

* **[Использование доставки логов транзакций и создания копий баз данных](https://github.com/YPermitin/SQLServerTools/tree/master/SQL-Server-Replication-And-High-Availability)**
* **[Создание реплик неограниченного количества](https://github.com/YPermitin/SQLServerTools/tree/master/SQL-Server-AlwaysOn)** и сложной топологии
* **[Применение механизма CDC для регистрации изменений](https://github.com/YPermitin/SQLServerTools/tree/master/SQL-Server-Track-Data-Changes)**
* И многие другие "плюшки".

Рассматривать шаги для безопасного перевода баз в полную модель мы не будем. Основное что отметим - в таком режиме все изменения сохраняются в лог транзакций и остаются там до тех пор, пока он не будет бэкапирован. Важно заметить, что только бэкап лога транзакций помечает их готовыми к удалению из файла лога. Полный бэкап файл лога транзакций не очистит.

Но использование полной модели накладывает особенности в работе обслуживания. Может случиться так, что перестроение индексов может заполнить лог транзакций, что остановит все операции модификации данных в базе. Другими словами, информационная система перестанет работать.

Чтобы этого не случилось - установим ограничения на использование файла логов транзакций. Дополним предыдущий скрипт параметром ограничения.

Ограничиваем использование лога транзакций

Скрипт тот же, только добавили один параметр в конце.
???- info "скрипт"
    ``` sql


    EXECUTE [SQLServerMaintenance].[dbo].[sp_IndexMaintenance] 

    @databaseName = 'BSL-ORIG'

    -- Разрешаем запуск скрипта с 21:00:00 до 22:30:00

    ,@timeFrom = '21:00:00'

    ,@timeTo = '22:30:00'

    -- 1 - обслуживать только те объекты, которые поддерживают онлайн-перестроение

    ,@useOnlineIndexRebuild = 1

    -- Процент фрагментации, с которого будет выполняться обслуживание

    ,@fragmentationPercentMinForMaintenance = 10

    -- Процент фрагментации с которого начинается полное перестроение.

    -- Все, что между 10 и 30 будет обслуживаться операцией реорганизации

    ,@fragmentationPercentForRebuild = 30

    -- Степень параллелизма для операций перестроения оставляем равной 8

    ,@maxDop = 8

    -- Настраиваем поведение онлайн-перестроения в ситуациях, когда переключение индекса на новый блокируется

    -- другими запросами

    -- 1 - операция обслуживания завершит себя по истечении таймаута ожидания.

    ,@onlineRebuildAbortAfterWaitMode = 1

    -- Время ожидания операции онлайн перестроения перед прерываниеем работы. Ставим 15 минут

    ,@onlineRebuildWaitMinutes = 15

    -- !!!

    -- Разрешаем использование только 50% файла лога транзакций.

    -- Если лог транзакций заполнен на больший процент, то новые объекты обслуживаться не будут

    ,@maxTransactionLogSizeUsagePercent = 50

    -- Можно также установить ограничение явно в мегабайтах

    --,@maxTransactionLogSizeMB bigint = 0

    -- !!!
    ```

Параметры *@maxTransactionLogSizeUsagePercent* и *@maxTransactionLogSizeMB* как раз и решают эту задачу. Но нужно учитывать, что проверка выполняется перед началом выполнения операции обслуживания. Если индекс начал перестраиваться, то ограничение на текущий процесс уже не повлияет. Поэтому также рекомендуется спланировать резерв места на диске с файлом лога транзакций. А если, мало ли, размер логов при перестроении может превысить 2 ТБ, то нужно создать несколько файлов логов транзакций, чтобы обойти это ограничение. Да, максимальный размер одного файла лога транзакций равен 2 ТБ.

### Разделяй и властвуй

В некоторых случаях есть смысл большие индексы обслуживать не ежедневно, а раз в неделю и, например, только полным перестроением. Для этого внесем следующие изменения:

* В ежедневном плане обслуживания индексов установим условие на максимальный размер обслуживаемых объектов. Расписание установим с понедельника по субботу.

Ограничиваем размер обслуживаемых объектов

Можно установить нижнюю (@maxIndexSizePages) и верхнюю (@maxIndexSizePages) границу. В примере мы ставим условие только на макс. размер объекта.
???- info "скрипт"
    ``` sql


    EXECUTE [SQLServerMaintenance].[dbo].[sp_IndexMaintenance] 

    @databaseName = 'BSL-ORIG'

    -- Разрешаем запуск скрипта с 21:00:00 до 22:30:00

    ,@timeFrom = '21:00:00'

    ,@timeTo = '22:30:00'

    -- 1 - обслуживать только те объекты, которые поддерживают онлайн-перестроение

    ,@useOnlineIndexRebuild = 1

    -- Процент фрагментации, с которого будет выполняться обслуживание

    ,@fragmentationPercentMinForMaintenance = 10

    -- Процент фрагментации с которого начинается полное перестроение.

    -- Все, что между 10 и 30 будет обслуживаться операцией реорганизации

    ,@fragmentationPercentForRebuild = 30

    -- Степень параллелизма для операций перестроения оставляем равной 8

    ,@maxDop = 8

    -- Настраиваем поведение онлайн-перестроения в ситуациях, когда переключение индекса на новый блокируется

    -- другими запросами

    -- 1 - операция обслуживания завершит себя по истечении таймаута ожидания.

    ,@onlineRebuildAbortAfterWaitMode = 1

    -- Время ожидания операции онлайн перестроения перед прерыванием работы. Ставим 15 минут

    ,@onlineRebuildWaitMinutes = 15

    -- !!!

    -- Разрешаем использование только 50% файла лога транзакций.

    -- Если лог транзакций заполнен на больший процент, то новые объекты обслуживаться не будут

    ,@maxTransactionLogSizeUsagePercent = 50

    -- Макс. размер обслуживаемых объектов будет равен 50 ГБ.

    -- Размер устанавливается в количестве страниц по 8 КБ.

    -- 50 ГБ = 6553600 страниц

    ,@maxIndexSizePages = 6553600
    ```

Теперь объекты размером 100 ГБ обслуживаться не будут.

* Добавим еще один субплан обслуживания с теми же операциями, что и в ежедневном плане обслуживания, но в скрипте обслуживания индексов снимем ограничение на макс. размер индекса.

Это позволит разделить обслуживание и избавится от массивных операций изменения в будние дни. Тут стоит отметить, что нужно понимать последствия таких решений. Предварительно нужно проанализировать что это за таблицы и как изменение стратегии обслуживания повлияет на работу информационной системы.

Есть и более точечный вариант настройки обслуживания, в котором ограничения будут опираться не на размер объектов, а на конкретные объекты. Например, можно добавить такое условие в процедуру обслуживания индексов:
???- info "скрипт"
    ``` sql


    -- Отбор по конкретной таблице

    ,@ConditionTableName = 'IN (''_AccumRg1265'',''_AccumRg505'')'
    ```

или даже поставить отбор на конкретный индекс:
???- info "скрипт"
    ``` sql


    -- Отбор на конкретный индекс (отбор по таблице в этом случае не обязателен)

    ,@ConditionIndexName = '= ''_AccumRg505_1'''
    ```

Для таких точечных операций обслуживания можно задать свое расписание, свои таймауты выполнения и так далее.

### Слишком большие объекты

Следующая проблема, которая может появиться в больших базах - это обслуживание ооооооочень больших, огромных объектов в пределах окна обслуживания. Например, у нас есть 2 часа на обслуживание индексов. Но что, если для перестроения индекса, размер которого 1 ТБ, нужно 5 часов.

Конечно, в онлайн режиме блокировок это не создаст, но замедление работы с индексом или выделяемые ресурсы для такого перестроения могут косвенно влиять на другие запросы. Кроме этого, файл лога транзакций будет заполняться полностью и даже дополнительные файлы могут не спасти ситуацию.

Тут на помощь приходят возобновляемые операции перестроения индексов, **[доступных со SQL Server 2017](https://www.mssqltips.com/sqlservertip/4987/sql-server-2017-resumable-online-index-rebuilds)**. Работает это так:

* Вы запускаете операцию перестроения индекса в онлайн режиме, указав параметр RESUMABLE = ON.
???- info "скрипт"
    ``` sql


    ALTER INDEX PK_1 

    ON [dbo].[_Acc1]

    REBUILD

    WITH (ONLINE = ON, RESUMABLE = ON);

    GO
    ```

* Процесс перестроения выполняется, пока Вы не прервете его явно командой завершения сессии (например, если таймаут сработал) или не указав остановку явно.
???- info "скрипт"
    ``` sql


    ALTER INDEX PK_1 ON [dbo].[_Acc1]  PAUSE
    ```

* После остановки операции файл журнала транзакций может быть освобожден после выполнения бэкапа логов. Да, перестроение индекса еще не завершено, но освободить файл журнала транзакций можно.
* Можно проверить список операций перестроения, доступных для продолжения, а также состояние прогресса перестроения
???- info "скрипт"
    ``` sql


    SELECT 

    total_execution_time, 

    percent_complete, 

    name,

    state_desc,

    last_pause_time,

    page_count

    FROM sys.index_resumable_operations;
    ```

* Возобновляем операцию перестроения при необходимости. Например, в следующем технологическом окне обслуживания.
???- info "скрипт"
    ``` sql


    ALTER INDEX PK_1 ON [dbo].[_Acc1]  RESUME
    ```

* Или можно прервать операцию окончательно.
???- info "скрипт"
    ``` sql


    ALTER INDEX PK_1 ON [dbo].[_Acc1]  ABORT
    ```

Это позволит выполнять даже самые тяжелые операции в заданное окно обслуживания, хоть и за несколько дней. А также обезопасить себя от переполнения лога транзакций.

В контексте нашей служебной базы использование возобновляемого перестроения включается через параметр @useResumableIndexRebuildIfAvailable, который может быть задействован только при использовании операций онлайн перестроения индексов.

Возобновляемое перестроение индексов

В примере ниже мы поставим отбор для конкретного индекса, по которому нужно выполнять возобновляемое перестроение. Использовать подобное обслуживание для всех объектов в базе не имеет смысла.
???- info "скрипт"
    ``` sql


    EXECUTE [SQLServerMaintenance].[dbo].[sp_IndexMaintenance] 

    @databaseName = 'BSL-ORIG'

    -- Разрешаем запуск скрипта с 21:00:00 до 22:30:00

    ,@timeFrom = '21:00:00'

    ,@timeTo = '22:30:00'

    -- (2) - обслуживание только тех объектов, в которых онлайн-перестроение не поддерживается.

    ,@useOnlineIndexRebuild = 2

    -- Процент фрагментации, с которого будет выполняться обслуживание

    ,@fragmentationPercentMinForMaintenance = 10

    -- Процент фрагментации с которого начинается полное перестроение.

    -- Все, что между 10 и 30 будет обслуживаться операцией реорганизации

    ,@fragmentationPercentForRebuild = 30

    -- Степень параллелизма для операций перестроения оставляем равной 8

    ,@maxDop = 8

    -- Включаем возобновляемое перестроение индексов

    ,@useResumableIndexRebuildIfAvailable = 1

    -- И только для конкретного индекса

    ,@ConditionIndexName = '= ''_AccumRg505_1'''
    ```

Последние два параметра тут основные для примера.

Но есть и подводный камень! Если индекс находится в списке операций возобновляемых операций перестроения, то его нельзя удалить или перестроить другими операциями. Нужно либо завершить перестроение, либо прервать его окончательно.

Если во время работы скрипта из примера операция будет прервана по таймауту (или вручную), то при следующем запуске этого же скрипта будет действовать такой алгоритм:

1. Проверяются списки объектов, для которых нужно выполнить возобновление перестроения. При этом учитываются условия по таблицам и индексам.
2. Если такая операция есть в списке, то возобновляем ее работу. Если нет, то идем дальше.
3. Далее выполняем обычные операции обслуживания. При этом если на прошлом шаге было завершено возобновляемое перестроение индекса, то на следующих шагах обслуживания эти объекты будут пропущены. Это сделано для того, чтобы один и тот же объект не был перестроен дважды.

Этот механизм является спасительным для многих больших баз данных. А некоторые люди еще говорят, что разницы между SQL Server 2012 и SQL Server 2019 нет, вот им пример.

### Да здравствует AlwaysOn

При включении групп высокой доступности AlwaysOn появляется особый нюанс - если реплика становится недоступной для отправки изменений, то записи в файле лога транзакций не будут удалены даже после выполнения бэкапа лога транзакций. Это сделано для того, чтобы при возобновлении передачи данных не потерять транзакции, которые еще не ушли на копии баз, реплики.

Для обслуживания это может означать, что при каких-то авариях с передачей данных в AlwaysOn лучше повременить с обслуживанием, пока передача данных не будет восстановлена и файл лога транзакций не будет освобождён.

Подобная ситуация может возникнуть не только при сбое связи, а, например, если после перестроения 1 ТБ индекса эти изменения будут отправляться на копии. Пока все изменения после перестрое ния не "уйдут" на копии, то лог транзакций также не сможет быть освобожден.

Но на самом деле решение этой проблемы у нас уже есть. Выше мы уже говорили про параметры ограничения файла лога транзакций в полной модели восстановления, а также кратко прошлись по созданию резерва для логов. Этот же подход нужно использовать и в нашем случае.

Также нужно понимать, что операции обслуживания нужно выполнять только в первичном узле (основной базе), а на копиях баз настраивать обслуживание просто нет смысла. Это же копии в режиме "только для чтения". Перестроение индексов или обновление статистик там недоступно.

Информацию про **[использование AlwaysOn](https://github.com/YPermitin/SQLServerTools/tree/master/SQL-Server-AlwaysOn)** Вы можете посмотреть здесь, в том числе описание некоторых нюансов и настроек.

### Комбо!

И напоследок - комбо! Все вышеописанные подходы к настройке можно комбинировать как Вам угодно, главное не делать это вслепую! И лучше всего усложнять обслуживание только по необходимости, подтверждая свои шаги через анализ логов обслуживания.

Если Вы будете использовать служебную базу, о конторой идет речь в этой статье, то все операции перестроения индексов или обновления объектов статистики логируются. Например, в таблице "MaintenanceActionsLog" можно посмотреть когда, как и почему объект обслуживался.

![](img/32c012e488a177d45cc1ad6058d5758d.png)Вы будете знать обслуживался ли он онлайн, было это перестроение или реорганизация, какой процент фрагментации был на момент обслуживания, сколько записей изменилось с момента обновления статистики, какая SQL-команда использовалась, длительность операции и вообще завершилась ли операция. Операция обслуживания будет взята под полный контроль!

Также может возникнуть необходимость запускать скрипты перестроения индексов параллельно друг другу, для ускорения обслуживания, например. Но тут возникает нюанс! Если есть активная операция перестроения индекса, то DMV [**sys.dm\_db\_index\_physical\_stats**](https://learn.microsoft.com/ru-ru/sql/relational-databases/system-dynamic-management-views/sys-dm-db-index-physical-stats-transact-sql?view=sql-server-ver15) не сможет получить текущее состояние индексов, пока перестроение не будет завершено. В итоге операции перестроения параллельно и не будут запущены. Но и тут есть выход!

Вы можете запустить сбор информации о состоянии объектов базы данных заранее через команду:
???- info "скрипт"
    ``` sql


    EXECUTE [SQLServerMaintenance].[dbo].[sp_FillDatabaseObjectsState] 

    @databaseName = 'BSL-ORIG'
    ```

Команда вызовет [**sys.dm\_db\_index\_physical\_stats**](https://learn.microsoft.com/ru-ru/sql/relational-databases/system-dynamic-management-views/sys-dm-db-index-physical-stats-transact-sql?view=sql-server-ver15) и сохранит результаты в таблицу "DatabaseObjectsState". А в скриптах перестроения индексов можно указать параметр @usePreparedInformationAboutObjectsStateIfExists, тогда повторного анализа объектов выполнено не будет и обслуживание будет использовать информацию из таблицы "DatabaseObjectsState".
???- info "скрипт"
    ``` sql


    -- Используем ранее сохраненную информацию о состоянии объектов базы данных

    ,@usePreparedInformationAboutObjectsStateIfExists = 1
    ```

Но есть нюанс! Информация об объектах базы должна быть собрана в последние 12 часов на момент вызова обслуживания. Иначе информация будет считаться устаревшей и запустится обычный анализ объектов.

Таким образом, можно запускать 2 и более процессов обслуживания индексов, не опасаясь их блокировки друг другом на этапе анализа объектов базы.

## Как отслеживать качество обслуживания

Это отдельная тема, но в самом простом виде можно действовать так:

* Следить, чтобы статистика была максимально актуальной. Например, таким скриптом. Так вы увидите есть ли объекты, по которым статистика давно не обновлялась, а также объекты, по которым изменения накапливаются очень быстро. Возможно, нужно делать обновление статистики для них чаще нескольких раз в день.

Анализ состояния статистики
???- info "скрипт"
    ``` sql


    select

        o.name AS [TableName],

        a.name AS [StatName],

        a.rowmodctr AS [RowsChanged],

        STATS_DATE(s.object_id, s.stats_id) AS [LastUpdate],

        o.is_ms_shipped,

        s.is_temporary,

        p.*

    from sys.sysindexes a

        inner join sys.objects o

        on a.id = o.object_id

            and o.type = 'U'

            and a.id > 100

            and a.indid > 0

        left join sys.stats s

        on a.name = s.name

        left join (

    SELECT

            p.[object_id]

    , p.index_id

    , total_pages = SUM(a.total_pages)

        FROM sys.partitions p WITH(NOLOCK)

            JOIN sys.allocation_units a WITH(NOLOCK) ON p.[partition_id] = a.container_id

        GROUP BY 

    p.[object_id]

    , p.index_id

    ) p ON o.[object_id] = p.[object_id] AND p.index_id = s.stats_id

    order by

        a.rowmodctr desc,

        STATS_DATE(s.object_id, s.stats_id) ASC
    ```

    * Следить за фрагментацией индексов.

    Анализ состояния индексов

    ``` sql


    SELECT OBJECT_NAME(ips.OBJECT_ID)

    ,i.NAME

    ,ips.index_id

    ,index_type_desc

    ,avg_fragmentation_in_percent

    ,avg_page_space_used_in_percent

    ,page_count

    FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'SAMPLED') ips

    INNER JOIN sys.indexes i ON (ips.object_id = i.object_id)

    AND (ips.index_id = i.index_id)

    ORDER BY avg_fragmentation_in_percent DESC
    ```

Можно использовать уровень детализации "DETAILED" для более точных данных.

Плюс тяжелые запросы с неоптимальными планами могут подсказать, какие таблицы являются проблемными. Скриптами выше проверьте их состояние.

Но в целом это отдельный разговор. Сегодня об этом речь не идет.

## Это еще не конец

Мы прошли долгий путь и каждый шаг дался нам не просто:

1. Сначала настроили базовое обслуживание с помощью поставляемых компонентов SQL Server.
2. Затем усложнили обслуживание для уменьшения влияния на информационную систему.
3. После вынужденно отказались от штатных компонентов и использовали свои скрипты и наработки, поставили ограничения на операции обслуживания, повысили надежность работы и пресекли выход за рамки технологического окна. А также улучшили онлайн обслуживание индексов.
4. Далее рассмотрели нюансы при работе в полной модели восстановления базы.
5. Разбили обслуживание на регулярное и еженедельное для оптимизации работы.
6. Изменили стратегию обслуживания для огромных объектов, внедрив возобновляемое перестроение индексов.
7. Рассмотрели особенности обслуживания при использовании AlwaysOn.
8. И напоследок обсудили логирование и контроль обслуживания.

Но это не конец пути! Все это лишь показывает, что обслуживание баз данных может адаптироваться под любые требования, было бы желание, деньги и время. Ну и знания, конечно же. Надеюсь, в последнем эта статья поможет и даст старт для изучения вопроса.

