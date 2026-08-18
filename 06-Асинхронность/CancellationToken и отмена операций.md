---
tags: [раздел-06, основы, собес, async]
aliases: [CancellationToken, CancellationTokenSource, Отмена операций, Таймауты, RequestAborted]
---

# CancellationToken и отмена операций

> [!abstract] Коротко
> Отмена в .NET **кооперативная**: никто не может прервать чужой код принудительно — операция сама периодически проверяет токен и прекращает работу. Источник (`CancellationTokenSource`) владеет отменой и раздаёт «только для чтения» токены (`CancellationToken`), которые передаются вниз по стеку вызовов. Отмена — не ошибка, а штатный исход: она выражается исключением `OperationCanceledException`, которое логируют на уровне информации и не считают сбоем. Правило, из которого растёт всё остальное: **каждый асинхронный метод принимает токен и передаёт его дальше**, вплоть до драйвера БД и HTTP-клиента.

## Зачем это нужно

Три практических повода. Первый: клиент разорвал HTTP-соединение — продолжать выполнять запрос бессмысленно, но без токена сервер честно доработает и сходит в базу. Второй: приложение останавливается (`SIGTERM` в Kubernetes), и фоновые задачи должны завершиться за grace-период, иначе процесс убьют жёстко. Третий: таймауты — операция, которая может висеть вечно, обязана иметь ограничение.

---

## Модель

```csharp
using var cts = new CancellationTokenSource();     // владелец отмены
CancellationToken token = cts.Token;                // «только чтение», передаётся вниз

var task = WorkAsync(token);
cts.Cancel();                                       // сигнал отмены (или await cts.CancelAsync() в .NET 8+)

static async Task WorkAsync(CancellationToken ct)
{
    for (int i = 0; i < 1000; i++)
    {
        ct.ThrowIfCancellationRequested();          // проверка в цикле
        await Task.Delay(100, ct);                   // токен проброшен в ожидание
    }
}
```

Разделение ролей принципиально: `CancellationTokenSource` — у того, кто **инициирует** отмену (хост, контроллер, вызывающий код); `CancellationToken` — у того, кто **реагирует**. Метод, принимающий токен, отменять его не может — только проверять.

Способы отреагировать:

```csharp
ct.ThrowIfCancellationRequested();          // основной: бросить OperationCanceledException
if (ct.IsCancellationRequested) return;      // тихий выход, когда исключение не нужно
await Task.Delay(1000, ct);                  // библиотечные методы сами бросают
ct.Register(() => connection.Abort());       // колбэк для нештатных ресурсов
```

`ThrowIfCancellationRequested` предпочтительнее тихого выхода: вызывающий должен отличать «работа выполнена» от «работа прервана», а молчаливый `return` эту разницу стирает.

---

## Источники токенов в приложении

```csharp
// 1. HTTP-запрос: клиент отключился
[HttpGet("{id:long}")]
public async Task<IResult> Get(long id, CancellationToken ct)   // биндится автоматически
{
    var order = await _service.GetAsync(id, ct);
    return order is null ? Results.NotFound() : Results.Ok(order);
}
// эквивалент: HttpContext.RequestAborted

// 2. Остановка приложения
public sealed class Worker(IHostApplicationLifetime lifetime) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await ProcessBatchAsync(stoppingToken);
            await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
        }
    }
}

// 3. Таймаут
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(30));
await LongOperationAsync(cts.Token);

// 4. Отсутствие отмены — явно
await DoAsync(CancellationToken.None);
```

В ASP.NET Core параметр типа `CancellationToken` в методе действия связывается автоматически с `HttpContext.RequestAborted` — его достаточно объявить. Это самый дешёвый способ не тратить ресурсы на запросы, которые никто уже не ждёт.

### Связанные токены

```csharp
// Отмена, если сработало ЛЮБОЕ из условий: запрос прерван, остановка приложения или таймаут
using var timeout = new CancellationTokenSource(TimeSpan.FromSeconds(10));
using var linked = CancellationTokenSource.CreateLinkedTokenSource(
    requestAborted, stoppingToken, timeout.Token);

await CallExternalApiAsync(linked.Token);
```

Связанный источник — стандартный приём для «таймаут поверх внешней отмены». Важно освобождать его (`using`): он подписывается на родительские токены, и без освобождения подписки накапливаются — это реальная утечка в долгоживущих сервисах ([[Слабые ссылки и утечки памяти в .NET]]).

Различить причину отмены помогает проверка конкретного токена:

```csharp
catch (OperationCanceledException) when (timeout.IsCancellationRequested)
{
    throw new TimeoutException("Внешний сервис не ответил за 10 секунд");
}
catch (OperationCanceledException)
{
    throw;      // отмена пользователем или остановка приложения — пробрасываем как есть
}
```

---

## Отмена — не ошибка

```csharp
try
{
    await ProcessAsync(ct);
}
catch (OperationCanceledException) when (ct.IsCancellationRequested)
{
    logger.LogInformation("Обработка отменена");     // Information, не Error
    throw;                                            // обычно пробрасываем дальше
}
```

Три правила:

1. **Не логировать как ошибку.** Иначе мониторинг заполнится ложными сбоями при каждом обновлении сервиса или закрытии вкладки браузером.
2. **Не «глотать».** Тихий `catch (OperationCanceledException) { }` превращает прерванную работу в успешную — вызывающий решит, что всё выполнено.
3. **Различать отмену и таймаут.** Если токен отменил ты сам по истечении времени, наружу правильнее отдать `TimeoutException`: для вызывающего это разные ситуации.

`TaskCanceledException` — наследник `OperationCanceledException`, поэтому ловить достаточно базовый тип ([[Обработка исключений]]).

---

## Проектирование API

```csharp
// Токен — последний параметр, без значения по умолчанию во внутреннем коде
public Task<Order> GetAsync(long id, CancellationToken ct);

// Значение по умолчанию допустимо в публичном API библиотеки
public Task<Order> GetAsync(long id, CancellationToken cancellationToken = default);
```

Соглашения платформы: имя `cancellationToken` (или `ct` во внутреннем коде — по договорённости команды), последний параметр, обязательная передача дальше во все асинхронные вызовы. Отсутствие значения по умолчанию во внутренних методах — полезная строгость: забыть токен становится невозможно.

```csharp
// Токен проходит сквозь весь стек — до самых нижних вызовов
public async Task<Order?> GetAsync(long id, CancellationToken ct)
{
    var cached = await _cache.GetAsync(id, ct);
    if (cached is not null) return cached;

    var order = await _db.Orders.FirstOrDefaultAsync(o => o.Id == id, ct);
    if (order is not null) await _cache.SetAsync(id, order, ct);
    return order;
}
```

EF Core, `HttpClient`, `Stream`, `Channel`, `SemaphoreSlim` — все принимают токен. Если метод его не принимает, отменить операцию невозможно, и это ограничение библиотеки, а не стиля.

---

## Отмена того, что не умеет отменяться

```csharp
// Синхронный или неотменяемый вызов: отменить нельзя, но можно перестать ждать
static async Task<T> WithTimeout<T>(Task<T> task, TimeSpan timeout, CancellationToken ct)
{
    using var timeoutCts = CancellationTokenSource.CreateLinkedTokenSource(ct);
    timeoutCts.CancelAfter(timeout);

    var completed = await Task.WhenAny(task, Task.Delay(Timeout.Infinite, timeoutCts.Token));
    if (completed != task) throw new TimeoutException();

    timeoutCts.Cancel();          // остановить таймер
    return await task;             // пробросить результат или исключение
}
```

Важно понимать ограничение: исходная операция **продолжит выполняться** в фоне — мы лишь перестали её ждать. Ресурсы (соединение, поток) остаются занятыми, поэтому такой приём допустим как временная мера, а правильное решение — использовать API с поддержкой отмены.

С .NET 8 у `Task` есть встроенный вариант ожидания с токеном:

```csharp
await task.WaitAsync(TimeSpan.FromSeconds(5), ct);   // бросит TimeoutException или OperationCanceledException
```

---

## Тонкости `CancellationTokenSource`

```csharp
var cts = new CancellationTokenSource();
cts.CancelAfter(TimeSpan.FromSeconds(30));      // отложенная отмена по таймеру
await cts.CancelAsync();                         // .NET 8: асинхронная отмена, дожидается колбэков
cts.Dispose();                                    // ОБЯЗАТЕЛЬНО: держит таймер и регистрации

// Переиспользование вместо создания нового (.NET 6+)
if (cts.TryReset()) { /* можно использовать снова */ }
```

- **Освобождать обязательно.** `CancellationTokenSource` содержит таймер и список регистраций; в цикле без `Dispose` это утечка.
- **`Cancel()` выполняет колбэки синхронно** в вызывающем потоке; исключение из колбэка попадёт в вызывающего. `CancelAsync` (.NET 8) решает это.
- **Токен одноразовый**: после отмены он остаётся отменённым навсегда, для новой операции нужен новый источник (или `TryReset`).
- **`ct.Register` возвращает `IDisposable`** — регистрацию надо освобождать, если токен живёт дольше подписки.

> [!warning] Подводные камни
> - **Токен принят, но не передан дальше.** Метод «поддерживает отмену» формально, а запрос к БД её игнорирует.
> - **`catch (OperationCanceledException) { }` вслепую.** Прерванная работа выглядит как успешная.
> - **Логирование отмены как ошибки.** Мониторинг заполняется ложными сбоями при каждом деплое.
> - **Отсутствие `Dispose` у `CancellationTokenSource`.** Утечка таймеров и регистраций в долгоживущих сервисах.
> - **Связанный источник без освобождения.** Подписка на родительский токен остаётся навсегда.
> - **Бесконечный цикл без проверки токена.** Фоновая служба не даёт приложению остановиться, и её убивают по истечении grace-периода.
> - **`CancellationToken.None` «чтобы не мешало».** Явный отказ от отмены должен быть осознанным решением, а не привычкой.
> - **Отмена как способ прервать вычисление.** Кооперативная модель не прерывает код: без проверок внутри цикла отмена ничего не даст.

> [!example] Как делают в бою
> В веб-сервисе токен приходит из `RequestAborted` (достаточно объявить параметр в методе действия) и проходит сквозь весь стек: сервис → репозиторий → EF Core / HTTP-клиент. Это даёт немедленное освобождение соединений с БД, когда клиент отвалился, — заметная экономия под нагрузкой.
> Фоновые службы строятся вокруг `stoppingToken`: цикл проверяет его в условии, все ожидания принимают его, обработка одного элемента оборачивается в try/catch, где отмена логируется на уровне Information и приводит к корректному выходу. Это то, что позволяет поду в Kubernetes завершиться штатно за grace-период, а не быть убитым `SIGKILL`.
> Для вызовов внешних сервисов делают связанный источник с таймаутом поверх токена запроса — тогда медленный внешний API не удерживает ресурсы дольше положенного, и при этом отмена клиентом тоже срабатывает.
> На ревью проверяют три вещи: токен принимается и передаётся дальше, отмена не логируется как ошибка, `CancellationTokenSource` освобождается. Анализаторы ловят часть случаев (передача `CancellationToken.None` в методы, принимающие токен).

---

## Вопросы с собеседований

> [!question]- Как устроена отмена в .NET и почему она кооперативная?
> Отмена реализована как сигнал, а не как принудительное прерывание: `CancellationTokenSource` владеет состоянием и раздаёт токены; операция периодически проверяет токен (`ThrowIfCancellationRequested`, `IsCancellationRequested`) или передаёт его в библиотечные методы, которые делают это сами. Принудительного прерывания в платформе нет намеренно — устаревший `Thread.Abort` мог остановить поток в произвольной точке, оставив данные в несогласованном состоянии, блокировки захваченными, а ресурсы неосвобождёнными; в .NET Core он вообще выбрасывает `PlatformNotSupportedException`. Кооперативная модель гарантирует, что код прерывается в известной точке и может корректно освободить ресурсы. Плата — дисциплина: метод, который не проверяет токен и не передаёт его дальше, отменить невозможно.

> [!question]- Как правильно обрабатывать `OperationCanceledException`?
> Считать отмену штатным исходом, а не сбоем. Практически это означает: логировать на уровне Information (или не логировать вовсе), не превращать в ошибку в мониторинге и, как правило, пробрасывать дальше, чтобы вызывающий тоже узнал о прерывании. Тихо проглатывать нельзя — тогда прерванная работа выглядит успешной. Полезно различать причины через фильтр исключений: если отменён «наш» таймаутный источник, наружу правильнее отдать `TimeoutException`, потому что для вызывающего таймаут и отмена клиентом — разные ситуации; если отменён внешний токен, исключение пробрасывается как есть. Ловить достаточно `OperationCanceledException`: `TaskCanceledException` является его наследником. И отдельная деталь для веб-приложений: отмена по `RequestAborted` — обычное дело (клиент закрыл вкладку), и заполнять ею логи ошибок вредно.

> [!question]- Что будет, если фоновая служба игнорирует `stoppingToken`?
> Приложение не сможет остановиться штатно. При получении `SIGTERM` хост инициирует остановку: отменяет `stoppingToken` и ждёт завершения фоновых служб в пределах таймаута (по умолчанию 30 секунд, настраивается через `HostOptions.ShutdownTimeout`). Служба, чей цикл не проверяет токен и чьи ожидания его не принимают, продолжит работать; по истечении таймаута хост завершит процесс, а оркестратор пришлёт `SIGKILL`. Последствия — прерванные посередине операции, незакоммиченные транзакции, потерянные сообщения, неотправленные метрики. Правильная реализация: условие цикла проверяет `IsCancellationRequested`, все асинхронные вызовы принимают токен, обработка одного элемента защищена try/catch, а отмена приводит к корректному выходу из метода. Для долгих операций внутри итерации добавляют промежуточные проверки токена.

---

## Задачи

### Задача 1. Добавить отмену и таймаут

```csharp
public async Task<Report> BuildAsync(long id)
{
    var order = await _db.Orders.FirstAsync(o => o.Id == id);
    var rates = await _http.GetFromJsonAsync<Rates>("https://api.rates/latest");
    return Render(order, rates);
}
```

> [!success]- Решение
> ```csharp
> public async Task<Report> BuildAsync(long id, CancellationToken ct)
> {
>     var order = await _db.Orders.FirstAsync(o => o.Id == id, ct);
>
>     using var timeout = CancellationTokenSource.CreateLinkedTokenSource(ct);
>     timeout.CancelAfter(TimeSpan.FromSeconds(5));
>
>     Rates rates;
>     try
>     {
>         rates = await _http.GetFromJsonAsync<Rates>("https://api.rates/latest", timeout.Token)
>                 ?? throw new InvalidOperationException("Пустой ответ сервиса курсов");
>     }
>     catch (OperationCanceledException) when (!ct.IsCancellationRequested)
>     {
>         throw new TimeoutException("Сервис курсов не ответил за 5 секунд");
>     }
>
>     return Render(order, rates);
> }
> ```
> Разбор: токен принимается и передаётся во все асинхронные вызовы, включая EF Core. Для внешнего API создан связанный источник с таймаутом — он отменится и при истечении времени, и при отмене исходного токена. Фильтр `when (!ct.IsCancellationRequested)` различает причины: если внешний токен не отменён, значит сработал наш таймаут, и наружу отдаётся понятный `TimeoutException`; иначе отмена пробрасывается как есть. Связанный источник обёрнут в `using`, иначе подписка на родительский токен останется висеть.

### Задача 2. Найти ошибки в фоновой службе

```csharp
public class CleanupService : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (true)
        {
            try
            {
                await CleanupAsync();
                await Task.Delay(TimeSpan.FromMinutes(10));
            }
            catch (Exception ex) { _logger.LogError(ex, "Ошибка очистки"); }
        }
    }
}
```

> [!success]- Решение
> Три ошибки. Цикл `while (true)` не проверяет токен, `Task.Delay` его не принимает, `CleanupAsync` его не получает — служба не остановится по `SIGTERM`, и через таймаут завершения процесс убьют жёстко. Плюс перехват `Exception` без исключения отмены: при остановке в лог попадёт ложная ошибка.
> ```csharp
> public class CleanupService(ILogger<CleanupService> logger) : BackgroundService
> {
>     protected override async Task ExecuteAsync(CancellationToken stoppingToken)
>     {
>         while (!stoppingToken.IsCancellationRequested)
>         {
>             try
>             {
>                 await CleanupAsync(stoppingToken);
>                 await Task.Delay(TimeSpan.FromMinutes(10), stoppingToken);
>             }
>             catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested)
>             {
>                 logger.LogInformation("Служба очистки останавливается");
>                 break;                                   // штатный выход
>             }
>             catch (Exception ex)
>             {
>                 logger.LogError(ex, "Ошибка очистки, продолжаем");
>                 await Task.Delay(TimeSpan.FromMinutes(1), stoppingToken);   // пауза перед повтором
>             }
>         }
>     }
> }
> ```
> Ключевые изменения: токен в условии цикла и во всех ожиданиях; отмена обрабатывается отдельным `catch` с уровнем Information и выходом из цикла; прочие ошибки не останавливают службу, но добавляют паузу, чтобы не крутить цикл с ошибкой в бесконечном темпе. Именно такая структура позволяет поду завершиться в пределах grace-периода.

---

## Итог

- Отмена кооперативная: сигнал получает тот, кто сам его проверяет; принудительного прерывания нет.
- `CancellationTokenSource` владеет отменой, `CancellationToken` — только читает; не путать роли.
- Каждый асинхронный метод принимает токен последним параметром и передаёт его дальше.
- Источники в приложении: `RequestAborted` для HTTP, `stoppingToken` для фоновых служб, таймауты через `CancelAfter`.
- Связанные источники объединяют условия отмены; их обязательно освобождать.
- Отмена — штатный исход: не логировать как ошибку, не проглатывать, отличать от таймаута.
- `CancellationTokenSource` требует `Dispose`: он держит таймер и регистрации.
- Ожидание с таймаутом (`WaitAsync`, `WhenAny`) не останавливает саму операцию — она продолжит выполняться.

## Связанное

- [[async и await: как это работает на самом деле]] · [[Task и Task Parallel Library]]
- [[Исключения в асинхронном коде]] — `OperationCanceledException` среди прочих
- [[Background services и IHostedService]] — `stoppingToken` и корректная остановка
- [[Дедлоки, async void и типичные ошибки]] · [[Parallel, PLINQ и параллелизм данных]]
- [[Устойчивость: retry, circuit breaker, Polly]] — таймауты и повторы вокруг внешних вызовов
- [[Командная строка Linux для .NET-разработчика]] — `SIGTERM` и grace-период
