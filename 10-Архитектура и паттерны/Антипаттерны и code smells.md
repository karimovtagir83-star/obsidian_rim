---
tags: [раздел-10, антипаттерны, code-smells, рефакторинг, middle, собес, подводный-камень]
aliases: [Antipatterns, Code smells, Плохой код, Запахи кода]
---

# Антипаттерны и code smells

> [!abstract] Коротко
> Code smell — не баг, а сигнал: код работает, но что-то в нём делает будущие изменения дороже. Антипаттерн — решение, которое выглядит как паттерн, но систематически создаёт больше проблем, чем решает. Ключевой навык мидла — не заучить список запахов, а уметь оценить цену: каждый запах имеет контекст, в котором он дешевле правильного решения. Заметка построена по схеме «как выглядит → почему дорожает → как рефакторить → когда это НЕ проблема».

## Зачем это нужно

Джуниор учит список «так писать нельзя». Мидл понимает механику: *что именно* дорожает от каждого запаха — время чтения, радиус изменений, невозможность тестирования, риск регрессии. И тогда становится видно, что часть запахов в конкретном проекте безобидна.

До того как Фаулер систематизировал запахи (1999), обсуждение качества кода сводилось к «мне не нравится» против «мне нравится». Каталог запахов дал общий язык: вместо «этот класс какой-то плохой» можно сказать «здесь God Object, оси изменчивости смешаны, любая правка в расчёте скидок трогает файл, который параллельно правят четыре человека». Второе — аргумент, первое — вкусовщина. Это же превращает code review из спора в разговор о стоимости (см. [[Code review: как давать и получать]]).

> [!info] Запах не равно «переписать»
> Обнаружение запаха — вход в разговор, а не приговор. Стоимость рефакторинга бывает выше стоимости жизни с запахом. Код, который никто не трогал два года и не собирается, можно оставить некрасивым: он не мешает — его читает только компилятор.

## God Object / God Service

### Как выглядит

```csharp
// OrderService.cs — 3000 строк, 47 публичных методов, 19 зависимостей в конструкторе
public class OrderService
{
    public OrderService(
        AppDbContext db, IEmailSender email, ISmsSender sms, IPdfGenerator pdf,
        IPaymentGateway payments, IWarehouseClient warehouse, ICurrencyRates rates,
        ILoyaltyService loyalty, IAnalytics analytics, ILogger<OrderService> logger,
        IConfiguration config, ICacheService cache, ITaxCalculator tax,
        IDeliveryCostCalculator delivery, IFraudDetector fraud, IInvoiceArchive archive,
        IUserContext user, IFeatureFlags flags, IClock clock) { /* ... */ }

    public Task<Order> CreateAsync(CreateOrderDto dto) { /* 240 строк */ }
    public Task ApplyDiscountAsync(int orderId, string promo) { /* 180 строк */ }
    public Task<byte[]> GenerateInvoicePdfAsync(int orderId) { /* 120 строк */ }
    public Task SyncWithWarehouseAsync() { /* 90 строк */ }
    public Task RecalculateTaxesForYearAsync(int year) { /* 200 строк */ }
    public Task SendMarketingEmailsAsync() { /* 70 строк */ }
    // ... ещё 41 метод
}
```

### Почему больно (механика)

- **Радиус изменения = весь файл.** Правка в генерации PDF требует пересборки и повторного прогона тестов всего, что зависит от `OrderService` — а зависит от него весь проект.
- **Конфликты в git.** Один файл правят пять человек в неделю. Каждый merge — ручное разрешение.
- **Тесты становятся неподъёмными.** Чтобы протестировать `ApplyDiscountAsync`, нужно замокать 19 зависимостей, из которых 17 в этом методе не используются. На практике такие методы просто не покрывают, потому что setup теста занимает 60 строк.
- **Время загрузки контекста в голову.** Чтобы понять один метод, приходится держать в уме, что делают поля класса. 3000 строк не влезают в рабочую память.
- **DI-граф раздувается.** Каждая зависимость строится при создании `OrderService`, даже если запрос дошёл до метода, которому нужны две из девятнадцати.

### Как рефакторить

Правильная ось разбиения — **изменчивость** (что меняется вместе — живёт вместе), а не «слои ради слоёв». Типичная ошибка: разрезать `OrderService` на `OrderService` + `OrderRepository` + `OrderMapper` + `OrderValidator` — файлов стало четыре, а меняются они всё равно вместе, и радиус изменения не уменьшился.

```csharp
// Разбито по причинам изменения, а не по техническим ролям.
// Каждый класс меняется по своему поводу и имеет свой набор зависимостей.

// Меняется, когда меняются правила оформления заказа
public sealed class OrderPlacement(
    AppDbContext db, IPaymentGateway payments, IFraudDetector fraud, TimeProvider clock)
{
    public async Task<Result<OrderId>> PlaceAsync(PlaceOrder cmd, CancellationToken ct) { /* ... */ }
}

// Меняется, когда меняется маркетинговая политика скидок — свой темп релизов
public sealed class DiscountPolicy(ILoyaltyService loyalty, ICurrencyRates rates)
{
    public Money Apply(Money subtotal, CustomerTier tier, PromoCode? promo) { /* ... */ }
}

// Меняется, когда бухгалтерия меняет форму документа
public sealed class InvoiceDocumentBuilder(IPdfGenerator pdf, ITaxCalculator tax)
{
    public byte[] Build(Order order) { /* ... */ }
}

// Фоновая синхронизация — вообще другой жизненный цикл (см. Background services и IHostedService)
public sealed class WarehouseSyncJob(IWarehouseClient warehouse, AppDbContext db) : BackgroundService { /* ... */ }
```

Практический приём для поиска осей: выпиши для каждого метода набор используемых полей класса. Методы группируются в кластеры по пересечению полей — это и есть будущие классы. Кластер, который использует одно поле из девятнадцати, почти наверняка отдельный класс.

### Когда это НЕ проблема

- Класс большой, но **однородный** — например, 800 строк арифметики над `decimal` без внешних зависимостей. Строк много, но ось изменчивости одна, и разрезать его — только запутать.
- Сгенерированный код (Source Generators, клиенты OpenAPI, `*.Designer.cs`). Его никто не читает.
- Код на выброс: скрипт миграции, который отработает один раз.
- Мало зависимостей, много строк из-за `switch` по 40 вариантам — это не God Object, это таблица решений. Иногда её честнее оставить одним куском, чем размазать по 40 классам-стратегиям.

## Anemic Domain Model

### Как выглядит

```csharp
// Сущность — мешок свойств с публичными сеттерами
public class Order
{
    public int Id { get; set; }
    public OrderStatus Status { get; set; }
    public decimal Total { get; set; }
    public DateTime? PaidAt { get; set; }
    public List<OrderLine> Lines { get; set; } = [];
}

// Вся логика — в сервисе
public class OrderService
{
    public void Pay(Order order, decimal amount)
    {
        if (order.Status == OrderStatus.Cancelled) throw new InvalidOperationException("Заказ отменён");
        if (amount < order.Total) throw new InvalidOperationException("Недоплата");
        order.Status = OrderStatus.Paid;
        order.PaidAt = DateTime.UtcNow;
    }
}
```

### Почему больно — и только в одном случае

Сама по себе анемичная модель не запах. Запах появляется, когда **инвариант проверяется не в одном месте**. Через полгода в проекте есть `OrderService.Pay`, `AdminController.MarkAsPaid` и `PaymentWebhookHandler.Handle`, и в двух из трёх проверка «заказ отменён» забыта. Модель не может себя защитить: `order.Status = OrderStatus.Paid` доступно любому коду в сборке.

Механика подорожания: количество мест, где нужно продублировать правило, растёт линейно с числом входных точек, а вероятность пропустить одно — растёт быстрее, чем линейно. Каждая новая точка входа — новый шанс создать заказ в невозможном состоянии, а невозможное состояние в БД лечится уже не кодом, а SQL-скриптом на проде.

### Как рефакторить

```csharp
public class Order
{
    private readonly List<OrderLine> _lines = [];

    public int Id { get; private set; }
    public OrderStatus Status { get; private set; }
    public Money Total { get; private set; }
    public DateTimeOffset? PaidAt { get; private set; }
    public IReadOnlyList<OrderLine> Lines => _lines;   // наружу — только чтение

    private Order() { }   // для EF Core

    // Единственный способ перевести заказ в оплаченный — этот метод.
    // Инвариант физически невозможно обойти: сеттеры закрыты.
    public Result Pay(Money amount, DateTimeOffset now)
    {
        if (Status is OrderStatus.Cancelled)
            return Result.Fail("Нельзя оплатить отменённый заказ");
        if (amount < Total)
            return Result.Fail($"Недоплата: нужно {Total}, получено {amount}");

        Status = OrderStatus.Paid;
        PaidAt = now;
        return Result.Ok();
    }
}
```

Про `Result` вместо исключений — [[Result pattern вместо исключений]], про то, как EF Core работает с приватными сеттерами и полями-коллекциями — [[EF Core: конфигурация модели и Fluent API]].

### Когда это НЕ проблема (важно)

**Для CRUD анемичная модель нормальна и дешевле.** Если сущность — справочник стран, а «бизнес-логика» состоит из «поле обязательно, длина до 100 символов», то богатая модель добавляет два уровня косвенности и ноль защиты: валидацию всё равно делает [[Model binding и валидация]] или [[FluentValidation]], а инварианта, который можно нарушить, просто нет.

Признаки, что анемичная модель — правильный выбор здесь:
- на сущность приходится 0–1 правило, и это правило — валидация формата, а не переход состояния;
- нет конечного автомата статусов;
- сущность не участвует в согласованности с другими сущностями;
- 90 % кода — маппинг DTO в сущность и обратно.

Признаки, что пора в богатую модель:
- есть статусы и запрещённые переходы между ними;
- одно и то же правило уже встретилось в двух разных сервисах;
- в БД периодически находят записи в невозможном состоянии;
- в код-ревью регулярно всплывает «а ты проверил, что заказ не отменён?».

Практичный подход: анемичные CRUD-модули и богатый домен в одном проекте — это нормально. Насильно применять DDD ко всему — само по себе over-engineering. Подробнее про агрегаты, инварианты и Value Object — [[DDD: тактические паттерны]].

## Service Locator

### Как выглядит

```csharp
public class ReportService(IServiceProvider services)
{
    public async Task<byte[]> BuildAsync(ReportKind kind)
    {
        // Зависимость достаётся из контейнера внутри метода
        var db = services.GetRequiredService<AppDbContext>();
        var pdf = services.GetRequiredService<IPdfGenerator>();
        if (kind == ReportKind.Yearly)
            services.GetRequiredService<IArchiveClient>().Touch();   // а про эту вообще никто не знает
        // ...
    }
}
```

### Почему больно

- **Зависимости невидимы.** Конструктор говорит «мне нужен контейнер», то есть «мне нужно всё». Понять, что реально требуется классу, можно только прочитав каждую строку каждого метода. Публичный контракт врёт.
- **Ломается валидация графа при старте.** ASP.NET Core с `ValidateOnBuild`/`ValidateScopes` проверяет, что все конструкторные зависимости разрешимы, при построении хоста. Зависимости, добытые через `GetRequiredService` в рантайме, не проверяются никогда: ошибка регистрации всплывёт в 3 часа ночи при первом вызове редкого метода, а не в CI. См. [[Dependency Injection: контейнер ASP.NET Core]].
- **Прячет ошибки времени жизни.** Разрешение scoped-сервиса из root-провайдера — классический источник [[Captive dependency и типичные ошибки DI]]: `DbContext`, полученный из корневого провайдера, живёт всё время работы приложения, накапливает change tracker и утекает.
- **Тесты требуют собирать контейнер.** Вместо передачи двух моков в конструктор приходится настраивать `ServiceCollection`, то есть юнит-тест превращается в полуинтеграционный.

### Как рефакторить

```csharp
// Зависимости явные — контракт честный, граф проверяется при старте
public sealed class ReportService(
    AppDbContext db,
    IPdfGenerator pdf,
    IArchiveClient archive)
{
    public async Task<byte[]> BuildAsync(ReportKind kind, CancellationToken ct) { /* ... */ }
}
```

Если реализация выбирается по ключу в рантайме — не тащите контейнер, возьмите фабрику или keyed services:

```csharp
// Вариант 1: явная фабрика — зависимость видна, тип известен
public interface IReportBuilderFactory { IReportBuilder Create(ReportKind kind); }

// Вариант 2: keyed services (см. Keyed services и продвинутая регистрация)
public sealed class ReportService([FromKeyedServices("yearly")] IReportBuilder yearly) { /* ... */ }

// Вариант 3: инжектим все реализации и выбираем — граф проверяем, зависимость явная
public sealed class ReportService(IEnumerable<IReportBuilder> builders)
{
    public IReportBuilder Pick(ReportKind kind) => builders.First(b => b.Supports(kind));
}
```

### Когда это НЕ проблема

- **Внутри инфраструктурного кода**, который по определению работает с контейнером: middleware, реализация `IHostedService`, который создаёт scope на каждую итерацию, фабрики, интеграция со сторонним фреймворком, который не умеет в конструкторную инъекцию.

```csharp
// Это не Service Locator, а корректное создание scope в singleton-хосте
public sealed class OutboxProcessor(IServiceScopeFactory scopeFactory) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            using var scope = scopeFactory.CreateScope();
            var handler = scope.ServiceProvider.GetRequiredService<OutboxHandler>();
            await handler.RunOnceAsync(ct);
        }
    }
}
```

- В **композиционном корне** (`Program.cs`) — там разрешать сервисы из провайдера нормально, это и есть его работа.

## Primitive obsession

### Как выглядит

```csharp
public Task TransferAsync(string fromIban, string toIban, decimal amount, string currency);

// Вызов, который компилируется и делает не то
await TransferAsync(to, from, 100m, "usd");        // аргументы перепутаны, валюта в нижнем регистре
await TransferAsync(a, b, -100m, "USD");           // отрицательная сумма
var total = usdAmount + eurAmount;                 // сложили доллары с евро, компилятор не против
```

### Почему больно

- **Типовая система не работает.** Все параметры — `string`/`decimal`, значит перестановка аргументов не ловится компилятором, а ловится в проде.
- **Валидация дублируется.** Проверка «это похоже на email» появляется в контроллере, в сервисе, в фоновом импортёре — и в трёх местах разная.
- **Инварианты недостижимы.** `decimal amount` не может гарантировать «не отрицательное», `string currency` — «код по ISO 4217». А сложение сумм в разных валютах — арифметически корректная, семантически катастрофическая операция.

### Как рефакторить

```csharp
// Value Object: неизменяемый, валидируется в момент создания, сравнивается по значению
public readonly record struct Currency
{
    public string Code { get; }

    private Currency(string code) => Code = code;

    public static Result<Currency> Create(string? raw) =>
        raw is { Length: 3 } s && s.All(char.IsAsciiLetterUpper)
            ? Result.Ok(new Currency(s))
            : Result.Fail($"Некорректный код валюты: '{raw}'");

    public override string ToString() => Code;
}

public readonly record struct Money(decimal Amount, Currency Currency)
{
    public static Money operator +(Money a, Money b) =>
        a.Currency == b.Currency
            ? new Money(a.Amount + b.Amount, a.Currency)
            // Сложение разных валют невозможно выразить — ошибка ловится сразу
            : throw new InvalidOperationException($"Нельзя сложить {a.Currency} и {b.Currency}");

    public static Money operator -(Money a, Money b) => a + new Money(-b.Amount, b.Currency);
}

// Сигнатура, в которой перепутать аргументы невозможно
public Task TransferAsync(Iban from, Iban to, Money amount, CancellationToken ct);
```

`readonly record struct` даёт бесплатно `Equals`/`GetHashCode`/`ToString` по значению и не аллоцирует на куче. Как такие типы маппятся в EF Core 10 (complex types) — [[EF Core: complex types и JSON-колонки]]; проектирование Value Object — [[DDD: тактические паттерны]]; почему иммутабельность здесь принципиальна — [[Иммутабельность как приём проектирования]].

### Когда это НЕ проблема

- **На границе процесса.** DTO, который приходит из HTTP или уходит в очередь, содержит `string`/`decimal` — так и должно быть: это формат обмена. Value Object создаётся сразу за границей, при разборе.
- **Одно-два использования.** Заводить `CustomerNote` вокруг `string`, который просто пишется в лог, — лишний тип.
- **Скрипты, миграции, тесты.** Там короткая жизнь и нет вызывающих.
- **Горячий путь с миллионами операций**, где даже struct-обёртка с валидацией измеримо дороже. Сначала измерьте ([[Как измерять: BenchmarkDotNet]]), потом упрощайте.

## Feature envy

### Как выглядит

```csharp
public class ShippingCalculator
{
    public decimal Calculate(Order order)
    {
        // Метод завидует чужому объекту: пять обращений к свойствам Order и его частям
        var weight = order.Lines.Sum(l => l.Product.Weight * l.Quantity);
        var zone = order.DeliveryAddress.Country == "UZ" ? 1 : 2;
        var express = order.DeliveryOptions.IsExpress;
        var discountable = order.Customer.Tier == CustomerTier.Gold;
        var bulky = order.Lines.Any(l => l.Product.Dimensions.IsOversized);
        return (weight * zone * (express ? 2 : 1)) * (discountable ? 0.9m : 1m) + (bulky ? 5m : 0m);
    }
}
```

### Почему больно

Логика физически лежит в одном классе, а данные — в другом. Каждое изменение структуры `Order` (переименовали свойство, вынесли адрес, поменяли модель линий) ломает `ShippingCalculator`, хотя правила доставки не менялись. Цепочки вида `order.Lines[i].Product.Dimensions` привязывают вызывающего к внутреннему устройству двух-трёх уровней объектов — связность (cohesion) низкая, связанность (coupling) высокая, см. [[Связанность и связность (coupling, cohesion)]].

### Как рефакторить

Переместить поведение к данным. Каждый объект отвечает на вопрос про себя, а не отдаёт свои кишки наружу:

```csharp
public class Order
{
    public Weight TotalWeight() => _lines.Aggregate(Weight.Zero, (acc, l) => acc + l.Weight());
    public bool HasOversizedItems() => _lines.Any(l => l.IsOversized());
    public DeliveryZone Zone() => DeliveryAddress.Zone();
}

public class OrderLine
{
    public Weight Weight() => Product.Weight * Quantity;
    public bool IsOversized() => Product.Dimensions.IsOversized;
}

public class ShippingCalculator
{
    // Теперь тут только правила доставки. Внутренности Order не важны.
    public Money Calculate(Order order) =>
        _tariff.For(order.Zone())
               .Times(order.TotalWeight())
               .Plus(order.HasOversizedItems() ? Money.Uzs(5) : Money.Zero);
}
```

### Когда это НЕ проблема

- **Отчёты, проекции, экспорт.** Код, который собирает CSV из десяти свойств сущности, обязан читать десять свойств. Переносить формирование CSV в домен — хуже.
- **Маппинг** (сущность в DTO) по определению «завидует» — так и надо.
- **Объект — намеренно тупая структура данных**: DTO, конфиг, ответ внешнего API. У них нет поведения, и добавлять его не нужно.

## Длинный метод и глубокая вложенность

### Как выглядит

```csharp
public async Task<IActionResult> Checkout(CheckoutDto dto)
{
    if (dto != null)
    {
        if (ModelState.IsValid)
        {
            var user = await _db.Users.FindAsync(dto.UserId);
            if (user != null)
            {
                if (!user.IsBlocked)
                {
                    var cart = await _db.Carts.Include(c => c.Items).FirstOrDefaultAsync(c => c.UserId == user.Id);
                    if (cart != null && cart.Items.Count > 0)
                    {
                        foreach (var item in cart.Items)
                        {
                            if (item.Quantity > 0)
                            {
                                var stock = await _warehouse.GetStockAsync(item.ProductId);
                                if (stock >= item.Quantity)
                                {
                                    // полезное действие где-то на восьмом уровне вложенности
                                }
                                else { return BadRequest("Нет на складе"); }
                            }
                        }
                    }
                    else { return BadRequest("Корзина пуста"); }
                }
                else { return Forbid(); }
            }
            else { return NotFound(); }
        }
        else { return BadRequest(ModelState); }
    }
    return BadRequest();
}
```

### Почему больно

Каждый уровень вложенности — ещё одно условие, которое читатель должен держать в голове одновременно. На пятом уровне человек уже не знает, при каких условиях выполняется строка перед ним. Цикломатическая сложность (число независимых путей) растёт мультипликативно: восемь вложенных `if` — до 256 путей, полное покрытие тестами недостижимо. И `else`-ветки отъезжают на 40 строк от своего `if`, поэтому при правке легко приписать обработку не к тому условию.

### Как рефакторить

Guard clauses (ранние выходы) выпрямляют поток: сначала все причины не продолжать, затем — основной сценарий на нулевом уровне вложенности. Подробнее — [[Когда исключение — плохой выбор]].

```csharp
public async Task<Results<Ok<CheckoutResponse>, BadRequest<ProblemDetails>, NotFound>> Checkout(
    CheckoutDto dto, CancellationToken ct)
{
    var user = await _db.Users.FindAsync([dto.UserId], ct);
    if (user is null) return TypedResults.NotFound();
    if (user.IsBlocked) return Problem("Пользователь заблокирован");

    var cart = await _carts.LoadAsync(user.Id, ct);
    if (cart is null or { IsEmpty: true }) return Problem("Корзина пуста");

    var stock = await _warehouse.CheckAsync(cart.ProductQuantities(), ct);
    if (stock is StockResult.Insufficient(var product))
        return Problem($"Недостаточно товара {product}");

    var order = await _placement.PlaceAsync(cart, ct);
    return TypedResults.Ok(CheckoutResponse.From(order));
}
```

Приёмы: guard clause вместо вложенного `if`; извлечение метода (`_carts.LoadAsync`, `cart.ProductQuantities()`); замена цепочки проверок на pattern matching (`is null or { IsEmpty: true }`, деконструкция в `is StockResult.Insufficient(var p)`) — см. [[Pattern matching]]. Инверсия условия (`if (x != null) { ... }` в `if (x is null) return;`) убирает уровень вложенности механически, без изменения семантики.

### Когда это НЕ проблема

- **Линейный метод на 120 строк без ветвлений** — например, конфигурация модели EF Core или регистрация 60 сервисов. Длина есть, сложности нет. Дробить на `ConfigurePart1/Part2` только хуже.
- **`switch` на 30 вариантов** в парсере или конечном автомате: цикломатическая сложность высокая, читаемость отличная.
- **Оптимизированный горячий метод**, где инлайн ручной и намеренный. Такие места помечают комментарием с бенчмарком, иначе следующий разработчик «отрефакторит» и потеряет 30 % производительности.

## Спекулятивная общность

### Как выглядит

```csharp
// Всегда в паре, реализация одна, никогда не будет второй
public interface IUserService
{
    Task<UserDto?> GetAsync(int id);
    Task<int> CreateAsync(CreateUserDto dto);
}

public class UserService : IUserService { /* единственная реализация */ }

// А ещё абстрактный базовый класс "на будущее", от которого наследуется один класс,
// и hook-метод, который никто не переопределяет
public abstract class BaseProcessor<TIn, TOut, TContext> where TContext : IProcessingContext
{
    protected virtual void OnBeforeProcess(TContext ctx) { }   // 0 переопределений
    protected virtual void OnAfterProcess(TContext ctx) { }    // 0 переопределений
}
```

### Почему больно

- **Навигация ломается.** Ctrl+Click по методу ведёт в интерфейс, а не в код. В большом проекте это плюс один шаг на каждое чтение — умножьте на количество чтений в день.
- **Абстракция врёт.** Интерфейс, выведенный из одной реализации, — это не абстракция, а её копия с ключевым словом `interface`. Он повторяет все особенности реализации, включая случайные. Когда появится вторая реализация, интерфейс придётся переделывать — то есть «задел на будущее» ничего не сэкономил.
- **Дженерики с тремя параметрами и constraints** делают сообщения об ошибках компилятора нечитаемыми и блокируют вывод типов.
- **Мёртвые точки расширения** приходится поддерживать: их видно, значит кто-то попробует использовать, значит нужны тесты и документация.

### Как рефакторить

Удалить. Класс регистрируется сам на себя, конструкторная инъекция работает по конкретному типу:

```csharp
builder.Services.AddScoped<UserService>();   // без интерфейса

public sealed class UsersEndpoint(UserService users) { /* ... */ }
```

### Когда интерфейс реально нужен

1. **Граница процесса или внешний ресурс:** HTTP-клиент чужого API, платёжный шлюз, почта, файловое хранилище, часы. Здесь интерфейс — не «на будущее», а способ не звонить в банк из юнит-теста.
2. **Подмена в тесте невозможна иначе:** класс `sealed` без виртуальных членов, или его создание тянет соединение с сетью. (Если класс не sealed и логика чистая — часто дешевле вообще не мокать и тестировать реальный объект.)
3. **Реализаций уже больше одной** — не «будет», а есть: `SqlServerRates` и `FixedRatesForTests` не считаются, а вот `SmsNotifier`/`EmailNotifier`/`PushNotifier` — считаются.
4. **Публичный API библиотеки**, где интерфейс — обещание совместимости для внешних потребителей.
5. **Декорирование:** кеширование, retry, логирование поверх сервиса требуют общего типа. Хотя в .NET это часто решается через `IHttpClientFactory` и `DelegatingHandler` или [[Устойчивость: retry, circuit breaker, Polly]] без своих интерфейсов.

## Over-engineering и архитектурная астронавтика

### Как выглядит

```csharp
// Свой ORM поверх EF Core — "чтобы можно было легко сменить EF"
public interface IUnitOfWork : IDisposable
{
    IRepository<T> Repository<T>() where T : class, IEntity;
    Task<int> CommitAsync();
}

public interface IRepository<T> where T : class
{
    IQueryable<T> Query();                       // абстракция протекла: наружу торчит IQueryable
    Task<T?> GetByIdAsync(object id);
    void Add(T entity);
    Task<IEnumerable<T>> FindAsync(ISpecification<T> spec);
}

// Свой DI-контейнер, потому что "встроенный слишком простой"
public interface IMyContainer { object Resolve(Type t, string? tag, LifetimeScope scope); }

// Конфигурируемость, которой никто не пользуется
public class OrderProcessingOptions
{
    public bool EnableLegacyMode { get; set; } = false;     // всегда false
    public int StrategyVersion { get; set; } = 2;           // всегда 2
    public string CalculationEngine { get; set; } = "v2";   // строкой, с switch внутри
}
```

### Почему больно

- **Абстракция над абстракцией.** `DbContext` — это уже Unit of Work, `DbSet<T>` — уже Repository. Обёртка добавляет слой, который не добавляет возможностей, зато отрезает половину EF Core: `Include`, split queries, `ExecuteUpdateAsync`, отслеживание, `AsNoTracking`. Через полгода в интерфейс дописывают `IQueryable<T> Query()`, чтобы вернуть отрезанное, — и абстракция становится честной ложью. Разбор с обеих сторон — [[Repository и Unit of Work: нужны ли поверх EF Core]].
- **Обещание «легко сменить БД» не выполняется.** Смена СУБД ломается не на API ORM, а на типах, транзакционной семантике, генерации ключей, поведении при конкурентности. Проекты, которые «на всякий случай» абстрагировались, при реальной смене всё равно переписывали слой данных.
- **Свой DI/логгер/маппер = свои баги и ноль документации.** Новый человек знает `Microsoft.Extensions.DependencyInjection`, но не знает `IMyContainer`. Onboarding дорожает навсегда.
- **Неиспользуемая конфигурируемость умножает пути исполнения.** Флаг, который всегда `false`, — это ветка кода, которую надо тестировать, но которую никто не тестирует. При этом она обязательно однажды окажется `true` в проде из-за опечатки в конфиге.
- **Стоимость «а вдруг понадобится» платится сейчас, а выгода — вероятностная.** Обычно вероятность мала, а сумма платежей велика.

### Правило трёх

Не абстрагируй по первому случаю. По второму — заметь дублирование и потерпи. На третьем — у тебя есть три реальных примера, и абстракция выведется из фактов, а не из фантазии. Два примера почти всегда дают неправильную абстракцию: невозможно отличить общее от случайного, когда точек всего две.

Обратная сторона правила: как только третий случай появился — абстрагируй, не тяни. Дублирование, которое уже втрое, начинает расходиться версиями. Про баланс — [[SOLID]].

### Когда «сложно» — правильно

- **Реальная граница с внешним миром**, где нужна подмена: платёжки, интеграции, время.
- **Требование, зафиксированное в контракте**: «должно работать на PostgreSQL и MS SQL у разных клиентов» — тогда абстракция не спекулятивная, а обязательная.
- **Стоимость ошибки высокая**: расчёты денег, медицина, ядро библиотеки на 200 потребителей. Там перестраховка дешевле инцидента.

## Короткий список остальных

### Shotgun surgery

Одно логическое изменение требует правок в 12 файлах. Симптом: добавили поле «ИНН» — правим сущность, конфигурацию EF, миграцию, DTO запроса, DTO ответа, маппер, валидатор, контроллер, тесты, документацию. Механика: связность разъехалась, знание про «ИНН» размазано. Лечение: собрать всё, что меняется вместе, рядом — это ровно то, что делает [[Vertical Slice Architecture]]. Когда НЕ проблема: часть правок неизбежна (миграция БД никуда не денется), и если файлов 3, а не 12 — это норма.

### Boolean-флаги в параметрах

```csharp
// Плохо: в точке вызова смысл потерян
await SaveAsync(order, true, false, true);

// Хорошо: параметр говорит сам за себя
await SaveAsync(order, SaveOptions.NotifyCustomer | SaveOptions.SkipValidation);
// или разные методы, если флаг переключает поведение целиком
await SaveDraftAsync(order);
await SubmitAsync(order);
```

Механика боли: `bool`-параметр обычно означает «внутри метода есть `if`, то есть два разных метода, слипшихся в один». Named arguments (`notify: true`) — дешёвый компромисс, но не решают проблемы двух сценариев в одном теле. Когда НЕ проблема: один флаг с говорящим именем в приватном методе — переживём.

### Магические числа и строки

```csharp
if (user.Status == 3) { ... }                     // что такое 3?
if (order.Total > 1000) { ... }                   // 1000 чего?
cache.Set(key, value, TimeSpan.FromMinutes(15));  // почему 15?
```

Проблема не в самом числе, а в том, что при изменении правила его надо найти во всех местах, а найти нельзя: `3` встречается в коде 400 раз. Лечение: `enum`, `const`, конфигурация через [[Options pattern и конфигурация сервисов]]. Когда НЕ проблема: `0`, `1`, `-1` в очевидном контексте, `i + 1` в цикле, размеры буфера рядом с бенчмарком.

### Проглатывание исключений

```csharp
try { await _payments.ChargeAsync(order); }
catch (Exception) { }                       // деньги не списались, никто не узнал
catch (Exception ex) { /* only */ _logger.LogError(ex.Message); }  // потерян stack trace
```

Механика: пустой `catch` превращает сбой в порчу данных — код продолжает работать, считая операцию успешной. Логирование только `ex.Message` теряет тип, стек и `InnerException`, то есть диагностика в проде невозможна. Ловите конкретные типы, логируйте объект исключения целиком (`LogError(ex, "...")`), не глотайте `OperationCanceledException` вместе со всем остальным. Подробно — [[Обработка исключений]] и [[Обработка исключений]]. Когда НЕ проблема: реально ожидаемый и обрабатываемый случай с комментарием, почему игнорируем (`// файл уже удалён другим процессом — это нормально`).

### Утечка EF-сущностей в API-контракт

```csharp
[HttpGet("{id}")]
public async Task<Order> Get(int id) => await _db.Orders.Include(o => o.Lines).FirstAsync(o => o.Id == id);
```

Три беды сразу: (1) циклические ссылки `Order.Lines[i].Order` кидают исключение сериализации или требуют `ReferenceHandler.Preserve`, портя формат ответа; (2) на запись — over-posting: клиент присылает поля, которые не должен уметь менять (`Status`, `Total`, `IsAdmin`); (3) миграция БД становится breaking change для клиентов API — переименовали колонку, сломали мобильное приложение. Лечение: отдельные DTO ответа/запроса. Когда НЕ проблема: внутренний одноразовый эндпоинт для админки, живущий один спринт. Подробнее — [[Слоистая архитектура]].

### `async void`, `.Result`, `.Wait()`

```csharp
public async void Handle(Event e) { await DoAsync(e); }   // исключение убьёт процесс
var user = GetUserAsync(id).Result;                        // риск дедлока и блокировка потока
GetUserAsync(id).Wait();                                   // то же самое
```

Механика: `async void` не даёт `Task`, поэтому исключение попадает не вызывающему, а в `SynchronizationContext`, и в ASP.NET Core валит процесс. `.Result`/`.Wait()` блокируют поток пула: под нагрузкой пул исчерпывается, latency взлетает, приложение встаёт (thread pool starvation), а в контекстах с однопоточным `SynchronizationContext` — прямой дедлок. Исключение из `.Result` ещё и заворачивается в `AggregateException`. Правило: `async void` только для обработчиков событий UI. Разбор — [[Дедлоки, async void и типичные ошибки]].

## Большая таблица: смелл, сигнал, рефакторинг, когда оставить

| Смелл | Сигнал (как заметить) | Рефакторинг | Когда оставить как есть |
|---|---|---|---|
| God Object | > 500 строк, > 7 зависимостей в конструкторе, топ файла по числу коммитов и авторов | Разбить по осям изменчивости; кластеризовать методы по используемым полям | Однородный код без зависимостей; генерированный код; большой `switch`-автомат |
| Anemic Domain Model | Одно правило встречается в 2+ сервисах; публичные сеттеры у статусов; записи в невозможном состоянии в БД | Инкапсуляция: приватные сеттеры, методы-переходы, Value Object | Чистый CRUD/справочник; нет автомата состояний; логика = валидация формата |
| Service Locator | `GetRequiredService` вне `Program.cs`, `IServiceProvider` в конструкторе бизнес-класса | Конструкторная инъекция, фабрика, keyed services, `IEnumerable<T>` | Middleware, `BackgroundService` со scope, композиционный корень |
| Primitive obsession | `string`/`decimal`/`int` в сигнатурах доменных методов; повторная валидация одного формата | Value Object (`readonly record struct`), фабричный метод с валидацией | DTO на границе процесса; 1–2 использования; измеренный горячий путь |
| Feature envy | Метод читает 3+ свойства чужого объекта; цепочки `a.B.C.D` | Переместить метод к данным; Tell, Don't Ask | Мапперы, отчёты, экспорт, DTO-структуры без поведения |
| Длинный метод / вложенность | Вложенность > 3; метод не влезает на экран; цикломатическая сложность > 10 | Guard clauses, извлечение методов, pattern matching, инверсия условий | Линейная конфигурация; большой `switch`; намеренно инлайненный горячий код |
| Спекулятивная общность | Интерфейс с одной реализацией и без тестовой подмены; `virtual` без `override`; дженерик с 3 параметрами | Удалить интерфейс, регистрировать конкретный тип | Граница процесса; несколько реализаций уже есть; публичный API; декораторы |
| Over-engineering | Свой ORM/DI/маппер; флаги конфигурации, всегда равные дефолту; абстракция над абстракцией | Удалить слой; правило трёх; вернуть родной API фреймворка | Контрактное требование мультиплатформенности; высокая цена ошибки |
| Shotgun surgery | Одно требование = правки в 8+ файлах в 4 проектах | Собрать код фичи рядом (vertical slice), поднять связность | Неизбежная часть (миграции); 2–3 файла — норма |
| Boolean-флаги | `Method(x, true, false)` в вызове | Enum-флаги, отдельные методы, named arguments | Один флаг с внятным именем в приватном методе |
| Магические числа | Литерал без имени в условии/расчёте | `const`, `enum`, Options | `0`/`1`/`-1` в очевидном контексте |
| `catch (Exception) {}` | Пустой блок; лог только `ex.Message` | Конкретные типы, `LogError(ex, ...)`, проброс наверх | Документированный ожидаемый случай с комментарием |
| EF-сущность в контракте | `Task<Order>` как тип возврата эндпоинта; `[FromBody] Order` | DTO запроса и ответа, явный маппинг | Внутренняя админка на один спринт |
| `async void` / `.Result` | `async void` вне UI-события; `.Result`/`.Wait()` в web-коде | `async Task`, `await` до самого верха | Обработчики событий UI; `Program.cs` до старта хоста |

## Как обнаруживать автоматически

Ревью не масштабируется: человек устаёт, а анализатор — нет. Всё, что можно проверить машиной, надо снять с людей и оставить им обсуждение архитектуры.

```xml
<!-- Directory.Build.props в корне решения: применится ко всем проектам -->
<Project>
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <!-- Встроенные анализаторы .NET: все категории, уровень latest -->
    <EnableNETAnalyzers>true</EnableNETAnalyzers>
    <AnalysisMode>All</AnalysisMode>
    <AnalysisLevel>latest</AnalysisLevel>
    <!-- Публичный API библиотеки: требовать документацию -->
    <GenerateDocumentationFile>true</GenerateDocumentationFile>
  </PropertyGroup>
</Project>
```

```ini
# .editorconfig — правила, релевантные запахам из этой заметки
[*.cs]
# CA2007 — отсутствие ConfigureAwait (для библиотек; в ASP.NET Core обычно выключают)
dotnet_diagnostic.CA2007.severity = none
# CA1031 — не ловить Exception без разбора
dotnet_diagnostic.CA1031.severity = warning
# CA2008/CA2012 — некорректная работа с Task/ValueTask
dotnet_diagnostic.CA2012.severity = error
# CA1062 — публичный метод не проверяет аргументы
dotnet_diagnostic.CA1062.severity = warning
# CA1848 — логирование без LoggerMessage (аллокации в горячем пути)
dotnet_diagnostic.CA1848.severity = suggestion
# CA1508 — мёртвое условие (флаг, который всегда false)
dotnet_diagnostic.CA1508.severity = warning
# IDE0058 — проигнорированное возвращаемое значение
dotnet_diagnostic.IDE0058.severity = suggestion
# Стиль: требовать явные фигурные скобки, чтобы не путать вложенность
csharp_prefer_braces = true:warning
```

Инструменты и что каждый реально ловит:

| Инструмент | Что ловит | Чего не ловит |
|---|---|---|
| Встроенные .NET-анализаторы (CAxxxx/IDExxxx) | `catch (Exception)`, `async void`, забытый `await`, мёртвые условия, сравнение строк без culture | Архитектурные запахи |
| `TreatWarningsAsErrors` | Всё, что выше, — на этапе CI, а не «когда-нибудь» | Ничего нового, но делает правила обязательными |
| Свои Roslyn-анализаторы | Проектные правила: «нет `GetRequiredService` вне `Program.cs`», «нет `DateTime.Now`», «эндпоинт не возвращает EF-сущность» | Требуют написания и поддержки — см. [[Roslyn-анализаторы и свои правила]] |
| SonarQube / SonarAnalyzer | Цикломатическая и когнитивная сложность, дублирование, длина метода, глубина вложенности, тренды по времени | Смысл кода; поднимает ложные срабатывания на DSL |
| NetArchTest / ArchUnitNET (тесты архитектуры) | Запрещённые зависимости между сборками и слоями, правила именования | Внутреннюю структуру классов |
| `dotnet format --verify-no-changes` в CI | Стилевые расхождения — снимает с ревью всю дискуссию о форматировании | Логику |
| Метрики Visual Studio / `dotnet-counters` | Maintainability Index, цикломатическая сложность, глубина наследования, coupling | Ничего архитектурного, но даёт список «куда смотреть» |

Пример архитектурного теста, который дешевле любого анализатора и запрещает обратные зависимости:

```csharp
[Fact]
public void Домен_не_должен_зависеть_от_инфраструктуры()
{
    var result = Types.InAssembly(typeof(Order).Assembly)
        .ShouldNot()
        .HaveDependencyOnAny("Microsoft.EntityFrameworkCore", "Microsoft.AspNetCore")
        .GetResult();

    Assert.True(result.IsSuccessful, string.Join(", ", result.FailingTypeNames ?? []));
}
```

> [!warning] Подводные камни
> - **`AnalysisMode=All` на существующем проекте даст сотни предупреждений сразу.** С `TreatWarningsAsErrors` сборка встанет, и команда включит `NoWarn` на всё — правила умрут. Внедряйте инкрементально: включите `TreatWarningsAsErrors` и точечный набор правил, зафиксируйте baseline, ужесточайте по одному правилу за спринт.
> - **Метрика не равна проблеме.** Файл с высокой цикломатической сложностью может быть корректным парсером, а идеальный по метрикам код — размазанной по 30 классам логикой, которую невозможно проследить. Метрики указывают, куда посмотреть, а не что переписать.
> - **Рефакторинг без тестов — это перепись с багами.** Прежде чем разбирать God Object, покройте его поведение тестами хотя бы на уровне «вход-выход». Иначе неотличимо, где вы сохранили поведение, а где случайно исправили баг, на который кто-то уже полагается ([[Работа с легаси]]).
> - **Погоня за нулём запахов создаёт свой антипаттерн.** Разбиение каждого класса до 20 строк даёт «ravioli code»: всё маленькое, но чтобы понять один сценарий, нужно открыть 15 файлов. Читаемость сценария важнее красоты отдельного класса.
> - **Внезапный «большой рефакторинг» без согласования — конфликт с командой.** Ветка на 200 файлов не проходит ревью, конфликтует со всем и живёт месяцами. Рефакторьте мелкими шагами в тех же PR, где меняете функциональность рядом.

> [!example] Как делают в бою
> Рабочая схема в командах, которые не тонут в техдолге:
> 1. **Boy Scout Rule** — трогаешь файл, оставь его чуть лучше. Не «отрефакторить модуль», а «пока правил метод — вынес guard clauses и убрал магическое число». Изменения идут в том же PR, поэтому не требуют отдельного согласования.
> 2. **Все механические правила — в CI.** `dotnet format --verify-no-changes`, анализаторы с `TreatWarningsAsErrors`, архитектурные тесты через NetArchTest. Что проверяет машина, того не обсуждают люди.
> 3. **Реестр техдолга рядом с кодом.** Не «TODO: отрефакторить», а тикет с описанием боли и её цены: «OrderService: 2400 строк, 14 конфликтов merge за квартал, средний PR по заказам ревьюится 3 дня». Так это конвертируется в аргумент для бизнеса ([[Технический долг: как говорить о нём с бизнесом]]).
> 4. **Правило соседства с фичей.** Крупный рефакторинг планируется в тот спринт, где по этому модулю всё равно есть фича. Тогда риск оправдан контекстом, и регрессия обнаружится тестированием фичи.
> 5. **Strangler Fig для God Object.** Новые сценарии пишутся в новых маленьких классах, старый сервис не переписывается, а постепенно теряет вызовы, пока не станет пустым и не удалится.

## Как говорить об этом на code review

Три вещи превращают замечание из вкусовщины в аргумент:

1. **Называй механику, не оценку.** Не «это God Object, плохо», а «этот класс правили 5 человек за месяц, из-за этого мы уже получили два конфликта; предлагаю вынести расчёт скидок — он меняется по другому поводу».
2. **Прикладывай цену обеих альтернатив.** «Сейчас: дешевле на 20 минут. Через полгода: любое изменение налоговых правил трогает файл, который параллельно правят». Мидл всегда показывает и цену рефакторинга тоже — иногда она выше.
3. **Различай уровни замечания.** Блокирующее («это утечка EF-сущности в контракт, миграция сломает мобильное приложение»), важное, но не блокирующее («стоит вынести»), и вкусовое (пометь как nit, не настаивай). Смешивание уровней — главная причина, почему ревью воспринимают как придирки. См. [[Code review: как давать и получать]].

Формулировки, которые работают:
- «Что произойдёт, когда придёт вторая валюта?» — вопрос вместо утверждения даёт автору самому найти проблему.
- «Я не блокирую, но зафиксирую в техдолг» — честный выход, когда рефакторинг сейчас дороже пользы.
- «Давай напишем анализатор» — если одно и то же замечание всплыло в третий раз, это уже не проблема автора, а проблема отсутствия автоматизации.

Формулировки, которые не работают: «так не пишут», «это антипаттерн», «прочитай Фаулера». Ноль информации о цене, максимум конфликта.

## Вопросы с собеседований

> [!question]- Чем антипаттерн отличается от code smell?
> Code smell — наблюдаемый признак в коде, который *может* указывать на проблему: длинный метод, дублирование, много параметров. Сам по себе он не ошибка, это повод присмотреться. Антипаттерн — конкретное решение проблемы, которое выглядит разумным, но систематически даёт отрицательный результат: Service Locator, свой ORM поверх ORM, Big Ball of Mud. Разница практическая: смелл оценивается в контексте (длинный метод в конфигурации — норма), антипаттерн почти всегда стоит перестать применять, хотя и у него бывают узкие оправданные случаи. Плюс антипаттерн обычно описывается вместе с рекомендуемой заменой.

> [!question]- Анемичная модель — это всегда плохо?
> Нет, и ответ «всегда плохо» — маркер того, что человек читал про DDD, но не применял. Для CRUD-модуля, где сущность — набор полей, а правила ограничиваются валидацией формата, анемичная модель дешевле: меньше кода, прямее маппинг, никакой косвенности. Проблема появляется в двух случаях: когда у сущности есть конечный автомат состояний (заказ, платёж, заявка) и переходы можно нарушить извне; и когда одно бизнес-правило продублировано в нескольких сервисах и уже разошлось. Признак, что пора менять подход: в БД периодически находят записи в невозможном состоянии. В одном проекте нормально держать анемичные CRUD-модули и богатый домен для сложного поддомена.

> [!question]- Почему Service Locator считают антипаттерном, если он «тот же DI»?
> Разница в направлении зависимости и в видимости. При конструкторной инъекции класс объявляет свои потребности в подписи — их видит компилятор, IDE, тест и валидатор графа контейнера при старте (`ValidateOnBuild`). При Service Locator класс зависит от контейнера и просит зависимости в рантайме: контракт врёт («мне нужен провайдер»), реальные потребности видны только при чтении всех методов, ошибка регистрации всплывает при первом вызове редкой ветки, а не в CI. Дополнительно легко разрешить scoped-сервис из корневого провайдера и получить captive dependency с утечкой `DbContext`. Легитимные исключения: композиционный корень, middleware, `BackgroundService`, который создаёт scope через `IServiceScopeFactory`.

> [!question]- Всегда ли нужен интерфейс для сервиса, который инжектится через DI?
> Нет. `IUserService` + `UserService` в вечной паре — спекулятивная общность: интерфейс, выведенный из одной реализации, повторяет её, включая случайные детали, и ничего не абстрагирует. Он стоит: лишний файл, лишний шаг навигации, ложное впечатление точки расширения. Интерфейс оправдан, когда за ним граница процесса или внешний ресурс (платёжка, почта, чужой API, часы), когда реализаций уже несколько, когда нужен декоратор, когда это публичный API библиотеки, или когда класс иначе невозможно подменить в тесте. Контейнер ASP.NET Core прекрасно регистрирует и инжектит конкретные типы: `AddScoped<UserService>()`.

> [!question]- Как объяснить бизнесу, что нужно время на рефакторинг?
> Через стоимость и наблюдаемые факты, а не через «код плохой». Работает связка: метрика (средний срок PR по модулю, число инцидентов, время onboarding нового человека), причина (конкретный смелл с числами: 2400 строк, 14 merge-конфликтов за квартал), прогноз (если не делать, следующие три фичи по этому модулю дадут +40 % к оценке) и предложение с ограниченным объёмом (не «переписать», а «вынести расчёт скидок, две недели, риск снижаем тестами»). Крупный рефакторинг лучше не продавать отдельным проектом, а привязывать к фиче в том же модуле. Подробнее — заметка про технический долг.

> [!question]- Как автоматически находить проблемные места в большом легаси-проекте?
> Комбинация трёх источников. Первый — статический анализ: встроенные .NET-анализаторы с `AnalysisMode=All`, Sonar для когнитивной сложности и дублирования, свои Roslyn-правила под проектные соглашения. Второй — история git: файлы с наибольшим числом коммитов и уникальных авторов почти всегда и есть точки боли, потому что метрика связана с реальной частотой изменений, а не с гипотетической красотой. Третий — данные с прода: где происходят инциденты и где самые долгие запросы. Пересечение трёх списков даёт приоритет. Только метрики без истории изменений уводят в рефакторинг кода, который никто не трогает.

> [!question]- Что такое «правило трёх» и почему не абстрагировать сразу?
> Правило: не выделяй абстракцию до третьего похожего случая. Причина в информации. По одному примеру абстракции нет вообще. По двум невозможно отличить сущностное сходство от случайного — сделанная обобщённая конструкция с высокой вероятностью окажется неправильной, а неправильная абстракция дороже дублирования: её надо и поддерживать, и обходить. По трём примерам видно, что варьируется, и абстракция выводится из фактов. Обратная сторона: на третьем случае надо действовать, иначе три копии начнут расходиться в поведении и синхронизировать их станет отдельной работой.

> [!question]- Почему нельзя возвращать EF-сущности из контроллера?
> Три независимые причины. Сериализация: двусторонние навигационные свойства дают циклы, `System.Text.Json` бросает исключение либо требует `ReferenceHandler.Preserve`, что портит формат для клиентов. Безопасность на запись: если тот же тип принимается через `[FromBody]`, клиент может прислать поля, которых не должен касаться, — over-posting (`IsAdmin`, `Status`, `Total`). Совместимость: форма таблицы становится публичным контрактом, любая миграция превращается в breaking change для потребителей API, а лениво загружаемые свойства при сериализации могут дать шквал дополнительных запросов. Решение — отдельные DTO и явный маппинг.

## Задачи

### Задача 1. Найти оси разбиения

Дан класс (сокращённо). Определи, на какие классы его разбить и по какому признаку.

```csharp
public class EmployeeService(AppDbContext db, IEmailSender email, IPdfGenerator pdf, IHrSystemClient hr)
{
    public Task<Employee> HireAsync(HireDto dto) { /* db, email */ }
    public Task PromoteAsync(int id, string newGrade) { /* db */ }
    public Task<byte[]> BuildPayslipAsync(int id, int month) { /* db, pdf */ }
    public Task<byte[]> BuildContractAsync(int id) { /* db, pdf */ }
    public Task SyncFromHrAsync() { /* db, hr */ }
    public Task SendBirthdayGreetingsAsync() { /* db, email */ }
}
```

> [!success]- Решение
> Выписываем зависимости по методам и находим кластеры:
>
> | Метод | Зависимости | Повод меняться |
> |---|---|---|
> | HireAsync, PromoteAsync | db (+email) | правила кадровых операций |
> | BuildPayslipAsync, BuildContractAsync | db, pdf | форма документов |
> | SyncFromHrAsync | db, hr | контракт внешней системы |
> | SendBirthdayGreetingsAsync | db, email | маркетинг/HR-коммуникации, фоновое расписание |
>
> ```csharp
> // Кадровые операции над сотрудником
> public sealed class EmployeeLifecycle(AppDbContext db, IEmailSender email)
> {
>     public Task<Employee> HireAsync(HireDto dto, CancellationToken ct) { /* ... */ }
>     public Task PromoteAsync(EmployeeId id, Grade newGrade, CancellationToken ct) { /* ... */ }
> }
>
> // Документы: меняются, когда меняется их форма
> public sealed class EmployeeDocuments(AppDbContext db, IPdfGenerator pdf)
> {
>     public Task<byte[]> PayslipAsync(EmployeeId id, YearMonth period, CancellationToken ct) { /* ... */ }
>     public Task<byte[]> ContractAsync(EmployeeId id, CancellationToken ct) { /* ... */ }
> }
>
> // Интеграция: меняется вместе с внешним API
> public sealed class HrSystemSync(AppDbContext db, IHrSystemClient hr) { /* ... */ }
>
> // Фоновая рассылка: другой жизненный цикл
> public sealed class BirthdayGreetingsJob(AppDbContext db, IEmailSender email) : BackgroundService { /* ... */ }
> ```
>
> Ключевое: ось — «повод меняться» и совместно используемые зависимости, а не «слои». Разбиение на `EmployeeService` + `EmployeeRepository` + `EmployeeMapper` дало бы больше файлов при том же радиусе изменений.

### Задача 2. Убрать Service Locator, не потеряв выбор в рантайме

Есть код, который выбирает уведомителя по каналу из контейнера. Перепиши без `IServiceProvider`, сохранив выбор реализации во время исполнения.

```csharp
public class NotificationService(IServiceProvider sp)
{
    public Task SendAsync(string channel, string to, string text) => channel switch
    {
        "sms"   => sp.GetRequiredService<ISmsSender>().SendAsync(to, text),
        "email" => sp.GetRequiredService<IEmailSender>().SendAsync(to, text),
        "push"  => sp.GetRequiredService<IPushSender>().SendAsync(to, text),
        _ => throw new ArgumentOutOfRangeException(nameof(channel))
    };
}
```

> [!success]- Решение
> ```csharp
> public enum Channel { Sms, Email, Push }
>
> public interface INotificationSender
> {
>     Channel Channel { get; }
>     Task SendAsync(string to, string text, CancellationToken ct);
> }
>
> public sealed class NotificationService
> {
>     private readonly Dictionary<Channel, INotificationSender> _senders;
>
>     // Все реализации приходят через DI: граф проверяется при старте,
>     // зависимость видна в конструкторе, словарь строится один раз.
>     public NotificationService(IEnumerable<INotificationSender> senders) =>
>         _senders = senders.ToDictionary(s => s.Channel);
>
>     public Task SendAsync(Channel channel, string to, string text, CancellationToken ct) =>
>         _senders.TryGetValue(channel, out var sender)
>             ? sender.SendAsync(to, text, ct)
>             : throw new ArgumentOutOfRangeException(nameof(channel), $"Нет отправителя для {channel}");
> }
>
> // Регистрация
> builder.Services.AddScoped<INotificationSender, SmsSender>();
> builder.Services.AddScoped<INotificationSender, EmailSender>();
> builder.Services.AddScoped<INotificationSender, PushSender>();
> builder.Services.AddScoped<NotificationService>();
> ```
>
> Что улучшилось: зависимости явные; `ValidateOnBuild` поймает незарегистрированного отправителя при старте, а не при первом СМС; `string channel` заменён на `enum`, то есть опечатка в канале — ошибка компиляции; в тесте достаточно передать список из одного фейкового отправителя. Альтернатива — keyed services, если реализации не должны инстанцироваться все сразу: тогда `IServiceProvider` заменяется на `[FromKeyedServices]` или на `IKeyedServiceProvider` в узкой фабрике.

### Задача 3. Выпрямить вложенность

Перепиши через guard clauses и pattern matching. Поведение должно сохраниться.

```csharp
public string Describe(Shipment s)
{
    if (s != null)
    {
        if (s.Items != null && s.Items.Count > 0)
        {
            if (s.DeliveredAt.HasValue) return "Доставлено";
            else
            {
                if (s.ShippedAt.HasValue)
                {
                    if (DateTime.UtcNow - s.ShippedAt.Value > TimeSpan.FromDays(30)) return "Потеряно";
                    else return "В пути";
                }
                else return "Готовится";
            }
        }
        else return "Пустая отправка";
    }
    return "Нет данных";
}
```

> [!success]- Решение
> ```csharp
> private static readonly TimeSpan LostAfter = TimeSpan.FromDays(30);   // вместо магических 30
>
> public string Describe(Shipment? s, DateTimeOffset now)
> {
>     // Guard clauses: сначала все причины не идти дальше
>     if (s is null) return "Нет данных";
>     if (s.Items is null or { Count: 0 }) return "Пустая отправка";
>     if (s.DeliveredAt is not null) return "Доставлено";
>     if (s.ShippedAt is not { } shipped) return "Готовится";
>
>     // Основной сценарий — на нулевом уровне вложенности
>     return now - shipped > LostAfter ? "Потеряно" : "В пути";
> }
> ```
>
> Что изменилось по механике: максимальная вложенность 1 вместо 4; каждый `return` виден рядом со своим условием, поэтому при добавлении нового статуса невозможно приписать его не к той ветке; `DateTime.UtcNow` вынесен в параметр — метод стал чистым и тестируемым без подмены времени (в продакшене время берут из `TimeProvider`); `30` получил имя. Паттерн `is not { } shipped` одновременно проверяет наличие значения и извлекает его.

### Задача 4. Оценить, стоит ли рефакторить

Проект: внутренний сервис отчётности, живёт 4 года, правки раз в квартал (обновление формата выгрузки). В нём есть `ReportBuilder` на 1800 строк с 12 зависимостями, анемичные модели, `catch (Exception) { _log.LogError(ex.Message); }` в трёх местах и `IReportService` с единственной реализацией. Тестов нет. Через месяц сервис заменяется покупным BI-решением. Что делать?

> [!success]- Решение
> Почти ничего. Разбор по стоимости:
>
> | Находка | Цена жизни с ней | Цена рефакторинга | Решение |
> |---|---|---|---|
> | `ReportBuilder` 1800 строк | Правка раз в квартал, один автор, конфликтов нет | Разбор без тестов = высокий риск регрессии | Оставить |
> | Анемичные модели | Отчётность — чтение, инвариантов нет | Богатая модель тут не нужна вообще | Оставить, это не смелл |
> | `IReportService` с одной реализацией | Один лишний файл | 10 минут | Оставить: удаление тоже стоит ревью и регрессии на месяц жизни |
> | `catch (Exception)` с логом только Message | Инцидент невозможно диагностировать; неудачная выгрузка выглядит успешной | 15 минут, риск нулевой | **Исправить**: `LogError(ex, "...")`, а лучше не глотать вовсе |
>
> Правильный вывод мидла: единственное, что стоит правки, — проглатывание исключений, потому что оно влияет на прод *в оставшийся месяц* и стоит четверть часа. Остальное — обоснованно оставить, и это тоже профессиональное решение, а не лень. Обратный ответ («переписать по SOLID») означает потратить недели на код, который будет удалён, и внести регрессии в работающий сервис. Если бы сервис жил ещё три года и правился еженедельно, приоритеты были бы обратными — но начали бы всё равно с тестов на поведение, а не с разбора класса.

## Итог

- Смелл — не приговор, а вход в разговор о стоимости. Мидл всегда называет, *что именно* дорожает: радиус изменения, конфликты merge, невозможность теста, риск невалидного состояния в БД.
- God Object разбивают по осям изменчивости (что меняется вместе), а не по техническим слоям: слои ради слоёв дают больше файлов при том же радиусе изменений.
- Анемичная модель нормальна для CRUD и вредна там, где есть автомат состояний или дублирование правил. Богатая модель — инструмент под конкретную проблему, а не признак хорошего разработчика.
- Спекулятивная общность и over-engineering дороже дублирования: неправильная абстракция требует и поддержки, и обхода. Правило трёх — дешёвая защита от преждевременных обобщений.
- Всё механическое (`catch (Exception)`, `async void`, `.Result`, форматирование, запрещённые зависимости) должно ловиться анализаторами и архитектурными тестами в CI, чтобы ревью занималось смыслом.
- Решение «оставить как есть» — законное и часто правильное: одноразовый код, редко изменяемый модуль, сервис перед выводом из эксплуатации.

## Связанное

- [[SOLID]]
- [[SOLID]]
- [[Связанность и связность (coupling, cohesion)]]
- [[Слоистая архитектура]]
- [[Vertical Slice Architecture]]
- [[Repository и Unit of Work: нужны ли поверх EF Core]]
- [[Технический долг: как говорить о нём с бизнесом]]
