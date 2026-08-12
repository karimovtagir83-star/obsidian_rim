---
tags: [раздел-93, рабочий-проект, outbox, inbox, события, kafka, идемпотентность, middle, собес]
aliases: [Outbox в ServiceShop, Inbox ServiceShop, Обмен событиями ServiceShop, DomainEventsDispatcher, Integration Events ServiceShop]
---

# ServiceShop: Outbox, Inbox и обмен событиями

> [!abstract] Коротко
> Модули ServiceShop не вызывают друг друга — они обмениваются событиями через таблицы БД. Цепочка выглядит громоздко: агрегат → `outbox_messages` → фоновая джоба → шина → `inbox_messages` получателя → фоновая джоба → отложенная команда в `internal_commands` → ещё одна джоба → хендлер. Шесть звеньев вместо одного вызова метода.
> Плата берётся за три вещи, которых прямым вызовом не получить: **атомарность** (данные и событие коммитятся вместе), **гарантию доставки** (упавшее переигрывается из таблицы) и **развязку** (получатель может быть недоступен, отправитель не заметит). Понять эту цепочку — значит понять, почему проект устроен именно так.

## Зачем это нужно

Задача: сохранить заказ в БД и сообщить об этом другим модулям. Прямолинейное решение — два действия подряд:

```csharp
await _dbContext.SaveChangesAsync();          // 1
await _kafkaProducer.ProduceAsync(evt);       // 2
```

Между строками 1 и 2 процесс может умереть. Тогда: данные в базе, события нет — Marketing не отправит уведомление, Cart не очистит корзину, и узнать об этом никак. Переставить строки местами — станет хуже: событие ушло, транзакция откатилась, все узнали о заказе, которого не существует.

Проблема фундаментальная: **БД и брокер — два разных ресурса, общей транзакции у них нет**. Распределённая транзакция (2PC) теоретически возможна, практически — не поддерживается Kafka и приносит больше проблем, чем решает.

**Outbox Pattern** обходит проблему: событие пишется **в ту же базу, в той же транзакции**, в отдельную таблицу. Один ресурс — одна транзакция — атомарность бесплатно. Дальше отдельный процесс читает таблицу и рассылает. Теория — [[Паттерн Transactional Outbox]].

---

## Domain Event против Integration Event

Их легко перепутать, и путаница дорого стоит.

| | Domain Event | Integration Event |
|---|---|---|
| Где живёт | `<M>.Domain/<Агрегат>/DomainEvents/` | проект `<M>.IntegrationEvents` |
| Аудитория | внутри своего модуля | другие модули, внешние сервисы |
| Транспорт | MediatR в памяти | Outbox → шина → Inbox |
| Базовый тип | `IDomainEvent : INotification` | `IntegrationEvent` |
| Менять поля | легко, никто снаружи не видит | **нельзя** — публичный контракт |
| Когда доставлено | в той же транзакции | позже, eventual consistency |

Практическое следствие: доменное событие можно свободно рефакторить. Интеграционное — нет: его формат знают другие модули, а сериализованные экземпляры лежат в таблицах `outbox_messages`/`inbox_messages` и будут разобраны после деплоя. Удалить поле из интеграционного события = сломать разбор старых сообщений в очереди.

```csharp
public class ModeratedOfferPriceChangedIntegrationEvent : IntegrationEvent
{
    public Guid ModeratedOfferId { get; }
    public ulong Price { get; }
    public bool IsVisible { get; }

    public ModeratedOfferPriceChangedIntegrationEvent(
        Guid id, DateTime occurredOn, Guid moderatedOfferId, ulong price, bool isVisible)
        : base(id, occurredOn) { ... }
}
```

Только данные, только `get`, никакого поведения. Между доменным и интеграционным событием есть промежуточное звено — `IDomainEventNotification<T>`, обёртка, которая и попадает в Outbox.

---

## DomainEventsDispatcher: где всё сходится

`BuildingBlocks.Infrastructure/DomainEventsDispatching/DomainEventsDispatcher.cs` — самый важный класс инфраструктуры. Вызывается из `UnitOfWork.CommitAsync`, то есть на каждой команде.

```csharp
public async Task DispatchEventsAsync()
{
    var domainEvents = _domainEventsProvider.GetAllDomainEvents();

    var domainEventNotifications = new List<IDomainEventNotification<IDomainEvent>>();

    foreach (var domainEvent in domainEvents)
    {
        // 1. Аудит: события с маркером ILoggableDomainEvent пишутся в лог доменных событий
        if (domainEvent is ILoggableDomainEvent)
        {
            var data = JsonConvert.SerializeObject(domainEvent, ...);
            await _domainEventLogger.AddAsync(new DomainEventLogEntry(
                domainEvent.Id, domainEvent.OccurredOn, domainEvent.GetType().FullName!, data));
        }

        // 2. Есть ли у события внешняя обёртка IDomainEventNotification<TEvent>?
        var notificationType = typeof(IDomainEventNotification<>).MakeGenericType(domainEvent.GetType());
        var domainNotification = _scope.ResolveOptional(notificationType, new List<Parameter>
        {
            new NamedParameter("domainEvent", domainEvent),
            new NamedParameter("id", domainEvent.Id),
        });

        if (domainNotification != null)
            domainEventNotifications.Add((domainNotification as IDomainEventNotification<IDomainEvent>)!);
    }

    // 3. Очистить список у сущностей — до публикации, чтобы не уйти в цикл
    _domainEventsProvider.ClearAllDomainEvents();

    // 4. Публикация ВНУТРИ модуля — обработчики выполнятся в этой же транзакции
    foreach (var domainEvent in domainEvents)
        await _mediator.Publish(domainEvent);

    // 5. Внешние события — в Outbox, той же транзакцией
    foreach (var notification in domainEventNotifications)
    {
        var type = _domainNotificationsMapper.GetName(notification.GetType());
        var data = JsonConvert.SerializeObject(notification, ...);

        _outbox.Add(new OutboxMessage(
            notification.Id, notification.DomainEvent.OccurredOn, type, data));
    }
}
```

Пять шагов, каждый со смыслом:

**Шаг 2, `ResolveOptional`** — ключевой. Не у каждого доменного события есть внешняя пара. Если в контейнере не зарегистрирован `IDomainEventNotification<OrderApprovedEvent>` — событие останется внутренним. Именно так решается, наружу оно уходит или нет: **наличием регистрации, а не флагом в коде события**.

**Шаг 3, `ClearAllDomainEvents` до публикации** — защита от бесконечного цикла. Обработчик события может изменить агрегат и добавить новое событие; если не очистить список заранее, следующий проход подхватит и старые.

**Шаг 4** — обработчики доменных событий выполняются **до** `SaveChanges`, внутри транзакции. Отсюда требование к ним: быстро и без внешних вызовов.

### Карта имён вместо имён типов

В Outbox пишется не `Type.FullName`, а имя из явной карты. `OrdersStartup.cs`:

```csharp
var domainNotificationsMap = new BiDictionary<string, Type>();

domainNotificationsMap.Add("OrderApprovedNotification", typeof(OrderApprovedNotification));
domainNotificationsMap.Add("OrderItemsReservedNotification", typeof(OrderItemsReservedNotification));
domainNotificationsMap.Add("NasiyaTransactionApprovedNotification", typeof(NasiyaApplicationApprovedNotification));
// ...около 27 записей

containerBuilder.RegisterModule(new OutboxModule(domainNotificationsMap));
```

`BiDictionary` — двусторонний словарь: имя → тип при чтении, тип → имя при записи.

Зачем это, если `FullName` есть бесплатно: **переименование или перенос C#-класса не должно ломать разбор сообщений, которые уже лежат в таблице**. В карте видно живой пример: строка `"NasiyaTransactionApprovedNotification"` указывает на класс `NasiyaApplicationApprovedNotification` — класс переименовали, ключ оставили, старые сообщения продолжают читаться.

Обратная сторона: **забыть добавить строку в карту — типовая ошибка**. `GetName` бросит `InvalidOperationException` в момент коммита, то есть новое событие уронит команду целиком. Ошибка находится быстро, но только когда сценарий выполнится.

---

## Outbox: отправка

`Processing/Outbox/ProcessOutboxCommandHandler.cs`, запускается джобой `ProcessOutboxJob` в Quartz ([[Планировщики задач: Hangfire, Quartz]]):

```csharp
const string sql = "select id, type, data from outbox_messages " +
                   "where processed_date is null order by occurred_on limit 500 for update";

var messages = await connection.QueryAsync<OutboxMessageDto>(sql);

foreach (var message in messagesList)
{
    var type = _domainNotificationsMapper.GetType(message.Type);
    var @event = JsonConvert.DeserializeObject(message.Data, type) as IDomainEventNotification;

    if (@event is null) continue;

    using (LogContext.Push(new OutboxMessageContextEnricher(@event)))
    {
        await _mediator.Publish(@event, cancellationToken);

        await connection.ExecuteAsync(
            "update outbox_messages set processed_date = @Date where id = @Id",
            new { Date = DateTime.UtcNow, message.Id });
    }
}
```

Детали, каждая из которых важна:

- **`order by occurred_on`** — порядок публикации соответствует порядку возникновения. Без этого «заказ отменён» могло бы уехать раньше «заказ создан».
- **`limit 500`** — батч. Без лимита джоба на просевшей очереди попыталась бы поднять в память всё.
- **`for update`** — блокировка строк на время транзакции, чтобы два экземпляра приложения (а в Kubernetes их несколько) не разослали одни и те же сообщения. Единственная защита от дублей при горизонтальном масштабировании.
- **`LogContext.Push`** — Serilog-обогащение: все логи внутри обработки помечаются `OutboxMessage:{id}`. Без этого разобрать в Kibana, что делала джоба, невозможно ([[Serilog]]).
- **`processed_date` обновляется после публикации** — именно этот порядок даёт at-least-once.

### At-least-once и почему это принципиально

Если процесс умрёт между `Publish` и `update`, сообщение останется с `processed_date = null` и будет отправлено повторно. Это **сознательный выбор**: потерять сообщение хуже, чем доставить дважды.

Обратный порядок (сначала отметка, потом публикация) дал бы at-most-once — сообщение могло бы потеряться безвозвратно. Exactly-once в распределённой системе не достигается в общем виде; достигается «at-least-once + идемпотентный получатель», что практически эквивалентно.

Отсюда прямое следствие для всего кода приложения: **любой обработчик должен быть идемпотентным**. Именно поэтому в методах агрегатов стоят проверки `if (Status.Equals(OrderStatus.Approved)) return;` — это не защита от кривого фронта, а требование архитектуры. Подробно — [[Идемпотентность]].

---

## Шина между модулями

`IEventsBus` — абстракция из трёх методов:

```csharp
public interface IEventsBus : IDisposable
{
    Task Publish<T>(T @event) where T : IntegrationEvent;
    void Subscribe<T>(IIntegrationEventHandler<T> handler) where T : IntegrationEvent;
    void StartConsuming();
}
```

Две реализации:

**`InMemoryEventBusClient`** — по умолчанию, обмен внутри процесса. Внутри — синглтон со словарём `Dictionary<string, List<IIntegrationEventHandler>>`, ключ — `Type.FullName` события. Публикация вызывает всех подписчиков напрямую.

**Kafka через KafkaFlow** — для обмена с внешними сервисами Alif ([[Apache Kafka: основы]]).

> [!warning] У `InMemoryEventBus.Publish` нет проверки на отсутствие ключа
> ```csharp
> var integrationEventHandlers = _handlersDictionary[eventType];   // KeyNotFoundException
> ```
> Индексатор словаря вместо `TryGetValue`. Если событие опубликовано, но ни один модуль на него не подписался — `KeyNotFoundException` в момент публикации. Практический вывод: **новое интеграционное событие требует хотя бы одной подписки**, иначе первая же публикация уронит обработку Outbox. Опубликовать «на будущее, потом подпишемся» — не получится.

### Подписки объявляются вручную

`Cart.Infrastructure/Configuration/EventsBus/EventsBusStartup.cs`:

```csharp
private static void SubscribeToIntegrationEvents(ILogger logger)
{
    var eventBus = CartCompositionRoot.BeginLifetimeScope().Resolve<IEventsBus>();

    SubscribeToIntegrationEvent<ModeratedOfferPriceChangedIntegrationEvent>(eventBus, logger);
    SubscribeToIntegrationEvent<ModeratedOfferQuantityChangedIntegrationEvent>(eventBus, logger);
    SubscribeToIntegrationEvent<ModeratedOfferDeactivatedIntegrationEvent>(eventBus, logger);
    SubscribeToIntegrationEvent<ModeratedOfferTransferredToAnotherStoreIntegrationEvent>(eventBus, logger);
    // ...около 20 подписок
}

private static void SubscribeToIntegrationEvent<T>(IEventsBus eventBus, ILogger logger)
    where T : IntegrationEvent
{
    logger.Information("Subscribe to {@IntegrationEvent}", typeof(T).FullName);
    eventBus.Subscribe(new IntegrationEventGenericHandler<T>());
}
```

Явный список вместо автосканирования сборок — сознательное решение: подписки видны в одном файле, и «магически» подписаться нельзя. Цена — про этот файл легко забыть, и тогда событие тихо не доходит.

> [!tip] Первое, что проверять при «событие не дошло»
> Открыть `EventsBusStartup` модуля-получателя и убедиться, что строка `SubscribeToIntegrationEvent<...>` там есть. В коде Cart над блоком подписок даже стоит комментарий `// ModeratedOffer consistency events - missing subscriptions fixed` — то есть на этом уже спотыкались, и симптом был именно такой: события публиковались, обработчики существовали, но не вызывались.

---

## Inbox: приём

Обработчик на стороне получателя **не обрабатывает событие**, а только записывает его в таблицу:

```csharp
internal class IntegrationEventGenericHandler<T> : IIntegrationEventHandler<T>
    where T : IntegrationEvent
{
    public async Task Handle(T @event)
    {
        using var scope = CartCompositionRoot.BeginLifetimeScope();
        using var connection = scope.Resolve<ISqlConnectionFactory>().GetOpenConnection();

        var type = @event.GetType().FullName;
        var data = JsonConvert.SerializeObject(@event, ...);

        const string sql = "insert into inbox_messages (id, occurred_on, type, data) " +
                           "values (@Id, @OccurredOn, @Type, @Data)";

        await connection.ExecuteScalarAsync(sql, new { @event.Id, @event.OccurredOn, type, data });
    }
}
```

Приём отделён от обработки. Что это даёт: отправитель узнаёт «принято» за один INSERT и не ждёт бизнес-логики; если обработка упадёт, сообщение останется в таблице и будет переиграно; получатель может быть занят или временно неработоспособен — сообщения накопятся, а не потеряются.

Здесь `Type.FullName`, а не карта имён — потому что тип интеграционного события и так публичный контракт, который не переименовывают.

Разбор — `ProcessInboxCommandHandler`, джоба `ProcessInboxJob`:

```csharp
const string sql = "select id, type, data from inbox_messages where processed_date is null order by occurred_on limit 500";

foreach (var message in messages)
{
    var messageAssembly = AppDomain.CurrentDomain.GetAssemblies()
        .SingleOrDefault(assembly => message.Type.Contains(assembly.GetName().Name!));

    var type = messageAssembly!.GetType(message.Type);
    var request = JsonConvert.DeserializeObject(message.Data, type!);

    await _mediator.Publish((INotification)request!, cancellationToken);

    await connection.ExecuteScalarAsync(
        "update inbox_messages set processed_date = @Date where id = @Id",
        new { Date = DateTime.UtcNow, message.Id });
}
```

> [!danger] Поиск сборки через `Contains` и `SingleOrDefault`
> Строка `assembly.GetName().Name!` ищется методом `Contains` в полном имени типа, а результат берётся через `SingleOrDefault` с последующим `!`. Два способа получить исключение: если ни одна сборка не подошла — `NullReferenceException` на `messageAssembly!`; если подошли две (а имена сборок здесь вложенные: `Alif.ServiceShop.Modules.Catalog.IntegrationEvents` содержит в себе `Alif.ServiceShop.Modules.Catalog`) — `SingleOrDefault` бросит `InvalidOperationException`.
> Работает потому, что имена сборок сейчас различаются достаточно. Хрупкое место: одно неудачное имя новой сборки — и разбор Inbox встанет целиком, для всех сообщений. Обрати внимание при добавлении новых проектов в модуль.

Дальше событие публикуется через MediatR внутри модуля и попадает в `INotificationHandler<T>`:

```csharp
public class ChangeModeratedItemPriceOnModeratedOfferPriceChangedIntegrationEventHandler
    : INotificationHandler<ModeratedOfferPriceChangedIntegrationEvent>
{
    private readonly ICommandsScheduler _commandsScheduler;

    public async Task Handle(ModeratedOfferPriceChangedIntegrationEvent notification, CancellationToken ct)
    {
        await _commandsScheduler.EnqueueAsync(new ChangeModeratedItemPriceCommand(
            Guid.NewGuid(), notification.ModeratedOfferId, notification.Price));
    }
}
```

Обработчик не делает работу — он ставит команду в `internal_commands`. Причина: изменение цены товара затрагивает тысячи корзин, и делать это внутри обработки одного сообщения Inbox значит держать транзакцию и блокировать разбор остальных сообщений. Отложенная команда делает обработку события константной по времени.

Имена таких обработчиков описывают всю связь целиком: `Change<Что>On<КакоеСобытие>IntegrationEventHandler`. Длинно, зато по имени файла видно, кто на что реагирует.

---

## Полная цепочка

```
МОДУЛЬ CATALOG                                    МОДУЛЬ CART
════════════════                                  ═══════════

ModeratedOffer.ChangePrice()
  └─ AddDomainEvent(PriceChangedDomainEvent)
        │
        ▼  UnitOfWork.CommitAsync
   DomainEventsDispatcher
        ├─ MediatR.Publish → внутренние обработчики (эта же транзакция)
        └─ INSERT outbox_messages ─────┐
                                       │ ОДНА ТРАНЗАКЦИЯ с бизнес-данными
   COMMIT ─────────────────────────────┘
        │
        ▼  ProcessOutboxJob (Quartz, периодически)
   SELECT ... WHERE processed_date IS NULL ORDER BY occurred_on LIMIT 500 FOR UPDATE
        │
        ▼  MediatR.Publish(notification)
   Notification handler → IEventsBus.Publish(IntegrationEvent)
        │
        ▼  InMemoryEventBus (или Kafka)
        └──────────────────────────────▶  IntegrationEventGenericHandler<T>
                                                │
                                                ▼  INSERT inbox_messages
                                          COMMIT
                                                │
                                                ▼  ProcessInboxJob (Quartz)
                                          MediatR.Publish
                                                │
                                                ▼
                                          INotificationHandler<T>
                                                │
                                                ▼  ICommandsScheduler.EnqueueAsync
                                          INSERT internal_commands
                                                │
                                                ▼  ProcessInternalCommandsJob (Quartz)
                                          CommandHandler → агрегат Cart
```

Три таблицы, три джобы, три границы транзакций. Каждая граница — точка, где система может упасть и продолжить без потерь. Ни одной прямой ссылки Catalog → Cart в коде нет.

Цена: **задержка**. От изменения цены в каталоге до обновления цены в корзине проходит минимум три тика Quartz. Это [[Согласованность данных, CAP и eventual consistency]] в чистом виде — согласованность есть, но не мгновенная. Продуктовое требование «цена в корзине обновляется сразу» этой архитектурой не выполняется в принципе, и это надо знать до того, как обещать сроки.

---

## Kafka: обмен с внешними сервисами

Модули Alif вне ServiceShop (платежи, доставка, кредиты) общаются через Kafka. Структура в `Orders.Infrastructure/Configuration/KafkaBus/Consume/` — по стриму на источник:

```
ApplicationStream/         кредитные заявки Nasiya
CreditsStream/             кредиты и BML
DeliveryStream/            доставка
ClientsPhonesShiftStream/  смена телефонов клиентов
```

Каждый стрим — одинаковый набор из пяти файлов:

| Файл | Роль |
|---|---|
| `*EventRegistry.cs` | какие типы событий ожидаются в этом стриме |
| `*Middleware.cs` | обработка сообщения в конвейере KafkaFlow |
| `*Resolver.cs` | по содержимому сообщения определить конкретный тип |
| `*DeserializatorExceptionHandler.cs` | что делать с сообщением, которое не разобралось |
| `*ConsumerGroupOptions.cs` | настройки consumer group |

Отдельный consumer group на стрим — стандартная практика Kafka: отставание по одному потоку не тормозит остальные, и масштабировать их можно независимо.

Плюс `OrdersShopDeadLetterQueueConventions` — **DLQ (dead letter queue)**: сообщения, которые не удалось обработать после ретраев, уезжают в отдельный топик, а не блокируют партицию бесконечными повторами.

Про устойчивость и ретраи — [[Отказоустойчивость: retry, backoff, circuit breaker]].

---

> [!example] Как делают в бою
> Диагностика «событие не доехало» по цепочке, от начала к концу — на каждом шаге либо находится проблема, либо она отсекается:
> 1. `select * from outbox_messages where processed_date is null order by occurred_on` в базе **отправителя**. Есть записи и они не убывают → джоба Outbox стоит или падает. Записей нет вовсе → событие не сгенерировано: проверить, что `AddDomainEvent` вызван и в `domainNotificationsMap` есть строка.
> 2. `EventsBusStartup` **получателя** — есть ли подписка.
> 3. `select * from inbox_messages where processed_date is null` в базе получателя. Пусто → до Inbox не дошло, проблема выше. Копится → джоба Inbox стоит.
> 4. `select * from internal_commands where processed_date is null` — обработчик поставил команду, но она не выполнилась.
>
> Три таблицы — это ещё и три точки наблюдаемости. Метрика «количество необработанных записей» по каждой из них — самый полезный алерт в такой архитектуре: он ловит остановку конвейера раньше, чем жалобы пользователей. Про метрики — [[Наблюдаемость: логи, метрики, трейсы]].

> [!warning] Подводные камни
> - **Забыть строку в `domainNotificationsMap`** — `InvalidOperationException` на коммите команды. Событие новое, а падает вся команда.
> - **Опубликовать событие без подписчиков** — `KeyNotFoundException` в `InMemoryEventBus.Publish`, потому что там индексатор вместо `TryGetValue`.
> - **Тяжёлый обработчик доменного события** — выполняется внутри транзакции команды, до `SaveChanges`. Внешний HTTP-вызов оттуда удлиняет транзакцию и держит блокировки MySQL.
> - **Изменение полей интеграционного события** ломает разбор сообщений, уже лежащих в `outbox_messages`/`inbox_messages`. Совместимость: только добавление опциональных полей.
> - **At-least-once означает дубли.** Обработчик, который не идемпотентен, при обычном рестарте пода даст двойное списание или два уведомления.
> - **`for update` в Outbox есть, в Inbox — нет.** `ProcessInboxJob` читает без блокировки. Два экземпляра приложения могут разобрать одно сообщение параллельно — ещё одна причина требовать идемпотентности.
> - **Три тика Quartz задержки.** Требования вида «должно обновиться немедленно» этой цепочкой не закрываются.

---

## Вопросы с собеседований

> [!question]- Какую проблему решает Outbox и почему нельзя просто отправить в брокер после `SaveChanges`?
> База и брокер — разные ресурсы без общей транзакции. Между коммитом в БД и отправкой в брокер процесс может умереть: данные сохранены, событие потеряно, и восстановить его нечем. Обратный порядок хуже — событие уходит, а транзакция откатывается, и потребители узнают о том, чего не было. Outbox сводит два ресурса к одному: событие пишется в таблицу той же базы в той же транзакции, поэтому либо коммитятся оба, либо ни одного. Отдельный процесс потом читает таблицу и рассылает. 2PC теоретически решил бы это, но Kafka его не поддерживает, и цена распределённой транзакции выше выгоды.

> [!question]- Что такое at-least-once и почему проект выбрал именно это?
> At-least-once — гарантия «сообщение доставлено минимум один раз», возможны дубли. Возникает из порядка: сначала публикация, потом отметка `processed_date`. Падение между ними даёт повторную отправку. Альтернатива at-most-once (сначала отметка) допускала бы безвозвратную потерю. Exactly-once в распределённой системе в общем виде недостижим; достижимо «at-least-once + идемпотентный получатель», что практически эквивалентно. Именно поэтому в методах агрегатов ServiceShop стоят проверки текущего состояния перед изменением.

> [!question]- Зачем Inbox, если Outbox уже гарантировал доставку?
> Outbox гарантирует, что сообщение **отправлено**, но не что получатель его **обработал**. Без Inbox обработка идёт синхронно внутри вызова `Publish`: падение бизнес-логики получателя означает потерю сообщения, а отправитель ждёт всю обработку. Inbox разделяет приём и обработку: приём — один INSERT, обработка — отдельная джоба, которая при падении переиграет сообщение из таблицы. Плюс Inbox — естественное место для дедупликации по `Id`, если строгая однократность нужна.

> [!question]- Чем доменное событие отличается от интеграционного?
> Аудиторией и, как следствие, стабильностью контракта. Доменное живёт внутри модуля, публикуется через MediatR в памяти в той же транзакции, и его можно свободно рефакторить — снаружи никто не видит. Интеграционное — публичный контракт: его знают другие модули, а сериализованные экземпляры лежат в таблицах и будут разобраны после деплоя. Поэтому в интеграционном событии допустимо только добавлять опциональные поля; удаление или переименование поля сломает разбор сообщений, уже стоящих в очереди.

> [!question]- Зачем в Outbox карта имён вместо `Type.FullName`?
> Чтобы переименование или перенос C#-класса не ломало разбор сообщений, уже лежащих в таблице. В `outbox_messages` записана строка из карты, и при чтении по ней восстанавливается актуальный тип — класс можно переименовать, оставив ключ прежним. В `OrdersStartup` виден живой пример: ключ `"NasiyaTransactionApprovedNotification"` указывает на класс `NasiyaApplicationApprovedNotification`. Цена решения — ручное сопровождение: забытая строка даёт исключение в момент коммита.

> [!question]- Почему в запросе Outbox стоит `for update` и что будет без него?
> `for update` блокирует выбранные строки до конца транзакции. В Kubernetes работает несколько экземпляров приложения, и джоба Outbox тикает в каждом. Без блокировки два экземпляра выберут одни и те же 500 необработанных сообщений и опубликуют их дважды — дубли не из-за сбоя, а систематически, на каждом тике. Показательно, что в `ProcessInboxJob` такой блокировки нет: там на дубли рассчитывают, полагаясь на идемпотентность обработчиков.

> [!question]- Почему обработчик интеграционного события ставит команду в очередь вместо выполнения работы?
> Чтобы обработка одного сообщения занимала константное время. Событие «цена товара изменилась» затрагивает тысячи корзин; сделать это внутри обработки сообщения — держать длинную транзакцию и заблокировать разбор остальных сообщений Inbox. Обработчик кладёт команду в `internal_commands` — один INSERT в текущей транзакции, гарантия сохраняется — и завершается за миллисекунды. Работу делает `ProcessInternalCommandsJob`.

---

## Задачи

### Задача 1. Атомарность

Разработчик решил, что Outbox — лишняя сложность, и заменил его на прямую публикацию в конце хендлера, сразу после `SaveChanges`. Какой конкретный сценарий сломается и как это проявится в проде?

> [!success]- Решение
> Сломается атомарность «данные + событие». Сценарий: заказ создан, `SaveChanges` прошёл, под перезапущен (деплой, OOM, вытеснение) до строки публикации. Заказ в базе есть, событие `OrderCreated` не отправлено — навсегда, восстановить нечем.
>
> Проявление в проде: заказ виден в админке, но корзина клиента не очищена, уведомление не пришло, товар не зарезервирован. При этом ошибок в логах нет — запрос завершился успешно. Такие случаи выглядят как «плавающий баг раз в неделю» и почти не воспроизводятся: частота равна частоте рестартов, помноженной на вероятность попадания в узкое окно между двумя строками.
>
> Второй, менее очевидный вариант того же кода: публикация **до** `SaveChanges`. Тогда при откате транзакции событие уже улетело, и потребители обработают заказ, которого нет в базе. Это хуже — не потеря, а рассогласование данных между модулями.

### Задача 2. Новое интеграционное событие

Нужно, чтобы при отмене заказа Marketing отправлял клиенту уведомление. Перечислить все шаги, включая те, о которых легко забыть.

> [!success]- Решение
> **В Orders (отправитель):**
> 1. Доменное событие `OrderCancelledDomainEvent` в `Domain/Order/DomainEvents/`, вызов `AddDomainEvent(...)` в `Order.CancelAsync`.
> 2. Обёртка `OrderCancelledNotification : DomainNotificationBase<OrderCancelledDomainEvent>` в `Application/Notifications/Orders/`.
> 3. **Строка в `domainNotificationsMap` в `OrdersStartup.cs`** — без неё падение на коммите.
> 4. Интеграционное событие `OrderCancelledIntegrationEvent : IntegrationEvent` в `Orders.IntegrationEvents/`.
> 5. Обработчик нотификации, публикующий интеграционное событие в `IEventsBus`.
>
> **В Marketing (получатель):**
> 6. **Подписка `SubscribeToIntegrationEvent<OrderCancelledIntegrationEvent>` в `EventsBusStartup`** — без неё событие тихо не дойдёт. И одновременно: без хотя бы одной подписки публикация упадёт с `KeyNotFoundException`.
> 7. `INotificationHandler<OrderCancelledIntegrationEvent>`, который ставит команду через `ICommandsScheduler`.
> 8. Команда и хендлер, отправляющие уведомление.
>
> Забывают обычно шаги 3 и 6 — они в файлах конфигурации, а не рядом с бизнес-кодом. Симптомы разные: 3 даёт падение команды, 6 — тишину.

### Задача 3. Идемпотентность обработчика

Обработчик начисляет бонусы за завершённый заказ:

```csharp
public async Task Handle(OrderCompletedIntegrationEvent notification, CancellationToken ct)
{
    var client = await _clientRepository.GetByIdAsync(notification.ClientId);
    client.AddBonuses(notification.TotalAmount / 100);
    await _clientRepository.SaveAsync(client);
}
```

Что произойдёт при повторной доставке и как исправить?

> [!success]- Решение
> Бонусы начислятся дважды. Outbox даёт at-least-once, а `ProcessInboxJob` читает без `for update` — повтор гарантированно случится при рестарте пода или при работе двух экземпляров приложения. Это прямые финансовые потери, и обнаружатся они не сразу.
>
> Исправление — сделать операцию идемпотентной по идентификатору события, а не по данным. Правильный способ: хранить факт начисления вместе с `Id` события и проверять его в той же транзакции.
>
> ```csharp
> public async Task Handle(OrderCompletedIntegrationEvent notification, CancellationToken ct)
> {
>     if (await _bonusAccrualRepository.ExistsForEventAsync(notification.Id))
>         return;
>
>     var client = await _clientRepository.GetByIdAsync(notification.ClientId);
>     client.AddBonuses(notification.TotalAmount / 100);
>
>     await _bonusAccrualRepository.AddAsync(new BonusAccrual(notification.Id, notification.ClientId));
>     await _clientRepository.SaveAsync(client);
> }
> ```
>
> Проверка «а не начислены ли уже бонусы за этот заказ» тоже работает, но хуже: она сломается, если по одному заказу начисление легально возможно дважды. Уникальный индекс по `event_id` в таблице начислений закрывает и гонку двух параллельных обработчиков — второй упадёт на нарушении уникальности, что здесь правильное поведение.

---

## Итог

- Модули обмениваются событиями через таблицы БД, а не вызовами: `outbox_messages` → шина → `inbox_messages` → `internal_commands`.
- Outbox решает проблему двух ресурсов: событие пишется в ту же базу той же транзакцией, что и бизнес-данные.
- `DomainEventsDispatcher` вызывается из `UnitOfWork.CommitAsync`: публикует доменные события внутри модуля и пишет внешние в Outbox.
- Наружу событие уходит только если в контейнере зарегистрирован `IDomainEventNotification<TEvent>` — решает регистрация, а не флаг.
- Гарантия at-least-once: дубли возможны и нормальны, поэтому все обработчики обязаны быть идемпотентными.
- В Outbox есть `for update` (защита от дублей при нескольких инстансах), в Inbox — нет.
- Карта `domainNotificationsMap` развязывает имя сообщения от имени C#-класса; забытая строка = падение на коммите.
- Плата за схему — задержка в три тика Quartz и eventual consistency между модулями.

## Связанное

- [[93 — Рабочий проект: Alif ServiceShop (обзор раздела)]]
- [[ServiceShop: CQRS и путь запроса]]
- [[ServiceShop: домен и тактический DDD]]
- [[Паттерн Transactional Outbox]]
- [[Идемпотентность]]
- [[Доменные события]]
- [[Apache Kafka: основы]]
- [[Согласованность данных, CAP и eventual consistency]]
- [[Отказоустойчивость: retry, backoff, circuit breaker]]
- [[Планировщики задач: Hangfire, Quartz]]
