---
tags: [раздел-93, рабочий-проект, архитектура, модульный-монолит, autofac, middle, собес]
aliases: [Modular Monolith в ServiceShop, Модульный монолит ServiceShop, Composition Root ServiceShop, Изоляция модулей, Фасад модуля]
---

# ServiceShop: модульный монолит и изоляция модулей

> [!abstract] Коротко
> ServiceShop — один процесс ASP.NET Core, внутри которого пять модулей, изолированных так же жёстко, как если бы это были отдельные сервисы. Механизм изоляции нестандартный и это главное, что надо понять про проект: **у каждого модуля свой собственный Autofac-контейнер**, своя строка подключения к БД, свой планировщик Quartz и своя точка входа.
> Наружу модуль выставляет ровно один интерфейс с тремя методами (`ExecuteCommandAsync`, `ExecuteQueryAsync`). Всё остальное — репозитории, `DbContext`, сервисы — физически недостижимо снаружи, потому что живёт в чужом контейнере.

## Зачем это нужно

Обычный монолит разваливается не потому, что он монолит, а потому что в нём **нет границ**: любой класс может внедрить любой другой, и через год граф зависимостей — полный. Микросервисы решают это радикально (границу обеспечивает сеть), но платят распределёнными транзакциями, сетевыми отказами и деплоем десяти сервисов.

Модульный монолит (modular monolith) — компромисс: границы такие же строгие, как между сервисами, но выполнение в одном процессе. Разбор выбора вообще — [[Монолит vs микросервисы: как решать]].

Ключевой вопрос: **чем обеспечена граница?** Соглашение «мы договорились не лазить в чужой модуль» не работает — его нарушат в первый же дедлайн. В ServiceShop граница обеспечена тремя механизмами, каждый из которых не даёт нарушить правило технически:

1. **Ссылки между проектами.** Если `Cart.Application` не ссылается на `Orders.Infrastructure`, классы оттуда просто не видны компилятору.
2. **Отдельный DI-контейнер на модуль.** Даже если тип виден, его нельзя получить: он не зарегистрирован в контейнере твоего модуля.
3. **ArchTests.** Проверяют то, что первые два механизма не покрывают (например, что контроллер не зависит от другого контроллера).

---

## Отдельный контейнер на каждый модуль

Это самое неожиданное решение проекта. Про жизненные циклы и обычный подход — [[Жизненные циклы сервисов: Singleton, Scoped, Transient]] и [[Dependency Injection: контейнер ASP.NET Core]]; здесь всё иначе.

`Orders.Infrastructure/Configuration/OrdersStartup.cs`:

```csharp
public class OrdersStartup
{
    private static IContainer _container = null!;

    public static void Initialize(
        IConfiguration configuration,
        string connectionString,
        string redisConnectionString,
        IExecutionContextAccessor executionContextAccessor,
        ILogger logger,
        Dictionary<Type, IModuleFacade> moduleFacades,
        IFeatureToggleService featureToggleService,
        IEventsBus? eventsBus,
        long? internalProcessingPoolingInterval = null)
    {
        var moduleLogger = logger.ForContext("Module", "Orders");

        ConfigureCompositionRoot(...);       // собрать СВОЙ контейнер
        FluentMigrationStartup.Initialize(moduleLogger);   // накатить СВОИ миграции
        QuartzStartup.Initialize(...);       // поднять СВОИ фоновые джобы
        EventsBusStartup.Initialize(...);    // подписаться на СВОИ события
        KafkaBusStartup.Initialize(...);     // поднять СВОИ консьюмеры Kafka
    }
}
```

Внутри `ConfigureCompositionRoot` создаётся отдельный `ContainerBuilder`, в него регистрируются Autofac-модули (`DataAccessModule`, `CacheModule`, `ProcessingModule`, `MediatorModule`, `OutboxModule`, `QuartzModule`, `KafkaBusModule`, `HttpModule`, `StorageModule`), и готовый контейнер кладётся в статическое поле:

```csharp
_container = containerBuilder.Build();
OrdersCompositionRoot.SetContainer(_container);
```

**Composition Root** — единственное место в приложении, которое знает обо всех классах сразу и связывает интерфейсы с реализациями. Здесь их не один, а пять — по одному на модуль:

```csharp
internal static class OrdersCompositionRoot
{
    private static IContainer _container = null!;

    internal static void SetContainer(IContainer container) => _container = container;

    internal static ILifetimeScope BeginLifetimeScope() => _container.BeginLifetimeScope();
}
```

Обрати внимание на `internal` — этот класс не виден даже из других проектов того же модуля, только внутри `Orders.Infrastructure`.

### Что это даёт и чего стоит

| Плюс | Минус |
|---|---|
| Нельзя случайно внедрить сервис чужого модуля — его нет в контейнере | Пять раз почти одинаковый код регистрации: `ProcessingModule`, `MediatorModule`, декораторы дублируются в каждом модуле |
| Модуль можно вынести в отдельный сервис почти механически: он уже сам себя собирает | Статический контейнер в статическом поле — глобальное состояние, неудобно в интеграционных тестах |
| Разный `lifetime` и разные настройки на модуль без конфликтов | Нет единой точки, где видно все регистрации системы |
| Логгер сразу обогащён контекстом модуля (`logger.ForContext("Module", "Orders")`) | Модуль обязан быть инициализирован до первого обращения, иначе `NullReferenceException` на `_container` |

Дублирование здесь — **осознанная плата за изоляцию**. Попытка вынести общий `ProcessingModule` в BuildingBlocks сразу создала бы связь «все модули зависят от общего кода регистрации», и любое изменение под нужды одного модуля ломало бы остальные. Это тот случай, когда DRY проигрывает — см. [[SOLID]].

---

## Запуск: кто всё это инициализирует

`Alif.ServiceShop.API/Startup.cs`, метод `InitializeModules`. Каждому модулю передаётся **своя** строка подключения:

```csharp
private const string CatalogConnectionString    = "CatalogConnectionString";
private const string ModerationConnectionString = "ModerationConnectionString";
private const string OrdersConnectionString     = "OrdersConnectionString";
private const string CartConnectionString       = "CartConnectionString";
private const string MarketingConnectionString  = "MarketingConnectionString";
private const string CartMongoDbConnectionString = "CartMongoDbConnectionString";
// ...

CatalogStartup.Initialize(...);
CartStartup.Initialize(...);
ModerationStartup.Initialize(...);
OrdersStartup.Initialize(...);
MarketingStartup.Initialize(...);
```

Разные строки подключения — не формальность. Это означает: **модуль не может прочитать таблицы чужого модуля даже SQL-запросом**, потому что у него нет соединения к этой схеме. Граница держится на уровне БД, а не только на уровне кода.

```
┌─────────────────────────── один процесс, один Kestrel ──────────────────────────┐
│                                                                                 │
│  Alif.ServiceShop.API                                                           │
│  ├── контроллеры Catalog ─┐  ├── контроллеры Orders ─┐  ├── контроллеры Cart ─┐  │
│                            │                          │                        │  │
│  ┌─────────────────────────▼──┐  ┌────────────────────▼─┐  ┌──────────────────▼┐ │
│  │  Catalog                   │  │  Orders              │  │  Cart             │ │
│  │  ─ свой Autofac-контейнер  │  │  ─ свой контейнер    │  │  ─ свой контейнер │ │
│  │  ─ свой Quartz             │  │  ─ свой Quartz       │  │  ─ свой Quartz    │ │
│  │  ─ CatalogConnectionString │  │  OrdersConnStr       │  │  CartConnStr      │ │
│  └───────────┬────────────────┘  └──────────┬───────────┘  └────────┬──────────┘ │
│              │                              │                       │            │
│         MySQL schema                   MySQL schema        MySQL + MongoDB       │
│              │                              │                       │            │
│              └────────── обмен только событиями через Outbox/Inbox ──┘            │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Фасад модуля: единственная публичная дверь

`Orders.Application/Contracts/IOrdersModule.cs`:

```csharp
public interface IOrdersModule
{
    Task<TResult> ExecuteCommandAsync<TResult>(ICommand<TResult> command);
    Task ExecuteCommandAsync(ICommand command);
    Task<TResult> ExecuteQueryAsync<TResult>(IQuery<TResult> query);
}
```

Три метода. Всё. Реализация — `Orders.Infrastructure/OrdersModule.cs`:

```csharp
public class OrdersModule : IOrdersModule
{
    public async Task<TResult> ExecuteCommandAsync<TResult>(ICommand<TResult> command)
        => await CommandsExecutor.Execute(command);

    public async Task<TResult> ExecuteQueryAsync<TResult>(IQuery<TResult> query)
    {
        using var scope = OrdersCompositionRoot.BeginLifetimeScope();
        var mediator = scope.Resolve<IMediator>();
        return await mediator.Send(query);
    }
}
```

И `CommandsExecutor` — та же схема:

```csharp
internal static class CommandsExecutor
{
    internal static async Task<TResult> Execute<TResult>(ICommand<TResult> command)
    {
        using var scope = OrdersCompositionRoot.BeginLifetimeScope();
        var mediator = scope.Resolve<IMediator>();
        return await mediator.Send(command);
    }
}
```

Здесь важна деталь про **область жизни (lifetime scope)**: на каждую команду открывается новый scope и закрывается по завершении. Это аналог scoped-зависимостей в ASP.NET Core, но управляемый вручную — потому что команда может прийти не из HTTP-запроса, а из фоновой джобы Quartz, где никакого request scope нет.

Из этого следует практическое правило: **`InstancePerLifetimeScope` в регистрациях модуля = «один экземпляр на команду»**, а не «на HTTP-запрос». Один HTTP-запрос, вызвавший две команды, получит два разных `DbContext`. И наоборот: одна команда, вызванная из джобы, всё равно получит корректный scope.

### Второй тип фасада: когда нужны данные синхронно

Помимо `IOrdersModule` (выполнить свою команду) есть `IOrdersModuleFacade` в проекте `Orders.IntegrationEvents` — для случая, когда другому модулю нужно **спросить** данные прямо сейчас, а событие не подходит. Пример: `PartnerPaymentMethodDto` — Cart спрашивает у Orders, какие способы оплаты поддерживает партнёр.

Почему фасад лежит в `IntegrationEvents`, а не в `Application`: этот проект и так публичный контракт модуля, на него ссылаются все. `Application` остаётся внутренним.

Фасады раздаются модулям при старте через словарь:

```csharp
var moduleFacades = new Dictionary<Type, IModuleFacade> { ... };
// в контейнере модуля:
foreach (var facadeKeyPair in moduleFacades)
    containerBuilder.RegisterInstance(facadeKeyPair.Value).As(facadeKeyPair.Key);
```

> [!warning] Синхронный фасад — костыль, а не основной путь
> Каждый вызов фасада создаёт связь «модуль A не работает без модуля B прямо сейчас». Это ровно то, от чего уходит event-driven-архитектура. Правильный дефолт — событие плюс локальная копия нужных данных; фасад — когда данные слишком объёмны для дублирования или нужны строго свежие. Прежде чем добавлять метод в фасад, стоит проверить, не решается ли задача подпиской на событие.

---

## Контроллер: где живёт HTTP

Контроллеры лежат не внутри модулей, а в `Alif.ServiceShop.API/Modules/<Модуль>/`. Это единственное место, где модули «встречаются» в одном проекте — и поэтому именно на контроллеры навешано больше всего ArchTests.

`API/Modules/Cart/WebAndMobile/Cart/CartController.cs`:

```csharp
[Route("api/cart")]
public class CartController(ICartModule cartModule) : BaseController
{
    [HttpPost("moderated-items")]
    public async Task<IActionResult> AddModeratedItemToCartAsync([FromBody] AddModeratedItemToCartRequest request)
    {
        var command = new AddModeratedItemToCartCommand(
            request.ModeratedOfferId, request.Quantity, request.ConditionId);

        var result = await cartModule.ExecuteCommandAsync(command);

        return Ok(result);
    }
}
```

Три строки: собрать команду, отдать модулю, вернуть результат. **Никакой логики.** Это не «упрощённый пример» — так выглядят все контроллеры проекта, и это проверяется автотестами.

Структура папок внутри модуля API — по типу клиента: `Web`, `Mobile`, `MiniApp`, `AlifshopAdminPanel`, `MerchantAdminPanel`, `Support`, `Integration`, `Webhooks`. У одного действия могут быть разные контроллеры для разных клиентов, потому что различаются права, формат ответа и версии — но команда под ними одна и та же.

Обрати внимание на `primary constructor` (`CartController(ICartModule cartModule)`) — синтаксис C# 12, в проекте используется активно.

---

## Как модули общаются

Прямых вызовов нет. Три легальных канала:

| Канал | Когда | Механизм |
|---|---|---|
| **Integration Event** | дефолт: «у меня произошло, реагируйте» | Outbox → EventBus → Inbox получателя |
| **Фасад модуля** | нужны данные синхронно, дублировать нельзя | прямой вызов интерфейса из `IntegrationEvents` |
| **Kafka** | обмен с внешними сервисами Alif (платежи, доставка, кредиты) | KafkaFlow, отдельные стримы на источник |

Детально первый канал — [[ServiceShop: Outbox, Inbox и обмен событиями]].

---

> [!example] Как делают в бою
> Признак, что изоляция начала протекать: в `<M>Startup.cs` растёт число регистраций фасадов чужих модулей, а в хендлерах появляются вызовы вида `_ordersFacade.GetSomething(...)` внутри циклов. Это значит, что модуль не имеет нужных данных локально и добирает их синхронно — потенциальный N+1 через границу модуля и жёсткая связь по доступности.
> Лечение: определить, какие данные модулю нужны постоянно, и завести на них локальную проекцию, обновляемую событиями. В ServiceShop так и сделано с ценами и остатками: Cart не спрашивает Catalog о цене товара — он хранит свою копию и обновляет её по событиям `ModeratedOfferPriceChanged`.

> [!warning] Подводные камни
> - **Порядок инициализации значим.** `EventsBusStartup` каждого модуля обращается к `CompositionRoot.BeginLifetimeScope()`, то есть требует уже собранного контейнера. Переставить вызовы в `Startup.InitializeModules` местами так, чтобы модуль подписался до сборки своего контейнера, — гарантированный `NullReferenceException` на старте.
> - **Статический контейнер и тесты.** `OrdersCompositionRoot._container` — статика. Два интеграционных теста, инициализирующих модуль по-разному в одном процессе, будут драться за это поле. Именно поэтому интеграционные тесты в проекте есть только у Marketing, а остальное покрыто юнит-тестами.
> - **`InstancePerLifetimeScope` ≠ per-request.** Легко ошибиться, если переносить привычки из ASP.NET Core DI: здесь scope открывается на команду. Кэш, положенный в scoped-сервис в расчёте «на весь запрос», между двумя командами одного запроса не переживёт.
> - **Фасад молча ломает изоляцию.** Компилятор не мешает добавить в фасад метод, возвращающий агрегат чужого модуля. Технически соберётся, архитектурно — прямая зависимость. Через фасад передаются только DTO, для этого в `IntegrationEvents/Facades/Dtos/` отдельная папка.

---

## Вопросы с собеседований

> [!question]- Что такое модульный монолит и чем он отличается от «просто монолита с папками»?
> Модульный монолит — это монолит, в котором границы модулей обеспечены **технически**, а не соглашением. Проверочный вопрос: может ли класс из модуля A внедрить репозиторий модуля B? В монолите с папками — да, ничто не мешает. В модульном монолите — нет: либо нет ссылки между проектами, либо тип не зарегистрирован в контейнере, либо архитектурный тест упадёт. Второе отличие — у модуля своя схема БД: он не читает чужие таблицы напрямую, поэтому изменение схемы одного модуля не может сломать другой.

> [!question]- Зачем отдельный DI-контейнер на модуль, если можно использовать один общий?
> Один общий контейнер означает, что любой зарегистрированный сервис доступен всем — граница держится только на честном слове. Отдельный контейнер делает нарушение невозможным: `Cart` не может получить `IOrderRepository`, потому что тот зарегистрирован в другом контейнере, до которого нет доступа (`internal static` composition root). Плюс модуль становится самодостаточным: чтобы вынести его в отдельный сервис, не нужно распутывать общие регистрации. Цена — дублирование кода регистрации в каждом модуле и статическое состояние, неудобное в тестах.

> [!question]- Что произойдёт, если контроллер напрямую внедрит репозиторий модуля?
> В ServiceShop это не скомпилируется: проект `API` не ссылается на `<M>.Domain`/`<M>.Infrastructure`, ему доступны только `Application.Contracts` и `IntegrationEvents`. Даже если ссылку добавить, зарегистрированного репозитория в контейнере ASP.NET Core нет — он в контейнере модуля. А если и это обойти, упадёт ArchTest `ShouldInjectOnlyInterfacesTest`. Три уровня защиты — иллюстрация принципа «архитектурное правило должно быть невыполнимо нарушить, а не просто описано в вики».

> [!question]- Почему `InstancePerLifetimeScope` в этом проекте не значит «на HTTP-запрос»?
> Потому что scope открывается не middleware ASP.NET Core, а вручную в `CommandsExecutor.Execute`/`ExecuteQueryAsync` — на каждую команду или запрос. Один HTTP-запрос, выполняющий две команды, получит два независимых scope и два `DbContext`. Так сделано потому, что команды приходят не только из HTTP: `ProcessOutboxJob`, `ProcessInboxJob` и `ProcessInternalCommandsJob` в Quartz выполняют те же команды вне всякого запроса, и им нужен тот же механизм жизненного цикла.

> [!question]- Как модулю получить данные другого модуля, если прямой вызов запрещён?
> Три способа по убыванию предпочтительности. Первый и основной — подписаться на интеграционное событие и держать у себя локальную проекцию нужных данных (так Cart хранит цены товаров). Второй — фасад модуля (`IOrdersModuleFacade`), синхронный вызов, возвращающий DTO; применяется, когда дублировать данные нецелесообразно. Третий — общая шина Kafka, если данные приходят из внешнего сервиса. Прямой доступ к чужим таблицам или репозиториям невозможен ни в одном варианте.

---

## Итог

- Изоляция модулей держится на трёх механизмах: ссылки проектов, отдельный DI-контейнер, ArchTests.
- У каждого модуля свой Autofac-контейнер (`<M>CompositionRoot`), своя строка подключения, свой Quartz и своя точка входа `<M>Startup.Initialize`.
- Наружу торчит фасад из трёх методов: `ExecuteCommandAsync` (×2) и `ExecuteQueryAsync`; всё остальное недостижимо.
- Scope открывается на команду, а не на HTTP-запрос — потому что команды приходят и из фоновых джоб.
- Контроллеры тонкие: собрать команду, отдать модулю, вернуть результат.
- Цена подхода: дублирование регистраций между модулями и статический контейнер, неудобный в интеграционных тестах. Это осознанный обмен на невозможность нарушить границу.

## Связанное

- [[93 — Рабочий проект: Alif ServiceShop (обзор раздела)]]
- [[ServiceShop: CQRS и путь запроса]]
- [[ServiceShop: Outbox, Inbox и обмен событиями]]
- [[Монолит vs микросервисы: как решать]]
- [[Clean Architecture]]
- [[Dependency Injection: контейнер ASP.NET Core]]
- [[Жизненные циклы сервисов: Singleton, Scoped, Transient]]
- [[DDD: стратегические паттерны]]
