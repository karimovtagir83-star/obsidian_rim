---
tags: [раздел-09, minimal-api, endpoints, typedresults, routing, middle, dotnet10, собес]
aliases: [Minimal API, MapGet, MapPost, MapPut, MapDelete, MapPatch, TypedResults, MapGroup, IEndpointRouteBuilder, Minimal APIs]
---

# Minimal API

> [!abstract] Коротко
> Minimal API — стиль описания HTTP-эндпоинтов прямо на `WebApplication`/`IEndpointRouteBuilder` через `MapGet`/`MapPost`/`MapPut`/`MapDelete`/`MapPatch`, без классов-контроллеров и без атрибутной магии `[ApiController]`. Обработчик — обычный делегат (метод или лямбда): его параметры биндятся из маршрута, query, заголовков, тела, DI и `HttpContext` по явным и предсказуемым правилам. С .NET 10 это **основной, рекомендуемый по умолчанию способ** строить Web API: меньше boilerplate, выше производительность (нет фильтр-пайплайна MVC, если он не нужен), эндпоинты тестируются как обычные методы. `TypedResults` вместо `Results` даёт компилируемую типизацию и автоматическую OpenAPI-метадату. `MapGroup` и `IEndpointFilter` заменяют часть роли, которую в MVC играли `[Route]`-префиксы и фильтры контроллеров.

## Зачем это нужно

До .NET 6 единственным способом писать Web API в ASP.NET Core были MVC-контроллеры: класс, наследующий `ControllerBase`, атрибуты `[HttpGet]`/`[FromBody]`/`[ApiController]`, конвенции именования экшенов, отдельный слой `IActionResult`. Для CRUD над одной сущностью это означало: файл контроллера, DI зависимостей через конструктор, атрибуты на каждом методе, отдельный слой моделей — и всё это ради пяти строк логики.

Проблемы, которые реально возникали в проде и на ревью:

- **Boilerplate против сигнала.** В контроллере на 200 строк реальной логики — 30 строк. Остальное — атрибуты, `[FromServices]` в конструкторе для сервисов, которые нужны только одному методу, `ActionResult<T>` обёртки.
- **Скрытая магия `[ApiController]`.** Автоматическая валидация модели до вызова экшена, автоматический `400` при невалидной модели, автоматическая инференция источника привязки — удобно, пока не нужно это отключить или понять, почему валидация сработала до вашего кода. На собеседовании про это спрашивают именно потому, что новички не знают, что это переключаемое поведение, а не часть ASP.NET Core.
- **Тестирование.** Модульный тест экшена контроллера часто тянет за собой мокирование `ControllerContext`, `HttpContext`, `TempData` — сущностей, которые логике не нужны, но нужны инфраструктуре MVC.
- **Производительность на простых сценариях.** Каждый запрос к контроллеру проходит через MVC action invocation pipeline: model binding через провайдеры, action filters, result filters, resource filters — даже если ни один из них не используется. Minimal API по умолчанию — это debug пайплайн: маршрутизация → биндинг параметров → делегат → результат, без лишних слоёв.
- **Явность.** В Minimal API сигнатура метода — это контракт: `int id, OrderDto dto, IOrderService svc, CancellationToken ct` читается без атрибутов, откуда что берётся (в большинстве случаев — см. ниже про правила инференции). Явное лучше неявного — тот же принцип, что двигал команду ASP.NET Core при проектировании фичи.

Minimal API не отменяет MVC — контроллеры остаются нормальным выбором для legacy-кода, Razor Views, и сценариев с сложными конвенциями (см. сравнение ниже и [[MVC и контроллеры]]). Но для нового Web API в .NET 10 старт — это `MapGet`/`MapPost`, а не `ControllerBase`.

## Базовый синтаксис

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddScoped<IOrderService, OrderService>();

var app = builder.Build();

app.MapGet("/orders/{id:int}", (int id, IOrderService svc) =>
    svc.GetById(id) is { } order ? Results.Ok(order) : Results.NotFound());

app.MapPost("/orders", (CreateOrderDto dto, IOrderService svc) =>
{
    var order = svc.Create(dto);
    return Results.Created($"/orders/{order.Id}", order);
});

app.MapPut("/orders/{id:int}", (int id, UpdateOrderDto dto, IOrderService svc) =>
    svc.Update(id, dto) ? Results.NoContent() : Results.NotFound());

app.MapDelete("/orders/{id:int}", (int id, IOrderService svc) =>
    svc.Delete(id) ? Results.NoContent() : Results.NotFound());

app.MapPatch("/orders/{id:int}/status", (int id, string status, IOrderService svc) =>
    svc.ChangeStatus(id, status) ? Results.NoContent() : Results.NotFound());

app.Run();
```

Ничего, кроме `WebApplication` (реализующего `IEndpointRouteBuilder`) и делегата, здесь не нужно. Роутинг разбирается в [[Роутинг]], здесь — фокус на том, что происходит с параметрами и результатом.

> [!info] `Map*` возвращает `IEndpointConventionBuilder` (точнее `RouteHandlerBuilder`)
> Это позволяет дальше в fluent-стиле навешивать метаданные и поведение: `.WithName(...)`, `.RequireAuthorization()`, `.AddEndpointFilter(...)`, `.CacheOutput(...)`, `.Produces<T>()`. Именно эта цепочка — основной способ настройки эндпоинта в Minimal API, аналог атрибутов в MVC.

## Источники привязки параметров

Minimal API биндит параметры делегата по набору правил, применяемых **в порядке приоритета**, без атрибутов в простых случаях:

| Источник | Как определяется | Пример |
|---|---|---|
| Маршрут (route values) | имя параметра совпадает с `{имя}` в шаблоне | `{id:int}` → `int id` |
| Query string | простой тип, имени нет в маршруте | `?page=2` → `int page` |
| Заголовок | явный атрибут `[FromHeader]` | `[FromHeader(Name = "X-Api-Version")] string version` |
| Тело запроса (JSON) | сложный тип (не примитив, не `IFormFile`, не сервис), **только один параметр за запрос** | `OrderDto dto` |
| DI-сервис | тип зарегистрирован в контейнере (инференция с .NET 7+), либо явный `[FromServices]` | `IOrderService svc` |
| `HttpContext`/`HttpRequest`/`HttpResponse` | специальный тип — биндится напрямую, не через DI | `HttpContext ctx` |
| `ClaimsPrincipal` | специальный тип | `ClaimsPrincipal user` |
| `CancellationToken` | специальный тип — токен отмены запроса (`RequestAborted`) | `CancellationToken ct` |
| `IFormFile`/`IFormFileCollection`/`IFormCollection` | форма/multipart | см. [[Загрузка файлов и работа с формами]] |
| Явные атрибуты | переопределяют инференцию | `[FromQuery]`, `[FromRoute]`, `[FromBody]`, `[FromServices]`, `[FromKeyedServices]`, `[AsParameters]` |

```csharp
app.MapGet("/search", (
    string q,                          // query: ?q=...
    int page,                          // query: ?page=...
    [FromHeader(Name = "Accept-Language")] string? lang,
    ISearchService search,             // DI-сервис — инференция по регистрации
    CancellationToken ct) =>
{
    return search.SearchAsync(q, page, lang, ct);
});
```

> [!warning] Только один параметр может биндиться из тела
> Если делегат имеет два комплексных типа без атрибутов, фреймворк на старте (или при первом запросе, в зависимости от сценария) выбросит исключение о неоднозначности источника. Тело запроса — это единственный `Stream`, прочитать его дважды в два разных объекта нельзя.
> ```csharp
> // Ошибка в рантайме: ambiguity — оба выглядят как "тело"
> app.MapPost("/bad", (OrderDto order, CustomerDto customer) => { ... });
> ```
> Решение — обернуть в один DTO, либо использовать `[AsParameters]` для группировки простых (не JSON-body) параметров в один класс/struct без создания второго "тела".

### `[AsParameters]` — группировка параметров

```csharp
public readonly record struct OrderQuery(
    [FromQuery] string? Status,
    [FromQuery] int Page,
    [FromQuery] int PageSize,
    [FromHeader(Name = "X-Tenant-Id")] string TenantId);

app.MapGet("/orders", ([AsParameters] OrderQuery query, IOrderService svc) =>
    svc.Search(query));
```

`[AsParameters]` — это не биндинг из тела, это "распаковка" набора простых параметров в один тип, чтобы не тащить пять аргументов в сигнатуру. Работает только с record/class/struct с публичными свойствами или конструктором — сам по себе комплексный тип, но не JSON body.

### `HttpContext`, `CancellationToken` и явное `[FromServices]`

```csharp
app.MapGet("/whoami", (HttpContext ctx) => ctx.User.Identity?.Name ?? "anonymous");

app.MapGet("/slow-report", async (IReportService svc, CancellationToken ct) =>
{
    // ct == HttpContext.RequestAborted — отменяется, если клиент разорвал соединение
    var report = await svc.BuildAsync(ct);
    return Results.Ok(report);
});

app.MapGet("/keyed", ([FromKeyedServices("primary")] ICacheProvider cache) =>
    Results.Ok(cache.Get("key")));
```

> [!tip] Инференция DI-сервисов — не всегда очевидна
> Начиная с .NET 7, если тип параметра зарегистрирован в `IServiceCollection`, фреймворк сам поймёт, что это сервис, и не будет пытаться биндить его из query/body. Но если у вас DTO **случайно совпадает по имени/типу** с зарегистрированным сервисом (редко, но бывает при generic-обёртках), поведение может удивить. Явный `[FromServices]` снимает любую неоднозначность и документирует намерение — многие команды требуют его на код-ревью именно из соображений читаемости, а не потому что он строго обязателен.

### Кастомный биндинг — `BindAsync`

Когда встроенных источников не хватает (например, нужно распарсить составной идентификатор из query или декодировать кастомный формат), тип параметра может сам объявить, как биндиться:

```csharp
public record struct SortSpec(string Field, bool Descending)
{
    public static ValueTask<SortSpec?> BindAsync(HttpContext context, ParameterInfo parameter)
    {
        var raw = context.Request.Query["sort"].ToString(); // "price:desc"
        if (string.IsNullOrEmpty(raw))
            return ValueTask.FromResult<SortSpec?>(null);

        var parts = raw.Split(':');
        var spec = new SortSpec(parts[0], parts.Length > 1 && parts[1] == "desc");
        return ValueTask.FromResult<SortSpec?>(spec);
    }
}

app.MapGet("/products", (SortSpec? sort, IProductService svc) =>
    svc.List(sort));
```

Компилятор ищет статический метод `BindAsync(HttpContext, ParameterInfo)` по конвенции (duck typing, аналогично `TryParse`) — реализовывать интерфейс необязательно, но есть и формальный контракт `IBindableFromHttpContext<T>` для тех, кто предпочитает явный интерфейс:

```csharp
public class SortSpec : IBindableFromHttpContext<SortSpec>
{
    public static ValueTask<SortSpec?> BindAsync(HttpContext context, ParameterInfo parameter)
        => /* та же логика */;
}
```

Оба варианта равнозначны для рантайма; интерфейс просто фиксирует контракт на уровне типов.

## `Results` vs `TypedResults`

`Results.Ok(...)`, `Results.NotFound()` и т.д. возвращают `IResult` — единый интерфейс без информации о типе на этапе компиляции. `TypedResults.Ok<T>(...)`, `TypedResults.NotFound()` возвращают **конкретные типы** (`Ok<T>`, `NotFound`, `Created<T>`...), которые тоже реализуют `IResult`, но дают компилятору и анализатору OpenAPI знать заранее, что именно может вернуть эндпоинт.

| | `Results` | `TypedResults` |
|---|---|---|
| Тип возврата | `IResult` (стёртый) | конкретный (`Ok<Order>`, `NotFound`, ...) |
| OpenAPI-метадата | нужно вручную `.Produces<T>(200)` | **выводится автоматически** из сигнатуры метода |
| Юнит-тестирование | нужно кастовать/проверять через рефлексию | `Assert.IsType<Ok<Order>>(result)` — прямая проверка |
| Компилятор ловит опечатки в статус-кодах/типах | нет | да — несоответствие видно на этапе компиляции |
| Поддержка union-возврата | нет прямой | `Results<T1, T2, ...>` — до 6 вариантов |

```csharp
// Results — работает, но OpenAPI не узнает про 404 без .Produces
app.MapGet("/orders/{id:int}", (int id, IOrderService svc) =>
{
    var order = svc.GetById(id);
    return order is null ? Results.NotFound() : Results.Ok(order);
});

// TypedResults + union-тип — современный идиоматичный вариант
app.MapGet("/orders/{id:int}", Results<Ok<Order>, NotFound> (int id, IOrderService svc) =>
{
    var order = svc.GetById(id);
    return order is null
        ? TypedResults.NotFound()
        : TypedResults.Ok(order);
});
```

### Union-возврат для методов с несколькими исходами

`Results<TResult1, TResult2, ...>` — специальный тип, реализующий неявное преобразование из каждого варианта. OpenAPI видит все ветки и генерирует и `200`, и `404`, и `400` в одном месте:

```csharp
app.MapPost("/orders", async Task<Results<Created<Order>, ValidationProblem, Conflict<string>>>
    (CreateOrderDto dto, IOrderService svc, IValidator<CreateOrderDto> validator) =>
{
    var validation = await validator.ValidateAsync(dto);
    if (!validation.IsValid)
        return TypedResults.ValidationProblem(validation.ToDictionary());

    if (await svc.ExistsAsync(dto.ExternalId))
        return TypedResults.Conflict($"Order {dto.ExternalId} already exists");

    var order = await svc.CreateAsync(dto);
    return TypedResults.Created($"/orders/{order.Id}", order);
});
```

> [!info] Почему именно это — "правильный" способ в 2026
> Сигнатура `Results<Created<Order>, ValidationProblem, Conflict<string>>` — это машиночитаемый контракт эндпоинта: любой, кто откроет метод, видит все возможные ответы без чтения тела. OpenAPI-документ строится по этой сигнатуре без единого атрибута `.Produces`. Юнит-тест может проверить `Assert.IsType<Created<Order>>(result.Result)` без обращения к `HttpContext`. Всё это невозможно с `IResult` — там компилятор и генератор документации видят только "что-то, реализующее IResult".

> [!warning] Максимум 6 типов в `Results<...>`
> Если исходов больше — используйте `IResult` и `.Produces<T>()` вручную, либо декомпозируйте эндпоинт (обычно это сигнал, что в одном методе слишком много веток бизнес-логики).

## `MapGroup` — группировка эндпоинтов

`MapGroup` создаёт `RouteGroupBuilder`, который сам реализует `IEndpointRouteBuilder` — на нём можно вызывать `MapGet`/`MapPost` с относительными путями и накатывать общую метадату/фильтры/авторизацию на всю группу разом.

```csharp
var orders = app.MapGroup("/orders")
    .WithTags("Orders")
    .RequireAuthorization()
    .AddEndpointFilter<ValidationFilter<object>>();

orders.MapGet("/{id:int}", GetOrder);
orders.MapPost("/", CreateOrder);
orders.MapPut("/{id:int}", UpdateOrder);
orders.MapDelete("/{id:int}", DeleteOrder);

// Вложенная группа — версионирование или под-ресурс
var orderItems = orders.MapGroup("/{orderId:int}/items")
    .WithTags("Order Items");

orderItems.MapGet("/", GetOrderItems);
orderItems.MapPost("/", AddOrderItem);
```

Итоговые маршруты: `/orders/{id}`, `/orders/`, `/orders/{orderId}/items/` и т.д. — префикс родительской группы всегда подставляется.

> [!tip] `MapGroup` — это не "controller", но решает ту же боль
> До `MapGroup` (введён в .NET 7) единственным способом не повторять `.RequireAuthorization()` на каждом эндпоинте было городить свои extension-методы. Теперь общая авторизация, теги для Swagger, версия API через префикс `/v{version:apiVersion}`, общий rate-limiting policy — всё вешается один раз на группу и наследуется всеми вложенными маршрутами и подгруппами.

## Endpoint Filters — введение

`IEndpointFilter` — механизм "middleware для одного эндпоинта", работающий уже после того, как маршрут выбран и параметры извлечены, но до вызова самого делегата (и после — на пути результата). Полная механика, сравнение с MVC-фильтрами и `IMiddleware` — в [[Фильтры и endpoint filters]]; здесь — минимальный пример, чтобы показать, где он встраивается в Minimal API.

```csharp
public class ValidationFilter<T> : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context, EndpointFilterDelegate next)
    {
        var arg = context.Arguments.OfType<T>().FirstOrDefault();
        if (arg is IValidatableDto dto && !dto.IsValid(out var errors))
            return TypedResults.ValidationProblem(errors);

        return await next(context);
    }
}

app.MapPost("/orders", CreateOrder)
   .AddEndpointFilter<ValidationFilter<CreateOrderDto>>();

// Инлайн-фильтр для логирования — без отдельного класса
app.MapGet("/orders/{id:int}", GetOrder)
   .AddEndpointFilter(async (context, next) =>
   {
       var sw = Stopwatch.StartNew();
       var result = await next(context);
       app.Logger.LogInformation("GetOrder took {Elapsed}ms", sw.ElapsedMilliseconds);
       return result;
   });
```

Фильтры выполняются в порядке добавления и оборачивают друг друга, как middleware; можно останавливать конвейер, вернув результат раньше `next(context)`.

## Route constraints и типизированные параметры маршрута

```csharp
app.MapGet("/products/{id:int}", (int id) => ...);              // только целое
app.MapGet("/products/{sku:alpha}", (string sku) => ...);         // только буквы
app.MapGet("/products/{id:int:min(1)}", (int id) => ...);         // составные constraints
app.MapGet("/files/{name:regex(^[\\w\\-]+\\.pdf$)}", (string name) => ...);
app.MapGet("/orders/{id:guid}", (Guid id) => ...);
app.MapGet("/archive/{year:int:range(2000,2100)}/{month:int:range(1,12)}", (int year, int month) => ...);
```

| Constraint | Пример | Смысл |
|---|---|---|
| `int`, `long`, `guid`, `bool`, `decimal`, `datetime` | `{id:int}` | тип значения |
| `alpha` | `{slug:alpha}` | только латинские буквы |
| `min(n)` / `max(n)` / `range(a,b)` | `{id:int:min(1)}` | диапазон числа |
| `length(n)` / `length(min,max)` | `{code:length(3,5)}` | длина строки |
| `regex(pattern)` | `{name:regex(...)}` | произвольный шаблон |
| `?` (опциональность) | `{id:int?}` | параметр необязателен |

Если constraint не совпал — маршрут просто не матчится (переходит к следующему кандидату или к `404`), в отличие от ошибки биндинга (`400`), которая возникает, когда путь совпал, но значение нельзя привести к типу параметра делегата.

Для повторяющихся правил, которых нет из коробки, можно зарегистрировать свой `IRouteConstraint` — подробнее в [[Роутинг]].

## Организация кода: как не превратить `Program.cs` в свалку

Когда эндпоинтов становится больше десятка, весь `Program.cs` быстро разрастается. Практические паттерны:

### 1. Extension-методы на `IEndpointRouteBuilder` по фиче

```csharp
public static class OrderEndpoints
{
    public static IEndpointRouteBuilder MapOrderEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/orders").WithTags("Orders");

        group.MapGet("/{id:int}", GetOrder).WithName("GetOrder");
        group.MapPost("/", CreateOrder);
        group.MapPut("/{id:int}", UpdateOrder);
        group.MapDelete("/{id:int}", DeleteOrder);

        return app;
    }

    private static Results<Ok<Order>, NotFound> GetOrder(int id, IOrderService svc) =>
        svc.GetById(id) is { } order ? TypedResults.Ok(order) : TypedResults.NotFound();

    private static async Task<Results<Created<Order>, ValidationProblem>> CreateOrder(
        CreateOrderDto dto, IOrderService svc) => /* ... */;

    // UpdateOrder, DeleteOrder аналогично
}
```

```csharp
// Program.cs остаётся коротким и читаемым
app.MapOrderEndpoints();
app.MapCustomerEndpoints();
app.MapInventoryEndpoints();
```

Это самый распространённый и рекомендуемый Microsoft подход: один статический класс на "фичу" (feature folder), с приватными методами-обработчиками — легко находить, легко юнит-тестировать (методы `internal`/`private static` с `InternalsVisibleTo`, либо `public static` для прямого доступа из тестов).

### 2. Интерфейс "модуля эндпоинтов"

Для больших приложений полезно ввести единый контракт, чтобы регистрация всех фич происходила рефлексией/сканированием сборки, а не списком вызовов:

```csharp
public interface IEndpointModule
{
    void MapEndpoints(IEndpointRouteBuilder app);
}

public class OrderEndpointsModule : IEndpointModule
{
    public void MapEndpoints(IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/orders");
        group.MapGet("/{id:int}", ...);
    }
}

// Program.cs
var modules = typeof(Program).Assembly.GetTypes()
    .Where(t => typeof(IEndpointModule).IsAssignableFrom(t) && !t.IsInterface)
    .Select(Activator.CreateInstance)
    .Cast<IEndpointModule>();

foreach (var module in modules)
    module.MapEndpoints(app);
```

### 3. REPR-паттерн (Request-Endpoint-Response) / библиотека Carter

Сообщество (в частности пакет **Carter**, надстройка над Minimal API) продвигает подход "один класс — один эндпоинт": каждый эндпоинт — отдельный небольшой класс, реализующий `ICarterModule`, с методом `AddRoutes`. Это доводит feature-folder идею до предела — вместо одного класса на всю фичу получаем по классу на каждый route. Плюс: максимальная изоляция, легко находить конкретный обработчик по имени файла. Минус: больше файлов, больше навигации между ними для понимания фичи целиком.

```csharp
// Пример в духе Carter (без подключения самой библиотеки)
public class GetOrderEndpoint : ICarterModule
{
    public void AddRoutes(IEndpointRouteBuilder app)
    {
        app.MapGet("/orders/{id:int}", (int id, IOrderService svc) =>
            svc.GetById(id) is { } o ? TypedResults.Ok(o) : Results.NotFound());
    }
}
```

> [!tip] Что выбрать
> - До ~20 эндпоинтов — extension-методы по фиче (вариант 1) достаточно, не нужна дополнительная библиотека.
> - Много команд/фич в одном сервисе, нужна строгая изоляция — модульный подход (вариант 2) или Carter.
> - Никогда не стоит держать больше 5–7 `Map*` вызовов прямо в `Program.cs` "для скорости" — это моментально деградирует до нечитаемого файла, который боятся трогать.

## Метадата и OpenAPI

```csharp
app.MapGet("/orders/{id:int}", GetOrder)
   .WithName("GetOrderById")
   .WithTags("Orders")
   .WithSummary("Получить заказ по идентификатору")
   .WithDescription("Возвращает заказ или 404, если не найден")
   .Produces<Order>(StatusCodes.Status200OK)
   .Produces(StatusCodes.Status404NotFound)
   .WithOpenApi();
```

С `TypedResults` большая часть `.Produces<T>()` не нужна — генератор OpenAPI выводит коды и типы прямо из сигнатуры (`Results<Ok<Order>, NotFound>`). `.WithName()` остаётся важным отдельно — это имя используется `LinkGenerator`/`Results.CreatedAtRoute` для построения URL без хардкода строк, и оно же становится `operationId` в OpenAPI-документе.

.NET 10 подключает встроенную генерацию OpenAPI через `builder.Services.AddOpenApi()` и `app.MapOpenApi()` — без Swashbuckle, с частично source-generated метадатой для меньшего рефлексийного оверхеда на старте. Полная механика, JSON Schema, Scalar UI вместо Swagger UI — в [[OpenAPI и Swagger в .NET 10]]; здесь важно только то, что чем идиоматичнее написан Minimal API-эндпоинт (`TypedResults`, явные DTO), тем меньше ручной метадаты нужно дописывать.

## Тестирование

Так как обработчик — обычный статический (или instance) метод с явными параметрами, а не член класса с зависимостью от `HttpContext`/`ControllerContext`, юнит-тест часто не требует вообще никакой веб-инфраструктуры:

```csharp
[Fact]
public async Task CreateOrder_ReturnsCreated_WhenValid()
{
    var svc = Substitute.For<IOrderService>();
    svc.CreateAsync(Arg.Any<CreateOrderDto>())
       .Returns(new Order { Id = 1, ExternalId = "X1" });

    var validator = new CreateOrderDtoValidator();
    var dto = new CreateOrderDto("X1", 100m);

    var result = await OrderEndpoints.CreateOrder(dto, svc, validator);

    var created = Assert.IsType<Created<Order>>(result.Result);
    Assert.Equal("/orders/1", created.Location);
}
```

Никакого `WebApplicationFactory`, никакого поднятого сервера — просто вызов метода с фейковыми зависимостями и проверка типа `TypedResults`-результата. Это ключевое практическое преимущество над MVC-контроллерами, где даже простой экшен часто завязан на `ControllerBase.Url`/`ControllerBase.User` и требует настройки `ControllerContext` в тесте.

Такие тесты покрывают бизнес-логику обработчика, но **не** покрывают: реальную маршрутизацию, биндинг параметров из настоящего HTTP-запроса, middleware-конвейер, сериализацию. Для этого нужен полноценный интеграционный тест поверх `WebApplicationFactory` — см. [[Интеграционные тесты и WebApplicationFactory]]. Практика: юнит-тесты на все ветки бизнес-логики в обработчиках, плюс небольшой набор интеграционных тестов "по счастливому пути" на каждый эндпоинт, чтобы поймать ошибки биндинга/роутинга/DI-регистрации, которые юнит-тест в принципе не видит.

## Minimal API vs MVC-контроллеры

| Критерий | Minimal API | MVC-контроллеры |
|---|---|---|
| Boilerplate | минимальный: делегат + маршрут | класс, атрибуты, конструктор для DI |
| Discoverability при малом числе эндпоинтов | высокая (всё в одном месте с `MapGroup`) | ниже — нужно искать класс контроллера |
| Discoverability при большом числе эндпоинтов | требует дисциплины (feature folders) | конвенция уже даёт структуру (один класс = один ресурс) |
| Фильтры | `IEndpointFilter`, тонкий пайплайн, применяется точечно | `IActionFilter`/`IResourceFilter`/`IExceptionFilter`/`IResultFilter` — богаче, но тяжелее |
| Модель биндинга | явные правила инференции по типу параметра | `[ApiController]` включает автоматическую инференцию + автоматический `400` на невалидной модели |
| Валидация "из коробки" | нет автоматической — нужно вызвать явно (или новый `AddValidation()` в .NET 10) | `[ApiController]` валидирует `ModelState` автоматически до вызова экшена |
| Производительность на простых сценариях | выше — нет накладных расходов action pipeline, если фильтры не добавлены | ниже — action invoker, filter pipeline присутствуют всегда |
| Razor Views / server-rendered HTML | не предназначен | нативная поддержка (`View()`, `PartialView()`) |
| Конвенции на уровне приложения (`ApplicationModel`, global filters через `MvcOptions`) | нет прямого аналога | есть — удобно для больших легаси-кодовых баз с едиными правилами |
| Версионирование, сложные атрибутные маршруты (`[Route]` шаблоны с явными constraint-наборами) | через `MapGroup` + constraints, чуть более "руками" | сложившиеся паттерны, много готовых библиотек заточены под контроллеры |
| Тестируемость обработчика | вызов метода напрямую, без веб-инфраструктуры | часто требует настройки `ControllerContext`/`ActionContext` |

Когда MVC-контроллеры по-прежнему разумный выбор:

- **Большая существующая кодовая база** уже на контроллерах — миграция ради миграции не окупается.
- **View-based приложения** (server-rendered HTML, Razor Pages/MVC Views) — Minimal API не заменяет эту модель.
- **Сложные, нестандартные сценарии биндинга модели**, завязанные на кастомные `IModelBinderProvider`, `IValueProvider` — инфраструктура MVC для этого богаче.
- **Единые конвенции на уровне всего приложения** (например, все экшены определённого типа должны получать один и тот же фильтр без явного указания на каждом) — `ApplicationModel`/`IApplicationModelConvention` есть только у MVC.

> [!question]- Значит ли "основной подход в 2026", что MVC-контроллеры устарели?
> Нет. Устарело писать *новый JSON Web API* через контроллеры, когда Minimal API решает ту же задачу с меньшим бойлерплейтом и более явной моделью биндинга. Razor Views, крупные легаси-системы и специфичные конвенции — по-прежнему законные причины остаться на контроллерах или смешивать оба стиля в одном приложении (это официально поддерживается: `app.MapControllers()` и `app.MapGet(...)` спокойно живут рядом).

## Полный пример: CRUD `/orders`

```csharp
// Endpoints/OrderEndpoints.cs
public static class OrderEndpoints
{
    public static IEndpointRouteBuilder MapOrderEndpoints(this IEndpointRouteBuilder app)
    {
        var orders = app.MapGroup("/orders")
            .WithTags("Orders")
            .RequireAuthorization()
            .AddEndpointFilter<LoggingFilter>();

        orders.MapGet("/", ListOrders);
        orders.MapGet("/{id:int:min(1)}", GetOrder).WithName("GetOrder");

        orders.MapPost("/", CreateOrder)
              .AddEndpointFilter<ValidationFilter<CreateOrderDto>>();

        orders.MapPut("/{id:int:min(1)}", UpdateOrder)
              .AddEndpointFilter<ValidationFilter<UpdateOrderDto>>();

        orders.MapDelete("/{id:int:min(1)}", DeleteOrder);

        return app;
    }

    private static Results<Ok<IReadOnlyList<Order>>, ValidationProblem> ListOrders(
        [AsParameters] OrderQuery query, IOrderService svc)
    {
        if (query.PageSize is < 1 or > 100)
            return TypedResults.ValidationProblem(new Dictionary<string, string[]>
            {
                ["pageSize"] = ["Должно быть от 1 до 100"]
            });

        return TypedResults.Ok(svc.Search(query));
    }

    private static Results<Ok<Order>, NotFound> GetOrder(int id, IOrderService svc) =>
        svc.GetById(id) is { } order
            ? TypedResults.Ok(order)
            : TypedResults.NotFound();

    private static async Task<Results<Created<Order>, Conflict<string>>> CreateOrder(
        CreateOrderDto dto, IOrderService svc, CancellationToken ct)
    {
        if (await svc.ExistsAsync(dto.ExternalId, ct))
            return TypedResults.Conflict($"Order '{dto.ExternalId}' already exists");

        var order = await svc.CreateAsync(dto, ct);
        return TypedResults.Created($"/orders/{order.Id}", order);
    }

    private static async Task<Results<NoContent, NotFound>> UpdateOrder(
        int id, UpdateOrderDto dto, IOrderService svc, CancellationToken ct) =>
        await svc.UpdateAsync(id, dto, ct)
            ? TypedResults.NoContent()
            : TypedResults.NotFound();

    private static async Task<Results<NoContent, NotFound>> DeleteOrder(
        int id, IOrderService svc, CancellationToken ct) =>
        await svc.DeleteAsync(id, ct)
            ? TypedResults.NoContent()
            : TypedResults.NotFound();
}

public readonly record struct OrderQuery(
    [FromQuery] string? Status,
    [FromQuery] int Page = 1,
    [FromQuery] int PageSize = 20);

public class ValidationFilter<T> : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context, EndpointFilterDelegate next)
    {
        var arg = context.Arguments.OfType<T>().FirstOrDefault();
        var validator = context.HttpContext.RequestServices.GetService<IValidator<T>>();
        if (validator is not null && arg is not null)
        {
            var result = await validator.ValidateAsync(arg);
            if (!result.IsValid)
                return TypedResults.ValidationProblem(result.ToDictionary());
        }
        return await next(context);
    }
}

public class LoggingFilter : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context, EndpointFilterDelegate next)
    {
        var logger = context.HttpContext.RequestServices
            .GetRequiredService<ILoggerFactory>()
            .CreateLogger("Orders");
        var sw = Stopwatch.StartNew();
        var result = await next(context);
        logger.LogInformation("{Method} {Path} -> {Elapsed}ms",
            context.HttpContext.Request.Method,
            context.HttpContext.Request.Path,
            sw.ElapsedMilliseconds);
        return result;
    }
}
```

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddScoped<IOrderService, OrderService>();
builder.Services.AddScoped<IValidator<CreateOrderDto>, CreateOrderDtoValidator>();
builder.Services.AddScoped<IValidator<UpdateOrderDto>, UpdateOrderDtoValidator>();
builder.Services.AddAuthentication().AddJwtBearer();
builder.Services.AddAuthorization();
builder.Services.AddOpenApi();

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();

app.MapOrderEndpoints();
app.MapOpenApi();

app.Run();
```

Здесь собраны почти все элементы заметки: `MapGroup` с общей авторизацией и фильтром логирования, route constraints (`{id:int:min(1)}`), `[AsParameters]` для query, endpoint filter для валидации, `TypedResults` с union-возвратом на каждом методе, extension-метод для регистрации всей фичи одним вызовом.

## Частые ошибки

> [!warning] Подводные камни
> - **Два комплексных параметра без атрибутов.** Фреймворк не может определить, какой из них — тело запроса; будет исключение о неоднозначности источника при старте/первом запросе. Решение — один DTO или `[AsParameters]` для не-body параметров.
> - **Совпадение имени параметра делегата с именем сегмента маршрута, которое вы не имели в виду биндить.** `app.MapGet("/users/{id}", (int id, string name) => ...)` — если в query тоже есть `name`, конфликта нет, но если ожидалось, что `name` — часть пути, а в шаблоне его не оказалось, биндинг молча уйдёт в query и вернёт `null`/дефолт вместо ошибки. Всегда сверяйте шаблон маршрута и сигнатуру визуально.
> - **Захват сервисов в статический делегат.** `IOrderService svc` в параметрах — это **инъекция на каждый запрос** (через DI), а не захват сервиса из внешней области видимости. Ошибка новичков — резолвить сервис один раз в `Program.cs` через `app.Services.GetRequiredService<T>()` и **захватить его в лямбду как замыкание**:
>   ```csharp
>   // ОПАСНО: сервис резолвится один раз при старте приложения
>   var svc = app.Services.GetRequiredService<IOrderService>();
>   app.MapGet("/orders/{id:int}", (int id) => svc.GetById(id));
>   ```
>   Если `IOrderService` зарегистрирован как `Scoped` (а `DbContext`-зависимые сервисы обычно именно `Scoped`), захваченный экземпляр стал **captive dependency**: один и тот же `DbContext` будет использоваться во всех запросах конкурентно — коррупция состояния, потоко-небезопасность, утечка памяти на долгоживущий scope. Правильно — объявлять сервис параметром делегата, тогда DI резолвит его в scope каждого конкретного запроса. Подробнее про сам механизм — в [[Captive dependency и типичные ошибки DI]].
> - **Чрезмерная опора на неявный биндинг делает сигнатуру нечитаемой.** `(string a, string b, string c, IService s)` — непонятно без документации, что откуда берётся. На команде из нескольких разработчиков полезно явно расставлять `[FromQuery]`/`[FromRoute]` даже там, где инференция и так справится — ради читаемости, а не потому что это обязательно.
> - **Ожидание автоматической валидации DataAnnotations, как в MVC.** `[ApiController]` в контроллерах включает автоматический `400` при нарушении `[Required]`/`[Range]` и т.п. В Minimal API этого нет по умолчанию — либо вызывайте валидацию сами (FluentValidation, см. [[FluentValidation]]), либо подключайте новый встроенный валидатор .NET 10 через `AddValidation()` (детали — в [[Model binding и валидация]]).
> - **`TypedResults.Ok(null)` там, где ожидался `NotFound`.** `TypedResults.Ok<T>(default)` сериализует `null` в тело с кодом `200` — семантически неверно для "ресурс не найден". Проверяйте на `null` до вызова `Ok` и явно возвращайте `TypedResults.NotFound()`.

## Вопросы с собеседований

> [!question]- Чем `TypedResults` лучше `Results` и почему это влияет на генерацию OpenAPI?
> `Results.Ok(...)` возвращает `IResult` — тип стёрт, компилятор и генератор OpenAPI не знают, что конкретно вернёт метод, поэтому метадату (`200`, тип тела) приходится прописывать вручную через `.Produces<T>()`. `TypedResults.Ok<T>(...)` возвращает конкретный тип `Ok<T>`, который реализует `IEndpointMetadataProvider` — по сигнатуре метода (особенно в связке с `Results<T1, T2, ...>`) OpenAPI-генератор автоматически строит список возможных ответов без единой ручной аннотации. Дополнительный бонус — юнит-тесты проверяют конкретный тип результата (`Assert.IsType<NotFound>(...)`), а не разбирают `IResult` рефлексией.

> [!question]- Почему у одного Minimal API эндпоинта может быть только один параметр, привязанный к телу запроса?
> `HttpContext.Request.Body` — это forward-only поток (см. [[HTTP: методы, коды, заголовки, кеширование]]): прочитать его дважды без явного буферизирования нельзя. Если бы фреймворк позволил два параметра из body, второй десериализатор получил бы уже пустой поток. Поэтому правило: не более одного комплексного типа без атрибута на эндпоинт — остальные данные передаются через маршрут, query, заголовки или группируются через `[AsParameters]`, которое НЕ создаёт второе "тело", а лишь распаковывает простые значения.

> [!question]- В чём разница между `[FromServices]` и автоматической инференцией DI-параметров?
> С .NET 7 фреймворк сам определяет, что параметр — DI-сервис, если его тип зарегистрирован в `IServiceCollection`, без явного атрибута. `[FromServices]` — явная аннотация, которая либо документирует намерение для читателей кода, либо разрешает неоднозначность в редких случаях (например, generic-обёртки, которые теоретически могли бы матчиться и как DTO). Функционально в подавляющем большинстве случаев они эквивалентны; команды часто требуют явный атрибут исключительно ради явности кода на ревью.

> [!question]- Чем `MapGroup` отличается от простого написания общего префикса в каждом `Map*`?
> `MapGroup` не просто добавляет префикс строкой — он создаёт `RouteGroupBuilder`, который агрегирует метадату (авторизация, фильтры, теги) и применяет её ко всем маршрутам, зарегистрированным через этот builder, включая вложенные группы. Помимо избавления от дублирования `.RequireAuthorization()` на каждом эндпоинте, это даёт единую точку для endpoint filters уровня всей группы (например, общий фильтр логирования или rate limiting), которые выполняются для каждого запроса в группе автоматически — без риска забыть навесить фильтр на новый эндпоинт.

> [!question]- Почему захват DI-сервиса в переменную снаружи `MapGet`-лямбды — опасная практика?
> `app.Services.GetRequiredService<T>()`, вызванный один раз при старте до `app.Run()`, резолвит сервис из **корневого провайдера** (root scope), который живёт всё время работы приложения. Если сервис зарегистрирован как `Scoped` (типично для всего, что зависит от `DbContext`), захваченный в лямбде экземпляр становится captive dependency: все параллельные запросы будут делить один и тот же `DbContext`/сервис, что при конкурентном доступе ломает состояние и потокобезопасность. Правильный способ — указывать сервис параметром делегата: тогда DI резолвит его из **scope конкретного запроса** при каждом вызове, и жизненный цикл соблюдается корректно.

> [!question]- Когда вы бы предпочли MVC-контроллеры Minimal API в новом проекте на .NET 10?
> Если приложение рендерит HTML на сервере через Razor Views — эта модель осталась только у MVC. Если команда переносит большую существующую кодовую базу на контроллерах — миграция ради стиля не окупает риск регрессий. Если нужны специфичные конвенции на уровне всего приложения (`IApplicationModelConvention`) или сложные кастомные `IModelBinderProvider` — инфраструктура MVC для этого богаче и обкатана дольше. Во всех остальных случаях — новый JSON API, микросервис, CRUD-бэкенд — Minimal API даёт меньше кода и такую же (или лучшую) производительность.

## Итог

- Minimal API — эндпоинты как делегаты на `IEndpointRouteBuilder`, без классов-контроллеров; с .NET 10 это основной способ строить Web API.
- Параметры биндятся по предсказуемым правилам: маршрут → query → сервис/специальный тип → тело (максимум один комплексный параметр без атрибута); `[AsParameters]` группирует, `BindAsync`/`IBindableFromHttpContext<T>` — кастомный биндинг.
- `TypedResults` вместо `Results` — компилируемая типизация, автоматическая OpenAPI-метадата, прямая тестируемость; `Results<T1, T2, ...>` — идиоматичный union-возврат для методов с несколькими исходами.
- `MapGroup` группирует маршруты, метадату, авторизацию и фильтры; `IEndpointFilter` — точечный аналог middleware на уровне одного эндпоинта или группы.
- Растущий `Program.cs` лечится extension-методами `Map<Feature>Endpoints` по фиче, модульным сканированием сборки или REPR-паттерном (Carter).
- Тестировать обработчики можно без веб-сервера — это прямое обычных методов; для покрытия роутинга/биндинга/middleware всё равно нужен `WebApplicationFactory`.
- Главные ловушки: два body-параметра, captive dependency при захвате Scoped-сервиса вне сигнатуры делегата, отсутствие автоматической DataAnnotations-валидации (в отличие от `[ApiController]`).

## Связанное

- [[MVC и контроллеры]]
- [[Model binding и валидация]]
- [[FluentValidation]]
- [[Фильтры и endpoint filters]]
- [[Роутинг]]
- [[Middleware и конвейер обработки запроса]]
- [[Обработка ошибок и ProblemDetails]]
- [[OpenAPI и Swagger в .NET 10]]
- [[Server-Sent Events и стриминг ответов]]
- [[Загрузка файлов и работа с формами]]
- [[Интеграционные тесты и WebApplicationFactory]]
- [[Captive dependency и типичные ошибки DI]]
- [[HTTP: методы, коды, заголовки, кеширование]]
