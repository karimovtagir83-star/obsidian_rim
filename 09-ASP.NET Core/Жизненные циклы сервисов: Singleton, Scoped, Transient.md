---
tags: [раздел-09, di, lifetime, singleton, scoped, transient, dbcontext, собес]
aliases: [Singleton, Scoped, Transient, service lifetime, жизненный цикл сервиса, ServiceLifetime, scope, IServiceScope, время жизни сервиса]
---

# Жизненные циклы сервисов: Singleton, Scoped, Transient

> [!abstract] Коротко
> DI-контейнер ASP.NET Core создаёт экземпляр сервиса по одному из трёх правил: **Transient** — новый объект на каждое разрешение (`Resolve`/инжекцию), **Scoped** — один объект на один scope (по умолчанию — один HTTP-запрос), **Singleton** — один объект на всё время жизни приложения. Контейнер сам вызывает `Dispose`/`DisposeAsync` у созданных им `IDisposable`-объектов, когда их scope завершается — но только у тех, что создал сам. Самая частая продовая авария на этой теме — **captive dependency**: Singleton захватывает Scoped-зависимость через конструктор, та живёт вечно и превращается в сломанный синглтон со протухшим состоянием (или роняет приложение, если включена валидация scope).

## Зачем это нужно

Это не абстрактная теория — вот реальные баги, которые случаются, если lifetime выбран неправильно:

- **`DbContext` как Singleton** → `InvalidOperationException: A second operation was started on this context before a previous operation completed` под нагрузкой, потому что `DbContext` не потокобезопасен, а один инстанс шарится между всеми параллельными запросами.
- **Кеш в Singleton, который тихо ссылается на Scoped-репозиторий** → через день работы кеш отдаёт данные первого пользователя, который его инициализировал — классический captive dependency.
- **Scoped-сервис заинжектен прямо в `BackgroundService`** → `ObjectDisposedException` или вообще `InvalidOperationException` при старте, потому что `BackgroundService` резолвится один раз как часть синглтон-хоста, а Scoped-зависимости вне HTTP-запроса просто не существует, если не создать scope вручную.
- **Transient с `IDisposable`, который никто не освобождает вовремя** → рост потребления памяти/хендлов до следующего конца scope (для Transient, зарезолвленного из корневого провайдера, это может означать «до конца работы приложения»).

Всё это — следствие непонимания одного факта: **lifetime — это не про удобство API, а про управление памятью, потоками и корректностью состояния**.

## Три lifetime: точные определения

```csharp
builder.Services.AddTransient<ITransientService, TransientService>();
builder.Services.AddScoped<IScopedService, ScopedService>();
builder.Services.AddSingleton<ISingletonService, SingletonService>();
```

| | Transient | Scoped | Singleton |
|---|---|---|---|
| **Правило создания** | новый экземпляр при **каждом** разрешении | один экземпляр на **scope** | один экземпляр на **весь процесс** |
| **Когда создаётся новый** | всегда, включая повторный вызов внутри того же метода | при первом разрешении внутри данного scope | один раз — при регистрации с готовым экземпляром либо лениво при первом `Resolve` |
| **Типичное применение** | лёгкие stateless-сервисы: мапперы, валидаторы, фабрики значений | всё, что привязано к запросу: `DbContext`, unit-of-work, «текущий пользователь» | конфигурация, кеши в памяти, HTTP-клиенты (через `IHttpClientFactory`), пулы соединений |
| **Кто и когда вызывает `Dispose`** | контейнер, при закрытии scope, в котором объект был создан | контейнер, в конце scope (для HTTP — в конце запроса) | контейнер, при `Dispose` корневого `IServiceProvider` (остановка приложения) |
| **Поток/конкурентность** | не важно — экземпляр не переживает вызов | не важно в рамках одного HTTP-запроса (он однопоточный по контракту ASP.NET Core), но **опасно**, если тот же scope трогают из нескольких задач | **обязан быть потокобезопасным** — к нему обращаются параллельно из всех запросов |

> [!info] Что такое «scope» на самом деле
> Scope — это `IServiceScope`, обёртка вокруг дочернего `IServiceProvider`, который помнит уже созданные Scoped-инстансы и список объектов на dispose. ASP.NET Core создаёт **один scope на один HTTP-запрос** автоматически — это делает хостинг-слой (`IHttpApplication`/middleware внутри Kestrel-конвейера) ещё до того, как запрос доходит до вашего middleware. Именно поэтому `HttpContext.RequestServices` — это уже scoped-провайдер запроса, а не корневой.
>
> Scope можно создавать и вручную, вне HTTP-запроса — это как раз нужно в фоновых задачах:
> ```csharp
> using var scope = serviceScopeFactory.CreateScope();
> var repo = scope.ServiceProvider.GetRequiredService<IOrderRepository>();
> ```

### Как Singleton создаётся: eager vs lazy

```csharp
// Lazy: экземпляр появится при первом Resolve
builder.Services.AddSingleton<ISingletonService, SingletonService>();

// Eager: экземпляр создан прямо сейчас, руками, до старта контейнера
var cache = new InMemoryFeatureFlagCache();
builder.Services.AddSingleton<IFeatureFlagCache>(cache);

// Eager через фабрику, выполняется при первом Resolve, но можно форсировать
builder.Services.AddSingleton<IFeatureFlagCache>(sp =>
    new InMemoryFeatureFlagCache(sp.GetRequiredService<IOptions<FeatureFlagOptions>>()));
```

> [!warning] `AddSingleton(instance)` — контейнер НЕ вызовет `Dispose`
> Если вы регистрируете уже готовый экземпляр (`AddSingleton<T>(instance)` или `AddSingleton<T>(new T(...))`), контейнер считает, что **не он владеет объектом**, и никогда не вызовет у него `Dispose`, даже если `T : IDisposable`. Ответственность за освобождение остаётся на том, кто создал экземпляр. Это частый источник утечек: разработчик ожидает, что DI «позаботится», а контейнер молча пропускает такие объекты.

## Полный рабочий пример: три сервиса и их Guid

Классический способ увидеть разницу lifetime вживую — завести три интерфейса с одинаковой формой, где конструктор фиксирует `Guid`, и посмотреть, как он меняется между двумя HTTP-запросами.

```csharp
public interface ITransientService { Guid Id { get; } }
public interface IScopedService    { Guid Id { get; } }
public interface ISingletonService { Guid Id { get; } }

public sealed class TransientService : ITransientService
{
    public Guid Id { get; } = Guid.NewGuid();
}

public sealed class ScopedService : IScopedService
{
    public Guid Id { get; } = Guid.NewGuid();
}

public sealed class SingletonService : ISingletonService
{
    public Guid Id { get; } = Guid.NewGuid();
}

// "Составной" сервис, который сам инжектирует все три —
// нужен, чтобы показать разницу между "напрямую в эндпоинт"
// и "через ещё один уровень DI"
public interface ILifetimeReporter
{
    Guid TransientId { get; }
    Guid ScopedId { get; }
    Guid SingletonId { get; }
}

public sealed class LifetimeReporter : ILifetimeReporter
{
    public LifetimeReporter(
        ITransientService transient,
        IScopedService scoped,
        ISingletonService singleton)
    {
        TransientId = transient.Id;
        ScopedId = scoped.Id;
        SingletonId = singleton.Id;
    }

    public Guid TransientId { get; }
    public Guid ScopedId { get; }
    public Guid SingletonId { get; }
}
```

Регистрация:

```csharp
builder.Services.AddTransient<ITransientService, TransientService>();
builder.Services.AddScoped<IScopedService, ScopedService>();
builder.Services.AddSingleton<ISingletonService, SingletonService>();
builder.Services.AddScoped<ILifetimeReporter, LifetimeReporter>();
```

Эндпоинт, который резолвит сервисы **дважды за один запрос** — напрямую и через `ILifetimeReporter`:

```csharp
app.MapGet("/api/lifetime-demo", (
    ITransientService transientDirect,
    IScopedService scopedDirect,
    ISingletonService singletonDirect,
    ILifetimeReporter reporter) =>
{
    return Results.Ok(new
    {
        Direct = new
        {
            Transient = transientDirect.Id,
            Scoped = scopedDirect.Id,
            Singleton = singletonDirect.Id
        },
        ViaReporter = new
        {
            Transient = reporter.TransientId,
            Scoped = reporter.ScopedId,
            Singleton = reporter.SingletonId
        }
    });
});
```

### Ожидаемый вывод для двух запросов подряд

Вызов `GET /api/lifetime-demo` дважды даёт примерно такую картину (реальные Guid будут другими, важна структура совпадений):

| | Request 1 — Direct | Request 1 — ViaReporter | Request 2 — Direct | Request 2 — ViaReporter |
|---|---|---|---|---|
| **Transient** | `aaa1...` | `aaa2...` | `bbb1...` | `bbb2...` |
| **Scoped** | `ccc1...` | `ccc1...` (тот же!) | `ddd1...` | `ddd1...` (тот же!) |
| **Singleton** | `eee1...` | `eee1...` (тот же!) | `eee1...` (тот же!) | `eee1...` (тот же!) |

Что это доказывает:

- **Transient** — все четыре значения разные: даже внутри одного запроса `transientDirect.Id` и `reporter.TransientId` не совпадают, потому что каждое разрешение создаёт новый объект.
- **Scoped** — внутри одного запроса `scopedDirect.Id == reporter.ScopedId` (один scope — один инстанс), но между запросом 1 и запросом 2 значения разные (новый scope на новый запрос).
- **Singleton** — одно и то же значение везде и всегда, пока процесс жив.

> [!tip] Быстрый тест на собеседовании
> Если вас просят «на бумаге» показать разницу lifetime — рисуйте именно эту таблицу 3×4 (или 3×2, если без reporter). Это ровно то, что интервьюер хочет увидеть: понимание, что «scope» — это не «запрос» по определению, а единица, для которой запрос — лишь самый частый (но не единственный) триггер создания.

## Кто вызывает `Dispose` и когда

Контейнер ASP.NET Core (`Microsoft.Extensions.DependencyInjection`) отслеживает все объекты, реализующие `IDisposable`/`IAsyncDisposable`, которые **он сам создал** через фабрику/конструктор. Правила освобождения:

1. **Transient и Scoped**, созданные внутри scope запроса, — освобождаются в конце этого scope, то есть в конце HTTP-запроса (когда middleware-конвейер завершает `RequestServices`-scope).
2. **Transient**, созданный из **корневого** провайдера (например, инжектирован напрямую в другой Singleton — что само по себе допустимо, в отличие от Scoped) — будет жить и освобождён только при остановке приложения, потому что у него нет «своего» короткого scope.
3. **Singleton** — освобождается при `Dispose`/`DisposeAsync` корневого `IServiceProvider`, то есть при штатном graceful shutdown хоста.

```csharp
public sealed class DisposableScopedService : IScopedService, IDisposable
{
    private readonly ILogger<DisposableScopedService> _logger;
    public Guid Id { get; } = Guid.NewGuid();

    public DisposableScopedService(ILogger<DisposableScopedService> logger)
        => _logger = logger;

    public void Dispose()
        => _logger.LogInformation("Scoped {Id} disposed at end of scope", Id);
}
```

> [!warning] Контейнер не освобождает то, что не создавал
> Три ситуации, где `Dispose` **не будет вызван контейнером**, и это по спецификации, а не баг:
> - `services.AddSingleton<IThing>(existingInstance)` — экземпляр передан готовым, контейнер им не владеет.
> - Объект создан вручную внутри вашего кода (`new SomeDisposable()`) и просто использован — DI тут вообще ни при чём.
> - Вы сами вытащили сервис через `IServiceProvider.GetService` из **корневого** провайдера в обход scope и держите ссылку дольше, чем нужно — формально контейнер его освободит при остановке приложения, но не тогда, когда вы ожидаете.
>
> Правило простое: **DI освобождает то, что сам создал, ровно в конце того scope, в котором создал**. Если вы регистрируете готовый инстанс — освобождение остаётся на вас (или на `IHostApplicationLifetime.ApplicationStopping`).

## `IHostedService`/`BackgroundService` и lifetime

`BackgroundService` регистрируется через `AddHostedService<T>()`, и хост резолвит его **один раз, как Singleton**, при старте приложения. Это значит: **инжектировать Scoped или Transient-с-состоянием-запроса прямо в конструктор `BackgroundService` — ошибка**, даже если компилятор и DI-контейнер её не поймают на старте (поймают только при включённой валидации scope, см. ниже, и то не всегда).

Неправильно:

```csharp
public sealed class OrderCleanupService : BackgroundService
{
    private readonly IOrderRepository _repository; // Scoped!

    public OrderCleanupService(IOrderRepository repository) // будет захвачен навсегда
        => _repository = repository;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await _repository.DeleteStaleOrdersAsync(stoppingToken); // работает с "протухшим" DbContext
            await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);
        }
    }
}
```

`_repository` внутри держит `DbContext`, созданный один раз при старте хоста и никогда не освобождаемый — тот же `DbContext` будет использоваться час за часом, накапливая отслеживаемые сущности в change tracker и рано или поздно упадёт с потокобезопасностью или утечкой памяти.

Правильно — инжектировать `IServiceScopeFactory` и создавать новый scope на каждую единицу работы:

```csharp
public sealed class OrderCleanupService : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<OrderCleanupService> _logger;

    public OrderCleanupService(
        IServiceScopeFactory scopeFactory,
        ILogger<OrderCleanupService> logger)
    {
        _scopeFactory = scopeFactory;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                using var scope = _scopeFactory.CreateScope();
                var repository = scope.ServiceProvider.GetRequiredService<IOrderRepository>();
                await repository.DeleteStaleOrdersAsync(stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Order cleanup iteration failed");
            }

            await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);
        }
    }
}
```

Каждая итерация получает свежий `DbContext` с чистым change tracker'ом и корректно освобождает его в конце `using`. Подробнее про сам механизм фоновых задач — в [[Background services и IHostedService]].

## `DbContext` и Scoped — почему именно так

`AddDbContext<TContext>()` по умолчанию регистрирует контекст как **Scoped**, и это не случайный выбор:

- `DbContext` **не потокобезопасен** — внутри у него мутабельное состояние (change tracker, активное соединение, транзакция). Если сделать его Singleton, конкурентные запросы начнут одновременно читать/писать одно и то же соединение → `InvalidOperationException: A second operation was started on this context before a previous operation completed`.
- Scoped даёт ровно нужную гранулярность: один контекст на один HTTP-запрос — change tracker живёт столько же, сколько бизнес-транзакция запроса, и корректно освобождается в конце.

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseNpgsql(connectionString)); // по умолчанию ServiceLifetime.Scoped
```

> [!danger] Что сломается, если сделать `DbContext` Singleton
> ```csharp
> // НЕ ДЕЛАТЬ ТАК
> builder.Services.AddDbContext<AppDbContext>(options => options.UseNpgsql(cs),
>     contextLifetime: ServiceLifetime.Singleton);
> ```
> Один физический `DbContext`, а значит одно соединение и один change tracker, на все параллельные запросы приложения. Результат: гонки при `SaveChangesAsync`, случайное «протекание» сущностей одного пользователя в запрос другого через shared change tracker, а в лучшем случае — просто исключение о повторном запуске операции.

Когда нужно несколько **коротких, независимых** контекстов внутри одного scope — например, параллельные запросы к БД в рамках одного HTTP-запроса, или Blazor Server, где один Circuit живёт долго и обычный Scoped-`DbContext` был бы слишком «долгоживущим» — используют `IDbContextFactory<TContext>`:

```csharp
builder.Services.AddDbContextFactory<AppDbContext>(options =>
    options.UseNpgsql(connectionString));

app.MapGet("/api/dashboard", async (IDbContextFactory<AppDbContext> factory) =>
{
    await using var ordersCtx = await factory.CreateDbContextAsync();
    await using var usersCtx = await factory.CreateDbContextAsync();

    var ordersTask = ordersCtx.Orders.CountAsync();
    var usersTask = usersCtx.Users.CountAsync();
    await Task.WhenAll(ordersTask, usersTask); // безопасно: два разных DbContext

    return Results.Ok(new { Orders = ordersTask.Result, Users = usersTask.Result });
});
```

`IDbContextFactory<T>` сама регистрируется как Singleton (она лишь фабрика, не хранит состояние запроса), а каждый `CreateDbContextAsync()` возвращает новый короткоживущий `DbContext`, за освобождение которого отвечаете вы сами (`await using`). Подробнее про сам EF Core и его особенности — [[08 — Данные и EF Core (обзор раздела)]].

## `IHttpContextAccessor` — Singleton, который отдаёт per-request данные

`IHttpContextAccessor` регистрируется как Singleton:

```csharp
builder.Services.AddHttpContextAccessor();
```

На первый взгляд это противоречит правилу «Singleton не должен держать per-request состояние» — но `IHttpContextAccessor` ничего не держит по значению. Внутри он использует `AsyncLocal<HttpContext>`: значение хранится не в самом объекте, а в **логическом контексте текущего потока выполнения (execution context)**, который .NET автоматически прокидывает через `await`, `Task.Run` и т.п., но не расшаривает между параллельными цепочками вызовов. Поэтому один и тот же Singleton-объект безопасно возвращает *разный* `HttpContext` в зависимости от того, из какого запроса вызван `.HttpContext`.

```csharp
public sealed class CurrentUserService : ICurrentUserService
{
    private readonly IHttpContextAccessor _accessor;

    public CurrentUserService(IHttpContextAccessor accessor) => _accessor = accessor;

    public Guid? UserId =>
        _accessor.HttpContext?.User.FindFirst("sub") is { } claim
            ? Guid.Parse(claim.Value)
            : null;
}
```

> [!warning] Когда `IHttpContextAccessor` — code smell
> - Если вы тянете `IHttpContextAccessor` в доменный/сервисный слой, который в принципе не должен знать про HTTP — это утечка веб-абстракции в бизнес-логику. Правильнее явно передавать нужные данные (userId, tenant) параметром или через выделенный Scoped-сервис (`ICurrentUserService`), который сам читает `HttpContext` один раз на границе.
> - `HttpContext` вне активного запроса (например, в `BackgroundService` или после `await` в fire-and-forget задаче, не привязанной к запросу) — `null`, потому что `AsyncLocal` не «убегает» за пределы обработки запроса. Код, который не проверяет на `null` и предполагает, что `HttpContext` всегда есть, падает с `NullReferenceException` в проде при первом же фоновом сценарии.
> - Держать `IHttpContextAccessor.HttpContext` в локальной переменной и использовать её после реального завершения запроса (например, в `Task.Run` без ожидания) — обращение к уже освобождённому/переиспользованному объекту.

## Валидация scope: `ValidateScopes` и `ValidateOnBuild`

`WebApplication.CreateBuilder(...)` включает валидацию lifetime **автоматически в Development**:

```csharp
var builder = WebApplication.CreateBuilder(args);
// эквивалент того, что билдер делает под капотом в Development:
builder.Host.UseDefaultServiceProvider((context, options) =>
{
    options.ValidateScopes = context.HostingEnvironment.IsDevelopment();
    options.ValidateOnBuild = context.HostingEnvironment.IsDevelopment();
});
```

- **`ValidateScopes = true`** — контейнер бросает исключение при попытке резолвить Scoped-сервис **из корневого провайдера** (то есть вне HTTP-запроса/вне явного scope) и, что важнее для нашей темы, при попытке инжектировать Scoped в Singleton через конструктор — потому что построение Singleton происходит в корневом scope.
- **`ValidateOnBuild = true`** — контейнер проверяет весь граф регистраций **сразу при старте**, а не лениво при первом обращении: неправильная композиция (например, не зарегистрированная зависимость) обнаруживается сразу, а не когда до неё дойдёт первый запрос в проде.

> [!danger] В Production валидация выключена по умолчанию
> Это сделано ради производительности: проверка графа при каждом резолве стоит времени. Следствие — баг с captive dependency может пройти незамеченным в Development (если сборка графа не триггерит нужный путь) и рвануть только в проде под нагрузкой, либо наоборот — молча закешировать протухшие данные без единого исключения. Практика: держите интеграционный тест, который строит `IServiceProvider` с `ValidateScopes = true, ValidateOnBuild = true` явно и резолвит все зарегистрированные сервисы — это ловит captive dependency на CI, а не на проде.

## Captive dependency: полный разбор механизма

Captive dependency — ситуация, когда сервис с **более долгим** lifetime получает через конструктор ссылку на сервис с **более коротким** lifetime, и эта ссылка «утекает» за пределы того времени, для которого короткоживущий сервис был спроектирован.

Буквальный пример: Singleton-кеш, который зачем-то тянет Scoped-репозиторий:

```csharp
public interface IUserRepository // Scoped, использует DbContext
{
    Task<User?> FindAsync(Guid id);
}

public sealed class InMemoryUserCache : IUserCache // задуман как Singleton
{
    private readonly IUserRepository _repository; // ОШИБКА: Scoped внутри Singleton
    private readonly ConcurrentDictionary<Guid, User> _cache = new();

    public InMemoryUserCache(IUserRepository repository)
        => _repository = repository; // repository "застревает" здесь навсегда

    public async Task<User?> GetAsync(Guid id)
    {
        if (_cache.TryGetValue(id, out var cached)) return cached;

        var user = await _repository.FindAsync(id); // использует DbContext из ПЕРВОГО запроса,
                                                       // создавшего этот Singleton
        if (user is not null) _cache[id] = user;
        return user;
    }
}
```

```csharp
builder.Services.AddScoped<IUserRepository, UserRepository>();
builder.Services.AddSingleton<IUserCache, InMemoryUserCache>(); // конфликт lifetime
```

Что происходит физически:

1. При первом резолве `IUserCache` контейнер строит граф зависимостей и обнаруживает, что нужен `IUserRepository`.
2. Если Singleton резолвится из **корневого** провайдера (а он резолвится именно так — Singleton'ы живут в корневом scope), а `IUserRepository` зарегистрирован как Scoped — с `ValidateScopes = true` вы немедленно получите:
   ```
   System.InvalidOperationException: Cannot consume scoped service
   'IUserRepository' from singleton 'IUserCache'.
   ```
3. Если валидация выключена (типично для Production), исключения не будет: контейнер молча создаст Scoped-экземпляр `IUserRepository` **в корневом scope** (а не в scope запроса) и передаст в конструктор Singleton. Этот экземпляр `IUserRepository`, а внутри него — `DbContext`, никогда не будет освобождён (кроме как при остановке приложения) и будет использоваться **всеми последующими запросами**, хотя формально зарегистрирован как Scoped. Получается «сломанный синглтон»: `DbContext` копит отслеживаемые сущности вечно, а разные пользовательские запросы делят одно соединение — те же проблемы потокобезопасности, что при явном `AddDbContext(..., ServiceLifetime.Singleton)`, только незаметные при чтении кода.

Фикс — не давать Singleton владеть Scoped-ссылкой напрямую. Инжектировать `IServiceScopeFactory` (или `IServiceProvider`) и создавать scope **на каждый вызов**:

```csharp
public sealed class InMemoryUserCache : IUserCache
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ConcurrentDictionary<Guid, User> _cache = new();

    public InMemoryUserCache(IServiceScopeFactory scopeFactory)
        => _scopeFactory = scopeFactory;

    public async Task<User?> GetAsync(Guid id)
    {
        if (_cache.TryGetValue(id, out var cached)) return cached;

        using var scope = _scopeFactory.CreateScope();
        var repository = scope.ServiceProvider.GetRequiredService<IUserRepository>();
        var user = await repository.FindAsync(id); // свежий DbContext на каждый промах кеша

        if (user is not null) _cache[id] = user;
        return user;
    }
}
```

Теперь `IServiceScopeFactory` сам по себе Singleton-safe (это фабрика без состояния), а каждый `CreateScope()` даёт независимый короткоживущий `DbContext`, который освобождается сразу после использования.

> [!info] Это только механика — таксономия ошибок дальше
> Captive dependency — не единственная ловушка DI-регистраций: есть ещё Service Locator анти-паттерн, циклические зависимости, неправильное использование `IEnumerable<T>` и другие частые промахи. Полный разбор с примерами каждого — в [[Captive dependency и типичные ошибки DI]]; здесь важно унести именно механизм — **почему** Scoped внутри Singleton опасен и как его лечит `IServiceScopeFactory`.

## Таблица: инжекция сервиса с одним lifetime в сервис с другим

Правило одной фразой: **безопасно внедрять сервис с lifetime не короче, чем у потребителя** (или через `IServiceScopeFactory`/`IServiceProvider`, если короче). «Короче» = Transient < Scoped < Singleton по времени жизни экземпляра.

| Кого внедряем ↓ \ Куда внедряем → | в Transient | в Scoped | в Singleton |
|---|---|---|---|
| **Transient** | ✅ безопасно — оба создаются заново, поведение ожидаемо | ✅ безопасно — transient-экземпляр живёт не дольше scope-потребителя | ⚠️ **опасно**: transient, захваченный Singleton-ом, фактически живёт как Singleton (не создаётся заново); работает без исключений, но ломает ожидание «новый объект каждый раз» |
| **Scoped** | ⚠️ **некорректно с точки зрения контракта**: Transient не имеет собственного scope, поэтому Scoped-зависимость в нём фактически привязывается к scope, в котором резолвился сам Transient — обычно совпадает со scope запроса, но легко перепутать | ✅ безопасно и типично — оба живут ровно один scope | ❌ **captive dependency**: с `ValidateScopes=true` — исключение при старте; без валидации — тихая утечка и протухшее состояние навсегда |
| **Singleton** | ✅ безопасно — Singleton переживёт Transient без проблем, просто разделяется между всеми | ✅ безопасно — Singleton можно свободно использовать из любого scope, он один для всех | ✅ безопасно — оба живут вечно, состояние симметрично |

> [!tip] Мнемоника
> Смотрите на диагональ и ниже неё (Singleton→Singleton, Scoped→Scoped, Transient→Transient, и всё что *длиннее* внедряется в *короче живущее*) — там всё нормально. Проблема начинается, когда **короткое время жизни внедряется в то, что живёт дольше** и потребитель держит ссылку в поле, а не резолвит заново на каждый вызов.

## Вопросы с собеседований

> [!question]- В чём разница между Scoped и Transient, если оба «не живут вечно»?
> Transient создаёт новый экземпляр на **каждое** разрешение зависимости, даже если два запроса на этот же интерфейс происходят в рамках одного HTTP-запроса. Scoped создаёт один экземпляр **на весь scope** — все, кто просит этот интерфейс в пределах одного HTTP-запроса (или другого явного scope), получают один и тот же объект. Проверяется это ровно тем демонстрационным примером с `Guid`: у Transient два разных Guid внутри одного запроса, у Scoped — один.

> [!question]- Что такое scope в терминах ASP.NET Core, если не «HTTP-запрос»?
> Scope — это `IServiceScope`, обёртка над дочерним `IServiceProvider`, которая кеширует Scoped-инстансы и отслеживает их для освобождения. ASP.NET Core создаёт scope на каждый HTTP-запрос автоматически — это самый частый случай, но не единственный: scope можно создать вручную через `IServiceScopeFactory.CreateScope()` где угодно, включая консольные приложения, фоновые задачи, обработчики сообщений очереди — везде, где нет запроса, но нужна изоляция состояния, аналогичная одному запросу.

> [!question]- Кто вызывает `Dispose` у ваших сервисов и всегда ли это делает контейнер?
> Контейнер вызывает `Dispose`/`DisposeAsync` только у объектов, которые **сам создал** через фабрику/конструктор, и делает это в конце того scope, в котором объект был создан: Scoped/Transient — в конце HTTP-запроса, Singleton — при остановке приложения (`Dispose` корневого `IServiceProvider`). Исключение: если сервис зарегистрирован готовым инстансом через `AddSingleton<T>(instance)`, контейнер не считает себя владельцем и никогда не вызовет `Dispose` — эта ответственность остаётся на вызывающем коде.

> [!question]- Почему инжекция `IOrderRepository` (Scoped) прямо в конструктор `BackgroundService` — ошибка?
> `BackgroundService` регистрируется через `AddHostedService` и резолвится хостом один раз при старте приложения, фактически как Singleton. Если в его конструктор попадёт Scoped-зависимость, она будет создана один раз в корневом scope и захвачена навсегда (captive dependency) — например, `DbContext` внутри репозитория будет использоваться час за часом без освобождения, копя отслеживаемые сущности и рискуя потокобезопасностью. Правильно — инжектировать `IServiceScopeFactory` и создавать новый scope на каждую единицу фоновой работы, резолвя Scoped-зависимости внутри него.

> [!question]- Почему `AddDbContext` регистрирует `DbContext` как Scoped, а не Singleton или Transient?
> `DbContext` хранит мутабельное состояние — change tracker и активное соединение — и не потокобезопасен. Singleton означало бы один физический контекст на все параллельные запросы приложения: гонки, `InvalidOperationException` о повторном запуске операции, смешение отслеживаемых сущностей разных пользователей. Transient создавал бы новый контекст на каждое разрешение зависимости, что внутри одного запроса раздробило бы единицу работы на несвязанные контексты без общей транзакции/change tracker. Scoped даёт ровно нужную гранулярность: один контекст на весь HTTP-запрос, синхронизированный с бизнес-транзакцией и корректно освобождаемый в конце.

> [!question]- Как безопасно получить несколько независимых `DbContext` в рамках одного запроса, например для параллельных запросов к БД?
> Через `IDbContextFactory<TContext>`, зарегистрированный `AddDbContextFactory<TContext>()`. Сама фабрика — Singleton, но каждый вызов `CreateDbContextAsync()` возвращает новый короткоживущий `DbContext`, который вы явно освобождаете (`await using`). Это позволяет запустить несколько независимых запросов параллельно (`Task.WhenAll`) без нарушения потокобезопасности одного контекста — то, что нельзя сделать с обычным Scoped-`DbContext`, потому что он один на весь запрос. Тот же механизм используется в Blazor Server, где Circuit живёт дольше одного «логического запроса».

> [!question]- `IHttpContextAccessor` зарегистрирован как Singleton — как он тогда отдаёт разный `HttpContext` для разных запросов без гонок?
> Он не хранит `HttpContext` как обычное поле объекта — внутри используется `AsyncLocal<HttpContext>`, значение которого привязано к логическому execution context текущей асинхронной цепочки вызовов, а не к экземпляру класса. .NET runtime автоматически копирует/восстанавливает это значение через `await`, `Task.Run` и другие точки продолжения, но не расшаривает между параллельными, несвязанными цепочками. Поэтому один физический Singleton-объект безопасно возвращает разный `HttpContext` в зависимости от того, из какого запроса (какой асинхронной цепочки) к нему обратились.

> [!question]- Объясните captive dependency на примере и как это ловит `ValidateScopes`
> Captive dependency — когда сервис с более долгим lifetime (обычно Singleton) получает через конструктор Scoped- или Transient-с-состоянием зависимость и держит ссылку на неё в поле дольше, чем предполагалось. Пример: Singleton-кеш с `IUserRepository` (Scoped, использует `DbContext`) в конструкторе — репозиторий и его `DbContext` будут созданы один раз в корневом scope и использоваться вечно всеми последующими запросами, копя состояние и рискуя потокобезопасностью. `ServiceProviderOptions.ValidateScopes = true` (включено по умолчанию в Development через `WebApplication.CreateBuilder`) ловит это на старте: контейнер бросает `InvalidOperationException: Cannot consume scoped service ... from singleton ...`, потому что построение Singleton происходит в корневом scope, где Scoped-сервисы разрешать нельзя. Без валидации (по умолчанию в Production) ошибки не будет — контейнер молча создаст Scoped-экземпляр в корневом scope, и баг проявится только как протухшие данные или гонки под нагрузкой. Фикс — не хранить Scoped-зависимость в поле Singleton, а инжектировать `IServiceScopeFactory` и резолвить нужный сервис внутри свежего scope на каждый вызов.

> [!question]- Что произойдёт, если внедрить Transient-сервис в Singleton?
> Формально исключения не будет ни при какой настройке валидации — Transient можно резолвить из любого scope, включая корневой. Но поведение будет неожиданным: Singleton резолвит Transient-зависимость один раз, при своём собственном создании, и держит эту единственную инстанцию в поле всё время жизни приложения. Контракт «Transient — новый объект на каждое разрешение» фактически нарушается: с точки зрения потребителя объект ведёт себя как ещё один Singleton. Если Transient-сервис задумывался как stateless-хелпер — вреда почти нет; если у него есть скрытое состояние (например, счётчик или неявный кеш) — это тихий баг.

## Итог

- **Transient** — новый экземпляр на каждое разрешение зависимости; **Scoped** — один экземпляр на scope (по умолчанию — один HTTP-запрос, но scope можно создать вручную через `IServiceScopeFactory`); **Singleton** — один экземпляр на весь процесс.
- Контейнер вызывает `Dispose` только у того, что сам создал, и ровно в конце того scope, где создал; `AddSingleton(instance)` — исключение: контейнер не считает себя владельцем и не освобождает.
- `BackgroundService` резолвится как Singleton — Scoped-зависимости внутрь него нужно получать через `IServiceScopeFactory.CreateScope()` на каждую единицу работы, а не через конструктор.
- `DbContext` — Scoped по умолчанию из-за потокобезопасности; Singleton ломает конкурентность, `IDbContextFactory<T>` — способ получить несколько коротких контекстов внутри одного scope.
- `IHttpContextAccessor` — Singleton, но безопасен благодаря `AsyncLocal`; тянуть его в бизнес-логику — code smell.
- `ValidateScopes`/`ValidateOnBuild` включены по умолчанию в Development и ловят captive dependency на старте — но выключены в Production, поэтому нужен явный тест на CI.
- Captive dependency: короткоживущая зависимость, захваченная более долгоживущим потребителем через конструктор, живёт дольше, чем должна, и превращается в источник протухшего состояния или гонок. Лечится инжекцией `IServiceScopeFactory`/`IServiceProvider` вместо самой зависимости.

## Связанное

- [[Dependency Injection: контейнер ASP.NET Core]]
- [[Captive dependency и типичные ошибки DI]]
- [[Middleware и конвейер обработки запроса]]
- [[Background services и IHostedService]]
- [[08 — Данные и EF Core (обзор раздела)]]
- [[Health checks]]
- [[Options pattern и конфигурация сервисов]]
- [[Keyed services и продвинутая регистрация]]
