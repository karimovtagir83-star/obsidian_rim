---
tags: [раздел-09, rest, api, дизайн, собес, middle]
aliases: [REST, RESTful API, Richardson Maturity Model, HATEOAS, API design]
---

# REST: принципы и проектирование API

> [!abstract] Коротко
> REST (Representational State Transfer) — не протокол и не стандарт, а набор архитектурных ограничений из диссертации Роя Филдинга (2000). На практике «REST API» почти всегда означает «HTTP+JSON API с ресурсно-ориентированными URL» — то есть уровень 2 по Ричардсону. Ключевая работа при проектировании — не выучить правила, а выбрать границы ресурсов, формат ошибок, схему пагинации и стратегию эволюции контракта.

## Зачем это нужно

До REST был SOAP: XML-конверт, WSDL-описание, один эндпоинт `/service.asmx` и RPC-вызовы внутри тела. Работало, но было тяжело: клиенту нужен генератор кода, промежуточные кеши бесполезны (всё `POST`), отладить руками нельзя.

REST предложил другое: **использовать HTTP как он задуман** — URI идентифицирует ресурс, метод описывает операцию, коды ответа несут смысл, кеширование работает бесплатно. Отсюда его победа: `curl` заменяет SDK, CDN кеширует `GET`, а балансировщик понимает `503`.

Обратная сторона: REST не описывает, как делать сложные операции («перевести деньги», «пересчитать корзину»). Отсюда постоянный спор в командах и появление альтернатив — gRPC, GraphQL, а внутри самого HTTP — «RPC-style эндпоинты».

## Шесть ограничений REST

| Ограничение | Что требует | Что даёт |
|---|---|---|
| **Клиент-сервер** | разделение ответственности UI и хранения | независимая эволюция сторон |
| **Stateless** | каждый запрос самодостаточен, сервер не хранит сессию | горизонтальное масштабирование, любой инстанс обслужит любой запрос |
| **Cacheable** | ответ явно помечен как кешируемый или нет | снижение latency и нагрузки |
| **Uniform interface** | единообразный способ работы с ресурсами | клиент не знает про внутренности сервера |
| **Layered system** | между клиентом и сервером могут стоять прокси/шлюзы | балансировка, кеши, безопасность |
| **Code on demand** (опционально) | сервер может отдать исполняемый код | почти не используется в API |

**Stateless — самое практически важное.** Оно означает: никакого `Session["cart"]`. Состояние либо у клиента (токен со claims), либо в общем хранилище (Redis, БД), но не в памяти конкретного инстанса. Именно поэтому в ASP.NET Core сессии — редкость для API, а токены — норма (см. [[JWT и Bearer-аутентификация]]).

**Uniform interface** раскладывается на четыре подограничения:
1. Идентификация ресурсов через URI.
2. Манипуляция через представления (клиент присылает JSON, а не команду «UPDATE»).
3. Самодостаточные сообщения (`Content-Type` говорит, как понимать тело).
4. HATEOAS — гипермедиа как движок состояния приложения.

Последнее почти никто не делает, и об этом ниже.

## Уровни зрелости Ричардсона

Модель Леонарда Ричардсона — удобная линейка, по которой измеряют «насколько API вообще REST».

```
Уровень 0: Болото POX
  POST /api  { "action": "getOrder", "id": 5 }
  Один URI, один метод, всё в теле. Это RPC поверх HTTP.

Уровень 1: Ресурсы
  POST /api/orders/5   { "action": "get" }
  Появились адресуемые сущности, но метод по-прежнему один.

Уровень 2: HTTP-глаголы + коды  ← ЗДЕСЬ 95 % РЕАЛЬНЫХ API
  GET    /api/orders/5      → 200
  POST   /api/orders        → 201 + Location
  DELETE /api/orders/5      → 204
  Метод несёт семантику, код ответа несёт результат.

Уровень 3: HATEOAS
  GET /api/orders/5 → 200
  {
    "id": 5, "status": "Pending",
    "_links": {
      "self":   { "href": "/api/orders/5" },
      "cancel": { "href": "/api/orders/5/cancel", "method": "POST" },
      "pay":    { "href": "/api/orders/5/payment", "method": "POST" }
    }
  }
  Клиент узнаёт доступные действия из ответа, а не из документации.
```

Филдинг настаивал, что без уровня 3 это не REST. Индустрия его не послушала. Про причины — в разделе про HATEOAS.

## Проектирование URL

### Базовые правила

| Правило | Плохо | Хорошо |
|---|---|---|
| Существительные, не глаголы | `/getUsers`, `/createUser` | `GET /users`, `POST /users` |
| Множественное число для коллекций | `/user/5` | `/users/5` |
| Иерархия для владения | `/comments?postId=5` | `/posts/5/comments` |
| kebab-case в путях | `/orderItems`, `/order_items` | `/order-items` |
| Без расширений | `/users.json` | `/users` + `Accept` |
| Без версии в имени ресурса | `/usersV2` | `/v2/users` |
| Без trailing slash | `/users/` | `/users` |

### Вложенность

Вложенный путь оправдан, когда дочерний ресурс **не существует без родителя**:

```
GET  /posts/42/comments          ← комментарии принадлежат посту
POST /posts/42/comments
GET  /comments/1001              ← но и прямой доступ полезен, если id глобальный
```

Правило: **не больше двух уровней вложенности**. `/users/1/posts/42/comments/1001/likes` — признак того, что вы моделируете граф в URL. Один уровень + фильтры почти всегда лучше.

### Когда ресурсы не работают: операции-действия

Не всё выражается через CRUD. «Отменить заказ», «отправить письмо», «пересчитать» — это операции, а не изменения представления. Три подхода:

```csharp
// 1. Sub-resource как состояние (наиболее «RESTful»)
// PUT /orders/5/cancellation  { "reason": "..." }

// 2. RPC-style действие в URL (наиболее читаемо, повсеместно)
// POST /orders/5/cancel  { "reason": "..." }

// 3. Изменение поля через PATCH
// PATCH /orders/5  { "status": "Cancelled" }
```

Практика: **вариант 2**. Он честно говорит «это команда», его невозможно перепутать с изменением данных, и он позволяет отдельно авторизовать действие. Вариант 3 плох тем, что бизнес-логика «можно ли перевести из Pending в Cancelled» прячется в валидации поля.

```csharp
var orders = app.MapGroup("/api/orders").RequireAuthorization();

orders.MapGet("/", GetOrders);
orders.MapGet("/{id:int}", GetOrder);
orders.MapPost("/", CreateOrder);
orders.MapPut("/{id:int}", ReplaceOrder);
orders.MapDelete("/{id:int}", DeleteOrder);

// Действия — отдельными эндпоинтами с отдельными политиками
orders.MapPost("/{id:int}/cancel", CancelOrder)
      .RequireAuthorization("orders:cancel");
orders.MapPost("/{id:int}/refund", RefundOrder)
      .RequireAuthorization("orders:refund");
```

Про `MapGroup` — см. [[Роутинг]] и [[Minimal API]].

## Фильтрация, сортировка, пагинация

Это то, что спрашивают на собеседовании чаще самих принципов REST.

### Фильтрация

```
GET /api/products?category=laptops&minPrice=500&maxPrice=1500&inStock=true
```

Простые equality-фильтры — отдельными query-параметрами. Не изобретайте свой язык запросов (`?filter=price>500 AND category='x'`) — его придётся парсить, он открывает дорогу к инъекциям и его никто не сможет использовать без вашей документации. Если нужен настоящий язык запросов — берите готовый стандарт (OData) целиком или переходите на GraphQL.

```csharp
public sealed record ProductQuery(
    string? Category = null,
    decimal? MinPrice = null,
    decimal? MaxPrice = null,
    bool? InStock = null,
    string? Sort = null,
    int Page = 1,
    int PageSize = 20);

app.MapGet("/api/products", async (
    [AsParameters] ProductQuery query, AppDbContext db, CancellationToken ct) =>
{
    var q = db.Products.AsNoTracking();

    if (query.Category is { Length: > 0 })
        q = q.Where(p => p.Category == query.Category);
    if (query.MinPrice is { } min)
        q = q.Where(p => p.Price >= min);
    if (query.MaxPrice is { } max)
        q = q.Where(p => p.Price <= max);
    if (query.InStock == true)
        q = q.Where(p => p.Stock > 0);

    // Сортировка — только по белому списку. Никогда не по строке из запроса напрямую.
    q = query.Sort switch
    {
        "price"      => q.OrderBy(p => p.Price),
        "-price"     => q.OrderByDescending(p => p.Price),
        "name"       => q.OrderBy(p => p.Name),
        "-createdAt" => q.OrderByDescending(p => p.CreatedAt),
        _            => q.OrderByDescending(p => p.Id) // детерминированный дефолт
    };

    var size = Math.Clamp(query.PageSize, 1, 100);
    var total = await q.CountAsync(ct);
    var items = await q.Skip((query.Page - 1) * size).Take(size)
                       .Select(p => new ProductDto(p.Id, p.Name, p.Price))
                       .ToListAsync(ct);

    return TypedResults.Ok(new PagedResult<ProductDto>(items, query.Page, size, total));
});

public sealed record PagedResult<T>(IReadOnlyList<T> Items, int Page, int PageSize, int Total)
{
    public int TotalPages => (int)Math.Ceiling(Total / (double)PageSize);
}
```

`[AsParameters]` собирает все свойства record из query-строки — см. [[Minimal API]].

> [!danger] Сортировка по строке из запроса — дыра
> `q.OrderBy(query.Sort)` через `System.Linq.Dynamic` или конкатенацию в raw SQL — это SQL-инъекция и утечка внутренней модели. Всегда белый список через `switch`. См. [[SQL-инъекции и параметризация]].

### Пагинация: два подхода

| | Offset (`page`/`skip`) | Cursor (keyset) |
|---|---|---|
| Запрос | `?page=5&pageSize=20` | `?after=eyJpZCI6MTAwfQ&limit=20` |
| SQL | `OFFSET 80 LIMIT 20` | `WHERE (created_at, id) < (@ts, @id) LIMIT 20` |
| Сложность на глубокой странице | O(offset) — БД считает и выбрасывает 80 000 строк | O(log n) по индексу |
| Прыжок на страницу 500 | можно | нельзя |
| Общее число элементов | есть (но `COUNT(*)` дорог) | обычно нет |
| Устойчивость при вставках | **нет**: новая запись сдвигает всё, элемент дублируется или пропадает | да |
| Где применять | админки, небольшие таблицы | ленты, экспорт, мобильные приложения |

```csharp
// Cursor-пагинация: курсор кодирует последнюю позицию, а не номер страницы
public sealed record Cursor(DateTime CreatedAt, int Id)
{
    public string Encode() => Convert.ToBase64String(
        JsonSerializer.SerializeToUtf8Bytes(this)).TrimEnd('=');

    public static Cursor? Decode(string? raw)
    {
        if (string.IsNullOrEmpty(raw)) return null;
        var padded = raw.PadRight((raw.Length + 3) / 4 * 4, '=');
        return JsonSerializer.Deserialize<Cursor>(Convert.FromBase64String(padded));
    }
}

app.MapGet("/api/feed", async (string? after, int limit, AppDbContext db, CancellationToken ct) =>
{
    limit = Math.Clamp(limit is 0 ? 20 : limit, 1, 100);
    var cursor = Cursor.Decode(after);

    var q = db.Posts.AsNoTracking()
        .OrderByDescending(p => p.CreatedAt).ThenByDescending(p => p.Id);

    if (cursor is not null)
    {
        // Составное сравнение — иначе записи с одинаковым CreatedAt потеряются
        q = q.Where(p => p.CreatedAt < cursor.CreatedAt
                      || (p.CreatedAt == cursor.CreatedAt && p.Id < cursor.Id));
    }

    // Берём limit + 1, чтобы узнать, есть ли следующая страница, без COUNT
    var rows = await q.Take(limit + 1).ToListAsync(ct);
    var hasMore = rows.Count > limit;
    var page = rows.Take(limit).ToList();

    return TypedResults.Ok(new
    {
        items = page.Select(p => new PostDto(p.Id, p.Title, p.CreatedAt)),
        nextCursor = hasMore && page.Count > 0
            ? new Cursor(page[^1].CreatedAt, page[^1].Id).Encode()
            : null
    });
});
```

Ключевой момент: сортировка **обязана** быть по уникальному составному ключу (`CreatedAt`, `Id`), иначе при равных `CreatedAt` записи будут теряться между страницами. Про индексы под такой запрос — [[Индексы: как работают и когда помогают]].

### Где отдавать метаданные пагинации

Три варианта, все встречаются:

```
1. В теле (самый практичный)
   { "items": [...], "page": 2, "pageSize": 20, "total": 4711 }

2. В заголовках (тело — чистый массив)
   X-Total-Count: 4711
   Link: </api/products?page=3>; rel="next", </api/products?page=1>; rel="prev"

3. RFC 5988 Link только — «правильно» по спеке, но неудобно читать глазами
```

Вариант 1 выигрывает: не требует CORS-настройки `Access-Control-Expose-Headers` (см. [[CORS]]) и виден в браузере. Вариант 2 нужен, если клиент — готовая библиотека вроде `react-admin`, которая ждёт заголовки.

## PATCH: два несовместимых формата

`PATCH` не описывает формат тела — формат задаётся `Content-Type`.

### JSON Merge Patch (RFC 7386) — `application/merge-patch+json`

```json
PATCH /api/products/5
Content-Type: application/merge-patch+json

{ "price": 1299.00, "description": null }
```
Присланные поля заменяются, `null` означает «удалить/обнулить», отсутствующие поля не трогаются. Простой и понятный.

Проблема в C#: как отличить «поле не пришло» от «пришло `null`»? С `record Dto(decimal? Price)` оба случая дают `null`.

```csharp
// Решение 1: JsonElement / JsonNode — читаем только присутствующие поля
app.MapPatch("/api/products/{id:int}", async (
    int id, JsonElement patch, AppDbContext db, CancellationToken ct) =>
{
    var product = await db.Products.FindAsync([id], ct);
    if (product is null) return Results.NotFound();

    if (patch.TryGetProperty("price", out var price))
        product.Price = price.GetDecimal();
    if (patch.TryGetProperty("description", out var desc))
        product.Description = desc.ValueKind == JsonValueKind.Null ? null : desc.GetString();

    await db.SaveChangesAsync(ct);
    return Results.NoContent();
});
```

```csharp
// Решение 2: обёртка Optional<T> — типобезопасно и валидируемо
public readonly struct Optional<T>
{
    public bool HasValue { get; }
    public T? Value { get; }
    public Optional(T? value) { HasValue = true; Value = value; }
}

public sealed class PatchProductDto
{
    public Optional<decimal> Price { get; init; }
    public Optional<string?> Description { get; init; }
}
// Требует своего JsonConverter, зато явно и работает с валидацией.
```

### JSON Patch (RFC 6902) — `application/json-patch+json`

```json
[
  { "op": "replace", "path": "/price", "value": 1299.00 },
  { "op": "add",     "path": "/tags/-", "value": "sale" },
  { "op": "remove",  "path": "/description" },
  { "op": "test",    "path": "/version", "value": 3 }
]
```

Мощнее (умеет массивы, `test` для оптимистичной блокировки), но многословнее и требует пакета `Microsoft.AspNetCore.JsonPatch`. Исторически работал только с `Newtonsoft.Json`; в современных версиях есть вариант на `System.Text.Json`. Клиенты его не любят.

**Вывод для 2026:** для публичных API берите merge patch; JSON Patch — только если клиент уже с ним работает.

> [!warning] `PATCH` и права
> С merge patch легко случайно разрешить изменение поля, которое менять нельзя: `{ "role": "Admin" }` или `{ "balance": 999999 }`. Никогда не применяйте патч к сущности домена напрямую — только к явному DTO с белым списком полей. Это тот же класс уязвимостей, что over-posting/mass assignment.

## HATEOAS: почему им почти не пользуются

Идея красивая: сервер в каждом ответе перечисляет доступные переходы, клиент не хардкодит URL и не дублирует бизнес-правила («кнопку Отменить показываем, если статус Pending»).

```json
{
  "id": 5,
  "total": 1299.00,
  "status": "Pending",
  "_links": {
    "self":    { "href": "/api/orders/5" },
    "cancel":  { "href": "/api/orders/5/cancel", "method": "POST" },
    "invoice": { "href": "/api/orders/5/invoice", "type": "application/pdf" }
  }
}
```

Почему не взлетело:

1. **Клиенты всё равно хардкодят.** Реальный SPA/мобильный клиент пишется под известный API одной командой. Динамическое обнаружение URL — решение проблемы, которой у них нет.
2. **Нет стандарта.** HAL, JSON:API, Siren, Collection+JSON, OData — пять несовместимых форматов. Нет единого — нет универсального клиента, а без универсального клиента нет смысла.
3. **OpenAPI решил задачу дешевле.** Схема контракта + генератор клиента дают типизацию и автодополнение — то, чего HATEOAS не даёт вовсе. См. [[OpenAPI и Swagger в .NET 10]].
4. **Стоимость.** Каждый ответ раздувается, каждый DTO обрастает логикой «какие ссылки показать», кеширование усложняется.
5. **Права всё равно проверяются на сервере.** Наличие ссылки `cancel` не заменяет авторизацию, значит логика дублируется.

Где HATEOAS реально живёт: очень долгоживущие публичные API с независимыми клиентами (GitHub API частично, платёжные системы, некоторые госсервисы), и там, где важна навигация по большому графу (OData в enterprise).

> [!info] Прагматичный компромисс
> Отдавайте не полный HAL, а плоский список разрешённых действий: `"allowedActions": ["cancel", "refund"]`. Фронт получает то, что ему реально нужно (какие кнопки рисовать), без формата гипермедиа и без раздувания ответа.

## Форматы, коды и ошибки

- Один формат тела: JSON, `camelCase` (`System.Text.Json` по умолчанию так и делает в ASP.NET Core). Даты — ISO 8601 с указанием смещения: `2026-08-05T14:30:00+05:00`. См. [[Дата и время: DateTime, DateTimeOffset, TimeProvider]].
- Деньги — не `double`. `decimal` в C#, строка или целое в минорных единицах в JSON.
- Enum'ы — строками, а не числами: числа ломаются при добавлении значения в середину. `JsonStringEnumConverter`.
- Ошибки — RFC 9457 ProblemDetails, единый формат на весь API. См. [[Обработка ошибок и ProblemDetails]].
- Никогда не отдавайте сущности EF Core напрямую: циклы навигационных свойств, утечка полей, ломающиеся контракты при рефакторинге модели. Всегда DTO/record.

```csharp
builder.Services.ConfigureHttpJsonOptions(options =>
{
    options.SerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
    options.SerializerOptions.DefaultIgnoreCondition =
        JsonIgnoreCondition.WhenWritingNull;
    options.SerializerOptions.Converters.Add(new JsonStringEnumConverter());
});
```

## Эволюция контракта

Совместимые (не ломающие) изменения:
- добавить необязательное поле в запрос;
- добавить поле в ответ (клиент обязан игнорировать неизвестные — и `System.Text.Json` по умолчанию так и делает);
- добавить новый эндпоинт;
- добавить новое значение enum, **если** клиенты умеют его игнорировать (спорно — часто ломает).

Ломающие изменения:
- удалить/переименовать поле;
- изменить тип поля (`int` → `string`);
- сделать необязательное поле обязательным;
- изменить формат ошибки или код ответа для существующего сценария;
- сузить диапазон допустимых значений.

На ломающие — версия. Подробно в [[Версионирование API]].

> [!warning] Подводные камни
> - **`total` через `COUNT(*)` на каждый запрос.** На таблице в 50 млн строк с фильтрами это половина времени ответа. Варианты: cursor-пагинация без total, приблизительный count из статистики БД, кеширование count на минуту.
> - **`PUT` как «частичное обновление».** `PUT` по спеке заменяет представление целиком: поля, которых нет в теле, должны обнулиться. Если ваш `PUT` игнорирует отсутствующие поля — это `PATCH`, названный `PUT`, и клиент рано или поздно потеряет данные, не отправив поле.
> - **Возврат сущностей EF Core.** Ленивая загрузка внутри сериализации даёт N+1 или `ObjectDisposedException` (контекст уже освобождён вместе со scope). См. [[EF Core: запросы и загрузка связанных данных]].
> - **Отсутствие детерминированной сортировки.** `Skip/Take` без `OrderBy` — недетерминированный результат: одна и та же страница вернёт разные элементы. PostgreSQL и SQL Server не обязаны сохранять порядок.
> - **Утечка внутренних id.** Последовательные `int` id позволяют перебирать чужие ресурсы и оценивать объём бизнеса. Для публичных API — `Guid` (лучше v7, монотонный) или отдельный внешний идентификатор.
> - **Множественное/единственное число вперемешку.** `/users/5/profile` и `/user/5/orders` в одном API — гарантированные баги интеграции. Выберите конвенцию и зафиксируйте в ADR ([[Документация: ADR и RFC]]).

> [!example] Как делают в бою
> - Публичный контракт описан отдельными `record`-DTO в проекте `*.Contracts`, который можно отдать клиенту как NuGet-пакет.
> - Валидация DTO — на входе, до домена ([[Model binding и валидация]]).
> - Формат ошибок один на весь сервис, зафиксирован в ProblemDetails с `type` = стабильный URI кода ошибки.
> - Пагинация: cursor для лент и экспорта, offset для админок. Максимальный `pageSize` жёстко ограничен на сервере (100–200), иначе клиент однажды закажет `pageSize=1000000`.
> - OpenAPI-документ генерируется на билде и линтуется в CI (Spectral) — контракт не может «уехать» незаметно.
> - Ломающие изменения проходят через депрекацию: заголовок `Sunset`, метрика использования старой версии, потом удаление.

## Вопросы с собеседований

> [!question]- Что такое stateless в REST и почему это важно для масштабирования?
> Stateless означает, что сервер не хранит клиентское состояние между запросами: каждый запрос несёт всё необходимое (токен, идентификаторы, параметры). Практическое следствие — любой инстанс приложения может обслужить любой запрос, поэтому балансировщику не нужны sticky sessions, а масштабирование сводится к добавлению подов. Обратная сторона: состояние переезжает либо в токен (растёт размер каждого запроса, ревокация усложняется), либо во внешнее хранилище (Redis), которое становится точкой отказа. В ASP.NET Core именно из-за stateless для API используют Bearer-токены, а не `HttpContext.Session`.

> [!question]- Ваш API возвращает список из 10 млн записей с offset-пагинацией. Клиент жалуется, что страница 40 000 грузится 30 секунд. Что происходит и как исправить?
> `OFFSET 800000 LIMIT 20` заставляет БД прочитать и отбросить 800 000 строк — стоимость линейна по offset. Плюс `COUNT(*)` по всей таблице с фильтрами. Исправление — keyset (cursor) пагинация: `WHERE (created_at, id) < (@lastTs, @lastId) ORDER BY created_at DESC, id DESC LIMIT 20` с составным индексом по `(created_at, id)`. Это O(log n) независимо от глубины. Платим тем, что нельзя прыгнуть на произвольную страницу и нет общего количества — для ленты это обычно и не нужно. Если total обязателен, отдают приблизительный из статистики (`pg_class.reltuples`).

> [!question]- Чем PUT отличается от PATCH и в чём частая ошибка реализации PUT?
> `PUT` заменяет представление ресурса целиком и идемпотентен: отправили тело — состояние стало ровно таким; поля, отсутствующие в теле, должны получить значения по умолчанию или быть очищены. `PATCH` применяет частичное изменение и в общем случае не идемпотентен. Частая ошибка — реализовать `PUT` как «обнови только переданные поля»: тогда клиент, который забыл отправить поле, думает, что оно сохранилось, а по спеке оно должно было обнулиться. Такой эндпоинт нужно называть `PATCH`. Вторая ошибка — маппить тело `PUT` прямо на entity, что открывает mass assignment.

> [!question]- Почему HATEOAS практически не используют, хотя Филдинг считал его обязательным?
> Три причины. Первая: нет единого формата гипермедиа (HAL, JSON:API, Siren, OData несовместимы), а без единого формата невозможен универсальный клиент — то есть исчезает главная выгода. Вторая: реальные клиенты пишутся под конкретный API одной командой и всё равно хардкодят маршруты; динамическое обнаружение решает проблему, которой у них нет. Третья: OpenAPI дал то, что действительно нужно — машиночитаемую схему и генерацию типизированного клиента, чего HATEOAS не даёт. Прагматичная замена — отдавать в ответе список разрешённых действий (`allowedActions`), чтобы фронт знал, какие кнопки рисовать, не дублируя бизнес-правила.

> [!question]- Как в C# отличить «поле не пришло в PATCH» от «пришло null»?
> Обычный DTO с nullable-свойствами этого не позволяет — оба случая дают `null`. Варианты: (1) принимать `JsonElement`/`JsonNode` и проверять `TryGetProperty` — просто, но теряется типизация и валидация; (2) обёртка `Optional<T>` со флагом `HasValue` и собственным `JsonConverter` — типобезопасно и работает с валидацией, но требует кода; (3) использовать JSON Patch (RFC 6902), где операции `replace`/`remove` различаются явно. В продакшене чаще берут вариант 2 для важных сущностей и вариант 1 для простых случаев.

> [!question]- REST, gRPC или GraphQL — как выбрать?
> REST+JSON: публичные API, много разных клиентов, важна отлаживаемость и кеширование по HTTP. gRPC: межсервисное взаимодействие внутри кластера, нужны низкая latency, строгий контракт (protobuf), стриминг; плохо работает из браузера без прокси. GraphQL: много разнородных клиентов, которым нужны разные подрезы одних данных, и вы готовы платить сложностью (N+1, кеширование, ограничение глубины запросов, авторизация на уровне полей). Практика: REST на границе, gRPC внутри. См. [[gRPC в .NET]] и [[Способы межсервисного взаимодействия]].

## Задачи

### Задача 1. Спроектируйте контракт
Спроектируйте REST-контракт для сервиса задач: проекты, внутри них задачи, у задач комментарии и исполнитель. Нужны: список задач с фильтром по статусу и исполнителю, назначение исполнителя, закрытие задачи, добавление комментария.

> [!success]- Решение
> ```
> GET    /api/projects                                  список проектов
> POST   /api/projects                                  создать
> GET    /api/projects/{projectId}                      детали
>
> GET    /api/projects/{projectId}/tasks?status=open&assigneeId=7&sort=-createdAt&page=1&pageSize=20
> POST   /api/projects/{projectId}/tasks                создать задачу в проекте
> GET    /api/tasks/{taskId}                            прямой доступ (id глобальный)
> PUT    /api/tasks/{taskId}                            полная замена
> PATCH  /api/tasks/{taskId}                            частичное изменение (merge-patch)
> DELETE /api/tasks/{taskId}
>
> PUT    /api/tasks/{taskId}/assignee   { "userId": 7 }  назначить (идемпотентно!)
> DELETE /api/tasks/{taskId}/assignee                    снять исполнителя
> POST   /api/tasks/{taskId}/close      { "resolution": "done" }  действие
>
> GET    /api/tasks/{taskId}/comments?after=<cursor>&limit=50
> POST   /api/tasks/{taskId}/comments
> DELETE /api/comments/{commentId}
> ```
> Обоснования: задачи вложены в проект для создания и листинга (принадлежность), но доступны напрямую по глобальному id — иначе клиенту пришлось бы таскать projectId. Назначение исполнителя — `PUT` на sub-resource, потому что операция идемпотентна и естественно выражается как «состояние поля assignee». Закрытие — `POST .../close`, потому что это команда с побочными эффектами (нотификации, метрики), которую нужно отдельно авторизовать. Комментарии — cursor-пагинация, их может быть много и они добавляются постоянно.

### Задача 2. Merge patch с белым списком
Реализуйте `PATCH /api/users/{id}` с `application/merge-patch+json`, где клиенту разрешено менять только `displayName` и `timeZone`. Попытка передать `role` или `email` должна вернуть `400` с указанием запрещённого поля.

> [!success]- Решение
> ```csharp
> using System.Text.Json;
> using System.Text.Json.Nodes;
>
> static readonly HashSet<string> Allowed = new(StringComparer.Ordinal)
> {
>     "displayName", "timeZone"
> };
>
> app.MapPatch("/api/users/{id:int}", async (
>     int id, JsonObject patch, AppDbContext db, CancellationToken ct) =>
> {
>     var unknown = patch.Select(kv => kv.Key).Where(k => !Allowed.Contains(k)).ToArray();
>     if (unknown.Length > 0)
>     {
>         return Results.ValidationProblem(new Dictionary<string, string[]>
>         {
>             ["$"] = unknown.Select(f => $"Поле '{f}' изменять нельзя.").ToArray()
>         });
>     }
>
>     var user = await db.Users.FindAsync([id], ct);
>     if (user is null) return Results.NotFound();
>
>     if (patch.TryGetPropertyValue("displayName", out var name))
>     {
>         var value = name?.GetValue<string>();
>         if (string.IsNullOrWhiteSpace(value) || value.Length > 100)
>             return Results.ValidationProblem(new Dictionary<string, string[]>
>             {
>                 ["displayName"] = ["Требуется 1–100 символов."]
>             });
>         user.DisplayName = value;
>     }
>
>     if (patch.TryGetPropertyValue("timeZone", out var tz))
>     {
>         var value = tz?.GetValue<string>();
>         if (value is not null && !TimeZoneInfo.TryFindSystemTimeZoneById(value, out _))
>             return Results.ValidationProblem(new Dictionary<string, string[]>
>             {
>                 ["timeZone"] = ["Неизвестная таймзона."]
>             });
>         user.TimeZone = value;   // null допустим — обнуляем
>     }
>
>     await db.SaveChangesAsync(ct);
>     return Results.NoContent();
> })
> .Accepts<JsonObject>("application/merge-patch+json");
> ```
> Важное: белый список **явный и проверяется до чтения полей** — так клиент сразу узнаёт об ошибке, а не молча теряет изменение. `Results.ValidationProblem` отдаёт `HttpValidationProblemDetails` (RFC 9457) — единый формат ошибок. Разница между «`timeZone` не пришёл» и «`timeZone: null`» здесь работает: во втором случае поле обнуляется.

### Задача 3. Уровень зрелости
Оцените уровень по Ричардсону и перепишите на уровень 2:
```
POST /api/service
{ "method": "order.cancel", "params": { "orderId": 5, "reason": "changed mind" } }
→ 200 { "success": false, "errorCode": "ORDER_ALREADY_SHIPPED" }
```

> [!success]- Решение
> Это **уровень 0** (болото POX): один URI, один метод, операция и результат закодированы в теле. Проблемы: retry небезопасен (всё `POST`), нельзя кешировать, балансировщик и APM видят один эндпоинт и не различают нагрузку, ошибка приходит с кодом 200 — метрики 5xx всегда зелёные.
>
> Уровень 2:
> ```csharp
> app.MapPost("/api/orders/{id:int}/cancel", async (
>     int id, CancelOrderRequest body, IOrderService svc, CancellationToken ct) =>
> {
>     var result = await svc.CancelAsync(id, body.Reason, ct);
>     return result switch
>     {
>         CancelResult.Cancelled   => Results.NoContent(),
>         CancelResult.NotFound    => Results.NotFound(),
>         CancelResult.AlreadyShipped => Results.Problem(
>             type: "https://api.example.com/errors/order-already-shipped",
>             title: "Заказ уже отправлен",
>             detail: "Отменить отправленный заказ нельзя, оформите возврат.",
>             statusCode: StatusCodes.Status409Conflict),
>         _ => throw new UnreachableException()
>     };
> })
> .RequireAuthorization("orders:cancel");
>
> public sealed record CancelOrderRequest(string Reason);
> ```
> Что улучшилось: ресурс адресуем, статус-код несёт результат (409 виден в метриках и алертах), ошибка в формате ProblemDetails со стабильным `type`, эндпоинт можно отдельно авторизовать и отдельно замерить в APM. `Results.Problem` вместо своего `{success:false}` — см. [[Обработка ошибок и ProblemDetails]] и [[Result pattern вместо исключений]].

## Итог

- REST — шесть ограничений, из которых практически важнее всего stateless и uniform interface. Реальные API живут на уровне 2 по Ричардсону.
- URL — существительные, коллекции во множественном числе, вложенность не глубже двух уровней; операции-команды оформляйте как `POST /resource/{id}/action`.
- Сортировка — только по белому списку. Пагинация: offset для админок, cursor по составному уникальному ключу для лент.
- `PUT` заменяет целиком и идемпотентен, `PATCH` изменяет частично; для PATCH нужен способ отличить «нет поля» от `null` и обязателен белый список полей.
- HATEOAS не взлетел из-за отсутствия единого формата и появления OpenAPI; практичная замена — список разрешённых действий в ответе.
- Никогда не отдавайте entity напрямую: DTO защищают контракт от рефакторинга модели и от утечки полей.

## Связанное

- [[HTTP: методы, коды, заголовки, кеширование]]
- [[Minimal API]]
- [[Версионирование API]]
- [[Обработка ошибок и ProblemDetails]]
- [[OpenAPI и Swagger в .NET 10]]
- [[Model binding и валидация]]
- [[Способы межсервисного взаимодействия]]
- [[Проект 3 — REST API «Блог»]]
