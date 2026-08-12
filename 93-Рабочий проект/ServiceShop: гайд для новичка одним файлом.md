---
tags: [раздел-93, рабочий-проект, onboarding, шпаргалка, основы]
aliases: [Гайд по ServiceShop, ServiceShop онбординг, Онбординг Alif ServiceShop, Alif ServiceShop одним файлом]
---

# ServiceShop: гайд для новичка одним файлом

> [!abstract] Коротко
> Исходный сквозной конспект по проекту `alifshop.backend.service-shop`, составленный при первом знакомстве с кодовой базой (август 2026). Один файл, читается за раз, от терминов .NET до чек-листа «как добавить фичу».
> Это **компактная версия** раздела: те же темы разобраны глубже, с вопросами собеседований и задачами, в пяти отдельных заметках — начиная с [[93 — Рабочий проект: Alif ServiceShop (обзор раздела)]]. Здесь дублирование оставлено сознательно: удобно перечитать целиком перед задачей или скинуть новому человеку в команде.

> [!info] Пути к файлам
> Пути вида `Alif.ServiceShop.Modules.Orders.Domain/...` — относительно корня репозитория `alifshop.backend.service-shop`, не этой базы. Кликабельными они здесь не будут.

---

## Часть 0. Что вообще нужно понять в первый день

Это **бэкенд маркетплейса Alif Shop**: витрина товаров, корзина, оформление заказа, оплата (в том числе в рассрочку через Nasiya/BML), модерация товаров продавцов, пуш-уведомления.

Технически это **один процесс** (одно приложение ASP.NET Core), внутри которого живут **5 почти независимых модулей**. Такая штука называется **модульный монолит** (modular monolith): не микросервисы (нет сети между модулями, деплой один), но и не «большой ком грязи» — модули изолированы жёстко, как если бы это были отдельные сервисы.

Три вещи, которые объясняют 80% странностей в коде:

1. **DDD** (Domain-Driven Design) — бизнес-логика живёт в объектах предметной области (`Order`, `Cart`), а не в сервисах и не в контроллерах.
2. **CQRS** — запись (Command) и чтение (Query) идут разными путями и даже разными библиотеками доступа к БД.
3. **Event-driven** — модули не вызывают друг друга напрямую, а обмениваются событиями через таблицы `outbox_messages` / `inbox_messages` и Kafka.

---

## Часть 1. Словарь: .NET и C# для тех, кто с другого стека

| Термин | Что это на самом деле |
|---|---|
| **.NET 8** | Платформа исполнения (как JVM для Java, как Node для JS). Задана в `Directory.Build.props`: `<TargetFramework>net8.0</TargetFramework>` |
| **Solution (`.sln`)** | Файл-оглавление: список всех проектов. Здесь — `ServiceShop.sln` |
| **Project (`.csproj`)** | Единица компиляции → одна .dll. Каждая папка `Alif.ServiceShop.*` = один проект. Именно на уровне проектов задаются **зависимости**: если проект A не ссылается на B, код B физически недоступен в A — так архитектура защищается компилятором |
| **NuGet** | Пакетный менеджер (аналог npm/composer). Версии пакетов централизованы в `Directory.Packages.props` — в `.csproj` версии не пишут |
| **Namespace** | Логическое пространство имён. Обычно совпадает с путём папок |
| **`interface I...`** | Контракт. В C# по соглашению имя начинается с `I`: `IOrdersModule`, `ICartRepository` |
| **`async/await`, `Task`** | Асинхронность. `Task<T>` — «обещание» значения (как Promise). `await` — дождаться, не блокируя поток |
| **DI (Dependency Injection)** | Зависимости не создаются через `new`, а приходят в конструктор. Кто их подставляет — «контейнер» |
| **`Nullable enable`** | Включена строгая проверка null: `string` — не может быть null, `string?` — может. Компилятор ругается |
| **`TreatWarningsAsErrors`** | Любое предупреждение компилятора = ошибка сборки. Плюс StyleCop-анализаторы. Код не соберётся из-за неправильного форматирования — это нормально, не баг |
| **`partial class`** | Один класс, разбитый на несколько файлов. Пример: `Order` лежит в `Alif.ServiceShop.Modules.Orders.Domain/Order/AggregateRoot/Order.cs` + `Order.Delivery.cs` + `Order.Fake.cs` |

### Библиотеки, которые встретятся сразу

| Библиотека | Роль | Где увидеть |
|---|---|---|
| **Autofac** | DI-контейнер. **В проекте используется только он** — не встроенный `Microsoft.Extensions.DependencyInjection` | `Alif.ServiceShop.Modules.Orders.Infrastructure/Configuration/OrdersStartup.cs` |
| **MediatR** | Реализация паттерна Mediator: ты отправляешь объект-сообщение (`Send`/`Publish`), библиотека сама находит его обработчик | `ICommand : IRequest` |
| **EF Core 8 + Pomelo** | ORM для MySQL — для записи (агрегаты) | `OrdersContext` |
| **Dapper** | Микро-ORM: сырой SQL → объекты. Для чтения (Query) и для служебных таблиц | `Alif.ServiceShop.Modules.Orders.Infrastructure/Configuration/Processing/Outbox/ProcessOutboxCommandHandler.cs` |
| **FluentMigrator** | Миграции схемы БД. **Не EF Migrations!** | `Alif.ServiceShop.Modules.Orders.Infrastructure/Migrations` |
| **FluentValidation** | Валидация входных команд | `ValidationCommandHandlerDecorator` |
| **Quartz** | Планировщик фоновых задач (cron внутри приложения) | `QuartzStartup` |
| **KafkaFlow** | Обёртка над Kafka-клиентом | `KafkaBusModule` |
| **Serilog** | Логирование. **Только он**, `Microsoft.Extensions.Logging` в проекте запрещён | `using ILogger = Serilog.ILogger;` |
| **NetArchTest** | Тесты, проверяющие саму архитектуру | `ArchTests` |
| **xUnit + FluentAssertions + NSubstitute/Moq** | Юнит-тесты, читаемые проверки, моки | `*.UnitTests` |

---

## Часть 2. Как читать названия проектов

Имя проекта = адрес. Формат: `Alif.ServiceShop.<Область>.<Имя>.<Слой>`

```
Alif.ServiceShop.Modules.Orders.Domain              ← бизнес-правила модуля Orders
Alif.ServiceShop.Modules.Orders.Application         ← сценарии (команды/запросы)
Alif.ServiceShop.Modules.Orders.Infrastructure      ← БД, DI, Kafka, Quartz
Alif.ServiceShop.Modules.Orders.IntegrationEvents   ← публичные события модуля
Alif.ServiceShop.Modules.Orders.Domain.UnitTests    ← тесты
```

Плюс общие проекты:

```
Alif.ServiceShop.API                       ← единственный запускаемый проект (HTTP)
Alif.ServiceShop.BuildingBlocks.Domain     ← базовые кирпичи DDD, общие для всех модулей
Alif.ServiceShop.BuildingBlocks.Application
Alif.ServiceShop.BuildingBlocks.Infrastructure
Alif.ServiceShop.BuildingBlocks.EventBus   ← in-memory шина событий
ArchTests                                  ← тесты архитектуры
```

**Правило зависимостей внутри модуля** (священное для DDD):

```
Infrastructure ──▶ Application ──▶ Domain
                                    ▲
                                    │  Domain не знает НИКОГО.
                                    │  Ни про БД, ни про HTTP, ни про Kafka.
```

Почему так: доменные правила («заказ нельзя отменить после отправки») переживают смену БД, фреймворка и транспорта. Если Domain зависит от EF Core, заменить хранилище невозможно, а протестировать правило без базы — тем более.

---

## Часть 3. DDD: кирпичики, из которых собран Domain

Все базовые типы лежат в `Alif.ServiceShop.BuildingBlocks.Domain`. Читать их стоит прямо сейчас — они маленькие.

### 3.1 Entity (сущность)

`Alif.ServiceShop.BuildingBlocks.Domain/Entity.cs`

**Что:** объект с идентичностью. Два заказа с одинаковыми полями, но разными `Id` — разные заказы.

**Что даёт базовый класс:**
- список `DomainEvents` — «что со мной произошло» (см. 3.4);
- `AddDomainEvent(...)` — защищённый метод для наследников;
- `CheckRuleAsync(rule)` — проверить бизнес-правило и бросить исключение, если нарушено.

```csharp
protected static async Task CheckRuleAsync(IBusinessRule rule)
{
    if (await rule.IsBrokenAsync())
        throw new BusinessRuleValidationException(rule);
}
```

**Альтернатива, которую здесь не выбрали:** анемичная модель (класс = набор полей, вся логика в `OrderService`). Проще на старте, но при 30+ бизнес-правилах логика расползается по сервисам и дублируется.

### 3.2 Value Object (объект-значение)

`Alif.ServiceShop.BuildingBlocks.Domain/ValueObject.cs`, `Alif.ServiceShop.BuildingBlocks.Domain/GuidValueObject.cs`

**Что:** объект без идентичности, сравнивается **по значению всех полей** (реализовано через рефлексию: `Equals` сравнивает все свойства и поля).

**Зачем:** вместо `Guid orderId` завести тип `OrderUuid`. Тогда компилятор не даст перепутать `OrderUuid` и `ClientId` — а обе были бы просто `Guid`. `GuidValueObject` дополнительно запрещает `Guid.Empty` прямо в конструкторе: невалидное значение невозможно создать в принципе.

Примеры в коде: `Alif.ServiceShop.Modules.Orders.Domain/Order/ValueObjects/`.

### 3.3 Aggregate Root (корень агрегата)

`Alif.ServiceShop.BuildingBlocks.Domain/IAggregateRoot.cs` — пустой маркерный интерфейс.

**Что:** группа объектов, которые изменяются только вместе и только через «главный» объект. `Order` — корень, `OrderItem` внутри него — часть агрегата. Снаружи нельзя взять `OrderItem` и поменять ему количество: только `order.AddOrderItem(...)`.

**Зачем:** агрегат — граница транзакции и граница инвариантов. Внутри одной операции сохраняется ровно один агрегат, и он сам отвечает за свою консистентность.

**Два жёстких правила проекта:**

1. **Конструкторы только `private`**, создание — через статические фабричные методы `Create(...)` / `PlaceAsPendingPayment(...)`. Причина: конструктор не может быть `async`, а создание заказа требует `await CheckRuleAsync(...)`. Фабричный метод может. Плюс имя метода объясняет, *какой именно* сценарий создания.
2. **Агрегат не ссылается на другой агрегат объектом — только по ID.** В `Order` нет поля `Client Client`, есть `int ClientId`. Иначе EF Core потянет полбазы одним запросом, а границы транзакций расплывутся.

Оба правила проверяются автотестами: `ArchTests/Modules/DomainLayer/Aggregates/OnlyPrivateConstructorsTest.cs`, `ArchTests/Modules/DomainLayer/Aggregates/CreatedThroughFactoryMethodOnlyTest.cs`.

Смотреть живьём: `Alif.ServiceShop.Modules.Cart.Domain/Aggregates/Cart/Cart.cs` (простой) → `Alif.ServiceShop.Modules.Orders.Domain/Order/AggregateRoot/Order.cs` (сложный, ~900 строк).

### 3.4 Business Rule (бизнес-правило)

`Alif.ServiceShop.BuildingBlocks.Domain/IBusinessRule.cs`

```csharp
public interface IBusinessRule
{
    Task<bool> IsBrokenAsync();
    string Message { get; }
}
```

**Что:** одно правило = один класс. Метод отвечает на вопрос «правило нарушено?».

Пример — `Alif.ServiceShop.Modules.Cart.Domain/Aggregates/Cart/BusinessRules/ModeratedOfferMustBeAvailableRule.cs`: нельзя положить в корзину больше, чем есть на складе.

**Зачем отдельный класс, а не `if` внутри метода:**
- имя класса — это документация («OrderMustBeApprovedToStartDeliveringRule»);
- правило переиспользуется в нескольких методах агрегата;
- на каждое правило легко написать юнит-тест;
- нарушение всегда даёт одинаковое исключение `BusinessRuleValidationException`, которое в API превращается в предсказуемый HTTP-ответ (см. `ProblemDetails` в `Alif.ServiceShop.API/Startup.cs`).

В модуле Orders таких правил ~30, все в `Domain/Order/BusinessRules/` — читая их имена, буквально читаешь бизнес-требования.

### 3.5 Domain Event (доменное событие)

`Alif.ServiceShop.BuildingBlocks.Domain/IDomainEvent.cs`

**Что:** факт «что-то в домене случилось», в прошедшем времени: `OrderApprovedEvent`, `OrderItemsReservedDomainEvent`.

Как выглядит внутри агрегата — `Alif.ServiceShop.Modules.Orders.Domain/Order/AggregateRoot/Order.cs:757`:

```csharp
public async Task ApproveAsync()
{
    await CheckRuleAsync(new EnsureAlifNasiyaTransactionExistRule(AlifNasiyaTransaction));  // 1. проверь правила
    if (Status.Equals(OrderStatus.Approved)) return;                                        // 2. идемпотентность
    StatusId = OrderStatus.Approved.Id;                                                     // 3. поменяй состояние
    Status  = OrderStatus.Approved;
    StatusUpdatedAt = DateTime.Now;
    AlifNasiyaTransaction!.Approve(ClientId, Origin, ClientPhone);
    AddDomainEvent(new OrderApprovedEvent(Id, ..., ExternalRef!));                           // 4. объяви факт
}
```

**Это канонический шаблон метода агрегата — запомни его.** Проверить правила → изменить себя → добавить событие. Никаких обращений к БД, HTTP или другим модулям внутри.

**Зачем:** агрегат не должен знать, что после одобрения заказа надо послать пуш, дёрнуть доставку и обновить статистику. Он просто сообщает факт. Кто на него подписан — его не касается. Добавление нового потребителя не меняет код агрегата.

### 3.6 Repository (репозиторий)

Интерфейс живёт в Domain (`Domain/<Aggregate>/Interfaces/I<X>Repository.cs`), **реализация — в Infrastructure**. Это паттерн **Dependency Inversion**: Domain объявляет, что ему нужно, Infrastructure это предоставляет; стрелка зависимости смотрит внутрь, к домену.

Пример: `Alif.ServiceShop.Modules.Cart.Domain/Aggregates/Cart/Interfaces/ICartRepository.cs` → реализация `MongoCartRepository`.

### 3.7 Enumeration

`Alif.ServiceShop.BuildingBlocks.Domain/Enumeration.cs` — «умный enum»: класс вместо `enum`, с `Id` и `Name`, к которому можно прицепить поведение.

`Alif.ServiceShop.Modules.Orders.Domain/Order/OrderStatus.cs` — весь жизненный цикл заказа:

```
Queued(0) → New(1) → Reviewing(2) → Approved(3) / ApprovedWithBan(104) / Rejected(4)
          → Delivering(7) → Completed(5) → Archived(8)
          Cancelled(6), Deleted(99)
          Processing(102), ProcessingFailed(103)
          PendingPayment(100) → ExpiredPayment(101)
```

Числа не по порядку, потому что новые статусы (100+) добавлялись позже, а старые id уже лежат в проде.

---

## Часть 4. CQRS: два разных пути данных

**CQRS** = Command Query Responsibility Segregation. Разделяем операции, которые **меняют** состояние (Command), и те, которые только **читают** (Query).

| | Command | Query |
|---|---|---|
| Смысл | «сделай» | «покажи» |
| Возврат | обычно ничего или id/VM | данные (ViewModel) |
| Путь | Handler → Агрегат → Repository → EF Core | Handler → Dapper → SQL |
| Транзакция | да (UnitOfWork) | нет |
| Проходит через домен | да | **нет, специально** |

**Почему чтение мимо домена.** Чтобы показать список заказов, домен не нужен — нужен быстрый SQL с join'ами. Прогонять это через агрегаты значит грузить лишние объекты и тормозить. Поэтому запросы читают напрямую через Dapper.

Это правило форсируется тестом `ArchTests/Modules/ApplicationLayer/QueryHandlersShouldNotInjectDomainRepositoriesTest.cs` — там же список унаследованных исключений с комментарием «не добавляй новые».

### Контракты (у каждого модуля свои, одинаковые по форме)

`Alif.ServiceShop.Modules.Orders.Application/Contracts/ICommand.cs`:

```csharp
public interface ICommand : IRequest              { Guid Id { get; } }
public interface ICommand<out TResult> : IRequest<TResult> { Guid Id { get; } }
```

`IRequest` — из MediatR. `Guid Id` у команды нужен для **идемпотентности**: по нему отложенная команда находится в таблице `internal_commands` и помечается обработанной.

Обработчик — `Alif.ServiceShop.Modules.Orders.Application/Configuration/Commands/ICommandHandler.cs`:

```csharp
public interface ICommandHandler<in TCommand, TResult> : IRequestHandler<TCommand, TResult>
    where TCommand : ICommand<TResult>;
```

Правило: класс называется `<Что-то>CommandHandler` и реализует `ICommandHandler<,>`. Проверяется тестом `CommandHandlersShouldImplementsICommandHandlerInterface`.

### Декораторы вокруг хендлера

**Декоратор** — объект, который оборачивает другой объект того же интерфейса и добавляет поведение до/после. Здесь их три, и они навешиваются автоматически при регистрации в Autofac:

1. **Logging** — `Alif.ServiceShop.Modules.Orders.Infrastructure/Configuration/Processing/LoggingCommandHandlerDecorator.cs`
2. **Validation** — `Alif.ServiceShop.Modules.Orders.Infrastructure/Configuration/Processing/ValidationCommandHandlerWithResultDecorator.cs`: прогоняет все `IValidator<T>` (FluentValidation), при ошибках бросает `InvalidCommandException` — **до** входа в хендлер
3. **UnitOfWork** — `Alif.ServiceShop.Modules.Orders.Infrastructure/Configuration/Processing/UnitOfWorkCommandHandlerWithResultDecorator.cs`: после хендлера помечает internal-команду обработанной и делает **один** `CommitAsync()`

**Зачем это, а не `try/catch` + `SaveChanges()` в каждом хендлере:** сквозная логика (логи, валидация, транзакция) написана один раз. Хендлер содержит только бизнес-сценарий, поэтому в нём никогда не увидишь `SaveChanges` или `BeginTransaction` — это не забыли, это принципиально.

---

## Часть 5. Модульный монолит: как модули изолированы

Ключевой факт, который надо принять сразу:

> **У каждого модуля свой собственный DI-контейнер Autofac, свой DbContext, своя схема таблиц, свой Quartz-планировщик и своя точка входа.**

Смотри `Alif.ServiceShop.Modules.Orders.Infrastructure/Configuration/OrdersStartup.cs` — там строится **отдельный** `ContainerBuilder` и кладётся в `Alif.ServiceShop.Modules.Orders.Infrastructure/Configuration/OrdersCompositionRoot.cs`. У Cart — свой `CartCompositionRoot`, у Catalog — свой, и т.д.

**Composition Root** — единственное место, где приложение знает обо всех классах сразу и связывает интерфейсы с реализациями.

Все модули стартуют из `Alif.ServiceShop.API/Startup.cs:294`, метод `InitializeModules`: `CatalogStartup.Initialize(...)`, `CartStartup.Initialize(...)` и т.д. — каждому передаётся своя строка подключения.

### Фасад модуля

Наружу модуль выставляет **один интерфейс** — `Alif.ServiceShop.Modules.Orders.Application/Contracts/IOrdersModule.cs`:

```csharp
public interface IOrdersModule
{
    Task<TResult> ExecuteCommandAsync<TResult>(ICommand<TResult> command);
    Task ExecuteCommandAsync(ICommand command);
    Task<TResult> ExecuteQueryAsync<TResult>(IQuery<TResult> query);
}
```

Реализация — `Alif.ServiceShop.Modules.Orders.Infrastructure/OrdersModule.cs`, она открывает scope в своём контейнере и отдаёт команду MediatR.

**Зачем:** снаружи (из контроллера) видны ровно три метода и типы команд. Ни репозитории, ни DbContext, ни сервисы модуля наружу не торчат — их и подключить-то нельзя, они в чужом контейнере.

Дополнительно есть `IOrdersModuleFacade` (в проекте `IntegrationEvents`) — для случаев, когда другому модулю нужно спросить данные синхронно.

---

## Часть 6. Полный путь HTTP-запроса (главная схема)

Разберём `POST /api/cart/moderated-items` — «добавить товар в корзину».

```
1. HTTP POST /api/cart/moderated-items
   ↓
2. CartController (Alif.ServiceShop.API/Modules/Cart/WebAndMobile/Cart/CartController.cs)
   → собирает AddModeratedItemToCartCommand из request-DTO
   → cartModule.ExecuteCommandAsync(command)
   ↓
3. CartModule → CommandsExecutor.Execute(command)
   → открывает LifetimeScope в контейнере модуля Cart
   → mediator.Send(command)
   ↓
4. MediatR находит AddModeratedItemToCartCommandHandler
   но сначала прогоняет декораторы:
      Logging → Validation (FluentValidation) → UnitOfWork
   ↓
5. AddModeratedItemToCartCommandHandler.Handle(...)
   → берёт корзину: _cartRepository.GetByBuyerAsync(buyer) ?? Cart.Create(buyer)
   → вызывает доменный метод: cart.AddModeratedCartItem(...)
        ↓ внутри агрегата: CheckRuleAsync(new ModeratedOfferMustBeAvailableRule(...))
        ↓                  изменение состояния
        ↓                  AddDomainEvent(...)
   → _cartRepository.SaveAsync(cart)
   ↓
6. UnitOfWork.CommitAsync()
   → DomainEventsDispatcher.DispatchEventsAsync()  (см. ниже)
   → SaveChanges в БД — одной транзакцией
   ↓
7. Handler возвращает ViewCartVm → контроллер отдаёт 200 OK
```

Контроллер здесь — **тонкий**: перевести HTTP в команду и обратно. Никакой логики. Это тоже проверяется ArchTests: контроллеры обязаны наследовать `BaseController`, иметь атрибуты, инжектить **только интерфейсы** и не зависеть от других контроллеров.

### Что происходит в шаге 6 подробнее

`Alif.ServiceShop.BuildingBlocks.Infrastructure/DomainEventsDispatching/DomainEventsDispatcher.cs` — важнейший класс. Он:

1. собирает все `DomainEvents` из изменённых сущностей;
2. те, что помечены `ILoggableDomainEvent`, пишет в лог доменных событий;
3. публикует доменные события **внутри модуля** через `_mediator.Publish(...)` — их ловят `DomainEventHandlers`;
4. для тех событий, у которых есть «внешняя» пара `IDomainEventNotification<>`, сериализует их в JSON и **кладёт в Outbox** (таблица `outbox_messages`) — в **той же транзакции**, что и бизнес-данные.

Имя типа для сериализации берётся не из C#-имени класса, а из явной карты `domainNotificationsMap` в `Alif.ServiceShop.Modules.Orders.Infrastructure/Configuration/OrdersStartup.cs` (`"OrderApprovedNotification" ↔ typeof(OrderApprovedNotification)`). Зачем: переименование C#-класса не должно ломать разбор сообщений, которые уже лежат в таблице.

---

## Часть 7. Outbox, Inbox, Internal Commands — почему всё через таблицы

### Проблема

Надо: (а) сохранить заказ в БД и (б) отправить событие в Kafka. Если сделать это двумя действиями, возможно: БД записалась, Kafka упала → событие потеряно навсегда. Или наоборот: событие ушло, транзакция откатилась → все узнали о заказе, которого нет.

### Outbox Pattern — решение

Событие пишется **в ту же БД, в той же транзакции**, в таблицу `outbox_messages`. Либо коммитятся оба, либо ни одного. Потом отдельная фоновая задача читает таблицу и рассылает.

`Alif.ServiceShop.Modules.Orders.Infrastructure/Configuration/Processing/Outbox/ProcessOutboxCommandHandler.cs`:

```sql
select id, type, data from outbox_messages
where processed_date is null order by occurred_on limit 500 for update
```

По каждой записи: восстановить тип по карте → десериализовать → `_mediator.Publish(...)` → проставить `processed_date`. Запускается Quartz-джобой `ProcessOutboxJob`.

Гарантия здесь — **at-least-once** (минимум один раз): если процесс упадёт между публикацией и обновлением `processed_date`, сообщение отправится повторно. Поэтому обработчики должны быть идемпотентны — отсюда проверки вида `if (Status.Equals(OrderStatus.Approved)) return;`.

### Inbox Pattern — зеркало на входящей стороне

Модуль-получатель не обрабатывает событие сразу, а сначала складывает его в `inbox_messages` — см. `IntegrationEventGenericHandler<T>` в `Alif.ServiceShop.Modules.Cart.Infrastructure/Configuration/EventsBus`. Затем `ProcessInboxJob` разбирает таблицу и публикует событие внутрь модуля через MediatR.

**Зачем:** приём и обработка развязаны. Если обработка упала — сообщение осталось в таблице и будет повторено; отправитель об этом ничего не знает и не ждёт.

### Internal Commands — отложенные команды

`ICommandsScheduler.EnqueueAsync(command)` кладёт команду в таблицу `internal_commands`, и её позже выполнит `ProcessInternalCommandsJob`.

Типичный пример — `Alif.ServiceShop.Modules.Cart.Application/Commands/ModeratedCartItems/ChangeModeratedItemPrice/ChangeModeratedItemPriceOnModeratedOfferPriceChangedIntegrationEventHandler.cs`: пришло событие «цена товара изменилась» → обработчик не меняет корзины сам, а ставит команду в очередь. Обработка события становится мгновенной и надёжной, тяжёлая работа уезжает в фон.

### Полная картина межмодульного обмена

```
Модуль Catalog                                    Модуль Cart
──────────────                                    ───────────
Aggregate.AddDomainEvent()
        ↓ (та же транзакция)
   outbox_messages
        ↓ ProcessOutboxJob (Quartz)
   MediatR.Publish → Notification handler
        ↓
   IEventsBus.Publish(IntegrationEvent)  ─────▶   IntegrationEventGenericHandler
   (InMemoryEventBus или Kafka)                          ↓
                                                   inbox_messages
                                                          ↓ ProcessInboxJob (Quartz)
                                                   MediatR.Publish
                                                          ↓
                                                   INotificationHandler<...IntegrationEvent>
                                                          ↓
                                                   ICommandsScheduler.EnqueueAsync(Command)
                                                          ↓
                                                   internal_commands
                                                          ↓ ProcessInternalCommandsJob
                                                   CommandHandler → Агрегат Cart
```

Выглядит громоздко — но именно эта цепочка даёт: атомарность, повторяемость при сбоях и полную развязку модулей. Ни одной прямой ссылки Catalog → Cart в коде нет.

### Domain Event vs Integration Event — не путать

| | Domain Event | Integration Event |
|---|---|---|
| Где живёт | `Domain/<Aggregate>/DomainEvents/` | проект `*.IntegrationEvents` |
| Аудитория | внутри своего модуля | другие модули / внешние сервисы |
| Транспорт | MediatR в памяти | Outbox → EventBus / Kafka |
| Можно менять поля | легко | **нет** — это публичный контракт |

Базовый класс интеграционного события: `Alif.ServiceShop.BuildingBlocks.Infrastructure/EventBus/IntegrationEvent.cs`. Пример: `Alif.ServiceShop.Modules.Catalog.IntegrationEvents/ModeratedOffer/ModeratedOfferPriceChangedIntegrationEvent.cs`.

### Шина событий

`Alif.ServiceShop.BuildingBlocks.Infrastructure/EventBus/IEventsBus.cs` — абстракция с тремя методами: `Publish`, `Subscribe`, `StartConsuming`. Реализации: `InMemoryEventBusClient` (по умолчанию, обмен внутри процесса) и Kafka (`KafkaBusModule` для обмена с внешними сервисами Alif: платежи, доставка, кредиты).

Подписки объявляются явным списком в `EventsBusStartup.SubscribeToIntegrationEvents(...)` каждого модуля. Если событие «не доходит» — проверь в первую очередь, есть ли строка `SubscribeToIntegrationEvent<...>` там.

---

## Часть 8. Хранилища данных

| Что | Где используется |
|---|---|
| **MySQL** | Основное хранилище. У каждого модуля **своя строка подключения** (`OrdersConnectionString`, `CatalogConnectionString`, ...) — см. константы в `Alif.ServiceShop.API/Startup.cs`. Записи через EF Core, чтение через Dapper |
| **MongoDB** | **Только модуль Cart**: сам агрегат `Cart` хранится в Mongo (отсюда атрибуты `[BsonId]`, `[BsonElement]` прямо в доменном классе и `MongoCartRepository`). При этом служебные таблицы Cart (inbox/outbox/internal_commands) живут в MySQL |
| **Redis** | Кэш (`IDistributedCache`), ключи DataProtection, сессии корзины |
| **MinIO / Firebase** | Файлы и изображения; Firebase — пуши |

> Про `[BsonId]` в домене: строго по DDD инфраструктурные атрибуты в Domain нежелательны. Здесь пошли на компромисс ради простоты маппинга. Знай, что это осознанное отступление, а не образец для подражания.

### Миграции — важное правило

Миграции делает **FluentMigrator**, а не EF Migrations. Файл называется `YYYY_MM_DD_HHMMSS_описание.cs`, а число в атрибуте `[Migration(...)]` обязано совпадать с таймстемпом в имени.

Эталон с подробным русским комментарием: `Alif.ServiceShop.Modules.Orders.Infrastructure/Migrations/2024_01_01_000001_create_examples_table.cs` — прочитай его до первой своей миграции.

**Никогда не пиши миграцию руками.** В проекте есть скилл `create-migration` — он ставит таймстемп по реальным часам и правильно именует файл. Выдуманный таймстемп ломает порядок накатки у всей команды.

---

## Часть 9. Модули по одному

### Orders — заказы (самый большой и сложный)

Отвечает за весь жизненный цикл заказа и за интеграции с платёжными системами.

Агрегаты: `Order`, `Purchase`, `BuyerCard`, `PartnerClone`, `OrderPromoCodeUsage`.

Способы оплаты (главный источник сложности):
- **AlifNasiya BNPL** — рассрочка, заявка в кредитную систему;
- **AlifNasiya BML** (Buy Me Later) — отдельный кредитный продукт со своими лимитами;
- **AlifMobi Native / Acquiring** — оплата картой внутри приложения AlifMobi;
- наличные / прочее.

Под каждый способ — своя цепочка: `Domain/Order/Strategies/` (`IOrderCreationStrategy` + `OrderCreationStrategyResolver`), `Domain/Order/Processing/` (`IOrderProcessor`), плюс шлюзы `Application/Gateways/` (Nasiya, Credits, Scoring, Delivery).

**Паттерн Strategy** здесь буквально: «как создать заказ» зависит от способа оплаты, резолвер выбирает нужную реализацию. Добавление нового способа оплаты = новая стратегия, а не новый `if` в общем методе.

Куда смотреть первым делом: `Domain/Order/BusinessRules/` — по именам файлов читаются все правила заказа.

### Catalog — каталог (самый широкий по числу сущностей)

Товары (`Model`), офферы продавцов (`Offer`), одобренные офферы (`ModeratedOffer`), категории, бренды, атрибуты, серии, вариации, промо, баннеры, карусели, витрины, курсы валют, интеграция с Billz, импорт офферов.

**FBO / FBS** — модели фулфилмента: FBO = товар на складе маркетплейса, FBS = на складе продавца. От этого зависят сроки и логика доставки.

`ModeratedOffer` — центральная сущность каталога: именно её изменения (цена, количество, активность) разлетаются интеграционными событиями по всем модулям.

### Cart — корзина

Агрегаты: `Cart`, `SharedCart` (корзина, которой можно поделиться), `Wishlist`.

Основная работа модуля — **поддерживать корзину в актуальном состоянии**: подписки на десяток событий каталога (цена изменилась, товар деактивировали, склад опустел, товар переехал в другой магазин) → отложенные команды `Actualize*` / `ChangeModerated*`. Это самый наглядный модуль, чтобы понять event-driven-часть: посмотри список подписок в `Cart.Infrastructure/Configuration/EventsBus/EventsBusStartup.cs`.

### Marketing — маркетинг и коммуникации

Клиенты, устройства (`Device`), адреса доставки клиента, «хотелки» (`ClientWish`), промокоды (`PromoCode`), пуш-кампании и уведомления, подавленные уведомления.

Модуль-«отправитель»: подписывается на события заказов и каталога и превращает их в уведомления.

### Moderation — модерация

`OfferDemand` — заявка продавца на публикацию оффера, причины отказа (`OfferDemandRejectionReason`), неверно заполненные секции (`WrongFilledSection`), клоны партнёров.

Цепочка: продавец создаёт заявку → модератор одобряет/отклоняет → событие → Catalog создаёт `ModeratedOffer` → товар появляется в витрине.

---

## Часть 10. Тесты

| Тип | Проект | Что проверяет |
|---|---|---|
| **ArchTests** | `ArchTests` | архитектурные правила через рефлексию (NetArchTest) |
| **Domain.UnitTests** | `*.Domain.UnitTests` | агрегаты и бизнес-правила — быстрые, без БД |
| **Application.UnitTests** | `*.Application.UnitTests` | хендлеры с замоканными зависимостями |
| **IntegrationTests** | `Marketing.Application.IntegrationTests` | сценарии с реальной инфраструктурой |

**ArchTests** — фишка проекта, которую редко встретишь. Это обычные xUnit-тесты, которые загружают сборку и проверяют структуру кода:

- агрегаты имеют только приватные конструкторы;
- агрегаты создаются только фабричными методами;
- Domain не зависит от Application;
- CommandHandler'ы реализуют `ICommandHandler<,>`;
- QueryHandler'ы не инжектят доменные репозитории;
- контроллеры наследуют `BaseController`, имеют нужные атрибуты, инжектят только интерфейсы, не зависят друг от друга;
- админские контроллеры имеют атрибут прав доступа;
- API-клиенты наследуют базовый клиент;
- параметры конструктора доменного события совпадают с самим событием.

**Запускай их после любого структурного изменения** — они ловят нарушения раньше ревьюера:

```bash
dotnet test ArchTests/
```

---

## Часть 11. Правила проекта, которые нельзя нарушать

1. **Логирование — только Serilog.** В коде увидишь `using ILogger = Serilog.ILogger;`. `Microsoft.Extensions.Logging` не использовать.
2. **DI — только Autofac.** Регистрации — в `AutofacDependencyConfigurator` / `*Startup` модуля.
3. **Конструкторы агрегатов — только `private`**, создание через `Create(...)`.
4. **Командный хендлер** называется `*CommandHandler` и реализует `ICommandHandler<,>`.
5. **Любое изменение схемы БД — только через скилл `create-migration`.** Даже одна колонка, даже внутри большой задачи.
6. **Межмодульное общение — только через Integration Event или фасад модуля.** Прямая ссылка на классы инфраструктуры чужого модуля запрещена.
7. **Интеграционные события публикуются через Outbox**, не синхронно.
8. **Сложные запросы — через Ardalis.Specification** (`Domain/<Aggregate>/Specifications/`), а не сырым SQL в хендлере.
9. **Предупреждения компилятора = ошибки.** Форматирование по StyleCop обязательно.

Перед изменением кода в проекте рекомендуется искать через граф кода (Neo4j, MCP-инструменты `mcp__code-graph__*`), а не грепом — см. `CLAUDE.md`.

---

## Часть 12. Как добавить фичу — чеклист

```
1. Определи модуль по ответственности (заказ? каталог? корзина?)
2. Domain:
   - нужен новый тип идентификатора → Value Object
   - новое ограничение → класс в Rules/, реализует IBusinessRule
   - метод агрегата: CheckRuleAsync(...) → изменить состояние → AddDomainEvent(...)
3. Application:
   - <Действие>Command.cs        (наследует CommandBase / CommandBase<TResult>)
   - <Действие>CommandHandler.cs (реализует ICommandHandler<,>)
   - при необходимости FluentValidation-валидатор
4. Infrastructure:
   - репозиторий/конфигурация EF, если появились новые поля
   - регистрация новых сервисов в *Startup.cs
   - миграция → ТОЛЬКО через скилл create-migration
5. API: тонкий метод контроллера → команда → module.ExecuteCommandAsync
6. Нужно оповестить другой модуль?
   - IntegrationEvent в *.IntegrationEvents
   - IDomainEventNotification + запись в domainNotificationsMap в *Startup.cs
   - подписка SubscribeToIntegrationEvent<T> в EventsBusStartup модуля-получателя
   - обработчик INotificationHandler<T> → ICommandsScheduler.EnqueueAsync(...)
7. Тесты: Domain.UnitTests на правило + dotnet test ArchTests/
```

---

## Часть 13. Маршрут чтения кода — первые дни

**День 1 — базовые кирпичи (маленькие файлы, читаются за час):**
1. `Alif.ServiceShop.BuildingBlocks.Domain/Entity.cs`
2. `Alif.ServiceShop.BuildingBlocks.Domain/IBusinessRule.cs`
3. `Alif.ServiceShop.BuildingBlocks.Domain/IDomainEvent.cs`
4. `Alif.ServiceShop.BuildingBlocks.Domain/ValueObject.cs`
5. `Alif.ServiceShop.Modules.Orders.Application/Contracts/ICommand.cs` + `Alif.ServiceShop.Modules.Orders.Application/Configuration/Commands/ICommandHandler.cs`

**День 2 — один сквозной сценарий, от HTTP до БД (модуль Cart, он самый простой):**
1. `Alif.ServiceShop.API/Modules/Cart/WebAndMobile/Cart/CartController.cs`
2. `Alif.ServiceShop.Modules.Cart.Application/Commands/ModeratedCartItems/AddModeratedItemToCart/AddModeratedItemToCartCommand.cs`
3. `Alif.ServiceShop.Modules.Cart.Application/Commands/ModeratedCartItems/AddModeratedItemToCart/AddModeratedItemToCartCommandHandler.cs`
4. `Alif.ServiceShop.Modules.Cart.Domain/Aggregates/Cart/Cart.cs`
5. `Alif.ServiceShop.Modules.Cart.Domain/Aggregates/Cart/BusinessRules/ModeratedOfferMustBeAvailableRule.cs`

**День 3 — инфраструктура и события:**
1. `Alif.ServiceShop.Modules.Orders.Infrastructure/Configuration/OrdersStartup.cs` — как собирается контейнер
2. `Alif.ServiceShop.BuildingBlocks.Infrastructure/DomainEventsDispatching/DomainEventsDispatcher.cs`
3. `Alif.ServiceShop.Modules.Orders.Infrastructure/Configuration/Processing/Outbox/ProcessOutboxCommandHandler.cs`
4. `Cart.Infrastructure/Configuration/EventsBus/` — подписки и Inbox

**День 4 — сложный домен:**
1. `Alif.ServiceShop.Modules.Orders.Domain/Order/AggregateRoot/Order.cs` — метод `ApproveAsync` и рядом
2. `Orders.Domain/Order/BusinessRules/` — пробежаться по именам всех файлов
3. `Alif.ServiceShop.Modules.Orders.Domain/Order/OrderStatus.cs`

**Не забудь про `docs/adr`** — Architecture Decision Records, короткие записки «почему решили так»:
- `0001-unified-reserve-release-events.md`
- `0002-alif-card-promotion-toggle-infrastructure.md`
- `0003-actor-audit-moderated-offer.md`

И `readme.md` — обзор продукта целиком.

---

## Часть 14. Что почитать вне проекта

**Именно этот стиль архитектуры** (проект почти буквально следует ему):
- Kamil Grzybek, *Modular Monolith with DDD* — https://github.com/kgrzybek/modular-monolith-with-ddd — тот же Outbox/Inbox, те же декораторы, те же ArchTests. Лучший источник по этому коду.

**DDD:**
- Vaughn Vernon, *Implementing Domain-Driven Design* (или его же «Domain-Driven Design Distilled» — короче)
- Microsoft: https://learn.microsoft.com/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/

**Паттерны, встречающиеся в коде:**
- Outbox — https://microservices.io/patterns/data/transactional-outbox.html
- CQRS — https://martinfowler.com/bliki/CQRS.html
- Decorator, Strategy, Repository, Specification — Refactoring Guru: https://refactoring.guru/design-patterns

**Библиотеки:**
- MediatR — https://github.com/jbogard/MediatR/wiki
- Autofac — https://autofac.readthedocs.io/
- EF Core — https://learn.microsoft.com/ef/core/
- Dapper — https://www.learndapper.com/
- FluentMigrator — https://fluentmigrator.github.io/
- FluentValidation — https://docs.fluentvalidation.net/
- Ardalis.Specification — https://specification.ardalis.com/
- Quartz.NET — https://www.quartz-scheduler.net/documentation/
- Serilog — https://serilog.net/

**Основы C#/.NET, если стек новый:**
- https://learn.microsoft.com/dotnet/csharp/ — раздел про `async/await` и nullable reference types в первую очередь

---

## Шпаргалка: «где искать, если…»

| Вопрос | Куда смотреть |
|---|---|
| Откуда взялся HTTP-эндпоинт? | `Alif.ServiceShop.API/Modules/<Модуль>/**/...Controller.cs` |
| Где бизнес-правило X? | `<Модуль>.Domain/<Агрегат>/BusinessRules/` |
| Почему заказ не меняет статус? | метод агрегата в `Order.cs` + правило, которое он проверяет |
| Кто зарегистрировал этот сервис? | `<Модуль>.Infrastructure/Configuration/<Модуль>Startup.cs` |
| Почему событие не дошло до модуля? | `EventsBusStartup` получателя (есть ли подписка) → таблица `inbox_messages` → `ProcessInboxJob` |
| Событие не ушло наружу? | `domainNotificationsMap` в `*Startup.cs` → таблица `outbox_messages` → `ProcessOutboxJob` |
| Что за фоновая задача? | `<Модуль>.Infrastructure/Configuration/Quartz/` и `Processing/*Job.cs` |
| Где SQL для списка? | `<Модуль>.Application/Queries/**/...QueryHandler.cs` (Dapper) |
| Как поменять схему БД? | скилл `create-migration`, файл ляжет в `<Модуль>.Infrastructure/Migrations/` |
| Сборка падает на непонятном warning'е | `TreatWarningsAsErrors` + StyleCop — почини формат/nullable |

---

## Связанное

- [[93 — Рабочий проект: Alif ServiceShop (обзор раздела)]]
- [[ServiceShop: модульный монолит и изоляция модулей]]
- [[ServiceShop: домен и тактический DDD]]
- [[ServiceShop: CQRS и путь запроса]]
- [[ServiceShop: Outbox, Inbox и обмен событиями]]
- [[ServiceShop: данные, миграции и тесты]]
