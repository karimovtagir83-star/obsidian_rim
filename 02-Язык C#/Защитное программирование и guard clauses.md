---
tags: [раздел-02, основы, dotnet10, собес]
aliases: [Guard clauses, Defensive programming, Защитное программирование, ThrowIfNull, Fail fast]
---

# Защитное программирование и guard clauses

> [!abstract] Коротко
> Guard clause — проверка в начале метода, которая при нарушении условия немедленно возвращает управление или бросает исключение. Смысл двойной: обнаружить проблему как можно раньше, у её источника (fail fast), и убрать вложенность — после серии guard-выражений основной код идёт без отступов, на «счастливом пути». В .NET 6–8 для этого появились готовые статические помощники: `ArgumentNullException.ThrowIfNull`, `ArgumentException.ThrowIfNullOrWhiteSpace`, `ArgumentOutOfRangeException.ThrowIfNegative`, `ObjectDisposedException.ThrowIf`. Ключевое ограничение — не путать защиту от ошибок программиста (guard) с валидацией пользовательского ввода (это результат, а не исключение).

## Зачем это нужно

Ошибка, обнаруженная в момент возникновения, стоит минуты. Та же ошибка, обнаруженная через три слоя вызовов, — часы:

```csharp
// Без guard: null уедет вглубь и всплывёт неизвестно где
public void Process(Order order)
{
    var total = order.Items.Sum(i => i.Price);   // NullReferenceException — но какой из двух null?
    _repository.Save(order);
}

// С guard: ошибка названа, место известно, имя параметра подставлено автоматически
public void Process(Order order)
{
    ArgumentNullException.ThrowIfNull(order);
    ArgumentException.ThrowIfNullOrEmpty(order.Currency);

    var total = order.Items.Sum(i => i.Price);
    _repository.Save(order);
}
```

Вторая выгода — структура кода. Сравни:

```csharp
// Стрелка судьбы: логика тонет в отступах
public decimal Calculate(Order? order)
{
    if (order != null)
    {
        if (order.Items.Count > 0)
        {
            if (order.Status != OrderStatus.Cancelled)
            {
                return order.Items.Sum(i => i.Price);
            }
            else { return 0; }
        }
        else { return 0; }
    }
    else { throw new ArgumentNullException(nameof(order)); }
}

// Guard clauses: сначала все отказы, потом одна главная строка
public decimal Calculate(Order? order)
{
    ArgumentNullException.ThrowIfNull(order);
    if (order.Items.Count == 0) return 0;
    if (order.Status == OrderStatus.Cancelled) return 0;

    return order.Items.Sum(i => i.Price);
}
```

Второй вариант читается сверху вниз: «вот условия, при которых мы сюда не идём, а вот что мы делаем». Это и есть основной приём борьбы с вложенностью.

---

## Готовые помощники .NET

```csharp
// null
ArgumentNullException.ThrowIfNull(order);                       // .NET 6
ArgumentNullException.ThrowIfNull(order.Customer, nameof(order.Customer));

// строки
ArgumentException.ThrowIfNullOrEmpty(name);                     // .NET 7
ArgumentException.ThrowIfNullOrWhiteSpace(name);                // .NET 8

// числа
ArgumentOutOfRangeException.ThrowIfNegative(amount);            // .NET 8
ArgumentOutOfRangeException.ThrowIfNegativeOrZero(quantity);
ArgumentOutOfRangeException.ThrowIfZero(divisor);
ArgumentOutOfRangeException.ThrowIfGreaterThan(page, maxPage);
ArgumentOutOfRangeException.ThrowIfLessThan(count, 1);
ArgumentOutOfRangeException.ThrowIfEqual(from, to);

// состояние объекта
ObjectDisposedException.ThrowIf(_disposed, this);               // .NET 7
```

Все они используют `[CallerArgumentExpression]`, поэтому имя параметра подставляется автоматически и не рассинхронизируется при переименовании ([[Атрибуты]]). Плюс они помечены `[DoesNotReturn]`-логикой внутри, что помогает анализу nullable: после `ThrowIfNull(order)` компилятор считает `order` не-null.

Дополнительный выигрыш — производительность: проверка вынесена в отдельный метод, и JIT может встроить вызывающий код, чего не делает при наличии `throw` прямо в теле.

Для своих условий пишут короткий помощник:

```csharp
public static class Guard
{
    public static T NotNull<T>(T? value,
        [System.Runtime.CompilerServices.CallerArgumentExpression(nameof(value))] string? name = null)
        where T : class
        => value ?? throw new ArgumentNullException(name);

    public static void Against(bool condition, string message,
        [System.Runtime.CompilerServices.CallerArgumentExpression(nameof(condition))] string? expr = null)
    {
        if (condition) throw new ArgumentException($"{message} (нарушено: {expr})");
    }
}

Guard.Against(order.Total < 0, "Сумма заказа не может быть отрицательной");
```

---

## Где проверять, а где не надо

Главная ошибка «защитного программирования» — проверять всё везде. Тогда половина кода превращается в проверки, которые всё равно не срабатывают, а настоящие дефекты прячутся за шумом.

Работающее правило — **проверять на границах**:

| Граница | Что проверять | Чем |
|---|---|---|
| Публичный API библиотеки | все аргументы | guard-выражения и исключения |
| HTTP-эндпоинт, обработчик сообщения | всю входящую модель | валидация → 400, не исключения |
| Конструктор доменного типа | инварианты типа | исключения |
| Внутренние приватные методы | ничего или `Debug.Assert` | доверяем вызывающему коду в своём модуле |

Логика простая: если метод виден снаружи сборки и его может вызвать кто угодно, аргументы проверяются в рантайме — даже при включённых nullable reference types, потому что вызывающий может быть скомпилирован без них или использовать рефлексию ([[Nullable reference types]]). Приватный метод, который вызывается тремя строчками ниже уже проверенным кодом, повторную проверку не требует — там уместен `Debug.Assert`, исчезающий в Release.

### Конструкторы: инвариант с самого начала

```csharp
public sealed class Order
{
    public Order(long customerId, string currency, IReadOnlyList<OrderItem> items)
    {
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(customerId);
        ArgumentException.ThrowIfNullOrWhiteSpace(currency);
        ArgumentNullException.ThrowIfNull(items);
        if (items.Count == 0)
            throw new ArgumentException("Заказ должен содержать хотя бы одну позицию", nameof(items));

        CustomerId = customerId;
        Currency = currency;
        Items = [..items];              // защитная копия: снаружи список изменить не смогут
    }

    public long CustomerId { get; }
    public string Currency { get; }
    public IReadOnlyList<OrderItem> Items { get; }
}
```

Тип, который невозможно создать в некорректном состоянии, избавляет от проверок во всех остальных методах — это и есть настоящая защита. Отсюда связь с иммутабельностью: неизменяемый объект, проверенный один раз при создании, остаётся валидным всегда ([[Иммутабельность как приём проектирования]]).

Обрати внимание на `[..items]` — защитная копия. Без неё вызывающий сохранит ссылку на переданный список и сможет менять содержимое заказа в обход его методов. Тот же приём нужен при возврате коллекций наружу: отдавать `IReadOnlyList<T>` или копию, а не внутренний список.

---

## Fail fast

Принцип: при обнаружении некорректного состояния лучше остановиться сразу, чем продолжать работу и портить данные.

```csharp
// Конфигурация: проверяем при старте, а не при первом использовании
var connectionString = builder.Configuration.GetConnectionString("Default")
    ?? throw new InvalidOperationException("Не задана строка подключения ConnectionStrings:Default");

// Options с валидацией: приложение не поднимется с некорректной конфигурацией
builder.Services.AddOptions<PaymentOptions>()
    .Bind(builder.Configuration.GetSection("Payment"))
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

Упасть при старте — лучший из возможных сбоев: он происходит на деплое, виден сразу, не затрагивает пользователей и не оставляет систему в половинчатом состоянии. Худший сценарий — приложение поднялось, работает, и только через сутки выясняется, что оно писало в неправильную базу.

Обратная сторона принципа: fail fast не означает «падать по любому поводу в рантайме». Сбой одного запроса не должен ронять процесс, а недоступность необязательной интеграции — отключать весь сервис. Граница проходит по вопросу «можем ли мы продолжать корректно».

---

## Guard или валидация

Разница принципиальная, и её путают чаще всего:

| | Guard | Валидация |
|---|---|---|
| Против чего | ошибка программиста | некорректные данные пользователя |
| Кто виноват | вызывающий код | внешний ввод |
| Что делать | бросить исключение | вернуть список ошибок |
| Сколько ошибок | первая же прерывает | нужны все сразу |
| Где | начало метода | граница системы |

```csharp
// Валидация: пользователь должен увидеть все проблемы разом
public IResult Create(CreateOrderRequest request)
{
    var errors = new List<string>();
    if (string.IsNullOrWhiteSpace(request.Address)) errors.Add("Адрес обязателен");
    if (request.Items.Count == 0) errors.Add("Добавьте хотя бы один товар");
    if (errors.Count > 0) return Results.ValidationProblem(...);

    // Guard: сюда уже приходят проверенные данные, и null здесь означал бы баг
    var order = _factory.Create(request);
    ...
}
```

Подробнее — [[Когда исключение — плохой выбор]] и [[Model binding и валидация]].

> [!warning] Подводные камни
> - **Проверки везде.** Дублирование guard-выражений на каждом уровне вызова не повышает надёжность, а прячет логику. Проверять на границах.
> - **Guard вместо валидации.** Исключение на каждое некорректное поле формы — и дорого, и пользователь видит ошибки по одной.
> - **`if (x == null) return;` в пустоту.** Тихий выход из метода при некорректном аргументе — худший вариант: ошибки нет, эффекта нет, диагностировать нечего.
> - **Ручной `throw new ArgumentNullException("orderr")`.** Опечатка в имени параметра и рассинхронизация после переименования; `ThrowIfNull` подставит имя сам.
> - **Отсутствие защитной копии.** Принятая в конструкторе коллекция остаётся под контролем вызывающего — инвариант можно нарушить снаружи.
> - **`Debug.Assert` для проверок, нужных в проде.** В Release эти вызовы вырезаются вместе с аргументами.
> - **Вера в nullable reference types как в защиту рантайма.** Это анализ компилятора; на границе публичного API проверки всё равно нужны.
> - **Слишком дорогие guard-проверки.** Обход коллекции или запрос к базе «на всякий случай» в начале каждого метода — заметная стоимость на горячем пути. Дешёвые проверки первыми, дорогие — только там, где действительно нужны.

> [!example] Как делают в бою
> Публичные методы библиотек и доменных сервисов начинаются с двух-трёх guard-выражений на готовых помощниках .NET — это одна строка на проверку, без ветвления и без ручных `nameof`. Внутренние методы модуля проверок не содержат: там действует контракт, что данные уже валидны.
> Доменные типы проектируют так, чтобы некорректное состояние было невыразимым: обязательные поля — в конструкторе, коллекции — копией, значения с ограничениями — value object'ами (`Money`, `Email`, `Quantity`), которые сами себя валидируют при создании. После этого guard-выражения в бизнес-методах почти исчезают: типы гарантируют то, что раньше проверялось руками.
> Конфигурация проверяется на старте через `ValidateOnStart`, чтобы неверный деплой падал сразу, а не через сутки. И отдельная договорённость: guard бросает исключение, валидация возвращает результат — так по коду видно, что является дефектом, а что нормальным отказом.

---

## Вопросы с собеседований

> [!question]- Что такое guard clause и зачем он нужен?
> Это проверка предусловия в начале метода, которая при нарушении немедленно завершает выполнение — броском исключения или ранним возвратом. Даёт две вещи. Первая — раннее обнаружение: ошибка фиксируется в точке, где известен виновник и есть контекст, а не тремя слоями глубже, где `NullReferenceException` уже ничего не объясняет. Вторая — структура: вместо вложенных `if` с логикой внутри получается плоский список отказов сверху и основной сценарий без отступов снизу, что заметно улучшает читаемость и упрощает добавление новых условий. В .NET 6–8 для типовых проверок появились статические помощники (`ArgumentNullException.ThrowIfNull`, `ArgumentException.ThrowIfNullOrWhiteSpace`, `ArgumentOutOfRangeException.ThrowIfNegative`, `ObjectDisposedException.ThrowIf`), которые подставляют имя параметра автоматически через `[CallerArgumentExpression]` и помогают анализу nullable.

> [!question]- Нужны ли проверки на null, если включены nullable reference types?
> На границе публичного API — да. NRT работает только на этапе компиляции: в метаданных сохраняются аннотации, но рантайм их не проверяет, и `string` может оказаться `null`, если вызывающий скомпилирован без NRT, использует рефлексию, десериализацию, мокинг или язык без такой поддержки. Поэтому публичные методы библиотек и точки входа проверяют аргументы явно. Внутри собственного проекта, где режим включён везде и предупреждения возведены в ошибки, повторные проверки в приватных методах избыточны: они дублируют то, что уже гарантировано компилятором, и только зашумляют код. Там, где хочется зафиксировать внутренний инвариант, уместнее `Debug.Assert` — он проверяет при разработке и исчезает в Release.

> [!question]- В чём разница между guard clause и валидацией?
> В том, кто виноват и что должно произойти. Guard защищает от ошибки программиста: аргумент `null`, отрицательное количество, вызов на освобождённом объекте. Такое не должно случаться в корректной программе, поэтому реакция — исключение, прерывающее выполнение на первой же проблеме. Валидация обрабатывает ожидаемо некорректные данные из внешнего мира: пользовательский ввод, тело HTTP-запроса, сообщение из брокера. Такое случается регулярно, поэтому реакция — собрать **все** ошибки и вернуть их как результат (ответ 400 с деталями), а не бросать исключение на первом же поле. Практическое следствие: валидация живёт на границе системы и выражается результатом, guard — в начале методов и выражается исключением; после валидации внутрь системы попадают данные, которые уже можно считать корректными.

---

## Задачи

### Задача 1. Переписать с guard clauses

```csharp
public string Format(Order order, string culture)
{
    if (order != null)
    {
        if (!string.IsNullOrEmpty(culture))
        {
            if (order.Items != null && order.Items.Count > 0)
            {
                var total = order.Items.Sum(i => i.Price);
                return total.ToString("C", new CultureInfo(culture));
            }
            else
            {
                return "пусто";
            }
        }
        else
        {
            throw new ArgumentException("culture");
        }
    }
    else
    {
        throw new ArgumentNullException("order");
    }
}
```

> [!success]- Решение
> ```csharp
> public string Format(Order order, string culture)
> {
>     ArgumentNullException.ThrowIfNull(order);
>     ArgumentException.ThrowIfNullOrWhiteSpace(culture);
>
>     if (order.Items.Count == 0) return "пусто";
>
>     var total = order.Items.Sum(i => i.Price);
>     return total.ToString("C", CultureInfo.GetCultureInfo(culture));
> }
> ```
> Изменения: строковые имена параметров в исключениях заменены автоматической подстановкой через `[CallerArgumentExpression]` — при переименовании они не разъедутся; `IsNullOrEmpty` заменён на `IsNullOrWhiteSpace`, потому что строка из пробелов не является допустимым названием культуры; проверка `order.Items != null` убрана, потому что коллекция должна быть гарантирована конструктором `Order`, а не проверяться в каждом методе; `new CultureInfo(culture)` заменён на `GetCultureInfo`, который возвращает кешированный экземпляр. Уровней вложенности стало ноль, и основная логика видна с первого взгляда.

### Задача 2. Сделать состояние невыразимым

Тип `Money` часто приходит с отрицательной суммой и пустой валютой, из-за чего проверки разбросаны по десятку методов. Как это исправить?

> [!success]- Решение
> ```csharp
> public readonly record struct Money
> {
>     public Money(decimal amount, string currency)
>     {
>         ArgumentOutOfRangeException.ThrowIfNegative(amount);
>         ArgumentException.ThrowIfNullOrWhiteSpace(currency);
>         if (currency.Length != 3)
>             throw new ArgumentException("Код валюты должен состоять из трёх символов", nameof(currency));
>
>         Amount = amount;
>         Currency = currency.ToUpperInvariant();
>     }
>
>     public decimal Amount { get; }
>     public string Currency { get; }
>
>     public static Money operator +(Money a, Money b) => a.Currency == b.Currency
>         ? new Money(a.Amount + b.Amount, a.Currency)
>         : throw new InvalidOperationException($"Нельзя складывать {a.Currency} и {b.Currency}");
> }
> ```
> Идея: проверка выполняется один раз — в момент создания значения, — после чего любой `Money` в системе корректен по построению, и десяток проверок в методах исчезает. Это перенос защиты из кода в тип. Дополнительно нормализуется регистр валюты (иначе `"uzs"` и `"UZS"` окажутся разными), а оператор сложения не даёт смешать валюты.
> Осталась одна тонкость, о которой надо знать: у структуры всегда есть `default`-значение, создаваемое в обход конструктора, — `default(Money)` даст сумму 0 и `Currency == null`. Если это критично, тип делают классом или добавляют проверку `Currency is null` в операциях. Для ссылочного типа проблема снимается полностью.

---

## Итог

- Guard clause — проверка предусловия в начале метода с немедленным отказом: раньше находит ошибку и убирает вложенность.
- Готовые помощники .NET (`ThrowIfNull`, `ThrowIfNullOrWhiteSpace`, `ThrowIfNegative`, `ObjectDisposedException.ThrowIf`) короче ручных проверок и сами подставляют имя параметра.
- Проверять надо на границах: публичный API и точки входа — обязательно, приватные методы модуля — нет.
- Валидация внешнего ввода — это результат со списком ошибок, а не исключение на первом поле.
- Лучшая защита — тип, который невозможно создать в некорректном состоянии; тогда проверки исчезают сами.
- Защитная копия коллекции в конструкторе и `IReadOnlyList<T>` наружу сохраняют инвариант.
- Fail fast при старте: неверная конфигурация должна ронять приложение на деплое, а не через сутки.
- NRT не заменяет проверки в рантайме на публичной границе.

## Связанное

- [[Методы и параметры]] — контракты методов и их подписи
- [[Обработка исключений]] · [[Иерархия исключений в .NET]] · [[Свои типы исключений]]
- [[Когда исключение — плохой выбор]] — граница между guard и валидацией
- [[Nullable reference types]] — что гарантирует компилятор, а что нет
- [[Конструкторы и инициализация]] · [[Иммутабельность как приём проектирования]]
- [[Атрибуты]] — `CallerArgumentExpression` под капотом помощников
- [[Model binding и валидация]] · [[Options pattern и конфигурация сервисов]]
- [[DDD: тактические паттерны]] — value objects, которые сами себя валидируют
