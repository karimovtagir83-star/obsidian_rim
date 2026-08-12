---
tags: [раздел-93, рабочий-проект, ddd, домен, агрегаты, middle, собес]
aliases: [Домен ServiceShop, DDD в ServiceShop, Агрегаты ServiceShop, Order aggregate, Business Rules ServiceShop]
---

# ServiceShop: домен и тактический DDD

> [!abstract] Коротко
> Домен ServiceShop — это учебник по тактическому DDD, только рабочий. Базовые кирпичи (`Entity`, `ValueObject`, `IBusinessRule`, `IDomainEvent`) лежат в `BuildingBlocks.Domain` и занимают по 20–100 строк каждый — их стоит прочитать целиком в первый же день.
> Главное, что нужно унести: **у метода агрегата в этом проекте всегда одна и та же форма** — проверить правила через `CheckRuleAsync`, изменить своё состояние, объявить факт через `AddDomainEvent`. Никаких обращений к БД, HTTP или другим модулям внутри домена. Одно бизнес-правило = один класс с говорящим именем.

## Зачем это нужно

Альтернатива, от которой ушли, называется **анемичная модель** (anemic domain model): класс — набор публичных свойств, вся логика в `OrderService`. Она проще на старте и разваливается предсказуемо:

- правило «заказ нельзя отменить после отгрузки» проверяется в четырёх местах и в трёх из них устарело;
- чтобы проверить одно арифметическое условие, нужен поднятый MySQL и HTTP-запрос;
- объект можно перевести в невалидное состояние снаружи — просто присвоив свойство.

DDD переносит инварианты внутрь объекта: **невалидное состояние невозможно создать**, потому что снаружи нет ни публичного конструктора, ни публичных сеттеров. Теория — [[DDD: тактические паттерны]].

В ServiceShop это оправдано конкретной сложностью: у заказа 15 статусов, 5 способов оплаты и около 30 бизнес-правил. При таком объёме анемичная модель не масштабируется.

---

## Entity

`BuildingBlocks.Domain/Entity.cs` — читается за минуту:

```csharp
public abstract class Entity
{
    private readonly List<IDomainEvent> _domainEvents = new();

    [JsonIgnore]
    public IReadOnlyCollection<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    public void ClearDomainEvents() => _domainEvents.Clear();

    protected void AddDomainEvent(IDomainEvent domainEvent) => _domainEvents.Add(domainEvent);

    protected static async Task CheckRuleAsync(IBusinessRule rule)
    {
        if (await rule.IsBrokenAsync())
            throw new BusinessRuleValidationException(rule);
    }
}

public abstract class Entity<TIndex> : Entity
{
    public TIndex Id { get; protected init; } = default!;
}
```

Три вещи, которые даёт базовый класс:

1. **Список доменных событий** — «что со мной произошло». Наружу отдаётся `IReadOnlyCollection`, добавлять может только наследник через `protected AddDomainEvent`. `[JsonIgnore]` — чтобы события не улетали в API-ответ.
2. **`CheckRuleAsync`** — единая точка проверки правил. Все нарушения бросают одно исключение `BusinessRuleValidationException`, которое в `Startup.cs` маппится в предсказуемый HTTP-ответ через ProblemDetails (см. [[Обработка ошибок и ProblemDetails]]).
3. **Дженерик-версия с `Id`** — потому что `Id` у разных агрегатов разного типа: у `Order` это `int`, у `Cart` — `string` (ObjectId из MongoDB), у некоторых — `Guid`.

Обрати внимание: `CheckRuleAsync` — **`static`** и `protected`. Статический потому, что не использует состояние сущности; правило само получает всё нужное через конструктор.

`Id { get; protected init; }` — `init`-аксессор (C# 9): присвоить можно только при создании объекта, дальше только чтение. Строже, чем `private set`.

---

## Value Object

Два разных базовых класса под разные задачи.

**`ValueObject`** — сравнение по значению всех полей через рефлексию:

```csharp
public override bool Equals(object? obj)
{
    if (obj == null || GetType() != obj.GetType()) return false;

    return GetProperties().All(p => PropertiesAreEqual(obj, p))
           && GetFields().All(f => FieldsAreEqual(obj, f));
}
```

Список свойств и полей кешируется в приватных полях `_properties`/`_fields` — рефлексия дорогая, вызывать её на каждом `Equals` нельзя. Исключить поле из сравнения можно атрибутом `[IgnoreMember]`.

**`GuidValueObject`** — узкоспециализированный, для идентификаторов:

```csharp
protected GuidValueObject(Guid value)
{
    if (value == Guid.Empty)
        throw new InvalidOperationException("Id value cannot be empty!");

    Value = value;
}
```

Смысл — **невалидный идентификатор невозможно создать**. Проверка в конструкторе, не в валидаторе и не в хендлере: если объект существует, он валиден. Это принцип «делай невалидные состояния непредставимыми».

### Зачем это вообще, если можно `Guid`

Сравни две подписи:

```csharp
// плохо: компилятор не заметит, если аргументы перепутать местами
void Process(Guid orderId, Guid clientId, Guid offerId);

// хорошо: перепутать невозможно, ошибка на этапе компиляции
void Process(OrderUuid orderId, ClientId clientId, ModeratedOfferId offerId);
```

Три `Guid` в подписи — мина. Три разных типа — гарантия. Плюс тип становится местом, куда можно положить проверки и поведение.

Value Objects модуля заказов лежат в `Orders.Domain/Order/ValueObjects/`. Про иммутабельность как приём — [[Иммутабельность как приём проектирования]].

---

## Aggregate Root

`IAggregateRoot` — пустой маркерный интерфейс:

```csharp
public interface IAggregateRoot { }
```

Пустой интерфейс здесь не бесполезен: по нему ArchTests находят все агрегаты в сборке и проверяют их правила, а Autofac-регистрации используют его для поиска типов. Это классический **маркер** — способ пометить типы так, чтобы метку можно было прочитать рефлексией.

**Агрегат** — группа объектов, изменяемых только вместе и только через корень. `Order` — корень, `OrderItem`, `OrderDelivery`, `AlifNasiyaTransaction` — части. Снаружи нельзя взять `OrderItem` и поменять ему количество; только `order.AddOrderItem(...)`.

Агрегат — это **граница транзакции**: в одной операции сохраняется один агрегат, и он сам отвечает за свою консистентность.

### Правило 1: только приватные конструкторы

```csharp
public partial class Order : Entity<int>, IAggregateRoot
{
    private Order() { }                    // для EF Core
    private Order(int partnerId, ...) { }   // рабочий

    public static Order PlaceAsPendingPayment(...) { ... }
    public static Order CreateAlifMobiOrder(...) { ... }
    public static Order CreateBnplNasiyaOrder(...) { ... }
    public static Order CreateBmlNasiyaOrder(...) { ... }
}
```

Три причины, по которым создание идёт через фабричные методы:

1. **Конструктор не может быть `async`.** Создание заказа требует `await CheckRuleAsync(...)` — асинхронной проверки правил, часто с запросом в БД (например, «хватает ли товара на складе»). Фабричный метод может быть `async`, конструктор — нет. Это техническая, а не идеологическая причина, и она решающая.
2. **Имя объясняет сценарий.** `new Order(...)` не говорит ничего. `PlaceAsPendingPayment(...)` и `CreateBmlNasiyaOrder(...)` — говорят всё. Четыре способа создать заказ — четыре метода с разными наборами параметров.
3. **Контроль инвариантов.** Ни один путь создания не обходит проверки.

Правило проверяется тестом `OnlyPrivateConstructorsTest`, с одним исключением: у `Cart` разрешён публичный конструктор с атрибутом `[BsonConstructor]` — его требует драйвер MongoDB для десериализации.

### Правило 2: ссылки на другие агрегаты только по ID

В `Order` нет поля `Client Client` — есть `int ClientId`. Нет `Partner Partner` — есть `int PartnerId`.

Зачем: если разрешить навигационные свойства между агрегатами, EF Core потянет половину базы одним запросом (`Include` по цепочке), а граница транзакции расплывётся — станет непонятно, кто кого сохраняет. Ссылка по ID делает границу явной: чтобы получить клиента, нужен отдельный запрос через свой репозиторий или фасад модуля.

> [!info] Почему `Order` — это `partial class`
> Файл разбит на три: `Order.cs` (основное), `Order.Delivery.cs` (логика доставки), `Order.Fake.cs` (фабрика тестовых объектов). `partial` позволяет держать один класс в нескольких файлах. Это способ жить с большим агрегатом, а не архитектурное решение: 900 строк в одном файле не читаются. Сигнал к тому, что агрегат перегружен, но разбивать его на два — задача с высоким риском.

---

## Business Rule

```csharp
public interface IBusinessRule
{
    Task<bool> IsBrokenAsync();
    string Message { get; }
}
```

Метод отвечает на вопрос «правило **нарушено**?» — не «выполнено?». Легко перепутать при написании нового правила, а тест на него часто пишут «зеркально», и ошибка проходит.

Пример — `Cart.Domain/Aggregates/Cart/BusinessRules/ModeratedOfferMustBeAvailableRule.cs`:

```csharp
public class ModeratedOfferMustBeAvailableRule : IBusinessRule
{
    private readonly Guid _moderatedOfferId;
    private readonly ushort _moderatedOfferAvailableQuantity;
    private readonly ushort _requestedQuantity;

    public ModeratedOfferMustBeAvailableRule(
        Guid moderatedOfferId, ushort moderatedOfferAvailableQuantity, ushort requestedQuantity)
    { ... }

    public async Task<bool> IsBrokenAsync()
        => await Task.FromResult(_requestedQuantity > _moderatedOfferAvailableQuantity);

    public string Message =>
        $"Offer is out of warehouse. Available quantity: {_moderatedOfferAvailableQuantity}, offerId: {_moderatedOfferId}";
}
```

### Почему класс, а не `if` внутри метода

| `if` в методе | Отдельный класс правила |
|---|---|
| смысл прячется в условии | имя класса = формулировка требования |
| повторное использование — copy-paste | правило подключается в любой метод одной строкой |
| тест требует поднять весь сценарий | тест на правило — три строки |
| каждое нарушение бросает своё исключение | всегда `BusinessRuleValidationException` → единый ответ API |

Практическая ценность видна на списке файлов `Orders.Domain/Order/BusinessRules/` — это буквально текст бизнес-требований:

```
OrderMustBeApprovedForDeliveryCreationRule
OrderMustBeApprovedToStartDeliveringRule
OrderMustBeCancellableByClientRule
OrderMustBeCompletedBeforeRefundRule
OrderMustBeReviewingToApproveRule
OrderMustNotHaveAppliedPromoCodeToChangeItemsRule
OrderItemsMustHaveMarkingCodeBeforeOrderCompletingRule
OrderItemQuantityMustNotBeMoreThanItemWarehouseQuantityRule
PartnerMustBeFbsToChangeOrderItemsRule
PaymentMethodDoesntAllowOrderCancelBrokenRule
...
```

Тридцать таких файлов дают понимание домена быстрее, чем любая документация. **Первое, что стоит сделать при знакомстве с модулем — прочитать имена файлов в его папке `BusinessRules/`.**

Есть и `IBusinessRuleWithCode` — вариант с кодом ошибки, чтобы фронт мог локализовать сообщение (в `Startup.cs` код прогоняется через `ILocalizer<PurchaseResource>`).

> [!info] Почему `IsBrokenAsync` асинхронный, если многие правила синхронные
> В синхронных правилах видно `Task.FromResult(...)` — выглядит как шум. Но часть правил обращается к внешним проверкам: `AuthorizedBuyerPurchaseLimitRule` дёргает `ICheckOfferPurchaseLimitForBuyer`, а `ModeratedOfferQuantityMustNotBeMoreThenInWarehouseRule` — `IModeratedOfferQuantityInWarehouseChecker`. Интерфейсы объявлены в домене, реализованы в инфраструктуре — это [[Инверсия зависимостей на практике]] в чистом виде: домену нужна информация, и он объявляет, в какой форме её ждёт, не зная про БД.
> Сделать интерфейс синхронным нельзя (запрос в БД), делить на два интерфейса — усложнение. Общий асинхронный контракт — меньшее из зол.

---

## Domain Event

```csharp
public interface IDomainEvent : INotification
{
    Guid Id { get; }
    DateTime OccurredOn { get; }
}
```

`INotification` — из MediatR, то есть доменное событие сразу является сообщением для медиатора ([[MediatR и альтернативы]]).

Имена — всегда в **прошедшем времени**: `OrderApprovedEvent`, `OrderItemsReservedDomainEvent`, `AlifMobiAcquiringOrderCancelledDomainEvent`. Событие — факт, который уже произошёл и который нельзя отменить.

Есть маркер `ILoggableDomainEvent`: помеченные им события дополнительно пишутся в лог доменных событий (`DomainEventLogEntry`) — аудит важных изменений.

Разбор того, что происходит с событием после `AddDomainEvent`, — [[ServiceShop: Outbox, Inbox и обмен событиями]]. Теория — [[Доменные события]].

---

## Канонический метод агрегата

Это главный шаблон, который надо запомнить. `Orders.Domain/Order/AggregateRoot/Order.cs`:

```csharp
public async Task ApproveAsync()
{
    // 1. Проверить правила — до любых изменений
    await CheckRuleAsync(new EnsureAlifNasiyaTransactionExistRule(AlifNasiyaTransaction));

    // 2. Идемпотентность: повторный вызов — не ошибка, а no-op
    if (Status.Equals(OrderStatus.Approved))
        return;

    // 3. Изменить своё состояние
    StatusId = OrderStatus.Approved.Id;
    Status   = OrderStatus.Approved;
    StatusUpdatedAt = DateTime.Now;
    UpdatedAt = DateTime.Now;

    // 4. Изменить свои части (сущности внутри агрегата)
    AlifNasiyaTransaction!.Approve(ClientId, Origin, ClientPhone);

    // 5. Объявить факт
    AddDomainEvent(new OrderApprovedEvent(Id, AlifNasiyaTransaction.ApplicationId, ExternalRef!));
}
```

Чего в этом методе **нет** и никогда не должно быть: обращений к БД, HTTP-вызовов, отправки уведомлений, вызовов других модулей, логирования инфраструктурным логгером.

Порядок «проверки → изменения → событие» не косметический. Если сначала менять состояние, а потом проверять, объект может остаться в невалидном состоянии при выброшенном исключении — и, если транзакция не откатится, это уйдёт в базу.

Шаг 2 — про идемпотентность ([[Идемпотентность]]): Outbox гарантирует доставку **at-least-once**, значит команда «одобрить заказ» может прийти дважды. Второй раз должен быть безвредным.

> [!warning] `DateTime.Now` вместо `DateTime.UtcNow`
> В домене используется `DateTime.Now` — локальное время сервера. В инфраструктуре (`processed_date` в Outbox) — `DateTime.UtcNow`. Это несогласованность реального кода: смена таймзоны сервера или переезд в другой регион даст расхождение, а сравнение локальной и UTC-метки — тихую ошибку в несколько часов.
> Правильный подход — `DateTimeOffset` и инжектируемый `TimeProvider` (появился в .NET 8), который к тому же делает время тестируемым. Переписывать существующий код рискованно, но в новом лучше не повторять.

---

## Enumeration вместо enum

`OrderStatus` — не `enum`, а класс, наследник `Enumeration`:

```csharp
public class OrderStatus : Enumeration
{
    public static readonly OrderStatus Queued           = new(0, "queued");
    public static readonly OrderStatus New              = new(1, "new");
    public static readonly OrderStatus Reviewing        = new(2, "reviewing");
    public static readonly OrderStatus Approved         = new(3, "approved");
    public static readonly OrderStatus ApprovedWithBan  = new(104, "approved-with-ban");
    public static readonly OrderStatus Rejected         = new(4, "rejected");
    public static readonly OrderStatus Completed        = new(5, "completed");
    public static readonly OrderStatus Cancelled        = new(6, "cancelled");
    public static readonly OrderStatus Delivering       = new(7, "delivering");
    public static readonly OrderStatus Archived         = new(8, "archived");
    public static readonly OrderStatus Deleted          = new(99, "deleted");
    public static readonly OrderStatus Processing       = new(102, "processing");
    public static readonly OrderStatus ProcessingFailed = new(103, "processing-failed");
    public static readonly OrderStatus PendingPayment   = new(100, "pending-payment");
    public static readonly OrderStatus ExpiredPayment   = new(101, "expired-payment");
}
```

Преимущества перед `enum`: у значения есть и число (для БД), и строковый ключ (для API и логов), к нему можно прицепить поведение и нельзя случайно скастовать произвольный `int`.

Про сами перечисления — [[Перечисления (enum)]].

Числа не по порядку: 100+ — статусы, добавленные позже, когда 8–99 уже были заняты в проде. Это нормальная археология живой системы: id в базе поменять нельзя.

### Жизненный цикл заказа

```
                    ┌──────────┐
                    │  Queued  │ (0)  заказ принят, ждёт обработки
                    └────┬─────┘
                         ▼
  PendingPayment(100) ──▶ New(1) ──▶ Reviewing(2) ──┬──▶ Approved(3)
        │                                            ├──▶ ApprovedWithBan(104)
        ▼                                            └──▶ Rejected(4)
  ExpiredPayment(101)                                       │
                                                             ▼
                                          Delivering(7) ──▶ Completed(5) ──▶ Archived(8)

  Processing(102) ──▶ ProcessingFailed(103)     ← асинхронный флоу создания (v2)
  Cancelled(6), Deleted(99)                     ← из нескольких состояний
```

Переходы не произвольные: каждый защищён правилом (`OrderMustBeReviewingToApproveRule`, `OrderMustBeApprovedToStartDeliveringRule`). История переходов хранится в `_statusTransitions` — коллекции `OrderStatusTransition` внутри агрегата.

---

## Repository: интерфейс в домене, реализация в инфраструктуре

```
Domain/Aggregates/Cart/Interfaces/ICartRepository.cs      ← контракт (что нужно домену)
Infrastructure/Repositories/MongoCartRepository.cs         ← реализация (как это сделано)
```

Это [[Инверсия зависимостей на практике]]: домен объявляет потребность, инфраструктура её удовлетворяет, стрелка зависимости смотрит внутрь.

Интерфейсы репозиториев в проекте «толстые» — у `ICartRepository` 14 методов: `GetByBuyerAsync`, `GetAllByStoreId`, `Get14DaysOldCarts`, `BulkUpdateCartsAsync` и так далее. Это отход от канонического DDD, где репозиторий — коллекция агрегатов с минимумом методов, а сложные выборки уходят в спецификации ([[Specification pattern]]). Причина прозаична: методы добавлялись под конкретные задачи. Про сам вопрос «нужен ли репозиторий поверх ORM» — [[Repository и Unit of Work: нужны ли поверх EF Core]].

Спецификации в проекте тоже есть (`Domain/<Агрегат>/Specifications/`, Ardalis.Specification), но применяются реже, чем методы репозитория.

---

## Структура папок домена

В Orders (эталон):

```
Orders.Domain/Order/
├── AggregateRoot/         Order.cs, Order.Delivery.cs, Order.Fake.cs
├── BusinessRules/         ~30 правил
├── DomainEvents/          события, в т.ч. подпапка NewFlow/
├── Interfaces/            IOrderRepository и доменные сервисы
├── Specifications/        спецификации Ardalis
├── Strategies/            IOrderCreationStrategy + резолвер
├── Processing/            IOrderProcessor под каждый способ оплаты
├── ValueObjects/          OrderUuid, GnkReceipt и т.д.
├── OrderStatus.cs         Enumeration
├── OrderItem.cs           сущность внутри агрегата
└── OrderDelivery.cs
```

В Cart иначе: `Cart.Domain/Aggregates/<Агрегат>/` — лишний уровень `Aggregates`. Расхождение в конвенциях между модулями: единого стандарта нет, ориентироваться надо на модуль, в котором работаешь.

### Strategy под способы оплаты

Пять способов оплаты (Nasiya BNPL, BML, AlifMobi Native, AlifMobi Acquiring, наличные) породили бы гигантский `switch` в методе создания заказа. Вместо этого — [[Паттерны GoF: поведенческие]], паттерн «Стратегия»:

```csharp
containerBuilder.RegisterType<AlifNasiyaOrderCreationStrategy>().As<IOrderCreationStrategy>();
containerBuilder.RegisterType<OrderCreationStrategyResolver>().As<IOrderCreationStrategyResolver>();

containerBuilder.RegisterType<AlifNasiyaBnplOrderProcessor>().As<IOrderProcessor>();
containerBuilder.RegisterType<AlifNasiyaBmlOrderProcessor>().As<IOrderProcessor>();
```

Резолвер выбирает реализацию по способу оплаты. Новый способ оплаты = новый класс, а не новая ветка в общем методе — открыто для расширения, закрыто для изменения (принцип OCP из [[SOLID]]).

---

> [!example] Как делают в бою
> Когда в задаче написано «нельзя оформить заказ, если у партнёра выключена оплата в рассрочку», последовательность действий такая:
> 1. Найти похожее правило в `BusinessRules/` — почти наверняка что-то рядом уже есть (`PartnersSupportPaymentMethodsChecker`, `PaymentMethodDoesntAllowOrderCancelBrokenRule`).
> 2. Если данных для проверки в агрегате нет — объявить интерфейс-проверяльщик в `Domain/<Агрегат>/Interfaces/`, реализовать в `Infrastructure`, зарегистрировать в `<M>Startup.cs`.
> 3. Написать класс правила, добавить `await CheckRuleAsync(...)` в начало нужного метода агрегата.
> 4. Юнит-тест на правило: два кейса — нарушено и не нарушено.
>
> Чего **не** делать: добавлять проверку в командный хендлер. Технически работает, но правило окажется вне домена и не сработает, когда тот же агрегат меняется из другого сценария — а сценариев создания заказа четыре.

> [!warning] Подводные камни
> - **`IsBrokenAsync` возвращает «нарушено», а не «ок».** Инвертированная логика — источник ошибок в новых правилах, особенно если тест написан зеркально и подтверждает ошибку.
> - **`AddDomainEvent` не публикует событие.** Оно лишь кладётся в список внутри сущности. Публикация случится позже, в `UnitOfWork.CommitAsync` → `DomainEventsDispatcher`. Если сохранение агрегата не дойдёт до коммита — события не будет вовсе, и это правильно, но неочевидно при отладке.
> - **`DateTime.Now` в домене против `DateTime.UtcNow` в инфраструктуре.** Расхождение времён в одной системе; при сравнении меток даёт тихую ошибку.
> - **Публичный сеттер, просочившийся в агрегат.** В `Order` есть `public Guid? PurchaseId { get; set; }` и `public string ClientPhone { get; set; }` — с публичным `set`. Это дыра в инкапсуляции: состояние меняется в обход доменных методов, минуя правила. Не образец, а долг; в новом коде — только `private set`.
> - **`[BsonId]` и `[BsonElement]` прямо в агрегате `Cart`.** Инфраструктурные атрибуты в домене — отступление от правила «домен не знает про хранилище». Осознанный компромисс ради простоты маппинга Mongo, но повторять его в других модулях не стоит.

---

## Вопросы с собеседований

> [!question]- Почему у агрегатов приватные конструкторы и создание через статические фабричные методы?
> Три причины. Техническая и главная: конструктор не может быть асинхронным, а создание агрегата требует `await CheckRuleAsync(...)` — проверок, которые ходят в БД. Смысловая: имя метода документирует сценарий (`PlaceAsPendingPayment` против безымянного `new Order(...)`), а у одного агрегата сценариев создания может быть несколько с разными параметрами. Защитная: ни один путь создания не обходит проверку инвариантов. В ServiceShop правило форсируется тестом `OnlyPrivateConstructorsTest`, с исключением для `[BsonConstructor]`, который нужен драйверу MongoDB.

> [!question]- Зачем оборачивать бизнес-правило в отдельный класс вместо `if`?
> Класс даёт четыре вещи, которых нет у `if`: имя, читаемое как формулировка требования; переиспользование в нескольких методах агрегата одной строкой; тестируемость в изоляции без поднятия сценария; единый тип исключения, который в одном месте маппится в HTTP-ответ. Побочный, но самый ценный эффект — папка `BusinessRules/` становится живой спецификацией домена: 30 имён файлов дают понимание предметной области быстрее любого документа.

> [!question]- Чем Value Object отличается от Entity и когда что выбирать?
> Entity имеет идентичность: два заказа с одинаковыми полями и разными `Id` — разные заказы, и сущность отслеживается во времени. Value Object идентичности не имеет и сравнивается по значению всех полей: две суммы «100 сомони» — одно и то же, различать их незачем. Правило выбора: если для объекта осмыслен вопрос «тот же самый или другой?» — Entity; если только «равны ли» — Value Object. VO обычно иммутабельны, что снимает целый класс ошибок с общими ссылками.

> [!question]- Почему в агрегате ссылка на другой агрегат по ID, а не навигационным свойством?
> Навигационные свойства между агрегатами размывают границу транзакции и провоцируют неконтролируемую загрузку: один `Include` по цепочке тянет половину базы. Ссылка по ID делает границу явной — чтобы получить связанный агрегат, нужен отдельный осознанный запрос через его репозиторий. Внутри одного агрегата навигационные свойства как раз уместны: `Order` содержит коллекцию `OrderItem`, потому что они меняются и сохраняются вместе.

> [!question]- Что такое анемичная модель и чем она плоха?
> Анемичная модель — классы из одних публичных свойств, вся логика в сервисах. Плоха тем, что инварианты не защищены: любой код может присвоить свойство и перевести объект в невалидное состояние. Правила при этом расползаются по сервисам и дублируются, а тестировать их приходится через весь сценарий с базой. Оговорка: для CRUD-приложения без сложных правил анемичная модель — правильный выбор, DDD там будет чистой накладной стоимостью. ServiceShop оправдывает DDD объёмом: 15 статусов заказа, 5 флоу оплаты, ~30 правил.

> [!question]- Зачем `Enumeration` вместо обычного `enum`?
> `enum` — это `int` с именами, к нему нельзя прицепить поведение, у него нет строкового ключа для API, и в него можно скастовать произвольное число, получив несуществующее значение. `Enumeration` — класс со статическими экземплярами: есть `Id` для БД, `Name` для логов и API, можно добавить методы (например, «из какого статуса можно перейти в этот»), нельзя создать значение вне списка. Цена — это ссылочный тип, требующий маппинга для EF Core.

---

## Задачи

### Задача 1. Правило на минимальную сумму заказа

Нужно запретить оформление заказа в рассрочку, если сумма меньше 500 000. Написать правило в стиле проекта.

> [!success]- Решение
> ```csharp
> namespace Alif.ServiceShop.Modules.Orders.Domain.Order.BusinessRules;
>
> public class OrderTotalAmountMustBeAtLeastMinimumForBnplRule : IBusinessRule
> {
>     private const ulong MinimumAmount = 500_000;
>
>     private readonly ulong _totalAmount;
>
>     public OrderTotalAmountMustBeAtLeastMinimumForBnplRule(ulong totalAmount)
>     {
>         _totalAmount = totalAmount;
>     }
>
>     public Task<bool> IsBrokenAsync() => Task.FromResult(_totalAmount < MinimumAmount);
>
>     public string Message =>
>         $"Order total amount {_totalAmount} is less than minimum {MinimumAmount} for BNPL payment";
> }
> ```
> Ключевое: `IsBrokenAsync` возвращает `true`, когда сумма **меньше** минимума — то есть когда правило нарушено, не когда выполнено. Имя класса читается как требование. Подключение — `await CheckRuleAsync(new OrderTotalAmountMustBeAtLeastMinimumForBnplRule(TotalAmount));` в начале `CreateBnplNasiyaOrder`.

### Задача 2. Найти ошибку в методе агрегата

```csharp
public async Task CancelAsync(int? cancelReason)
{
    StatusId = OrderStatus.Cancelled.Id;
    Status = OrderStatus.Cancelled;

    await CheckRuleAsync(new OrderMustBeCancellableByClientRule(Status));

    AddDomainEvent(new OrderCancelledDomainEvent(Id, cancelReason));
}
```

Найти две ошибки.

> [!success]- Решение
> **Первая:** правило проверяется **после** изменения состояния. Порядок обязан быть обратным — иначе при нарушении правила объект уже переведён в `Cancelled` и, если вызывающий код проглотит исключение или транзакция не откатится, невалидное состояние утечёт в базу.
>
> **Вторая, тоньше:** правилу передан уже изменённый `Status`, то есть проверка всегда получит `Cancelled` и никогда не сработает по назначению. Правило проверяет не то, что задумано, — молча всегда проходит.
>
> Плюс отсутствует проверка идемпотентности: повторный вызов при уже отменённом заказе добавит второе доменное событие, и потребители получат дубль.
>
> ```csharp
> public async Task CancelAsync(int? cancelReason)
> {
>     await CheckRuleAsync(new OrderMustBeCancellableByClientRule(Status));
>
>     if (Status.Equals(OrderStatus.Cancelled))
>         return;
>
>     StatusId = OrderStatus.Cancelled.Id;
>     Status = OrderStatus.Cancelled;
>
>     AddDomainEvent(new OrderCancelledDomainEvent(Id, cancelReason));
> }
> ```

### Задача 3. Value Object для суммы

Завести Value Object `Money` для суммы в сомони так, чтобы отрицательная сумма была непредставима, и суммы можно было складывать.

> [!success]- Решение
> ```csharp
> public class Money : ValueObject
> {
>     public ulong Amount { get; }
>
>     private Money(ulong amount) => Amount = amount;
>
>     public static Money From(ulong amount) => new(amount);
>
>     public Money Add(Money other) => new(Amount + other.Amount);
>
>     public override string ToString() => $"{Amount}";
> }
> ```
> Отрицательное значение непредставимо по построению — тип `ulong` не имеет знака, проверка в конструкторе не нужна вовсе. Это сильнее, чем `int` с проверкой: там ошибку ловит рантайм, здесь — компилятор.
>
> `Add` возвращает **новый** объект, а не меняет текущий: Value Object иммутабелен, иначе две ссылки на одну сумму начнут влиять друг на друга. Сравнение по значению даётся базовым классом бесплатно: `Money.From(100) == Money.From(100)` вернёт `true`.
>
> В реальном проекте суммы хранятся как `ulong` прямо в агрегате (`SubTotalAmount`, `TotalAmount`, `Discount`) — VO для денег не завели. Это упущение: единицы измерения нигде не зафиксированы, и «в тийинах или в сомони» приходится выяснять по контексту.

---

## Итог

- `Entity` даёт три вещи: список доменных событий, `AddDomainEvent` и `CheckRuleAsync`.
- Агрегат создаётся только фабричным методом — прежде всего потому, что конструктор не может быть `async`, а проверки правил асинхронные.
- Ссылки между агрегатами — только по ID; навигационные свойства только внутри агрегата.
- Одно правило = один класс с говорящим именем; папка `BusinessRules/` читается как спецификация домена.
- Канонический метод агрегата: проверить правила → проверить идемпотентность → изменить состояние → `AddDomainEvent`.
- `AddDomainEvent` ничего не публикует — только копит; публикация происходит в `UnitOfWork.CommitAsync`.
- Известные отступления от канона в этом коде: публичные сеттеры в `Order`, `[Bson*]`-атрибуты в `Cart`, `DateTime.Now` вместо UTC, толстые интерфейсы репозиториев. Знать как долг, не тиражировать.

## Связанное

- [[93 — Рабочий проект: Alif ServiceShop (обзор раздела)]]
- [[ServiceShop: CQRS и путь запроса]]
- [[ServiceShop: Outbox, Inbox и обмен событиями]]
- [[DDD: тактические паттерны]]
- [[Доменные события]]
- [[Инверсия зависимостей на практике]]
- [[Иммутабельность как приём проектирования]]
- [[Specification pattern]]
- [[SOLID]]
