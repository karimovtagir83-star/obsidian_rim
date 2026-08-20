---
tags: [раздел-09, middleware, pipeline, aspnetcore, конвейер, собес]
aliases: [Middleware, Request pipeline, Конвейер запроса, RequestDelegate, Use Run Map, middleware pipeline]
---

# Middleware и конвейер обработки запроса

> [!abstract] Коротко
> Middleware — это конвейер вложенных друг в друга делегатов (`RequestDelegate`), через которые последовательно проходит каждый `HttpContext`: каждый компонент может что-то сделать до вызова следующего, вызвать `next()` и что-то сделать после, либо вообще не вызывать `next()` и закоротить обработку. Модель — «матрёшка» (onion): запрос идёт внутрь по слоям, ответ выходит обратно наружу в обратном порядке. Порядок регистрации middleware — это порядок выполнения, и он не случаен: `UseExceptionHandler` должен быть снаружи всех, `UseRouting` должен выбрать эндпоинт раньше, чем `UseAuthorization` сможет проверить его метаданные, а `UseAuthentication` обязан отработать раньше `UseAuthorization`, потому что авторизация читает `HttpContext.User`, который заполняет именно аутентификация. Endpoint routing разрывает этот конвейер на две логические фазы — `UseRouting` находит `Endpoint` и кладёт его в контекст, а терминальное middleware (или `UseEndpoints`) его исполняет, — и между этими фазами можно писать middleware, которое уже видит метаданные эндпоинта (например, требуется ли авторизация), но ещё не выполняет сам обработчик.

## Зачем это нужно

Middleware — это не абстрактная тема для собеседования, а источник вполне конкретных продовых инцидентов:

- эндпоинт без `[AllowAnonymous]` внезапно доступен анонимно — потому что кастомный middleware стоит **до** `UseAuthentication` и уже отвечает `200 OK`, а до `UseAuthorization` дело просто не доходит;
- `CORS`-заголовки не появляются на ответе, хотя `AddCors`/`UseCors` вроде бы настроены — потому что `UseCors` стоит после `UseRouting`, но после эндпоинта, который сам закоротил пайплайн;
- в проде раз в сутки «пропадают» логи корреляции — потому что кто-то написал middleware класс, который захватил `IServiceProvider` через конструктор как singleton, а внутри дергает scoped-сервис, который переживает несколько запросов и хранит протухшие данные;
- `System.InvalidOperationException: Headers are read-only, response has already started` — потому что middleware пытается поставить заголовок после того, как ниже по конвейеру начали писать тело ответа (например, застримили `StreamWriter`) ;
- забыли `await next()` в самописном middleware — вся ветка приложения просто перестаёт обрабатывать запросы дальше этой точки, а тесты локально «работают», потому что тестовый мидлварь регистрируется в другом порядке.

Все эти баги — не про C#, а про то, как устроен конвейер и в какой момент какие данные в нём доступны.

## Модель: вложенные делегаты

Каждый middleware — это функция `RequestDelegate`:

```csharp
public delegate Task RequestDelegate(HttpContext context);
```

Регистрация цепочки — это, по сути, замыкание одной функции над другой:

```csharp
// Псевдокод того, что происходит при builder.Build()
RequestDelegate pipeline = ctx => endpoint(ctx); // самый внутренний слой — терминальный
pipeline = ctx => middlewareC(ctx, pipeline);
pipeline = ctx => middlewareB(ctx, pipeline);
pipeline = ctx => middlewareA(ctx, pipeline);
// итог: A вызывает B, B вызывает C, C вызывает эндпоинт
```

Отсюда и матрёшка: `A` выполняет код «до», передаёт управление в `B`, тот — в `C`, `C` вызывает эндпоинт, и после возврата из эндпоинта управление раскручивается обратно: сначала «после»-код `C`, потом `B`, потом `A`.

```
Запрос →  [A: до] → [B: до] → [C: до] → Endpoint → [C: после] → [B: после] → [A: после]  → Ответ
```

> [!info] `HttpContext` — общая мутабельная сущность
> Через весь конвейер проходит **один и тот же объект** `HttpContext`. Middleware не передают друг другу копии — они читают/пишут в общие `Request`, `Response`, `Items`, `Features`. Это и удобно (можно прокинуть correlation id через `HttpContext.Items`), и опасно (один middleware может испортить состояние для всех последующих).

## `Use`, `Run`, `Map`, `MapWhen`, `UseWhen`

| Метод | Получает `next` | Может не вызывать `next` | Ветвление | Типичное применение |
|---|---|---|---|---|
| `app.Use(...)` | да | да (короткое замыкание) | нет | обычный проходной middleware |
| `app.Run(...)` | **нет** | всегда терминальный | нет | последний обработчик в ветке/приложении |
| `app.Map(path, branch)` | — | — | по **точному сегменту пути**, дальше `Path` обрезается | смонтировать под-приложение (например, старое legacy API на `/legacy`) |
| `app.MapWhen(predicate, branch)` | — | — | по **произвольному предикату** над `HttpContext`, путь не режется | ветвление по заголовку, query, версии API |
| `app.UseWhen(predicate, branch)` | да, ветка **вливается обратно** в основной конвейер | да | по предикату, но с возвратом | выполнить middleware только для части запросов, не теряя основной пайплайн |

```csharp
var app = builder.Build();

// Use — обычный проходной middleware
app.Use(async (context, next) =>
{
    context.Response.Headers["X-Powered-By"] = "ASP.NET Core 10";
    await next(context);
});

// Run — терминальный, next вызвать физически нельзя (его нет в сигнатуре)
app.Map("/health-simple", branch =>
{
    branch.Run(async context => await context.Response.WriteAsync("OK"));
});

// Map — ветвление по сегменту пути, Path внутри ветки урезается
app.Map("/legacy", legacyApp =>
{
    legacyApp.UseRouting();
    legacyApp.UseEndpoints(endpoints => endpoints.MapGet("/status", () => "legacy alive"));
});

// MapWhen — ветвление по произвольному условию, путь НЕ режется
app.MapWhen(
    context => context.Request.Headers.ContainsKey("X-Api-Version") &&
               context.Request.Headers["X-Api-Version"] == "1",
    v1App => v1App.Run(async ctx => await ctx.Response.WriteAsync("v1 branch")));

// UseWhen — ветка выполняется, но потом управление ВОЗВРАЩАЕТСЯ в основной конвейер
app.UseWhen(
    context => context.Request.Path.StartsWithSegments("/api/admin"),
    adminBranch => adminBranch.Use(async (ctx, next) =>
    {
        // например, доп. аудит только для /api/admin/*
        Console.WriteLine($"Admin access: {ctx.Request.Path}");
        await next();
    }));
```

> [!warning] `Map`/`MapWhen` создают отдельную ветку конвейера — навсегда
> Если запрос попал в `Map("/legacy", ...)`, он **не возвращается** в основной конвейер после ветки — она сама терминальная (если явно не сконфигурирована иначе). `UseWhen`, в отличие от них, специально спроектирован так, чтобы влиться обратно. Путать `Map` и `UseWhen` — частая ошибка: разработчик хочет «доп. логику для части путей», ставит `Map`, и middleware, зарегистрированные после него в основном конвейере (например, `UseAuthorization`), для этих путей просто не выполняются.

> [!tip] `Map` режет `HttpContext.Request.Path`
> Внутри `app.Map("/legacy", ...)` `context.Request.Path` для `GET /legacy/status` внутри ветки будет равен `/status`, а `/legacy` уедет в `context.Request.PathBase`. Это ловушка при построении абсолютных URL внутри ветки — `LinkGenerator` учитывает `PathBase` автоматически, а ручная конкатенация строк — нет.

## Endpoint routing: два прохода через конвейер

Начиная с ASP.NET Core 3.0, роутинг разбит на два **разделённых** middleware, между которыми можно вставлять свой код:

```csharp
var app = builder.Build();

app.UseRouting();          // фаза 1: сматчить запрос → Endpoint, положить в HttpContext

// ЗДЕСЬ middleware уже видит HttpContext.GetEndpoint(), но эндпоинт ЕЩЁ НЕ выполнен
app.Use(async (context, next) =>
{
    var endpoint = context.GetEndpoint();
    var requiresAuth = endpoint?.Metadata.GetMetadata<IAuthorizeData>() is not null;
    Console.WriteLine($"Matched: {endpoint?.DisplayName}, requires auth: {requiresAuth}");
    await next();
});

app.UseAuthentication();
app.UseAuthorization();    // читает метаданные эндпоинта (policy/roles), решает 401/403

app.MapGet("/api/orders", () => "orders");  // фаза 2 неявно происходит здесь же (терминальный middleware)

app.Run();
```

Что реально происходит:

1. **`UseRouting`** — middleware, которое запускает `EndpointRoutingMiddleware`: сопоставляет `Request.Path` + `Method` с зарегистрированными эндпоинтами и кладёт результат через `HttpContext.SetEndpoint(...)`. Сам эндпоинт **ещё не вызван** — только выбран.
2. Любое middleware между `UseRouting` и точкой исполнения эндпоинта может прочитать `HttpContext.GetEndpoint()` и его `Metadata` — коллекцию произвольных объектов, приклеенных к маршруту через `.RequireAuthorization()`, `.WithMetadata()`, атрибуты и т.д.
3. **`UseEndpoints`** (в minimal-hosting модели .NET 6+ это неявно происходит там, где вызваны `MapGet`/`MapPost`/`MapGroup` — отдельный `app.UseEndpoints(...)` в шаблонах больше не пишут) — это `EndpointMiddleware`, терминальное middleware, которое реально выполняет `Endpoint.RequestDelegate`.

Именно на этом механизме построены `[Authorize]`, `[EnableCors]`, endpoint filters — все они не «магия», а метаданные, которые читает конкретное middleware между двумя фазами роутинга. См. [[Роутинг]] и [[Фильтры и endpoint filters]].

> [!info] Почему нельзя просто проверить `context.Request.Path` в middleware вместо метаданных
> Путь — это строка, а метаданные — это типизированные объекты, приклеенные декларативно к конкретному маршруту (`RequireAuthorization("AdminOnly")`, `RequireCors("Default")`, кастомные атрибуты). Если проверять `Path.StartsWith("/api/admin")` руками, любой рефакторинг маршрутов молча ломает защиту. `UseAuthorization` работает именно через `IAuthorizeData` в `endpoint.Metadata` — поэтому она обязана идти после `UseRouting`, иначе `GetEndpoint()` вернёт `null` и авторизация не найдёт, что проверять (и по умолчанию пропустит запрос).

## Канонический порядок middleware в ASP.NET Core 10

```csharp
var app = builder.Build();

app.UseExceptionHandler();       // 1. ловит всё, что происходит ниже — должен быть снаружи всех
app.UseHsts();                   // 2. только для не-Development, ставит Strict-Transport-Security

app.UseHttpsRedirection();       // 3. редирект http → https, до статики и роутинга

app.UseStaticFiles();            // 4. отдаёт файлы напрямую, минуя роутинг/авторизацию (если не защищено отдельно)

app.UseRouting();                // 5. выбирает Endpoint, кладёт в HttpContext

app.UseCors("Default");          // 6. между Routing и Auth — CORS должен знать эндпоинт (policy может быть per-endpoint)

app.UseAuthentication();         // 7. заполняет HttpContext.User
app.UseAuthorization();          // 8. проверяет метаданные эндпоинта против HttpContext.User

// 9. ваши кастомные middleware (аудит, кастомные заголовки, доп. логирование) —
//    после Authorization, если им не нужно работать с анонимными/незалогиненными запросами

app.MapControllers();            // 10. терминальные — реально выполняют обработчики
app.MapGet("/api/health", () => Results.Ok());

app.Run();
```

Таблица «что будет, если переставить»:

| Пара | Правильный порядок | Что сломается при перестановке |
|---|---|---|
| `UseExceptionHandler` vs всё остальное | должен быть первым | необработанное исключение из `UseRouting`/`UseAuthentication` улетит наружу как «сырое» 500 без ProblemDetails |
| `UseHttpsRedirection` vs `UseStaticFiles` | редирект раньше | статика может утечь по http, если редирект после неё (не критично, но нарушает политику HTTPS-only) |
| `UseRouting` vs `UseCors`/`UseAuthentication`/`UseAuthorization` | роутинг раньше | `GetEndpoint()` вернёт `null`, CORS/Auth не увидят per-endpoint policy и либо всех пустят, либо всех завернут |
| `UseAuthentication` vs `UseAuthorization` | Authentication раньше | `HttpContext.User` останется анонимным `ClaimsPrincipal`, `[Authorize]` всегда будет отдавать 401, даже с валидным токеном |
| `UseCors` vs `UseAuthorization` | CORS раньше | preflight `OPTIONS`-запрос не пройдёт `[Authorize]` (у него нет токена по спеке), получит 401/403 вместо корректного CORS-ответа |
| middleware-логика доступа vs `MapXxx` | до терминального вызова | код после `Map*`/`Run` для этого маршрута никогда не выполнится — терминальное middleware не вызывает `next()` |

> [!warning] Классический вопрос: «Почему `UseAuthentication` должна стоять до `UseAuthorization`?»
> Механика, а не просто правило:
> 1. `UseAuthentication` запускает `AuthenticationMiddleware`, которая берёт зарегистрированную схему (`AddJwtBearer`, `AddCookie`, ...), парсит токен/куку из запроса и, если он валиден, **создаёт `ClaimsPrincipal`** и кладёт его в `HttpContext.User`. Если аутентификации нет вообще, `HttpContext.User` остаётся анонимным principal без claims.
> 2. `UseAuthorization` запускает `AuthorizationMiddleware`, которая берёт `endpoint.Metadata` (найденный `UseRouting`), достаёт из него требования политики (`RequireRole`, `RequireClaim`, кастомный `IAuthorizationRequirement`) и **проверяет их против того самого `HttpContext.User`**, который заполнила аутентификация.
> 3. Если поменять порядок местами, авторизация увидит анонимного пользователя **всегда** — токен ещё не распарсен, `User.Identity.IsAuthenticated` будет `false`, и любой `[Authorize]`-эндпоинт начнёт стабильно отдавать `401`, даже с абсолютно корректным `Bearer`-токеном в заголовке.
> Итог: авторизация — это «разрешено ли *этому* пользователю *это* действие», а без предыдущего шага «*этот* пользователь» ещё не определён.

## Три способа написать своё middleware

### 1. Инлайн-лямбда через `Use`

Самый простой способ, живёт прямо в `Program.cs`. Плюс — минимум церемоний; минус — не тестируется изолированно, нет DI-конструктора, тяжело переиспользовать.

```csharp
app.Use(async (context, next) =>
{
    var sw = System.Diagnostics.Stopwatch.StartNew();
    await next(context);
    sw.Stop();
    context.Response.Headers["X-Elapsed-Ms"] = sw.ElapsedMilliseconds.ToString();
});
```

### 2. Convention-based класс с `InvokeAsync`

Класс без интерфейса — фреймворк находит метод `InvokeAsync`/`Invoke` через reflection при `app.UseMiddleware<T>()`. Конструктор резолвится через DI **один раз**, как singleton, а зависимости, специфичные для запроса, передаются **параметрами метода** `InvokeAsync` — их DI резолвит на каждый вызов из текущего scope.

```csharp
public sealed class RequestTimingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestTimingMiddleware> _logger; // singleton-логгер, это ок

    public RequestTimingMiddleware(RequestDelegate next, ILogger<RequestTimingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    // ITimeProvider — scoped/transient сервис, резолвится DI на каждый запрос через параметр метода
    public async Task InvokeAsync(HttpContext context, ICorrelationIdAccessor correlation)
    {
        var correlationId = correlation.GetOrCreate(context);
        using var scope = _logger.BeginScope(new Dictionary<string, object> { ["CorrelationId"] = correlationId });

        var sw = System.Diagnostics.Stopwatch.StartNew();
        try
        {
            await _next(context);
        }
        finally
        {
            sw.Stop();
            _logger.LogInformation(
                "{Method} {Path} responded {StatusCode} in {ElapsedMs}ms",
                context.Request.Method, context.Request.Path,
                context.Response.StatusCode, sw.ElapsedMilliseconds);
        }
    }
}

// регистрация
app.UseMiddleware<RequestTimingMiddleware>();
```

> [!warning] Не резолвьте scoped-сервисы через конструктор convention-based middleware
> Экземпляр `RequestTimingMiddleware` создаётся **один раз за жизнь приложения** (фреймворк кеширует его как singleton). Если внедрить `DbContext` или другой scoped-сервис **через конструктор**, DI либо кинет исключение при старте («Cannot consume scoped service from singleton»), либо — что хуже — молча создаст его из корневого (root) `IServiceProvider`, и вы получите классический **captive dependency**: один и тот же `DbContext` будет использоваться всеми запросами конкурентно. Правильно — принимать scoped-зависимости **параметром `InvokeAsync`**: DI подставляет их из scope конкретного запроса. Подробнее — [[Captive dependency и типичные ошибки DI]] и [[Жизненные циклы сервисов: Singleton, Scoped, Transient]].

### 3. `IMiddleware` — фабричный, через DI

Middleware как обычный сервис с явным интерфейсом `IMiddleware`. Фреймворк резолвит его на **каждый запрос** через `IMiddlewareFactory`, поэтому его lifetime в контейнере контролируете вы сами — обычно `Scoped`.

```csharp
public sealed class AuditMiddleware : IMiddleware
{
    private readonly IAuditWriter _auditWriter; // можно смело scoped — экземпляр создаётся на запрос

    public AuditMiddleware(IAuditWriter auditWriter) => _auditWriter = auditWriter;

    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        await next(context);

        if (context.Response.StatusCode is >= 200 and < 300)
        {
            await _auditWriter.WriteAsync(context.Request.Path, context.User.Identity?.Name);
        }
    }
}

// регистрация: сервис — в DI, middleware — в пайплайне
builder.Services.AddScoped<AuditMiddleware>();
// ...
app.UseMiddleware<AuditMiddleware>();
```

Сравнение трёх подходов:

| Способ | Где живёт | Lifetime экземпляра | DI зависимостей | Тестируемость | Когда использовать |
|---|---|---|---|---|---|
| Инлайн `Use`-лямбда | `Program.cs` | не применимо (замыкание) | через параметры `HttpContext`/сервисы, разрешённые вручную из `context.RequestServices` | низкая | быстрый прототип, 3–5 строк логики |
| Convention-based класс | отдельный класс | singleton (создан один раз) | конструктор — только singleton-зависимости; scoped/transient — параметрами `InvokeAsync` | средняя | стандартный случай, минимум церемоний |
| `IMiddleware` | отдельный класс + `AddScoped/Transient` | контролируется регистрацией в DI | конструктор — любые зависимости, DI сам резолвит с правильным lifetime | высокая | нужен полноценный scoped-сервис как middleware, юнит-тесты через мок `RequestDelegate` |

> [!tip] `IMiddleware` — самый «безопасный» вариант для junior/middle-команд
> Он structурно не даёт совершить ошибку с captive dependency: конструктор резолвится DI на каждый запрос, как у обычного контроллера. Цена — явная регистрация сервиса и чуть больше кода.

## Короткое замыкание (short-circuiting) и его опасности

Middleware вправе не вызывать `next()` вообще — это и есть короткое замыкание: ответ формируется на этом слое, всё, что зарегистрировано ниже (включая сам эндпоинт), не выполняется.

```csharp
app.Use(async (context, next) =>
{
    if (context.Request.Headers["X-Maintenance-Bypass"] != "secret")
    {
        context.Response.StatusCode = StatusCodes.Status503ServiceUnavailable;
        context.Response.Headers.RetryAfter = "300";
        await context.Response.WriteAsync("Service under maintenance");
        return; // next() НЕ вызван — конвейер обрывается здесь
    }
    await next(context);
});
```

Легитимные случаи: maintenance-режим, rate limiting (см. [[Rate limiting]]), ранняя отбраковка запросов без `Content-Type`, health-check эндпоинты, вынесенные до тяжёлой инициализации.

> [!danger] Забытый `await next()` — самый частый баг в самописном middleware
> ```csharp
> app.Use(async (context, next) =>
> {
>     LogRequest(context);
>     // забыли await next(context); — компилируется, warning легко пропустить
> });
> ```
> Приложение компилируется и стартует без ошибок. Симптом в проде: конкретная ветка (или всё приложение, если middleware стоит в начале) отвечает пустым `200 OK` со статусом по умолчанию для всех запросов, эндпоинты не вызываются, метрики показывают 0 обработанных бизнес-запросов при полном трафике на балансировщике. Анализатор `ASP0016`/линтеры такие места иногда подсвечивают, но полагаться на компилятор нельзя — это ревьюабельная и тестируемая (интеграционным тестом) вещь.

> [!warning] Middleware, зарегистрированный после `Map*`/`Run`, никогда не выполнится
> ```csharp
> app.MapGet("/", () => "hello");
> app.Use(async (ctx, next) => { /* ... */ await next(ctx); }); // мёртвый код
> ```
> Терминальные вызовы `Map*` регистрируют `EndpointMiddleware` в конце общего конвейера в момент вызова `app.Run()`/первого обращения к пайплайну, но порядок в коде **всё равно имеет значение** — middleware, объявленный текстуально после того, как в конвейер попал терминальный `Run`/ветка `Map`, для этой ветки не участвует. Практическое правило: **весь инфраструктурный middleware (`Use...`) объявляйте раньше вызовов `MapGet/MapPost/MapControllers`.**

## Исключения: `UseExceptionHandler`, `IExceptionHandler`, `UseStatusCodePages`

Три разных механизма, которые часто путают:

| Механизм | Реагирует на | Типичный сценарий |
|---|---|---|
| `UseExceptionHandler` + `IExceptionHandler` | необработанные **исключения** | превратить `Exception` в ProblemDetails с 500 (или другим кодом по типу исключения) |
| `UseStatusCodePages` | ответ с пустым телом и кодом **4xx/5xx**, выставленным явно (`return Results.NotFound()`), без исключения | добавить тело/ProblemDetails к «голому» статус-коду |
| `try/catch` внутри конкретного endpoint filter или обработчика | локальная, точечная логика | превратить конкретное доменное исключение в конкретный ответ прямо в месте вызова |

`IExceptionHandler` (появился в .NET 8, стал основным способом в .NET 10) — DI-сервис, который `UseExceptionHandler` вызывает по цепочке, пока один из обработчиков не вернёт `true`:

```csharp
public sealed class ValidationExceptionHandler : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext, Exception exception, CancellationToken cancellationToken)
    {
        if (exception is not ValidationException validationEx)
            return false; // не наш случай — передать дальше по цепочке обработчиков

        httpContext.Response.StatusCode = StatusCodes.Status422UnprocessableEntity;
        await httpContext.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Title = "Validation failed",
            Status = StatusCodes.Status422UnprocessableEntity,
            Detail = validationEx.Message
        }, cancellationToken);

        return true; // обработано, дальше не идём
    }
}

// Program.cs
builder.Services.AddExceptionHandler<ValidationExceptionHandler>();
builder.Services.AddProblemDetails(); // общий формат для необработанных случаев

var app = builder.Build();
app.UseExceptionHandler(); // без лямбды — использует зарегистрированные IExceptionHandler + AddProblemDetails как fallback
```

```csharp
// UseStatusCodePages — добавляет тело к статус-кодам без исключений
app.UseStatusCodePages(async context =>
{
    context.HttpContext.Response.ContentType = "application/problem+json";
    await context.HttpContext.Response.WriteAsJsonAsync(new ProblemDetails
    {
        Status = context.HttpContext.Response.StatusCode,
        Title = ReasonPhrases.GetReasonPhrase(context.HttpContext.Response.StatusCode)
    });
});
```

Подробный разбор форматов и RFC 9457 — в [[Обработка ошибок и ProblemDetails]].

> [!info] Почему `UseExceptionHandler` должен быть первым middleware
> Он оборачивает **весь** нижележащий конвейер в `try/catch`. Если поставить его после `UseRouting` или `UseAuthentication`, исключения из них самих (например, из кастомной схемы аутентификации) не будут перехвачены и уйдут в дефолтный обработчик хоста — голая страница разработчика или пустой 500 без ProblemDetails.

## Производительность: порядок и стоимость

- Каждый middleware — это как минимум один виртуальный вызов и, как правило, аллокация делегата/замыкания при первом построении конвейера (после `Build()` конвейер собирается один раз и переиспользуется, дополнительных аллокаций на каждый запрос в стандартном `Use` нет — но лямбды, захватывающие переменные, могут аллоцировать на каждый вызов, если замыкание не статично).
- **Дешёвые проверки — раньше дорогих.** Проверка авторизации (обычно — сравнение claims в памяти) должна стоять раньше, чем middleware, которое читает тело запроса, обращается к базе или внешнему сервису. Иначе неаутентифицированный клиент бесплатно грузит бэкенд дорогой работой до того, как получит `401`.
- **`UseWhen`/`MapWhen` для дорогих middleware.** Если middleware нужно не всем маршрутам (например, детальный аудит только для `/api/admin/*` или ограничение размера тела только для upload-эндпоинтов), не вешайте его глобально — заверните в `UseWhen`, чтобы остальные запросы вообще не проходили через его код.
- **`UseStaticFiles` до `UseRouting`.** Отдача статики не должна проходить через роутинг, авторизацию и вообще матчинг эндпоинтов — это чистый файловый I/O, и лишние слои конвейера на каждый `.js`/`.css` — заметные накладные расходы при большом трафике.
- **Response compression и buffering** (`UseResponseCompression`, `UseResponseBuffering`) — тяжёлые по CPU/памяти middleware, их стоит подключать выборочно, а не глобально, если часть эндпоинтов уже отдаёт сжатый контент (например, файлы).

```csharp
// Пример: дорогое middleware выполняется только там, где реально нужно
app.UseWhen(
    ctx => ctx.Request.Path.StartsWithSegments("/api/reports"),
    branch => branch.UseMiddleware<ExpensiveReportAuditMiddleware>());
```

## Полный пример: middleware логирования длительности запроса и correlation id

```csharp
using System.Diagnostics;

public sealed class RequestLoggingMiddleware
{
    private const string CorrelationHeaderName = "X-Correlation-Id";

    private readonly RequestDelegate _next;
    private readonly ILogger<RequestLoggingMiddleware> _logger;

    public RequestLoggingMiddleware(RequestDelegate next, ILogger<RequestLoggingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // 1. Взять correlation id из входящего заголовка или сгенерировать новый
        var correlationId = context.Request.Headers.TryGetValue(CorrelationHeaderName, out var incoming)
            ? incoming.ToString()
            : Guid.NewGuid().ToString("N");

        context.Items["CorrelationId"] = correlationId;

        // 2. Поставить заголовок ответа заранее через OnStarting —
        //    после того, как тело начали писать, менять заголовки уже нельзя (см. ниже)
        context.Response.OnStarting(() =>
        {
            context.Response.Headers[CorrelationHeaderName] = correlationId;
            return Task.CompletedTask;
        });

        using var _ = _logger.BeginScope(new Dictionary<string, object>
        {
            ["CorrelationId"] = correlationId
        });

        var stopwatch = Stopwatch.StartNew();
        try
        {
            await _next(context);
        }
        catch
        {
            // логируем и здесь тоже — UseExceptionHandler отработает выше по конвейеру
            _logger.LogError(
                "Request {Method} {Path} failed after {ElapsedMs}ms",
                context.Request.Method, context.Request.Path, stopwatch.ElapsedMilliseconds);
            throw;
        }
        finally
        {
            stopwatch.Stop();
            _logger.LogInformation(
                "{Method} {Path} → {StatusCode} in {ElapsedMs}ms [{CorrelationId}]",
                context.Request.Method,
                context.Request.Path,
                context.Response.StatusCode,
                stopwatch.ElapsedMilliseconds,
                correlationId);
        }
    }
}

public static class RequestLoggingMiddlewareExtensions
{
    public static IApplicationBuilder UseRequestLogging(this IApplicationBuilder app) =>
        app.UseMiddleware<RequestLoggingMiddleware>();
}
```

```csharp
// Program.cs
var app = builder.Build();

app.UseExceptionHandler();
app.UseRequestLogging();   // сразу после обработчика исключений — ловим тайминг ВСЕГО, включая ошибки
app.UseHttpsRedirection();
app.UseRouting();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

Такой подход к extension-методу (`UseRequestLogging`) — стандартная практика: вся регистрация middleware в `Program.cs` читается как список `Use...` с говорящими именами, а не как набор безымянных лямбд.

## Частые ошибки и ловушки

> [!warning] Изменение ответа после `HttpResponse.HasStarted == true`
> Как только в тело ответа записан хотя бы один байт (или явно вызван flush), заголовки становятся неизменяемыми. Попытка поставить заголовок или сменить `StatusCode` после этого момента бросает `InvalidOperationException: Headers are read-only, response has already started`. Middleware, которое хочет модифицировать заголовки **после** выполнения нижележащего кода (как в примере с `Cache-Control` выше или с correlation id), должно подписываться на `context.Response.OnStarting(...)` — колбэк вызывается **непосредственно перед** отправкой заголовков, а не пытаться менять их постфактум после `await next()`.
>
> ```csharp
> // Неправильно — если next() уже начал писать тело, здесь будет исключение
> await next(context);
> context.Response.Headers["X-Custom"] = "value"; // может упасть
>
> // Правильно
> context.Response.OnStarting(() =>
> {
>     context.Response.Headers["X-Custom"] = "value";
>     return Task.CompletedTask;
> });
> await next(context);
> ```

> [!warning] Захват `IServiceProvider`/scoped-сервисов в singleton-контексте
> Convention-based middleware создаётся **один раз** как singleton всего приложения. Если такое middleware в конструкторе сохранит ссылку на scoped-сервис (например, `DbContext`, полученный вручную через `serviceProvider.GetService<T>()` в конструкторе, а не через параметр `InvokeAsync`), эта ссылка «переживёт» запрос, для которого была создана, и будет молча переиспользована для всех последующих запросов — классический **captive dependency**. Итог — гонки данных, случайные `ObjectDisposedException`, утечки контекста между пользователями. Правило: **scoped/transient зависимости в convention-based middleware принимаются только параметрами `InvokeAsync`**, никогда — через конструктор.

> [!warning] Порядок `UseCors` относительно `UseRouting`/`UseAuthorization`
> `UseCors` должен стоять **после** `UseRouting` (иначе не увидит per-endpoint `[EnableCors]`/`RequireCors`) и **до** `UseAuthorization`/терминальных эндпоинтов — preflight `OPTIONS`-запрос браузера не несёт `Authorization` заголовка и не должен попадать под требования `[Authorize]`. Подробности CORS-политик — в [[CORS]].

> [!warning] `UseStaticFiles` без ограничения — раздача чувствительных файлов
> Если `UseStaticFiles()` стоит раньше `UseRouting`/`UseAuthorization` (а по канону так и должно быть), любой файл из `wwwroot` отдаётся **без авторизации**. Это ожидаемо для CSS/JS/изображений, но опасно, если в `wwwroot` случайно попали служебные файлы (`appsettings.json`, дампы, бэкапы). Для защищённой раздачи файлов нужен отдельный эндпоинт с `[Authorize]`, а не статический middleware.

> [!question]- Почему нельзя просто засунуть всю логику авторизации в один самописный `Use`-middleware в начале конвейера вместо `UseAuthentication`/`UseAuthorization`?
> Формально можно, но вы теряете всю инфраструктуру ASP.NET Core: декларативные политики (`[Authorize(Policy = "AdultOnly")]`), кэшируемые `IAuthorizationRequirement`, поддержку нескольких схем аутентификации одновременно (`Bearer` + `Cookie`), интеграцию с endpoint filters и `[AllowAnonymous]`. Кроме того, самописный middleware в начале конвейера **не видит `HttpContext.GetEndpoint()`**, потому что стоит до `UseRouting`, — то есть не может узнать, какая именно политика нужна для конкретного маршрута, и вынужден либо парсить `Path` руками (хрупко), либо требовать авторизацию для всех запросов без исключений (ломает публичные эндпоинты и preflight CORS).

## Вопросы для собеседования

> [!question]- Опишите модель middleware как «матрёшку». Что произойдёт, если middleware не вызовет `next()`?
> Каждый middleware — обёртка вокруг следующего: код до `await next()` выполняется на пути «внутрь», код после — на пути «наружу», после того как все внутренние слои и терминальный эндпоинт отработали. Если `next()` не вызван, конвейер обрывается на этом месте (короткое замыкание) — все middleware и эндпоинт, зарегистрированные ниже, не выполняются вообще, а ответ формируется тем, что записано в текущем middleware. Это либо намеренная техника (maintenance-режим, rate limiting), либо баг (забытый `await next()`), который проявляется как «приложение отвечает пустым 200 на всё».

> [!question]- В чём разница между `Map`, `MapWhen` и `UseWhen`?
> `Map` ветвит конвейер по точному сегменту пути и обрезает `PathBase`/`Path` внутри ветки; ветка терминальна и не возвращается в основной конвейер. `MapWhen` ветвит по произвольному предикату над `HttpContext` (не только путь), путь не режется, ветка тоже терминальна. `UseWhen` — единственный из трёх, чья ветка **вливается обратно** в основной конвейер после выполнения: подходит, когда нужно выполнить дополнительную логику для подмножества запросов, не теряя остальной пайплайн (например, `UseAuthorization`, зарегистрированный позже).

> [!question]- Почему `UseAuthentication` обязана стоять до `UseAuthorization`, а не наоборот?
> `UseAuthentication` парсит учётные данные запроса (токен, куку) и заполняет `HttpContext.User` конкретным `ClaimsPrincipal`. `UseAuthorization` не аутентифицирует — она читает уже заполненный `HttpContext.User` и сверяет его claims/роли с требованиями политики, взятой из метаданных эндпоинта. Если поменять порядок, авторизация всегда видит анонимного пользователя (аутентификация ещё не отработала), и любой защищённый эндпоинт стабильно отдаёт 401 независимо от валидности токена.

> [!question]- Что делает `UseRouting` физически и почему middleware, стоящее до него, не может прочитать `HttpContext.GetEndpoint()`?
> `UseRouting` запускает сопоставление входящего запроса (путь + метод + constraints) с зарегистрированными маршрутами и, найдя совпадение, кладёт объект `Endpoint` (включая всю коллекцию метаданных: политики авторизации, CORS-политику, атрибуты) в `HttpContext` через `SetEndpoint`. До выполнения этого middleware `HttpContext.GetEndpoint()` возвращает `null` по построению — сопоставление ещё не произошло, эту информацию физически неоткуда взять.

> [!question]- Чем `IExceptionHandler` отличается от `UseStatusCodePages`?
> `IExceptionHandler` (совместно с `UseExceptionHandler`) реагирует на **необработанные исключения** — код, который упал с `throw`, перехватывается и превращается в контролируемый ответ (обычно ProblemDetails с 500 или другим статусом в зависимости от типа исключения). `UseStatusCodePages` реагирует на ответы, у которых уже **явно выставлен** код 4xx/5xx (например, `return Results.NotFound()`), но тело осталось пустым — он добавляет к такому ответу контент. Это разные источники: исключение — незапланированный сбой; статус-код без тела — запланированный, но «немой» ответ.

> [!question]- Convention-based middleware внедрил `AppDbContext` через конструктор и упал в бою с странными данными между разными пользователями. В чём причина и как это должно было выглядеть?
> Convention-based middleware создаётся один раз как singleton всего приложения; всё, что попадает в его конструктор, живёт всё время работы приложения. `AppDbContext` — scoped-сервис, рассчитанный на один HTTP-запрос; будучи захваченным в singleton-контексте, он становится общим для всех запросов одновременно — классический captive dependency, дающий гонки данных и утечку состояния между пользователями. Правильно — принимать `AppDbContext` не через конструктор, а как параметр метода `InvokeAsync(HttpContext context, AppDbContext db)`: тогда DI резолвит его из scope конкретного запроса на каждый вызов.

> [!question]- Middleware пытается добавить заголовок `X-Response-Time` после `await next(context)`, но иногда получает `InvalidOperationException`. Почему не всегда, и как исправить правильно?
> Исключение возникает только тогда, когда нижележащий код (эндпоинт или другое middleware) уже начал запись тела ответа — `HttpResponse.HasStarted` стал `true`, и заголовки заморожены сервером (Kestrel) перед отправкой первых байт. Для маленьких буферизуемых ответов запись тела может произойти уже после того, как весь `_next(context)` вернул управление и не успел стриминговать данные, поэтому баг «иногда» не проявляется. Правильное решение — не полагаться на момент возврата из `next()`, а подписаться на `context.Response.OnStarting(callback)`: колбэк гарантированно выполняется непосредственно перед отправкой заголовков, независимо от того, буферизован ответ или стримится.

## Итог

- Middleware — цепочка вложенных `RequestDelegate`; порядок регистрации = порядок выполнения, ответ раскручивается в обратном порядке.
- `Use` — обычный проходной, `Run` — терминальный, `Map`/`MapWhen` — необратимое ветвление, `UseWhen` — ветвление с возвратом в основной конвейер.
- Endpoint routing — двухфазный: `UseRouting` выбирает `Endpoint` и кладёт его в `HttpContext`, терминальное middleware его исполняет; между фазами можно читать метаданные эндпоинта.
- Каноничный порядок: `UseExceptionHandler`/`UseHsts` → `UseHttpsRedirection` → `UseStaticFiles` → `UseRouting` → `UseCors` → `UseAuthentication` → `UseAuthorization` → кастомные → терминальные `Map*`. `UseAuthentication` строго до `UseAuthorization`, потому что вторая читает `HttpContext.User`, заполненный первой.
- Три способа писать своё middleware: инлайн-лямбда (быстро, не тестируется), convention-based класс (singleton, scoped-зависимости — только параметрами `InvokeAsync`), `IMiddleware` (полноценный DI-сервис с контролируемым lifetime).
- Забытый `await next()` — тихий баг: приложение отвечает 200 на всё, но код ниже не выполняется.
- После `HttpResponse.HasStarted` заголовки менять нельзя — используйте `Response.OnStarting(...)`.

## Связанное

- [[Введение и устройство ASP.NET Core]]
- [[Роутинг]]
- [[Фильтры и endpoint filters]]
- [[Dependency Injection: контейнер ASP.NET Core]]
- [[Жизненные циклы сервисов: Singleton, Scoped, Transient]]
- [[Captive dependency и типичные ошибки DI]]
- [[Обработка ошибок и ProblemDetails]]
- [[CORS]]
- [[Rate limiting]]
- [[Аутентификация: обзор схем]]
- [[Логирование и структурные логи]]
- [[Health checks]]
- [[Кеширование: Output, Response, HybridCache]]
- [[HTTP: методы, коды, заголовки, кеширование]]
