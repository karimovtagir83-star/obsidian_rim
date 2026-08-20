---
tags: [раздел-09, di, dependency-injection, ioc, контейнер, собес]
aliases: [Dependency Injection, DI container, IServiceCollection, IServiceProvider, DI в ASP.NET Core, Инверсия управления, IoC]
---

# Dependency Injection: контейнер ASP.NET Core

> [!abstract] Коротко
> DI в ASP.NET Core — не библиотека поверх фреймворка, а часть Generic Host: `IServiceCollection` собирает граф регистраций на старте, `BuildServiceProvider()` компилирует их в `IServiceProvider`, а дальше рантайм резолвит зависимости через конструкторы. Три метода регистрации (`AddTransient`/`AddScoped`/`AddSingleton`), два семейства API для избежания дублей (`TryAdd*`/`Add*`), поддержка множественных реализаций через `IEnumerable<T>` и открытых дженериков — вот из чего состоит 90% продовых конфигураций `Program.cs`. Контейнер сознательно простой: без property injection, без сложного перехвата (interception) — для этого есть Autofac/Lamar, но большинству Middle-проектов они не нужны. Отдельные заметки разбирают жизненные циклы ([[Жизненные циклы сервисов: Singleton, Scoped, Transient]]) и типичные ошибки вроде captive dependency ([[Captive dependency и типичные ошибки DI]]) — здесь фокус на механике самого контейнера.

## Зачем это нужно

DI — не «модный паттерн для собеседований», а конкретная защита от конкретных багов, которые ловятся в проде месяцами:

- **Захваченная (captive) зависимость.** `Scoped`-репозиторий с `DbContext` внедрили в `Singleton`-кеш. Первый запрос создал экземпляр репозитория и держит его вечно — все последующие запросы работают с одним и тем же `DbContext`, который не потокобезопасен. Итог: `InvalidOperationException: A second operation started on this context` под нагрузкой, а на слабой нагрузке — тихая порча данных между несвязанными запросами.
- **Утечка памяти от неверного lifetime.** Сервис, реализующий `IDisposable` и владеющий `HttpClient`/файловым хендлом, зарегистрирован как `Transient` и резолвится вручную через `IServiceProvider` без scope — контейнер не знает, когда его освобождать, и объекты копятся до `OutOfMemoryException`.
- **Service Locator вместо DI.** Кто-то протащил `IServiceProvider` в бизнес-класс и резолвит зависимости внутри метода вместо конструктора. Явные зависимости класса стали неявными, тесты превратились в мучение, а `Scoped`-сервис, резолвнутый из root-провайдера, оказывается синглтоном без предупреждения.
- **Двойная регистрация в библиотеке.** NuGet-пакет вызвал `services.AddScoped<IEmailSender, SmtpSender>()` вместо `TryAddScoped`, и приложение-хост, которое хочет подменить реализацию своей, получает второй экземпляр в `IEnumerable<IEmailSender>` — письма уходят дважды.
- **Забытая валидация графа.** Опечатка в конструкторе (`IOrderRepository` вместо `IOrderReadRepository`) не поймана компилятором, потому что оба интерфейса зарегистрированы, но не тем классом, — и падает не при старте, а на первом запросе прод-трафика ночью.

Все эти сценарии — не про «правильный код Java-стиля», а про то, как работает конкретный контейнер ASP.NET Core: что он резолвит, когда создаёт объекты и когда их убивает.

## Зачем встроенный контейнер, а не Autofac/Ninject/StructureMap

До ASP.NET Core (в классическом ASP.NET MVC/Web API) DI не было в фреймворке вообще — каждый проект тянул сторонний контейнер и писал адаптер `IDependencyResolver`. Начиная с ASP.NET Core, DI встроен на уровне `Microsoft.Extensions.DependencyInjection` (пакет `Microsoft.Extensions.DependencyInjection.Abstractions` + реализация), и весь фреймворк — middleware, MVC, `IHostedService`, логирование, `IOptions` — сам построен на этом контейнере и резолвит через него свои внутренности.

Причины, почему Microsoft сделала свой контейнер, а не выбрала один из существующих:

- **Скорость и предсказуемость.** Встроенный контейнер — это в первую очередь дерево делегатов, скомпилированное один раз при `Build()`. Разрешение зависимостей должно быть быстрым на каждый HTTP-запрос — сложные контейнеры с рефлексией на каждый resolve сюда не годятся.
- **Ноль внешних зависимостей во фреймворке.** Microsoft не может обязать всех пользователей ASP.NET Core тянуть конкретный сторонний DI-контейнер как часть ядра. Абстракция `IServiceProvider` существует в BCL с .NET Framework 1.1 — вокруг неё и построили минимальную полноценную реализацию.
- **Достаточность для 90% случаев.** Большинству приложений не нужны property injection, module-based конфигурация, сложный AOP-перехват или assembly-scanning «из коробки» — а именно это отличает «взрослые» контейнеры.
- **Расширяемость через `IServiceProviderFactory<T>`.** Если возможностей штатного контейнера не хватает, `Host.CreateApplicationBuilder()` позволяет подменить фабрику провайдера на Autofac/Lamar без переписывания регистраций — `UseServiceProviderFactory(new AutofacServiceProviderFactory())`.

```csharp
var builder = WebApplication.CreateBuilder(args);

// Явная подмена контейнера на Autofac при необходимости:
// builder.Host.UseServiceProviderFactory(new AutofacServiceProviderFactory());
// builder.Host.ConfigureContainer<ContainerBuilder>(cb => cb.RegisterModule<AppModule>());

builder.Services.AddScoped<IOrderService, OrderService>();

var app = builder.Build();
```

> [!info] `IServiceCollection` vs `IServiceProvider`
> `IServiceCollection` — просто список записей (`ServiceDescriptor`): интерфейс, реализация, lifetime. Это **этап конфигурации**, доступен только до `Build()`. `IServiceProvider` — скомпилированный из этого списка резолвер, доступен в рантайме через `HttpContext.RequestServices` или прямую инъекцию. После `builder.Build()` менять `IServiceCollection` уже нельзя (кроме экспериментального `IServiceCollection` через `IServiceProviderIsService` для проверок — но не для добавления сервисов).

## Регистрация: три lifetime, три способа задать реализацию

Полная механика lifetime-ов (когда именно освобождается объект, что такое scope в контексте HTTP-запроса) — в [[Жизненные циклы сервисов: Singleton, Scoped, Transient]]. Здесь — только сигнатуры и формы регистрации, общие для всех трёх методов.

| Метод | Когда создаётся новый экземпляр |
|---|---|
| `AddTransient<TService, TImpl>()` | при каждом запросе из контейнера |
| `AddScoped<TService, TImpl>()` | один раз на scope (для веб-запроса — один раз на HTTP-запрос) |
| `AddSingleton<TService, TImpl>()` | один раз за время жизни приложения |

Каждый метод имеет одинаковый набор перегрузок — форма регистрации не зависит от lifetime:

```csharp
// 1. Тип → тип. Самая частая форма: интерфейс к реализации.
builder.Services.AddScoped<IOrderService, OrderService>();

// 2. Только конкретный тип (без интерфейса) — для внутренних классов.
builder.Services.AddScoped<OrderValidator>();

// 3. Готовый экземпляр — только для Singleton, экземпляр создаётся ДО контейнера.
builder.Services.AddSingleton(new RecyclableMemoryStreamManager());

// 4. Фабричный делегат — когда создание требует логики или других сервисов.
builder.Services.AddScoped<IOrderService>(sp =>
{
    var repo = sp.GetRequiredService<IOrderRepository>();
    var clock = sp.GetRequiredService<TimeProvider>();
    return new OrderService(repo, clock, enableAudit: true);
});

// 5. Регистрация одного и того же экземпляра под несколькими интерфейсами.
builder.Services.AddSingleton<ICacheWarmer, CacheWarmer>();
builder.Services.AddSingleton<IHostedService>(sp => (CacheWarmer)sp.GetRequiredService<ICacheWarmer>());
```

> [!warning] Форма №5 — частая ловушка
> Если зарегистрировать `AddSingleton<ICacheWarmer, CacheWarmer>()` и отдельно `AddHostedService<CacheWarmer>()`, вы получите **два разных экземпляра** `CacheWarmer` — контейнер не связывает регистрации по типу реализации автоматически. Чтобы это был один и тот же объект под двумя интерфейсами, нужно регистрировать явно через фабрику, как в примере выше, либо регистрировать конкретный тип синглтоном и подписывать интерфейсы через `sp.GetRequiredService<CacheWarmer>()`.

### Резолвинг: `GetService` vs `GetRequiredService`

```csharp
var svc = app.Services.GetService<IOrderService>();     // null, если не зарегистрирован
var svc2 = app.Services.GetRequiredService<IOrderService>(); // InvalidOperationException, если не зарегистрирован
```

> [!tip] Правило на собеседовании
> В прикладном коде почти всегда нужен `GetRequiredService` — падение при старте лучше, чем `NullReferenceException` глубоко в бизнес-логике посреди обработки запроса. `GetService` уместен только там, где отсутствие сервиса — валидный сценарий (опциональный плагин).

## `TryAdd*` и `Replace` — регистрация без дублей

Ключевая идея: **приложение (хост) должно иметь возможность переопределить регистрацию библиотеки**, а библиотека не должна случайно задублировать регистрацию, если она уже была сделана раньше (например, другим пакетом или самим хостом).

```csharp
// Обычный Add — добавляет ВСЕГДА, даже если сервис уже зарегистрирован.
services.AddScoped<IEmailSender, SmtpSender>();
services.AddScoped<IEmailSender, SmtpSender>(); // теперь в контейнере ДВЕ записи

// TryAdd — добавляет, только если для этого TService ещё нет НИ ОДНОЙ регистрации.
services.TryAddScoped<IEmailSender, SmtpSender>();
services.TryAddScoped<IEmailSender, SmtpSender>(); // вторая регистрация проигнорирована
```

| Метод | Поведение |
|---|---|
| `Add*` | всегда добавляет новую запись, даже дублирующую |
| `TryAdd*` | добавляет, только если сервиса **с таким `TService`** ещё нет |
| `TryAddEnumerable` | добавляет, только если нет записи **с такой же парой (`TService`, `TImplementation`)** |
| `Replace` | удаляет существующую регистрацию `TService` (если есть) и добавляет новую |
| `RemoveAll<TService>` | удаляет все регистрации данного `TService` |

```csharp
// Хост хочет свою реализацию вместо библиотечной — Replace вместо накопления дублей.
services.Replace(ServiceDescriptor.Scoped<IEmailSender, MailgunSender>());
```

### `TryAddEnumerable` — для случая нескольких реализаций одного интерфейса

`TryAdd` не подходит, когда интерфейс сознательно регистрируется несколько раз под разные реализации (см. следующий раздел про `IEnumerable<T>`) — он заблокирует вторую и последующие регистрации. Для этого случая есть `TryAddEnumerable`, которая сверяет **конкретную пару интерфейс+реализация**, а не просто факт наличия интерфейса:

```csharp
services.TryAddEnumerable(ServiceDescriptor.Scoped<INotifier, EmailNotifier>());
services.TryAddEnumerable(ServiceDescriptor.Scoped<INotifier, EmailNotifier>()); // дубль отфильтрован
services.TryAddEnumerable(ServiceDescriptor.Scoped<INotifier, SmsNotifier>());   // добавится — другая реализация
```

Так, например, устроена регистрация `IPostConfigureOptions<T>` и валидаторов Options pattern внутри самого ASP.NET Core — фреймворк использует `TryAddEnumerable`, чтобы разные вызовы `AddOptions()` из разных частей кода не плодили дублирующиеся обработчики.

> [!info] Почему библиотеки обязаны использовать `TryAdd*`
> Экосистема .NET построена на композиции пакетов: `AddAuthentication()`, `AddControllers()`, сторонние NuGet-пакеты — каждый вызывает свои `AddXxx()`-методы, и они выполняются в произвольном порядке относительно кода хоста. Если библиотека использует обычный `Add`, то порядок регистрации `services.AddLibraryFoo(); services.AddScoped<IFoo, MyFoo>();` даст в `IEnumerable<IFoo>` **обе** реализации, а `GetRequiredService<IFoo>()` вернёт **последнюю** зарегистрированную — то есть либо библиотечную, либо вашу, в зависимости от порядка строк, что хрупко и неочевидно. Правило экосистемы: **библиотеки регистрируют через `TryAdd*`, чтобы хост мог всегда безопасно переопределить их вызовом `Add*`/`Replace` после**.

## `GetRequiredService<T>()` резолвит последнюю регистрацию

Важный, часто путаемый на собеседовании факт: если для одного `TService` есть несколько регистраций (через `Add`, не `TryAdd`), одиночный `GetRequiredService<T>()` вернёт **последнюю по порядку регистрации**, а не первую и не выбросит исключение о неоднозначности (в отличие от, скажем, Autofac по умолчанию без явной настройки).

```csharp
builder.Services.AddScoped<INotifier, EmailNotifier>();
builder.Services.AddScoped<INotifier, SmsNotifier>();

var notifier = app.Services.GetRequiredService<INotifier>();
// notifier — это SmsNotifier: последняя регистрация "выигрывает" при одиночном резолве
```

## Резолвинг нескольких реализаций через `IEnumerable<T>`

Если интерфейс зарегистрирован несколько раз, инъекция `IEnumerable<TService>` в конструктор вернёт **все** реализации **в порядке регистрации** — это гарантия контейнера, а не случайность.

```csharp
public interface INotifier
{
    Task NotifyAsync(string userId, string message);
}

public sealed class EmailNotifier : INotifier
{
    public Task NotifyAsync(string userId, string message) => /* ... */ Task.CompletedTask;
}

public sealed class SmsNotifier : INotifier
{
    public Task NotifyAsync(string userId, string message) => /* ... */ Task.CompletedTask;
}

public sealed class PushNotifier : INotifier
{
    public Task NotifyAsync(string userId, string message) => /* ... */ Task.CompletedTask;
}

// Регистрация — порядок важен для порядка вызова:
builder.Services.AddScoped<INotifier, EmailNotifier>();
builder.Services.AddScoped<INotifier, SmsNotifier>();
builder.Services.AddScoped<INotifier, PushNotifier>();

// Использование — «разослать всеми каналами» (strategy / composite):
public sealed class NotificationDispatcher(IEnumerable<INotifier> notifiers)
{
    public async Task NotifyAllAsync(string userId, string message)
    {
        foreach (var notifier in notifiers)
            await notifier.NotifyAsync(userId, message);
    }
}
```

Типовые применения этой гарантии:

- **Strategy pattern** — набор алгоритмов (валидаторов, форматтеров, экспортёров), где вызывающий код перебирает все или выбирает нужный по условию (`notifiers.FirstOrDefault(n => n.Supports(channel))`).
- **Chain of Responsibility / pipeline** — обработчики выполняются по очереди в порядке регистрации: middleware самого ASP.NET Core внутри устроен похожим образом (хотя технически не через `IEnumerable`, а через делегаты, идея та же).
- **Health checks, `IValidateOptions<T>`, `IConfigureOptions<T>`** — сам фреймворк использует этот механизм: `services.AddHealthChecks().AddCheck<DbHealthCheck>()` добавляет ещё одну запись в `IEnumerable<IHealthCheck>` под капотом.

> [!warning] Порядок регистрации — это порядок из кода, а не порядок сборки/загрузки
> Если регистрации INotifier разбросаны по разным `AddXxx()`-методам расширения из разных частей `Program.cs`, порядок в `IEnumerable<INotifier>` определяется порядком **вызова** этих методов, а не алфавитным или каким-либо ещё. Изменение порядка строк в `Program.cs` меняет порядок обработки — это легко сломать при рефакторинге, если логика полагается на порядок неявно.

## Декоратор: вручную и через Scrutor

Штатный `IServiceCollection` **не имеет** метода `Decorate<T>()` — оборачивание одной реализации в другую («добавить логирование/кеширование поверх существующего сервиса, не трогая его код») нужно либо писать руками через фабричную регистрацию, либо подключить пакет **Scrutor**, который решает это в одну строку.

### Вручную — через фабрику и явную оборачиваемую реализацию

```csharp
public interface IOrderRepository
{
    Task<Order?> GetAsync(int id);
}

public sealed class SqlOrderRepository(AppDbContext db) : IOrderRepository
{
    public Task<Order?> GetAsync(int id) => db.Orders.FindAsync(id).AsTask();
}

// Декоратор с кешированием — оборачивает "внутреннюю" реализацию.
public sealed class CachingOrderRepository(IOrderRepository inner, IMemoryCache cache) : IOrderRepository
{
    public async Task<Order?> GetAsync(int id)
    {
        if (cache.TryGetValue(id, out Order? cached))
            return cached;

        var order = await inner.GetAsync(id);
        if (order is not null)
            cache.Set(id, order, TimeSpan.FromMinutes(5));

        return order;
    }
}
```

```csharp
// Регистрация: "внутренний" сервис под собственным конкретным типом,
// а публичный интерфейс — за фабрикой, которая строит цепочку декораторов.
builder.Services.AddScoped<SqlOrderRepository>();
builder.Services.AddMemoryCache();
builder.Services.AddScoped<IOrderRepository>(sp =>
    new CachingOrderRepository(
        sp.GetRequiredService<SqlOrderRepository>(),
        sp.GetRequiredService<IMemoryCache>()));
```

Это работает, но плохо масштабируется: каждая новая обёртка (логирование, метрики, retry) требует ручного изменения фабрики, а порядок оборачивания легко перепутать.

### Через Scrutor — `Decorate<T>()`

```csharp
// dotnet add package Scrutor
using Scrutor;

builder.Services.AddScoped<IOrderRepository, SqlOrderRepository>();

// Регистрация декораторов ПОСЛЕ основной реализации, в порядке "снаружи внутрь" по вызову:
builder.Services.Decorate<IOrderRepository, CachingOrderRepository>();
builder.Services.Decorate<IOrderRepository, LoggingOrderRepository>();

// Итоговая цепочка вызовов при резолве IOrderRepository:
// LoggingOrderRepository → CachingOrderRepository → SqlOrderRepository
```

`Decorate<TService, TDecorator>()` находит существующую регистрацию `TService`, регистрирует её под новым внутренним ключом и подменяет публичную регистрацию на `TDecorator`, конструктор которого принимает `TService` (исходную реализацию) как один из параметров — сам декоратор пишется точно так же, как в примере выше (`CachingOrderRepository(IOrderRepository inner, ...)`), только регистрацию Scrutor берёт на себя.

Scrutor умеет также сканировать сборки и регистрировать сервисы по конвенции (`services.Scan(scan => scan.FromAssemblyOf<OrderService>().AddClasses(...).AsImplementedInterfaces().WithScopedLifetime())`) — это ближе к тому, что «из коробки» умеют Autofac/Lamar.

> [!tip] Когда декоратор, а когда middleware
> Декоратор оборачивает **конкретный сервис** и виден только коду, который его вызывает через DI. Middleware оборачивает **весь HTTP-конвейер** и видит любой запрос независимо от того, какой сервис его в итоге обработает. Кросс-cutting логика, привязанная к бизнес-операции (закешировать конкретный метод репозитория) — декоратор; логика, привязанная к HTTP-запросу целиком (логировать все запросы, добавить заголовок) — [[Middleware и конвейер обработки запроса]].

## Конструкторная инъекция: правила выбора конструктора

Контейнер ASP.NET Core поддерживает **только конструкторную инъекцию** — нет property injection и method injection из коробки (в отличие от Autofac, где это доступно через `PropertiesAutowired()`).

Правила выбора конструктора при нескольких публичных конструкторах:

```csharp
public sealed class OrderService : IOrderService
{
    private readonly IOrderRepository _repo;
    private readonly ILogger<OrderService> _logger;

    // Если конструктор один — контейнер использует его безусловно,
    // требуя, чтобы ВСЕ параметры были разрешимы.
    public OrderService(IOrderRepository repo, ILogger<OrderService> logger)
    {
        _repo = repo;
        _logger = logger;
    }
}
```

Если публичных конструкторов несколько, контейнер выбирает **тот, у которого можно разрешить все параметры и в котором их больше всего** (жадный алгоритм — "the greediest constructor that can be satisfied"):

```csharp
public sealed class ReportGenerator
{
    // Если IAuditLog не зарегистрирован в контейнере, будет выбран короткий конструктор.
    public ReportGenerator(IDataSource source) { /* ... */ }
    public ReportGenerator(IDataSource source, IAuditLog auditLog) { /* ... */ }
}
```

> [!danger] Неоднозначность конструкторов — исключение при резолве, не при компиляции
> Если два конструктора требуют **разное** число параметров, но оба разрешимы, выбирается тот, где параметров больше. Но если существуют **два конструктора с одинаковым числом параметров**, оба из которых полностью разрешимы, — контейнер выбросит `InvalidOperationException: Multiple constructors accepting all given argument types have been found` **в момент первого резолва**, а не при старте приложения и не на этапе компиляции. Это одна из причин держать по одному публичному конструктору на класс, инжектируемый через DI — вторую (не публичную) перегрузку для тестов делайте `internal`/приватной с фабричным методом, а не вторым `public`.

Опциональные параметры конструктора контейнер тоже поддерживает: если сервис не зарегистрирован, но у параметра есть значение по умолчанию, конструктор всё равно резолвится:

```csharp
public sealed class FeatureToggleService(IFeatureStore store, ILogger<FeatureToggleService>? logger = null)
{
    // logger будет null, если ILogger<FeatureToggleService> не удаётся разрешить —
    // но для ILogger<T> это фактически невозможно: логирование зарегистрировано всегда.
}
```

## `IServiceProvider` изнутри: как устроено разрешение

Концептуально, без деталей реализации:

1. При `builder.Build()` (или `services.BuildServiceProvider()`) все записи `IServiceCollection` (`ServiceDescriptor`: тип сервиса, тип реализации/фабрика, lifetime) компилируются во внутреннюю структуру — по сути, словарь `Type → фабричный делегат`, с кешированием собранных цепочек вызовов конструкторов через `Expression`/`ILEmit` там, где это быстрее прямой рефлексии.
2. `GetService<T>()` ищет фабрику для `T`, вызывает её; если у фабрики есть параметры-зависимости — рекурсивно резолвит их точно так же.
3. `Scoped` и `Singleton` регистрации кешируют созданный экземпляр: `Singleton` — в корневом провайдере на всё время жизни приложения, `Scoped` — в текущем `IServiceScope` до его `Dispose()`.
4. `Transient` не кешируется вообще — каждый вызов `GetService<T>()` создаёт новый объект (но если `Transient`-сервис реализует `IDisposable`, контейнер всё равно держит ссылку на него в текущем scope, чтобы вызвать `Dispose()` при завершении scope — отсюда возможна утечка при резолве множества `Transient` из долгоживущего scope/root).

### Scope: `IServiceScopeFactory` и `CreateScope()`

Для каждого HTTP-запроса ASP.NET Core сам создаёт scope и кладёт провайдер в `HttpContext.RequestServices` — обычный код в контроллерах/эндпоинтах об этом не думает. Но там, где HTTP-запроса нет (фоновый сервис, консьюмер очереди, `Timer`), scope нужно создавать вручную:

```csharp
public sealed class OrderCleanupService(IServiceScopeFactory scopeFactory) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            // IHostedService — Singleton. Scoped-зависимости внутри него
            // нужно резолвить через новый scope на каждую итерацию, а не через конструктор.
            using (var scope = scopeFactory.CreateScope())
            {
                var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
                await CleanupExpiredOrdersAsync(db, stoppingToken);
            } // Dispose(scope) освобождает всё, что было Scoped внутри него

            await Task.Delay(TimeSpan.FromMinutes(10), stoppingToken);
        }
    }

    private static Task CleanupExpiredOrdersAsync(AppDbContext db, CancellationToken ct) =>
        Task.CompletedTask;
}
```

Подробнее про фоновые сервисы и их жизненный цикл — [[Background services и IHostedService]]. Полная механика того, почему `Scoped` нельзя внедрять напрямую в `Singleton`/`IHostedService` через конструктор, — [[Captive dependency и типичные ошибки DI]].

### Валидация графа: `ValidateOnBuild` и `ValidateScopes`

По умолчанию в `Development`-окружении (через `WebApplication.CreateBuilder`) ASP.NET Core включает две проверки, которые в `Production` выключены ради производительности:

```csharp
var builder = WebApplication.CreateBuilder(args);

// Эквивалент того, что уже включено автоматически в Development;
// показано явно для не-Development окружений или консольных хостов:
builder.Host.UseDefaultServiceProvider((context, options) =>
{
    options.ValidateScopes = context.HostingEnvironment.IsDevelopment();
    options.ValidateOnBuild = context.HostingEnvironment.IsDevelopment();
});
```

| Опция | Что проверяет | Когда стреляет |
|---|---|---|
| `ValidateOnBuild` | что **каждый** зарегистрированный сервис можно построить (все зависимости разрешимы) | сразу на `Build()`, до первого запроса |
| `ValidateScopes` | что `Scoped`-сервис не резолвится из корневого (`Singleton`-области) провайдера | в момент резолва такого сервиса |

```csharp
// Без ValidateScopes это "тихо" сработает и создаст captive dependency:
var scoped = app.Services.GetRequiredService<IOrderRepository>(); // из root provider!

// С ValidateScopes=true в Development — сразу исключение:
// InvalidOperationException: Cannot resolve scoped service
// 'IOrderRepository' from root provider.
```

> [!warning] Почему это выключено в Production
> Обе проверки требуют пройтись по всему графу регистраций (`ValidateOnBuild`) или добавляют проверку на каждый resolve (`ValidateScopes`) — заметные накладные расходы на старте и в рантайме соответственно. Идея: ошибки конфигурации DI должны ловиться **на этапе разработки и в CI** (запуском приложения или интеграционным тестом с `ValidateOnBuild=true`), а не платить за проверку каждым запросом в проде. Если у вас нет интеграционного теста, поднимающего `WebApplicationFactory` целиком, — считайте, что `ValidateOnBuild` в проде не подстрахует, потому что до прода он просто не добежал.

## Открытые дженерики

Контейнер умеет регистрировать generic-типы без указания конкретного параметра — резолв конкретизирует их автоматически:

```csharp
public interface IRepository<T> where T : class
{
    Task<T?> GetAsync(int id);
    Task AddAsync(T entity);
}

public sealed class Repository<T>(AppDbContext db) : IRepository<T> where T : class
{
    public Task<T?> GetAsync(int id) => db.Set<T>().FindAsync(id).AsTask();
    public Task AddAsync(T entity) => db.Set<T>().AddAsync(entity).AsTask();
}

// Одна регистрация покрывает IRepository<Order>, IRepository<Customer>, IRepository<Product>, ...
builder.Services.AddScoped(typeof(IRepository<>), typeof(Repository<>));
```

```csharp
// Использование — конкретный T подставляется автоматически при резолве:
public sealed class OrderService(IRepository<Order> orders) { /* ... */ }
```

Открытые дженерики можно комбинировать с `IEnumerable<T>` и декораторами — например, обобщённый `LoggingRepositoryDecorator<T>`, зарегистрированный через `Decorate(typeof(IRepository<>), typeof(LoggingRepository<>))` в Scrutor, обернёт **все** закрытые дженерик-версии сразу.

> [!info] Смешивание открытого и закрытого дженерика
> Если для конкретного `T` (скажем, `Order`) нужна отдельная реализация, отличная от общей `Repository<T>`, — зарегистрируйте закрытый дженерик **после** открытого: `services.AddScoped<IRepository<Order>, SpecialOrderRepository>();`. Явная (закрытая) регистрация имеет приоритет при резолве конкретного `IRepository<Order>`, потому что в реализации контейнера сначала ищутся точные совпадения типа, и только при их отсутствии — открытые дженерик-регистрации.

## Что штатный контейнер сознательно не умеет

| Возможность | Встроенный DI | Autofac / Lamar |
|---|---|---|
| Конструкторная инъекция | да | да |
| Property/method injection | **нет** | да |
| `Decorate<T>()` из коробки | нет (нужен Scrutor) | да, есть встроенный |
| Автоматическое сканирование сборок (convention-based регистрация) | нет (нужен Scrutor `Scan()`) | да, мощный DSL |
| Дочерние (именованные) контейнеры / модули | нет | да (Autofac modules, lifetime scopes с тегами) |
| Перехват вызовов (interception/AOP) | нет | ограниченно, обычно через доп. пакеты (Castle) |
| Keyed services | да, начиная с .NET 8 (`AddKeyedScoped` и т.п.) | да, давно |
| Диагностика графа регистраций визуально | нет встроенного UI | у Autofac есть сторонние инструменты |
| Условная регистрация по окружению | вручную через `if` в `Program.cs` | встроенные механизмы регистрации по условию |

Keyed services подробно разобраны в [[Keyed services и продвинутая регистрация]].

> [!tip] Когда реально нужен Autofac/Lamar
> Для подавляющего большинства Middle/Middle+ веб-проектов встроенного контейнера достаточно: явная регистрация в `Program.cs` — это не "лишний бойлерплейт", а документация состава приложения, которую видно без стороннего DSL. Переходить на Autofac имеет смысл, когда: (1) нужна **модульная сборка** — десятки независимых сборок с собственной DI-конфигурацией, объединяемых в разные хосты (модули Autofac); (2) требуется **перехват** (логирование/транзакции через прокси без явных декораторов на каждый класс); (3) legacy-код мигрирует с классического ASP.NET, где Autofac уже был ядром DI. Для большинства собеседований на Middle 2 достаточно знать *что* не умеет встроенный контейнер и *почему* этого обычно достаточно — а не наизусть API Autofac.

## Частые ошибки и антипаттерны

### 1. Service Locator вместо инъекции зависимостей

```csharp
// ПЛОХО: сервис прячет свои реальные зависимости внутри метода
public sealed class OrderController(IServiceProvider services) : ControllerBase
{
    public async Task<IActionResult> Create(OrderDto dto)
    {
        var orderService = services.GetRequiredService<IOrderService>(); // антипаттерн
        await orderService.CreateAsync(dto);
        return Ok();
    }
}
```

```csharp
// ХОРОШО: зависимость видна в сигнатуре конструктора — её видно в тестах и в IDE
public sealed class OrderController(IOrderService orderService) : ControllerBase
{
    public async Task<IActionResult> Create(OrderDto dto)
    {
        await orderService.CreateAsync(dto);
        return Ok();
    }
}
```

Service Locator не просто "некрасив" — он скрывает реальный граф зависимостей класса от компилятора, от тестов и от `ValidateOnBuild`, а резолв `Scoped`-сервиса из неправильного провайдера (например, сохранённого в поле `Singleton`-класса `IServiceProvider`) — прямой путь к captive dependency, разобранной в [[Captive dependency и типичные ошибки DI]].

### 2. Построение временного `ServiceProvider` только для валидации

```csharp
// ПЛОХО: BuildServiceProvider() создаёт ВТОРОЙ, отдельный контейнер,
// у Singleton-сервисов в котором будут СВОИ экземпляры, не те, что в приложении
public void ConfigureServices(IServiceCollection services)
{
    services.AddSingleton<ICacheWarmer, CacheWarmer>();

    var sp = services.BuildServiceProvider(); // временный провайдер!
    var warmer = sp.GetRequiredService<ICacheWarmer>();
    warmer.WarmUp(); // это НЕ тот CacheWarmer, что будет резолвиться дальше в приложении
}
```

`BuildServiceProvider()`, вызванный вручную посреди конфигурации, — почти всегда ошибка: он строит независимый граф провайдера, синглтоны в котором никак не связаны с синглтонами основного `WebApplication`-провайдера. Единственное легитимное место для ручного `BuildServiceProvider()` — конец `Program.cs` (собственно `builder.Build()`, который делает это за вас) или изолированные unit/integration-тесты, где временный контейнер — это и есть весь смысл теста.

### 3. Резолв `Scoped` из root-провайдера в `Program.cs`

```csharp
var app = builder.Build();

// ПЛОХО: app.Services — это ROOT provider, у него своего HTTP-scope нет
var repo = app.Services.GetRequiredService<IOrderRepository>(); // если репозиторий Scoped — captive dependency

// ХОРОШО: создать явный scope
using (var scope = app.Services.CreateScope())
{
    var repo2 = scope.ServiceProvider.GetRequiredService<IOrderRepository>();
    // использовать repo2 и dispose-нуть scope здесь же
}

app.Run();
```

Это типично для миграций/seed-данных при старте приложения — код, который выполняется один раз до `app.Run()`, но нуждается в `Scoped`-зависимостях вроде `DbContext`.

## Полный пример: сборка небольшой фичи

Собираем воедино: репозиторий, декоратор с кешированием через Scrutor, несколько `INotifier` через `IEnumerable<T>`.

```csharp
// --- Контракты и реализации ---

public interface IOrderRepository
{
    Task<Order?> GetAsync(int id);
    Task<int> CreateAsync(Order order);
}

public sealed class SqlOrderRepository(AppDbContext db) : IOrderRepository
{
    public Task<Order?> GetAsync(int id) => db.Orders.FindAsync(id).AsTask();

    public async Task<int> CreateAsync(Order order)
    {
        db.Orders.Add(order);
        await db.SaveChangesAsync();
        return order.Id;
    }
}

public sealed class CachingOrderRepository(IOrderRepository inner, IMemoryCache cache) : IOrderRepository
{
    public async Task<Order?> GetAsync(int id)
    {
        if (cache.TryGetValue($"order:{id}", out Order? cached))
            return cached;

        var order = await inner.GetAsync(id);
        if (order is not null)
            cache.Set($"order:{id}", order, TimeSpan.FromMinutes(2));

        return order;
    }

    public Task<int> CreateAsync(Order order) => inner.CreateAsync(order); // кеш не участвует в записи
}

public interface INotifier
{
    Task NotifyAsync(int orderId, string message);
}

public sealed class EmailNotifier(ILogger<EmailNotifier> logger) : INotifier
{
    public Task NotifyAsync(int orderId, string message)
    {
        logger.LogInformation("Email for order {OrderId}: {Message}", orderId, message);
        return Task.CompletedTask;
    }
}

public sealed class SmsNotifier(ILogger<SmsNotifier> logger) : INotifier
{
    public Task NotifyAsync(int orderId, string message)
    {
        logger.LogInformation("SMS for order {OrderId}: {Message}", orderId, message);
        return Task.CompletedTask;
    }
}

public interface IOrderService
{
    Task<int> PlaceOrderAsync(Order order);
}

public sealed class OrderService(
    IOrderRepository repository,
    IEnumerable<INotifier> notifiers) : IOrderService
{
    public async Task<int> PlaceOrderAsync(Order order)
    {
        var id = await repository.CreateAsync(order);

        foreach (var notifier in notifiers)
            await notifier.NotifyAsync(id, "Заказ принят в обработку");

        return id;
    }
}
```

```csharp
// --- Program.cs ---

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDbContext<AppDbContext>(opt =>
    opt.UseNpgsql(builder.Configuration.GetConnectionString("Default")));

builder.Services.AddMemoryCache();

// Базовая регистрация + декоратор поверх неё (порядок вызовов важен для Scrutor):
builder.Services.AddScoped<IOrderRepository, SqlOrderRepository>();
builder.Services.Decorate<IOrderRepository, CachingOrderRepository>();

// Несколько реализаций одного интерфейса — резолвятся вместе через IEnumerable<INotifier>:
builder.Services.AddScoped<INotifier, EmailNotifier>();
builder.Services.AddScoped<INotifier, SmsNotifier>();

// Библиотека уведомлений использует TryAdd, чтобы хост мог переопределить свою реализацию:
builder.Services.TryAddScoped<IOrderService, OrderService>();

var app = builder.Build();

app.MapPost("/api/orders", async (Order order, IOrderService service) =>
{
    var id = await service.PlaceOrderAsync(order);
    return Results.Created($"/api/orders/{id}", new { id });
});

app.Run();
```

При резолве `IOrderService` контейнер построит цепочку: `OrderService` ← `CachingOrderRepository` (обёртка) ← `SqlOrderRepository` (реальный доступ к БД), плюс `IEnumerable<INotifier>` с двумя элементами в порядке регистрации — `EmailNotifier`, затем `SmsNotifier`.

## Вопросы с собеседований

> [!question]- В чём разница между `Add<T>`, `TryAdd<T>` и `TryAddEnumerable<T>`? Зачем нужны все три?
> `Add` всегда добавляет новую запись, даже если для этого сервиса уже есть регистрация — это создаёт дубли. `TryAdd` добавляет только если для данного `TService` **вообще ещё нет** записи — используется библиотеками для "регистрации по умолчанию, если хост не задал своё". `TryAddEnumerable` добавляет, только если нет записи с такой же **парой** (интерфейс, реализация) — используется, когда интерфейс сознательно регистрируется несколько раз под разные реализации (например, несколько `INotifier`), но нужно защититься именно от повторной регистрации той же самой пары при повторном вызове метода расширения.

> [!question]- Что произойдёт, если запросить `IOrderRepository` через `GetRequiredService`, если он зарегистрирован дважды разными реализациями?
> Вернётся **последняя** зарегистрированная реализация — контейнер не бросает исключение о неоднозначности при одиночном резолве (в отличие от многих других DI-контейнеров). Если нужны все реализации — инжектировать нужно `IEnumerable<IOrderRepository>`, а не сам интерфейс.

> [!question]- Как реализовать декоратор для сервиса без стороннего пакета?
> Зарегистрировать исходную реализацию под её конкретным типом (`services.AddScoped<SqlOrderRepository>()`), а публичный интерфейс — через фабричный делегат, который создаёт декоратор, оборачивающий разрешённую конкретную реализацию (`services.AddScoped<IOrderRepository>(sp => new CachingOrderRepository(sp.GetRequiredService<SqlOrderRepository>(), ...))`). Для нескольких слоёв декораторов это быстро становится громоздким — тогда используют Scrutor и его `Decorate<TService, TDecorator>()`, который делает то же самое автоматически и позволяет складывать декораторы цепочкой в порядке регистрации.

> [!question]- Почему `Scoped`-сервис нельзя внедрить через конструктор в `Singleton`?
> Формально это можно "заставить" работать, но это создаёт captive dependency: `Singleton` создаётся один раз в root-scope, и если ему в конструктор передать `Scoped`-зависимость, она резолвится **один раз**, в момент создания синглтона, и живёт вместе с ним вечно — вместо того чтобы обновляться на каждый scope/запрос. С `ValidateScopes = true` (по умолчанию в Development) это либо не даст создать такую регистрацию, либо бросит исключение при резолве. Правильный подход — внедрить в `Singleton`/`IHostedService` фабрику `IServiceScopeFactory` и создавать `Scoped`-сервисы через `CreateScope()` там, где они реально нужны. Подробнее — [[Captive dependency и типичные ошибки DI]].

> [!question]- Что делает `ValidateOnBuild` и почему оно выключено в Production по умолчанию?
> `ValidateOnBuild` проверяет на этапе `Build()`, что каждый зарегистрированный сервис может быть построен — все его зависимости разрешимы, конструктор не даёт неоднозначности и т.п. Ошибка конфигурации ловится сразу при старте, а не при первом запросе, который её затронет. Выключено в Production по умолчанию, потому что это добавляет время к старту приложения (обход всего графа регистраций), а Production предполагает, что конфигурация уже проверена в Development/CI — например, интеграционным тестом, который поднимает `WebApplicationFactory` целиком.

> [!question]- Почему у ASP.NET Core нет property injection, если он есть у Autofac?
> Это осознанное архитектурное решение, а не недоработка: конструкторная инъекция делает зависимости класса **обязательными и видимыми** в его публичном API — без экземпляра всех зависимостей объект просто нельзя создать. Property injection допускает частично сконструированный объект с `null`-полями, что переносит проверку "все ли зависимости заданы" из компилятора и `ValidateOnBuild` в рантайм, в момент фактического использования свойства. Команда ASP.NET Core сознательно выбрала более строгую, но более безопасную модель как поведение по умолчанию для всего фреймворка.

> [!question]- Как зарегистрировать репозиторий-дженерик `IRepository<T>` без явной регистрации под каждый `T`?
> Через регистрацию открытого generic-типа: `services.AddScoped(typeof(IRepository<>), typeof(Repository<>))`. Контейнер конкретизирует `T` автоматически при резолве `IRepository<Order>`, `IRepository<Customer>` и т.д., используя один и тот же generic-класс реализации. Если для конкретного `T` нужна особая реализация — регистрируется отдельно закрытый дженерик (`services.AddScoped<IRepository<Order>, SpecialOrderRepository>()`), который имеет приоритет над открытой регистрацией при резолве именно этого `T`.

> [!question]- Чем плох вызов `services.BuildServiceProvider()` посреди `ConfigureServices`/`Program.cs`?
> Он создаёт **отдельный, независимый** экземпляр `IServiceProvider` со своими `Singleton`-экземплярами, не связанными с тем провайдером, который реально будет использовать приложение после `builder.Build()`. Использование сервисов из такого временного провайдера (например, для "прогрева" кеша при старте) даёт объекты-призраки, которые никогда больше не встретятся в реальном графе запроса — а также плодит несколько экземпляров `Singleton`-сервисов, что нарушает саму гарантию Singleton.

## Итог

- Встроенный DI-контейнер ASP.NET Core — быстрый, простой, конструктор-only; для 90% Middle/Middle+ проектов этого достаточно, Autofac/Lamar нужны для модульности, перехвата и property injection.
- `AddTransient`/`AddScoped`/`AddSingleton` имеют одинаковый набор форм регистрации: тип-к-типу, экземпляр, фабричный делегат.
- `TryAdd*` — правило хорошего тона для библиотек, чтобы хост мог безопасно переопределить регистрацию; `TryAddEnumerable` защищает от дублей конкретной пары (интерфейс, реализация) там, где несколько реализаций — это нормально.
- `IEnumerable<TService>` возвращает все реализации в порядке регистрации — основа strategy/pipeline-паттернов внутри DI.
- Декоратор — не встроенная возможность контейнера; вручную через фабрику или через `Scrutor.Decorate<T>()`.
- `ValidateOnBuild`/`ValidateScopes` включены в Development и ловят captive dependency и недостроимые графы до продакшена — не полагайтесь на них в проде.
- Открытые дженерики (`typeof(IRepository<>)`) закрывают целый класс регистраций одной строкой.
- Главные антипаттерны: Service Locator вместо инъекции, резолв `Scoped` из root-провайдера, лишний ручной `BuildServiceProvider()`.

## Связанное

- [[Жизненные циклы сервисов: Singleton, Scoped, Transient]]
- [[Captive dependency и типичные ошибки DI]]
- [[Options pattern и конфигурация сервисов]]
- [[Keyed services и продвинутая регистрация]]
- [[SOLID]]
- [[Middleware и конвейер обработки запроса]]
- [[Background services и IHostedService]]
- [[09 — ASP.NET Core (обзор раздела)]]
