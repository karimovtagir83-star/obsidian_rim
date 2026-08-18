---
tags: [раздел-10, архитектура, паттерны, mediatr, cqrs, лицензии, middle, собес, подводный-камень]
aliases: [MediatR, Mediator pattern in .NET, Медиатор, In-process messaging, MediatR alternatives]
---

# MediatR и альтернативы

> [!abstract] Коротко
> MediatR — библиотека, реализующая паттерн Mediator внутри процесса: код отправляет объект-запрос, а библиотека находит обработчик и вызывает его. Главная её практическая ценность — не «развязка», а pipeline behaviors: одна точка, куда вешаются валидация, логирование, транзакция и метрики для всех команд сразу. С июля 2025 MediatR коммерческая: версии 13 и выше требуют либо лицензии, либо копилефтной RPL-1.5, и это в 2026 году главный фактор выбора. Мидл обязан уметь и обосновать MediatR, и обосновать отказ от него, и написать свою версию на 50 строк.

## Зачем это нужно

Возьмём типичный обработчик HTTP-запроса без всякого медиатора:

```csharp
app.MapPost("/orders", async (CreateOrderDto dto, OrderService service, IValidator<CreateOrderDto> validator,
                              AppDbContext db, ILogger<Program> log, CancellationToken ct) =>
{
    var validation = await validator.ValidateAsync(dto, ct);
    if (!validation.IsValid) return Results.ValidationProblem(validation.ToDictionary());

    var sw = Stopwatch.StartNew();
    await using var tx = await db.Database.BeginTransactionAsync(ct);
    var id = await service.CreateAsync(dto, ct);
    await db.SaveChangesAsync(ct);
    await tx.CommitAsync(ct);
    log.LogInformation("Заказ {OrderId} создан за {Elapsed} мс", id, sw.ElapsedMilliseconds);

    return Results.Created($"/orders/{id}", new { id });
});
```

Теперь представьте сорок таких эндпоинтов. Валидация, транзакция, замер времени, лог, обработка ошибок — копипаста в каждом. Стоит забыть `CommitAsync` в одном месте — и появляется баг, который не поймает ни один тест, потому что тест этот путь не покрывает.

Хочется, чтобы «вокруг» каждой операции автоматически оборачивались одни и те же слои, а сам обработчик содержал только бизнес-логику. Именно это и продаёт MediatR: команда становится объектом, обработчик — классом с единственным методом, а всё окружающее — набором behaviors, зарегистрированных один раз.

Оборотная сторона: вы вводите в проект слой косвенности, который прячет вызовы от IDE и от читателя. Дальше — про то, стоит ли это того в 2026 году, и с чего начинается разговор.

---

## Лицензионный статус на август 2026

Этот раздел идёт первым не из юридического педантизма. В 2026 году именно лицензия, а не архитектурные вкусы, определяет, появится MediatR в новом проекте или нет.

### Что произошло

Jimmy Bogard, автор MediatR и AutoMapper, передал библиотеки в компанию **Lucky Penny Software**. Репозиторий переехал на `github.com/LuckyPennySoftware/MediatR`. **2 июля 2025** вышел первый коммерческий релиз — версия **13.0.0**.

| Версия MediatR | Лицензия | Стоимость | Комментарий |
|---|---|---|---|
| **12.5.0 и ниже** | Apache 2.0 | бесплатно, без условий | Остаются свободными навсегда. Развития не будет |
| **13.0.0+** (июль 2025) | двойная: RPL-1.5 **или** коммерческая | бесплатно только по RPL-1.5 либо по ключу | Первый коммерческий релиз |
| **14.2.0** (2 июля 2026) | двойная: RPL-1.5 **или** коммерческая | то же | Актуальная на август 2026. Таргеты `net8.0`, `netstandard2.0`, `net462`; на .NET 10 работает |

Пакет на NuGet называется по-прежнему **MediatR** — то есть обычное `dotnet add package MediatR` без указания версии затянет коммерческую 14.x. Это самая частая ловушка: разработчик обновляет пакеты «по кнопке» и незаметно меняет юридический статус продукта.

> [!danger] RPL-1.5 — не MIT
> **RPL-1.5** (Reciprocal Public License) — это сильный копилефт с «реципрокным» условием: производные работы и, в трактовке лицензии, использование в сервисе требуют раскрытия исходников. Для закрытого коммерческого продукта или внутреннего сервиса компании она практически неприменима. Если вы взяли MediatR 13+ и не купили лицензию — вы формально работаете по RPL-1.5 со всеми её последствиями. Без валидного лицензионного ключа библиотека при старте пишет предупреждение в лог; категория логгера содержит `LuckyPennySoftware.MediatR.License`. Если в проде в логах внезапно появилась такая строка — это сигнал, что кто-то обновил пакет.

### Тиры

Есть бесплатная **Community**-редакция: неограниченное число разработчиков и деплоев. Но она доступна только организации, удовлетворяющей **всем** условиям одновременно:

- годовая выручка (для НКО — годовой бюджет) **меньше 5 000 000 USD**;
- организация **никогда** не получала больше **10 000 000 USD** внешнего капитала (венчурного или private equity);
- организация **не является** государственным учреждением или вузом.

Регистрация self-service, по результату выдаётся лицензионный ключ.

Платные тиры считаются по числу разработчиков, **реально пишущих код**: QA, дизайн и менеджмент не учитываются.

| Тир | Разработчиков | Цена в год | Цена в месяц | Дополнительно |
|---|---|---|---|---|
| **Community** | без ограничений | 0 USD | 0 USD | Только при выполнении всех трёх условий выше |
| **Standard** | 1–10 | 799 USD | 80 USD | — |
| **Professional** | 11–50 | 1499 USD | 150 USD | — |
| **Enterprise** | без ограничений | 6399 USD | 640 USD | Приоритетные багфиксы, email-поддержка |

> [!warning] Цифры надо перепроверять
> Лицензионные условия и цены меняются, а эта заметка — снимок на август 2026. Перед покупкой или перед юридическим решением обязательно проверьте актуальные условия на сайте Lucky Penny Software и покажите текст лицензии юристу. Не принимайте решение о лицензировании по заметке в чьей-то базе знаний — включая эту.

### То же самое случилось с соседями

- **AutoMapper** — та же история: тот же владелец, коммерческая модель с 2025 года, те же тиры. Для AutoMapper это, кстати, куда более простой разговор: маппинг руками или через source-generator-библиотеку (Mapperly) почти всегда лучше AutoMapper по читаемости и производительности, так что «выпилить» — реалистичный вариант.
- **MassTransit v9** (2025) тоже стал коммерческим: порядок цен — от ~400 USD/мес, с бесплатным тиром для организаций с выручкой менее 1 млн USD. Версия **v8** остаётся open-source, но у неё **EOL в конце 2026**. Подробнее — [[MassTransit]].

Вывод из этого тренда шире, чем MediatR: инфраструктурная зависимость, которая «просто есть у всех», — это тоже риск. Оценивайте библиотеки не только по API, но и по тому, что будет, если её лицензия изменится.

### Что делать существующему проекту

Четыре варианта, все рабочие. Выбор зависит от того, сколько у вас команд, кода и денег.

| Вариант | Трудозатраты | Риск | Когда разумно |
|---|---|---|---|
| **Заморозить на 12.5.0** | почти нулевые | средний, растёт со временем | Проект стабилен, обновляться не планируете |
| **Купить лицензию** | нулевые технически | нулевой | Компания платит за инструменты, MediatR глубоко врос |
| **Миграция на альтернативу** | от дня до недель | низкий | Есть время, хочется убрать лицензионный вопрос навсегда |
| **Выпилить медиатор совсем** | недели, зато чище | низкий, но затрагивает много файлов | Медиатор использовался как «модно», а не по нужде |

Про заморозку на 12.5.0 нужно сказать честно, потому что это самый популярный выбор и его обычно недоговаривают:

- багфиксов **не будет**, включая потенциальные проблемы безопасности;
- поддержки новых TFM **не будет** — 12.5.0 таргетирует старые платформы; сегодня на .NET 10 это работает, но за .NET 12/13 никто не поручится;
- добавьте `<PackageReference Include="MediatR" Version="[12.5.0]" />` — квадратные скобки задают точную версию и защищают от случайного апгрейда;
- запретите обновление в Renovate/Dependabot явным правилом и оставьте комментарий в `Directory.Packages.props`, почему версия зафиксирована. Иначе через полгода новый разработчик «почистит зависимости».

---

## Что паттерн Mediator на самом деле решает

Mediator — поведенческий паттерн из каталога GoF (см. [[Паттерны GoF: что реально встречается]]). В оригинале он про то, что множество объектов не общаются друг с другом напрямую, а через посредника, чтобы не превратиться в граф «все со всеми».

MediatR реализует узкую версию этой идеи:

```
Без медиатора:                       С медиатором:

Endpoint ──> OrderService            Endpoint ──> ISender.Send(new CreateOrder(...))
         ──> EmailService                              │
         ──> StockService                              ▼
                                             (резолв обработчика по типу)
                                                       │
                                                       ▼
                                              CreateOrderHandler
```

**Что решается реально:** отправитель не имеет ссылки на тип обработчика. Эндпоинт зависит только от `ISender` и от типа команды. Это даёт две настоящие выгоды: одна точка для кросс-каттинга (behaviors) и однородность — все операции выглядят одинаково.

**Что НЕ решается**, вопреки распространённому убеждению:

| Заблуждение | Реальность |
|---|---|
| «Это развязка компонентов» | Развязки во времени нет: `Send` — обычный `await` метода в том же потоке |
| «Обработчик выполняется отдельно» | Тот же процесс, тот же DI-скоуп, та же транзакция, тот же `HttpContext` |
| «Это очередь сообщений» | Нет ни персистентности, ни ретраев, ни доставки при перезапуске. Упало — потеряли |
| «Publish даёт event-driven» | `IPublisher.Publish` по умолчанию вызывает обработчики **последовательно**, ждёт всех, и исключение первого прерывает остальные |
| «Можно потом легко вынести в микросервис» | Границу вы действительно очертили, но транспорт, сериализация, идемпотентность и outbox всё равно придётся писать заново |

Если нужна настоящая асинхронность и надёжность — это [[Паттерн Transactional Outbox]], брокер и [[Способы межсервисного взаимодействия]], а не медиатор. Смешивать эти две вещи в голове — самая частая ошибка на собеседовании.

---

## API MediatR: минимум, который нужно знать

### Запрос и команда

```csharp
// Команда: меняет состояние, возвращает идентификатор созданного заказа.
// IRequest<TResponse> — маркерный интерфейс, несущий тип ответа в системе типов.
public sealed record CreateOrder(Guid CustomerId, IReadOnlyList<OrderLine> Lines) : IRequest<Guid>;

public sealed class CreateOrderHandler(AppDbContext db, TimeProvider clock)
    : IRequestHandler<CreateOrder, Guid>
{
    public async Task<Guid> Handle(CreateOrder request, CancellationToken ct)
    {
        var order = Order.Create(request.CustomerId, request.Lines, clock.GetUtcNow());
        db.Orders.Add(order);
        // SaveChangesAsync здесь НЕ вызываем: это делает транзакционный behaviour ниже
        return order.Id;
    }
}

// Запрос: только читает. Проекция сразу в DTO, без загрузки сущностей в трекер.
public sealed record GetOrderById(Guid Id) : IRequest<OrderDto?>;

public sealed class GetOrderByIdHandler(AppDbContext db) : IRequestHandler<GetOrderById, OrderDto?>
{
    public Task<OrderDto?> Handle(GetOrderById request, CancellationToken ct) =>
        db.Orders
          .AsNoTracking()
          .Where(o => o.Id == request.Id)
          .Select(o => new OrderDto(o.Id, o.CustomerId, o.Total, o.CreatedAt))
          .FirstOrDefaultAsync(ct);
}
```

Разделение на «команды меняют, запросы читают» — это [[CQRS]] в самом простом виде. MediatR его не требует, но именно с ним обычно применяется.

### Уведомления

```csharp
// INotification: получателей может быть 0, 1 или много. Возвращаемого значения нет.
public sealed record OrderCreated(Guid OrderId, Guid CustomerId) : INotification;

public sealed class SendConfirmationEmail(IEmailSender sender) : INotificationHandler<OrderCreated>
{
    public Task Handle(OrderCreated notification, CancellationToken ct) =>
        sender.SendOrderConfirmationAsync(notification.OrderId, ct);
}

public sealed class ReserveStock(IStockService stock) : INotificationHandler<OrderCreated>
{
    public Task Handle(OrderCreated notification, CancellationToken ct) =>
        stock.ReserveAsync(notification.OrderId, ct);
}
```

### Отправка и регистрация

```csharp
// ISender — только Send (запросы/команды). IPublisher — только Publish (уведомления).
// Разделение появилось в MediatR 12: если эндпоинту нужно только Send,
// не давайте ему возможность публиковать события.
app.MapPost("/orders", async (CreateOrderDto dto, ISender sender, CancellationToken ct) =>
{
    var id = await sender.Send(new CreateOrder(dto.CustomerId, dto.Lines), ct);
    return Results.Created($"/orders/{id}", new { id });
});
```

```csharp
// Program.cs
builder.Services.AddMediatR(cfg =>
{
    // Сканирование сборки: находит все IRequestHandler / INotificationHandler
    cfg.RegisterServicesFromAssembly(typeof(CreateOrder).Assembly);

    // Открытые дженерик-behaviors. ПОРЯДОК РЕГИСТРАЦИИ = ПОРЯДОК ВЫПОЛНЕНИЯ.
    cfg.AddOpenBehavior(typeof(LoggingBehavior<,>));       // самый внешний
    cfg.AddOpenBehavior(typeof(ValidationBehavior<,>));    // затем
    cfg.AddOpenBehavior(typeof(TransactionBehavior<,>));   // самый внутренний, ближе к обработчику

    // В MediatR 13+ здесь же задаётся лицензионный ключ:
    // cfg.LicenseKey = builder.Configuration["MediatR:LicenseKey"];
});
```

Обработчики по умолчанию регистрируются как `Transient`, `IMediator` — как `Transient` (настраивается через `cfg.Lifetime`). Про последствия — [[Жизненные циклы сервисов: Singleton, Scoped, Transient]] и [[Captive dependency и типичные ошибки DI]].

---

## Pipeline behaviors — главная ценность

`IPipelineBehavior<TRequest, TResponse>` — это matryoshka вокруг обработчика, ровно как middleware вокруг эндпоинта ([[Middleware и конвейер обработки запроса]]).

```
Send(CreateOrder)
   │
   ├─ LoggingBehavior       ─── до
   │    ├─ ValidationBehavior ─── до
   │    │    ├─ TransactionBehavior ─── BeginTransaction
   │    │    │    └─ CreateOrderHandler  ← полезная работа
   │    │    ├─ TransactionBehavior ─── SaveChanges + Commit
   │    ├─ ValidationBehavior ─── (после — ничего)
   ├─ LoggingBehavior       ─── лог + метрика
   ▼
результат
```

### Behaviour 1: валидация

```csharp
using FluentValidation;
using MediatR;

// Проверяет запрос ДО обработчика. Обработчик получает только валидные данные
// и может не писать защитных if-ов. Про правила — [[FluentValidation]].
public sealed class ValidationBehavior<TRequest, TResponse>(
    IEnumerable<IValidator<TRequest>> validators)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct)
    {
        // Валидаторов может не быть вовсе — тогда просто пропускаем дальше.
        if (!validators.Any())
            return await next(ct);

        var results = await Task.WhenAll(validators.Select(v => v.ValidateAsync(request, ct)));
        var failures = results.SelectMany(r => r.Errors).Where(f => f is not null).ToArray();

        if (failures.Length > 0)
            throw new ValidationException(failures);

        return await next(ct);
    }
}
```

> [!info] Исключение или Result
> Бросать `ValidationException` и ловить её в глобальном обработчике ([[Обработка ошибок и ProblemDetails]]) — просто, но использует исключения как поток управления. Альтернатива — вернуть `Result<T>` с ошибками; тогда behaviour должен уметь сконструировать `TResponse`, что требует ограничения вида `where TResponse : IResult` и фабрики. Разбор компромисса — [[Result pattern вместо исключений]].

### Behaviour 2: логирование и метрики

```csharp
using System.Diagnostics;
using System.Diagnostics.Metrics;
using MediatR;
using Microsoft.Extensions.Logging;

public sealed class LoggingBehavior<TRequest, TResponse>(
    ILogger<LoggingBehavior<TRequest, TResponse>> logger,
    IMeterFactory meterFactory)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    private static readonly string RequestName = typeof(TRequest).Name;

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct)
    {
        var meter = meterFactory.Create("App.Requests");
        var duration = meter.CreateHistogram<double>("app.request.duration", "ms");

        // BeginScope добавляет RequestName во ВСЕ логи внутри обработчика.
        // Это и есть смысл структурных логов: см. [[Логирование и структурные логи]].
        using var scope = logger.BeginScope(new Dictionary<string, object>
        {
            ["RequestName"] = RequestName,
            ["RequestId"] = Activity.Current?.Id ?? "none"
        });

        logger.LogInformation("Начало обработки {RequestName}", RequestName);
        var sw = Stopwatch.StartNew();

        try
        {
            var response = await next(ct);
            sw.Stop();

            // Плейсхолдеры в фигурных скобках — структурные поля, а не строковая интерполяция.
            // Именно поэтому по RequestName потом можно фильтровать в Loki/Seq.
            logger.LogInformation("{RequestName} обработан за {ElapsedMs} мс",
                RequestName, sw.Elapsed.TotalMilliseconds);

            duration.Record(sw.Elapsed.TotalMilliseconds,
                new KeyValuePair<string, object?>("request", RequestName),
                new KeyValuePair<string, object?>("outcome", "success"));

            return response;
        }
        catch (Exception ex)
        {
            sw.Stop();
            logger.LogError(ex, "{RequestName} упал за {ElapsedMs} мс",
                RequestName, sw.Elapsed.TotalMilliseconds);

            duration.Record(sw.Elapsed.TotalMilliseconds,
                new KeyValuePair<string, object?>("request", RequestName),
                new KeyValuePair<string, object?>("outcome", "failure"));
            throw;
        }
    }
}
```

Одна регистрация — и у вас появилась гистограмма длительности **каждой** бизнес-операции с разбивкой по имени. Вот это действительно трудно повторить сорока копипастами.

### Behaviour 3: транзакция вокруг команды

```csharp
using MediatR;
using Microsoft.EntityFrameworkCore;

// Маркер: транзакция нужна только командам, не запросам.
public interface ITransactionalRequest;

public sealed class TransactionBehavior<TRequest, TResponse>(
    AppDbContext db,
    IPublisher publisher,
    ILogger<TransactionBehavior<TRequest, TResponse>> logger)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct)
    {
        // Запросы (IRequest без ITransactionalRequest) идут мимо транзакции.
        if (request is not ITransactionalRequest)
            return await next(ct);

        // Вложенный Send внутри уже открытой транзакции не должен открывать вторую.
        if (db.Database.CurrentTransaction is not null)
            return await next(ct);

        var strategy = db.Database.CreateExecutionStrategy();

        // ExecutionStrategy умеет повторять всю операцию при transient-сбое,
        // поэтому транзакцию открываем ВНУТРИ него. Подробнее —
        // [[EF Core: транзакции и конкурентность]].
        return await strategy.ExecuteAsync(async () =>
        {
            await using var tx = await db.Database.BeginTransactionAsync(ct);

            var response = await next(ct);

            // Собираем доменные события до сохранения: они уже накоплены в сущностях.
            var events = db.ChangeTracker.Entries<IHasDomainEvents>()
                .SelectMany(e => e.Entity.DequeueDomainEvents())
                .ToList();

            await db.SaveChangesAsync(ct);

            // Диспетчеризация ПОСЛЕ SaveChanges, но ДО Commit:
            // обработчики событий пишут в ту же транзакцию.
            // Подробнее — [[Доменные события]].
            foreach (var domainEvent in events)
                await publisher.Publish(domainEvent, ct);

            // Обработчики могли добавить новые изменения.
            if (db.ChangeTracker.HasChanges())
                await db.SaveChangesAsync(ct);

            await tx.CommitAsync(ct);
            logger.LogDebug("Транзакция для {Request} зафиксирована", typeof(TRequest).Name);
            return response;
        });
    }
}
```

### Порядок выполнения

**Порядок behaviors равен порядку регистрации.** Первый зарегистрированный — самый внешний. Это следствие того, как контейнер отдаёт коллекцию `IEnumerable<IPipelineBehavior<,>>`: MediatR берёт её в порядке регистрации и сворачивает в цепочку делегатов с конца.

Практические следствия:

- **логирование — первым**, иначе оно не увидит ни времени валидации, ни исключений внешних слоёв;
- **валидация — до транзакции**, иначе на каждый невалидный запрос вы будете открывать и откатывать транзакцию, зря расходуя соединение из пула;
- **транзакция — последней**, максимально близко к обработчику, чтобы транзакция жила как можно короче;
- кеширование ответов ставят между логированием и валидацией — так попадание в кеш видно в логах, но не тратит время на валидацию.

> [!warning] Порядок легко сломать
> `cfg.RegisterServicesFromAssembly` тоже подхватывает behaviors, если они найдены сканированием, и тогда фактический порядок зависит от порядка типов в сборке — то есть неопределён. Регистрируйте behaviors **только** явно через `AddOpenBehavior`/`AddBehavior` и держите их в одном файле, чтобы порядок был виден глазами.

---

## Критика: за что MediatR ругают, и что на это отвечают

Разбирать по-честному, потому что на собеседовании про это спрашивают, а в проекте вам с этим жить.

### Обоснованные претензии

**Лишний слой косвенности.** Между «HTTP пришёл» и «бизнес-логика выполнилась» появляется резолв по типу через DI. Для отладки это плюс-минус один прыжок; для понимания — целый концептуальный слой, который нужно объяснить новичку.

**«Go to definition» не работает.** F12 на `sender.Send(...)` уводит в интерфейс `ISender`, а не в `CreateOrderHandler`. Придётся искать обработчик по имени команды через полнотекстовый поиск. Rider и ReSharper частично лечат это навигацией по MediatR, но в чистой VS Code с C# Dev Kit — нет.

**Типы команд как единственная связь.** Компилятор проверяет только то, что `CreateOrder : IRequest<Guid>`. Наличие обработчика он не проверяет **вообще**. Удалили обработчик или переименовали сборку так, что сканирование её не нашло — код собирается, а на рантайме летит `InvalidOperationException: Handler was not found`. Рефакторинг ломается молча.

**Хендлеры-однострочники.** В CRUD-проекте вы получаете четыре файла (`GetOrderById`, `GetOrderByIdHandler`, валидатор, DTO) на один `SELECT` по первичному ключу. Соотношение церемонии к пользе становится абсурдным.

**Граф вызовов не виден.** Нельзя нажать «Find usages» на обработчике и увидеть, кто его вызывает. Кто отправляет `OrderCreated`? Неизвестно, пока не прогрепаете строку. В большой кодовой базе это реальная потеря.

**Медиатор как замаскированный Service Locator.** Обработчик, который внутри себя делает `_sender.Send(new AnotherCommand(...))`, скрыл зависимость от `AnotherCommandHandler`. Она не видна в конструкторе, не видна в тестах, не видна на графе зависимостей. Это ровно тот антипаттерн, от которого уходили к DI ([[Антипаттерны и code smells]]). Вложенные `Send` — самый явный признак, что медиатор используется не по назначению.

**Рефлексия и цена.** Сканирование сборки на старте — это `Assembly.GetTypes()` и перебор интерфейсов, десятки миллисекунд на средней сборке. На каждый `Send` — обращение к словарю кешированных wrapper-типов и обход `IEnumerable` behaviors. Для веб-запроса это шум, но для Native AOT ([[Native AOT: когда стоит]]) отражение и `MakeGenericType` — проблема принципиальная: обработчики не видны trimmer'у.

**Стек вызовов при отладке.** Исключение из обработчика приходит с стеком, в котором пять кадров behaviors, `RequestHandlerWrapperImpl`, лямбды и `MoveNext` асинхронных машин состояний. Найти реальную строку — упражнение на внимательность.

### Контраргументы, которые тоже правда

**Кросс-каттинг в одном месте.** Три behaviour выше заменяют примерно 40 × 15 строк копипасты. И, что важнее, обеспечивают, что ни один обработчик её не пропустит. Это не «красиво» — это дешевле в поддержке.

**Единообразие.** Любая операция в проекте выглядится одинаково: запись-команда, класс-обработчик, метод `Handle`. Новый разработчик после первого use case ориентируется во всех. В проекте с 200 use case это существенно.

**Дешёвая точка вставки поведения.** «Добавь идемпотентность всем командам», «пиши аудит по всем операциям с деньгами», «кешируй все запросы этого типа» — каждое из этих требований в мире behaviors решается одним классом и одной строкой регистрации. Без медиатора — обходом всех эндпоинтов.

**Естественная граница use case.** Тип команды — это явно названная бизнес-операция. Она отлично тестируется: собрали объект, вызвали `Handle`, проверили. Никакого HTTP, никакого `HttpContext`.

> [!tip] Как решать
> Спросите себя: **сколько у меня будет кросс-каттинга и сколько use case?** Пять эндпоинтов и одна валидация — медиатор лишний, хватит endpoint filters. Двести операций, из которых половине нужна транзакция, аудит и идемпотентность — behaviors себя оправдывают. Между этими полюсами решает вкус команды, и это нормально. См. [[Как выбрать архитектуру под задачу]].

---

## Альтернативы

| Библиотека | Лицензия | Механизм | AOT | Pipeline | Месседжинг | Порог входа | Близость API к MediatR |
|---|---|---|---|---|---|---|---|
| **Wolverine** (JasperFx) | MIT | кодогенерация | частично | да (middleware) | да: брокеры, outbox, саги | средний | низкая: другая модель |
| **Immediate.Handlers** | MIT | source generator | да | да (behaviors) | нет | низкий | средняя |
| **martinothamar/Mediator** | MIT | source generator | да | да | нет | низкий | **очень высокая** |
| **LiteBus** | MIT | рефлексия | нет | да | нет | низкий | средняя, CQS-first |
| **Brighter** | MIT | рефлексия | нет | да | да | средний | средняя |
| **Rebus** | MIT | рефлексия | нет | да | да: шина сообщений | средний | низкая |
| **FastEndpoints** | Apache 2.0 | рефлексия + REPR | нет | да (фильтры) | встроенные команды/события | низкий | низкая |
| **Shiny.Mediator** | MIT | source generator | да | да | частично | низкий | средняя |
| **Своя реализация** | ваша | что напишете | да | да | нет | нулевой | какая захотите |
| **Никакого медиатора** | — | обычный DI | да | endpoint filters | нет | нулевой | — |

Практические заметки по выбору:

- **Хотите минимальный дифф при миграции** — берите `martinothamar/Mediator`. Пространство имён и интерфейсы намеренно повторяют MediatR: часто миграция — это замена пакета, правка `using` и регистрации. Плюс source generator: никакой рефлексии, всё разрешается на компиляции.
- **Хотите ещё и месседжинг** — Wolverine. Он закрывает и медиатор, и брокер, и outbox, и саги одной моделью. Если вы всё равно смотрели на MassTransit v9 и испугались цены — это направление.
- **Хотите AOT и минимум магии** — Immediate.Handlers. У него есть спутник **Immediate.Apis**, который сам генерирует эндпоинты Minimal API из обработчиков.
- **Хотите чёткое CQS на уровне типов** — LiteBus: у него отдельные абстракции для команд, запросов и событий, а не один `IRequest` на всё.

### Wolverine: обработчик — обычный метод

```csharp
// Никаких интерфейсов и базовых классов. Обнаружение по соглашению:
// публичный класс с суффиксом Handler и публичным методом Handle/Consume.
public static class CreateOrderHandler
{
    // Зависимости внедряются как ПАРАМЕТРЫ МЕТОДА, а не через конструктор.
    // Wolverine генерирует код, который их достаёт из DI, — рефлексии на вызове нет.
    public static async Task<Guid> Handle(
        CreateOrder command,
        AppDbContext db,
        TimeProvider clock,
        CancellationToken ct)
    {
        var order = Order.Create(command.CustomerId, command.Lines, clock.GetUtcNow());
        db.Orders.Add(order);
        await db.SaveChangesAsync(ct);
        return order.Id;
    }
}

// Отправка:
// await bus.InvokeAsync<Guid>(new CreateOrder(...), ct);   // локально, синхронно
// await bus.SendAsync(new CreateOrder(...));               // через брокер, если настроен
```

Плюс: обработчик — статический метод без наследования, тестируется вызовом напрямую. Минус: обнаружение по соглашению — «магия по имени», компилятор её не проверяет; кодогенерация Wolverine усложняет чтение стеков; и порог входа выше, чем кажется, из-за большого числа возможностей.

### Immediate.Handlers: source generator

```csharp
using Immediate.Handlers.Shared;

// Атрибут говорит генератору: сделай для этого типа обработчик-обёртку и pipeline.
[Handler]
public static partial class GetOrderById
{
    public sealed record Query(Guid Id);

    // Метод HandleAsync генератор находит по соглашению внутри partial-класса.
    private static async ValueTask<OrderDto?> HandleAsync(
        Query query,
        AppDbContext db,
        CancellationToken ct)
        => await db.Orders
            .AsNoTracking()
            .Where(o => o.Id == query.Id)
            .Select(o => new OrderDto(o.Id, o.CustomerId, o.Total, o.CreatedAt))
            .FirstOrDefaultAsync(ct);
}

// Использование: генератор создаёт GetOrderById.Handler, его и внедряем.
app.MapGet("/orders/{id:guid}", async (Guid id, GetOrderById.Handler handler, CancellationToken ct) =>
{
    var dto = await handler.HandleAsync(new GetOrderById.Query(id), ct);
    return dto is null ? Results.NotFound() : Results.Ok(dto);
});
```

Ключевое отличие от MediatR: обработчик внедряется **по конкретному типу**. F12 работает, «Find usages» работает, компилятор проверяет наличие обработчика, ноль рефлексии, AOT-совместимо. Про механику — [[Source Generators]].

---

## Своя реализация на ~50 строк

Полезно не только как альтернатива, но и чтобы перестать считать MediatR магией. Код ниже компилируется целиком.

```csharp
using System.Collections.Concurrent;
using System.Reflection;
using Microsoft.Extensions.DependencyInjection;

namespace MiniMediator;

// ── Контракты ──────────────────────────────────────────────────────────────

// Маркер запроса. out TResponse делает интерфейс ковариантным,
// благодаря чему Send<TResponse>(IRequest<TResponse>) выводит тип ответа.
public interface IRequest<out TResponse>;

public interface IRequestHandler<in TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    Task<TResponse> Handle(TRequest request, CancellationToken ct);
}

// Следующее звено конвейера. Принимает токен, чтобы behaviour мог его подменить
// (например, добавить таймаут через CreateLinkedTokenSource).
public delegate Task<TResponse> RequestHandlerDelegate<TResponse>(CancellationToken ct);

public interface IPipelineBehavior<in TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct);
}

public interface IDispatcher
{
    Task<TResponse> Send<TResponse>(IRequest<TResponse> request, CancellationToken ct = default);
}

// ── Реализация ─────────────────────────────────────────────────────────────

public sealed class Dispatcher(IServiceProvider provider) : IDispatcher
{
    // Кеш обёрток: MakeGenericType дорог, вызывать его на каждый Send нельзя.
    private static readonly ConcurrentDictionary<Type, object> Wrappers = new();

    public Task<TResponse> Send<TResponse>(IRequest<TResponse> request, CancellationToken ct = default)
    {
        ArgumentNullException.ThrowIfNull(request);

        // Проблема: на этапе компиляции мы знаем только TResponse, но не конкретный
        // тип запроса. Решение — обёртка, замкнутая по обоим типам, созданная один раз.
        var wrapper = (Wrapper<TResponse>)Wrappers.GetOrAdd(
            request.GetType(),
            requestType => Activator.CreateInstance(
                typeof(WrapperImpl<,>).MakeGenericType(requestType, typeof(TResponse)))!);

        return wrapper.Send(request, provider, ct);
    }

    private abstract class Wrapper<TResponse>
    {
        public abstract Task<TResponse> Send(IRequest<TResponse> request, IServiceProvider sp, CancellationToken ct);
    }

    private sealed class WrapperImpl<TRequest, TResponse> : Wrapper<TResponse>
        where TRequest : IRequest<TResponse>
    {
        public override Task<TResponse> Send(IRequest<TResponse> request, IServiceProvider sp, CancellationToken ct)
        {
            var handler = sp.GetRequiredService<IRequestHandler<TRequest, TResponse>>();
            var typed = (TRequest)request;

            // Ядро цепочки: обработчик — терминальное звено.
            RequestHandlerDelegate<TResponse> pipeline = token => handler.Handle(typed, token);

            // Оборачиваем в обратном порядке, чтобы ПЕРВЫЙ зарегистрированный
            // behaviour оказался САМЫМ ВНЕШНИМ — как в MediatR.
            var behaviors = sp.GetServices<IPipelineBehavior<TRequest, TResponse>>().Reverse();
            foreach (var behavior in behaviors)
            {
                var next = pipeline;              // копия в локальную переменную:
                var current = behavior;           // иначе замыкание захватит переменную цикла
                pipeline = token => current.Handle(typed, next, token);
            }

            return pipeline(ct);
        }
    }
}

// ── Регистрация ────────────────────────────────────────────────────────────

public static class MiniMediatorExtensions
{
    public static IServiceCollection AddMiniMediator(
        this IServiceCollection services, params Assembly[] assemblies)
    {
        services.AddScoped<IDispatcher, Dispatcher>();

        var handlerTypes = assemblies
            .SelectMany(a => a.GetTypes())
            .Where(t => t is { IsAbstract: false, IsInterface: false, IsGenericTypeDefinition: false });

        foreach (var type in handlerTypes)
        foreach (var iface in type.GetInterfaces()
                     .Where(i => i.IsGenericType &&
                                 i.GetGenericTypeDefinition() == typeof(IRequestHandler<,>)))
        {
            services.AddScoped(iface, type);
        }

        return services;
    }

    // Открытый дженерик: контейнер сам закроет его по нужным типам.
    public static IServiceCollection AddBehavior(this IServiceCollection services, Type openGeneric)
    {
        services.AddScoped(typeof(IPipelineBehavior<,>), openGeneric);
        return services;
    }
}
```

Использование ничем не отличается от MediatR:

```csharp
builder.Services.AddMiniMediator(typeof(CreateOrder).Assembly);
builder.Services.AddBehavior(typeof(LoggingBehavior<,>));
builder.Services.AddBehavior(typeof(TransactionBehavior<,>));
```

### Чего здесь нет по сравнению с MediatR

Честный список, чтобы не выяснить это в бою:

- **Уведомления** (`INotification` / `IPublisher`) со стратегиями публикации (`Parallel`, `Sequential`, свои). Добавляется ещё ~20 строками, но добавляется.
- **Стриминг** (`IStreamRequest<T>` / `IStreamPipelineBehavior`) — конвейер над `IAsyncEnumerable<T>`, см. [[IAsyncEnumerable и асинхронные потоки]].
- **`IRequestExceptionHandler` и `IRequestPreProcessor`/`IPostProcessor`** — точки расширения помимо behaviors. На практике всё то же делается обычным behaviour с `try/catch`.
- **Отправка `object`-запроса** (`Send(object request)`) — нужно, когда тип известен только на рантайме: dispatch по имени из очереди, из HTTP-роута, из скрипта.
- **Более агрессивное кеширование дескрипторов** и оптимизированный резолв.
- **Диагностика**: понятные сообщения об отсутствующем обработчике, проверки на этапе регистрации, интеграция с `ActivitySource`.

Для 90 % проектов этого набора не хватает ровно в одном месте — уведомления. Их добавляют, и на этом развитие останавливается. Итог: ~80 строк своего кода вместо внешней зависимости с лицензией. Цена: этот код теперь ваш, включая его баги.

### Вариант без медиатора вообще

Самый недооценённый. Обработчик остаётся классом, но внедряется напрямую — по своему типу.

```csharp
// Обработчик — обычный сервис. Никаких маркерных интерфейсов.
public sealed class CreateOrderHandler(AppDbContext db, TimeProvider clock)
{
    public async Task<Guid> HandleAsync(CreateOrder command, CancellationToken ct)
    {
        var order = Order.Create(command.CustomerId, command.Lines, clock.GetUtcNow());
        db.Orders.Add(order);
        await db.SaveChangesAsync(ct);
        return order.Id;
    }
}

// Регистрация — одна строка на обработчик. Скучно, зато компилятор всё проверяет.
builder.Services.AddScoped<CreateOrderHandler>();

// Эндпоинт зависит от КОНКРЕТНОГО типа: F12 работает, Find usages работает.
var orders = app.MapGroup("/orders")
    .AddEndpointFilter<ValidationFilter>()      // валидация
    .AddEndpointFilter<TransactionFilter>();    // транзакция

orders.MapPost("/", async (CreateOrder command, CreateOrderHandler handler, CancellationToken ct) =>
{
    var id = await handler.HandleAsync(command, ct);
    return Results.Created($"/orders/{id}", new { id });
});
```

Кросс-каттинг здесь берут на себя две существующие механики фреймворка:

- **Endpoint filters** — прямой аналог behaviors, но привязанный к эндпоинту, а не к типу команды. Работают с типизированными аргументами эндпоинта, применяются к группе через `MapGroup`. См. [[Фильтры и endpoint filters]].
- **Middleware** — для того, что должно работать до роутинга: корреляционный идентификатор, глобальная обработка исключений, метрики HTTP. См. [[Middleware и конвейер обработки запроса]].

Что теряется: кросс-каттинг привязан к транспорту. Если ту же команду надо выполнить из `BackgroundService` или из consumer'а RabbitMQ, endpoint filters не применятся — их придётся дублировать. Это и есть настоящий критерий: **если бизнес-операции вызываются только из HTTP, медиатор вам не нужен; если из HTTP, из брокера и из планировщика — нужна транспортно-независимая точка вставки поведения.**

---

> [!warning] Подводные камни

> [!warning] Подводные камни
> **1. `dotnet add package MediatR` тянет коммерческую версию.** Без явного `Version="[12.5.0]"` вы получите 14.x и юридический вопрос. Проверьте прямо сейчас, какая версия у вас в `Directory.Packages.props`, и есть ли в логах прода строки от `LuckyPennySoftware.MediatR.License`.
>
> **2. Отсутствующий обработчик — ошибка рантайма, а не компиляции.** Компилятор проверяет только `IRequest<T>`. Переименовали сборку, забыли `RegisterServicesFromAssembly` для нового проекта, перенесли обработчик — собирается, падает в проде. Лечится smoke-тестом, который резолвит обработчик для каждого типа `IRequest<>`, найденного отражением. Такой тест писать обязательно.
>
> **3. `Publish` — синхронный и последовательный.** По умолчанию `IPublisher.Publish` вызывает обработчики один за другим и **ждёт всех**. Исключение во втором обработчике не даст выполниться третьему, а первый уже сработал — получаете частично применённый эффект без возможности откатить. Если события критичны — outbox, а не `Publish`.
>
> **4. Порядок behaviors — это порядок регистрации, и его легко потерять.** Смешанная регистрация (часть через сканирование, часть явно) даёт порядок, зависящий от порядка типов в сборке. Симптом: валидация выполняется внутри транзакции, и на каждый невалидный запрос вы открываете и откатываете транзакцию.
>
> **5. Транзакционный behaviour ломается на вложенных `Send`.** Обработчик, отправляющий другую команду, попадёт во второй `BeginTransactionAsync` — EF Core бросит исключение о том, что транзакция уже начата. Проверка `db.Database.CurrentTransaction is not null` — минимально необходимая защита; лучше вообще запретить вложенные `Send` анализатором.
>
> **6. Behaviour, который «съедает» `CancellationToken`.** Если в behaviour вызвать `next()` без токена (в старых версиях сигнатура `next` была без параметров и токен передавался отдельно), отмена перестанет доходить до обработчика. Симптом: клиент отвалился, а запрос к базе продолжает жить и занимать соединение из пула.
>
> **7. Captive dependency через behaviour.** Behaviour, зарегистрированный как `Singleton`, но внедряющий `AppDbContext` (`Scoped`), либо не соберётся при валидации скоупов, либо утащит один контекст на всё приложение. Behaviors должны быть `Scoped` или `Transient`. См. [[Captive dependency и типичные ошибки DI]].
>
> **8. Native AOT и MediatR несовместимы без работы.** Сканирование сборок, `MakeGenericType`, `Activator.CreateInstance` — всё это trimmer не видит. Обработчики вырежутся, приложение упадёт на первом `Send`. Если целитесь в AOT — только source-generator-решения.

> [!example] Как делают в бою
> **Продуктовый монолит на 200+ use case, компания с выручкой выше порога Community.** Купили Standard за 799 USD/год и не думают об этом. Восемьсот долларов — это меньше двух дней работы разработчика; неделя миграции обошлась бы дороже. Ключ лежит в Key Vault и подставляется через конфигурацию.
>
> **Стартап, выручка 300 тысяч USD, венчур не поднимали.** Зарегистрировались на Community, получили ключ бесплатно. Единственная забота — не забыть, что при раунде на 15 млн условие «никогда не получала больше 10 млн внешнего капитала» перестанет выполняться. Это записали в ADR ([[Документация: ADR и RFC]]), чтобы вопрос всплыл на due diligence, а не после.
>
> **Аутсорс-контора с 60 разработчиками и 30 небольшими проектами.** Enterprise за 6399 USD выходил дороже, чем переписать шаблон стартера. Заменили MediatR на `martinothamar/Mediator` в шаблоне: замена пакета, правка `using`, правка регистрации, прогон тестов. На проект — от двух часов до дня. Ноль лицензионных вопросов на годы вперёд.
>
> **Новый сервис в 2026, команда из трёх человек, 15 эндпоинтов.** Не взяли медиатор вовсе. Обработчики — обычные классы, внедряются по типу; валидация и транзакция — endpoint filters на `MapGroup`. Ни одной строки инфраструктурного кода, полностью AOT-совместимо, навигация в IDE работает.
>
> **Легаси на MediatR 11 с 40 behaviors и вложенными `Send` в четыре уровня.** Здесь сначала выпрямляют вложенность (обработчики начинают зависеть от сервисов напрямую), и только потом обсуждают лицензию. Мигрировать неисправный код — удвоить проблему.

---

## Вопросы с собеседований

> [!question]- Что паттерн Mediator даёт, а чего не даёт?
> Даёт: отправитель не зависит от типа обработчика (только от типа сообщения и от `ISender`), появляется единая точка для кросс-каттинга через pipeline behaviors, все операции выглядят однородно. Не даёт: развязки во времени. `Send` — это обычный `await` метода в том же потоке, том же процессе, том же DI-скоупе и той же транзакции. Нет персистентности, нет ретраев, нет доставки после перезапуска процесса — упало, значит потеряли. Называть MediatR «шиной сообщений» — ошибка: шина требует брокера, а внутрипроцессная надёжность требует [[Паттерн Transactional Outbox]].

> [!question]- Какая главная практическая ценность MediatR и как она работает?
> Pipeline behaviors — `IPipelineBehavior<TRequest, TResponse>` с методом `Handle(request, next, ct)`. MediatR сворачивает зарегистрированные behaviors в цепочку делегатов вокруг обработчика: первый зарегистрированный оказывается самым внешним. Каждый behaviour решает, вызывать `next` или нет, и что делать до и после. Это даёт валидацию, логирование, метрики, транзакцию, кеширование и идемпотентность одной строкой регистрации на всё приложение. Внутренне цепочка собирается ровно так же, как конвейер middleware в ASP.NET Core: агрегирование делегатов в обратном порядке.

> [!question]- Каков порядок выполнения behaviors и как его контролировать?
> Порядок равен порядку регистрации: первый зарегистрированный — самый внешний. Практическая раскладка: логирование первым (иначе не увидит время валидации и внешние исключения), валидация вторым (до транзакции, чтобы невалидный запрос не открывал транзакцию зря), транзакция последней (чтобы жила максимально коротко). Контролировать нужно явной регистрацией через `AddOpenBehavior`, все вызовы — в одном месте. Если behaviors подхватываются сканированием сборки, порядок определяется порядком типов в сборке, то есть непредсказуем.

> [!question]- Почему MediatR называют замаскированным Service Locator?
> Потому что `_sender.Send(new SomeCommand(...))` внутри обработчика скрывает зависимость от `SomeCommandHandler`: её нет в конструкторе, она не видна на графе зависимостей, тест не заметит её появления. Это ровно та проблема, из-за которой Service Locator считается антипаттерном. Отличие MediatR в том, что через него проходят только явно объявленные типы сообщений, и на верхнем уровне (эндпоинт → обработчик) это оправдано: там отправитель и не должен знать обработчика. Вредно именно на уровне обработчик → обработчик. Правило: `Send` вызывается только на границе приложения, внутри обработчиков зависимости внедряются напрямую.

> [!question]- Что случилось с лицензией MediatR и какие варианты у проекта?
> В 2025 году библиотека перешла к Lucky Penny Software; версия 13.0.0 (2 июля 2025) стала коммерческой. Версии 12.5.0 и ниже остаются под Apache 2.0. Начиная с 13 — двойная лицензия: RPL-1.5 (сильный копилефт, для закрытого продукта неприменима) либо коммерческая. Есть бесплатная Community-редакция при выручке менее 5 млн USD, отсутствии более 10 млн внешнего капитала и статусе не-госучреждения. Платные тиры: Standard 1–10 разработчиков — 799 USD/год, Professional 11–50 — 1499 USD/год, Enterprise без ограничений — 6399 USD/год. Варианты для проекта: заморозить 12.5.0 (без багфиксов и новых TFM), купить лицензию, миграция на MIT-альтернативу, выпилить медиатор.

> [!question]- Какие бесплатные альтернативы MediatR есть в 2026 и чем отличаются?
> `martinothamar/Mediator` — source generator с API, максимально близким к MediatR: самая дешёвая миграция. `Immediate.Handlers` — source generator, AOT-friendly, обработчик внедряется по конкретному типу (навигация в IDE работает), есть спутник Immediate.Apis для Minimal API. `Wolverine` (MIT, JasperFx) — медиатор плюс месседжинг, outbox и саги; обработчик — обычный публичный метод, зависимости внедряются как параметры метода, обнаружение по соглашению, кодогенерация вместо рефлексии. `LiteBus` — CQS-first с отдельными абстракциями для команд, запросов и событий. `Brighter` и `Rebus` — MIT, с уклоном в месседжинг. `FastEndpoints` — REPR-паттерн со встроенными командами и событиями. Плюс всегда есть вариант «своя реализация на 80 строк» и «без медиатора вообще».

> [!question]- Как написать свой медиатор и в чём его сложность?
> Нужны: маркер `IRequest<out TResponse>` (ковариантный, чтобы выводился тип ответа), `IRequestHandler<in TRequest, TResponse>`, делегат следующего звена и `IPipelineBehavior<,>`. Основная техническая сложность — в `Send<TResponse>(IRequest<TResponse> request)` статически известен только `TResponse`, а обработчик нужно резолвить по конкретному типу запроса. Решается абстрактной обёрткой `Wrapper<TResponse>` и её дженерик-реализацией `WrapperImpl<TRequest, TResponse>`, созданной через `MakeGenericType` один раз и закешированной в `ConcurrentDictionary`. Дальше цепочка behaviors собирается агрегированием делегатов в обратном порядке. Не хватать будет уведомлений, стриминга, `Send(object)` и хорошей диагностики.

> [!question]- Когда медиатор точно не нужен?
> Когда бизнес-операции вызываются только из HTTP — тогда кросс-каттинг закрывается endpoint filters и middleware, которые уже есть во фреймворке и не требуют лишнего слоя. Когда use case мало (единицы-десятки) и кросс-каттинга почти нет. Когда проект — CRUD и обработчик состоит из одного вызова EF Core: четыре файла на один `SELECT` не окупаются. Когда цель — Native AOT: рефлексивные медиаторы там просто не работают. Когда команда не понимает, зачем медиатор нужен, — тогда он превратится в карго-культ и вложенные `Send` в четыре уровня.

---

## Задачи

### Задача 1. Behaviour идемпотентности

Напишите pipeline behaviour, который для команд с ключом идемпотентности не выполняет операцию повторно: если ключ уже видели, возвращает сохранённый результат. Ключ приходит в самой команде.

> [!success]- Решение
> ```csharp
> // Маркер: команда несёт ключ идемпотентности.
> public interface IIdempotentRequest
> {
>     Guid IdempotencyKey { get; }
> }
>
> public sealed class IdempotencyBehavior<TRequest, TResponse>(
>     IDistributedCache cache,
>     ILogger<IdempotencyBehavior<TRequest, TResponse>> logger)
>     : IPipelineBehavior<TRequest, TResponse>
>     where TRequest : notnull
> {
>     public async Task<TResponse> Handle(
>         TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
>     {
>         if (request is not IIdempotentRequest idempotent)
>             return await next(ct);
>
>         var key = $"idem:{typeof(TRequest).Name}:{idempotent.IdempotencyKey}";
>
>         var cached = await cache.GetStringAsync(key, ct);
>         if (cached is not null)
>         {
>             logger.LogInformation("Повторный вызов {Request} с ключом {Key}, отдаём кеш",
>                 typeof(TRequest).Name, idempotent.IdempotencyKey);
>             return JsonSerializer.Deserialize<TResponse>(cached)!;
>         }
>
>         var response = await next(ct);
>
>         await cache.SetStringAsync(
>             key,
>             JsonSerializer.Serialize(response),
>             new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(24) },
>             ct);
>
>         return response;
>     }
> }
> ```
> Почему так: маркерный интерфейс избавляет от отдельной регистрации behaviour на каждый тип — открытый дженерик применяется ко всем, а внутри одна проверка типа отсекает неподходящие.
>
> Чего в этом решении не хватает для настоящего прода: между `GetStringAsync` и `SetStringAsync` есть окно, в которое два параллельных запроса с одним ключом оба пройдут в обработчик. Настоящее решение — записать ключ в базу с уникальным индексом **в той же транзакции**, что и сама операция, и ловить нарушение уникальности. Кеш здесь оптимизация, а не гарантия. См. [[Идемпотентность]].

### Задача 2. Тест на отсутствующие обработчики

Отсутствие обработчика для команды — ошибка рантайма. Напишите тест, который падает на CI, если хоть для одного типа `IRequest<>` в сборке нет зарегистрированного обработчика.

> [!success]- Решение
> ```csharp
> [Fact]
> public void Каждый_запрос_имеет_зарегистрированный_обработчик()
> {
>     // Поднимаем реальный DI приложения.
>     using var app = new WebApplicationFactory<Program>();
>     using var scope = app.Services.CreateScope();
>
>     var assembly = typeof(CreateOrder).Assembly;
>
>     // Находим все закрытые типы, реализующие IRequest<T>.
>     var requestTypes = assembly.GetTypes()
>         .Where(t => t is { IsAbstract: false, IsInterface: false })
>         .Select(t => new
>         {
>             RequestType = t,
>             Interface = t.GetInterfaces().FirstOrDefault(i =>
>                 i.IsGenericType && i.GetGenericTypeDefinition() == typeof(IRequest<>))
>         })
>         .Where(x => x.Interface is not null)
>         .ToList();
>
>     Assert.NotEmpty(requestTypes); // защита от того, что сканирование само сломалось
>
>     var missing = new List<string>();
>
>     foreach (var item in requestTypes)
>     {
>         var responseType = item.Interface!.GetGenericArguments()[0];
>         var handlerType = typeof(IRequestHandler<,>).MakeGenericType(item.RequestType, responseType);
>
>         if (scope.ServiceProvider.GetService(handlerType) is null)
>             missing.Add(item.RequestType.FullName!);
>     }
>
>     Assert.True(missing.Count == 0,
>         "Нет обработчиков для: " + string.Join(", ", missing));
> }
> ```
> Почему это важно: тест возвращает компилятору роль, которую медиатор у него отобрал. Стоит он двадцать строк, а ловит целый класс ошибок — забытую регистрацию сборки, переименованный обработчик, обработчик, оставшийся `internal` в чужой сборке. `Assert.NotEmpty(requestTypes)` тут не формальность: без неё тест будет зелёным, даже если сканирование ничего не нашло.

### Задача 3. Заменить behaviour на endpoint filter

Есть `ValidationBehavior<TRequest, TResponse>` из MediatR. Переведите ту же валидацию на endpoint filter, чтобы избавиться от медиатора в этом месте.

> [!success]- Решение
> ```csharp
> // Фильтр находит среди аргументов эндпоинта первый аргумент,
> // для которого в DI есть валидатор, и проверяет именно его.
> public sealed class ValidationFilter : IEndpointFilter
> {
>     public async ValueTask<object?> InvokeAsync(
>         EndpointFilterInvocationContext context, EndpointFilterDelegate next)
>     {
>         foreach (var argument in context.Arguments)
>         {
>             if (argument is null) continue;
>
>             var validatorType = typeof(IValidator<>).MakeGenericType(argument.GetType());
>             if (context.HttpContext.RequestServices.GetService(validatorType) is not IValidator validator)
>                 continue;
>
>             var validationContext = new ValidationContext<object>(argument);
>             var result = await validator.ValidateAsync(validationContext, context.HttpContext.RequestAborted);
>
>             if (!result.IsValid)
>                 return TypedResults.ValidationProblem(result.ToDictionary());
>         }
>
>         return await next(context);
>     }
> }
>
> // Применяется к группе — один раз на все эндпоинты внутри.
> var orders = app.MapGroup("/orders").AddEndpointFilter<ValidationFilter>();
> ```
> Что изменилось к лучшему: фильтр возвращает `ValidationProblem` напрямую, без исключения как средства управления потоком, и без глобального обработчика ошибок. Работает без медиатора.
>
> Что стало хуже: валидация теперь привязана к HTTP. Если ту же команду вызвать из `BackgroundService` или из consumer'а очереди, фильтр не применится. Плюс `MakeGenericType` на каждый запрос — здесь стоит закешировать. Это и есть та самая цена «без медиатора», о которой нужно знать заранее.

### Задача 4. Behaviour с таймаутом

Напишите behaviour, который прерывает обработчик, если он выполняется дольше заданного времени, и логирует это. Таймаут должен быть настраиваемым для конкретного типа команды.

> [!success]- Решение
> ```csharp
> // Атрибут на команде задаёт её собственный таймаут.
> [AttributeUsage(AttributeTargets.Class)]
> public sealed class TimeoutAttribute(int milliseconds) : Attribute
> {
>     public int Milliseconds { get; } = milliseconds;
> }
>
> public sealed class TimeoutBehavior<TRequest, TResponse>(
>     ILogger<TimeoutBehavior<TRequest, TResponse>> logger)
>     : IPipelineBehavior<TRequest, TResponse>
>     where TRequest : notnull
> {
>     // Атрибут читается один раз на закрытый дженерик-тип, а не на каждый вызов.
>     private static readonly int? TimeoutMs =
>         typeof(TRequest).GetCustomAttribute<TimeoutAttribute>()?.Milliseconds;
>
>     public async Task<TResponse> Handle(
>         TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
>     {
>         if (TimeoutMs is null)
>             return await next(ct);
>
>         // Связанный источник: отмена приходит либо от клиента, либо по таймауту.
>         using var timeoutCts = new CancellationTokenSource(TimeSpan.FromMilliseconds(TimeoutMs.Value));
>         using var linked = CancellationTokenSource.CreateLinkedTokenSource(ct, timeoutCts.Token);
>
>         try
>         {
>             // Передаём вниз ИМЕННО linked.Token — иначе таймаут не дойдёт до EF Core.
>             return await next(linked.Token);
>         }
>         catch (OperationCanceledException) when (timeoutCts.IsCancellationRequested && !ct.IsCancellationRequested)
>         {
>             // Отличаем «истёк наш таймаут» от «клиент отвалился».
>             logger.LogWarning("{Request} прерван по таймауту {TimeoutMs} мс",
>                 typeof(TRequest).Name, TimeoutMs);
>             throw new TimeoutException($"{typeof(TRequest).Name} превысил {TimeoutMs} мс");
>         }
>     }
> }
> ```
> Два ключевых момента. Первый: `static readonly` поле в дженерик-классе инициализируется один раз **на каждую закрытую комбинацию типов**, поэтому чтение атрибута отражением не стоит ничего на вызовах. Второй: токен обязательно передавать в `next(linked.Token)` — если вызвать `next(ct)`, обработчик про таймаут не узнает, задача продолжит выполняться, и вы получите утечку соединений вместо отмены. Фильтр `when` в `catch` отличает наш таймаут от отмены клиентом — иначе в логах будет шум от каждого закрытого браузера.

---

## Итог

- **Лицензия — первый вопрос в 2026, не архитектура.** MediatR 12.5.0 и ниже — Apache 2.0 навсегда; 13.0.0+ (с июля 2025) — RPL-1.5 или деньги. Community бесплатна при выручке < 5 млн USD, отсутствии > 10 млн внешнего капитала и статусе не-госучреждения. Standard 799 / Professional 1499 / Enterprise 6399 USD в год. Цифры перепроверяйте перед покупкой.
- **Mediator развязывает отправителя и обработчика по типу, но не по времени.** `Send` — обычный синхронный вызов в том же процессе, скоупе и транзакции. Это не очередь, не шина и не гарантия доставки.
- **Реальная ценность — pipeline behaviors.** Порядок behaviors равен порядку регистрации; первый зарегистрированный — внешний. Логирование → валидация → транзакция, именно в таком порядке.
- **Главная цена — компилятор перестаёт проверять связи.** Отсутствующий обработчик, переименованный тип, забытая сборка — всё это ошибки рантайма. Тест-проверка регистраций обязательна.
- **Свой медиатор — это ~80 строк.** Ковариантный `IRequest<out T>`, обёртка через `MakeGenericType` с кешем, агрегирование делегатов. Не хватать будет уведомлений и стриминга.
- **Рекомендация для нового проекта в 2026:** по умолчанию — **без медиатора**, обработчики как обычные классы, кросс-каттинг через endpoint filters и middleware. Если нужна транспортно-независимая точка вставки поведения — **своя тонкая реализация** или **Immediate.Handlers** (AOT, навигация в IDE, ноль рефлексии). **MediatR оправдан только если** команда уже знает его в совершенстве **и** лицензия закрыта. Не берите его «потому что так во всех туториалах».

## Связанное

- [[CQRS]] — разделение команд и запросов, с которым медиатор обычно ходит в паре
- [[Доменные события]] — что публиковать через `INotification`, а что через outbox
- [[Vertical Slice Architecture]] — стиль, где обработчик и его команда живут в одном файле
- [[Фильтры и endpoint filters]] — кросс-каттинг без медиатора
- [[Middleware и конвейер обработки запроса]] — та же техника агрегирования делегатов
- [[FluentValidation]] — валидация, которую вешают в behaviour
- [[Логирование и структурные логи]] — что и как писать из behaviour логирования
- [[EF Core: транзакции и конкурентность]] — механика транзакционного behaviour и ExecutionStrategy
- [[Result pattern вместо исключений]] — альтернатива `ValidationException` в конвейере
- [[MassTransit]] — когда нужен настоящий брокер, а не внутрипроцессный медиатор
- [[Source Generators]] — механизм, на котором построены AOT-friendly альтернативы
- [[Антипаттерны и code smells]] — Service Locator и почему вложенные `Send` это он
- [[Как выбрать архитектуру под задачу]] — методика решения «нужен ли здесь этот слой»
