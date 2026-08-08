---
tags: [раздел-03, ооп, проектирование, принципы, middle, собес]
aliases: [SOLID, SRP, OCP, LSP, ISP, DIP, Принципы SOLID, Solid principles]
---

# SOLID

> [!abstract] Коротко
> SOLID — пять принципов объектно-ориентированного дизайна, собранных Робертом Мартином из работ разных авторов в конце 1990-х. Это не законы и не чек-лист для код-ревью: это набор эвристик, которые уменьшают стоимость изменения кода. Каждый принцип решает конкретную боль — «правка в одном месте ломает другое», «чтобы добавить вариант, надо трогать старый код», «наследник врёт про контракт», «реализую 12 методов ради одного», «бизнес-логика прибита к SQL-драйверу».
> Главное, что нужно понять: SOLID — это производная от [[Связанность и связность (coupling, cohesion)]]. Все пять принципов — разные способы сказать «низкая связанность, высокая связность».

## Зачем это нужно

Софт живёт годами, и 80 % денег тратится не на написание, а на изменение. Стоимость изменения растёт, когда:

- одна правка тянет за собой правки в несоседних местах (высокая связанность);
- нельзя понять, что сломается, не прочитав весь модуль;
- нельзя протестировать кусок логики без базы, HTTP и часов реального времени;
- нельзя добавить новый вариант поведения, не переписывая работающий код.

SOLID — это пять конкретных техник против этих четырёх бед. Их придумали не ради красоты: каждый принцип родился из разбора реального провалившегося проекта.

> [!info] Откуда взялась аббревиатура
> Принципы формулировались Робертом Мартином в статьях 1995–2002 годов вразнобой (SRP, OCP, LSP, ISP, DIP). Аббревиатуру SOLID предложил Майкл Фэзерс примерно в 2004-м. LSP при этом старше всех — Барбара Лисков сформулировала подстановочность в 1987-м, и в оригинале это строгое формальное утверждение о подтипах, а не «совет по наследованию».

---

## S — Single Responsibility Principle

> [!quote] Формулировка
> «У модуля должна быть одна и только одна причина для изменения».
> Позднее Мартин переформулировал точнее: «Модуль должен отвечать перед одним и только одним актором».

### Что это значит на самом деле

Самая частая ошибка — читать SRP как «класс должен делать одну вещь». Тогда логичный вывод — дробить всё до классов из одного метода, и получается 200 классов `CreateOrderValidator`, `CreateOrderMapper`, `CreateOrderPersister`, между которыми невозможно ориентироваться.

SRP не про размер и не про количество методов. Он про **источник требований на изменение**. Если код меняется по требованию бухгалтерии и по требованию отдела логистики — это два актора, и им нужны два модуля. Иначе правка ради бухгалтерии сломает логистику, и наоборот.

### Нарушение

```csharp
// ПЛОХО: три актора в одном классе
public class Employee
{
    public decimal Salary { get; init; }
    public int HoursWorked { get; init; }
    public string Name { get; init; } = "";

    // Актор: финансовый отдел. Меняется, когда меняются правила расчёта.
    public decimal CalculatePay() => Salary + HoursWorked * 500m;

    // Актор: отдел кадров. Меняется, когда меняется форма отчёта.
    public string ReportHours() => $"{Name}: {HoursWorked} ч";

    // Актор: DBA / архитектура. Меняется, когда меняется хранилище.
    public void Save(SqlConnection connection) { /* INSERT INTO ... */ }
}
```

Классический сценарий беды (пример самого Мартина): финансисты и кадровики оба используют формулу «регулярные часы», кто-то вынес её в общий приватный метод, финансисты попросили изменить округление — и отчёт кадров тихо поехал. Никто не заметил до аудита.

### Рефакторинг

```csharp
// ХОРОШО: данные отдельно, поведение — по акторам

// Иммутабельные данные без поведения — здесь это уместно, см. заметку про
// анемичную модель в [[Закон Деметры и Tell, Don't Ask]]
public sealed record EmployeeData(string Name, decimal Salary, int HoursWorked);

// Меняется только по требованию финансов
public sealed class PayCalculator
{
    public decimal Calculate(EmployeeData e) => e.Salary + e.HoursWorked * 500m;
}

// Меняется только по требованию кадров
public sealed class HoursReporter
{
    public string Report(EmployeeData e) => $"{e.Name}: {e.HoursWorked} ч";
}

// Меняется только при смене хранилища
public interface IEmployeeRepository
{
    Task SaveAsync(EmployeeData employee, CancellationToken ct);
}
```

### Почему стало лучше

| Было | Стало |
|---|---|
| Правка формулы задевает файл, который читают три команды | Правка формулы — один файл, одна команда |
| Тест расчёта зарплаты требует `SqlConnection` | Тест расчёта — чистая функция, без инфраструктуры |
| Мердж-конфликты между командами в одном файле | Конфликтов нет |
| Непонятно, кому принадлежит класс | У каждого класса есть владелец |

### Как SRP выглядит в ASP.NET Core

Реальный симптом нарушения — «жирный контроллер»:

```csharp
// ПЛОХО: контроллер знает про HTTP, валидацию, бизнес-правила, БД и почту
[HttpPost]
public async Task<IActionResult> Create(OrderRequest request)
{
    if (request.Items.Count == 0) return BadRequest("Пустой заказ");
    if (request.Items.Sum(i => i.Price * i.Qty) > 1_000_000)
        return BadRequest("Слишком большой заказ");

    var customer = await _db.Customers.FindAsync(request.CustomerId);
    if (customer is null) return NotFound();
    if (customer.IsBlocked) return Forbid();

    var discount = customer.OrdersCount > 10 ? 0.1m : 0m;
    var total = request.Items.Sum(i => i.Price * i.Qty) * (1 - discount);

    var order = new Order { CustomerId = customer.Id, Total = total };
    _db.Orders.Add(order);
    await _db.SaveChangesAsync();

    await _smtp.SendAsync(customer.Email, "Заказ принят", $"Сумма: {total}");
    return Ok(order.Id);
}
```

```csharp
// ХОРОШО: контроллер занимается только HTTP
[HttpPost]
public async Task<IActionResult> Create(OrderRequest request, CancellationToken ct)
{
    var result = await _createOrder.HandleAsync(request.ToCommand(), ct);
    return result.Match<IActionResult>(
        id => Ok(id),
        error => Problem(title: error.Title, statusCode: error.StatusCode));
}
```

Бизнес-правила уезжают в `CreateOrderHandler`, валидация — в валидатор ([[Model binding и валидация]] или [[FluentValidation]]), письмо — в доменное событие или фоновый обработчик ([[Доменные события]], [[Background services и IHostedService]]).

### Критика SRP

- **Формулировка расплывчата.** «Одна причина для изменения» не операционализируется: два человека посмотрят на класс и насчитают разное число причин. Практическая замена — задать вопрос «кто заказчик правок этого файла?». Если ответ содержит «и», это сигнал.
- **Слепое применение плодит анемию.** Разнесение данных и всего поведения по разным классам приводит к процедурному коду в объектной обёртке. Домен, где вся логика в `*Service`, а сущности — мешки свойств, формально проходит SRP и при этом плох.
- **Классы-однометодники — не самоцель.** `IOrderTotalCalculatorFactoryProvider` — не SRP, а карго-культ. Смотри [[DRY, KISS, YAGNI и когда они врут]].

---

## O — Open/Closed Principle

> [!quote] Формулировка
> «Программные сущности должны быть открыты для расширения, но закрыты для изменения» (Бертран Мейер, 1988).

Оригинальная формулировка Мейера была про наследование: класс закрыт, потому что скомпилирован и используется, но открыт, потому что от него можно унаследоваться. Современная (полиморфная) трактовка — про абстракции: новое поведение добавляется новой реализацией, старый код не трогается.

### Нарушение

```csharp
// ПЛОХО: каждый новый способ оплаты — правка работающего switch
public sealed class PaymentProcessor
{
    public Task<Receipt> PayAsync(Payment p) => p.Method switch
    {
        "card"   => ChargeCardAsync(p),
        "sbp"    => ChargeSbpAsync(p),
        "wallet" => ChargeWalletAsync(p),
        _ => throw new NotSupportedException(p.Method)
    };
}
```

Симптом: добавление криптоплатежей требует изменить файл, покрытый тестами и работающий в проде. Риск регрессии в коде, который вообще не должен был меняться. Плюс такой `switch` обычно не один — рядом живут `switch` для комиссии, для отображения, для возврата, и их забывают синхронизировать.

### Рефакторинг

```csharp
// ХОРОШО: новая стратегия = новый файл, старый код не трогаем
public interface IPaymentMethod
{
    string Code { get; }
    Task<Receipt> ChargeAsync(Payment payment, CancellationToken ct);
}

public sealed class CardPayment : IPaymentMethod
{
    public string Code => "card";
    public Task<Receipt> ChargeAsync(Payment p, CancellationToken ct) => /* ... */;
}

public sealed class PaymentProcessor(IEnumerable<IPaymentMethod> methods)
{
    // Собираем словарь один раз; FrozenDictionary оптимален для read-only lookup
    private readonly FrozenDictionary<string, IPaymentMethod> _byCode =
        methods.ToFrozenDictionary(m => m.Code, StringComparer.Ordinal);

    public Task<Receipt> PayAsync(Payment p, CancellationToken ct) =>
        _byCode.TryGetValue(p.Method, out var m)
            ? m.ChargeAsync(p, ct)
            : throw new NotSupportedException(p.Method);
}

// Program.cs — регистрируем все реализации разом
builder.Services.AddSingleton<IPaymentMethod, CardPayment>();
builder.Services.AddSingleton<IPaymentMethod, SbpPayment>();
builder.Services.AddSingleton<PaymentProcessor>();
```

Обрати внимание: `IEnumerable<IPaymentMethod>` внедряется контейнером автоматически — это штатная возможность DI в ASP.NET Core ([[Dependency Injection: контейнер ASP.NET Core]]). Про `FrozenDictionary` — [[Frozen и Immutable коллекции]].

### Когда OCP работает против тебя

OCP платит только на той оси, где вариация действительно происходит. Если ты угадал ось — расширение стоит копейки. Если не угадал — ты построил абстракцию, которая мешает.

```csharp
// Гипотетический OCP «на будущее»: три интерфейса, одна реализация каждого,
// вариаций так и не появилось. Чистый минус: навигация, косвенность, тесты моков.
public interface IOrderNumberGenerationStrategy { string Next(); }
public interface IOrderNumberFormatter { string Format(string raw); }
public interface IOrderNumberValidator { bool IsValid(string number); }
```

Практическое правило: **не абстрагируй ось, по которой ещё ни разу не было вариации**. Дождись второго варианта. Смотри «Rule of Three» в [[DRY, KISS, YAGNI и когда они врут]].

### Критика OCP

- Мейеровская трактовка через наследование сегодня почти мертва: наследование как способ расширения приносит fragile base class problem ([[Композиция vs наследование]]).
- «Никогда не менять существующий код» — недостижимо и вредно. Рефакторинг — это изменение кода, и он полезен. OCP означает «не должно быть нужды править устоявшуюся логику ради добавления однотипного варианта», а не «код неприкосновенен».
- В домене с закрытым множеством вариантов (три статуса заказа, которые никогда не станут четырьмя) `switch` с pattern matching лучше иерархии: он даёт исчерпывающую проверку компилятором и держит логику в одном читаемом месте. Смотри [[Pattern matching]].

---

## L — Liskov Substitution Principle

> [!quote] Формулировка (Барбара Лисков, 1987, в вольном переводе)
> Если S — подтип T, то объекты типа T в программе можно заменить объектами типа S, не изменив ни одного желаемого свойства программы.

LSP — единственный принцип SOLID с формальным содержанием. Он говорит о **поведенческом подтипировании**: наследник обязан соблюдать контракт предка, а не только его сигнатуры. Компилятор проверяет сигнатуры; контракт проверять некому, поэтому нарушения LSP — это баги времени выполнения.

### Правила контракта

| Правило | Что можно наследнику |
|---|---|
| Предусловия (preconditions) | Только **ослаблять**. Нельзя требовать больше, чем базовый тип |
| Постусловия (postconditions) | Только **усиливать**. Нельзя гарантировать меньше |
| Инварианты | Обязан сохранять все инварианты базового типа |
| Исключения | Может бросать только те типы, что объявлены/подразумеваются базовым контрактом |
| История (history rule) | Не может делать изменяемым то, что в базовом типе неизменяемо |

### Нарушение: усиление предусловия

```csharp
public class FileStorage
{
    // Контракт: сохраняет любой непустой массив байт
    public virtual void Save(string name, byte[] content)
    {
        if (content.Length == 0) throw new ArgumentException(nameof(content));
        File.WriteAllBytes(name, content);
    }
}

// ПЛОХО: наследник требует БОЛЬШЕ, чем базовый тип
public class S3Storage : FileStorage
{
    public override void Save(string name, byte[] content)
    {
        // Новое предусловие, о котором вызывающий код не знает
        if (content.Length > 5 * 1024 * 1024)
            throw new ArgumentException("S3 не принимает больше 5 МБ");
        // ...
    }
}
```

Код, написанный против `FileStorage`, внезапно падает, когда в него подсунули `S3Storage`. Формально всё компилируется, фактически подстановка невозможна.

### Нарушение: ослабление постусловия и history rule

Каноничный пример — `Square : Rectangle`. Обычно его объясняют «квадрат не прямоугольник», и это неверно: в геометрии квадрат — прямоугольник. Проблема в **изменяемости**.

```csharp
public class Rectangle
{
    public virtual int Width { get; set; }
    public virtual int Height { get; set; }
}

// ПЛОХО
public class Square : Rectangle
{
    public override int Width { set { base.Width = base.Height = value; } }
    public override int Height { set { base.Width = base.Height = value; } }
}

// Клиентский код, написанный против контракта Rectangle:
static void Resize(Rectangle r)
{
    r.Width = 4;
    r.Height = 5;
    // Постусловие Rectangle: после присваивания Width равен присвоенному
    Debug.Assert(r.Width == 4);   // Для Square падает: Width == 5
}
```

Контракт `Rectangle` неявно содержит «Width и Height независимы». `Square` его нарушает. Если бы прямоугольник был иммутабельным (`WithWidth` возвращает новый объект), проблема бы исчезла — ещё один аргумент в пользу [[Иммутабельность как приём проектирования]].

### Нарушение из BCL: `Array` как `IList<T>`

```csharp
IList<int> list = new[] { 1, 2, 3 };
list.Add(4);       // NotSupportedException во время выполнения
```

Массив реализует `IList<T>`, но не поддерживает половину контракта. Это реальный, живущий в BCL с 2005 года пример нарушения LSP — его допустили ради удобства. Отсюда же `ReadOnlyCollection<T>` и потребность в `IReadOnlyList<T>`.

Второй пример — `Stream`: `Position`, `Length`, `Seek` бросают `NotSupportedException` у сетевых потоков. BCL пришлось добавить `CanSeek`/`CanRead`/`CanWrite` — это признак того, что интерфейс слишком широк (см. ISP ниже).

### Рефакторинг

```csharp
// ХОРОШО: разделяем контракты вместо того, чтобы наследник их нарушал
public interface IReadableStorage
{
    Task<byte[]> ReadAsync(string name, CancellationToken ct);
}

public interface IWritableStorage
{
    /// <summary>Максимальный размер объекта. Часть контракта, а не сюрприз.</summary>
    long MaxObjectSize { get; }
    Task WriteAsync(string name, ReadOnlyMemory<byte> content, CancellationToken ct);
}
```

Ограничение стало явной частью контракта. Клиент может его проверить, а не поймать исключение в проде.

### Как проверять LSP на практике

Пиши тесты **против абстракции**, а не против реализации, и прогоняй их на всех наследниках. В xUnit это делается абстрактным базовым классом теста:

```csharp
public abstract class StorageContractTests
{
    protected abstract IWritableStorage CreateSut();

    [Fact]
    public async Task Write_Then_Read_Returns_Same_Bytes()
    {
        var sut = CreateSut();
        // один и тот же тест обязан пройти для файловой системы, S3 и in-memory
    }
}

public sealed class FileStorageTests : StorageContractTests
{
    protected override IWritableStorage CreateSut() => new FileStorage(TempDir());
}
```

Это единственный надёжный способ поймать нарушение LSP автоматически. Подробнее — [[xUnit: параметризация и фикстуры]].

### Критика LSP

- В строгой формальной форме принцип неприменим к большинству промышленного кода: у нас нет записанных пред- и постусловий, значит нечего проверять. Contracts в .NET (Code Contracts) умерли.
- Многие «нарушения LSP» на код-ревью — это на самом деле нарушения ISP или просто плохо документированный контракт.
- LSP касается и интерфейсов, не только наследования классов. Реализация `IRepository<T>`, которая молча игнорирует `Delete`, — нарушение LSP без единого `override`.

---

## I — Interface Segregation Principle

> [!quote] Формулировка
> «Клиентов не следует принуждать зависеть от методов, которые они не используют».

### Нарушение

```csharp
// ПЛОХО: один интерфейс на все случаи
public interface IUserService
{
    Task<User?> GetByIdAsync(Guid id, CancellationToken ct);
    Task<IReadOnlyList<User>> SearchAsync(string query, CancellationToken ct);
    Task<Guid> CreateAsync(CreateUserDto dto, CancellationToken ct);
    Task UpdateAsync(Guid id, UpdateUserDto dto, CancellationToken ct);
    Task DeleteAsync(Guid id, CancellationToken ct);
    Task<byte[]> ExportToCsvAsync(CancellationToken ct);
    Task SendWelcomeEmailAsync(Guid id, CancellationToken ct);
    Task<UserStatistics> GetStatisticsAsync(CancellationToken ct);
}
```

Цена в реальном проекте:

1. Контроллер, которому нужен один `GetByIdAsync`, зависит от всех восьми методов. Любая правка сигнатуры `ExportToCsvAsync` перекомпилирует и потенциально ломает его.
2. Юнит-тест этого контроллера вынужден мокать интерфейс из восьми методов. Мок молча возвращает `null`/`default` для остальных — тест зелёный, прод падает.
3. Невозможно сделать частичную реализацию: read-only реплика обязана «реализовать» `DeleteAsync` заглушкой, и получается нарушение LSP.

### Рефакторинг

```csharp
// ХОРОШО: интерфейсы нарезаны по потребностям клиентов
public interface IUserReader
{
    Task<User?> GetByIdAsync(Guid id, CancellationToken ct);
    Task<IReadOnlyList<User>> SearchAsync(string query, CancellationToken ct);
}

public interface IUserWriter
{
    Task<Guid> CreateAsync(CreateUserDto dto, CancellationToken ct);
    Task UpdateAsync(Guid id, UpdateUserDto dto, CancellationToken ct);
    Task DeleteAsync(Guid id, CancellationToken ct);
}

public interface IUserExporter
{
    Task<byte[]> ExportToCsvAsync(CancellationToken ct);
}

// Одна реализация может закрывать несколько интерфейсов — это нормально
public sealed class UserService : IUserReader, IUserWriter { /* ... */ }
```

Регистрация в DI без дублирования экземпляра:

```csharp
builder.Services.AddScoped<UserService>();
builder.Services.AddScoped<IUserReader>(sp => sp.GetRequiredService<UserService>());
builder.Services.AddScoped<IUserWriter>(sp => sp.GetRequiredService<UserService>());
```

### Ключевая мысль: интерфейс принадлежит клиенту

Самая ценная часть ISP — **интерфейс определяется потребителем, а не реализацией**. Правильный вопрос не «какие методы есть у моего сервиса», а «что нужно вот этому конкретному обработчику». Отсюда следует, что число методов в интерфейсе стремится к 1–3, а интерфейсы живут рядом с потребителем, а не рядом с реализацией. Это напрямую ведёт к DIP — подробно в [[Инверсия зависимостей на практике]].

### Критика ISP

- Механическое дробление даёт «интерфейсный суп»: 40 однометодных интерфейсов, каждый реализован ровно одним классом. Навигация по коду умирает.
- Часто вместо интерфейса достаточно делегата: `Func<Guid, CancellationToken, Task<User?>>`. Меньше церемонии, тот же эффект. Для одного метода делегат обычно честнее интерфейса.
- ISP иногда конфликтует с удобством клиента: `HttpClient` с двумя методами вместо `IHttpClient` на 20 членов был бы неудобен. Практичность важнее принципа.

---

## D — Dependency Inversion Principle

> [!quote] Формулировка
> 1. Модули верхних уровней не должны зависеть от модулей нижних уровней. Оба должны зависеть от абстракций.
> 2. Абстракции не должны зависеть от деталей. Детали должны зависеть от абстракций.

### Нарушение

```csharp
// ПЛОХО: доменная логика знает про SMTP, Postgres и системные часы
public sealed class SubscriptionService
{
    public void Renew(Guid userId)
    {
        using var conn = new NpgsqlConnection("Host=prod-db;...");
        conn.Open();
        // ... SQL ...

        if (DateTime.UtcNow.Day == 1) { /* особая логика для первого числа */ }

        var smtp = new SmtpClient("smtp.company.local");
        smtp.Send(/* ... */);
    }
}
```

Проблемы: нельзя протестировать без БД и почтового сервера; нельзя проверить логику «первого числа», не переводя часы; смена Postgres на что-то другое — правка домена; строка подключения захардкожена.

### Рефакторинг

```csharp
// ХОРОШО: домен зависит только от собственных абстракций
public interface ISubscriptionRepository
{
    Task<Subscription?> GetAsync(Guid userId, CancellationToken ct);
    Task SaveAsync(Subscription subscription, CancellationToken ct);
}

public interface INotificationSender
{
    Task NotifyAsync(Guid userId, NotificationKind kind, CancellationToken ct);
}

public sealed class SubscriptionService(
    ISubscriptionRepository repository,
    INotificationSender notifications,
    TimeProvider time)                     // системные часы — тоже зависимость
{
    public async Task RenewAsync(Guid userId, CancellationToken ct)
    {
        var sub = await repository.GetAsync(userId, ct)
                  ?? throw new SubscriptionNotFoundException(userId);

        sub.RenewUntil(time.GetUtcNow().AddMonths(1));

        await repository.SaveAsync(sub, ct);
        await notifications.NotifyAsync(userId, NotificationKind.Renewed, ct);
    }
}
```

`TimeProvider` — штатная абстракция времени из .NET 8+, в тестах подменяется `FakeTimeProvider` из пакета `Microsoft.Extensions.TimeProvider.Testing`. Подробнее — [[Дата и время: DateTime, DateTimeOffset, TimeProvider]].

### Инверсия — это про направление ссылки, а не про наличие интерфейса

Главное недоразумение вокруг DIP: люди добавляют интерфейс и считают, что инвертировали зависимость. Инверсия происходит только тогда, когда **интерфейс принадлежит верхнему уровню**.

```
БЕЗ инверсии (интерфейс в инфраструктуре):

   Domain ────────────────────────────► Infrastructure
                                        ├── IUserRepository
                                        └── EfUserRepository

   Домен всё равно ссылается на сборку инфраструктуры. Ничего не изменилось.


С инверсией (интерфейс в домене):

   Domain                               Infrastructure
   ├── IUserRepository  ◄────────────── EfUserRepository
   └── SubscriptionService

   Стрелка компиляционной зависимости развёрнута против направления вызова.
```

Именно это делает возможной Clean Architecture: домен не ссылается ни на что. Детальный разбор, включая случаи, когда интерфейс всё-таки правильно держать в инфраструктуре, — в [[Инверсия зависимостей на практике]].

### Критика DIP

- Инверсия ради инверсии удорожает код: интерфейс, реализация, регистрация в DI, мок в тесте — четыре места вместо одного. Для стабильных зависимостей (`System.Text.Json`, `Math`, коллекции BCL) инверсия бессмысленна: они не меняются и не мешают тестам.
- Правило проще принципа: **инвертируй волатильные зависимости** — те, что ходят по сети, в файловую систему, к часам, к генератору случайных чисел, к внешним API. Всё остальное можно вызывать напрямую.
- DIP часто путают с DI и IoC. Это три разные вещи — разобрано в [[Инверсия зависимостей на практике]].

---

## SOLID целиком: как это выглядит в одном проекте

```
src/
├── Shop.Domain/                      ← ни одной ссылки на инфраструктуру (DIP)
│   ├── Orders/
│   │   ├── Order.cs                  ← поведение внутри сущности (Tell, Don't Ask)
│   │   ├── IOrderRepository.cs       ← интерфейс принадлежит домену (DIP)
│   │   └── Discounts/
│   │       ├── IDiscountRule.cs      ← ось вариации (OCP)
│   │       ├── LoyaltyDiscount.cs
│   │       └── PromoCodeDiscount.cs
│   └── ...
├── Shop.Application/
│   ├── Orders/CreateOrderHandler.cs  ← один сценарий = один класс (SRP)
│   └── Abstractions/IClock.cs
├── Shop.Infrastructure/              ← реализует интерфейсы домена
│   ├── EfOrderRepository.cs          ← контрактные тесты гоняются и на нём (LSP)
│   └── SmtpNotificationSender.cs
└── Shop.Api/
    └── Endpoints/OrdersEndpoints.cs  ← только HTTP (SRP)
```

> [!warning] Подводные камни
> **1. SOLID применяют к коду, который не будет меняться.** Пять уровней абстракции вокруг скрипта миграции данных, который запустят один раз, — чистый убыток. Принципы окупаются только там, где изменения реально происходят. Признак «горячей» зоны — история коммитов: `git log --format=format: --name-only | sort | uniq -c | sort -rg | head -20` покажет самые изменяемые файлы. Вот их и стоит проектировать тщательно.
>
> **2. Интерфейс с одной реализацией не даёт ни OCP, ни DIP автоматически.** `IOrderService` + `OrderService`, созданные ради мока в тесте, — это не инверсия зависимостей, а налог на навигацию. Если абстракция не отражает реальную ось вариации, она только добавляет косвенность. Тестируемость лучше достигать через инверсию волатильных зависимостей, а не через зеркалирование каждого класса интерфейсом.
>
> **3. SRP путают с «одним методом на класс».** Результат — сотни микроклассов и полная потеря контекста: чтобы понять сценарий, надо открыть двенадцать файлов. Связность (cohesion) — такая же часть хорошего дизайна, как связанность; см. [[Связанность и связность (coupling, cohesion)]].
>
> **4. OCP через наследование ломает LSP.** Расширять поведение через `virtual`-методы базового класса — прямой путь к fragile base class. Предпочитай композицию и стратегии: [[Композиция vs наследование]].
>
> **5. Нарушение LSP не ловится компилятором.** Наследник, который бросает `NotSupportedException`, компилируется идеально. Единственная автоматическая защита — контрактные тесты, общие для всех реализаций интерфейса.
>
> **6. «SOLID = обязательно» на код-ревью.** Требование соблюдать принцип без объяснения, какую конкретную боль это снимает, — карго-культ. Хорошее замечание звучит как «когда добавим второй способ оплаты, придётся править этот switch и ещё два рядом — давай вынесем стратегию», а не «здесь нарушен OCP».

> [!example] Как делают в бою
> **Что реально применяют в промышленном .NET-коде:**
>
> - **SRP** — на уровне «один сценарий = один handler». CQRS с командами и запросами ([[CQRS]], [[MediatR и альтернативы]]) — это SRP, доведённый до архитектурного стиля. Контроллер/endpoint при этом не содержит логики вообще.
> - **OCP** — там, где ось вариации доказана: платёжные провайдеры, каналы уведомлений, правила скидок, экспортёры отчётов, стратегии ретраев. Реализуется через `IEnumerable<IStrategy>` в DI или keyed services ([[Keyed services и продвинутая регистрация]]).
> - **LSP** — через контрактные тесты на интерфейсы репозиториев: один и тот же набор тестов гоняется на реализации поверх реального Postgres в Testcontainers и на in-memory фейке ([[Testcontainers]]).
> - **ISP** — читающие и пишущие абстракции разделены, особенно если в системе есть реплика для чтения. `IUserReader` идёт в реплику, `IUserWriter` — в мастер.
> - **DIP** — интерфейсы инфраструктуры лежат в `Application`/`Domain`, реализации в `Infrastructure`, связывание в `Program.cs`. Проверяется архитектурным тестом, чтобы не сползло.
>
> **Чего не делают:** не оборачивают `DbContext` в `IRepository` «ради DIP» — `DbContext` уже реализует Unit of Work и `DbSet<T>` уже репозиторий; разбор в [[Repository и Unit of Work: нужны ли поверх EF Core]]. Не создают интерфейс для каждого класса. Не абстрагируют `ILogger<T>` — он уже абстракция.

---

## Вопросы с собеседований

> [!question]- Что означает «одна причина для изменения» в SRP и как её определить?
> Причина для изменения — это **актор**: человек или роль, которая заказывает правку. Мартин уточнил формулировку именно так, потому что «одна ответственность» слишком размыто. Практический способ определить: посмотреть в историю коммитов файла и на тикеты, из-за которых он менялся. Если правки приходят от бухгалтерии, от отдела логистики и от DBA — это три причины и, скорее всего, три модуля.
> Важно: SRP не запрещает классу иметь много методов. `List<T>` имеет десятки методов и не нарушает SRP, потому что все они служат одной задаче — управлению упорядоченной коллекцией, и меняются по одной причине.

> [!question]- Приведи пример нарушения LSP в стандартной библиотеке .NET.
> Самый известный — `Array`, реализующий `IList<T>`: вызов `Add`, `Remove` или `Clear` на массиве, приведённом к `IList<T>`, бросает `NotSupportedException`. Контракт `IList<T>` обещает изменяемую коллекцию, массив его не выполняет.
> Второй — `Stream`: `Seek`, `Position` и `Length` бросают `NotSupportedException` у `NetworkStream` или `DeflateStream`. BCL пришлось добавить свойства-щупы `CanSeek`, `CanRead`, `CanWrite` — это признание того, что базовый контракт слишком широк, то есть заодно и нарушение ISP.
> Ещё пример — `ReadOnlyCollection<T>`, реализующий `ICollection<T>`: `Add` бросает исключение. Именно из-за этих проблем в .NET позже появились `IReadOnlyList<T>` и `IReadOnlyCollection<T>` — сегментированные интерфейсы вместо одного широкого.

> [!question]- В чём разница между DIP, IoC и DI?
> Три разных уровня.
> **DIP (Dependency Inversion Principle)** — принцип проектирования: направление компиляционной зависимости должно быть развёрнуто, интерфейс принадлежит верхнему уровню. Это про то, в каком проекте лежит файл интерфейса.
> **IoC (Inversion of Control)** — более широкий принцип: управление потоком выполнения передаётся фреймворку («не звони нам, мы позвоним тебе»). Событийная модель, middleware-конвейер ASP.NET Core, шаблонный метод — всё это IoC без всякого DI.
> **DI (Dependency Injection)** — конкретная техника: зависимости передаются объекту извне (через конструктор, свойство, метод), а не создаются внутри. **DI-контейнер** — всего лишь инструмент автоматизации DI; DI прекрасно работает и вручную (`new Service(new Repo(conn))`).
> Можно применять DI без соблюдения DIP: если интерфейс лежит в той же сборке, что реализация, и домен на неё ссылается — инверсии нет, есть только внедрение. Подробно — [[Инверсия зависимостей на практике]].

> [!question]- Всегда ли нужно следовать SOLID? Приведи случаи, когда не нужно.
> Не всегда. SOLID оптимизирует стоимость изменения, а изменение бывает не всегда.
> Случаи, когда принципы не окупаются: одноразовые скрипты и миграции данных; прототипы и спайки; код, где ось вариации не доказана (OCP «на будущее» почти всегда оказывается не той осью); маленькие внутренние утилиты; горячие участки, где косвенность виртуального вызова и лишние аллокации стоят реальных денег ([[Аллокации и как их избегать]]).
> Обратная сторона тоже реальна: избыточное применение SOLID даёт «интерфейсный суп», сотни микроклассов, невозможность прочитать сценарий целиком. Хорошая эвристика — смотреть, где код реально менялся за последние полгода, и вкладываться в дизайн именно там.

> [!question]- Как OCP реализуется без наследования?
> Несколько способов, все предпочтительнее наследования.
> **Стратегия через интерфейс + DI**: `IEnumerable<IDiscountRule>` внедряется контейнером, новая скидка — новый класс и одна строка регистрации.
> **Делегаты**: если вариация — одна операция, `Func<Order, decimal>` вместо интерфейса. Меньше церемонии.
> **Дженерики с ограничениями**: `Process<T>(T handler) where T : IHandler` — расширение без виртуального вызова, JIT специализирует код по типу ([[Дженерики в CLR: как устроены]]).
> **Static abstract members** (C# 11) для generic math и фабрик по типу — расширение на уровне типов ([[Интерфейсы]]).
> **Композиция и декораторы**: обернуть существующее поведение, не трогая исходный класс, — кеширование, логирование, ретраи ([[Паттерны GoF: структурные]]).
> **Конвейеры**: middleware в ASP.NET Core — OCP в чистом виде: добавляешь звено, не трогая существующие ([[Middleware и конвейер обработки запроса]]).

> [!question]- Класс `OrderService` содержит методы Create, Update, Delete, GetById, Search, ExportCsv, SendEmail. Какие принципы нарушены?
> Как минимум три.
> **SRP** — минимум три актора: бизнес-правила заказа, отчётность (CSV) и коммуникации (почта). Изменение формата CSV задевает файл, в котором лежат бизнес-правила.
> **ISP** — если у этого класса есть интерфейс `IOrderService` с теми же членами, каждый клиент зависит от семи методов ради одного. Тесты вынуждены мокать всё.
> **OCP** — экспорт почти наверняка написан под один формат; добавление XLSX потребует правки класса.
> Косвенно страдает и **DIP**, потому что отправка почты и генерация файла обычно тянут за собой конкретные `SmtpClient` и работу с файловой системой прямо в этом классе.
> Рефакторинг: разделить на `CreateOrderHandler`/`UpdateOrderHandler`/..., вынести экспорт в `IOrderExporter` с реализацией на формат, а отправку письма — в обработчик доменного события.

> [!question]- Чем «квадрат не является прямоугольником» на самом деле? Почему пример с Square/Rectangle корректен только для мутабельных типов?
> Проблема не в геометрии, а в **изменяемости**. Контракт мутабельного `Rectangle` неявно включает постусловие «после `Width = 4` свойство `Width` равно 4» и инвариант «`Width` и `Height` независимы». `Square`, синхронизирующий стороны, нарушает и то, и другое — код, написанный против `Rectangle`, ломается.
> Если сделать типы иммутабельными, проблема исчезает: `Square` может иметь `Side`, а `Rectangle` — `Width`/`Height`, и оба могут реализовывать общий интерфейс `IShape { double Area { get; } }`. Никаких сеттеров, нечего нарушать. Это одна из причин, по которой иммутабельность упрощает соблюдение LSP — [[Иммутабельность как приём проектирования]].

> [!question]- Что не так с формулировкой «интерфейсы нужны для тестируемости»?
> Она подменяет причину следствием. Тестируемость — побочный эффект низкой связанности, а не цель абстракции. Если создавать интерфейс под каждый класс ради моков, получаешь: интерфейсы, которые ничего не абстрагируют; тесты, проверяющие взаимодействие с моком, а не поведение системы; хрупкие тесты, падающие при любом рефакторинге.
> Правильный подход: инвертировать нужно **волатильные** зависимости — сеть, БД, файловая система, часы, случайность, внешние API. Чистую логику тестируй напрямую, без моков. Классы без волатильных зависимостей вообще не нуждаются в интерфейсах. Смотри [[Юнит-тесты: что тестировать, а что нет]].

---

## Задачи

### Задача 1. Найди нарушения и отрефактори

```csharp
public class ReportGenerator
{
    public string Generate(int reportType, DateTime from, DateTime to)
    {
        var conn = new SqlConnection(ConfigurationManager.AppSettings["Db"]);
        var rows = conn.Query("SELECT * FROM Sales WHERE Date BETWEEN @from AND @to",
                              new { from, to });

        if (reportType == 1) return string.Join("\n", rows.Select(r => $"{r.Date};{r.Sum}"));
        if (reportType == 2) return "<html>" + string.Join("<br>", rows) + "</html>";
        if (reportType == 3) return JsonSerializer.Serialize(rows);

        throw new ArgumentException("Неизвестный тип");
    }
}
```

Перечисли нарушенные принципы и предложи структуру.

> [!success]- Решение
> **Нарушения:**
> - **SRP** — класс отвечает за доступ к данным, за конфигурацию и за три формата вывода. Три причины для изменения минимум.
> - **OCP** — новый формат = правка цепочки `if`.
> - **DIP** — прямая зависимость от `SqlConnection` и глобального `ConfigurationManager`; протестировать без БД невозможно.
> - Магические числа вместо типа; `int reportType` — control coupling (см. [[Связанность и связность (coupling, cohesion)]]).
>
> ```csharp
> // 1. Источник данных — за абстракцией (DIP)
> public interface ISalesRepository
> {
>     Task<IReadOnlyList<SaleRow>> GetAsync(DateOnly from, DateOnly to, CancellationToken ct);
> }
>
> // 2. Формат — ось вариации (OCP)
> public interface IReportFormatter
> {
>     string Format { get; }                       // "csv", "html", "json"
>     string Render(IReadOnlyList<SaleRow> rows);
> }
>
> public sealed class CsvReportFormatter : IReportFormatter
> {
>     public string Format => "csv";
>     public string Render(IReadOnlyList<SaleRow> rows) =>
>         string.Join('\n', rows.Select(r => $"{r.Date:O};{r.Sum}"));
> }
>
> // 3. Оркестрация — тонкая, без бизнес-логики (SRP)
> public sealed class ReportGenerator(
>     ISalesRepository repository,
>     IEnumerable<IReportFormatter> formatters)
> {
>     private readonly FrozenDictionary<string, IReportFormatter> _formatters =
>         formatters.ToFrozenDictionary(f => f.Format, StringComparer.OrdinalIgnoreCase);
>
>     public async Task<string> GenerateAsync(
>         string format, DateOnly from, DateOnly to, CancellationToken ct)
>     {
>         if (!_formatters.TryGetValue(format, out var formatter))
>             throw new NotSupportedException($"Формат '{format}' не поддерживается");
>
>         var rows = await repository.GetAsync(from, to, ct);
>         return formatter.Render(rows);
>     }
> }
> ```
>
> Теперь: новый формат — новый файл плюс строка регистрации; смена хранилища не трогает форматирование; каждый форматтер тестируется чистой функцией без БД.
> Обрати внимание на `DateOnly` вместо `DateTime` — тип точнее выражает намерение, и это тоже часть дизайна.

### Задача 2. Почини нарушение LSP

```csharp
public class Bird
{
    public virtual void Fly() => Console.WriteLine("Лечу");
}

public class Penguin : Bird
{
    public override void Fly() => throw new NotSupportedException("Пингвины не летают");
}

public static void MigrateAll(IEnumerable<Bird> birds)
{
    foreach (var b in birds) b.Fly();   // падает на пингвине
}
```

> [!success]- Решение
> Проблема в том, что `Fly` попал в базовый тип, хотя не является частью контракта «птица». Возможности летать — это отдельная способность, а не свойство всех птиц.
>
> ```csharp
> public abstract class Bird
> {
>     public abstract void Eat();
>     public string Species { get; }
>     protected Bird(string species) => Species = species;
> }
>
> // Способность вынесена в отдельный контракт (ISP + LSP)
> public interface IFlyingBird
> {
>     void Fly();
> }
>
> public sealed class Sparrow(string species) : Bird(species), IFlyingBird
> {
>     public override void Eat() { }
>     public void Fly() => Console.WriteLine("Лечу");
> }
>
> public sealed class Penguin(string species) : Bird(species)
> {
>     public override void Eat() { }
>     // Fly просто нет — и незачем бросать исключение
> }
>
> public static void MigrateAll(IEnumerable<Bird> birds)
> {
>     // Компилятор и pattern matching обеспечивают корректность
>     foreach (var b in birds.OfType<IFlyingBird>()) b.Fly();
> }
> ```
>
> Ключевая мысль: **если наследник вынужден бросать `NotSupportedException`, метод стоит не в том типе**. Проверка «а можно ли?» через свойство-щуп (`CanFly`) — рабочий, но худший вариант: ошибку теперь ловит не компилятор, а рантайм.

### Задача 3. Спроектируй уведомления

Требования: система шлёт уведомления по email, SMS и в Telegram. Пользователь настраивает каналы. Для критичных уведомлений — все каналы сразу. Если канал недоступен, попробовать следующий по приоритету. Через полгода добавится push.

Спроектируй так, чтобы добавление push не потребовало правок существующих файлов.

> [!success]- Решение
> ```csharp
> public interface INotificationChannel
> {
>     NotificationChannelKind Kind { get; }
>     Task<bool> TrySendAsync(Notification notification, CancellationToken ct);
> }
>
> public enum NotificationChannelKind { Email, Sms, Telegram, Push }
>
> // Каждый канал — отдельный файл, реализующий один интерфейс (SRP + OCP)
> public sealed class EmailChannel(IEmailClient client, ILogger<EmailChannel> log)
>     : INotificationChannel
> {
>     public NotificationChannelKind Kind => NotificationChannelKind.Email;
>
>     public async Task<bool> TrySendAsync(Notification n, CancellationToken ct)
>     {
>         try
>         {
>             await client.SendAsync(n.Recipient.Email, n.Subject, n.Body, ct);
>             return true;
>         }
>         catch (Exception ex)
>         {
>             log.LogWarning(ex, "Канал email недоступен для {Recipient}", n.Recipient.Id);
>             return false;
>         }
>     }
> }
>
> // Политика доставки — отдельная ось вариации, тоже за интерфейсом
> public interface IDeliveryPolicy
> {
>     IEnumerable<NotificationChannelKind> Select(
>         Notification notification, UserPreferences preferences);
> }
>
> public sealed class CriticalDeliveryPolicy : IDeliveryPolicy
> {
>     public IEnumerable<NotificationChannelKind> Select(
>         Notification n, UserPreferences p) =>
>         n.Severity == Severity.Critical
>             ? Enum.GetValues<NotificationChannelKind>()   // все каналы
>             : p.PreferredChannels;                        // по приоритету
> }
>
> public sealed class NotificationDispatcher(
>     IEnumerable<INotificationChannel> channels,
>     IDeliveryPolicy policy)
> {
>     private readonly FrozenDictionary<NotificationChannelKind, INotificationChannel>
>         _channels = channels.ToFrozenDictionary(c => c.Kind);
>
>     public async Task<bool> DispatchAsync(
>         Notification n, UserPreferences prefs, CancellationToken ct)
>     {
>         var anyDelivered = false;
>         foreach (var kind in policy.Select(n, prefs))
>         {
>             if (!_channels.TryGetValue(kind, out var channel)) continue;
>
>             if (await channel.TrySendAsync(n, ct))
>             {
>                 anyDelivered = true;
>                 if (n.Severity != Severity.Critical) break;   // достаточно одного
>             }
>         }
>         return anyDelivered;
>     }
> }
> ```
>
> Добавление push: новый файл `PushChannel.cs`, новое значение enum, одна строка `AddSingleton<INotificationChannel, PushChannel>()`. Ни один существующий файл с логикой не меняется.
> Обрати внимание, что **две** оси вариации вынесены отдельно: канал и политика выбора каналов. Если бы политика жила внутри диспетчера, требование «для VIP-клиентов сначала Telegram» заставило бы править диспетчер.

### Задача 4. Разбери, есть ли здесь нарушение SRP

```csharp
public sealed class Money : IEquatable<Money>, IComparable<Money>
{
    public decimal Amount { get; }
    public string Currency { get; }

    public Money Add(Money other) { /* ... */ }
    public Money Multiply(decimal factor) { /* ... */ }
    public string ToString(IFormatProvider provider) { /* ... */ }
    public bool Equals(Money? other) { /* ... */ }
    public int CompareTo(Money? other) { /* ... */ }
    public override int GetHashCode() { /* ... */ }
    public static Money Parse(string s, IFormatProvider provider) { /* ... */ }
}
```

Восемь публичных членов, четыре разных «темы». Нарушен ли SRP?

> [!success]- Решение
> **Нет, не нарушен.** Все члены обслуживают одну ответственность — быть значением денежной суммы. Причина для изменения одна: изменились правила работы с деньгами в системе. Арифметика, сравнение, форматирование и разбор — это не разные акторы, а разные грани одной абстракции значения.
>
> Это важный контрпример к трактовке «SRP = один метод на класс». Value object по определению собирает вокруг значения всё поведение, которое от него неотделимо. `decimal`, `DateTime`, `string` в BCL устроены точно так же и имеют десятки членов.
>
> **Что было бы нарушением:** добавить в `Money` метод `SaveToDatabase()`, `ConvertToUsdViaApiAsync()` или `RenderAsHtml()`. Вот это уже другие акторы: DBA, интеграция с курсовым сервисом, фронтенд.
>
> Практическая проверка: спроси «если изменится X, придётся ли трогать этот файл?». Для изменения правил округления денег — да, и это правильно. Для изменения схемы БД — не должно.
> Правильная реализация `Equals`/`GetHashCode`/`CompareTo` для такого типа — в [[Object: Equals, GetHashCode, ToString]] и [[Сравнение объектов: IEquatable, IComparable, компараторы]].

---

## Итог

- SOLID — это пять эвристик снижения стоимости изменения, а не законы. Все пять сводятся к «низкая связанность, высокая связность».
- **SRP**: модуль отвечает перед одним актором. Причина для изменения — это человек, который заказывает правку, а не абстрактная «ответственность».
- **OCP**: новое поведение — новым кодом, но только на той оси, где вариация действительно происходит. Абстракция без доказанной вариации — убыток.
- **LSP**: наследник обязан соблюдать контракт, а не только сигнатуры. Компилятор это не проверит — проверяют контрактные тесты, общие для всех реализаций.
- **ISP**: интерфейс определяется потребителем, а не реализацией. Отсюда узкие интерфейсы на 1–3 метода, живущие рядом с клиентом.
- **DIP**: инвертируй **волатильные** зависимости (сеть, БД, часы, файлы) и держи интерфейс в верхнем слое. Интерфейс в слое реализации — это не инверсия.
- Применяй там, где код меняется. В одноразовых скриптах, прототипах и горячих циклах SOLID не окупается.

## Связанное

- [[Связанность и связность (coupling, cohesion)]] — фундамент, из которого выводится весь SOLID
- [[Инверсия зависимостей на практике]] — DIP подробно: DIP vs IoC vs DI, где размещать интерфейсы
- [[DRY, KISS, YAGNI и когда они врут]] — вторая половина принципов проектирования и критика преждевременной абстракции
- [[Композиция vs наследование]] — почему OCP через наследование опасен
- [[Закон Деметры и Tell, Don't Ask]] — инкапсуляция поведения, антипод анемичной модели
- [[Интерфейсы]] — механика контрактов, ISP на уровне языка
- [[Абстрактные классы]] — Template Method и когда абстрактный класс уместнее интерфейса
- [[Clean Architecture]] — SOLID, поднятый на уровень структуры решения
- [[Антипаттерны и code smells]] — как выглядят нарушения в живом коде
- [[Вопросы — ООП]] — подборка для подготовки к собеседованию
