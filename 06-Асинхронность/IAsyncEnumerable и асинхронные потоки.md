---
tags: [раздел-06, middle, async, dotnet10]
aliases: [IAsyncEnumerable, await foreach, Async streams, Асинхронные потоки, EnumeratorCancellation]
---

# IAsyncEnumerable и асинхронные потоки

> [!abstract] Коротко
> `IAsyncEnumerable<T>` — последовательность, элементы которой появляются асинхронно: каждый шаг перечисления может ждать ввода-вывода. Это середина между `Task<IEnumerable<T>>` (сначала загрузить всё, потом обрабатывать) и подпиской на события: данные обрабатываются по мере поступления, память не зависит от объёма, а потребитель управляет темпом — он сам запрашивает следующий элемент. Пишется через `async IAsyncEnumerable<T>` с `yield return`, читается через `await foreach`. В .NET 10 к ним наконец добавили встроенные LINQ-операторы.

## Зачем это нужно

Три сценария, где обычная коллекция не подходит: выборка из базы на миллион строк (в память не влезает), поток событий от брокера или SSE (конца нет), постраничный обход внешнего API (страницы приходят по одной).

```csharp
// Всё в память: пик потребления пропорционален объёму
public async Task<List<Order>> GetAllAsync(CancellationToken ct)
    => await _db.Orders.ToListAsync(ct);

// Потоково: постоянная память, обработка начинается сразу
public IAsyncEnumerable<Order> StreamAll(CancellationToken ct)
    => _db.Orders.AsAsyncEnumerable();
```

---

## Как писать и читать

```csharp
public async IAsyncEnumerable<Order> ReadOrdersAsync(
    string path,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    using var reader = new StreamReader(path);

    while (await reader.ReadLineAsync(ct) is { } line)
    {
        if (TryParse(line, out var order))
            yield return order;                 // отдать элемент и приостановиться
    }
}

// Потребление
await foreach (var order in ReadOrdersAsync(path, ct))
{
    await ProcessAsync(order, ct);
    if (order.IsLast) break;                     // прекращение перечисления освободит ресурсы
}
```

Компилятор превращает такой метод в конечный автомат, реализующий `IAsyncEnumerable<T>`/`IAsyncEnumerator<T>` — тот же механизм, что у `async` и у обычных итераторов вместе взятых ([[async и await: как это работает на самом деле]], [[IEnumerable, IEnumerator и yield return]]).

Важные детали:

- **Метод ленив.** Тело не выполняется, пока не начнётся `await foreach`.
- **`break` корректно завершает перечисление**: вызывается `DisposeAsync` перечислителя, отрабатывают `finally` и `using` внутри итератора.
- **`await foreach` требует `IAsyncDisposable`-семантики** — она реализуется автоматически.

### Отмена: атрибут EnumeratorCancellation

```csharp
// Токен, переданный в метод
await foreach (var x in StreamAsync(ct)) { }

// Токен, переданный при перечислении
await foreach (var x in StreamAsync().WithCancellation(ct)) { }
```

Чтобы второй вариант работал, параметр токена в итераторе помечают `[EnumeratorCancellation]`: компилятор связывает токен, переданный в `WithCancellation`, с этим параметром. Без атрибута токен из `WithCancellation` просто игнорируется — тихая и неприятная ошибка ([[CancellationToken и отмена операций]]).

Если переданы оба токена, они объединяются связанным источником — отмена сработает от любого.

```csharp
await foreach (var x in StreamAsync(ct).ConfigureAwait(false)) { }   // в библиотеках
```

---

## LINQ над асинхронными потоками

До .NET 10 операторов не было — приходилось подключать пакет `System.Linq.Async`. В **.NET 10 LINQ для `IAsyncEnumerable<T>` встроен в платформу**:

```csharp
var total = await source
    .Where(o => o.IsPaid)
    .Select(o => o.Total)
    .SumAsync(ct);

var firstBig = await source.FirstOrDefaultAsync(o => o.Total > 1000, ct);
await foreach (var batch in source.Chunk(100).WithCancellation(ct)) { }
```

Операторы, возвращающие одно значение, асинхронны (`SumAsync`, `CountAsync`, `ToListAsync`), а промежуточные остаются ленивыми и возвращают `IAsyncEnumerable<T>`.

---

## Когда что выбирать

| | `Task<List<T>>` | `IAsyncEnumerable<T>` | `Channel<T>` |
|---|---|---|---|
| Модель | всё сразу | по требованию (pull) | передача (push) |
| Память | пропорциональна объёму | постоянная | ограничена ёмкостью |
| Начало обработки | после загрузки всего | с первого элемента | по мере поступления |
| Управление темпом | нет | у потребителя | ёмкость и режим ожидания |
| Несколько потребителей | тривиально (список) | нет, перечисление одноразовое | да, несколько читателей |
| Типичный случай | небольшая выборка | обход большой выборки, стриминг ответа | развязка producer/consumer |

Практическое правило: если данных немного и они нужны целиком — обычный список; если объём велик или бесконечен — асинхронный поток; если производитель и потребитель работают независимо и с разной скоростью — канал ([[System.Threading.Channels]]).

---

## Где это встречается в .NET

```csharp
// EF Core: потоковое чтение без материализации
await foreach (var order in _db.Orders.Where(o => o.IsPaid).AsAsyncEnumerable().WithCancellation(ct))
    await ExportAsync(order, ct);

// System.Text.Json: разбор большого массива по элементам
await foreach (var item in JsonSerializer.DeserializeAsyncEnumerable<Order>(stream, cancellationToken: ct))
    Process(item);

// Файл построчно
await foreach (var line in File.ReadLinesAsync(path, ct))
    Handle(line);

// Каналы
await foreach (var message in channel.Reader.ReadAllAsync(ct))
    await HandleAsync(message, ct);
```

Отдача потоком из ASP.NET Core — отдельный полезный сценарий: возвращая `IAsyncEnumerable<T>` из метода действия, получаем ответ, который начинает передаваться клиенту до завершения выборки, без буферизации всего результата в памяти ([[Server-Sent Events и стриминг ответов]]).

```csharp
[HttpGet("export")]
public IAsyncEnumerable<OrderDto> Export(CancellationToken ct)
    => _db.Orders.AsAsyncEnumerable().Select(Map).WithCancellation(ct);
```

> [!warning] Подводные камни
> - **Забытый `[EnumeratorCancellation]`.** Токен из `WithCancellation` игнорируется, отмена не работает.
> - **Открытое соединение на всё время обхода.** Потоковое чтение из БД удерживает соединение и курсор: долгая обработка каждого элемента блокирует его надолго. Иногда лучше читать пакетами.
> - **Повторное перечисление.** Второй `await foreach` по тому же источнику выполнит запрос заново (или упадёт, если источник одноразовый).
> - **`await foreach` без обработки исключений.** Ошибка в середине потока прерывает обход; частично обработанные данные надо учитывать.
> - **Смешение с `Parallel`.** Асинхронный поток последовательный по своей природе; для параллельной обработки элементов нужен `Parallel.ForEachAsync` над источником или канал с несколькими читателями.
> - **`ToListAsync` над бесконечным потоком.** Приведёт к бесконечной работе и исчерпанию памяти.
> - **Тяжёлая синхронная работа в теле итератора.** Она выполняется в потоке потребителя и блокирует его.

> [!example] Как делают в бою
> Основное применение — выгрузки и отчёты: данные читаются из базы потоком и сразу пишутся в ответ или в файл, поэтому сервис с лимитом памяти в 512 МБ спокойно экспортирует миллионы строк. Ключевой момент — не вставлять между чтением и записью материализацию (`ToList`), иначе выигрыш исчезает.
> Второе применение — обработка сообщений: `channel.Reader.ReadAllAsync(ct)` в фоновой службе даёт естественный цикл потребителя с отменой и без ручной работы с примитивами синхронизации.
> Третье — интеграции с постраничными API: итератор внутри запрашивает следующую страницу, а потребитель видит просто последовательность элементов и не знает про пагинацию.
> Чего избегают: долгой обработки каждого элемента при открытом курсоре БД (соединение занято) и параллельной обработки «внутри» асинхронного потока без явного ограничения — для этого берут `Parallel.ForEachAsync` с `MaxDegreeOfParallelism`.

---

## Вопросы с собеседований

> [!question]- Чем `IAsyncEnumerable<T>` отличается от `Task<IEnumerable<T>>`?
> `Task<IEnumerable<T>>` — одна асинхронная операция, которая возвращает **готовую** коллекцию: обработка начинается только после загрузки всех данных, а пик потребления памяти пропорционален объёму выборки. `IAsyncEnumerable<T>` — последовательность, в которой асинхронным является каждый шаг: элементы отдаются по мере готовности, потребитель обрабатывает их сразу, а память не зависит от общего объёма. Дополнительно у асинхронного потока есть естественное управление темпом: следующий элемент запрашивается только тогда, когда потребитель готов, — источник не может «завалить» его данными. Ограничения тоже есть: перечисление одноразовое и последовательное, поэтому для нескольких потребителей или параллельной обработки нужны другие инструменты (список, канал, `Parallel.ForEachAsync`). Практический выбор: небольшая выборка — список, большая или бесконечная — асинхронный поток.

> [!question]- Зачем нужен атрибут `[EnumeratorCancellation]`?
> Токен отмены в асинхронный поток можно передать двумя путями: как обычный параметр метода-итератора либо при перечислении через `WithCancellation`. Второй способ существует потому, что между созданием последовательности и её обходом может пройти время, и вызывающий хочет управлять отменой именно обхода. Чтобы токен из `WithCancellation` попал внутрь итератора, компилятору нужно знать, в какой параметр его подставить, — это и указывает атрибут `[EnumeratorCancellation]`. Без него код компилируется и работает, но токен из `WithCancellation` просто игнорируется: отмена не срабатывает, и обнаруживается это обычно в проде, когда запросы перестают прерываться. Если переданы оба токена (в метод и в `WithCancellation`), они объединяются связанным источником, и отмена срабатывает от любого.

> [!question]- Когда лучше `Channel<T>`, а когда `IAsyncEnumerable<T>`?
> `IAsyncEnumerable<T>` — модель «по требованию»: потребитель сам запрашивает следующий элемент, источник вычисляет его лениво. Это естественно для чтения данных — из базы, файла, постраничного API — и даёт управление темпом бесплатно. `Channel<T>` — модель передачи: производитель кладёт элементы, потребитель забирает, и они работают независимо, с разной скоростью, возможно в разных потоках и в разном количестве. Канал нужен, когда производитель существует сам по себе (фоновый приём сообщений, события из внешней системы), когда потребителей несколько или когда требуется буфер с явной ёмкостью и стратегией переполнения. Часто их комбинируют: канал получает данные от производителя, а потребитель читает его как асинхронный поток через `ReadAllAsync`, получая привычный `await foreach`.

---

## Задачи

### Задача 1. Переписать выгрузку

```csharp
public async Task<byte[]> ExportAsync(CancellationToken ct)
{
    var orders = await _db.Orders.ToListAsync(ct);       // 2 млн строк
    var sb = new StringBuilder();
    foreach (var o in orders) sb.AppendLine($"{o.Id};{o.Total}");
    return Encoding.UTF8.GetBytes(sb.ToString());
}
```

> [!success]- Решение
> Здесь три материализации подряд: список всех заказов, строка со всем содержимым и массив байтов — пиковое потребление в несколько раз больше самих данных, причём строка и массив уйдут в LOH.
> ```csharp
> public async Task ExportAsync(Stream output, CancellationToken ct)
> {
>     await using var writer = new StreamWriter(output, leaveOpen: true);
>
>     await foreach (var o in _db.Orders.AsAsyncEnumerable().WithCancellation(ct))
>         await writer.WriteLineAsync($"{o.Id};{o.Total}".AsMemory(), ct);
>
>     await writer.FlushAsync(ct);
> }
> ```
> Данные читаются из базы потоком и сразу пишутся в выходной поток: память постоянна и не зависит от числа строк, а клиент начинает получать ответ немедленно. Метод теперь принимает `Stream` вместо возврата массива — это ключевое изменение, без него потоковость теряется на последнем шаге.
> Оговорки: соединение с БД будет занято на всё время выгрузки, поэтому для очень долгих экспортов лучше выгружать пакетами или выносить в фоновую задачу; и стоит убедиться, что запрос не тянет лишние поля — потоковость не спасёт от `SELECT *` по широкой таблице.

### Задача 2. Найти ошибку

```csharp
public async IAsyncEnumerable<Message> ReadAsync(CancellationToken ct)
{
    while (true)
    {
        var batch = await _api.PollAsync(ct);
        foreach (var m in batch) yield return m;
    }
}

// Использование
await foreach (var m in ReadAsync().WithCancellation(cancellationToken))
    await HandleAsync(m);
```

> [!success]- Решение
> Основная ошибка: параметр `ct` не помечен `[EnumeratorCancellation]`, поэтому токен из `WithCancellation` в итератор не попадёт — внутри будет `CancellationToken.None`, и бесконечный цикл никогда не прервётся. Вызов `ReadAsync()` без аргумента передаёт значение по умолчанию, и опрос API продолжится даже после остановки приложения.
> Дополнительно: в теле потребителя `HandleAsync` вызывается без токена, а бесконечный поток без ограничения по времени и без обработки ошибок опроса остановится при первой же сетевой ошибке.
> ```csharp
> public async IAsyncEnumerable<Message> ReadAsync(
>     [EnumeratorCancellation] CancellationToken ct = default)
> {
>     while (!ct.IsCancellationRequested)
>     {
>         IReadOnlyList<Message> batch;
>         try { batch = await _api.PollAsync(ct); }
>         catch (Exception ex) when (ex is not OperationCanceledException)
>         {
>             _logger.LogError(ex, "Ошибка опроса, повтор через 5 секунд");
>             await Task.Delay(TimeSpan.FromSeconds(5), ct);
>             continue;
>         }
>
>         foreach (var m in batch) yield return m;
>     }
> }
>
> await foreach (var m in ReadAsync(ct))
>     await HandleAsync(m, ct);
> ```
> Обрати внимание: `yield return` нельзя размещать внутри блока `try` с `catch`, поэтому опрос и отдача элементов разделены — это типичное ограничение итераторов, о котором стоит помнить.

---

## Итог

- `IAsyncEnumerable<T>` — последовательность с асинхронным получением каждого элемента; пишется `yield return` в `async`-итераторе.
- Обработка начинается с первого элемента, память не зависит от объёма, темп задаёт потребитель.
- `[EnumeratorCancellation]` обязателен, иначе токен из `WithCancellation` игнорируется.
- `break` корректно завершает перечисление и освобождает ресурсы итератора.
- В .NET 10 LINQ-операторы для асинхронных потоков встроены в платформу.
- Источники в BCL: EF Core `AsAsyncEnumerable`, `DeserializeAsyncEnumerable`, `File.ReadLinesAsync`, `Channel.ReadAllAsync`.
- Перечисление одноразовое и последовательное: для нескольких потребителей — канал, для параллельности — `Parallel.ForEachAsync`.
- Потоковое чтение из БД удерживает соединение: долгую обработку элементов выносят за пределы обхода.

## Связанное

- [[IEnumerable, IEnumerator и yield return]] — синхронный прообраз
- [[System.Threading.Channels]] — push-модель и развязка производителя с потребителем
- [[async и await: как это работает на самом деле]] · [[CancellationToken и отмена операций]]
- [[Отложенное и немедленное выполнение]] — ленивость и повторное перечисление
- [[Server-Sent Events и стриминг ответов]] — отдача потока клиенту
- [[EF Core: запросы и загрузка связанных данных]] — `AsAsyncEnumerable` и курсоры
- [[Файлы, потоки и System.IO]]
