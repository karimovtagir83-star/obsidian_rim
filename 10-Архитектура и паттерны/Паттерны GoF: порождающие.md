---
tags: [раздел-10, паттерны, middle, собес, dotnet10]
aliases: [Creational patterns, Порождающие паттерны, GoF creational, Singleton, Factory Method, Abstract Factory, Builder, Prototype]
---

# Паттерны GoF: порождающие

> [!abstract] Коротко
> Порождающие паттерны отвечают на один вопрос: кто и как создаёт объект, если `new` в точке использования не подходит. Из пяти классических паттернов GoF в современном C# три (Singleton, Abstract Factory, Prototype) почти полностью растворились в языке и DI-контейнере, а два (Factory Method, Builder) живут и здравствуют — но в форме, которую Гамма с соавторами в 1994 году не узнали бы. Ниже — задача, минимальный код, готовая реализация в BCL и главное: когда паттерн применять НЕ надо и сколько это стоит.

## Зачем это нужно

Каталог GoF (Gang of Four, «Банда четырёх» — Гамма, Хелм, Джонсон, Влиссидес, 1994) писался под C++ и Smalltalk. В C++ 1994 года не было ни дженериков в нормальном виде, ни делегатов, ни лямбд, ни рефлексии, ни DI-контейнеров, ни сборщика мусора. Половина паттернов книги — это обходные пути вокруг отсутствующих языковых средств.

Отсюда два практических вывода, из которых и состоит эта заметка:

1. **Знать паттерн нужно, чтобы читать чужой код и проходить собеседование.** Legacy-код на C# 2005 года набит фабриками и синглтонами. Вы обязаны их узнавать.
2. **Писать паттерн руками нужно гораздо реже, чем кажется.** Признак мидла — не «знаю 23 паттерна», а «вижу, что здесь Strategy уже есть в виде DI-регистрации, и не надо строить второй уровень абстракции».

Худшее, что делает джун, начитавшийся GoF: заводит `ICustomerFactory` с одной реализацией, чтобы «создавать `Customer`». Это не паттерн, это лишний файл.

## Что произошло с каталогом за 30 лет

| Механизм языка/платформы | Какие паттерны съел |
|---|---|
| DI-контейнер (`IServiceProvider`) | Singleton, Abstract Factory, часть Factory Method, Service Locator |
| Делегаты и лямбды (`Func<T>`) | Factory Method в вырожденном виде, Strategy, Command |
| Дженерики | Abstract Factory (параметризация типом вместо иерархии фабрик) |
| `record` + `with` + `required` | Prototype, значительная часть Builder |
| Итераторы (`yield return`) | Iterator (полностью) |
| `IAsyncEnumerable`, `Channel<T>` | часть Observer |
| Pattern matching и switch-expression | часть State, часть Visitor |

Не съеденными остались те паттерны, которые описывают **не создание объекта, а управление его жизненным циклом и композицией** — Factory Method (когда решение о типе принимается в рантайме) и Builder (когда объект собирается пошагово с валидацией в конце).

---

## Singleton

**Задача:** гарантировать, что в процессе существует ровно один экземпляр типа, и дать к нему глобальную точку доступа.

Заметьте: в формулировке GoF **два** требования, склеенных в одно. Это и есть корень проблемы — «один экземпляр» и «глобальный доступ» это разные вещи, и современный подход их разделяет.

### Классическая реализация: статическое поле

```csharp
public sealed class ConfigCache
{
    // Инициализация статического поля — рантайм гарантирует потокобезопасность:
    // статические инициализаторы выполняются под блокировкой типа один раз.
    private static readonly ConfigCache _instance = new();

    // Приватный конструктор — снаружи new() невозможен.
    private ConfigCache() { }

    public static ConfigCache Instance => _instance;

    public string Get(string key) => /* ... */ key.ToUpperInvariant();
}
```

Здесь есть тонкость на уровне CLR, которую любят спрашивать. Если у типа есть **только** инициализаторы статических полей и нет явного статического конструктора, компилятор помечает тип в метаданных флагом `beforefieldinit`. Это разрешает JIT инициализировать поля в любой момент **до** первого обращения к полю — не обязательно точно в момент обращения. Если нужна строгая ленивость «инициализируй ровно при первом касании», нужен явный статический конструктор:

```csharp
public sealed class ConfigCache
{
    private static readonly ConfigCache _instance;

    // Явный статический конструктор убирает флаг beforefieldinit.
    // Теперь рантайм обязан отложить инициализацию до первого доступа к типу.
    static ConfigCache() => _instance = new ConfigCache();

    private ConfigCache() { }
    public static ConfigCache Instance => _instance;
}
```

Цена явного статического конструктора: JIT не может выкинуть проверку инициализации типа из горячего кода так же агрессивно, как при `beforefieldinit`. На практике это единицы наносекунд, но в микробенчмарках видно.

### Ленивая потокобезопасная версия: `Lazy<T>`

```csharp
public sealed class MetadataCache
{
    // Lazy<T> по умолчанию использует LazyThreadSafetyMode.ExecutionAndPublication:
    // фабрика выполнится ровно один раз, остальные потоки подождут на блокировке.
    private static readonly Lazy<MetadataCache> _lazy = new(() => new MetadataCache());

    public static MetadataCache Instance => _lazy.Value;

    private MetadataCache() { /* дорогая инициализация: скан сборок, рефлексия */ }
}
```

Режимы `LazyThreadSafetyMode`:

| Режим | Поведение | Когда |
|---|---|---|
| `ExecutionAndPublication` (по умолчанию для `new Lazy<T>(factory)`) | фабрика вызывается один раз, остальные ждут | почти всегда |
| `PublicationOnly` | фабрика может вызваться несколько раз параллельно, победит первый результат, лишние объекты выбросятся | если фабрика дешёвая и без побочных эффектов, а блокировка нежелательна |
| `None` | без синхронизации | только если доступ гарантированно однопоточный |

> [!danger] Исключение внутри фабрики `Lazy<T>` в режиме `ExecutionAndPublication` **кешируется**. Все последующие обращения к `.Value` будут кидать то же самое исключение, повторной попытки не будет. Если инициализация может упасть по временной причине (сеть, БД), `Lazy<T>` — неподходящий инструмент: вы навсегда отравите синглтон.

Двойная проверка блокировки (double-checked locking) руками — то, что писали до `Lazy<T>`:

```csharp
// #устарело — так писали до .NET 4.0. Встречается в легаси.
public sealed class LegacySingleton
{
    private static volatile LegacySingleton? _instance;   // volatile обязателен
    private static readonly object _gate = new();

    public static LegacySingleton Instance
    {
        get
        {
            if (_instance is null)               // быстрая проверка без блокировки
            {
                lock (_gate)
                {
                    _instance ??= new LegacySingleton();  // медленная под блокировкой
                }
            }
            return _instance;
        }
    }
}
```

Без `volatile` этот код формально сломан: модель памяти .NET не запрещает процессору увидеть ненулевую ссылку раньше, чем записанные поля объекта. На x86/x64 из-за сильной модели памяти оно почти всегда «работает», на ARM — нет. Подробности — в [[Interlocked, volatile и модель памяти]]. Сейчас так писать не надо, есть `Lazy<T>`.

### Почему в эпоху DI классический Singleton — антипаттерн

Разберём по пунктам, потому что «это антипаттерн» без объяснения — карго-культ.

**1. Зависимость становится невидимой.** Сигнатура ничего не говорит о том, от чего класс зависит:

```csharp
public sealed class OrderService
{
    public void Place(Order order)
    {
        // Ниоткуда из публичного API класса не видно, что OrderService
        // зависит от ConfigCache, Clock и AuditLog. Это скрытая связанность.
        var currency = ConfigCache.Instance.Get("currency");
        var now = SystemClock.Instance.UtcNow;
        AuditLog.Instance.Write($"order {order.Id} at {now}");
    }
}
```

Конструктор — это контракт. Через конструктор видно, что нужно классу, и компилятор заставит это предоставить. Статический доступ обходит контракт.

**2. Тест невозможно изолировать.** Чтобы подменить `SystemClock.Instance`, нужно либо сделать сеттер (и тогда тесты начинают влиять друг на друга через общее состояние, особенно при параллельном запуске xUnit), либо не тестировать. Оба варианта плохие.

**3. Жизненный цикл жёстко зашит в тип.** Синглтон живёт до конца процесса. Захотели «один на запрос» или «один на арендатора (tenant)» — придётся переписывать все точки использования, а не одну строку регистрации.

**4. Внутри синглтона нельзя держать scoped-зависимость.** Это самая частая продовая авария на этой теме. Синглтон живёт всё время процесса; `DbContext` по умолчанию scoped и живёт время HTTP-запроса. Если синглтон захватил `DbContext`, после завершения первого запроса контекст будет задиспоужен, а синглтон продолжит им пользоваться — получите `ObjectDisposedException` или, хуже, гонки в change tracker'е из разных потоков. Это называется captive dependency, см. [[Captive dependency и типичные ошибки DI]].

**Правильный ответ:**

```csharp
// Program.cs
builder.Services.AddSingleton<IConfigCache, ConfigCache>();

// Тип обычный: публичный конструктор, никаких статических полей.
public sealed class ConfigCache(IConfiguration configuration) : IConfigCache
{
    public string Get(string key) => configuration[key] ?? throw new KeyNotFoundException(key);
}

// Потребитель объявляет зависимость явно.
public sealed class OrderService(IConfigCache config, TimeProvider time)
{
    public void Place(Order order) { /* ... */ }
}
```

Вы получаете **тот же самый один экземпляр на процесс** — контейнер это гарантирует — но:
- зависимость видна в конструкторе;
- в тесте подменяется моком без глобального состояния;
- жизненный цикл меняется одной строкой регистрации;
- контейнер сам не даст (в Development, при `ValidateScopes = true`) заинжектить scoped в singleton и упадёт на старте вместо продакшена.

Подробности про жизненные циклы — [[Жизненные циклы сервисов: Singleton, Scoped, Transient]].

### Где статический синглтон всё же оправдан

Не всё, что статично, — зло. Критерий: **тип не имеет изменяемого состояния, зависящего от прикладной логики, и его подмена в тестах не нужна либо предусмотрена отдельно.**

| Пример из BCL | Почему статика уместна |
|---|---|
| `Encoding.UTF8`, `Encoding.ASCII` | иммутабельный, не имеет зависимостей, подменять нечего |
| `CultureInfo.InvariantCulture` | иммутабельная константа-описание |
| `TimeProvider.System` | это **дефолтная реализация абстракции**, а не глобальная точка доступа: в код инжектится `TimeProvider`, а `System` — просто значение, которое регистрируют в DI |
| `ArrayPool<T>.Shared` | общий пул по определению; подменять его в тестах бессмысленно |
| `Task.CompletedTask`, `Array.Empty<T>()` | кеш иммутабельных значений, чистая оптимизация аллокаций |
| Приватный `static readonly` кеш метаданных внутри типа | деталь реализации, не видна снаружи, не является API |
| `FrozenDictionary` с константной таблицей соответствий | иммутабельная таблица, вычисляемая один раз, см. [[Frozen и Immutable коллекции]] |

Обратите внимание на `TimeProvider`: команда .NET могла сделать `Clock.Now`. Вместо этого сделали абстрактный класс `TimeProvider` с готовым экземпляром `TimeProvider.System`. Регистрируете `builder.Services.AddSingleton(TimeProvider.System)`, инжектите `TimeProvider`, в тестах подставляете `FakeTimeProvider` из `Microsoft.Extensions.TimeProvider.Testing`. Это ровно тот приём, о котором речь: один экземпляр — да, глобальная точка доступа — нет. См. [[Дата и время: DateTime, DateTimeOffset, TimeProvider]].

---

## Factory Method

**Задача:** класс знает, что ему нужно создать объект, но не знает конкретный тип — тип определяется наследником или конфигурацией в рантайме.

Классическая форма GoF — виртуальный метод-фабрика в базовом классе:

```csharp
// Базовый класс задаёт алгоритм, но делегирует создание продукта наследнику.
public abstract class ReportJob
{
    // Это и есть фабричный метод.
    protected abstract IReportWriter CreateWriter();

    public async Task RunAsync(ReportData data, CancellationToken ct)
    {
        await using var writer = CreateWriter();   // не знаем конкретный тип
        await writer.WriteHeaderAsync(ct);
        foreach (var row in data.Rows)
            await writer.WriteRowAsync(row, ct);
    }
}

public sealed class CsvReportJob : ReportJob
{
    protected override IReportWriter CreateWriter() => new CsvReportWriter();
}

public sealed class ExcelReportJob : ReportJob
{
    protected override IReportWriter CreateWriter() => new ExcelReportWriter();
}
```

Более практичная форма — отдельная фабрика с выбором по рантайм-параметру:

```csharp
public interface IPaymentGatewayFactory
{
    IPaymentGateway Create(string providerCode);
}

public sealed class PaymentGatewayFactory(IServiceProvider services) : IPaymentGatewayFactory
{
    public IPaymentGateway Create(string providerCode) => providerCode switch
    {
        // Резолвим через контейнер, чтобы шлюзы получили свои зависимости
        // (HttpClient, логгер, опции), а не собирались руками через new.
        "stripe" => services.GetRequiredKeyedService<IPaymentGateway>("stripe"),
        "click"  => services.GetRequiredKeyedService<IPaymentGateway>("click"),
        "payme"  => services.GetRequiredKeyedService<IPaymentGateway>("payme"),
        _ => throw new NotSupportedException($"Неизвестный провайдер: {providerCode}")
    };
}
```

### Где Factory Method уже есть в .NET

**`ILoggerFactory`** — `CreateLogger(string categoryName)`. Фабрика нужна потому, что логгер параметризован категорией, которую невозможно знать на этапе регистрации. Обычно вы инжектите `ILogger<T>`, и за этим стоит вызов фабрики с категорией `typeof(T).FullName`.

**`IServiceProviderFactory<TContainerBuilder>`** — точка расширения хоста для замены встроенного контейнера на Autofac/Lamar. Хост не знает тип билдера контейнера, поэтому получает фабрику.

**`IHttpClientFactory`** — важный случай, разберём подробно, потому что это любимый вопрос собеса. Почему тут фабрика, а не просто `AddSingleton<HttpClient>()`?

```csharp
builder.Services.AddHttpClient("billing", c =>
{
    c.BaseAddress = new Uri("https://billing.internal/");
    c.Timeout = TimeSpan.FromSeconds(10);
});

public sealed class BillingClient(IHttpClientFactory factory)
{
    public async Task<Invoice?> GetAsync(Guid id, CancellationToken ct)
    {
        // Дешёвый короткоживущий HttpClient поверх переиспользуемого handler'а.
        using var http = factory.CreateClient("billing");
        return await http.GetFromJsonAsync<Invoice>($"invoices/{id}", ct);
    }
}
```

Суть в том, что дорогой ресурс — не `HttpClient`, а `HttpMessageHandler` под ним: именно он держит пул TCP-соединений. Крайности плохи обе:

- `new HttpClient()` на каждый запрос → каждый раз новый handler → новый TCP-коннект → на нагрузке исчерпание портов (socket exhaustion), потому что закрытые сокеты висят в состоянии `TIME_WAIT`;
- один `HttpClient` на всё время процесса → handler живёт вечно → DNS-запись закеширована навсегда, при смене IP за DNS-именем (типично для облачных балансировщиков и Kubernetes-сервисов) клиент стучится на мёртвый адрес.

Фабрика решает обе проблемы: она держит **пул handler'ов** и переиспользует их между вызовами `CreateClient`, но принудительно ротирует каждые 2 минуты (`SetHandlerLifetime` меняет интервал). `HttpClient`, который вы получаете, — тонкая дешёвая обёртка, её можно и нужно выбрасывать. Это Factory Method, чья настоящая работа — управление жизненным циклом невидимого потребителю ресурса. Подробнее — [[HttpClient и IHttpClientFactory]].

### Когда фабрика не нужна

- **Реализация одна и типов на выбор нет.** `ICustomerFactory.Create(...)` вокруг `new Customer(...)` — просто лишний слой. Пишите `new`.
- **Нужен просто отложенный/повторный экземпляр.** Хватит делегата: контейнер умеет резолвить `Func<T>` при явной регистрации, либо вы регистрируете фабричную лямбду сами.

```csharp
// Вместо интерфейса-фабрики с одним методом — делегат.
builder.Services.AddSingleton<Func<ReportKind, IReportWriter>>(sp => kind => kind switch
{
    ReportKind.Csv   => sp.GetRequiredService<CsvReportWriter>(),
    ReportKind.Excel => sp.GetRequiredService<ExcelReportWriter>(),
    _ => throw new ArgumentOutOfRangeException(nameof(kind))
});

public sealed class ExportService(Func<ReportKind, IReportWriter> writerFor)
{
    public IReportWriter Resolve(ReportKind kind) => writerFor(kind);
}
```

Цена делегата: имя типа-фабрики нигде не фигурирует, поиск по коду сложнее, `Func<ReportKind, IReportWriter>` как тип регистрации выглядит неочевидно в трассировке ошибок DI. Если фабрика с логикой сложнее одного `switch` — делайте нормальный именованный интерфейс.

- **Все варианты нужны сразу, а не по одному.** Тогда не фабрика, а `IEnumerable<IHandler>` — контейнер отдаст все зарегистрированные реализации.

---

## Abstract Factory

**Задача:** создавать **семейства** согласованных объектов, не завязываясь на конкретные классы. Ключевое слово — семейство: важно, чтобы `Connection`, `Command` и `Parameter` были от одного провайдера, иначе всё сломается.

```csharp
// Семейство объектов для работы с конкретной СУБД.
public interface IStorageFactory
{
    IDocumentReader CreateReader();
    IDocumentWriter CreateWriter();
    IDocumentIndex  CreateIndex();
}

public sealed class S3StorageFactory(IAmazonS3 s3) : IStorageFactory
{
    public IDocumentReader CreateReader() => new S3DocumentReader(s3);
    public IDocumentWriter CreateWriter() => new S3DocumentWriter(s3);
    public IDocumentIndex  CreateIndex()  => new S3DocumentIndex(s3);
}

public sealed class LocalStorageFactory(string root) : IStorageFactory
{
    public IDocumentReader CreateReader() => new FileDocumentReader(root);
    public IDocumentWriter CreateWriter() => new FileDocumentWriter(root);
    public IDocumentIndex  CreateIndex()  => new FileDocumentIndex(root);
}
```

### Где это есть в .NET

**`DbProviderFactory`** (`System.Data.Common`) — эталонный пример. Абстрактный класс с методами `CreateConnection()`, `CreateCommand()`, `CreateParameter()`, `CreateCommandBuilder()`. Каждый ADO.NET-провайдер поставляет наследника: `NpgsqlFactory.Instance`, `SqlClientFactory.Instance`. Смысл — написать код, работающий с любой СУБД:

```csharp
// Провайдер выбирается в рантайме, код ниже про конкретную СУБД не знает.
DbProviderFactory factory = NpgsqlFactory.Instance;

await using var connection = factory.CreateConnection()!;
connection.ConnectionString = connectionString;
await connection.OpenAsync(ct);

await using var command = factory.CreateCommand()!;
command.Connection = connection;
command.CommandText = "SELECT id, total FROM orders WHERE created_at > @from";

var p = factory.CreateParameter()!;
p.ParameterName = "@from";
p.Value = DateTime.UtcNow.AddDays(-7);
command.Parameters.Add(p);
```

Обратите внимание, насколько это неудобно: `CreateParameter()` возвращает `DbParameter`, который надо настраивать в три строки вместо `new NpgsqlParameter("@from", value)`. Это цена абстрактной фабрики — вы теряете удобные конструкторы и специфичные для провайдера возможности. На практике 99 % проектов не меняют СУБД и пишут прямо против `NpgsqlConnection` или используют EF Core, а `DbProviderFactory` нужна инструментам (профайлерам, миграторам, дизайнерам), которые обязаны работать с любым провайдером. См. [[ADO.NET: как всё устроено под капотом]].

**`ILoggerProvider`** — `CreateLogger(string categoryName)`. Формально это одноместная фабрика, но роль та же: `ILoggerFactory` держит набор провайдеров (консоль, файл, OTLP) и на каждую категорию собирает композит из их логгеров.

### Чем заменяется в C#

**Дженериком**, если семейство параметризуется одним типом:

```csharp
// Вместо иерархии фабрик — один открытый дженерик, зарегистрированный в DI.
public interface IRepository<TEntity> where TEntity : class
{
    ValueTask<TEntity?> FindAsync(object id, CancellationToken ct);
}

builder.Services.AddScoped(typeof(IRepository<>), typeof(EfRepository<>));
// Резолв IRepository<Order> сам подставит EfRepository<Order>.
```

**Keyed-сервисами**, если вариант выбирается по строковому/enum-ключу:

```csharp
builder.Services.AddKeyedSingleton<IDocumentReader, S3DocumentReader>("s3");
builder.Services.AddKeyedSingleton<IDocumentWriter, S3DocumentWriter>("s3");
builder.Services.AddKeyedSingleton<IDocumentReader, FileDocumentReader>("local");
builder.Services.AddKeyedSingleton<IDocumentWriter, FileDocumentWriter>("local");

// Потребитель фиксирует «семейство» через атрибут — согласованность обеспечивается ключом.
public sealed class ArchiveService(
    [FromKeyedServices("s3")] IDocumentReader reader,
    [FromKeyedServices("s3")] IDocumentWriter writer);
```

Цена: ключ — это строка, компилятор не проверит согласованность. Если в одном конструкторе один параметр помечен `"s3"`, а другой `"local"`, всё скомпилируется и упадёт в рантайме на несовместимости. Абстрактная фабрика такую ошибку делает невозможной по конструкции — это её единственное реальное преимущество. Подробнее про keyed-регистрацию — [[Keyed services и продвинутая регистрация]].

---

## Builder

**Задача:** собрать сложный объект пошагово, когда конструктор с 12 параметрами нечитаем, часть параметров опциональна, а валидация возможна только когда собрано всё.

```csharp
public sealed class HttpRequestSpec
{
    public required Uri Url { get; init; }
    public HttpMethod Method { get; init; } = HttpMethod.Get;
    public IReadOnlyDictionary<string, string> Headers { get; init; } =
        new Dictionary<string, string>();
    public TimeSpan Timeout { get; init; } = TimeSpan.FromSeconds(30);
    public int RetryCount { get; init; }
}

public sealed class HttpRequestSpecBuilder
{
    private Uri? _url;
    private HttpMethod _method = HttpMethod.Get;
    private readonly Dictionary<string, string> _headers = [];
    private TimeSpan _timeout = TimeSpan.FromSeconds(30);
    private int _retryCount;

    // Каждый шаг возвращает this — это и даёт fluent-цепочку.
    public HttpRequestSpecBuilder To(string url) { _url = new Uri(url); return this; }
    public HttpRequestSpecBuilder Post() { _method = HttpMethod.Post; return this; }
    public HttpRequestSpecBuilder WithHeader(string name, string value)
    {
        _headers[name] = value;
        return this;
    }
    public HttpRequestSpecBuilder WithTimeout(TimeSpan t) { _timeout = t; return this; }
    public HttpRequestSpecBuilder WithRetries(int count)
    {
        // Валидация шага — сразу, чтобы стек указывал на место ошибки.
        ArgumentOutOfRangeException.ThrowIfNegative(count);
        _retryCount = count;
        return this;
    }

    // Валидация целого — только в Build.
    public HttpRequestSpec Build()
    {
        if (_url is null)
            throw new InvalidOperationException("Не задан URL: вызовите To().");
        if (_retryCount > 0 && _method == HttpMethod.Post && !_headers.ContainsKey("Idempotency-Key"))
            throw new InvalidOperationException(
                "Ретраи для POST без Idempotency-Key приведут к дублям.");

        return new HttpRequestSpec
        {
            Url = _url,
            Method = _method,
            Headers = _headers.ToFrozenDictionary(),
            Timeout = _timeout,
            RetryCount = _retryCount
        };
    }
}

// Использование
var spec = new HttpRequestSpecBuilder()
    .To("https://billing.internal/invoices")
    .Post()
    .WithHeader("Idempotency-Key", Guid.CreateVersion7().ToString())
    .WithRetries(3)
    .Build();
```

Ключевая мысль, которую часто упускают: **ценность Builder — не в fluent-синтаксисе, а в том, что валидация инварианта происходит один раз в `Build()`**, когда виден весь набор значений. Проверка «ретраи для POST требуют ключ идемпотентности» невозможна ни в одном отдельном сеттере.

### Где это есть в .NET

| Тип | Что собирает | Особенность |
|---|---|---|
| `StringBuilder` | строку | не fluent-паттерн GoF, а мутабельный аккумулятор ради производительности |
| `WebApplicationBuilder` | `WebApplication` | `builder.Build()` фиксирует контейнер и конфигурацию: после `Build()` регистрировать сервисы уже нельзя |
| `HostApplicationBuilder` | `IHost` | облегчённый вариант для консольных сервисов и воркеров |
| `DbContextOptionsBuilder` | `DbContextOptions` | накапливает расширения провайдера, `UseNpgsql`, `UseLazyLoadingProxies` и прочее |
| `ModelBuilder` / `EntityTypeBuilder<T>` | модель EF Core | вложенные билдеры, см. [[EF Core: конфигурация модели и Fluent API]] |
| `ConfigurationBuilder` | `IConfigurationRoot` | порядок вызовов задаёт приоритет источников |
| `AuthorizationPolicyBuilder` | политику авторизации | `RequireRole`, `RequireClaim`, потом `Build()` |

Обратите внимание на `WebApplicationBuilder`: он показывает вторую важную функцию Builder — **фазу «конфигурация закрыта»**. До `Build()` контейнер мутабельный, после — заморожен. Без такой фазы вы бы не знали, безопасно ли уже резолвить сервисы.

### Современная альтернатива: `record` + `required` + `with`

Если валидации «целого» нет, а нужна только читаемость — Builder не нужен:

```csharp
public sealed record RetryPolicySpec
{
    // required — компилятор не даст создать объект без этих значений.
    public required int MaxAttempts { get; init; }
    public required TimeSpan BaseDelay { get; init; }
    public bool UseJitter { get; init; } = true;
    public double Multiplier { get; init; } = 2.0;
}

var basePolicy = new RetryPolicySpec { MaxAttempts = 3, BaseDelay = TimeSpan.FromMilliseconds(200) };

// with даёт «пошаговость» без билдера: каждый with — новый иммутабельный объект.
var aggressive = basePolicy with { MaxAttempts = 6, Multiplier = 1.5 };
```

Это покрывает большинство случаев: объектные инициализаторы читаются как fluent-цепочка, `required` заменяет проверку `if (_url is null)`, `with` заменяет «взять шаблон и подправить». Валидацию инварианта можно повесить на первичный конструктор record'а или в отдельный статический `Create`, возвращающий `Result<T>` (см. [[Result pattern вместо исключений]]). Про `record` и `init` — [[Записи (record) и структуры]], [[Иммутабельность как приём проектирования]].

Цена `with` по сравнению с билдером: каждый `with` создаёт новый объект. Для цепочки из 8 шагов в горячем цикле это 8 аллокаций против одного мутабельного билдера. В настройке приложения на старте — не важно; в обработке миллиона сообщений — измеряйте.

### Тест-билдеры: случай, где Builder всегда оправдан

```csharp
// Билдер для тестов: разумные значения по умолчанию + точечное переопределение.
internal sealed class OrderBuilder
{
    private Guid _id = Guid.CreateVersion7();
    private string _customerEmail = "test@example.com";
    private OrderStatus _status = OrderStatus.New;
    private readonly List<OrderLine> _lines = [];

    public OrderBuilder WithStatus(OrderStatus status) { _status = status; return this; }
    public OrderBuilder WithLine(string sku, int qty, decimal price)
    {
        _lines.Add(new OrderLine(sku, qty, price));
        return this;
    }

    public Order Build() => new(_id, _customerEmail, _status, _lines);

    // Неявное преобразование, чтобы в тесте писать без .Build()
    public static implicit operator Order(OrderBuilder b) => b.Build();
}

// В тесте видно ровно то, что важно для этого теста, остальное — дефолты.
Order order = new OrderBuilder()
    .WithStatus(OrderStatus.Paid)
    .WithLine("SKU-1", qty: 2, price: 19.99m);
```

Это лучшее вложение в читаемость тестов: когда тест падает, вы видите только значимые для него поля. Подробнее — [[Тестовые данные: Bogus и билдеры]].

---

## Prototype

**Задача:** создать копию существующего объекта, не зная его конкретного типа и не вызывая конструктор.

### Почему `ICloneable` сломан

```csharp
namespace System;

public interface ICloneable
{
    object Clone();   // и всё
}
```

Проблемы, из-за которых Microsoft сама рекомендует не реализовывать этот интерфейс:

1. **Контракт не определяет глубину копирования.** Получив `ICloneable`, вы не знаете, будет копия глубокой или поверхностной. Значит, не можете безопасно ею пользоваться: если внутри список, разделяемый с оригиналом, изменение копии сломает оригинал.
2. **Возвращает `object`.** Каждый вызов требует приведения типа. До дженериков это было терпимо, сейчас — нет.
3. **Не работает с иммутабельностью и `required`.** Для иммутабельного типа клонирование бессмысленно, а для типа с `required`-свойствами клон приходится собирать вручную.

```csharp
// Так делать не надо — просто чтобы вы узнали это в легаси. #устарело
public sealed class Basket : ICloneable
{
    public List<string> Items { get; } = [];

    // Поверхностная копия: MemberwiseClone скопирует ССЫЛКУ на список.
    // Изменение clone.Items изменит и original.Items. Классическая ловушка.
    public object Clone() => MemberwiseClone();
}
```

### Современно: `record` и `with`

```csharp
public sealed record Address(string City, string Street, string Zip);

public sealed record CustomerProfile(string Name, Address Address, ImmutableArray<string> Tags);

var original = new CustomerProfile("Иван", new Address("Ташкент", "Амира Темура 1", "100000"), ["vip"]);

// Компилятор генерирует копирующий конструктор: поверхностное копирование всех полей
// + переопределение указанных. Для иммутабельных членов этого достаточно.
var moved = original with { Address = original.Address with { City = "Самарканд" } };
```

Важно понимать механику: `with` — это **поверхностное** копирование. Компилятор генерирует защищённый копирующий конструктор `protected CustomerProfile(CustomerProfile other)`, который присваивает поля один к одному, а затем применяет указанные в `with` изменения. Если внутри record'а лежит мутабельный `List<T>`, копия и оригинал будут делить один список. Поэтому иммутабельные record'ы должны содержать иммутабельные члены — `ImmutableArray<T>`, `FrozenSet<T>`, другие record'ы.

### Ручной `Clone` с явной семантикой

```csharp
public sealed class MutableOrderDraft
{
    public required string CustomerEmail { get; set; }
    public List<OrderLine> Lines { get; set; } = [];

    // Имя метода говорит о глубине копирования — то, чего не хватает ICloneable.
    public MutableOrderDraft DeepCopy() => new()
    {
        CustomerEmail = CustomerEmail,
        // Новый список + копии элементов, если элементы мутабельны.
        Lines = [.. Lines.Select(l => l with { })]
    };
}
```

### Копирование через сериализацию — «клонирование бедняка»

```csharp
// Работает для чего угодно сериализуемого, но цена высокая.
public static T DeepCopyViaJson<T>(T source) =>
    JsonSerializer.Deserialize<T>(JsonSerializer.SerializeToUtf8Bytes(source))!;
```

Цена, которую надо назвать, если предлагаете это на собеседовании:

- **производительность** — сериализация в UTF-8 и обратно на порядки дороже прямого копирования полей: аллокации буфера, парсинг, рефлексия или сгенерированный код;
- **потеря информации** — приватные поля, ссылочная идентичность (два поля, ссылавшиеся на один объект, станут двумя разными объектами), циклические ссылки требуют `ReferenceHandler.Preserve`;
- **изменение типов** — интерфейсные и полиморфные поля десериализуются в объявленный тип, наследник теряется, если не настроен полиморфный маппинг;
- **тихая порча данных** — свойство без сеттера просто не восстановится, и вы узнаете об этом в проде.

Допустимо в тестах и одноразовых утилитах. В горячем пути — нет. См. [[Сериализация: System.Text.Json]].

---

## Сводная таблица

| Паттерн | Задача | Чем заменён в современном C# | Когда всё ещё нужен явно |
|---|---|---|---|
| **Singleton** | один экземпляр + глобальный доступ | `AddSingleton<IFoo, Foo>()` — один экземпляр без глобального доступа | иммутабельные значения BCL-уровня (`Encoding.UTF8`), приватный статический кеш метаданных внутри типа, `ArrayPool<T>.Shared` |
| **Factory Method** | тип продукта неизвестен на этапе компиляции | `Func<T>`/`Func<TKey, T>` в DI, keyed services, `IEnumerable<IHandler>` | управление жизненным циклом ресурса (`IHttpClientFactory`), параметризация именем (`ILoggerFactory`), выбор по данным запроса |
| **Abstract Factory** | согласованное семейство продуктов | открытый дженерик `AddScoped(typeof(IRepo<>), typeof(EfRepo<>))`, keyed-регистрация | когда рассогласование семейства должно быть невозможно по конструкции; плагинные архитектуры и провайдеры (`DbProviderFactory`) |
| **Builder** | пошаговая сборка + валидация целого | `record` + `required` + `with`, объектные инициализаторы | валидация комбинации значений в `Build()`, фаза «конфигурация закрыта» (`WebApplicationBuilder`), тест-билдеры, DSL |
| **Prototype** | копия без знания типа | `record` + `with` (поверхностная копия) | иерархии мутабельных объектов, где нужен настраиваемый deep copy; снапшоты состояния (см. Memento в [[Паттерны GoF: поведенческие]]) |

> [!warning] Подводные камни
> - **`Lazy<T>` кеширует исключение.** В режиме по умолчанию `ExecutionAndPublication` упавшая фабрика отравляет объект навсегда: каждое обращение к `.Value` перебрасывает то же исключение. Для инициализации, которая может упасть по временной причине (недоступна БД), `Lazy<T>` не подходит — нужен явный retry-механизм.
> - **Синглтон, захвативший scoped-зависимость, — бомба с таймером.** `AddSingleton<Cache>()`, где `Cache` принимает `DbContext`, скомпилируется и запустится. Упадёт `ObjectDisposedException` или тихо повредит данные под нагрузкой. Держите `ValidateScopes` включённым (по умолчанию так в Development) или инжектите `IServiceScopeFactory` и создавайте скоуп на операцию.
> - **`with` копирует поверхностно.** `var copy = order with { };` даст новый объект, который делит с оригиналом тот же `List<OrderLine>`. Иммутабельные record'ы обязаны содержать иммутабельные коллекции, иначе иммутабельность мнимая.
> - **Фабрика, резолвящая из `IServiceProvider`, — это Service Locator внутри.** Приемлемо для инфраструктурной фабрики на границе, но если такой `IServiceProvider` протекает в доменные классы, вы потеряли compile-time-проверку зависимостей: ошибка регистрации выстрелит в рантайме и только на том пути кода, который вызвали.
> - **`MemberwiseClone` не вызывает конструктор.** Инварианты, которые обеспечивает конструктор, при клонировании не проверяются. Клон может оказаться в невозможном состоянии — например, с полем, вычисленным до изменения другого поля.
> - **Абстрактная фабрика ради «а вдруг сменим БД» почти никогда не окупается.** Смена СУБД — это переписывание запросов, миграций, типов и индексов. Абстракция над созданием соединения покрывает 2 % работы, а платите вы за неё каждый день неудобным API.

> [!example] Как делают в бою
> **Платёжные провайдеры в fintech-сервисе.** Три-пять шлюзов (Stripe, локальные Click/Payme, банковский эквайринг). Каждый со своим HTTP-клиентом, ретраями, схемой подписи. Рабочая схема: `AddKeyedScoped<IPaymentGateway, StripeGateway>("stripe")` для каждого + тонкая фабрика `IPaymentGatewayFactory`, единственная задача которой — превратить код провайдера из БД в `GetRequiredKeyedService`. Фабрика нужна, потому что провайдер выбирается по данным платежа в рантайме, а `[FromKeyedServices]` работает только со статически известным ключом.
>
> **Никаких `Singleton.Instance` в новом коде.** Ревью отклоняет статические точки доступа с состоянием. Исключения перечислены в правилах проекта явным списком: `TimeProvider.System` (только в `Program.cs` при регистрации), `Encoding`, `JsonSerializerOptions` как `static readonly` кеш (важно: `JsonSerializerOptions` дорог в создании и потокобезопасен после первого использования, поэтому его переиспользование — обязательная оптимизация, а не роскошь).
>
> **Билдеры живут в двух местах**: в composition root (там их дал фреймворк) и в тестах. В доменном коде — `record` + `required`. Если в домене захотелось билдер, обычно это сигнал, что у сущности слишком много опциональных полей и её пора разделить.
>
> **Deep copy через JSON** используется ровно в одном месте: снапшот конфигурации в аудит-лог, где нужна человекочитаемая форма и производительность не важна. Всё остальное копируется явными методами.

## Вопросы с собеседований

> [!question]- Почему Singleton считают антипаттерном, если `AddSingleton` в DI делают все?
> Потому что это разные вещи. Паттерн GoF склеивает два требования: «один экземпляр» и «глобальная статическая точка доступа». Вредно именно второе: зависимость становится невидимой в сигнатуре, класс нельзя изолировать в тесте, жизненный цикл зашит в тип и не меняется без правки всех потребителей. `AddSingleton<IFoo, Foo>()` даёт первое требование без второго — экземпляр один, но получают его через конструктор, значит зависимость явная, подменяемая и с управляемым временем жизни. Плюс контейнер проверяет корректность графа: заинжектить scoped в singleton он не даст при включённой валидации скоупов.

> [!question]- Что не так с `Lazy<T>` для ленивой инициализации соединения с внешним сервисом?
> `Lazy<T>` в режиме по умолчанию (`ExecutionAndPublication`) кеширует не только результат, но и исключение. Если фабрика упала из-за недоступности сервиса, каждое последующее обращение к `.Value` будет перебрасывать то же исключение до конца жизни объекта — восстановления не будет. Для ресурсов, чья инициализация может упасть временно, нужен либо `LazyThreadSafetyMode.PublicationOnly` (там исключение не кешируется, но фабрика может выполниться параллельно несколько раз), либо собственный механизм с `SemaphoreSlim` и явным сбросом состояния при ошибке, либо `AsyncLazy`-подобная конструкция с retry.

> [!question]- Зачем `IHttpClientFactory`, если `HttpClient` реализует `IDisposable` и его положено оборачивать в `using`?
> Дорогой ресурс — не `HttpClient`, а `HttpMessageHandler` под ним: он держит пул TCP-соединений. `new HttpClient()` в `using` на каждый запрос создаёт новый handler, соединение закрывается и висит в `TIME_WAIT`, под нагрузкой это исчерпание портов. Обратная крайность — один статический `HttpClient` навсегда: handler живёт бесконечно и держит устаревшую DNS-запись, что ломается при смене IP за DNS-именем. Фабрика держит пул handler'ов, переиспользует их между вызовами `CreateClient` и ротирует по таймауту (по умолчанию 2 минуты). Возвращаемый `HttpClient` — дешёвая обёртка, её выбрасывать нормально.

> [!question]- Чем Factory Method отличается от Abstract Factory?
> Factory Method создаёт один продукт, и вариативность обычно достигается переопределением метода в наследнике или ветвлением по ключу. Abstract Factory создаёт **семейство** связанных продуктов, и её главная задача — гарантировать их согласованность: нельзя получить `NpgsqlConnection` вместе с `SqlCommand`. Если у вас в интерфейсе фабрики один метод `Create` — это Factory Method, и почти наверняка его можно заменить на `Func<T>` или keyed-сервис. Если методов несколько и они обязаны быть от одной «семьи» — есть смысл в Abstract Factory, потому что она делает рассогласование невозможным по конструкции.

> [!question]- Когда Builder оправдан, а когда достаточно `record` с `required` и `with`?
> Builder оправдан, если валидацию можно провести только когда собрано всё: «ретраи для POST требуют заголовок идемпотентности», «либо `ConnectionString`, либо пара `Host`+`Port`, но не оба». Такую проверку нельзя разместить ни в одном отдельном сеттере — нужен момент `Build()`. Второй случай — фаза «конфигурация закрыта» (как `WebApplicationBuilder.Build()`), после которой объект замораживается. Третий — тест-билдеры с разумными дефолтами. Если ничего из этого нет, `record` с `required`-свойствами и объектным инициализатором читается так же хорошо, а `required` даёт compile-time-проверку обязательных полей, чего билдер дать не может — он проверяет в рантайме.

> [!question]- Что вернёт `MemberwiseClone` для класса с полем `List<int>`?
> Новый объект, у которого поле `List<int>` содержит **ту же самую ссылку**, что у оригинала. Это поверхностное копирование: копируются значения полей, а для ссылочного типа значение поля — это ссылка. Добавление элемента в список копии будет видно в оригинале. Кроме того, `MemberwiseClone` не вызывает конструктор, поэтому инварианты, обеспечиваемые конструктором, не проверяются, и клон может оказаться в состоянии, которое обычным путём получить нельзя. Именно из-за этой неопределённости `ICloneable` считают ошибкой дизайна: интерфейс не говорит, какая копия получится.

> [!question]- Где в современном ASP.NET Core-приложении статический синглтон всё ещё нормален?
> Там, где нет прикладного изменяемого состояния и нет смысла подменять: `Encoding.UTF8`, `CultureInfo.InvariantCulture`, `ArrayPool<T>.Shared`, `Array.Empty<T>()`, `Task.CompletedTask`. Отдельный важный случай — `static readonly JsonSerializerOptions`: создание опций дорого, а после первого использования они становятся иммутабельными и потокобезопасными, поэтому переиспользование обязательно, иначе каждая сериализация заново строит метаданные типов. Также нормально держать приватный `static readonly` кеш метаданных внутри типа — это деталь реализации, а не публичная точка доступа. `TimeProvider.System` — пограничный случай: это не синглтон-паттерн, а дефолтное значение абстракции, которое регистрируют в DI.

> [!question]- Как заменить `if/switch` по типу провайдера, не заводя фабрику?
> Если реализаций несколько и все нужны сразу — зарегистрировать их все под одним интерфейсом и инжектить `IEnumerable<IProvider>`, выбирая нужную предикатом `CanHandle`. Если выбор идёт по ключу — keyed services: `AddKeyedSingleton<IProvider, StripeProvider>("stripe")` плюс `GetRequiredKeyedService<IProvider>(code)`. Фабрика нужна только как тонкая прослойка, если ключ приходит из данных рантайма и вы не хотите тащить `IServiceProvider` в прикладные классы. Заводить интерфейс-фабрику с единственным методом и `switch` внутри, когда keyed-сервисы решают то же самое, — лишний слой.

## Задачи

### Задача 1. Починить синглтон с scoped-зависимостью

Дан код, который падает в проде с `ObjectDisposedException` после первого запроса. Найдите причину и исправьте, сохранив «один экземпляр кеша на процесс».

```csharp
public sealed class ProductCache
{
    private readonly AppDbContext _db;
    private readonly Dictionary<int, string> _names = [];

    public ProductCache(AppDbContext db) => _db = db;

    public async Task<string> GetNameAsync(int id, CancellationToken ct)
    {
        if (_names.TryGetValue(id, out var name)) return name;
        name = await _db.Products.Where(p => p.Id == id).Select(p => p.Name).SingleAsync(ct);
        _names[id] = name;
        return name;
    }
}

// Program.cs
builder.Services.AddSingleton<ProductCache>();
builder.Services.AddDbContext<AppDbContext>(o => o.UseNpgsql(cs));
```

> [!success]- Решение
> Две ошибки. Первая — captive dependency: singleton захватил scoped `AppDbContext`, который диспоузится в конце первого HTTP-запроса. Вторая — обычный `Dictionary` мутируется из разных потоков одновременно, это гонка, которая может привести к бесконечному циклу внутри словаря.
>
> ```csharp
> public sealed class ProductCache(IServiceScopeFactory scopeFactory)
> {
>     // ConcurrentDictionary вместо Dictionary: singleton по определению
>     // используется несколькими запросами параллельно.
>     private readonly ConcurrentDictionary<int, string> _names = new();
>
>     public async Task<string> GetNameAsync(int id, CancellationToken ct)
>     {
>         if (_names.TryGetValue(id, out var cached)) return cached;
>
>         // Скоуп создаётся на одну операцию — контекст живёт ровно столько, сколько нужно.
>         await using var scope = scopeFactory.CreateAsyncScope();
>         var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
>
>         var name = await db.Products
>             .Where(p => p.Id == id)
>             .Select(p => p.Name)
>             .SingleAsync(ct);
>
>         // GetOrAdd, а не индексатор: два потока могли прочитать одновременно.
>         return _names.GetOrAdd(id, name);
>     }
> }
> ```
>
> Почему `IServiceScopeFactory`, а не `IServiceProvider`: явно видно, что класс создаёт скоупы, и нельзя случайно резолвить scoped-сервис из корневого провайдера. Обратите внимание, что `GetOrAdd` здесь не защищает от двух параллельных запросов к БД — если это важно, нужен `Lazy<Task<string>>` в словаре или готовый `HybridCache`, см. [[Кеширование: Output, Response, HybridCache]].

### Задача 2. Заменить абстрактную фабрику на keyed-сервисы

Есть иерархия из трёх фабрик нотификаций (`EmailNotifierFactory`, `SmsNotifierFactory`, `PushNotifierFactory`), каждая создаёт `INotifier` и `INotificationTemplateResolver`. Перепишите на keyed services так, чтобы не потерять согласованность пары.

> [!success]- Решение
> ```csharp
> // Регистрация: ключ один и тот же для всей «семьи».
> const string Email = "email";
> const string Sms   = "sms";
>
> builder.Services.AddKeyedScoped<INotifier, EmailNotifier>(Email);
> builder.Services.AddKeyedScoped<INotificationTemplateResolver, EmailTemplateResolver>(Email);
> builder.Services.AddKeyedScoped<INotifier, SmsNotifier>(Sms);
> builder.Services.AddKeyedScoped<INotificationTemplateResolver, SmsTemplateResolver>(Sms);
>
> // Согласованность гарантируется тем, что ключ — один параметр, а не два.
> public sealed class NotificationSender(IServiceProvider services)
> {
>     public async Task SendAsync(string channel, NotificationRequest request, CancellationToken ct)
>     {
>         var notifier = services.GetRequiredKeyedService<INotifier>(channel);
>         var resolver = services.GetRequiredKeyedService<INotificationTemplateResolver>(channel);
>
>         var body = await resolver.ResolveAsync(request.TemplateId, request.Data, ct);
>         await notifier.SendAsync(request.Recipient, body, ct);
>     }
> }
> ```
>
> Честная оценка: мы убрали три класса-фабрики и их интерфейс, но потеряли compile-time-гарантию согласованности — ключ стал строкой. Компенсация: константы вместо литералов и один тест, который для каждого канала резолвит всю пару и проверяет, что резолв не падает. Если каналов много и они приходят из плагинов, абстрактная фабрика была бы обоснованнее — регистрацию семьи тогда делает сам плагин одним методом.

### Задача 3. Тест-билдер с обязательным полем

Напишите билдер для агрегата `Payment`, у которого есть обязательные `Amount` и `Currency` и опциональные `Status`, `ProviderRef`, `CreatedAt`. Требование: если в тесте забыли задать сумму, тест должен упасть с понятным сообщением, а не с `Amount = 0`.

> [!success]- Решение
> ```csharp
> internal sealed class PaymentBuilder
> {
>     private decimal? _amount;                       // nullable — маркер «не задано»
>     private string _currency = "UZS";
>     private PaymentStatus _status = PaymentStatus.Pending;
>     private string? _providerRef;
>     private DateTimeOffset _createdAt = new(2026, 1, 1, 0, 0, 0, TimeSpan.Zero);
>
>     public PaymentBuilder WithAmount(decimal amount)
>     {
>         ArgumentOutOfRangeException.ThrowIfNegativeOrZero(amount);
>         _amount = amount;
>         return this;
>     }
>
>     public PaymentBuilder InCurrency(string currency) { _currency = currency; return this; }
>     public PaymentBuilder Status(PaymentStatus s) { _status = s; return this; }
>     public PaymentBuilder FromProvider(string reference) { _providerRef = reference; return this; }
>     public PaymentBuilder CreatedAt(DateTimeOffset at) { _createdAt = at; return this; }
>
>     public Payment Build()
>     {
>         // Понятное сообщение вместо тихого нуля.
>         if (_amount is null)
>             throw new InvalidOperationException(
>                 $"{nameof(PaymentBuilder)}: не задана сумма. Вызовите {nameof(WithAmount)}().");
>
>         // Валидация целого: Succeeded без ссылки провайдера невозможен.
>         if (_status is PaymentStatus.Succeeded && _providerRef is null)
>             throw new InvalidOperationException(
>                 "Успешный платёж обязан иметь ссылку провайдера.");
>
>         return new Payment(_amount.Value, _currency, _status, _providerRef, _createdAt);
>     }
>
>     public static implicit operator Payment(PaymentBuilder b) => b.Build();
> }
> ```
>
> Ключевые приёмы: `decimal?` как маркер «не задано» (0 — валидное значение типа, поэтому по нему нельзя различить «не задали» и «задали ноль»); фиксированная дата по умолчанию вместо `DateTimeOffset.UtcNow`, чтобы тесты были детерминированными; валидация комбинации полей в `Build()` — то, что билдер умеет, а объектный инициализатор нет.

### Задача 4. Почему `with` не помог

Разработчик написал функцию отката черновика заказа и жалуется, что снапшот меняется вместе с оригиналом. Объясните и исправьте.

```csharp
public record OrderDraft(string Customer, List<OrderLine> Lines);

var draft = new OrderDraft("ivan@example.com", [new OrderLine("SKU-1", 1, 10m)]);
var snapshot = draft with { };

draft.Lines.Add(new OrderLine("SKU-2", 5, 99m));
Console.WriteLine(snapshot.Lines.Count);   // ожидалось 1
```

> [!success]- Решение
> Выведет `2`. `with` делает поверхностную копию: компилятор генерирует копирующий конструктор, который присваивает поля один к одному. Поле `Lines` — ссылка, копируется ссылка, а не список. Оба record'а указывают на один `List<OrderLine>`.
>
> Правильно — сделать состав иммутабельным на уровне типа, а не чинить в месте копирования:
>
> ```csharp
> public record OrderDraft(string Customer, ImmutableArray<OrderLine> Lines)
> {
>     // Мутация теперь возвращает новый объект — «изменить оригинал» физически нельзя.
>     public OrderDraft AddLine(OrderLine line) => this with { Lines = Lines.Add(line) };
> }
>
> var draft = new OrderDraft("ivan@example.com", [new OrderLine("SKU-1", 1, 10m)]);
> var snapshot = draft;                       // копия даже не нужна: тип иммутабельный
> var updated = draft.AddLine(new OrderLine("SKU-2", 5, 99m));
>
> Console.WriteLine(snapshot.Lines.Length);   // Вывод: 1
> Console.WriteLine(updated.Lines.Length);    // Вывод: 2
> ```
>
> Мораль: иммутабельный record с мутабельной коллекцией внутри — мнимая иммутабельность. Если оставить `List<T>` необходимо, `with` придётся дополнять явным копированием: `draft with { Lines = [.. draft.Lines] }` — но это уже ручной deep copy, и лучше назвать его методом с честным именем.

## Итог

- Порождающие паттерны GoF — это в основном компенсация отсутствовавших языковых средств. В C# 14 три из пяти покрываются DI-контейнером и `record`.
- Singleton-как-статика вреден не «одним экземпляром», а глобальной точкой доступа: скрытая зависимость, невозможность подмены в тестах, зашитый жизненный цикл, риск захвата scoped-зависимости. `AddSingleton` даёт тот же экземпляр с явной зависимостью.
- Factory Method жив там, где фабрика управляет жизненным циклом невидимого ресурса (`IHttpClientFactory` и пул handler'ов) или где тип выбирается по данным рантайма. Одиночный метод `Create` заменяется `Func<T>` или keyed-сервисом.
- Abstract Factory нужна ровно тогда, когда рассогласование семейства должно быть невозможно по конструкции. Иначе — открытый дженерик или keyed-регистрация, ценой того, что ключ — строка без compile-time-проверки.
- Builder оправдан валидацией целого в `Build()`, фазой «конфигурация закрыта» и тестами. Без этого `record` + `required` + `with` читается лучше и проверяется компилятором.
- `ICloneable` сломан отсутствием контракта на глубину копирования; `with` копирует поверхностно; копирование через JSON теряет приватные поля, ссылочную идентичность и полиморфизм — в горячем пути не применять.

## Связанное

- [[Паттерны GoF: структурные]]
- [[Паттерны GoF: поведенческие]]
- [[Паттерны в .NET: где они уже используются]]
- [[Keyed services и продвинутая регистрация]]
- [[Жизненные циклы сервисов: Singleton, Scoped, Transient]]
- [[Captive dependency и типичные ошибки DI]]
- [[Тестовые данные: Bogus и билдеры]]
- [[Записи (record) и структуры]]
