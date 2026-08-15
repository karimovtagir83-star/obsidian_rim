---
tags: [раздел-06, middle, async, dotnet10]
aliases: [Channels, Channel, Producer-consumer, Очередь в памяти, Backpressure]
---

# System.Threading.Channels

> [!abstract] Коротко
> `Channel<T>` — асинхронная очередь между производителями и потребителями внутри процесса. Она решает ту же задачу, что `BlockingCollection`, но не блокирует потоки: запись и чтение асинхронные. Главная возможность, ради которой её выбирают, — **ограниченная ёмкость с обратным давлением**: если потребитель не успевает, производитель начинает ждать (или сообщение отбрасывается по заданной политике), и приложение не съедает всю память. Типичные применения: фоновая обработка задач из HTTP-запросов, буферизация событий, конвейеры обработки, пакетирование записей в базу.

## Зачем это нужно

Классическая задача: HTTP-запрос должен быстро ответить, а тяжёлую работу сделать потом. Наивное решение — `_ = Task.Run(...)` — теряет ошибки, не ограничивает нагрузку и ломается при остановке приложения ([[Дедлоки, async void и типичные ошибки]]).

```csharp
// Запрос кладёт задание в канал и сразу отвечает
await _channel.Writer.WriteAsync(new EmailJob(userId, template), ct);
return Results.Accepted();

// Фоновая служба разбирает канал
await foreach (var job in _channel.Reader.ReadAllAsync(stoppingToken))
    await SendAsync(job, stoppingToken);
```

Получаем контроль: известна ёмкость очереди, есть отмена, есть обработка ошибок, при остановке приложения обработка завершается корректно.

---

## Создание и базовое использование

```csharp
using System.Threading.Channels;

// Неограниченный: расти будет до памяти — использовать осторожно
Channel<Job> unbounded = Channel.CreateUnbounded<Job>();

// Ограниченный: главный вариант для продакшена
Channel<Job> bounded = Channel.CreateBounded<Job>(new BoundedChannelOptions(capacity: 1000)
{
    FullMode = BoundedChannelFullMode.Wait,   // что делать при переполнении
    SingleReader = false,
    SingleWriter = false,
    AllowSynchronousContinuations = false
});

// Производитель
await bounded.Writer.WriteAsync(job, ct);          // ждёт, если очередь полна
bool written = bounded.Writer.TryWrite(job);        // не ждёт: false, если места нет
bounded.Writer.Complete();                          // больше записей не будет

// Потребитель
await foreach (var job in bounded.Reader.ReadAllAsync(ct))
    await ProcessAsync(job, ct);

// Или вручную
while (await bounded.Reader.WaitToReadAsync(ct))
    while (bounded.Reader.TryRead(out var job))
        await ProcessAsync(job, ct);
```

Разделение на `Writer` и `Reader` не косметическое: производителю передают только `ChannelWriter<T>`, потребителю — `ChannelReader<T>`, и роли не путаются.

### Политики переполнения

| `FullMode` | Поведение при заполненной очереди |
|---|---|
| `Wait` | `WriteAsync` ждёт освобождения места — обратное давление |
| `DropOldest` | вытесняется самый старый элемент |
| `DropNewest` | вытесняется самый новый (из очереди) |
| `DropWrite` | новый элемент просто не записывается |

Выбор зависит от смысла данных: для задач, которые нельзя терять, — `Wait`; для метрик и телеметрии, где важна свежесть, — `DropOldest`; для необязательных уведомлений — `DropWrite`.

**Обратное давление** (backpressure) — ключевое понятие. Если производитель быстрее потребителя, неограниченная очередь просто растёт, пока не кончится память; ограниченная заставляет производителя притормозить. В веб-приложении это означает, что при перегрузке запросы начнут ждать или получать 503 — что честнее, чем падение по OOM через час.

---

## Оптимизации

```csharp
new BoundedChannelOptions(1000)
{
    SingleReader = true,      // ровно один потребитель
    SingleWriter = true,      // ровно один производитель
    AllowSynchronousContinuations = false
}
```

- **`SingleReader`/`SingleWriter`** — обещание, позволяющее реализации использовать более быстрые внутренние структуры без части синхронизации. Нарушение обещания приводит к неопределённому поведению, поэтому ставить их надо, только если это действительно гарантировано.
- **`AllowSynchronousContinuations = true`** позволяет выполнять продолжение читателя прямо в потоке писателя: быстрее, но писатель может «застрять» на чужой обработке. По умолчанию `false`, и менять это стоит осознанно.

---

## Завершение

```csharp
// Производитель сообщает, что записей больше не будет
channel.Writer.Complete();

// или с ошибкой — потребитель получит её при чтении
channel.Writer.Complete(new InvalidOperationException("источник данных недоступен"));

// Потребитель: ReadAllAsync завершится сам после Complete и вычерпывания очереди
await foreach (var item in channel.Reader.ReadAllAsync(ct)) { }

// Дождаться полного завершения обработки
await channel.Reader.Completion;
```

Это важное отличие от «просто очереди»: канал имеет явное состояние завершения, поэтому потребитель знает, когда данные закончились, и цикл завершается сам. Для фоновой службы это означает корректную остановку: при `SIGTERM` производитель вызывает `Complete`, потребитель дочитывает остаток и выходит ([[Background services и IHostedService]]).

---

## Типовые конвейеры

### Фоновая обработка задач из HTTP

```csharp
public sealed class JobQueue
{
    private readonly Channel<Job> _channel = Channel.CreateBounded<Job>(
        new BoundedChannelOptions(1000) { FullMode = BoundedChannelFullMode.Wait });

    public ValueTask EnqueueAsync(Job job, CancellationToken ct) => _channel.Writer.WriteAsync(job, ct);
    public IAsyncEnumerable<Job> ReadAllAsync(CancellationToken ct) => _channel.Reader.ReadAllAsync(ct);
}

public sealed class JobWorker(JobQueue queue, ILogger<JobWorker> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await foreach (var job in queue.ReadAllAsync(stoppingToken))
        {
            try { await job.RunAsync(stoppingToken); }
            catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested) { break; }
            catch (Exception ex) { logger.LogError(ex, "Задача {JobId} упала", job.Id); }
        }
    }
}
```

Обрати внимание на обработку ошибок: исключение одной задачи логируется и **не останавливает** обработчик — иначе первая же ошибка убьёт всю фоновую обработку.

### Несколько потребителей

```csharp
var consumers = Enumerable.Range(0, 4).Select(async _ =>
{
    await foreach (var job in reader.ReadAllAsync(ct))
        await ProcessAsync(job, ct);
});
await Task.WhenAll(consumers);
```

Каждый элемент достаётся ровно одному потребителю (это очередь, а не рассылка), поэтому параллелизм получается естественно. Порядок обработки при этом больше не гарантируется — если он важен, потребитель должен быть один либо нужно партиционирование по ключу.

### Конвейер из этапов

```csharp
// Чтение → преобразование → запись, каждый этап со своей ёмкостью
var raw = Channel.CreateBounded<string>(100);
var parsed = Channel.CreateBounded<Order>(100);

var reading = Task.Run(async () => { await foreach (var line in ReadLinesAsync(ct)) await raw.Writer.WriteAsync(line, ct); raw.Writer.Complete(); });
var parsing = Task.Run(async () => { await foreach (var line in raw.Reader.ReadAllAsync(ct)) await parsed.Writer.WriteAsync(Parse(line), ct); parsed.Writer.Complete(); });
var writing = Task.Run(async () => { await foreach (var order in parsed.Reader.ReadAllAsync(ct)) await SaveAsync(order, ct); });

await Task.WhenAll(reading, parsing, writing);
```

Каждый этап работает в своём темпе, а ограниченные каналы между ними не дают быстрому этапу переполнить память.

---

## Сравнение с альтернативами

| | `Channel<T>` | `BlockingCollection<T>` | `ConcurrentQueue<T>` | Брокер (RabbitMQ, Kafka) |
|---|---|---|---|---|
| Асинхронность | да | нет (блокирует потоки) | нет ожидания | да |
| Обратное давление | да | да | нет | да |
| Переживает перезапуск | **нет** | нет | нет | да |
| Между процессами | нет | нет | нет | да |
| Гарантии доставки | нет | нет | нет | да |
| Стоимость | наносекунды | наносекунды + блокировки | наносекунды | сеть |

Главное ограничение канала: он живёт **в памяти процесса**. При перезапуске всё, что в очереди, теряется. Поэтому задачи, потеря которых недопустима (платежи, отправка документов), кладут в персистентную очередь или в таблицу-outbox, а канал используют для того, что можно потерять или что легко повторить ([[Паттерн Transactional Outbox]], [[RabbitMQ: основы и паттерны]]).

> [!warning] Подводные камни
> - **Неограниченный канал в проде.** При отставании потребителя память растёт до OOM. Ёмкость задают всегда.
> - **Потеря данных при перезапуске.** Канал в памяти: для критичных сообщений нужен брокер или таблица.
> - **Исключение в потребителе.** Без обработки внутри цикла первая ошибка останавливает всю фоновую обработку.
> - **Забытый `Complete`.** Потребитель ждёт вечно, приложение не останавливается.
> - **`SingleReader`/`SingleWriter` без гарантии.** Обещание нарушено — поведение неопределённое.
> - **`AllowSynchronousContinuations = true` по незнанию.** Продолжение потребителя выполняется в потоке производителя.
> - **Ожидание порядка при нескольких потребителях.** Очередь гарантирует порядок извлечения, но не порядок завершения обработки.
> - **Канал как замена очереди сообщений между сервисами.** Он работает только внутри процесса.

> [!example] Как делают в бою
> Самый частый сценарий — «принять и ответить»: HTTP-эндпоинт кладёт задание в ограниченный канал и возвращает 202, а `BackgroundService` разбирает очередь. Это заметно улучшает время отклика и делает нагрузку предсказуемой: ёмкость канала прямо ограничивает объём незавершённой работы.
> Второй сценарий — пакетирование: потребитель собирает элементы до размера пакета или до истечения таймаута и записывает их в базу одной операцией. Это в разы снижает число обращений к БД по сравнению с записью по одному.
> Третий — буферизация телеметрии и логов, где допустима потеря: канал создают с `FullMode = DropOldest`, чтобы всплеск нагрузки не влиял на основную работу.
> Обязательные атрибуты продового использования: ограниченная ёмкость, метрика длины очереди (её рост — ранний признак того, что потребитель не справляется), обработка исключений внутри цикла потребителя и корректное завершение при остановке. И осознанное решение о том, что происходит при перезапуске: если терять нельзя — канал не подходит.

---

## Вопросы с собеседований

> [!question]- Что такое `Channel<T>` и чем он лучше `BlockingCollection<T>`?
> Это асинхронная очередь для передачи данных между производителями и потребителями внутри процесса, с разделением на `ChannelWriter<T>` и `ChannelReader<T>`. Главное преимущество перед `BlockingCollection<T>` — отсутствие блокировок: `WriteAsync` и `ReadAsync` асинхронны, поэтому ожидание не занимает поток пула. При работе с `BlockingCollection` каждый ожидающий потребитель держит поток, и десяток потребителей означает десяток заблокированных потоков — под нагрузкой это прямой путь к исчерпанию пула. Дополнительно канал даёт явное состояние завершения (`Complete`, `Completion`), удобное перечисление через `ReadAllAsync`, интеграцию с `CancellationToken` и настраиваемые политики переполнения. `BlockingCollection` остаётся уместным только в чисто синхронном коде.

> [!question]- Что такое обратное давление и зачем ограничивать ёмкость канала?
> Обратное давление — механизм, при котором медленный потребитель заставляет производителя притормозить. В канале это реализуется ограниченной ёмкостью: когда очередь заполнена, `WriteAsync` не завершается, пока не освободится место (при `FullMode.Wait`). Без ограничения очередь растёт до тех пор, пока хватает памяти, и приложение падает по `OutOfMemoryException` — причём обычно через часы работы под нагрузкой, когда причина уже неочевидна. Ограниченная ёмкость превращает эту ситуацию в предсказуемую: производитель ждёт, время отклика растёт, и по метрике длины очереди видно, что потребитель не справляется. Альтернативные политики (`DropOldest`, `DropNewest`, `DropWrite`) применяют там, где данные можно терять — телеметрия, необязательные уведомления. Выбор политики — это явное решение о том, что важнее: не потерять данные или не задерживать производителя.

> [!question]- Можно ли использовать `Channel<T>` вместо очереди сообщений?
> Только для задач, потеря которых допустима, и только внутри одного процесса. Канал живёт в памяти: при перезапуске приложения, падении процесса или переезде пода всё содержимое исчезает без следа, а гарантий доставки, повторов и подтверждений в нём нет. Поэтому для платежей, отправки документов, интеграционных событий используют брокер (RabbitMQ, Kafka) либо таблицу-outbox в той же транзакции, что и бизнес-операция. Канал же отлично подходит для внутрипроцессных сценариев: разгрузить HTTP-запрос от фоновой работы, буферизовать телеметрию, собрать записи в пакет перед вставкой в базу, построить конвейер обработки. Часто их сочетают: сообщение читается из брокера, кладётся в канал для параллельной обработки внутри процесса, а подтверждение брокеру отправляется только после успешной обработки.

---

## Задачи

### Задача 1. Разгрузить эндпоинт

Эндпоинт отправки уведомлений отвечает за 3 секунды, потому что синхронно шлёт письма. Нужно отвечать сразу.

> [!success]- Решение
> ```csharp
> // Регистрация
> builder.Services.AddSingleton(_ => Channel.CreateBounded<EmailJob>(
>     new BoundedChannelOptions(5_000) { FullMode = BoundedChannelFullMode.Wait }));
> builder.Services.AddHostedService<EmailWorker>();
>
> // Эндпоинт
> app.MapPost("/notify", async (NotifyRequest request, Channel<EmailJob> channel, CancellationToken ct) =>
> {
>     await channel.Writer.WriteAsync(new EmailJob(request.UserId, request.Template), ct);
>     return Results.Accepted();
> });
>
> // Обработчик
> public sealed class EmailWorker(Channel<EmailJob> channel, IEmailSender sender, ILogger<EmailWorker> logger)
>     : BackgroundService
> {
>     protected override async Task ExecuteAsync(CancellationToken stoppingToken)
>     {
>         await foreach (var job in channel.Reader.ReadAllAsync(stoppingToken))
>         {
>             try { await sender.SendAsync(job, stoppingToken); }
>             catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested) { break; }
>             catch (Exception ex) { logger.LogError(ex, "Не удалось отправить {JobId}", job.Id); }
>         }
>     }
> }
> ```
> Что здесь важно: ёмкость ограничена, поэтому при недоступности почтового сервиса приложение не съест память — вместо этого запросы начнут ждать и станет видно проблему; ошибка отправки логируется и не останавливает обработчик; отмена обрабатывается штатно, и при остановке приложения обработчик дочитывает очередь и выходит.
> Оговорка, которую надо проговорить честно: при перезапуске сервиса непереданные уведомления теряются. Если это неприемлемо, задание сначала пишут в базу в одной транзакции с бизнес-операцией, а канал используют лишь как ускоритель ([[Паттерн Transactional Outbox]]).

### Задача 2. Пакетная запись

Поток событий: до 10 000 в секунду, каждое записывать в базу отдельно — слишком дорого. Нужно писать пакетами по 500 или раз в секунду.

> [!success]- Решение
> ```csharp
> protected override async Task ExecuteAsync(CancellationToken ct)
> {
>     var buffer = new List<Event>(500);
>     using var timer = new PeriodicTimer(TimeSpan.FromSeconds(1));
>
>     var reader = _channel.Reader;
>     while (!ct.IsCancellationRequested)
>     {
>         // Ждём либо появления данных, либо срабатывания таймера
>         var readTask = reader.WaitToReadAsync(ct).AsTask();
>         var tickTask = timer.WaitForNextTickAsync(ct).AsTask();
>         var completed = await Task.WhenAny(readTask, tickTask);
>
>         if (completed == readTask && await readTask)
>             while (buffer.Count < 500 && reader.TryRead(out var e))
>                 buffer.Add(e);
>
>         if (buffer.Count >= 500 || (completed == tickTask && buffer.Count > 0))
>         {
>             await _db.BulkInsertAsync(buffer, ct);
>             buffer.Clear();
>         }
>     }
>
>     if (buffer.Count > 0) await _db.BulkInsertAsync(buffer, CancellationToken.None);   // дописать остаток
> }
> ```
> Идея: накапливать элементы до порога размера или до истечения интервала — что наступит раньше. `TryRead` в цикле забирает всё, что уже есть в канале, не создавая задачу на каждый элемент. Остаток буфера обязательно дописывается при остановке, иначе последние события потеряются.
> Практическая деталь: `Task.WhenAny` в цикле создаёт задачи на каждой итерации — для очень горячих путей это заметно, и там пишут более экономный вариант с одним таймером и флагом. Но для десяти тысяч событий в секунду приведённая схема работает без проблем и остаётся читаемой.

---

## Итог

- `Channel<T>` — асинхронная очередь производитель-потребитель внутри процесса, без блокировки потоков.
- Ограниченная ёмкость даёт обратное давление: производитель ждёт вместо бесконтрольного роста памяти.
- Политики переполнения (`Wait`, `DropOldest`, `DropNewest`, `DropWrite`) выбираются по ценности данных.
- `Complete` и `Completion` дают явное завершение: потребитель знает, когда данные кончились.
- `ReadAllAsync` превращает канал в асинхронный поток, удобный для `await foreach` и фоновых служб.
- Несколько потребителей получают элементы по одному каждый; порядок завершения обработки не гарантирован.
- Исключение в цикле потребителя надо ловить внутри, иначе обработка остановится навсегда.
- Канал живёт в памяти: для данных, которые нельзя терять, нужен брокер или outbox.

## Связанное

- [[IAsyncEnumerable и асинхронные потоки]] — `ReadAllAsync` как асинхронный поток
- [[Background services и IHostedService]] — типичный потребитель канала
- [[Конкурентные коллекции]] — очереди без асинхронного ожидания
- [[Task и Task Parallel Library]] · [[CancellationToken и отмена операций]]
- [[Паттерн Transactional Outbox]] · [[RabbitMQ: основы и паттерны]] — когда канала недостаточно
- [[Parallel, PLINQ и параллелизм данных]] — параллельная обработка вместо очереди
- [[Наблюдаемость: логи, метрики, трейсы]] — метрика длины очереди
