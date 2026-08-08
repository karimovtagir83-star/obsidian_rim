---
tags: [раздел-09, http, основы, собес, сеть]
aliases: [HTTP, HTTP methods, HTTP status codes, HTTP headers, ETag, Cache-Control]
---

# HTTP: методы, коды, заголовки, кеширование

> [!abstract] Коротко
> HTTP — текстовый (в HTTP/1.1) запрос-ответный протокол поверх TCP: клиент посылает метод + URI + заголовки + необязательное тело, сервер отвечает кодом + заголовками + телом. Вся «магия» веб-фреймворков — это надстройка над этими четырьмя сущностями. Знание семантики методов (безопасность, идемпотентность), кодов и кеширующих заголовков — обязательный минимум: без него нельзя ни спроектировать API, ни отладить прод.

## Зачем это нужно

ASP.NET Core прячет HTTP за `MapGet`, `TypedResults.Ok()` и биндингом параметров. Пока всё работает — знать протокол не обязательно. Проблемы начинаются, когда:

- клиент повторил запрос после таймаута и создал две одинаковые оплаты — потому что `POST` не идемпотентен, а вы не сделали ключ идемпотентности;
- прокси закешировал ответ с персональными данными и отдал их другому пользователю — потому что вы не поставили `Cache-Control: private`;
- мобильное приложение качает 2 МБ JSON на каждый запуск — потому что нет `ETag`;
- балансировщик режет соединение на 60 секунде, а вы стримите — потому что не понимаете, чем HTTP/1.1 отличается от HTTP/2.

Все эти баги решаются не в C#, а в понимании протокола.

## Анатомия запроса и ответа

```
POST /api/orders?draft=true HTTP/1.1          ← стартовая строка: метод, target, версия
Host: api.example.com                          ← обязателен в HTTP/1.1
Content-Type: application/json; charset=utf-8
Content-Length: 34
Authorization: Bearer eyJhbGciOi...
Idempotency-Key: 6f1c...                       ← прикладной заголовок
                                               ← пустая строка = конец заголовков
{"productId":42,"quantity":2}                  ← тело
```

```
HTTP/1.1 201 Created
Location: /api/orders/1001
Content-Type: application/json; charset=utf-8
ETag: "a3f9c1"
Cache-Control: no-store

{"id":1001,"status":"Pending"}
```

В ASP.NET Core эта структура ровно отображена на `HttpContext`:

| HTTP | Код |
|---|---|
| Метод | `HttpContext.Request.Method` |
| Путь | `HttpContext.Request.Path` |
| Query | `HttpContext.Request.Query` |
| Заголовки запроса | `HttpContext.Request.Headers` |
| Тело запроса | `HttpContext.Request.Body` (`Stream`) |
| Код ответа | `HttpContext.Response.StatusCode` |
| Заголовки ответа | `HttpContext.Response.Headers` |
| Тело ответа | `HttpContext.Response.Body` / `BodyWriter` |

## Методы: безопасность и идемпотентность

Два независимых свойства, которые постоянно путают.

- **Безопасный (safe)** — не меняет состояние на сервере. Читающий метод.
- **Идемпотентный (idempotent)** — повторное выполнение того же запроса даёт тот же результат состояния. Один раз или пять раз — состояние одинаково.

| Метод | Безопасный | Идемпотентный | Тело в запросе | Кешируемый | Назначение |
|---|---|---|---|---|---|
| `GET` | да | да | нет (по спеке — не имеет смысла) | да | получить представление |
| `HEAD` | да | да | нет | да | только заголовки `GET` |
| `OPTIONS` | да | да | нет | нет | возможности ресурса, CORS preflight |
| `POST` | **нет** | **нет** | да | почти нет | создать/выполнить действие |
| `PUT` | нет | **да** | да | нет | заменить представление целиком |
| `PATCH` | нет | **нет**\* | да | нет | частично изменить |
| `DELETE` | нет | **да** | опционально | нет | удалить |

\* `PATCH` может быть сделан идемпотентным (JSON Merge Patch с абсолютными значениями — идемпотентен; `JSON Patch` с операцией `add` в массив — нет).

> [!info] Почему `DELETE` идемпотентен, если второй раз он вернёт 404
> Идемпотентность — про **состояние сервера**, а не про код ответа. После первого `DELETE /orders/5` ресурса нет; после второго его тоже нет. Состояние одинаково — метод идемпотентен. Разные коды ответа этому не противоречат.

Практический вывод: **retry можно делать автоматически только для идемпотентных методов**. Библиотеки вроде Polly по умолчанию так и настраивают (см. [[Устойчивость: retry, circuit breaker, Polly]]). Для `POST` нужен ключ идемпотентности — см. [[Идемпотентность]].

## Коды ответа

Первая цифра задаёт класс, и именно её обычно проверяет инфраструктура.

### 1xx — информационные

- `100 Continue` — сервер готов принять тело (клиент послал `Expect: 100-continue`).
- `101 Switching Protocols` — переход на WebSocket ([[SignalR]]).

### 2xx — успех

| Код | Когда |
|---|---|
| `200 OK` | обычный успешный ответ с телом |
| `201 Created` | создан ресурс. **Обязателен** заголовок `Location` |
| `202 Accepted` | принято в обработку, результата ещё нет (асинхронная операция) |
| `204 No Content` | успех без тела. Типично для `DELETE` и `PUT` |
| `206 Partial Content` | ответ на `Range` (докачка, видео) |

### 3xx — перенаправления и кеш

| Код | Когда |
|---|---|
| `301 Moved Permanently` | ресурс переехал навсегда. Браузеры кешируют агрессивно и надолго |
| `302 Found` | временно. Метод может быть изменён на `GET` (историческая практика) |
| `307 Temporary Redirect` | временно, метод и тело **сохраняются** |
| `308 Permanent Redirect` | навсегда, метод сохраняется |
| `304 Not Modified` | у клиента актуальная версия. Тела нет — экономия трафика |

### 4xx — ошибка клиента

| Код | Когда | Частая путаница |
|---|---|---|
| `400 Bad Request` | синтаксически/семантически невалидный запрос | не используйте как «всё плохо» |
| `401 Unauthorized` | **не аутентифицирован** (кто ты?) | название кода врёт |
| `403 Forbidden` | аутентифицирован, но **нет прав** | |
| `404 Not Found` | ресурса нет (или скрываем его существование) | |
| `405 Method Not Allowed` | путь есть, метод не тот. Обязателен `Allow` | |
| `406 Not Acceptable` | не можем отдать в запрошенном `Accept` | |
| `409 Conflict` | конфликт состояния (дубликат, версия) | |
| `410 Gone` | был, удалён навсегда | |
| `412 Precondition Failed` | не выполнено `If-Match`/`If-Unmodified-Since` | оптимистичная блокировка |
| `415 Unsupported Media Type` | не тот `Content-Type` | |
| `422 Unprocessable Content` | синтаксис верный, бизнес-правила нарушены | многие API отдают 400, и это тоже нормально |
| `428 Precondition Required` | требуем `If-Match` | |
| `429 Too Many Requests` | rate limit. Обязателен `Retry-After` | [[Rate limiting]] |

### 5xx — ошибка сервера

| Код | Когда |
|---|---|
| `500 Internal Server Error` | необработанное исключение |
| `501 Not Implemented` | метод не поддерживается сервером в принципе |
| `502 Bad Gateway` | прокси получил мусор от upstream |
| `503 Service Unavailable` | временно недоступен (деплой, перегрузка). `Retry-After` |
| `504 Gateway Timeout` | upstream не ответил вовремя |

> [!danger] Никогда не отдавайте 200 с телом `{"error": "..."}`
> Это ломает всё: клиентские SDK, retry-политики, метрики, алерты, кеши. Половина инфраструктуры смотрит только на статус-код. Ошибка — это 4xx/5xx плюс тело в формате ProblemDetails, см. [[Обработка ошибок и ProblemDetails]].

## Ключевые заголовки

### Согласование содержимого (content negotiation)

| Заголовок | Направление | Смысл |
|---|---|---|
| `Content-Type` | оба | формат тела: `application/json; charset=utf-8` |
| `Accept` | запрос | какие форматы клиент понимает: `application/json, */*;q=0.8` |
| `Accept-Encoding` | запрос | `gzip, br, zstd` |
| `Content-Encoding` | ответ | чем реально сжали |
| `Accept-Language` | запрос | язык. Связано с [[Культура, локализация и форматирование]] |
| `Vary` | ответ | по каким заголовкам запроса ответ различается — **критично для кешей** |

`q=` — «quality value», вес предпочтения от 0 до 1. `Accept: application/json;q=1.0, text/xml;q=0.5` означает «лучше JSON, XML тоже приму».

### Условные запросы и валидаторы

| Заголовок | Смысл |
|---|---|
| `ETag` | версия представления: `"a3f9c1"` (сильный) или `W/"a3f9c1"` (слабый) |
| `Last-Modified` | дата изменения, точность до секунды |
| `If-None-Match` | «отдай, только если ETag НЕ совпадает» → иначе `304` |
| `If-Modified-Since` | то же по дате → иначе `304` |
| `If-Match` | «изменяй, только если ETag совпадает» → иначе `412` |
| `If-Unmodified-Since` | то же по дате |

Разница сильного и слабого ETag: **сильный** гарантирует побайтовую идентичность, **слабый** (`W/`) — только семантическую эквивалентность. Слабый нельзя использовать для `Range`-запросов.

### Кеширование: `Cache-Control`

Директивы ответа:

| Директива | Смысл |
|---|---|
| `no-store` | не сохранять нигде вообще. Для персональных данных и платежей |
| `no-cache` | сохранять можно, но перед выдачей **обязательно перепроверить** на сервере |
| `private` | кешировать может только браузер, но не CDN/прокси |
| `public` | может кешировать любой промежуточный кеш |
| `max-age=600` | свежий 600 секунд |
| `s-maxage=60` | то же, но для shared-кешей (CDN); переопределяет `max-age` |
| `must-revalidate` | после истечения нельзя отдавать stale |
| `stale-while-revalidate=30` | можно отдать протухший, обновляя в фоне |
| `immutable` | никогда не перепроверять (для файлов с хешем в имени) |

> [!warning] `no-cache` ≠ «не кешировать»
> `no-cache` означает «кешируй, но всегда валидируй». «Не кешируй» — это `no-store`. Ошибка стоит утечки данных: `Cache-Control: no-cache` на ответе с чужими персональными данными позволяет CDN держать копию.

Для API по умолчанию правильно так:

```csharp
// Ответ, зависящий от пользователя, кешировать нельзя
app.MapGet("/api/me", (ClaimsPrincipal user) =>
{
    return TypedResults.Ok(new { Name = user.Identity!.Name });
})
.RequireAuthorization();
// Middleware, который проставит заголовки для всех приватных ответов:
```

```csharp
app.Use(async (ctx, next) =>
{
    ctx.Response.OnStarting(() =>
    {
        // Если эндпоинт сам ничего не сказал про кеш — запрещаем
        if (!ctx.Response.Headers.ContainsKey("Cache-Control"))
            ctx.Response.Headers.CacheControl = "no-store";
        return Task.CompletedTask;
    });
    await next();
});
```

### ETag на практике

```csharp
using System.Globalization;
using Microsoft.Net.Http.Headers;

app.MapGet("/api/products/{id:int}", async (int id, IProductService svc, HttpContext ctx) =>
{
    var product = await svc.GetAsync(id);
    if (product is null) return Results.NotFound();

    // ETag считаем от версии/RowVersion, а не от всего JSON — так дешевле
    var etag = new EntityTagHeaderValue($"\"{Convert.ToBase64String(product.Version)}\"");

    var ifNoneMatch = ctx.Request.GetTypedHeaders().IfNoneMatch;
    if (ifNoneMatch.Any(t => t.Compare(etag, useStrongComparison: false)))
        return Results.StatusCode(StatusCodes.Status304NotModified);

    ctx.Response.GetTypedHeaders().ETag = etag;
    ctx.Response.Headers.CacheControl = "private, max-age=0, must-revalidate";
    return Results.Ok(product);
});
```

Оптимистичная блокировка на запись — тот же ETag, но через `If-Match`:

```csharp
app.MapPut("/api/products/{id:int}", async (
    int id, ProductDto dto, IProductService svc, HttpContext ctx) =>
{
    var ifMatch = ctx.Request.GetTypedHeaders().IfMatch;
    if (ifMatch.Count == 0)
        return Results.StatusCode(StatusCodes.Status428PreconditionRequired);

    var expectedVersion = Convert.FromBase64String(ifMatch[0].Tag.Value!.Trim('"'));

    var updated = await svc.UpdateAsync(id, dto, expectedVersion);
    return updated
        ? Results.NoContent()
        : Results.StatusCode(StatusCodes.Status412PreconditionFailed);
});
```

Это тот же механизм, что `RowVersion` в EF Core — см. [[EF Core: транзакции и конкурентность]].

### `Vary` — заголовок, о котором забывают

Если ответ зависит от `Accept-Language` или `Authorization`, а вы не указали `Vary`, CDN отдаст ответ одного пользователя другому.

```csharp
ctx.Response.Headers.Vary = "Accept-Encoding, Accept-Language";
```

Правило: **любой заголовок запроса, который влияет на тело ответа, должен быть в `Vary`** — либо ответ должен быть `private`/`no-store`.

## HTTP/1.1 vs HTTP/2 vs HTTP/3

| | HTTP/1.1 (1997) | HTTP/2 (2015) | HTTP/3 (2022) |
|---|---|---|---|
| Транспорт | TCP | TCP | **QUIC поверх UDP** |
| Формат | текст | бинарные фреймы | бинарные фреймы |
| Мультиплексирование | нет (или pipelining, который не работает) | да, стримы в одном соединении | да |
| Head-of-line blocking | на уровне запросов | устранён на уровне HTTP, **остался на уровне TCP** | устранён полностью |
| Сжатие заголовков | нет | HPACK | QPACK |
| Server push | нет | был, **устарел и выпилен** | нет |
| TLS | опционален | де-факто обязателен | встроен в QUIC |
| Установка соединения | TCP (1 RTT) + TLS (1–2 RTT) | то же | 1 RTT, 0-RTT при возобновлении |

Практические следствия:

- В HTTP/2 нет смысла в domain sharding и склейке спрайтов — старые «оптимизации» стали вредны.
- HTTP/2 требует ALPN, то есть TLS. Открытый HTTP/2 (h2c) возможен, но браузеры его не поддерживают — только для service-to-service (см. [[gRPC в .NET]], который поверх HTTP/2 работает по умолчанию).
- HTTP/3 отлично лечит мобильные сети с потерями пакетов: потеря в одном стриме не тормозит остальные.
- Заголовки в HTTP/2+ — **всегда в нижнем регистре**. Код, который сравнивает имя заголовка регистрозависимо, ломается.

Настройка в ASP.NET Core:

```csharp
builder.WebHost.ConfigureKestrel(options =>
{
    options.ConfigureEndpointDefaults(listen =>
    {
        // Http1AndHttp2AndHttp3 требует TLS для h2/h3
        listen.Protocols = HttpProtocols.Http1AndHttp2AndHttp3;
    });
});
```

```json
// appsettings.json — то же декларативно
{
  "Kestrel": {
    "EndpointDefaults": { "Protocols": "Http1AndHttp2" }
  }
}
```

Для HTTP/3 нужен ещё заголовок `Alt-Svc`, чтобы браузер узнал о поддержке — Kestrel добавляет его сам, если HTTP/3-эндпоинт включён. Подробности хостинга — в [[Kestrel, reverse proxy и хостинг]].

## Тело запроса: как оно реально читается

`HttpContext.Request.Body` — это **forward-only поток**. Прочитали один раз — второй раз ничего не получите:

```csharp
app.MapPost("/echo", async (HttpContext ctx) =>
{
    using var reader = new StreamReader(ctx.Request.Body);
    var first = await reader.ReadToEndAsync();
    ctx.Request.Body.Position = 0;   // ← NotSupportedException на нередактируемом потоке
    return first;
});
```

Чтобы читать дважды (например, логировать тело и потом биндить модель) — нужен `EnableBuffering()`, см. [[Middleware и конвейер обработки запроса]].

> [!warning] Chunked transfer и отсутствие `Content-Length`
> Если клиент отправляет `Transfer-Encoding: chunked`, `Content-Length` нет вовсе, и `Request.ContentLength` вернёт `null`. Код вида `if (ctx.Request.ContentLength > maxSize)` в этом случае молча пропускает запрос любого размера. Ограничивать нужно через `MaxRequestBodySize`, а не по заголовку.

> [!warning] Подводные камни
> - **`GET` с телом.** Формально спека не запрещает, но прокси, кеши и `HttpClient` могут его выбросить. Никогда не проектируйте API так — используйте `POST /search`.
> - **URL-длина.** Kestrel по умолчанию ограничивает стартовую строку 8 КБ (`MaxRequestLineSize`). Фильтр из 200 id в query упадёт с `414`. Такие запросы — `POST` с телом.
> - **`Location` относительный или абсолютный.** RFC 9110 разрешает оба, но за reverse proxy относительный безопаснее: абсолютный, собранный из `Request.Scheme`/`Host`, без `ForwardedHeaders` выдаст `http://internal-pod:8080/...`.
> - **`204 No Content` с телом.** Если вы записали в `Response.Body` при статусе 204, HTTP/1.1-клиент сойдёт с ума: он не ждёт тела и прочитает его как начало следующего ответа в keep-alive соединении.
> - **Регистр заголовков.** `Request.Headers["content-type"]` работает (словарь регистронезависимый), но собственные сравнения строк — нет. Используйте `StringComparison.OrdinalIgnoreCase`.

> [!example] Как делают в бою
> - Все ответы API — `Cache-Control: no-store` по умолчанию; кеширование включается точечно на конкретных публичных эндпоинтах через [[Кеширование: Output, Response, HybridCache]].
> - `ETag` ставится на «тяжёлые» справочники (списки категорий, конфигурация фронта) — мобильные клиенты получают `304` и экономят батарею.
> - `If-Match` обязателен на `PUT`/`PATCH` для сущностей, которые редактируют несколько пользователей.
> - Корреляция: входящий `traceparent` (W3C Trace Context) прокидывается дальше, см. [[OpenTelemetry в .NET]].
> - `429` всегда с `Retry-After`, `503` при деплое — тоже, иначе клиенты устроят thundering herd.

## Вопросы с собеседований

> [!question]- Чем отличаются идемпотентность и безопасность метода? Приведите метод, который идемпотентен, но не безопасен.
> Безопасный метод не меняет состояние сервера (`GET`, `HEAD`, `OPTIONS`). Идемпотентный — при повторе даёт то же состояние. `PUT` и `DELETE` идемпотентны, но не безопасны: они меняют состояние, но повтор ничего дополнительно не портит. `POST` не обладает ни одним из свойств. Практический смысл: автоматический retry (в Polly, в балансировщике, в браузере) допустим только для идемпотентных методов, иначе таймаут на `POST /payments` превратится в двойное списание.

> [!question]- Клиент послал `PUT` с `If-Match: "v1"`, а на сервере уже `"v2"`. Что вернуть и почему не 409?
> `412 Precondition Failed`. Код 412 говорит именно «предусловие, которое ты выразил заголовком, не выполнилось» — клиент знает, что нужно перечитать ресурс и повторить. `409 Conflict` уместен, когда конфликт не выражен через условный заголовок: например, попытка создать пользователя с уже занятым email. Различие важно для клиентских библиотек: на 412 они автоматически делают refetch, на 409 — показывают ошибку пользователю.

> [!question]- Что произойдёт, если отдать `Cache-Control: public, max-age=300` на ответ, зависящий от заголовка `Authorization`, и не указать `Vary`?
> CDN или корпоративный прокси закеширует ответ первого пользователя и в течение 300 секунд будет отдавать его всем остальным, включая неаутентифицированных. Это классическая утечка данных через кеш. Правильно: либо `Cache-Control: private, no-store` для персонализированных ответов, либо `Vary: Authorization` (что практически убивает эффективность shared-кеша, поэтому обычно выбирают первое).

> [!question]- Почему в HTTP/2 отпала необходимость склеивать CSS-файлы и раскладывать ресурсы по разным доменам?
> В HTTP/1.1 браузер держал 6 соединений на хост, и каждый запрос в соединении ждал завершения предыдущего (head-of-line blocking). Отсюда трюки: sharding по доменам ради большего числа соединений и склейка файлов ради меньшего числа запросов. HTTP/2 мультиплексирует произвольное число стримов в одном TCP-соединении и сжимает заголовки HPACK, так что 50 мелких запросов дешевле одного огромного бандла (лучше кешируются по отдельности). Sharding в HTTP/2 прямо вреден: он заставляет открывать лишние TCP+TLS-соединения.

> [!question]- В чём принципиальная разница между HTTP/2 и HTTP/3, если оба мультиплексируют?
> HTTP/2 мультиплексирует на уровне HTTP, но лежит на TCP, где потеря одного сегмента останавливает доставку всех байтов после него — head-of-line blocking просто переехал на транспортный уровень. HTTP/3 работает поверх QUIC (UDP), где каждый стрим имеет независимое управление потерями: потеря пакета в стриме A не блокирует стрим B. Плюс QUIC совмещает установку соединения и TLS-хендшейк (1 RTT вместо 2–3) и умеет 0-RTT при переподключении, а также переживает смену IP (миграция соединения) — важно для мобильных клиентов.

> [!question]- Почему `Request.ContentLength` — ненадёжный способ ограничить размер загрузки?
> При `Transfer-Encoding: chunked` заголовка `Content-Length` вообще нет, и свойство равно `null`. Сравнение `ContentLength > limit` даст `false` и пропустит запрос любого объёма. Кроме того, клиент может соврать в `Content-Length`. Ограничивать нужно на уровне сервера: `KestrelServerLimits.MaxRequestBodySize`, атрибут/метаданные `RequestSizeLimit` или `IHttpMaxRequestBodySizeFeature` — тогда Kestrel оборвёт соединение по факту прочитанных байтов. См. [[Загрузка файлов и работа с формами]].

## Задачи

### Задача 1. Условный GET
Напишите Minimal API эндпоинт `GET /api/config`, который отдаёт объект конфигурации и корректно поддерживает `If-None-Match`, возвращая `304` без тела, если версия не изменилась. Версию считайте как SHA-256 от сериализованного JSON.

> [!success]- Решение
> ```csharp
> using System.Security.Cryptography;
> using System.Text.Json;
> using Microsoft.Net.Http.Headers;
>
> app.MapGet("/api/config", (IConfigProvider provider, HttpContext ctx) =>
> {
>     var config = provider.GetCurrent();
>     var json = JsonSerializer.SerializeToUtf8Bytes(config);
>     var hash = Convert.ToHexString(SHA256.HashData(json))[..16];
>     var etag = new EntityTagHeaderValue($"\"{hash}\"");
>
>     var typed = ctx.Request.GetTypedHeaders();
>     if (typed.IfNoneMatch.Any(t => t.Compare(etag, useStrongComparison: true)))
>         return Results.StatusCode(StatusCodes.Status304NotModified);
>
>     ctx.Response.GetTypedHeaders().ETag = etag;
>     ctx.Response.Headers.CacheControl = "public, max-age=0, must-revalidate";
>     return Results.Ok(config);
> });
> ```
> Ключевые моменты: ETag считаем от **байтов сериализованного** представления (иначе разный порядок свойств даст разные хеши при одинаковом смысле); `must-revalidate` с `max-age=0` заставляет клиента каждый раз спрашивать, но ответ в 99 % случаев будет пустой `304`. Тело при `304` отдавать нельзя — спека запрещает.

### Задача 2. Правильные коды
Для каждой ситуации укажите код ответа и обязательные заголовки:
1. Создали заказ.
2. Токен просрочен.
3. Пользователь не админ, а эндпоинт только для админов.
4. Клиент прислал XML, а вы принимаете только JSON.
5. Клиент превысил квоту 100 запросов в минуту.
6. Успешно удалили ресурс.
7. Обновление отклонено из-за конкурентного изменения (клиент прислал `If-Match`).

> [!success]- Решение
> 1. `201 Created` + `Location: /api/orders/{id}`. Тело — созданный ресурс (необязательно, но удобно).
> 2. `401 Unauthorized` + `WWW-Authenticate: Bearer error="invalid_token"`. Именно 401, не 403.
> 3. `403 Forbidden`. Тело — ProblemDetails без деталей о том, каких именно прав не хватает (иначе это подсказка атакующему).
> 4. `415 Unsupported Media Type` + `Accept: application/json` в ответе.
> 5. `429 Too Many Requests` + `Retry-After: 37` (секунды или дата). Полезно добавить `RateLimit-*` заголовки.
> 6. `204 No Content`, тела нет. Если удаление асинхронное — `202 Accepted` + `Location` на статус операции.
> 7. `412 Precondition Failed`. Не 409: клиент явно выразил предусловие через `If-Match`.

### Задача 3. Найдите утечку
Ревью кода. Что здесь опасно и как исправить?
```csharp
app.MapGet("/api/dashboard", async (ClaimsPrincipal user, IDashboardService svc) =>
{
    var data = await svc.GetForUserAsync(user.GetUserId());
    return Results.Ok(data);
})
.RequireAuthorization()
.CacheOutput(p => p.Expire(TimeSpan.FromMinutes(5)));
```

> [!success]- Решение
> `CacheOutput` по умолчанию кеширует по URL и **не различает пользователей**: первый вошедший пользователь наполнит кеш своим дашбордом, остальные получат его данные. Output caching в ASP.NET Core по умолчанию вообще не кеширует ответы на аутентифицированные запросы, но это поведение легко случайно ослабить, а на уровне CDN — сломать заголовками.
>
> Правильно — либо не кешировать:
> ```csharp
> app.MapGet("/api/dashboard", /* ... */)
>    .RequireAuthorization();
> // + Cache-Control: no-store глобальным middleware
> ```
> Либо кешировать с явным разделением по пользователю и понимая риски:
> ```csharp
> .CacheOutput(p => p
>     .Expire(TimeSpan.FromMinutes(5))
>     .SetVaryByRouteValue()          // нет — не поможет
>     .VaryByValue(ctx => new KeyValuePair<string, string>(
>         "uid", ctx.User.GetUserId().ToString())));
> ```
> Но для персональных данных практика такая: кешировать на клиенте (`private, max-age=60`), а не на сервере. См. [[Кеширование: Output, Response, HybridCache]].

## Итог

- Метод + URI + заголовки + тело — всё, что есть в HTTP; `HttpContext` отображает это один-в-один.
- Безопасность и идемпотентность — разные свойства. Автоматический retry допустим только для идемпотентных методов; `POST` требует ключа идемпотентности.
- `401` — «не знаю кто ты», `403` — «знаю, но нельзя», `412` — «твоё предусловие не выполнилось», `409` — «конфликт состояния».
- `no-cache` значит «кешируй и валидируй», `no-store` значит «не кешируй». Путаница даёт утечку данных.
- Любой заголовок запроса, влияющий на тело ответа, должен попасть в `Vary` — либо ответ должен быть `private`/`no-store`.
- HTTP/2 убрал head-of-line blocking на уровне HTTP, HTTP/3 (QUIC/UDP) — на уровне транспорта; старые оптимизации HTTP/1.1 в HTTP/2 вредны.

## Связанное

- [[REST: принципы и проектирование API]]
- [[Шпаргалка — HTTP коды и заголовки]]
- [[Кеширование: Output, Response, HybridCache]]
- [[Обработка ошибок и ProblemDetails]]
- [[Kestrel, reverse proxy и хостинг]]
- [[Идемпотентность]]
- [[Rate limiting]]
- [[HttpClient и IHttpClientFactory]]
