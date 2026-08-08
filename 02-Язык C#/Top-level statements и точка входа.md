---
tags: [раздел-02, основы, middle, собес, dotnet10, подводный-камень]
aliases: [Top-level statements, Entry point, Main method, Точка входа, Операторы верхнего уровня]
---

# Top-level statements и точка входа

> [!abstract] Коротко
> Точка входа (entry point) — это статический метод `Main`, который CLR вызывает при старте процесса. Допустимо ровно восемь его сигнатур и ровно один подходящий `Main` на сборку. С C# 9 можно вообще не писать `Main`: операторы верхнего уровня (top-level statements) в одном файле проекта компилятор сам заворачивает в `internal class Program` с методом `<Main>$(string[] args)`. Понимать эту развёртку важно на практике: от неё зависят доступность `args`, коды возврата, возможность поставить `[STAThread]` и работа `WebApplicationFactory<Program>` в интеграционных тестах.

## Зачем это нужно

Раньше даже «Hello, world» на C# стоил девяти строк шаблонного кода: namespace, класс, `static void Main(string[] args)`, и только потом одна полезная строка. Для новичка это девять строк, каждую из которых надо объяснять («что такое static?», «зачем класс, если я просто печатаю текст?»). Для опытного разработчика это шум в каждом консольном скрипте и в каждом `Program.cs` веб-приложения.

Сейчас: C# 9 (.NET 5) ввёл top-level statements, .NET 6 переписал на них шаблоны проектов, а C# 12 добавил `public partial class Program;` — однострочное объявление, которым чинят интеграционные тесты. В .NET 10 эволюция дошла до логического конца: `dotnet run app.cs` запускает один-единственный `.cs`-файл вообще без `.csproj`, а зависимости объявляются директивами `#:package`.

Почему так: точка входа — единственное место, где императивный код неизбежно начинается «на пустом месте». Требовать вокруг него ООП-обвязку было данью традиции Java/C++, а не необходимостью рантайма. Сама CLR ищет в метаданных сборки токен `EntryPointToken` — ей всё равно, как метод назывался в исходнике и в каком классе лежал. Об этом подробнее в [[Сборки, метаданные и IL]].

## Классический Main

```csharp
namespace TaskManager;

internal static class Program
{
    // Классическая точка входа: статический метод, аргументы командной строки
    private static void Main(string[] args)
    {
        Console.WriteLine($"Аргументов: {args.Length}");
        foreach (string arg in args)
        {
            Console.WriteLine($"  {arg}");
        }
    }
}
// Запуск: dotnet run -- alpha beta
// Вывод: Аргументов: 2
// Вывод:   alpha
// Вывод:   beta
```

Обратите внимание: `args` **не содержит** имени исполняемого файла — в отличие от `argv[0]` в C/C++. Полный набор, включая путь к хосту, даёт `Environment.GetCommandLineArgs()`, где элемент `[0]` — путь к исполняемому файлу.

### Все восемь допустимых сигнатур

| # | Сигнатура | Async | Код возврата процесса | Типичное применение |
|---|---|---|---|---|
| 1 | `static void Main()` | нет | всегда 0 (или `Environment.ExitCode`) | простейшие утилиты без аргументов |
| 2 | `static void Main(string[] args)` | нет | всегда 0 (или `Environment.ExitCode`) | классический шаблон до .NET 6 |
| 3 | `static int Main()` | нет | возвращённое значение | CLI-утилита без параметров |
| 4 | `static int Main(string[] args)` | нет | возвращённое значение | CLI-утилита, стандарт для скриптов CI |
| 5 | `static async Task Main()` | да | 0 (или `Environment.ExitCode`) | асинхронный старт без аргументов |
| 6 | `static async Task Main(string[] args)` | да | 0 (или `Environment.ExitCode`) | шаблон хоста ASP.NET Core до .NET 6 |
| 7 | `static async Task<int> Main()` | да | результат `Task<int>` | асинхронная утилита без параметров |
| 8 | `static async Task<int> Main(string[] args)` | да | результат `Task<int>` | асинхронный CLI с кодами выхода |

Ключевой момент: `async` — не часть сигнатуры точки входа, компилятор смотрит на **тип возврата**. Формально допустимы `Task` и `Task<int>`; вернуть `ValueTask`, `Task<string>` или объявить `async void Main` нельзя.

Как это работает под капотом: для async-`Main` компилятор генерирует настоящий синхронный `Main`, который вызывает ваш метод и делает `.GetAwaiter().GetResult()`. То есть блокирующее ожидание на главном потоке всё равно есть — просто его написали за вас. Механику state machine разбираем в [[async и await: как это работает на самом деле]].

```csharp
// Что вы пишете
internal static class Program
{
    private static async Task<int> Main(string[] args)
    {
        using var http = new HttpClient();
        string body = await http.GetStringAsync("https://example.com");
        return body.Length > 0 ? 0 : 1;
    }
}

// Что примерно генерирует компилятор (метод $EntryPoint, невыразимое имя)
// private static int $EntryPoint(string[] args)
//     => Main(args).GetAwaiter().GetResult();
```

### Требования к Main

- **Обязан быть `static`.** Нестатический `Main` даёт предупреждение CS0028 («has the wrong signature to be an entry point») и, если другого кандидата нет, ошибку CS5001 («does not contain a static 'Main' method suitable for an entry point»).
- **Может иметь любую доступность.** Частое заблуждение — что `Main` должен быть `public` или, наоборот, обязательно `private`. Ни то ни другое: `private`, `internal`, `public`, `protected` — все работают одинаково. Старые шаблоны писали `static void Main` без модификатора, то есть `private`. CLR вызывает точку входа по токену метаданных, модификаторы доступа при этом не проверяются.
- **Не может быть generic.** `static void Main<T>(string[] args)` точкой входа не будет.
- **Не может лежать в generic-типе.** `class Program<T> { static void Main() { } }` — не кандидат.
- **Может лежать во вложенном типе.** `class Outer { class Inner { static void Main() { } } }` — вполне легально, точка входа `Outer.Inner.Main`.
- **Ровно один подходящий `Main` на сборку.** Два кандидата — ошибка CS0017: «Program has more than one entry point defined». Лечится либо удалением лишнего, либо `StartupObject` (см. ниже).
- Не может быть `abstract`, `virtual`, `override`, не может быть локальной функцией.

## Top-level statements (C# 9)

Тот же «Hello, world» целиком:

```csharp
// Program.cs — весь файл
Console.WriteLine("Привет");
// Вывод: Привет
```

Правила, которые надо помнить:

1. **Только в одном файле проекта.** Второй файл с операторами верхнего уровня — ошибка CS8802: «Only one compilation unit can have top-level statements». Имя файла роли не играет (`Program.cs` — соглашение, не требование).
2. **Порядок в файле строгий:** сначала `using`-директивы (и `extern alias`), затем операторы верхнего уровня, и только после них — объявления типов и namespace. Нарушите порядок — CS8803/CS8805.
3. **Локальные функции разрешены** и живут внутри синтезированного метода со всеми вытекающими: видят локальные переменные верхнего уровня, могут быть рекурсивными, могут захватывать замыкания. Детали — в [[Локальные функции и замыкания]].
4. **Одновременно с явным `Main` — можно, но бессмысленно.** Компилятор выберет операторы верхнего уровня и выдаст предупреждение CS7022: «The entry point of the program is global code; ignoring 'Main()' entry point».

```csharp
using System.Globalization;

// операторы верхнего уровня
int[] numbers = [5, 3, 9, 1];        // коллекционное выражение, C# 12
Console.WriteLine(Sum(numbers));      // Вывод: 18
Console.WriteLine(Describe(numbers)); // Вывод: 4 элемента, максимум 9

// локальная функция — видна выше по тексту, живёт внутри <Main>$
static int Sum(ReadOnlySpan<int> values)
{
    int total = 0;
    foreach (int v in values) total += v;
    return total;
}

static string Describe(int[] values) =>
    string.Create(CultureInfo.InvariantCulture,
        $"{values.Length} элемента, максимум {values.Max()}");

// объявления типов — строго после операторов
internal readonly record struct Point(int X, int Y);
```

### Во что это разворачивается

Компилятор синтезирует класс `Program` и метод с «невыразимым» (unspeakable) именем `<Main>$`. Угловые скобки и `$` недопустимы в идентификаторах C#, поэтому вызвать этот метод из C#-кода нельзя — но он совершенно легален в IL и виден через рефлексию.

```
Program.cs
  Console.WriteLine("Привет");
  return 0;
        |
        | компилятор Roslyn
        v
internal class Program                 <- имя фиксировано: Program
{
    private static int <Main>$(string[] args)   <- имя невыразимо из C#
    {
        Console.WriteLine("Привет");
        return 0;
    }
}
        |
        v
IL: EntryPointToken -> метод <Main>$
```

Эквивалентный код, который можно написать руками:

```csharp
// Эквивалент top-level statements, но с выразимым именем метода
internal class Program
{
    private static int Main(string[] args)
    {
        Console.WriteLine("Привет");
        return 0;
    }
}
```

Три следствия, важных на практике:

- Класс называется именно `Program` — на это имя завязаны шаблоны тестов и `WebApplicationFactory<Program>`.
- Класс `internal`, а не `public`. Инстанцировать его смысла нет: полезных членов, кроме скрытого `<Main>$`, там нет.
- Компилятор объявляет его частичным (partial) неявно, поэтому вы можете дописать свою часть `partial class Program` — этим и пользуются в тестах.

### `args`, `return` и `await` на верхнем уровне

`args` — «магическая» переменная типа `string[]`, доступная в операторах верхнего уровня без объявления: это просто параметр синтезированного `<Main>$`.

```csharp
// Program.cs
if (args.Length == 0)
{
    Console.Error.WriteLine("Использование: app <файл>");
    return 2;   // тип возврата <Main>$ становится int
}

string path = args[0];
if (!File.Exists(path))
{
    Console.Error.WriteLine($"Файл не найден: {path}");
    return 1;
}

// await прямо на верхнем уровне: <Main>$ становится async Task<int>
string text = await File.ReadAllTextAsync(path);
Console.WriteLine($"Символов: {text.Length}");
return 0;
```

Что произошло с сигнатурой синтезированного метода:

| Что есть в файле | Сигнатура `<Main>$` |
|---|---|
| ни `return`, ни `await` | `static void <Main>$(string[] args)` |
| есть `return <int>` | `static int <Main>$(string[] args)` |
| есть `await`, нет `return <int>` | `static async Task <Main>$(string[] args)` |
| есть и `await`, и `return <int>` | `static async Task<int> <Main>$(string[] args)` |

Голый `return;` без значения просто завершает программу с кодом 0 и тип возврата не меняет.

## Классический Main против top-level statements

| Критерий | `static Main` | Top-level statements |
|---|---|---|
| Строк шаблона | 6–9 | 0 |
| Файлов с точкой входа | любой, но кандидат должен быть один | строго один файл |
| `[STAThread]`, `[MTAThread]` | можно | нельзя, атрибут вешать не на что |
| Доступность класса | какую написали | всегда `internal` (чинится `partial`) |
| Явное имя метода для рефлексии | есть | `<Main>$`, из C# не вызвать |
| Несколько точек входа + `StartupObject` | поддерживается | нет |
| Шаблон CLI | `dotnet new console --use-program-main` | `dotnet new console` (по умолчанию) |

Оба варианта дают идентичный IL с точностью до имён. Выбор — вопрос стиля и наличия ограничений (`[STAThread]`, несколько точек входа).

## Program для интеграционных тестов

`WebApplicationFactory<TEntryPoint>` из `Microsoft.AspNetCore.Mvc.Testing` принимает тип-маркер, по которому находит сборку приложения и через `HostFactoryResolver` дёргает её точку входа, перехватывая построенный `IHost`. Проблема: `TEntryPoint` — параметр обобщённого типа, а синтезированный `Program` объявлен `internal`. Из тестовой сборки он недоступен, и `WebApplicationFactory<Program>` не компилируется (CS0122 — «Program is inaccessible due to its protection level»).

Способ первый, канонический — дописать частичное объявление в конце `Program.cs`:

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/health", () => Results.Ok(new { status = "ok" }));

app.Run();

// C# 12: объявление класса без тела, точка с запятой вместо { }
// Делает синтезированный Program публичным для тестовой сборки
public partial class Program;
```

До C# 12 писали `public partial class Program { }` — работает и сейчас, просто длиннее.

Способ второй — оставить `Program` внутренним и открыть сборку тестам через `InternalsVisibleTo` в `.csproj`:

```xml
<ItemGroup>
  <InternalsVisibleTo Include="TaskManager.Api.Tests" />
</ItemGroup>
```

Первый способ предпочтительнее: он локален, не открывает тестам вообще все внутренности сборки и не ломается при переименовании тестового проекта. Подробности сценария — в [[Интеграционные тесты и WebApplicationFactory]], про устройство хоста — в [[Host, конфигурация и окружения]].

```csharp
// Тестовый проект
public sealed class HealthTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;

    public HealthTests(WebApplicationFactory<Program> factory) => _factory = factory;

    [Fact]
    public async Task Health_возвращает_200()
    {
        HttpClient client = _factory.CreateClient();
        HttpResponseMessage response = await client.GetAsync("/health");
        Assert.Equal(HttpStatusCode.OK, response.StatusCode);
    }
}
```

## Несколько Main в одном проекте: StartupObject

Ситуация из жизни: в проекте есть основное приложение и вспомогательная утилита-мигратор, у обеих свой `Main`. Компилятор упирается в CS0017. Решение — явно указать тип с точкой входа: ключ компилятора `-main:<полное имя типа>` или, что практичнее, свойство MSBuild в [[Файл проекта .csproj изнутри]]:

```xml
<PropertyGroup>
  <OutputType>Exe</OutputType>
  <StartupObject>TaskManager.Tools.MigratorProgram</StartupObject>
</PropertyGroup>
```

Указывается **полное имя типа** (с namespace), а не метода. Тип должен содержать ровно один подходящий `Main`. С операторами верхнего уровня это не работает — их класс всегда `Program`, и второй кандидат всё равно вызовет CS0017 или CS7022. Про `dotnet build`/`dotnet run` и передачу свойств — в [[dotnet CLI: полный обзор]].

## Коды возврата процесса

Соглашение UNIX, которому следует и Windows: **0 — успех, любое ненулевое — ошибка**. CI-системы, `make`, оболочки, Docker `HEALTHCHECK` — все смотрят именно на это число.

```csharp
// Коды возврата в стиле CLI-утилиты
const int ExitOk = 0;
const int ExitBadArgs = 2;      // соглашение GNU: 2 — ошибка использования
const int ExitNotFound = 66;    // sysexits.h: EX_NOINPUT

if (args.Length != 1) return ExitBadArgs;
if (!File.Exists(args[0])) return ExitNotFound;

Console.WriteLine("OK");
return ExitOk;
```

На Unix ядро отдаёт вызывающему процессу только младший байт кода: `return 256` превратится в `0`, `return -1` — в `255`. Держите коды в диапазоне 0–255 и не используйте 0 как «ошибку по модулю 256». В Windows код 32-битный целиком, поэтому кроссплатформенный код должен ориентироваться на более строгое ограничение.

### Три способа завершить процесс

| Способ | `finally` / `using` | Финализаторы, `ProcessExit` | Корректная остановка `IHost` | Когда применять |
|---|---|---|---|---|
| `return code;` из `Main` | выполняются | выполняются | да | всегда, если можно |
| `Environment.ExitCode = code;` + обычный выход | выполняются | выполняются | да | когда `Main` возвращает `void`/`Task` |
| `Environment.Exit(code)` | **не выполняются** | `ProcessExit` вызывается | нет | почти никогда |
| `Environment.FailFast(msg)` | **не выполняются** | **не выполняются** | нет | повреждённое состояние процесса |

```csharp
// Опасно: finally не выполнится, файл останется недописанным
static void Bad()
{
    using var writer = new StreamWriter("out.txt");   // Dispose пропущен
    try
    {
        Environment.Exit(1);   // процесс умирает прямо здесь
    }
    finally
    {
        Console.WriteLine("сюда управление не придёт");
    }
}

// Правильно: возвращаем код, раскрутка стека проходит нормально
static int Good()
{
    using var writer = new StreamWriter("out.txt");   // Dispose выполнится
    writer.WriteLine("данные");
    return 1;
}
```

Про `Environment.ExitCode`: если `Main` возвращает `int` или `Task<int>`, выигрывает возвращённое значение, а `ExitCode` игнорируется. Для `void`/`Task`-версий работает именно `ExitCode`.

`Environment.FailFast(string message, Exception? exception)` — это «аварийный молоток»: процесс убивается немедленно, создаётся crash-дамп, на Windows пишется запись в журнал событий. Смысл в том, чтобы не дать программе с повреждённым состоянием (испорченная структура данных, нарушенный инвариант, сбой в критической секции) продолжить работу и испортить внешние данные. В обычном коде — не применять.

## `[STAThread]`

```csharp
internal static class Program
{
    // Главный поток входит в однопоточное COM-апартамент (Single-Threaded Apartment)
    [STAThread]
    private static void Main()
    {
        ApplicationConfiguration.Initialize();
        Application.Run(new MainForm());
    }
}
```

Зачем это было нужно: WinForms и WPF работают поверх Win32-окон и активно используют COM-компоненты (буфер обмена, drag-and-drop, диалоги выбора файла — это всё OLE). COM-объекты с моделью STA обязаны вызываться из того же потока, где созданы, а маршалинг обеспечивает насос сообщений апартамента. Без `[STAThread]` поток попадает в MTA, и первый же вызов `OpenFileDialog` падает с `ThreadStateException`.

Почему сейчас чаще не нужен: консольные приложения и ASP.NET Core не трогают OLE, работают из пула потоков, и апартамент им безразличен. По умолчанию главный поток .NET — MTA, и это то, что нужно серверному коду.

Важное ограничение: атрибут вешается на **метод**, а у операторов верхнего уровня явного метода нет — `<Main>$` из C# не адресуется. Поэтому WinForms/WPF-шаблоны до сих пор генерируют классический `Main`. Нужен STA в проекте на top-level statements — переписывайте на явный `Main` (или, для WinForms в .NET, используйте свойство `<ApplicationDefaultFont>`-подобные генераторы шаблона, которые сами создают `Main` с атрибутом).

## File-based apps: .NET 10

Логичный финал упрощения точки входа: один `.cs`-файл вообще без проекта.

```bash
dotnet run app.cs
```

Файл — это те же операторы верхнего уровня плюс новые директивы препроцессора `#:`, которые заменяют собой `.csproj`. Директивы обязаны идти в начале файла, до любого C#-кода.

| Директива | Что делает | Аналог в `.csproj` |
|---|---|---|
| `#:package Humanizer@2.14.1` | подключает NuGet-пакет нужной версии | `<PackageReference>` |
| `#:sdk Microsoft.NET.Sdk.Web` | меняет SDK — можно писать веб-приложение | атрибут `Sdk` в `<Project>` |
| `#:property LangVersion=preview` | задаёт свойство MSBuild | `<PropertyGroup>` |
| `#:project ../MyLib` | ссылка на соседний проект | `<ProjectReference>` |
| `#:include utils.cs` | добавляет к компиляции ещё один файл | `<Compile Include>` |

```csharp
#!/usr/bin/env -S dotnet --
#:sdk Microsoft.NET.Sdk
#:package Humanizer@2.14.1
#:property LangVersion=preview

using Humanizer;

TimeSpan uptime = TimeSpan.FromMinutes(197);
Console.WriteLine(uptime.Humanize(precision: 2, culture: null));
// Вывод: 3 hours, 17 minutes
```

Shebang в первой строке плюс `chmod +x app.cs` превращают файл в исполняемый скрипт:

```bash
chmod +x app.cs
./app.cs        # запускается напрямую, как bash- или python-скрипт
```

Когда скрипт перерос формат — разворачиваем его в полноценный проект одной командой:

```bash
dotnet project convert app.cs
# создаёт папку app/ с app.csproj и Program.cs,
# директивы #: превращаются в элементы MSBuild
```

Ограничения, о которых стоит знать заранее:

- Один файл — одна программа: вторая точка входа невозможна, `StartupObject` не поддерживается.
- Первый запуск компилирует и кеширует сборку во временный каталог; последующие запуски используют кеш и стартуют заметно быстрее. Кеш инвалидируется по содержимому файла и директивам.
- Файл не должен попадать в область действия соседнего `.csproj` — иначе SDK попытается собрать его как часть проекта.
- Для CI и продакшена всё равно нужен обычный проект: file-based apps — это инструмент для скриптов, прототипов и утилит. См. [[Шпаргалка — dotnet CLI]].

## Что генерируют шаблоны

```bash
dotnet new console -o App                     # top-level statements (по умолчанию)
dotnet new console -o App --use-program-main  # классический static void Main
```

Начиная с .NET 6 `dotnet new console` даёт `Program.cs` из одной строки. Флаг `--use-program-main` возвращает старый вид — он полезен для WinForms-подобных сценариев, для команд с жёстким стайлгайдом и для обучения, когда хочется видеть структуру явно.

> [!warning] Подводные камни
>
> **Два файла с операторами верхнего уровня.** Скопировали `Program.cs` в `Program.Old.cs` «на всякий случай» — получили CS8802. Компилятор не подскажет, какой файл лишний, он просто откажется собирать проект. Причина: синтезированный метод `<Main>$` может быть только один, объединить два набора операторов компилятору нечем.
>
> **`Environment.Exit` внутри веб-приложения или хоста.** Вызов убивает процесс мимо `IHostApplicationLifetime`: не отработает graceful shutdown, не дождутся завершения `IHostedService`, не сбросятся буферы логгера, не закроются соединения с БД. В Kubernetes это превращается в 503 для запросов, которые уже были в обработке. Правильный путь — `IHostApplicationLifetime.StopApplication()`, см. [[Background services и IHostedService]].
>
> **`WebApplicationFactory<Program>` не компилируется.** Причина всегда одна: синтезированный `Program` — `internal`. Разработчики ищут баг в тестовой инфраструктуре, а лечится это строкой `public partial class Program;` в конце `Program.cs`.
>
> **`args[0]` — это не путь к программе.** Привычка из C/C++ приводит к тому, что первый реальный аргумент считают вторым. В .NET `args` содержит только аргументы; путь к исполняемому файлу берётся из `Environment.GetCommandLineArgs()[0]` или `Environment.ProcessPath`.
>
> **Код возврата больше 255 на Linux.** `return 300;` в контейнере превратится в `44` (300 mod 256), и CI решит, что программа упала по другой причине. На Windows тот же код вернётся как 300 — получаем разное поведение на разных платформах.
>
> **`Main` не статический.** Компилятор сначала выдаёт предупреждение CS0028 (сигнатура не подходит для точки входа), а потом ошибку CS5001. Люди читают только последнюю строку и начинают искать отсутствующий `Main`, хотя он на месте — просто без `static`.

> [!example] Как делают в бою
>
> **Веб-сервисы.** `Program.cs` — это top-level statements: `CreateBuilder`, регистрация сервисов, `Build`, middleware, `Run`. Последней строкой файла почти всегда стоит `public partial class Program;` — её ставят сразу, при создании проекта, чтобы через месяц не выяснять, почему не собираются тесты. Регистрацию сервисов при этом выносят в extension-методы, чтобы файл не разросся до трёхсот строк; см. [[Minimal API]].
>
> **CLI-утилиты.** Явные константы кодов возврата и `async Task<int> Main` (или `return` из top-level). Коды документируются в README: 0 — успех, 1 — ошибка выполнения, 2 — неверные аргументы. Это то, на что смотрит шаг pipeline в CI.
>
> **Скрипты автоматизации.** С .NET 10 вместо `bash`-скрипта с `jq` пишут `deploy.cs` с shebang и `#:package`. Плюс: типизация, отладчик, знакомый язык и никакого `.csproj` в репозитории с инфраструктурой. Минус: холодный старт первого запуска.
>
> **Легаси и `StartupObject`.** В старых решениях встречается проект с несколькими `Main` — обычно основное приложение и «служебный» режим. Современный подход — вынести утилиту в отдельный проект или сделать её подкомандой основного CLI, а не жонглировать `StartupObject`. #устарело

## Вопросы с собеседований

> [!question]- Сколько существует допустимых сигнатур `Main` и чем они различаются?
> Восемь: четыре синхронные (`void Main()`, `void Main(string[])`, `int Main()`, `int Main(string[])`) и четыре асинхронные (`Task Main()`, `Task Main(string[])`, `Task<int> Main()`, `Task<int> Main(string[])`). Различия по двум осям: принимает ли метод аргументы командной строки и возвращает ли код завершения процесса. Async-варианты появились в C# 7.1; ключевое слово `async` формально не входит в сигнатуру — компилятор ориентируется на тип возврата, поэтому `Task`/`Task<int>` можно вернуть и без `async`. Для async-версий компилятор генерирует настоящую синхронную точку входа, которая делает `GetAwaiter().GetResult()` на возвращённой задаче, так что главный поток всё равно блокируется до завершения.

> [!question]- Обязан ли `Main` быть `public`?
> Нет. Единственное жёсткое требование к модификаторам — `static`. Доступность может быть любой: `private`, `internal`, `protected`, `public`. Шаблоны до .NET 6 генерировали `static void Main` без модификатора, то есть `private`, и это никогда не мешало запуску. Причина в том, что CLR не вызывает точку входа как обычный метод из внешнего кода: она читает токен `EntryPointToken` из заголовка сборки и передаёт управление напрямую, проверки доступности при этом не производятся. Синтезированный компилятором `<Main>$` для top-level statements тоже объявлен `private`.

> [!question]- Во что компилятор разворачивает операторы верхнего уровня?
> В класс с фиксированным именем `Program`, объявленный `internal` и неявно частичным, с единственным статическим методом `<Main>$(string[] args)`. Имя метода «невыразимо» из C#: угловые скобки и `$` запрещены в идентификаторах, поэтому вызвать его из исходного кода нельзя, хотя в IL и через рефлексию он доступен. Тип возврата метода определяется содержимым файла: `void` по умолчанию, `int` при наличии `return` со значением, `Task`/`Task<int>` при наличии `await`. Именно из этой развёртки следуют две практические вещи: переменная `args` — это просто параметр метода, а `WebApplicationFactory<Program>` требует дописать `public partial class Program;`, потому что синтезированный класс внутренний.

> [!question]- Почему `WebApplicationFactory<Program>` не компилируется в тестовом проекте и как это чинить?
> Потому что `Program`, синтезированный из top-level statements, объявлен `internal`, а тип-аргумент обобщённого типа должен быть доступен из тестовой сборки — получаем CS0122. `WebApplicationFactory<TEntryPoint>` использует этот тип как маркер: по нему определяется сборка приложения, из которой через `HostFactoryResolver` вызывается точка входа и перехватывается построенный `IHost`. Канонический фикс — дописать в конец `Program.cs` строку `public partial class Program;` (синтаксис объявления без тела из C# 12); компилятор объявляет синтезированный класс частичным, поэтому части сливаются. Альтернатива — `<InternalsVisibleTo Include="MyApp.Tests" />` в `.csproj`, но она открывает тестам все внутренности сборки и ломается при переименовании тестового проекта.

> [!question]- Чем `Environment.Exit(1)` отличается от `return 1` из `Main`?
> `return` завершает метод нормально: раскручивается стек, отрабатывают все `finally` и `using`, корректно завершается `IHost`, освобождаются неуправляемые ресурсы. `Environment.Exit` убивает процесс прямо в точке вызова: `finally`-блоки текущих кадров стека не выполняются, `Dispose` не вызывается, graceful shutdown хоста не происходит, хотя событие `AppDomain.ProcessExit` всё-таки поднимается. Практическое следствие: недописанные файлы, незакрытые соединения, потерянные логи и 503 для запросов, которые были в обработке. Ещё жёстче ведёт себя `Environment.FailFast` — он не выполняет ни `finally`, ни финализаторы, ни `ProcessExit`, зато пишет crash-дамп; его применяют только когда состояние процесса повреждено и продолжать опаснее, чем упасть.

> [!question]- Зачем нужен `[STAThread]` и почему его нельзя использовать с top-level statements?
> Атрибут переводит главный поток приложения в однопоточное COM-апартамент (Single-Threaded Apartment). Это требование WinForms и WPF: буфер обмена, drag-and-drop, файловые диалоги — это OLE-компоненты, которые обязаны вызываться из потока-владельца и полагаются на насос сообщений STA; без атрибута поток попадает в MTA и первый же `OpenFileDialog` бросает `ThreadStateException`. Консольным и серверным приложениям апартамент не нужен, поэтому по умолчанию главный поток .NET — MTA. С top-level statements атрибут поставить некуда: он применяется к методу, а синтезированный `<Main>$` из C# не адресуется, поэтому UI-шаблоны до сих пор генерируют классический `Main`.

> [!question]- Что произойдёт, если в проекте окажется два подходящих `Main`?
> Компилятор выдаст ошибку CS0017 «Program has more than one entry point defined» — он не выбирает точку входа сам. Разрешается это либо удалением лишнего кандидата, либо явным указанием типа: ключ компилятора `-main:Полное.Имя.Типа` или, что удобнее, свойство MSBuild `<StartupObject>Полное.Имя.Типа</StartupObject>` в `.csproj`. Указывается именно тип, а не метод, и в нём должен быть ровно один подходящий `Main`. Отдельный случай — сочетание операторов верхнего уровня и явного `Main`: тогда ошибки не будет, компилятор выберет операторы верхнего уровня и выдаст предупреждение CS7022 о том, что `Main` проигнорирован.

## Задачи

### Задача 1. CLI с кодами возврата

Напишите консольную утилиту на операторах верхнего уровня: она принимает путь к текстовому файлу, асинхронно считает в нём количество непустых строк и печатает результат. Коды возврата: 0 — успех, 2 — не передан аргумент, 66 — файл не найден, 70 — ошибка чтения. Ошибки печатаются в `stderr`.

> [!success]- Решение
> ```csharp
> // Program.cs — весь файл
> const int ExitOk = 0;
> const int ExitBadArgs = 2;
> const int ExitNoInput = 66;
> const int ExitSoftware = 70;
>
> if (args.Length != 1)
> {
>     Console.Error.WriteLine("Использование: linecount <файл>");
>     return ExitBadArgs;
> }
>
> string path = args[0];
> if (!File.Exists(path))
> {
>     Console.Error.WriteLine($"Файл не найден: {path}");
>     return ExitNoInput;
> }
>
> try
> {
>     int count = 0;
>     await foreach (string line in File.ReadLinesAsync(path))
>     {
>         if (!string.IsNullOrWhiteSpace(line)) count++;
>     }
>
>     Console.WriteLine($"Непустых строк: {count}");
>     return ExitOk;
> }
> catch (IOException ex)
> {
>     Console.Error.WriteLine($"Ошибка чтения: {ex.Message}");
>     return ExitSoftware;
> }
> ```
> Наличие `await` и `return <int>` делает синтезированный метод `static async Task<int> <Main>$(string[] args)`. Ловим именно `IOException`, а не `Exception`: неожиданные ошибки должны падать с трассировкой, а не маскироваться кодом 70. Диагностика идёт в `stderr`, чтобы `stdout` можно было направить в пайп.

### Задача 2. Ручная развёртка

Дан файл с операторами верхнего уровня. Перепишите его в классический вид с явным `Main`, сохранив поведение один в один, и объясните, какой будет сигнатура.

```csharp
using System.Text.Json;

var config = await LoadAsync("config.json");
Console.WriteLine(config?.Name ?? "(без имени)");
return config is null ? 1 : 0;

static async Task<AppConfig?> LoadAsync(string path)
{
    await using FileStream stream = File.OpenRead(path);
    return await JsonSerializer.DeserializeAsync<AppConfig>(stream);
}

internal sealed record AppConfig(string Name);
```

> [!success]- Решение
> ```csharp
> using System.Text.Json;
>
> internal class Program
> {
>     // Есть await и return со значением -> async Task<int>
>     private static async Task<int> Main(string[] args)
>     {
>         var config = await LoadAsync("config.json");
>         Console.WriteLine(config?.Name ?? "(без имени)");
>         return config is null ? 1 : 0;
>
>         // локальная функция переезжает внутрь Main — там она и была
>         static async Task<AppConfig?> LoadAsync(string path)
>         {
>             await using FileStream stream = File.OpenRead(path);
>             return await JsonSerializer.DeserializeAsync<AppConfig>(stream);
>         }
>     }
> }
>
> internal sealed record AppConfig(string Name);
> ```
> Ключевой момент: локальная функция `LoadAsync` в исходнике не была методом класса — она жила внутри `<Main>$`. Поэтому корректная развёртка помещает её в тело `Main`, а не рядом с ним. Тип возврата — `Task<int>`, потому что в файле есть и `await`, и `return` со значением. Класс `Program` остаётся `internal`, как и синтезированный.

### Задача 3. Точка входа, пригодная для тестов

Есть Minimal API на операторах верхнего уровня. Тестовый проект не компилируется: `WebApplicationFactory<Program>` даёт CS0122. Почините минимальными правками и напишите тест, который проверяет эндпоинт `GET /sum?a=2&b=3`.

> [!success]- Решение
> ```csharp
> // Program.cs приложения
> var builder = WebApplication.CreateBuilder(args);
> var app = builder.Build();
>
> app.MapGet("/sum", (int a, int b) => Results.Ok(a + b));
>
> app.Run();
>
> // Одна строка, которая чинит тесты. Синтаксис C# 12: объявление без тела.
> public partial class Program;
> ```
>
> ```csharp
> // Тестовый проект
> public sealed class SumTests : IClassFixture<WebApplicationFactory<Program>>
> {
>     private readonly WebApplicationFactory<Program> _factory;
>
>     public SumTests(WebApplicationFactory<Program> factory) => _factory = factory;
>
>     [Fact]
>     public async Task Sum_складывает_числа()
>     {
>         HttpClient client = _factory.CreateClient();
>         int result = await client.GetFromJsonAsync<int>("/sum?a=2&b=3");
>         Assert.Equal(5, result);   // Вывод теста: passed
>     }
> }
> ```
> Компилятор объявляет синтезированный `Program` частичным, поэтому наше `public partial class Program;` не создаёт новый тип, а расширяет существующий и поднимает его доступность до `public`. Тестовой сборке нужен пакет `Microsoft.AspNetCore.Mvc.Testing` и `<FrameworkReference Include="Microsoft.AspNetCore.App" />`. Альтернатива через `InternalsVisibleTo` тоже сработает, но открывает тестам всю сборку целиком.

## Итог

- Точка входа — статический метод, ровно один подходящий кандидат на сборку (иначе CS0017); допустимо восемь сигнатур — комбинации «есть/нет `args`», «`void`/`int`», «синхронно/`Task`». `static` обязателен, доступность любая, generic нельзя ни у метода, ни у объемлющего типа.
- Top-level statements (C# 9) — синтаксический сахар: компилятор синтезирует `internal partial class Program` с методом `<Main>$(string[] args)`, чьё имя невыразимо из C#. Разрешены только в одном файле, идут после `using` и до объявлений типов.
- `args`, `return <int>` и `await` на верхнем уровне напрямую меняют сигнатуру `<Main>$` — от `void` до `async Task<int>`.
- Синтезированный `Program` внутренний, поэтому для `WebApplicationFactory<Program>` в конец файла добавляют `public partial class Program;` (C# 12) или прописывают `InternalsVisibleTo`.
- Завершать процесс следует `return`-ом кода из `Main`: `Environment.Exit` пропускает `finally`, `Dispose` и корректную остановку `IHost`, а `FailFast` не выполняет вообще ничего и пишет дамп. Держите коды в диапазоне 0–255 ради Linux.
- `[STAThread]` нужен только UI-фреймворкам поверх COM и несовместим с top-level statements — там придётся вернуть явный `Main` через `dotnet new console --use-program-main`.
- .NET 10 довёл идею до file-based apps: `dotnet run app.cs`, директивы `#:package`/`#:sdk`/`#:property`/`#:project`/`#:include`, shebang и `dotnet project convert` для перехода к обычному проекту.

## Связанное

- [[Синтаксис и структура программы]]
- [[Методы и параметры]]
- [[Локальные функции и замыкания]]
- [[Namespace, using, глобальные using]]
- [[async и await: как это работает на самом деле]]
- [[Сборки, метаданные и IL]]
- [[Интеграционные тесты и WebApplicationFactory]]
- [[Файл проекта .csproj изнутри]]
- [[dotnet CLI: полный обзор]]
- [[Новое в C# 12, 13, 14]]
- [[Проект 1 — Консольный менеджер задач]]
- [[02 — Язык C# (обзор раздела)]]
