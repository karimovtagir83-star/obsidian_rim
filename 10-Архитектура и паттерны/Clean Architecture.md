---
tags: [раздел-10, архитектура, паттерны, проектирование, middle, собес, dotnet10]
aliases: [Clean Architecture, Чистая архитектура, Hexagonal Architecture, Ports and Adapters, Onion Architecture, Луковая архитектура]
---

# Clean Architecture

> [!abstract] Коротко
> Clean Architecture — это не структура папок и не шаблон солюшена из шести проектов. Это одно правило: **зависимости направлены только внутрь, к бизнес-логике**. Домен ничего не знает про EF Core, HTTP и Kafka; знание идёт в обратную сторону через интерфейсы, объявленные внутри и реализованные снаружи.
> Всё остальное — сборки, папки, `IRepository`, `MediatR` — это способы это правило соблюсти, а не сама архитектура. И у каждого способа есть цена, которую в туториалах не считают: одно новое поле в сущности стоит 8–10 файлов вместо двух.
> Мидл отличается от джуна не тем, что умеет развернуть шаблон Clean Architecture, а тем, что понимает, когда этот шаблон дороже задачи.

## Зачем это нужно

Возьмём типичный «быстрый» бэкенд: контроллер получает запрос, внутри контроллера — `DbContext`, LINQ-запрос, проверка правил, отправка письма через `SmtpClient`, возврат сущности как JSON. Это работает. Ровно до момента, когда происходит одно из:

- **Правило начинает дублироваться.** «Нельзя отменить заказ после отгрузки» проверяется в `CancelOrder`, в `RefundOrder`, в фоновом джобе очистки корзин и в админском эндпоинте. Четыре реализации, три из них устарели.
- **Правило нельзя протестировать без базы.** Чтобы проверить «скидка не больше 30 %», нужно поднять PostgreSQL, накатить миграции, засеять данные и сделать HTTP-запрос. Тест на одну строчку арифметики идёт 4 секунды и падает, когда сеть моргнула.
- **Инфраструктура протекает в модель.** В сущность добавили `[JsonIgnore]`, `[Column("ord_total")]`, `virtual` ради ленивой загрузки и публичный сеттер, потому что EF Core так удобнее. Теперь сущность — это не модель предметной области, а форма таблицы плюс форма JSON.
- **Меняется внешняя система.** SMTP заменили на транзакционную рассылку через HTTP API. Правки — в 14 местах, потому что `SmtpClient` был вкраплён везде.

Clean Architecture решает ровно это: **бизнес-правила складываются в место, которое не зависит ни от чего, и потому тестируется, читается и переиспользуется**. Всё остальное объявляется деталью.

> [!quote] Формулировка Роберта Мартина
> «Архитектура системы должна позволять отложить решение о базе данных и о вебе на как можно более поздний срок. База данных — это деталь. Веб — это деталь ввода-вывода».
> Это утверждение полезно как мышление и опасно как буквальная инструкция. На практике решение о базе не откладывают: выбор PostgreSQL против Kafka-as-source-of-truth меняет всё. Правильное прочтение: «не позволяй SQL диктовать форму бизнес-правил», а не «пиши приложение, готовое к смене СУБД».

---

## Родословная: Hexagonal, Onion, Clean

Три названия, одна идея, разные словари. Это важно знать, потому что на собеседовании и в чужом коде вы встретите все три, и человек напротив может думать, что это разные вещи.

| Год | Название | Автор | Ключевая метафора | Словарь |
|---|---|---|---|---|
| 2005 | Hexagonal Architecture (Ports and Adapters) | Алистер Кокбёрн | Приложение — шестиугольник, снаружи адаптеры | Порт (интерфейс), адаптер (реализация), driving/driven side |
| 2008 | Onion Architecture | Джеффри Палермо | Концентрические слои-луковица | Domain Model, Domain Services, Application Services, Infrastructure |
| 2012 | Clean Architecture | Роберт Мартин | Те же круги + явное «правило зависимостей» | Entities, Use Cases, Interface Adapters, Frameworks & Drivers |

### Что у них общего

Ровно одно и то же ядро:

1. Бизнес-логика в центре и не зависит ни от чего внешнего.
2. Взаимодействие с внешним миром — через абстракции, которые принадлежат центру.
3. Технологии на периферии подключаются, а не встраиваются.

### Чем отличаются формулировки

- **Hexagonal** делает акцент на **симметрии ввода-вывода**. У Кокбёрна нет иерархии «HTTP выше базы»: и то, и другое — просто адаптеры вокруг одного и того же приложения. Отсюда сильная идея: тест — это тоже адаптер, он подключается к тем же портам, что и HTTP. Слоёв как таковых у Кокбёрна нет — есть «внутри» и «снаружи». Шесть сторон не значат ничего, их просто удобно рисовать.
- **Onion** первым явно нарисовал **несколько внутренних слоёв** и ввёл разделение «модель домена» / «доменные сервисы» / «сервисы приложения». Именно Onion популяризировал структуру солюшена, которую сегодня зовут Clean Architecture.
- **Clean** дал самое чёткое высказывание — Dependency Rule — и добавил словарь use case'ов: сценарий приложения становится **явным объектом** (`CreateOrderHandler`), а не методом в разросшемся `OrderService`. Плюс Мартин ввёл идею «граничных DTO»: через границу слоя передаются простые структуры данных, а не сущности и не фреймворочные типы.

> [!info] Практический вывод
> Если коллега говорит «у нас гексагональная», а вы видите `Domain`/`Application`/`Infrastructure`/`Api` — спорить не о чем, это одно и то же с точностью до слов. Различия начинаются только в деталях: где живут интерфейсы и есть ли отдельный слой use case'ов.

---

## Правило зависимостей

Единственное настоящее содержание Clean Architecture.

> **Зависимости в исходном коде направлены только внутрь. Внутренний слой ничего не знает об именах, типах и существовании внешнего.**

```
                    снаружи внутрь: направление зависимостей
        ────────────────────────────────────────────────────────►

  ┌──────────────────────────────────────────────────────────────────┐
  │  Frameworks & Drivers / Infrastructure                           │
  │  ASP.NET Core, EF Core, Npgsql, RabbitMQ, SMTP, Redis, файлы     │
  │                                                                  │
  │   ┌────────────────────────────────────────────────────────────┐ │
  │   │  Interface Adapters                                        │ │
  │   │  Endpoints/Controllers, Repositories (реализации),         │ │
  │   │  EntityTypeConfiguration, Presenters, DTO-мапперы          │ │
  │   │                                                            │ │
  │   │   ┌──────────────────────────────────────────────────────┐ │ │
  │   │   │  Use Cases (Application)                             │ │ │
  │   │   │  CreateOrderHandler, CancelOrderHandler,             │ │ │
  │   │   │  IOrderRepository (объявление!), IEmailSender        │ │ │
  │   │   │                                                      │ │ │
  │   │   │   ┌────────────────────────────────────────────────┐ │ │ │
  │   │   │   │  Entities (Domain)                             │ │ │ │
  │   │   │   │  Order, OrderLine, Money, DiscountPolicy,      │ │ │ │
  │   │   │   │  инварианты, доменные события                  │ │ │ │
  │   │   │   │                                                │ │ │ │
  │   │   │   │  Зависимости: только BCL. Ноль пакетов.        │ │ │ │
  │   │   │   └────────────────────────────────────────────────┘ │ │ │
  │   │   └──────────────────────────────────────────────────────┘ │ │
  │   └────────────────────────────────────────────────────────────┘ │
  └──────────────────────────────────────────────────────────────────┘

  Поток управления во время работы:  HTTP → Endpoint → Handler → Domain
  Поток зависимостей в коде:         HTTP → Endpoint → Handler → Domain
  Поток обращения к БД:              Handler → IOrderRepository ← EfOrderRepository
                                     (управление идёт наружу, зависимость — внутрь)
```

### «Слой» — это про направление зависимостей, а не про папку

Самое частое непонимание. Люди создают четыре проекта, радуются и на этом останавливаются, а внутри `Application` спокойно лежит `using Microsoft.EntityFrameworkCore;` и `DbContext` с `Include`. Папки есть, архитектуры нет.

И наоборот: можно держать всё в **одной сборке** с папками `Domain/`, `Application/`, `Infrastructure/` и соблюдать правило зависимостей строже, чем в многие «правильные» солюшены — если это проверяется архитектурным тестом.

Практический критерий, по которому проверяется наличие архитектуры, ровно один:

> Возьмите файл из внутреннего слоя. Посмотрите его `using`-и. Если там есть тип из внешнего слоя — правило нарушено, независимо от того, сколько у вас проектов.

Сборки полезны тем, что превращают правило из соглашения в **ошибку компиляции**. Это единственное их преимущество и единственная причина их плодить. Всё остальное — цена, см. раздел с критикой.

---

## Инверсия зависимостей как механизм

Правило зависимостей физически невозможно соблюсти без инверсии: сценарию нужно сохранить заказ, а хранилище — снаружи. Разрешает это конфликт то, что подробно разобрано в [[Инверсия зависимостей на практике]] и в DIP из [[SOLID]]: **интерфейс объявляется там, где он потребляется, а не там, где реализуется**.

```csharp
// ============ src/Domain ============
// Ноль знаний о хранилище, HTTP, DI. Только правила.
namespace Domain.Entities;

public sealed class Order
{
    private readonly List<OrderLine> _lines = [];

    public Guid Id { get; private set; }
    public Guid CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public IReadOnlyList<OrderLine> Lines => _lines;

    // Инвариант: сумма считается, а не хранится как изменяемое поле извне.
    public Money Total => _lines.Aggregate(Money.Zero("RUB"), (sum, l) => sum + l.Subtotal);

    private Order() { }   // для материализации ORM — см. раздел о протекании

    public static Order Place(Guid customerId, IEnumerable<OrderLine> lines)
    {
        var list = lines.ToList();
        if (list.Count == 0)
            throw new DomainException("Заказ не может быть пустым.");

        return new Order
        {
            Id = Guid.CreateVersion7(),   // .NET 9+: UUIDv7, монотонный — дружелюбен к индексу
            CustomerId = customerId,
            Status = OrderStatus.Placed,
            _lines = { } // заполняется ниже, чтобы список остался приватным
        }.WithLines(list);
    }

    private Order WithLines(List<OrderLine> lines)
    {
        _lines.AddRange(lines);
        return this;
    }

    // Настоящее бизнес-правило. Тестируется без базы, без моков, за микросекунды.
    public void Cancel(DateTimeOffset now)
    {
        if (Status is OrderStatus.Shipped or OrderStatus.Delivered)
            throw new DomainException("Нельзя отменить отгруженный заказ.");

        Status = OrderStatus.Cancelled;
        CancelledAt = now;
    }

    public DateTimeOffset? CancelledAt { get; private set; }
}
```

```csharp
// ============ src/Application ============
// Интерфейс объявлен ЗДЕСЬ, потому что ЗДЕСЬ он нужен.
// Application формулирует свою потребность на своём языке.
namespace Application.Common.Interfaces;

public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(Guid id, CancellationToken ct);
    void Add(Order order);
}

public interface IEmailSender
{
    Task SendAsync(string to, string subject, string body, CancellationToken ct);
}
```

```csharp
// ============ src/Application ============
// Сценарий. Знает Domain и свои интерфейсы. Не знает EF Core, HTTP, SMTP.
namespace Application.Orders.Commands.CancelOrder;

public sealed class CancelOrderHandler(
    IOrderRepository orders,
    IUnitOfWork uow,
    IEmailSender email,
    TimeProvider clock)
{
    public async Task<Result> HandleAsync(CancelOrderCommand command, CancellationToken ct)
    {
        var order = await orders.GetByIdAsync(command.OrderId, ct);
        if (order is null)
            return Result.NotFound($"Заказ {command.OrderId} не найден.");

        order.Cancel(clock.GetUtcNow());       // правило живёт в домене
        await uow.SaveChangesAsync(ct);        // транзакционная граница — здесь

        await email.SendAsync(command.CustomerEmail, "Заказ отменён", "...", ct);
        return Result.Success();
    }
}
```

```csharp
// ============ src/Infrastructure ============
// Реализация. Знает и Application (чтобы реализовать интерфейс), и EF Core.
namespace Infrastructure.Persistence.Repositories;

internal sealed class EfOrderRepository(AppDbContext db) : IOrderRepository
{
    public Task<Order?> GetByIdAsync(Guid id, CancellationToken ct) =>
        db.Orders.Include(o => o.Lines).FirstOrDefaultAsync(o => o.Id == id, ct);

    public void Add(Order order) => db.Orders.Add(order);
}
```

```csharp
// ============ src/WebApi/Program.cs — точка входа ============
// Единственное место, которое знает ВСЁ. Здесь склейка.
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddApplication();        // хендлеры, валидаторы
builder.Services.AddInfrastructure(builder.Configuration);  // DbContext, репозитории, SMTP
builder.Services.AddSingleton(TimeProvider.System);

var app = builder.Build();
app.MapOrderEndpoints();
app.Run();
```

Что здесь произошло механически: `Application.dll` ссылается на `Domain.dll`; `Infrastructure.dll` ссылается на `Application.dll`; `WebApi.dll` ссылается на всё. **Ни один из внутренних проектов не ссылается на `Microsoft.EntityFrameworkCore`** — потому что тип `IOrderRepository` компилируется без него. Это и есть ответ на вопрос «почему Domain не знает про EF Core»: не потому что кто-то так решил, а потому что нужного `using` в его графе ссылок физически нет.

Подробнее про то, как контейнер это собирает и какие бывают ошибки времени старта, — [[Dependency Injection: контейнер ASP.NET Core]] и [[Captive dependency и типичные ошибки DI]].

---

## Дерево солюшена на .NET 10

Полный, реальный расклад. Это не «канон», это распространённый вариант, от которого удобно отсчитывать упрощения.

```
OrderService.sln
├── Directory.Build.props            общие свойства: LangVersion, Nullable, TreatWarningsAsErrors
├── Directory.Packages.props         Central Package Management: версии пакетов в одном месте
├── .editorconfig
├── src/
│   ├── Domain/
│   │   ├── Domain.csproj
│   │   ├── Common/
│   │   │   ├── Entity.cs                    базовый класс: Id, доменные события
│   │   │   ├── AggregateRoot.cs
│   │   │   ├── ValueObject.cs               Equals/GetHashCode по значению
│   │   │   └── IDomainEvent.cs
│   │   ├── Entities/
│   │   │   ├── Order.cs
│   │   │   ├── OrderLine.cs
│   │   │   └── Customer.cs
│   │   ├── ValueObjects/
│   │   │   ├── Money.cs
│   │   │   ├── Address.cs
│   │   │   └── Email.cs
│   │   ├── Events/
│   │   │   ├── OrderPlacedEvent.cs
│   │   │   └── OrderCancelledEvent.cs
│   │   ├── Exceptions/
│   │   │   ├── DomainException.cs
│   │   │   └── OrderCannotBeCancelledException.cs
│   │   └── Enums/
│   │       └── OrderStatus.cs
│   │
│   ├── Application/
│   │   ├── Application.csproj
│   │   ├── DependencyInjection.cs           AddApplication()
│   │   ├── Common/
│   │   │   ├── Interfaces/
│   │   │   │   ├── IOrderRepository.cs
│   │   │   │   ├── IUnitOfWork.cs
│   │   │   │   ├── IEmailSender.cs
│   │   │   │   ├── ICurrentUser.cs
│   │   │   │   └── IOrderReadStore.cs       для запросов, отдельно от записи
│   │   │   ├── Behaviors/
│   │   │   │   ├── ValidationDecorator.cs
│   │   │   │   ├── LoggingDecorator.cs
│   │   │   │   └── TransactionDecorator.cs
│   │   │   ├── Models/
│   │   │   │   ├── Result.cs
│   │   │   │   └── PagedList.cs
│   │   │   └── Mapping/
│   │   │       └── OrderMappings.cs         ручные extension-мапперы
│   │   └── Orders/
│   │       ├── Commands/
│   │       │   ├── CreateOrder/
│   │       │   │   ├── CreateOrderCommand.cs
│   │       │   │   ├── CreateOrderHandler.cs
│   │       │   │   └── CreateOrderValidator.cs
│   │       │   └── CancelOrder/
│   │       │       ├── CancelOrderCommand.cs
│   │       │       ├── CancelOrderHandler.cs
│   │       │       └── CancelOrderValidator.cs
│   │       └── Queries/
│   │           ├── GetOrderById/
│   │           │   ├── GetOrderByIdQuery.cs
│   │           │   ├── GetOrderByIdHandler.cs
│   │           │   └── OrderDetailsDto.cs
│   │           └── ListOrders/
│   │               ├── ListOrdersQuery.cs
│   │               ├── ListOrdersHandler.cs
│   │               └── OrderListItemDto.cs
│   │
│   ├── Infrastructure/
│   │   ├── Infrastructure.csproj
│   │   ├── DependencyInjection.cs           AddInfrastructure(IConfiguration)
│   │   ├── Persistence/
│   │   │   ├── AppDbContext.cs
│   │   │   ├── UnitOfWork.cs
│   │   │   ├── Configurations/
│   │   │   │   ├── OrderConfiguration.cs    IEntityTypeConfiguration<Order>
│   │   │   │   ├── OrderLineConfiguration.cs
│   │   │   │   └── CustomerConfiguration.cs
│   │   │   ├── Repositories/
│   │   │   │   ├── EfOrderRepository.cs
│   │   │   │   └── DapperOrderReadStore.cs  чтение мимо EF Core
│   │   │   └── Interceptors/
│   │   │       ├── AuditableEntityInterceptor.cs
│   │   │       └── DomainEventsInterceptor.cs
│   │   ├── Migrations/
│   │   │   ├── 20260315090412_Initial.cs
│   │   │   └── AppDbContextModelSnapshot.cs
│   │   └── Services/
│   │       ├── SendGridEmailSender.cs
│   │       └── BlobFileStorage.cs
│   │
│   └── WebApi/
│       ├── WebApi.csproj
│       ├── Program.cs
│       ├── appsettings.json
│       ├── Endpoints/
│       │   ├── OrderEndpoints.cs
│       │   └── CustomerEndpoints.cs
│       ├── Middleware/
│       │   └── ExceptionHandlingMiddleware.cs
│       └── Contracts/
│           ├── CreateOrderRequest.cs        контракт API ≠ команда Application
│           └── OrderResponse.cs
│
└── tests/
    ├── Domain.UnitTests/                    чистые правила, ноль моков
    ├── Application.UnitTests/               сценарии с подменёнными интерфейсами
    ├── Infrastructure.IntegrationTests/     реальный PostgreSQL через Testcontainers
    ├── WebApi.FunctionalTests/              WebApplicationFactory, HTTP насквозь
    └── ArchitectureTests/                   правило зависимостей как тест
```

### Ключевые `.csproj`

```xml
<!-- src/Domain/Domain.csproj — НИ ОДНОЙ ссылки. Ни проектной, ни пакетной. -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>
</Project>
```

```xml
<!-- src/Application/Application.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>
  <ItemGroup>
    <ProjectReference Include="..\Domain\Domain.csproj" />
  </ItemGroup>
  <ItemGroup>
    <!-- только абстракции и валидация, никаких драйверов -->
    <PackageReference Include="Microsoft.Extensions.DependencyInjection.Abstractions" />
    <PackageReference Include="Microsoft.Extensions.Logging.Abstractions" />
    <PackageReference Include="FluentValidation" />
  </ItemGroup>
</Project>
```

```xml
<!-- src/Infrastructure/Infrastructure.csproj — здесь весь «грязный» мир -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
  </PropertyGroup>
  <ItemGroup>
    <ProjectReference Include="..\Application\Application.csproj" />
  </ItemGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.EntityFrameworkCore" />
    <PackageReference Include="Npgsql.EntityFrameworkCore.PostgreSQL" />
    <PackageReference Include="Dapper" />
  </ItemGroup>
</Project>
```

```xml
<!-- src/WebApi/WebApi.csproj — единственная точка, знающая обо всём -->
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
  </PropertyGroup>
  <ItemGroup>
    <ProjectReference Include="..\Application\Application.csproj" />
    <ProjectReference Include="..\Infrastructure\Infrastructure.csproj" />
  </ItemGroup>
</Project>
```

### Почему у Domain.csproj нет ни одного пакета

Это не эстетика, это три конкретных следствия.

1. **Компилятор становится сторожем.** Написать `[Column("total")]` в `Order` физически нельзя — тип не найден. Правило зависимостей соблюдается не дисциплиной, а сборкой.
2. **Домен не наследует ничьих версий.** Обновление EF Core с 9 на 10 не может сломать сущность. Транзитивные конфликты версий (см. [[NuGet: пакеты, версии, транзитивные зависимости]]) не доходят до ядра.
3. **Тесты домена мгновенные.** `Domain.UnitTests` тянет только xUnit. Нет `WebApplicationFactory`, нет прогрева EF-модели, тысяча тестов идёт меньше секунды.

Единственное, что домен всё-таки использует, — **BCL**: `Guid`, `DateTimeOffset`, `decimal`, `TimeProvider`, `List<T>`. Это часть рантайма, а не зависимость, поэтому правило не нарушается.

> [!tip] `TreatWarningsAsErrors` и `InternalsVisibleTo`
> В `Directory.Build.props` включите `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` и `<EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>`. Классы `Infrastructure` делайте `internal` — тогда `WebApi` не сможет случайно взять `EfOrderRepository` напрямую, только через `AddInfrastructure()`. Для интеграционных тестов откройте доступ через `<InternalsVisibleTo Include="Infrastructure.IntegrationTests" />`. Подробнее про механику файла проекта — [[Файл проекта .csproj изнутри]].

---

## Где размещать интерфейсы

Ошибка джуна — сложить все интерфейсы в отдельный проект `Contracts` или `Abstractions`. Это возвращает нас к обычной слоистой архитектуре ([[Слоистая архитектура]]): интерфейс перестаёт принадлежать потребителю и становится «общим», а значит начинает обслуживать реализацию.

Правило: **интерфейс живёт в том слое, который его вызывает**.

| Интерфейс | Объявлен | Реализован | Комментарий |
|---|---|---|---|
| `IOrderRepository` | Application | Infrastructure | Метод `GetByIdAsync` — потому что так нужно сценарию, а не потому что так удобно EF |
| `IEmailSender` | Application | Infrastructure | Порт наружу. Реализация — SendGrid, SMTP, заглушка в тестах |
| `IUnitOfWork` | Application | Infrastructure | Один метод `SaveChangesAsync` — транзакционная граница сценария |
| `ICurrentUser` | Application | WebApi | Реализация читает `HttpContext`, поэтому живёт там, где `HttpContext` есть |
| `IOrderReadStore` | Application | Infrastructure | Отдельный порт для чтения — см. [[CQRS]] |
| ~~`IDateTimeProvider`~~ | — | — | В .NET 10 не нужен, см. ниже |

### `TimeProvider` вместо своего `IDateTimeProvider`

Годами в каждом проекте писали одно и то же:

```csharp
// #устарело — так делали до .NET 8
public interface IDateTimeProvider { DateTime UtcNow { get; } }
public sealed class SystemDateTimeProvider : IDateTimeProvider
{
    public DateTime UtcNow => DateTime.UtcNow;
}
```

Начиная с .NET 8 в BCL есть абстрактный класс `TimeProvider`, и в .NET 10 он — правильный дефолт. Причины, почему свой интерфейс теперь хуже:

1. **`TimeProvider` лежит в BCL**, значит его может использовать даже `Domain` без единого пакета. Свой `IDateTimeProvider` пришлось бы объявлять в `Application`, и домен не смог бы его принять.
2. **Он умеет больше, чем «сейчас»**: `GetUtcNow()`, `GetLocalNow()`, `GetTimestamp()` для замеров, `CreateTimer()` и `Task.Delay(delay, timeProvider)`. Тестировать можно не только текущее время, но и таймауты с ретраями — раньше это было почти невозможно.
3. **Один общий язык для всех библиотек.** `Polly`, `Microsoft.Extensions.Http.Resilience`, `Channels` принимают `TimeProvider`. Ваш собственный интерфейс с ними не стыкуется.
4. **Есть готовая тестовая реализация** `FakeTimeProvider` в пакете `Microsoft.Extensions.TimeProvider.Testing` — умеет `Advance(TimeSpan)`, что двигает и время, и виртуальные таймеры.

```csharp
// Регистрация в точке входа
builder.Services.AddSingleton(TimeProvider.System);

// Домен принимает время параметром — самый чистый вариант
order.Cancel(clock.GetUtcNow());

// Тест
var clock = new FakeTimeProvider(new DateTimeOffset(2026, 8, 5, 10, 0, 0, TimeSpan.Zero));
var handler = new CancelOrderHandler(repo, uow, email, clock);
// ...
clock.Advance(TimeSpan.FromHours(25));   // сработают и таймеры, созданные через CreateTimer
```

Подробности API — [[Дата и время: DateTime, DateTimeOffset, TimeProvider]].

### Спорный вопрос: `IApplicationDbContext`

Очень популярный приём — вместо репозиториев отдать `Application` интерфейс над `DbContext`:

```csharp
// Application/Common/Interfaces/IApplicationDbContext.cs
public interface IApplicationDbContext
{
    DbSet<Order> Orders { get; }
    DbSet<Customer> Customers { get; }
    Task<int> SaveChangesAsync(CancellationToken ct);
}
```

Соблазн понятен: не нужно писать по репозиторию на агрегат, в хендлере доступен весь LINQ, проекции пишутся напрямую. Так делают многие шаблоны, включая известный `CleanArchitecture` от Джейсона Тейлора.

Проблемы, о которых надо знать, прежде чем это выбрать:

1. **Абстракция фиктивная.** Чтобы объявить `DbSet<Order>`, `Application.csproj` обязан сослаться на `Microsoft.EntityFrameworkCore`. Домен чист, а слой сценариев — уже нет. Архитектурный тест «Application не зависит от EF Core» после этого невозможен.
2. **Она не изолирует поведение.** `SaveChangesAsync` внутри выполняет change tracking, генерирует SQL, применяет интерцепторы и глобальные фильтры. Подменив интерфейс моком, вы не воспроизводите ничего из этого — тест проверяет, что вы вызвали метод, а не что данные сохранились.
3. **Нельзя честно замокать LINQ.** `IQueryable` над `List<T>` ведёт себя иначе, чем над PostgreSQL: другой регистр строк при сравнении, другая семантика `null`, работают методы, которые провайдер не переведёт. Тест зелёный, прод падает. Про это — [[IEnumerable vs IQueryable]] и [[EF Core: тестирование доступа к данным]].
4. **Утекает транзакционность.** `SaveChangesAsync` в интерфейсе — это уже `IUnitOfWork`, только с лишними десятью членами.

Практический вывод, который считается нормальным в 2026-м:

- Для **записи** — репозитории с намеренно узкими методами (`GetByIdAsync`, `Add`), возвращающие агрегаты, не `IQueryable`. Так интерфейс отражает нужды сценария.
- Для **чтения** — либо отдельный порт `IOrderReadStore` с методами вида `Task<OrderListItemDto[]> ListAsync(...)`, либо честное признание, что запросы идут через `DbContext` в `Infrastructure`, а `Application` их только вызывает.
- **Не мокать доступ к данным вообще.** Тестировать его интеграционно через [[Testcontainers]]. Тогда `IApplicationDbContext` теряет главную причину существования.

Полный разбор «нужен ли вообще репозиторий поверх EF Core» — в [[Repository и Unit of Work: нужны ли поверх EF Core]], там же про [[Specification pattern]].

---

## DTO и маппинг

### Почему сущности не должны утекать в API

Четыре независимые причины, каждой достаточно:

1. **Утечка данных.** Сериализовали `Customer` — отдали `PasswordHash`, `InternalNotes`, `CreditLimit`. Атрибут `[JsonIgnore]` в сущности означает, что домен теперь знает о JSON.
2. **Циклы и лишние запросы.** `Order.Customer.Orders.Customer...` — либо `JsonException`, либо N+1 при ленивой загрузке.
3. **Контракт API начинает диктовать модель.** Фронтенду нужно поле `customerName` рядом с заказом — и в сущность добавляют денормализованное поле, которого в предметной области нет.
4. **Молчаливое ломание клиентов.** Переименовали приватное свойство при рефакторинге — сломали чужой парсер JSON. DTO делает контракт явным и версионируемым ([[Версионирование API]]).

### Ручной маппинг через extension-методы

AutoMapper с 2025 года коммерческий (как и MediatR, и MassTransit v9 — см. [[MediatR и альтернативы]]), поэтому обсуждать «удобство» больше не приходится: ручной маппинг стал дефолтом. И это не потеря — он быстрее, отлаживается пошагово, а ошибку показывает компилятор, а не тест на `AssertConfigurationIsValid`.

```csharp
namespace Application.Common.Mapping;

// C# 14: extension-блок вместо набора статических методов с this
public static class OrderMappings
{
    extension(Order order)
    {
        public OrderDetailsDto ToDetailsDto() => new(
            Id: order.Id,
            Status: order.Status.ToString(),
            Total: order.Total.Amount,
            Currency: order.Total.Currency,
            Lines: order.Lines.Select(l => l.ToDto()).ToArray());
    }

    extension(OrderLine line)
    {
        public OrderLineDto ToDto() =>
            new(line.ProductId, line.ProductName, line.Quantity, line.UnitPrice.Amount);
    }
}

// DTO — record. Позиционный конструктор, значимое равенство, ничего лишнего.
public sealed record OrderDetailsDto(
    Guid Id,
    string Status,
    decimal Total,
    string Currency,
    OrderLineDto[] Lines);

public sealed record OrderLineDto(Guid ProductId, string ProductName, int Quantity, decimal UnitPrice);
```

### Проекция в DTO прямо в LINQ

Для чтения ручной маппинг из сущности — вообще лишний шаг. Проецируйте в `Select` сразу:

```csharp
// Плохо: тянем всю сущность со всеми колонками и связями, потом отбрасываем 80 %
var orders = await db.Orders
    .Include(o => o.Lines)
    .Where(o => o.CustomerId == customerId)
    .ToListAsync(ct);
return orders.Select(o => o.ToDetailsDto()).ToArray();

// Хорошо: SQL выбирает ровно нужные колонки, сущности не создаются,
// change tracker не заполняется, аллокаций меньше
var dtos = await db.Orders
    .Where(o => o.CustomerId == customerId)
    .OrderByDescending(o => o.PlacedAt)
    .Select(o => new OrderListItemDto(
        o.Id,
        o.Status.ToString(),
        o.Lines.Sum(l => l.Quantity * l.UnitPrice.Amount),   // SUM выполнит СУБД
        o.PlacedAt))
    .Take(50)
    .ToArrayAsync(ct);
```

Разница не косметическая: во втором варианте `SELECT` содержит четыре выражения вместо всех колонок двух таблиц, `Include` не порождает декартово произведение строк, а `AsNoTracking` не нужен — при проекции в не-сущностный тип отслеживания нет по определению. Механика подробно разобрана в [[EF Core: запросы и загрузка связанных данных]] и [[EF Core: change tracking и AsNoTracking]].

> [!info] Три уровня DTO — не паранойя, а следствие
> Строго говоря, в полном раскладе типов три: `CreateOrderRequest` (контракт HTTP, живёт в `WebApi`), `CreateOrderCommand` (вход сценария, живёт в `Application`), `Order` (сущность). Это выглядит избыточно, и в 90 % случаев первые два совпадают один-в-один. Практический компромисс: держать один тип-команду и принимать её прямо в эндпоинте, а разделять только там, где контракты действительно расходятся (например, `CustomerId` берётся не из тела, а из токена).

---

## Автоматическая проверка правила зависимостей

Соглашение, которое не проверяется, умирает за один спринт. Правило зависимостей проверяется тестом.

Пакет `NetArchTest.Rules` (или `ArchUnitNET`) читает метаданные сборки и отвечает на вопросы про типы и ссылки. Тесты идут миллисекунды и падают на CI до код-ревью.

```csharp
using NetArchTest.Rules;
using Xunit;

public class DependencyRuleTests
{
    private const string DomainNs = "Domain";
    private const string ApplicationNs = "Application";

    private static readonly System.Reflection.Assembly DomainAssembly = typeof(Order).Assembly;
    private static readonly System.Reflection.Assembly ApplicationAssembly = typeof(CancelOrderHandler).Assembly;

    [Fact]
    public void Domain_не_должен_зависеть_от_EF_Core()
    {
        var result = Types.InAssembly(DomainAssembly)
            .Should()
            .NotHaveDependencyOnAny(
                "Microsoft.EntityFrameworkCore",
                "Npgsql",
                "System.Data")
            .GetResult();

        Assert.True(result.IsSuccessful, Describe(result));
    }

    [Fact]
    public void Domain_не_должен_знать_ни_про_Application_ни_про_Infrastructure()
    {
        var result = Types.InAssembly(DomainAssembly)
            .Should()
            .NotHaveDependencyOnAny(ApplicationNs, "Infrastructure", "WebApi")
            .GetResult();

        Assert.True(result.IsSuccessful, Describe(result));
    }

    [Fact]
    public void Application_не_должен_знать_про_ASP_NET_Core()
    {
        var result = Types.InAssembly(ApplicationAssembly)
            .Should()
            .NotHaveDependencyOnAny("Microsoft.AspNetCore", "Infrastructure")
            .GetResult();

        Assert.True(result.IsSuccessful, Describe(result));
    }

    [Fact]
    public void Хендлеры_должны_быть_sealed_и_называться_Handler()
    {
        var result = Types.InAssembly(ApplicationAssembly)
            .That().ResideInNamespaceContaining("Orders.Commands")
            .And().HaveNameEndingWith("Handler")
            .Should().BeSealed()
            .GetResult();

        Assert.True(result.IsSuccessful, Describe(result));
    }

    private static string Describe(TestResult result) =>
        result.IsSuccessful
            ? "ok"
            : "Нарушители: " + string.Join(", ", result.FailingTypeNames ?? []);
}
```

> [!tip] Эти тесты — единственное, что позволяет упрощать структуру безопасно
> Если правило зависимостей проверяется тестом, дробить проекты становится не обязательно: можно держать `Core` одной сборкой с папками и проверять ограничения по неймспейсам через `ResideInNamespace`. Именно так работает прагматичный компромисс из конца заметки.

---

## Честная критика: сколько это стоит

Раздел, которого нет в туториалах. Мидл — это тот, кто может назвать цену.

### Стоимость одного нового поля

Задача из жизни: «добавьте к заказу комментарий покупателя». Одна строка `string? Comment`. Считаем файлы.

| № | Файл | Что делаем |
|---|---|---|
| 1 | `Domain/Entities/Order.cs` | свойство + параметр в фабричном методе |
| 2 | `Infrastructure/Persistence/Configurations/OrderConfiguration.cs` | `HasMaxLength(500)` |
| 3 | `Infrastructure/Migrations/2026..._AddOrderComment.cs` | сгенерированная миграция + snapshot |
| 4 | `Application/Orders/Commands/CreateOrder/CreateOrderCommand.cs` | поле в record |
| 5 | `Application/Orders/Commands/CreateOrder/CreateOrderHandler.cs` | передать в домен |
| 6 | `Application/Orders/Commands/CreateOrder/CreateOrderValidator.cs` | правило длины |
| 7 | `Application/Orders/Queries/GetOrderById/OrderDetailsDto.cs` | поле в DTO |
| 8 | `Application/Common/Mapping/OrderMappings.cs` или `Select` в хендлере | маппинг |
| 9 | `WebApi/Contracts/CreateOrderRequest.cs` (+ `OrderResponse.cs`) | контракт API |
| 10 | `WebApi/Endpoints/OrderEndpoints.cs` | проброс request → command |
| 11 | `Application.UnitTests/CreateOrderHandlerTests.cs` | обновить билдеры и ассерты |
| 12 | `WebApi.FunctionalTests/CreateOrderTests.cs` | обновить payload |

**10–12 файлов и примерно 40 минут** с прогоном тестов. Для сравнения, в подходе «контроллер + EF Core напрямую»: свойство в сущности, миграция, поле в DTO-record — **2–3 файла, 5 минут**.

Это не аргумент против Clean Architecture. Это цена. Умножьте на количество полей, которые бизнес добавит за год, и сравните с выгодой, которую вы за этот же год получите от изолированного домена. Если домен — это `{ get; set; }` без единого правила, выгоды нет, а цена есть.

### Потеря навигируемости

Чтобы прочитать сценарий «создать заказ», вы открываете: эндпоинт → команду → валидатор → хендлер → интерфейс репозитория → реализацию репозитория → `DbContext` → конфигурацию сущности → сущность. Девять файлов в четырёх проектах. «Перейти к реализации» на интерфейсе с одной реализацией — это лишний Ctrl+F12 каждый раз.

Именно из этой боли выросла [[Vertical Slice Architecture]]: сложить всё, что относится к одному сценарию, в **один файл**, а связность обеспечивать по фиче, а не по техническому слою.

### «Application из хендлеров-однострочников»

Через год живого проекта половина `Application` выглядит так:

```csharp
// Три файла, шестнадцать строк, ноль логики.
// Всё, что здесь есть — это вызов метода EF Core с лишним слоем косвенности.
public sealed record GetCustomerByIdQuery(Guid Id);

public sealed class GetCustomerByIdHandler(IApplicationDbContext db)
{
    public Task<CustomerDto?> HandleAsync(GetCustomerByIdQuery q, CancellationToken ct) =>
        db.Customers
          .Where(c => c.Id == q.Id)
          .Select(c => new CustomerDto(c.Id, c.Name, c.Email))
          .FirstOrDefaultAsync(ct);
}
```

Слой не добавил ни правила, ни защиты, ни возможности замены — только файл и прыжок при чтении. Простые запросы честнее звать прямо из эндпоинта. Смешанный подход (сложное — через сценарии, справочники — напрямую) выглядит непоследовательно, зато дешевле; последовательность ради последовательности — это [[DRY, KISS, YAGNI и когда они врут]] в чистом виде.

### Интерфейс на каждый сервис ради тестов, которые всё равно интеграционные

Классический самообман. Пишем `IOrderRepository`, мокаем в юнит-тесте хендлера, тест зелёный. Что он проверил? Что хендлер вызвал `GetByIdAsync` и `SaveChangesAsync`. Он не поймает: неправильный `Include`, отсутствующий индекс, нарушение уникальности, неверный маппинг value object, ошибку в глобальном фильтре, дедлок при параллельной записи. Всё это ловится только интеграционным тестом на реальной СУБД.

Вывод: интерфейс над хранилищем оправдан **не для тестов**, а для того, чтобы хендлер говорил на языке домена. Если у вас единственный аргумент «чтобы мокать» — вы платите за иллюзию. См. [[Пирамида тестирования и что чем покрывать]] и [[Моки: NSubstitute и Moq]].

### Протекающие абстракции

Самая тонкая проблема. Абстракция, которая обещает независимость и не даёт её, хуже отсутствия абстракции — потому что создаёт ложную уверенность.

- **`IQueryable` в репозитории.** `IQueryable<Order> Query()` — это уже EF Core в подписи: семантика отложенного выполнения, набор переводимых методов, поведение `null`. Смените провайдер — половина вызовов упадёт в рантайме. Про механику — [[IEnumerable vs IQueryable]].
- **Спецификации.** [[Specification pattern]] в варианте с `Expression<Func<T, bool>>` и списком `Includes` — это тонкая обёртка над EF Core, копирующая его модель. Она добавляет свой язык поверх LINQ, но не устраняет привязку.
- **`SaveChangesAsync` в `IUnitOfWork`.** Обещает «транзакцию», а даёт change tracking EF Core с его порядком операций, каскадами и интерцепторами. Другая реализация (Dapper) с той же сигнатурой поведёт себя по-другому.
- **`IApplicationDbContext`.** Разобрано выше: тянет пакет EF Core в `Application` и ничего не изолирует.
- **Транзакции вообще.** Уровень изоляции, поведение при конфликте, семантика повторной попытки — всё это специфика СУБД, и абстракция её не скрывает ([[Транзакции и уровни изоляции]], [[EF Core: транзакции и конкурентность]]).

Честная позиция: считать EF Core **частью доменного слоя по факту** и не притворяться, что его можно вынуть. Тогда репозиторий пишется не «на случай смены ORM», а ради узкого API для сценариев — и это уже нормальная, оплаченная причина.

### Шаблоны с шестью проектами на десять эндпоинтов

Отдельный жанр: сервис из десяти CRUD-эндпоинтов, `Domain` / `Application` / `Application.Contracts` / `Infrastructure` / `Infrastructure.Shared` / `WebApi`, шесть `.csproj`, `Directory.Packages.props`, `DependencyInjection.cs` в каждом. Плата:

- сборка вместо 3 секунд идёт 25;
- инкрементальный билд перебирает шесть проектов;
- любой рефакторинг сущности — это правка в четырёх проектах;
- «где это лежит» перестаёт быть очевидным, и появляется `Common`, `Shared`, `Helpers` — свалки, куда падает всё несортированное.

Соотношение простое: количество проектов должно расти от **количества настоящих границ**, а не от количества слов в диаграмме.

### Цена онбординга

Новый разработчик на проекте с обычным контроллером и EF Core делает первую задачу за день. На проекте с полным Clean Architecture — за три-четыре, и первые две недели задаёт вопросы «а зачем здесь ещё один DTO» и «почему валидатор не рядом с эндпоинтом». За год с ротацией из четырёх человек это ощутимая сумма. Она окупается, если проект живёт пять лет; она не окупается, если сервис через год выключат.

> [!danger] Отдельно: культ карго
> Худший вариант — Clean Architecture без домена. Четыре проекта, `IRepository<T>`, сорок интерфейсов, а внутри сущностей `{ get; set; }` и вся логика в хендлерах. Это анемичная модель ([[DDD: тактические паттерны]]) в дорогой упаковке: заплачено за структуру, не получено главное. Такой код сложнее менять, чем честный однослойный CRUD, потому что цена есть, а выгоды нет.

---

## Когда Clean Architecture оправдана

Каждый пункт — про **выгоду**, которая перекрывает посчитанную выше цену.

- **В домене есть настоящие правила.** Не валидация «поле обязательно», а расчёт лимитов, тарифы, начисления, статусные машины со запретами, правила, которые формулирует не разработчик, а предметный эксперт. Если правила есть, их изоляция даёт мгновенно тестируемое ядро — это главная и часто единственная реальная выгода.
- **Система живёт 5+ лет.** Стоимость структуры платится один раз, стоимость запутанности — каждый спринт. На длинной дистанции второе больше.
- **Команда 5+ человек с разделением зон.** Явные границы сборок превращают договорённости в ошибки компиляции. Ревью «ты потащил EF в домен» делает CI, а не человек.
- **Регуляторные требования к тестируемости бизнес-правил.** Финтех, медицина, страхование: нужно доказать аудитору, что правило начисления процентов покрыто тестами. Тест на чистый домен — доказательство; тест через HTTP и базу — нет.
- **Реальная вероятность смены инфраструктуры.** Именно реальная: не «а вдруг уйдём с PostgreSQL», а «уведомления сейчас через SMS-шлюз А, через полгода тендер, будет Б» или «часть чтения переезжает в Elasticsearch». Порты `IEmailSender`, `ISmsGateway`, `IOrderReadStore` окупаются сразу.
- **Несколько точек входа.** HTTP API, gRPC, консольные утилиты миграции, фоновые обработчики очередей — все вызывают одни и те же сценарии. Слой `Application` перестаёт быть церемонией и становится настоящим переиспользованием.

### Когда точно не нужна

- **CRUD-сервис-справочник.** Читает и пишет таблицы, правил нет. Минимальный API + EF Core + проекции в record. Всё.
- **Прототип, MVP, хакатон, вещь на выброс.** Оптимизируйте скорость изменений, а не долгосрочную структуру.
- **Один разработчик и десять эндпоинтов.** Границы между сборками нужны, чтобы договариваться. Договариваться не с кем.
- **Интеграционная прослойка.** Читает из очереди, дёргает три HTTP API, пишет в базу. Домена нет, есть маппинг и устойчивость — важнее [[Устойчивость: retry, circuit breaker, Polly]] и [[Идемпотентность]], а не слои.
- **Отчёты и аналитика.** Тут доминирует SQL. Слои поверх сложных запросов только мешают; правильный ответ — [[Dapper: когда микро-ORM лучше]].

---

## Прагматичный компромисс: два проекта вместо шести

Вариант, который в 2026-м стоит брать по умолчанию для нового сервиса среднего размера. Правило зависимостей сохраняется, цена — вдвое ниже.

```
OrderService.sln
├── src/
│   ├── Core/                       ← Domain + Application в ОДНОЙ сборке
│   │   ├── Core.csproj             только FluentValidation и Abstractions
│   │   ├── Domain/
│   │   │   ├── Entities/
│   │   │   ├── ValueObjects/
│   │   │   └── Events/
│   │   └── Features/               ← вертикальные срезы, а не техслои
│   │       ├── Orders/
│   │       │   ├── CreateOrder.cs        команда + валидатор + хендлер в ОДНОМ файле
│   │       │   ├── CancelOrder.cs
│   │       │   ├── GetOrderById.cs
│   │       │   └── IOrderRepository.cs
│   │       └── Customers/
│   └── Web/                        ← Infrastructure + WebApi
│       ├── Web.csproj              EF Core, Npgsql, ASP.NET Core
│       ├── Program.cs
│       ├── Persistence/
│       ├── Endpoints/
│       └── Migrations/
└── tests/
    ├── Core.UnitTests/
    ├── Web.IntegrationTests/
    └── ArchitectureTests/          ← правило зависимостей внутри Core как тест
```

Что удерживает правило зависимостей без физической границы:

```csharp
[Fact]
public void Domain_не_должен_знать_про_Features()
{
    var result = Types.InAssembly(typeof(Order).Assembly)
        .That().ResideInNamespace("Core.Domain")
        .Should().NotHaveDependencyOnAny("Core.Features", "Microsoft.EntityFrameworkCore")
        .GetResult();

    Assert.True(result.IsSuccessful);
}

[Fact]
public void Features_не_должны_знать_про_ASP_NET_Core_и_EF_Core()
{
    var result = Types.InAssembly(typeof(CreateOrder).Assembly)
        .That().ResideInNamespace("Core.Features")
        .Should().NotHaveDependencyOnAny("Microsoft.AspNetCore", "Microsoft.EntityFrameworkCore")
        .GetResult();

    Assert.True(result.IsSuccessful);
}
```

Что получили: изолированный тестируемый домен, порты для внешних систем, дисциплина под контролем CI. Что не платим: шесть `.csproj`, шесть `DependencyInjection.cs`, долгий билд, прыжки между проектами. Если проект вырастет — разрезать `Core` на `Domain` и `Application` можно за час, потому что неймспейсы уже разделены и тесты это гарантируют.

Как выбирать между этим, полным вариантом, [[Vertical Slice Architecture]] и просто [[Слоистая архитектура]] — в [[Как выбрать архитектуру под задачу]]. Решение стоит зафиксировать письменно, см. [[Документация: ADR и RFC]].

---

> [!warning] Подводные камни
> **1. Слои есть, правила нет.** Четыре проекта созданы, а в `Application` лежит `using Microsoft.EntityFrameworkCore` и хендлеры с `Include`. Проверяется одной командой: посмотрите `PackageReference` в `Application.csproj`. Если там EF Core — это слоистая архитектура с красивыми именами. Лечится архитектурными тестами, а не разговорами.
>
> **2. Анемичный домен в дорогой упаковке.** Сущности из `{ get; set; }`, вся логика в хендлерах. Домен превратился в набор DTO, а `Application` — в процедурный код над таблицами. Заплачена вся цена структуры, не получена главная выгода — изолированные правила. Признак: в `Domain` нет ни одного метода, только свойства.
>
> **3. `IRepository<T>` с двадцатью методами и `IQueryable`.** Генерик-репозиторий возвращает `IQueryable<T>`, принимает `Expression<Func<T,bool>>`, имеет `Include`-параметры. Это EF Core, переименованный дважды, с потерей половины возможностей. Причина: интерфейс писали от реализации, а не от потребности сценария. Правильно — узкий интерфейс на агрегат с методами, названными на языке домена.
>
> **4. Захват `IServiceProvider` внутрь Application через Service Locator.** Хендлер получает `IServiceProvider` и достаёт зависимости вручную, чтобы «не раздувать конструктор». Зависимости перестали быть видны в подписи, отсутствие регистрации ловится только в рантайме, а `scope` легко утечь. Симптом лечится разбиением сценария, а не локатором.
>
> **5. Домен зависит от `Microsoft.Extensions.*` «чуть-чуть».** Сначала `ILogger` в сущность, потом `IOptions`, потом `IMemoryCache`. Каждый шаг кажется безобидным, но домен перестаёт компилироваться отдельно, а тест начинает требовать инфраструктуру. Логирование — дело `Application` и декораторов; конфигурация приходит в домен параметром, а не через `IOptions`.
>
> **6. Транзакция размазана по слоям.** `SaveChangesAsync` вызывается и в репозитории, и в хендлере, и в декораторе. Частичные записи, потерянные доменные события, конфликты. Правило: транзакционная граница — это ровно один сценарий, и коммит делает ровно одно место (хендлер или транзакционный декоратор), а репозиторий только `Add`/`Remove`.
>
> **7. Domain-события отправляются до коммита.** `order.Cancel()` публикует событие, обработчик отправляет письмо, потом `SaveChangesAsync` падает по конкурентности. Письмо ушло, заказ не отменён. Решается сбором событий в сущности и публикацией после успешного коммита — [[Доменные события]], а для внешних систем [[Паттерн Transactional Outbox]].
>
> **8. Автомаппинг «по соглашению» между DTO и сущностью.** Даже без AutoMapper соблазн остаётся (рефлексия, `Adapt`). Переименовали свойство — маппинг молча перестал его копировать, поле уехало в базу как `null`. Ручной маппинг ловит это компилятором. Исходное правило: маппинг должен ломаться на сборке, а не в проде.

> [!example] Как делают в бою
> **Что реально встречается в живых .NET-проектах 2026 года:**
>
> - **Полный четырёхпроектный расклад** — в финтехе, страховании, биллинге, ERP: там, где домен настоящий, а срок жизни системы измеряется годами. Обычно с [[CQRS]] уровня 3, узкими репозиториями для записи и Dapper/проекциями для чтения.
> - **Два проекта `Core` + `Web`** — самый частый выбор для нового сервиса в продуктовой команде. Дисциплина держится архитектурными тестами.
> - **Vertical slices внутри одного проекта** — для сервисов с большим числом простых сценариев и одним-двумя сложными.
> - **Гибрид «CRUD напрямую, сложное через сценарии»** — сознательное решение, зафиксированное в ADR: справочники читаются из эндпоинта запросом EF Core, а платежи и заказы идут через полный слой сценариев. Непоследовательно и дешевле; спорить с этим можно только измеренными числами.
>
> **Что почти всегда есть на зрелых проектах вне зависимости от расклада:**
> - `Directory.Packages.props` с Central Package Management — версии пакетов в одном месте;
> - `TreatWarningsAsErrors` плюс `.editorconfig` с включённым `EnforceCodeStyleInBuild`;
> - архитектурные тесты в CI как отдельный быстрый шаг перед юнит-тестами;
> - интеграционные тесты на реальной СУБД через [[Testcontainers]], а не моки `DbContext`;
> - `internal` для всего в `Infrastructure` плюс `InternalsVisibleTo` для тестов;
> - `TimeProvider` вместо своего провайдера времени, `FakeTimeProvider` в тестах.
>
> **Чего в бою почти не бывает, хотя туториалы обещают:** смены СУБД «благодаря архитектуре». За десять лет проекты меняют ORM и базу единицы раз, и когда это происходит, слои помогают меньше, чем ожидалось: течёт SQL, миграции, специфика типов и транзакций. Настоящая окупаемость — тестируемость домена и понятность кода, а не переносимость.

---

## Вопросы с собеседований

> [!question]- В чём разница между Clean Architecture, Onion и Hexagonal?
> Ядро у всех одно: правило зависимостей — зависимости направлены только внутрь, к бизнес-логике, взаимодействие с внешним миром через абстракции, принадлежащие внутреннему слою. Отличаются акценты и словарь. Hexagonal (Кокбёрн, 2005) говорит о портах и адаптерах и подчёркивает симметрию: HTTP и база данных — равноправные адаптеры, а тест — тоже адаптер, слоёв как таковых нет. Onion (Палермо, 2008) первым нарисовал концентрические слои и разделил модель домена, доменные сервисы и сервисы приложения — именно оттуда пошёл знакомый расклад проектов. Clean (Мартин, 2012) дал самую чёткую формулировку Dependency Rule и добавил явные use case'ы как объекты плюс идею простых DTO на границах. Практически это одно и то же, и если коллега называет ваш `Domain/Application/Infrastructure` гексагональным — он не ошибается.

> [!question]- Где объявлять интерфейс репозитория и почему именно там?
> В том слое, который его вызывает, — в `Application` (в строгих вариантах DDD — в `Domain`). Причина в инверсии зависимостей: если интерфейс лежит в `Infrastructure` или в отдельном `Contracts`, то `Application` ссылается наружу, и правило зависимостей нарушено. Когда интерфейс объявлен у потребителя, он формулируется на языке потребности сценария (`GetByIdAsync`, `Add`), а не на языке хранилища (`Find`, `Query`, `IQueryable`). Побочный эффект правильного размещения — интерфейсы получаются узкими: у сценария три потребности, значит три метода, а не двадцать «на всякий случай». Отдельный проект `Contracts` — типичная ошибка: он возвращает обычную слоистую архитектуру, где абстракция принадлежит всем и потому обслуживает реализацию.

> [!question]- Почему Domain-проект не должен ссылаться на пакеты? Что будет, если добавить туда EF Core?
> Три следствия. Первое: компилятор перестаёт быть сторожем — теперь в сущность можно поставить `[Column]`, `virtual`-навигацию и публичные сеттеры «чтобы EF было удобно», и модель начнёт описывать таблицу, а не предметную область. Второе: домен наследует версии и транзитивные зависимости, и обновление EF Core становится потенциально ломающим изменением для ядра. Третье: тесты домена перестают быть мгновенными — тянется вся инфраструктура. Формально код продолжит работать; проблема не в компиляции, а в том, что исчезает механизм, который удерживает правило зависимостей автоматически. BCL-типы (`Guid`, `decimal`, `DateTimeOffset`, `TimeProvider`) зависимостью не считаются — они часть рантайма.

> [!question]- Как проверить, что правило зависимостей не нарушено, кроме код-ревью?
> Архитектурными тестами: `NetArchTest.Rules` или `ArchUnitNET`. Они читают метаданные сборки и утверждают факты про типы и ссылки: `Types.InAssembly(domainAssembly).Should().NotHaveDependencyOnAny("Microsoft.EntityFrameworkCore", "Npgsql").GetResult().IsSuccessful`. Такие тесты идут миллисекунды, падают на CI до ревью и работают по неймспейсам, а не только по сборкам — поэтому позволяют держать правило даже внутри одного проекта. Дополнительно правило страхуется структурой: отсутствие `PackageReference` в `Domain.csproj` делает нарушение ошибкой компиляции, а `internal` в `Infrastructure` не даёт обойти `AddInfrastructure()`. Ключевая мысль: соглашение, которое не проверяется автоматически, разваливается за один-два спринта.

> [!question]- В чём проблема с `IApplicationDbContext`? Это же абстракция.
> Формально абстракция, практически — нет. Чтобы объявить в интерфейсе `DbSet<Order>`, проект `Application` обязан сослаться на `Microsoft.EntityFrameworkCore`, то есть зависимость от ORM просто переехала на слой выше и стала невидимой. Она ничего не изолирует: `SaveChangesAsync` выполняет change tracking, генерацию SQL, интерцепторы и глобальные фильтры, и подмена интерфейса моком не воспроизводит ни одного из этих эффектов — тест проверяет факт вызова, а не результат. Плюс `IQueryable` над `List<T>` в тестах ведёт себя иначе, чем над PostgreSQL: другая семантика `null` и сравнения строк, работают методы, которые провайдер не переведёт, — тест зелёный, прод падает. Разумный вывод: для записи — узкие репозитории на агрегат, для чтения — отдельный порт или прямые запросы в `Infrastructure`, а доступ к данным тестировать интеграционно на реальной СУБД.

> [!question]- Сколько файлов нужно тронуть, чтобы добавить одно поле, и что вы с этим делаете?
> В полном четырёхпроектном раскладе — 10–12: сущность, конфигурация EF, миграция со снапшотом, команда, хендлер, валидатор, DTO, маппинг, контракт API, эндпоинт, юнит-тест, функциональный тест. В подходе «эндпоинт плюс EF Core» — 2–3. Это самая честная метрика цены архитектуры, и её стоит назвать до того, как её назовут вам. Что с этим делают: сокращают число слоёв там, где нет выгоды (простые запросы — прямо из эндпоинта), объединяют контракт API и команду в один тип, схлопывают `Domain` и `Application` в один проект `Core` с папками и архитектурными тестами, переходят к вертикальным срезам, чтобы файлы одного сценария лежали рядом. Что не делают — не отменяют изоляцию домена там, где в домене есть настоящие правила: именно она и оплачивает всю остальную церемонию.

> [!question]- Когда вы откажетесь от Clean Architecture и что предложите вместо?
> Откажусь, когда домена как такового нет: CRUD-справочник, интеграционная прослойка между тремя API, отчётный сервис, прототип на выброс, проект одного разработчика на десять эндпоинтов. Признак простой — в сущностях нет ни одного метода с правилом, только свойства; тогда изоляция домена изолирует пустоту, а цена в 10 файлов на поле остаётся. Предложу минимальный API плюс EF Core с проекциями в record прямо в эндпоинте, для отчётов — Dapper, для интеграций — упор на идемпотентность и устойчивость вместо слоёв. Промежуточный вариант, который беру по умолчанию для нового среднего сервиса: два проекта `Core` и `Web`, внутри `Core` — папки `Domain` и `Features` с вертикальными срезами, правило зависимостей держится архитектурными тестами по неймспейсам. Разрезать `Core` на две сборки потом — час работы, а сэкономленное на старте время реально.

> [!question]- Что такое протекающая абстракция и какие из них есть в типичном Clean-проекте?
> Протекающая абстракция обещает скрыть детали реализации, но её поведение и подпись всё равно этими деталями определяются. В типичном .NET-проекте таких минимум четыре. `IQueryable` в репозитории — это уже EF Core в контракте: отложенное выполнение, ограниченный набор переводимых методов, специфичная семантика `null` и сравнения строк. Спецификации с `Expression<Func<T,bool>>` и списком `Include` — тонкая обёртка над той же моделью EF Core, просто со своим синтаксисом. `IUnitOfWork.SaveChangesAsync` обещает транзакцию, а даёт change tracking с его порядком операций и каскадами; Dapper-реализация с той же подписью поведёт себя иначе. `IApplicationDbContext` тянет пакет EF Core в слой сценариев. Опасность не в самих абстракциях, а в ложной уверенности «мы сможем поменять ORM». Честнее считать EF Core частью домена по факту и строить репозиторий ради узкого API для сценариев, а не ради мифической переносимости.

---

## Задачи

### Задача 1. Найти и починить нарушение правила зависимостей

Ниже фрагмент из проекта `Application`. Найдите все нарушения правила зависимостей и перепишите так, чтобы `Application.csproj` не нуждался ни в `Microsoft.EntityFrameworkCore`, ни в `Microsoft.AspNetCore.*`.

```csharp
using Microsoft.AspNetCore.Http;
using Microsoft.EntityFrameworkCore;

namespace Application.Orders.Commands;

public sealed class ArchiveOrderHandler(AppDbContext db, IHttpContextAccessor http)
{
    public async Task<IResult> HandleAsync(Guid orderId, CancellationToken ct)
    {
        var userId = http.HttpContext!.User.FindFirst("sub")!.Value;
        var order = await db.Orders.FirstOrDefaultAsync(o => o.Id == orderId, ct);
        if (order is null) return Results.NotFound();

        order.Archive(DateTime.UtcNow, Guid.Parse(userId));
        await db.SaveChangesAsync(ct);
        return Results.NoContent();
    }
}
```

> [!success]- Решение
> Нарушений четыре: конкретный `AppDbContext` из `Infrastructure`, `IHttpContextAccessor` и `IResult`/`Results` из ASP.NET Core, а также прямое обращение к `DateTime.UtcNow` (не нарушение слоя, но неустранимая зависимость от системных часов, из-за которой сценарий нельзя протестировать детерминированно).
>
> ```csharp
> // ---- Application/Common/Interfaces/ICurrentUser.cs ----
> public interface ICurrentUser
> {
>     Guid? UserId { get; }
> }
>
> // ---- Application/Common/Interfaces/IOrderRepository.cs ----
> public interface IOrderRepository
> {
>     Task<Order?> GetByIdAsync(Guid id, CancellationToken ct);
> }
>
> // ---- Application/Orders/Commands/ArchiveOrder/ArchiveOrderHandler.cs ----
> // Ни одного using из EF Core или ASP.NET Core.
> public sealed record ArchiveOrderCommand(Guid OrderId);
>
> public sealed class ArchiveOrderHandler(
>     IOrderRepository orders,
>     IUnitOfWork uow,
>     ICurrentUser currentUser,
>     TimeProvider clock)
> {
>     public async Task<Result> HandleAsync(ArchiveOrderCommand command, CancellationToken ct)
>     {
>         if (currentUser.UserId is not { } userId)
>             return Result.Unauthorized();
>
>         var order = await orders.GetByIdAsync(command.OrderId, ct);
>         if (order is null)
>             return Result.NotFound($"Заказ {command.OrderId} не найден.");
>
>         order.Archive(clock.GetUtcNow(), userId);   // правило внутри домена
>         await uow.SaveChangesAsync(ct);
>         return Result.Success();
>     }
> }
>
> // ---- WebApi/Endpoints/OrderEndpoints.cs ----
> // Перевод Result → HTTP живёт здесь, потому что HTTP — деталь ввода-вывода.
> group.MapPost("/{id:guid}/archive", async (
>     Guid id, ArchiveOrderHandler handler, CancellationToken ct) =>
> {
>     var result = await handler.HandleAsync(new ArchiveOrderCommand(id), ct);
>     return result.ToHttpResult();
> });
>
> // ---- WebApi/Services/HttpCurrentUser.cs ----
> // Реализация ICurrentUser живёт там, где есть HttpContext.
> internal sealed class HttpCurrentUser(IHttpContextAccessor accessor) : ICurrentUser
> {
>     public Guid? UserId =>
>         Guid.TryParse(accessor.HttpContext?.User.FindFirst("sub")?.Value, out var id)
>             ? id
>             : null;
> }
> ```
>
> Что изменилось по существу: сценарий стал тестируемым без базы и без HTTP — три подставленных интерфейса и `FakeTimeProvider`. Возврат `Result` вместо `IResult` убрал зависимость от веба; про этот приём — [[Result pattern вместо исключений]], про перевод в HTTP-ответ — [[Обработка ошибок и ProblemDetails]].

### Задача 2. Архитектурный тест, который ловит нарушения заранее

Напишите тесты, которые упадут на: (а) любом типе в `Domain`, зависящем от EF Core или Npgsql; (б) любом публичном классе в `Infrastructure` (всё должно быть `internal`, кроме `DependencyInjection`); (в) любом типе в `Application`, зависящем от `Microsoft.AspNetCore`.

> [!success]- Решение
> ```csharp
> using NetArchTest.Rules;
> using Xunit;
>
> public class ArchitectureTests
> {
>     private static readonly Assembly Domain = typeof(Order).Assembly;
>     private static readonly Assembly Application = typeof(CreateOrderHandler).Assembly;
>     private static readonly Assembly Infrastructure = typeof(AppDbContext).Assembly;
>
>     [Fact]
>     public void Domain_чист_от_инфраструктуры()
>     {
>         var result = Types.InAssembly(Domain)
>             .Should()
>             .NotHaveDependencyOnAny("Microsoft.EntityFrameworkCore", "Npgsql",
>                                     "Microsoft.AspNetCore", "System.Data.Common")
>             .GetResult();
>
>         Assert.True(result.IsSuccessful,
>             "Нарушители: " + string.Join(", ", result.FailingTypeNames ?? []));
>     }
>
>     [Fact]
>     public void Application_не_знает_про_веб()
>     {
>         var result = Types.InAssembly(Application)
>             .Should().NotHaveDependencyOn("Microsoft.AspNetCore")
>             .GetResult();
>
>         Assert.True(result.IsSuccessful,
>             "Нарушители: " + string.Join(", ", result.FailingTypeNames ?? []));
>     }
>
>     [Fact]
>     public void Реализации_Infrastructure_скрыты()
>     {
>         var result = Types.InAssembly(Infrastructure)
>             .That().AreClasses()
>             .And().DoNotHaveName("DependencyInjection")
>             .And().DoNotResideInNamespaceContaining("Migrations")
>             .Should().NotBePublic()
>             .GetResult();
>
>         Assert.True(result.IsSuccessful,
>             "Публичные типы: " + string.Join(", ", result.FailingTypeNames ?? []));
>     }
> }
> ```
>
> Тонкость с миграциями: сгенерированные EF Core классы миграций публичные, и переписывать их вручную бессмысленно — поэтому неймспейс `Migrations` исключается из проверки. Такие тесты стоит держать отдельным проектом и запускать первым шагом CI: они выполняются за миллисекунды и дают самый дешёвый сигнал о размывании границ.

### Задача 3. Посчитать цену изменения и предложить упрощение

Сервис на полном четырёхпроектном раскладе. За квартал бизнес добавил 14 новых полей в 5 сущностей и 3 новых сценария. Оцените трудозатраты по формуле из заметки и предложите два конкретных упрощения, которые снизят цену без потери правила зависимостей.

> [!success]- Решение
> **Оценка.** Одно поле — 10–12 файлов, примерно 40 минут с тестами и ревью. 14 полей — около 9–10 часов чистой работы плюс 14 миграций (каждая — отдельный деплой-шаг с риском). Три сценария по полному расклад — команда, валидатор, хендлер, DTO, эндпоинт, два теста, регистрация в DI, то есть 8–9 файлов и 3–4 часа каждый, итого около 11 часов. Суммарно порядка 20 человеко-часов, из которых механической церемонией (маппинги, дублирование контрактов, регистрации) занято примерно 40 %.
>
> **Упрощение 1: убрать дублирование контракта API и команды.** Принимать `CreateOrderCommand` прямо в минимальном эндпоинте, а отдельный `CreateOrderRequest` вводить только там, где формы действительно расходятся (например, `CustomerId` берётся из токена, а не из тела). Экономия: 2 файла и 2 маппинга на каждое поле в командных сценариях. Правило зависимостей не страдает — команда живёт в `Application`, а `WebApi` на него и так ссылается.
>
> **Упрощение 2: простые запросы — без слоя сценариев.** Справочные `GetById` и `List` без правил читать проекцией EF Core прямо в эндпоинте или в тонком `IOrderReadStore` в `Infrastructure`. Экономия: 3 файла и один прыжок при чтении на каждый запрос. Правило зависимостей сохраняется: чтение — это выход наружу, `Application` при этом не обрастает знанием об EF Core, а домен не участвует вообще, потому что в проекции нет инвариантов.
>
> **Что зафиксировать.** Оба решения непоследовательны по отношению к «канону», поэтому их нужно записать в ADR ([[Документация: ADR и RFC]]) с критерием: сценарии с бизнес-правилами и транзакцией идут полным путём, чтение без правил — коротким. Без письменного критерия через полгода в кодовой базе будут оба стиля вперемешку и без причины.
>
> **Чего делать не надо.** Не снимать изоляцию домена, если правила есть: 14 полей — это симптом активной эволюции модели, и именно она делает тестируемое ядро ценным. Про то, как говорить об этой цене с бизнесом, — [[Технический долг: как говорить о нём с бизнесом]].

---

## Итог

- Clean Architecture — это **одно правило**: зависимости направлены только внутрь. Всё остальное (проекты, папки, репозитории, медиаторы) — способы его соблюсти, взаимозаменяемые и небесплатные.
- Hexagonal, Onion и Clean — одна идея в трёх словарях. Спорить о названиях смысла нет; проверять надо `using`-и во внутренних слоях.
- Механизм — инверсия зависимостей: интерфейс объявлен там, где потребляется (`Application`), реализован снаружи (`Infrastructure`), склеен в точке входа. Именно поэтому `Domain.csproj` может не иметь ни одного пакета — и должен.
- В .NET 10 свой `IDateTimeProvider` больше не нужен: `TimeProvider` лежит в BCL, доступен домену без зависимостей, умеет таймеры и имеет готовый `FakeTimeProvider` для тестов.
- Абстракции над данными протекают: `IQueryable`, спецификации, `IUnitOfWork`, `IApplicationDbContext` тянут за собой семантику EF Core. Строить их «на случай смены ORM» — самообман; строить ради узкого API для сценариев — нормально.
- Цена измерима: **10–12 файлов на одно новое поле** против 2–3 в прямом подходе, плюс медленный билд, потеря навигируемости и три-четыре дня онбординга вместо одного.
- Оправдана при настоящем домене, сроке жизни 5+ лет, команде от пяти человек, регуляторных требованиях к тестируемости правил и реальной вероятности замены внешних систем. Не оправдана для CRUD, интеграционных прослоек, отчётов, прототипов и проектов одного разработчика.
- Дефолт 2026 года для нового сервиса среднего размера — **два проекта `Core` + `Web`**, вертикальные срезы внутри и правило зависимостей под контролем архитектурных тестов по неймспейсам.

## Связанное

- [[Слоистая архитектура]]
- [[Vertical Slice Architecture]]
- [[Как выбрать архитектуру под задачу]]
- [[CQRS]]
- [[Repository и Unit of Work: нужны ли поверх EF Core]]
- [[Инверсия зависимостей на практике]]
- [[DDD: тактические паттерны]]
- [[MediatR и альтернативы]]
- [[Доменные события]]
- [[Result pattern вместо исключений]]
- [[SOLID]]
- [[EF Core: запросы и загрузка связанных данных]]
