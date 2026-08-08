---
tags: [раздел-08, ef-core, orm, dbcontext, основы, middle, собес, dotnet10]
aliases: [DbContext, EF Core intro, Entity Framework Core, ORM в .NET]
---

# EF Core: введение и DbContext

> [!abstract] Коротко
> EF Core — объектно-реляционный маппер (object-relational mapper, ORM): он переводит LINQ-запросы в SQL и материализует строки в объекты. Центр всего — `DbContext`: одновременно Unit of Work (накапливает изменения и пишет их одной транзакцией) и Identity Map (гарантирует, что одна строка БД = один объект в памяти). Контекст не потокобезопасен и живёт ровно один запрос/операцию — это главное правило, из которого вытекает всё остальное.

## Зачем это нужно

До ORM код доступа к данным выглядел так (см. [[ADO.NET: как всё устроено под капотом]]):

```csharp
await using var conn = new NpgsqlConnection(cs);
await conn.OpenAsync();
await using var cmd = new NpgsqlCommand("SELECT id, name, balance FROM accounts WHERE id = @id", conn);
cmd.Parameters.AddWithValue("id", id);
await using var reader = await cmd.ExecuteReaderAsync();
if (!await reader.ReadAsync()) return null;
var account = new Account
{
    Id = reader.GetGuid(0),
    Name = reader.GetString(1),
    Balance = reader.GetDecimal(2)
};
```

Двадцать строк ради одной сущности. Умножь на 200 таблиц и на CRUD для каждой — получишь тысячи строк рутины, где каждая опечатка в индексе колонки даёт ошибку в рантайме, а не при компиляции.

ORM решает три задачи:
1. **Маппинг** — описываешь соответствие класс↔таблица один раз, дальше материализация автоматическая.
2. **Генерация SQL** — LINQ-выражение транслируется в SQL, компилятор проверяет имена свойств.
3. **Отслеживание изменений** — меняешь свойства объекта, EF сам вычисляет `UPDATE` только по изменённым колонкам.

### Цена ORM

За удобство платят. Честный список:

| Проблема | В чём суть |
|---|---|
| Утечка абстракции | «Это же просто объекты» — до тех пор, пока `.Where()` внутри цикла не даст 10 000 запросов. Придётся знать и SQL, и EF |
| Непрозрачный SQL | Между твоим кодом и БД — генератор. Иногда он выдаёт неожиданно плохой план |
| Overhead материализации | Change tracking, снапшоты оригинальных значений, построение графов объектов — это CPU и аллокации |
| Object–relational impedance mismatch | Наследование, коллекции, значение vs ссылка — всё это плохо ложится на таблицы |
| Соблазн модели «anemic CRUD» | ORM подталкивает к таблице-на-класс, а не к нормальному доменному дизайну |

> [!quote] Правило
> ORM — это инструмент для 90 % запросов, которые скучные. Оставшиеся 10 % горячих/сложных пиши руками: [[Dapper: когда микро-ORM лучше]] или [[EF Core: сырой SQL и хранимые процедуры]]. Полиглотный доступ к данным — норма, а не поражение.

## Что такое DbContext

`DbContext` реализует сразу два паттерна.

**Unit of Work.** Контекст копит изменения в памяти и применяет их одним вызовом `SaveChanges()`, обёрнутым в одну транзакцию БД. Либо всё, либо ничего.

**Identity Map.** Внутри одного контекста строка с данным первичным ключом материализуется ровно один раз. Второй запрос той же строки вернёт **тот же самый объект** (по ссылке).

```csharp
var a = await db.Accounts.FirstAsync(x => x.Id == id);
var b = await db.Accounts.FirstAsync(x => x.Id == id);
Console.WriteLine(ReferenceEquals(a, b)); // Вывод: True — SQL выполнился дважды,
                                          // но второй результат отброшен в пользу уже отслеживаемого объекта
```

Отсюда важное следствие: **данные из второго запроса игнорируются**. Если между запросами строку в БД изменил кто-то другой, ты этого не увидишь — в объекте останутся старые значения. Это не баг, это осознанный выбор: иначе EF затирал бы твои несохранённые правки. Подробнее — [[EF Core: change tracking и AsNoTracking]].

```
┌─────────────────────────────────────────────────────────────┐
│                        DbContext                            │
│                                                             │
│  ┌───────────────┐   ┌──────────────────────────────────┐   │
│  │  Model        │   │  ChangeTracker                   │   │
│  │  (IModel)     │   │  ┌────────────────────────────┐  │   │
│  │  singleton,   │   │  │ EntityEntry: Account#1     │  │   │
│  │  кешируется   │   │  │  State = Modified          │  │   │
│  │  на процесс   │   │  │  Current / Original values │  │   │
│  └───────────────┘   │  └────────────────────────────┘  │   │
│                      │  ┌────────────────────────────┐  │   │
│  ┌───────────────┐   │  │ EntityEntry: Payment#7     │  │   │
│  │  DbSet<T>     │   │  │  State = Added             │  │   │
│  │  точки входа  │   │  └────────────────────────────┘  │   │
│  │  в запросы    │   └──────────────────────────────────┘   │
│  └───────────────┘                                          │
│                      ┌──────────────────────────────────┐   │
│                      │  DatabaseFacade → DbConnection   │   │
│                      └──────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                                   │ SaveChangesAsync()
                                   ▼
                     BEGIN; UPDATE...; INSERT...; COMMIT;
```

### DbSet<T>

`DbSet<T>` — это `IQueryable<T>` плюс методы записи (`Add`, `Remove`, `Update`, `Attach`, `Find`). Он не хранит данные: это точка входа, из которой строится дерево выражений (см. [[IEnumerable vs IQueryable]] и [[Деревья выражений]]).

```csharp
public sealed class LedgerContext(DbContextOptions<LedgerContext> options) : DbContext(options)
{
    public DbSet<Account> Accounts => Set<Account>();
    public DbSet<Payment> Payments => Set<Payment>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
        => modelBuilder.ApplyConfigurationsFromAssembly(typeof(LedgerContext).Assembly);
}
```

> [!tip] `=> Set<T>()` вместо `{ get; set; }`
> Свойство-выражение через `Set<T>()` избавляет от предупреждений nullable-анализа (`DbSet` никогда не null) и не даёт присвоить `DbSet` извне. Функционально идентично.

### Find и FindAsync

`Find` сначала смотрит в Identity Map и только при промахе идёт в БД. Это единственный API EF, который умеет ответить без запроса:

```csharp
var a1 = await db.Accounts.FindAsync(id); // SELECT ...
var a2 = await db.Accounts.FindAsync(id); // запроса нет, объект из трекера
```

`FirstOrDefaultAsync(x => x.Id == id)` всегда идёт в БД. Разница заметна в циклах и в обработчиках, где один и тот же агрегат читают несколько раз.

## Регистрация в DI

```csharp
builder.Services.AddDbContext<LedgerContext>(opt =>
    opt.UseNpgsql(builder.Configuration.GetConnectionString("Ledger"), npg =>
        {
            npg.EnableRetryOnFailure(maxRetryCount: 3);
            npg.MigrationsHistoryTable("__ef_migrations_history", "ledger");
        })
       .UseSnakeCaseNamingConvention());   // из EFCore.NamingConventions — принято в мире PostgreSQL
```

`AddDbContext` регистрирует контекст как **Scoped**. В ASP.NET Core scope = HTTP-запрос, значит один запрос — один контекст — один Unit of Work. Это ровно то, что нужно: все изменения за запрос уходят одной транзакцией.

### Почему именно Scoped

| Время жизни | Что будет |
|---|---|
| Transient | Каждая инъекция — свой контекст. Два сервиса в одном запросе не увидят изменений друг друга, `SaveChanges` в разных транзакциях. Обычно ошибка |
| **Scoped** | Один на запрос. Правильный дефолт |
| Singleton | Катастрофа: контекст не потокобезопасен, ChangeTracker растёт бесконечно (утечка памяти), одно соединение на всё приложение |

Классическая ловушка — singleton-сервис, которому в конструктор внедрили `DbContext`: контекст «залипает» в синглтоне на весь срок жизни приложения. Это [[Captive dependency и типичные ошибки DI]]. См. также [[Жизненные циклы сервисов: Singleton, Scoped, Transient]].

> [!danger] DbContext не потокобезопасен
> Ни один экземпляр `DbContext` не поддерживает параллельные операции. Забытый `await` — и получаешь
> `InvalidOperationException: A second operation was started on this context instance before a previous operation completed`.
> Особенно легко напороться так:
> ```csharp
> // НЕПРАВИЛЬНО: параллельные запросы к одному контексту
> var tasks = ids.Select(id => db.Accounts.FirstAsync(a => a.Id == id));
> var result = await Task.WhenAll(tasks);   // взрыв
> ```
> Правильно — либо один запрос `Where(a => ids.Contains(a.Id))`, либо контекст на задачу через фабрику.

## Пулинг контекстов: AddDbContextPool

Создание `DbContext` — не бесплатно: аллокация внутренней инфраструктуры (сервис-провайдер контекста, ChangeTracker, StateManager). При десятках тысяч RPS это заметно.

```csharp
builder.Services.AddDbContextPool<LedgerContext>(
    opt => opt.UseNpgsql(cs),
    poolSize: 1024);           // по умолчанию 1024
```

Пул переиспользует **экземпляры контекста**: при `Dispose()` контекст не уничтожается, а сбрасывается (`ChangeTracker.Clear()`, сброс состояния) и возвращается в пул.

> [!warning] Ограничения пулинга
> - Конструктор контекста должен принимать **только** `DbContextOptions` (или `DbContextOptions<T>`). Никаких других зависимостей — иначе они «застрянут» в переиспользуемом экземпляре.
> - Значит, паттерн «мультитенантность через `tenantId` в конструкторе» с пулингом напрямую не работает. Обходят через `IDbContextFactory` либо через scoped-сервис-холдер, читаемый из фильтра ([[EF Core: глобальные и именованные фильтры]]).
> - Собственное состояние в контексте (`private List<DomainEvent> _events`) переживёт возврат в пул. Сбрасывай его вручную, переопределив `OnDisposing`/используя `IResettableService`.
> - Выигрыш реален только на действительно высоких нагрузках. Не начинай с пулинга — начни с профиля.

Пулинг контекстов — это НЕ [[Пулинг соединений]]. Соединения к PostgreSQL пулятся драйвером Npgsql независимо и по умолчанию. Это два разных пула на разных уровнях.

## IDbContextFactory: контекст вне HTTP-запроса

Фоновые сервисы ([[Background services и IHostedService]]) — синглтоны. Внедрить в них scoped-контекст нельзя. Два выхода.

**Вариант 1. Создавать scope вручную:**

```csharp
public sealed class SettlementWorker(IServiceScopeFactory scopeFactory) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            using var scope = scopeFactory.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<LedgerContext>();
            await ProcessBatchAsync(db, ct);
            await Task.Delay(TimeSpan.FromSeconds(10), ct);
        }
    }
}
```

**Вариант 2. Фабрика контекстов** — когда нужен именно контекст, а не весь scope, или когда нужно несколько контекстов одновременно:

```csharp
builder.Services.AddDbContextFactory<LedgerContext>(opt => opt.UseNpgsql(cs));

public sealed class ReportBuilder(IDbContextFactory<LedgerContext> factory)
{
    public async Task<Report> BuildAsync(CancellationToken ct)
    {
        // Параллельные независимые запросы — каждому свой контекст
        var totalsTask = SumAsync(ct);
        var countsTask = CountAsync(ct);
        await Task.WhenAll(totalsTask, countsTask);
        return new Report(await totalsTask, await countsTask);

        async Task<decimal> SumAsync(CancellationToken t)
        {
            await using var db = await factory.CreateDbContextAsync(t);
            return await db.Payments.SumAsync(p => p.Amount, t);
        }

        async Task<int> CountAsync(CancellationToken t)
        {
            await using var db = await factory.CreateDbContextAsync(t);
            return await db.Payments.CountAsync(t);
        }
    }
}
```

`AddDbContextFactory` регистрирует фабрику как singleton, а создаваемые контексты нужно диспозить самому. Если нужны обе регистрации (scoped-контекст для запросов + фабрика для фона), используй:

```csharp
builder.Services.AddDbContextFactory<LedgerContext>(opt => opt.UseNpgsql(cs),
    lifetime: ServiceLifetime.Scoped);
// или
builder.Services.AddPooledDbContextFactory<LedgerContext>(opt => opt.UseNpgsql(cs));
```

## Жизненный цикл: полный путь запроса

```
1. HTTP-запрос → DI создаёт scope → создаётся (или берётся из пула) DbContext
2. Первый запрос к БД → берётся соединение из пула Npgsql
3. Материализация → объекты попадают в ChangeTracker (если трекинг включён)
4. Бизнес-логика меняет свойства объектов
5. SaveChangesAsync():
      DetectChanges() → сравнение с оригинальными значениями
      построение UPDATE/INSERT/DELETE, батчинг
      BEGIN; ... ; COMMIT;
      состояния Added/Modified → Unchanged
6. Конец запроса → scope Dispose → контекст Dispose → соединение в пул
```

Соединение к БД EF держит **не всё время жизни контекста**, а только на время выполнения команды (кроме случая явной транзакции — тогда до `Commit`/`Rollback`). Это важно: длинный контекст сам по себе не удерживает коннект.

> [!warning] Подводные камни
> - **`DbContext` — не репозиторий на всё приложение.** Долгоживущий контекст = растущий ChangeTracker = замедление `DetectChanges` (он O(число отслеживаемых сущностей)) и утечка памяти. Правило: контекст короткий.
> - **Модель строится один раз на процесс.** `OnModelCreating` вызывается однократно, результат кешируется по ключу (тип контекста + провайдер + `IModelCacheKeyFactory`). Не пиши туда логику, зависящую от запроса, — она «запечётся» в кеш. Исключение: параметры фильтров читаются из инстанса контекста и вычисляются при запросе.
> - **`SaveChanges` не откатывает изменения в объектах при ошибке.** Если транзакция упала, объекты в памяти остались изменёнными, состояния — `Modified`. Повторный `SaveChanges` попробует ещё раз. Не переиспользуй контекст после неизвестной ошибки — выбрось его.
> - **Async-методы обязательны.** Синхронный `SaveChanges()` в ASP.NET Core блокирует поток пула. При нагрузке это thread pool starvation.
> - **`EnableSensitiveDataLogging()` только в Dev.** Он пишет значения параметров в логи — в финтехе это утечка PII и данных карт.

> [!example] Как делают в бою
> ```csharp
> // Program.cs финтех-сервиса
> builder.Services.AddDbContext<LedgerContext>((sp, opt) =>
> {
>     var cs = sp.GetRequiredService<IConfiguration>().GetConnectionString("Ledger");
>     opt.UseNpgsql(cs, npg => npg
>            .EnableRetryOnFailure(3, TimeSpan.FromSeconds(2), null)
>            .CommandTimeout(30))
>        .UseSnakeCaseNamingConvention();
>
>     if (builder.Environment.IsDevelopment())
>     {
>         opt.EnableSensitiveDataLogging()
>            .EnableDetailedErrors()
>            .LogTo(Console.WriteLine, LogLevel.Information);
>     }
>
>     // Ловим случайный трекинг там, где его быть не должно
>     opt.ConfigureWarnings(w => w.Throw(CoreEventId.LazyLoadOnDisposedContextWarning));
> });
> ```
> Плюс health-check `AddNpgSql(cs)` ([[Health checks]]) и метрики EF через [[OpenTelemetry в .NET]].

## Вопросы с собеседований

> [!question]- Какие паттерны реализует `DbContext`?
> Unit of Work и Identity Map (а `DbSet<T>` — Repository). Unit of Work означает, что контекст накапливает изменения и применяет их одним `SaveChanges()` в одной транзакции. Identity Map — что строка с конкретным первичным ключом в рамках одного контекста материализуется в один и тот же объект; повторный запрос вернёт ту же ссылку и **не перезапишет** значения свойств данными из БД. Из-за этого писать собственные `IRepository`/`IUnitOfWork` поверх EF обычно избыточно — подробнее в [[Repository и Unit of Work: нужны ли поверх EF Core]].

> [!question]- Почему `DbContext` регистрируют как Scoped, а не Singleton?
> Три причины. Первая: контекст не потокобезопасен, параллельные операции на одном экземпляре бросают `InvalidOperationException`. Вторая: ChangeTracker в singleton растёт бесконечно — это и утечка памяти, и деградация `DetectChanges`, сложность которого линейна по числу отслеживаемых сущностей. Третья: Identity Map singleton'а отдавал бы всем запросам устаревшие данные, потому что повторный `SELECT` игнорируется в пользу уже загруженного объекта. Scoped даёт естественную границу «один HTTP-запрос — одна бизнес-транзакция».

> [!question]- Чем `AddDbContextPool` отличается от `AddDbContext`? Когда он вреден?
> `AddDbContextPool` переиспользует экземпляры `DbContext`: на `Dispose` контекст сбрасывается и возвращается в пул вместо сборки мусора. Это экономит аллокации на высоком RPS. Цена: конструктор контекста должен принимать только `DbContextOptions`, поэтому нельзя внедрять в контекст `ICurrentUser`, `ITenantProvider` и подобное. Любое собственное поле в контексте переживает возврат в пул и требует ручного сброса. Это не имеет отношения к пулу соединений — тот живёт в Npgsql и работает всегда.

> [!question]- Как использовать `DbContext` в `BackgroundService`?
> `BackgroundService` — синглтон, внедрять в него scoped-контекст нельзя (captive dependency). Либо создавать scope вручную через `IServiceScopeFactory.CreateScope()` на каждую итерацию, либо зарегистрировать `AddDbContextFactory<T>()` и получать контекст через `IDbContextFactory<T>.CreateDbContextAsync()`, диспозя его самому. Фабрика удобнее, когда нужно несколько независимых контекстов параллельно — например, для параллельных read-only запросов.

> [!question]- Что произойдёт, если дважды прочитать одну строку в одном контексте, а между чтениями её изменит другая транзакция?
> Ты получишь **старые** данные. Первое чтение положило объект в Identity Map. Второе чтение реально сходит в БД (SQL выполнится, это видно в логах), но при материализации EF обнаружит, что сущность с таким ключом уже отслеживается, и вернёт существующий объект, отбросив свежие значения. Чтобы получить актуальные данные: `AsNoTracking()` (новый объект каждый раз), `entry.ReloadAsync()` (перезаписать значения из БД) или `ChangeTracker.Clear()`.

> [!question]- Держит ли `DbContext` соединение с БД всё время своей жизни?
> Нет. По умолчанию EF открывает соединение непосредственно перед выполнением команды и закрывает сразу после — «connection resiliency»-модель. Соединение удерживается дольше только если: открыта явная транзакция (`BeginTransaction`), соединение открыли вручную через `db.Database.OpenConnection()`, или идёт стриминг результата (`AsAsyncEnumerable`, `await foreach` без буферизации). Поэтому «долгий контекст» опасен памятью, а не исчерпанием пула коннектов.

## Задачи

### Задача 1. Найти утечку

Сервис работает часами и постепенно съедает память. Найди причину и исправь.

```csharp
public sealed class RateCache
{
    private readonly LedgerContext _db;
    public RateCache(LedgerContext db) => _db = db;

    public async Task<decimal> GetRateAsync(string pair)
        => (await _db.Rates.FirstAsync(r => r.Pair == pair)).Value;
}

builder.Services.AddSingleton<RateCache>();
```

> [!success]- Решение
> `RateCache` — singleton, в него внедрён scoped `DbContext`. Это captive dependency: контекст живёт вечно, ChangeTracker накапливает все когда-либо прочитанные `Rate`, память растёт, а `GetRateAsync` со временем начинает возвращать устаревшие курсы из Identity Map. Плюс при параллельных запросах будет `InvalidOperationException`.
>
> ```csharp
> public sealed class RateCache(IDbContextFactory<LedgerContext> factory)
> {
>     public async Task<decimal> GetRateAsync(string pair, CancellationToken ct = default)
>     {
>         await using var db = await factory.CreateDbContextAsync(ct);
>         return await db.Rates
>             .AsNoTracking()
>             .Where(r => r.Pair == pair)
>             .Select(r => r.Value)
>             .FirstAsync(ct);
>     }
> }
>
> builder.Services.AddDbContextFactory<LedgerContext>(opt => opt.UseNpgsql(cs));
> builder.Services.AddSingleton<RateCache>();
> ```
> `AsNoTracking()` + проекция в `decimal` убирают трекинг совсем; контекст живёт микросекунды. Если нужен реальный кеш — см. [[Redis и стратегии кеширования]].

### Задача 2. Параллельные запросы

Код падает с `A second operation was started on this context instance`. Почини двумя способами.

```csharp
var accountIds = new[] { id1, id2, id3 };
var accounts = await Task.WhenAll(
    accountIds.Select(id => db.Accounts.FirstAsync(a => a.Id == id)));
```

> [!success]- Решение
> **Способ 1 (правильный в 95 % случаев) — один запрос вместо трёх:**
> ```csharp
> var accounts = await db.Accounts
>     .Where(a => accountIds.Contains(a.Id))
>     .ToListAsync();
> // SQL (EF 10): ... WHERE a.id IN (@accountIds1, @accountIds2, @accountIds3)
> ```
> Три round-trip'а превращаются в один. Это тот же паттерн, что лечит N+1.
>
> **Способ 2 — если запросы действительно разные и тяжёлые, контекст на задачу:**
> ```csharp
> var tasks = accountIds.Select(async id =>
> {
>     await using var scoped = await factory.CreateDbContextAsync();
>     return await scoped.Accounts.AsNoTracking().FirstAsync(a => a.Id == id);
> });
> var accounts = await Task.WhenAll(tasks);
> ```
> Помни: каждый контекст возьмёт своё соединение из пула. Параллелизм ограничен размером пула Npgsql (по умолчанию 100), не устраивай там штурм.

### Задача 3. Мультитенантность и пулинг

Нужен `tenantId` внутри контекста для глобального фильтра, но команда хочет `AddDbContextPool`. Как совместить?

> [!success]- Решение
> Прямо в конструктор контекста `tenantId` внедрить нельзя — пулинг запрещает любые зависимости кроме `DbContextOptions`. Решение: scoped-холдер, который контекст читает через `IServiceProvider`, доступный из опций.
>
> ```csharp
> public sealed class TenantContext { public Guid TenantId { get; set; } }
>
> builder.Services.AddScoped<TenantContext>();
> builder.Services.AddDbContextPool<LedgerContext>((sp, opt) =>
>     opt.UseNpgsql(cs)
>        .AddInterceptors(sp.GetRequiredService<TenantInterceptor>()));
>
> public sealed class LedgerContext(DbContextOptions<LedgerContext> options) : DbContext(options)
> {
>     // Заполняется интерцептором/фабрикой на каждый scope
>     public Guid TenantId { get; internal set; }
>
>     protected override void OnModelCreating(ModelBuilder b)
>         => b.Entity<Payment>().HasQueryFilter("TenantFilter", p => p.TenantId == TenantId);
> }
> ```
> Ключевой момент: выражение фильтра захватывает `this.TenantId`, а не константу. EF превращает такое обращение в параметр запроса и вычисляет его при каждом выполнении — значит, кеш модели остаётся корректным. Именованные фильтры — [[EF Core: глобальные и именованные фильтры]].
>
> Более простая альтернатива, если пулинг не критичен: обычный `AddDbContext` с фабрикой опций и `tenantId` в конструкторе.

## Итог

- EF Core — ORM: LINQ→SQL, материализация, отслеживание изменений. Цена — утечка абстракции и непрозрачный SQL; знать SQL всё равно обязательно.
- `DbContext` = Unit of Work (одна транзакция на `SaveChanges`) + Identity Map (одна строка = один объект, повторное чтение не обновляет значения).
- Регистрируй Scoped. Singleton — гарантированная утечка и гонки; Transient — разорванные Unit of Work.
- `AddDbContextPool` переиспользует экземпляры контекста ради аллокаций. Требует конструктор только с `DbContextOptions`. Не путать с пулом соединений Npgsql.
- Вне HTTP-запроса (фоновые сервисы, параллельные чтения) — `IServiceScopeFactory` или `IDbContextFactory<T>`.
- Контекст короткий, асинхронный, не разделяемый между потоками. Соединение он держит только на время команды.

## Связанное

- [[EF Core: конфигурация модели и Fluent API]]
- [[EF Core: change tracking и AsNoTracking]]
- [[EF Core: запросы и загрузка связанных данных]]
- [[EF Core: производительность и типичные грабли]]
- [[EF Core 10: что нового]]
- [[Жизненные циклы сервисов: Singleton, Scoped, Transient]]
- [[Captive dependency и типичные ошибки DI]]
- [[Repository и Unit of Work: нужны ли поверх EF Core]]
- [[ADO.NET: как всё устроено под капотом]]
- [[Пулинг соединений]]
