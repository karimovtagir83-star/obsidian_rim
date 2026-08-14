---
tags: [раздел-07, основы, этап-1, подводный-камень]
aliases: [File IO, System.IO, Streams, Файлы, Потоки, FileStream, Path]
---

# Файлы, потоки и System.IO

> [!abstract] Коротко
> В `System.IO` два уровня. Верхний — статические помощники `File`, `Directory`, `Path`: одна строка на операцию, всё содержимое сразу в памяти. Нижний — абстракция `Stream`: последовательность байтов, у которой один и тот же интерфейс для файла, памяти, сети и сжатия, что позволяет обрабатывать данные потоком, не загружая целиком. Правило выбора простое: файл маленький и известен — верхний уровень; данные большие, неизвестного размера или приходят извне — потоки. И всё, что открывает ресурс, обязано освобождаться через `using`.

## Зачем это нужно

Файлы встречаются раньше, чем кажется: сохранение состояния в первом консольном проекте, чтение конфигурации, импорт CSV, выгрузка отчёта, загрузка вложений в API, логи. Ошибки здесь стоят дорого и одинаковы у всех:

- прочитали файл на 2 ГБ целиком в строку — `OutOfMemoryException`;
- забыли `using` — файл остаётся заблокированным до сборки мусора;
- собрали путь конкатенацией — сломалось при переносе в Linux-контейнер;
- писали поверх существующего файла и упали посередине — данные потеряны;
- взяли имя файла из запроса пользователя — получили чтение `/etc/passwd`.

---

## Верхний уровень: File, Directory, Path

```csharp
// Чтение и запись целиком — для небольших файлов
string text = await File.ReadAllTextAsync(path, ct);
string[] lines = await File.ReadAllLinesAsync(path, ct);
byte[] bytes = await File.ReadAllBytesAsync(path, ct);

await File.WriteAllTextAsync(path, content, ct);       // перезапишет
await File.AppendAllTextAsync(path, line, ct);         // допишет

// Ленивое чтение построчно — память не зависит от размера файла
foreach (var line in File.ReadLines(path))
    Process(line);

await foreach (var line in File.ReadLinesAsync(path, ct))   // .NET 7+
    await ProcessAsync(line, ct);

// Файловые операции
File.Exists(path);
File.Copy(source, destination, overwrite: true);
File.Move(source, destination, overwrite: true);
File.Delete(path);                                     // не бросает, если файла нет
var info = new FileInfo(path);
Console.WriteLine($"{info.Length} байт, изменён {info.LastWriteTimeUtc:O}");

// Каталоги
Directory.CreateDirectory(dir);                        // создаёт всю цепочку, не бросает если есть
foreach (var file in Directory.EnumerateFiles(dir, "*.csv", SearchOption.AllDirectories))
    Process(file);                                      // ленивое перечисление
```

`EnumerateFiles` против `GetFiles`: первый возвращает `IEnumerable<string>` и отдаёт имена по мере обхода, второй сначала собирает весь массив. Для каталога с сотней тысяч файлов разница принципиальна — и по памяти, и по времени до первого результата. То же с `EnumerateDirectories`/`EnumerateFileSystemEntries`.

### Path: пути без ручной склейки

```csharp
var full = Path.Combine("data", "2026", "orders.csv");   // разделитель по платформе
var absolute = Path.GetFullPath(full);                    // нормализует . и ..
Path.GetFileName(full);                                    // orders.csv
Path.GetFileNameWithoutExtension(full);                    // orders
Path.GetExtension(full);                                   // .csv
Path.GetDirectoryName(absolute);
Path.ChangeExtension(full, ".json");
Path.GetTempFileName();                                    // создаёт файл во временном каталоге
Path.Combine(Path.GetTempPath(), Guid.NewGuid().ToString("N"));

// Базовый каталог приложения — единственный надёжный якорь для относительных путей
var configPath = Path.Combine(AppContext.BaseDirectory, "config", "settings.json");
```

Никогда не собирать путь строкой: `dir + "/" + name` ломается на Windows, `dir + "\\" + name` — везде остальном, а лишние или отсутствующие разделители дают несуществующие пути. `Path.Combine` решает это и заодно корректно обрабатывает случай, когда второй аргумент уже абсолютный.

Отдельно про регистр: macOS по умолчанию регистронезависима, Linux — нет. Файл, который открывается локально как `Data.json`, не найдётся в контейнере, если на диске он `data.json`. Это самая частая «мистическая» ошибка первого деплоя ([[Командная строка Linux для .NET-разработчика]]).

---

## Потоки

`Stream` — абстракция «последовательность байтов с позицией». Одинаковый интерфейс у файла, памяти, сети, сжатия, шифрования, что позволяет строить конвейеры.

```csharp
await using var file = new FileStream(
    path,
    new FileStreamOptions
    {
        Mode = FileMode.Open,
        Access = FileAccess.Read,
        Share = FileShare.Read,          // разрешить другим читать параллельно
        Options = FileOptions.Asynchronous | FileOptions.SequentialScan,
        BufferSize = 64 * 1024
    });

using var reader = new StreamReader(file, System.Text.Encoding.UTF8);
while (await reader.ReadLineAsync(ct) is { } line)
    Process(line);
```

Основные реализации: `FileStream`, `MemoryStream` (буфер в памяти), `NetworkStream`, `GZipStream`/`BrotliStream` (сжатие), `CryptoStream` (шифрование), `PipeStream`. Поверх них — «читатели и писатели»: `StreamReader`/`StreamWriter` для текста, `BinaryReader`/`BinaryWriter` для примитивов.

Ключевые свойства и параметры:

| Что | Смысл |
|---|---|
| `FileMode` | `Create`, `CreateNew`, `Open`, `OpenOrCreate`, `Append`, `Truncate` |
| `FileAccess` | `Read`, `Write`, `ReadWrite` |
| `FileShare` | что разрешено другим процессам одновременно: `None`, `Read`, `Write`, `ReadWrite` |
| `FileOptions.Asynchronous` | настоящий асинхронный ввод-вывод на уровне ОС |
| `FileOptions.SequentialScan` | подсказка ОС о последовательном чтении |
| `BufferSize` | размер внутреннего буфера; по умолчанию 4096 |

Конвейер из потоков собирается вложением:

```csharp
// Записать JSON, сжатый gzip, прямо в файл — без промежуточных строк и массивов
await using var output = File.Create("orders.json.gz");
await using var gzip = new System.IO.Compression.GZipStream(output,
    System.IO.Compression.CompressionLevel.Optimal);
await System.Text.Json.JsonSerializer.SerializeAsync(gzip, orders, cancellationToken: ct);
```

Именно так делают потоковую обработку: данные идут через конвейер порциями, и пиковая память не зависит от объёма.

### Асинхронность

Файловый ввод-вывод — операция ожидания, поэтому в серверном коде используются асинхронные версии: они освобождают поток на время работы диска. Важная деталь: асинхронность реально задействуется, только если поток открыт с `FileOptions.Asynchronous` (методы `File.*Async` делают это сами). Иначе `ReadAsync` выполнит синхронное чтение и просто вернёт готовую задачу.

И обязательный `CancellationToken`: чтение большого файла без возможности отмены не даст приложению корректно завершиться по `SIGTERM` ([[CancellationToken и отмена операций]]).

---

## Освобождение ресурсов

```csharp
// using-объявление: Dispose в конце области видимости
await using var stream = File.OpenRead(path);

// Классическая форма с явной областью
await using (var stream2 = File.OpenRead(path))
{
    // ...
}   // здесь DisposeAsync
```

Открытый файл — это дескриптор ОС и блокировка. Без освобождения файл остаётся занятым до финализации объекта сборщиком мусора, то есть неопределённо долго: на Windows это приводит к «файл используется другим процессом», в Linux — к исчерпанию лимита дескрипторов. `using` — не стилистическое предпочтение, а требование. Механика — [[IDisposable, using и паттерн Dispose]].

`await using` вызывает `DisposeAsync`: для файловых потоков это важно, потому что сброс буфера на диск — операция ввода-вывода, и синхронный `Dispose` заблокирует поток.

---

## Приёмы, которые стоит знать сразу

### Атомарная запись

```csharp
// Плохо: падение посередине оставит повреждённый файл
await File.WriteAllTextAsync(path, json, ct);

// Хорошо: пишем во временный, затем атомарно заменяем
var temp = path + ".tmp";
await File.WriteAllTextAsync(temp, json, ct);
File.Move(temp, path, overwrite: true);       // на уровне ФС это атомарная операция
```

Приём обязателен для файлов состояния и конфигурации: читатель либо видит старую версию целиком, либо новую целиком, но никогда не половину.

### Проверка существования — гонка

```csharp
if (File.Exists(path))            // между проверкой и открытием файл может исчезнуть
    using var s = File.OpenRead(path);   // FileNotFoundException

// Правильно: пробовать открыть и обрабатывать исключение
try { await using var s = File.OpenRead(path); }
catch (FileNotFoundException) { /* обработать */ }
catch (DirectoryNotFoundException) { /* обработать */ }
```

Это классическая ошибка TOCTOU (time-of-check to time-of-use). `File.Exists` полезен для подсказок пользователю, но не как гарантия перед операцией.

### Путь из пользовательского ввода

```csharp
// Опасно: "../../etc/passwd" уводит за пределы каталога
var unsafePath = Path.Combine(uploadDir, userFileName);

// Безопасно: нормализовать и проверить, что результат остался внутри
var root = Path.GetFullPath(uploadDir);
var candidate = Path.GetFullPath(Path.Combine(root, userFileName));
if (!candidate.StartsWith(root + Path.DirectorySeparatorChar, StringComparison.Ordinal))
    throw new UnauthorizedAccessException("Недопустимый путь");
```

Ещё надёжнее — вообще не использовать пользовательское имя: сохранять файл под сгенерированным именем (`Guid`), а исходное хранить в базе как метаданные. Подробнее об этом классе уязвимостей — [[OWASP Top 10 для .NET]].

### Кодировка

`File.*` по умолчанию читают и пишут UTF-8 без BOM — это правильное поведение. Однобайтовые кодировки (`windows-1251` в выгрузках из старых систем) в .NET требуют регистрации провайдера:

```csharp
System.Text.Encoding.RegisterProvider(System.Text.CodePagesEncodingProvider.Instance);
var legacy = await File.ReadAllTextAsync("export.csv", System.Text.Encoding.GetEncoding(1251), ct);
```

Про BOM и почему он ломает разбор CSV и JSON — [[Системы счисления и представление данных]].

> [!warning] Подводные камни
> - **Чтение большого файла целиком.** `ReadAllText` на 2 ГБ создаёт строку в 4 ГБ (UTF-16) и уходит в LOH. Нужен `ReadLines`/поток.
> - **Отсутствие `using`.** Файл остаётся заблокированным, дескрипторы утекают; в контейнере это упирается в лимит открытых файлов.
> - **Склейка путей строкой.** Ломается между платформами; `Path.Combine` и `AppContext.BaseDirectory`.
> - **Регистр в именах.** Работает на macOS, падает в Linux-контейнере.
> - **`File.Exists` как гарантия.** Между проверкой и открытием состояние меняется; ловить исключение.
> - **Относительные пути от текущего каталога.** `Directory.GetCurrentDirectory()` зависит от того, откуда запущено приложение, и в контейнере обычно не совпадает с ожиданиями. Якорь — `AppContext.BaseDirectory` или явный путь из конфигурации.
> - **Синхронный ввод-вывод в веб-приложении.** Блокирует поток пула на время работы диска; под нагрузкой это исчерпание пула.
> - **Запись поверх без временного файла.** Сбой посередине оставляет повреждённые данные.
> - **Путь из пользовательского ввода без нормализации.** Обход каталога и чтение чужих файлов.

> [!example] Как делают в бою
> В серверном коде прямая работа с файлами встречается реже, чем ожидается: конфигурация читается через `IConfiguration`, логи пишет провайдер логирования, загруженные файлы уходят в объектное хранилище (S3-совместимое), а не на диск контейнера — контейнеры эфемерны, и локальный диск исчезает при перезапуске.
> Там, где файлы всё же нужны (импорт и экспорт больших выгрузок), работают потоком: `IFormFile.OpenReadStream()` → парсер → база, без промежуточной материализации в памяти. Для CSV берут потоковый парсер, для JSON — `JsonSerializer.DeserializeAsyncEnumerable`, чтобы обрабатывать элементы по мере чтения.
> Временные файлы создают в `Path.GetTempPath()` с уникальным именем и удаляют в `finally`. Пути к рабочим каталогам всегда приходят из конфигурации, а не зашиваются в код: в контейнере это будет смонтированный том, локально — папка проекта.

---

## Вопросы с собеседований

> [!question]- Чем `File.ReadAllLines` отличается от `File.ReadLines`?
> `ReadAllLines` читает файл целиком и возвращает массив строк: память пропорциональна размеру файла, а обработка начинается только после полного чтения. `ReadLines` возвращает ленивое `IEnumerable<string>` и читает файл порциями по мере перечисления: потребление памяти постоянно и не зависит от размера, первая строка доступна сразу, а перечисление можно прервать в любой момент, не дочитывая. Для файла на несколько гигабайт первый вариант приведёт к `OutOfMemoryException` или как минимум к аллокации в LOH, второй отработает без проблем. Обратная сторона ленивости: файл остаётся открытым на всё время перечисления, поэтому нельзя сохранять последовательность и перечислять её позже, а повторное перечисление читает файл заново. Тот же принцип у пары `Directory.GetFiles`/`EnumerateFiles`.

> [!question]- Зачем нужен `Stream`, если есть `File.ReadAllBytes`?
> `Stream` — единая абстракция последовательного доступа к данным, у которой одинаковый интерфейс для файла, памяти, сети, сжатия и шифрования. Это даёт две вещи. Первая — постоянное потребление памяти: данные обрабатываются порциями, поэтому файл на 10 ГБ проходит через приложение с буфером в десятки килобайт. Вторая — композиция: потоки вкладываются друг в друга, и «прочитать из сети, распаковать, десериализовать» становится конвейером без промежуточных массивов. Плюс `Stream` поддерживает настоящий асинхронный ввод-вывод и отмену. `File.ReadAllBytes` удобен и уместен для небольших файлов известного размера — конфигурации, шаблона, картинки, — но на неизвестном объёме он превращается в мину.

> [!question]- Что произойдёт, если не вызвать `Dispose` у `FileStream`?
> Дескриптор файла ОС останется занятым до тех пор, пока объект не будет собран сборщиком мусора и не отработает финализатор, — а это происходит недетерминированно и может не случиться вовсе до завершения процесса. Последствия: на Windows файл остаётся заблокированным, и другие процессы получают «файл используется»; на Linux расходуются дескрипторы, и при достаточном количестве утечек процесс упирается в лимит `ulimit -n` и перестаёт открывать файлы и сокеты. Отдельная опасность при записи: данные могут остаться в буфере и не попасть на диск, то есть файл окажется неполным. Поэтому любой поток открывается через `using`/`await using`; для асинхронного кода предпочтителен `await using`, потому что сброс буфера — это операция ввода-вывода, и синхронный `Dispose` блокирует поток.

---

## Задачи

### Задача 1. Обработать большой CSV

Файл на 5 ГБ, формат `id;amount;currency`. Нужно посчитать сумму по каждой валюте. Память ограничена 512 МБ.

> [!success]- Решение
> ```csharp
> static async Task<Dictionary<string, decimal>> SumByCurrencyAsync(string path, CancellationToken ct)
> {
>     var totals = new Dictionary<string, decimal>(StringComparer.OrdinalIgnoreCase);
>
>     await foreach (var line in File.ReadLinesAsync(path, ct))
>     {
>         var span = line.AsSpan();
>         int first = span.IndexOf(';');
>         if (first < 0) continue;
>         int second = span[(first + 1)..].IndexOf(';');
>         if (second < 0) continue;
>
>         var amountSpan = span.Slice(first + 1, second);
>         var currency = new string(span[(first + second + 2)..].Trim());
>
>         if (!decimal.TryParse(amountSpan, System.Globalization.NumberStyles.Number,
>                               System.Globalization.CultureInfo.InvariantCulture, out var amount))
>             continue;
>
>         totals[currency] = totals.GetValueOrDefault(currency) + amount;
>     }
>
>     return totals;
> }
> ```
> Ключевые решения: `ReadLinesAsync` читает построчно, поэтому память не зависит от размера файла; разбор идёт по `ReadOnlySpan<char>` без `Split`, который создавал бы массив строк на каждую из десятков миллионов строк; `TryParse` по спану с инвариантной культурой — и без исключений на битых строках; словарь по валютам занимает единицы килобайт, потому что валют мало. Единственная неизбежная аллокация — строка ключа валюты, и её можно убрать, если заранее известен набор валют. Итог: постоянная память, один проход, отмена поддержана.

### Задача 2. Найти ошибки

```csharp
public void SaveReport(string userName, string content)
{
    var path = "C:\\reports\\" + userName + ".txt";
    if (File.Exists(path)) File.Delete(path);
    var writer = new StreamWriter(path);
    writer.Write(content);
}
```

> [!success]- Решение
> Ошибок пять. Абсолютный путь в стиле Windows — код не заработает в Linux-контейнере; путь должен приходить из конфигурации, а собираться `Path.Combine`. Имя файла берётся из пользовательского ввода без нормализации — обход каталога через `../`. Удаление перед записью создаёт окно, в котором файла нет вообще, а при сбое запись теряется целиком. `StreamWriter` не освобождается — дескриптор течёт, а буфер может не попасть на диск. И метод синхронный, то есть в веб-приложении блокирует поток пула.
> ```csharp
> public async Task SaveReportAsync(string reportId, string content, CancellationToken ct)
> {
>     var root = Path.GetFullPath(_options.ReportsDirectory);
>     Directory.CreateDirectory(root);
>
>     var safeName = $"{Guid.NewGuid():N}.txt";        // имя генерируем сами
>     var path = Path.Combine(root, safeName);
>     var temp = path + ".tmp";
>
>     await File.WriteAllTextAsync(temp, content, ct);  // пишем во временный
>     File.Move(temp, path, overwrite: true);            // атомарно заменяем
> }
> ```
> Исходное имя, если оно нужно пользователю, хранится в базе как метаданные — так и путь безопасен, и файл не перезапишется чужим.

---

## Итог

- `File`/`Directory`/`Path` — для небольших файлов и путей; `Stream` — для больших данных и конвейеров.
- Ленивые версии (`ReadLines`, `EnumerateFiles`) держат постоянную память и начинают работу сразу.
- Любой открытый поток освобождается через `using`/`await using`: иначе блокировки и утечка дескрипторов.
- Пути собираются `Path.Combine`, якорь — `AppContext.BaseDirectory` или конфигурация, а не текущий каталог.
- Регистр в именах файлов различается между macOS и Linux — источник ошибок при первом деплое.
- Запись важных файлов — во временный с последующим `File.Move(overwrite: true)`.
- `File.Exists` не гарантирует ничего: между проверкой и открытием состояние меняется.
- Путь из пользовательского ввода нормализуется и проверяется на выход за корень, а лучше вообще не используется.

## Связанное

- [[IDisposable, using и паттерн Dispose]] — почему освобождение обязательно
- [[Строки и работа с текстом]] · [[Системы счисления и представление данных]] — кодировки и BOM
- [[Span, ReadOnlySpan и Memory]] — разбор строк без аллокаций
- [[Сериализация: System.Text.Json]] — сериализация прямо в поток
- [[CancellationToken и отмена операций]] · [[async и await: как это работает на самом деле]]
- [[Командная строка Linux для .NET-разработчика]] — регистр, права, пути
- [[OWASP Top 10 для .NET]] — обход каталога и загрузка файлов
- [[Проект 1 — Консольный менеджер задач]] — первое практическое применение
