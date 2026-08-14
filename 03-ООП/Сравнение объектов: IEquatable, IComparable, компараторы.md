---
tags: [раздел-03, основы, собес, ооп]
aliases: [IEquatable, IComparable, IComparer, IEqualityComparer, Сравнение объектов, Сортировка]
---

# Сравнение объектов: IEquatable, IComparable, компараторы

> [!abstract] Коротко
> Сравнений два вида, и их путают: **равенство** (равны или нет) и **порядок** (что раньше). За равенство отвечают `Equals`/`GetHashCode` и интерфейс `IEquatable<T>`, за порядок — `IComparable<T>` с методом `CompareTo`, возвращающим отрицательное число, ноль или положительное. Оба вида имеют «встроенную» форму (тип сам знает, как себя сравнивать) и «внешнюю» — `IEqualityComparer<T>` и `IComparer<T>`, которые передаются в коллекцию или в LINQ. Правило выбора: естественное сравнение для типа одно, и оно живёт внутри; все альтернативные — снаружи, отдельными компараторами.

## Зачем это нужно

```csharp
var orders = new List<Order> { ... };

orders.Sort();                                    // InvalidOperationException, если нет IComparable
orders.Sort((a, b) => a.Total.CompareTo(b.Total)); // сравнение делегатом
orders.Sort(new OrderByDateComparer());            // сравнение объектом-компаратором
var sorted = orders.OrderBy(o => o.Total).ThenByDescending(o => o.CreatedAt).ToList();  // LINQ

var set = new HashSet<Order>(OrderBySkuComparer.Instance);       // своё равенство
var dict = new Dictionary<string, int>(StringComparer.OrdinalIgnoreCase);
var sortedSet = new SortedSet<Order>(new OrderByDateComparer()); // свой порядок
```

Понимать нужно, какой механизм включается в каждом случае и что произойдёт, если нужного интерфейса нет.

---

## Равенство: IEquatable\<T\>

```csharp
public sealed class Sku : IEquatable<Sku>
{
    public required string Value { get; init; }

    public bool Equals(Sku? other)                       // типизированная версия — без упаковки
        => other is not null && string.Equals(Value, other.Value, StringComparison.Ordinal);

    public override bool Equals(object? obj) => Equals(obj as Sku);
    public override int GetHashCode() => Value.GetHashCode(StringComparison.Ordinal);
}
```

Зачем интерфейс, если есть `override Equals(object)`: коллекции и LINQ сравнивают через `EqualityComparer<T>.Default`, а тот, обнаружив `IEquatable<T>`, вызывает типизированный метод напрямую. Для структуры это критично — иначе каждое сравнение упаковывает оба операнда ([[Boxing и unboxing]]); для класса выигрыш меньше, но приведение типа и проверка всё равно исчезают.

Правила и контракт разобраны в [[Object: Equals, GetHashCode, ToString]]; здесь главное: `IEquatable<T>` не заменяет `override Equals(object)` и `GetHashCode`, а дополняет их. `record` и `record struct` генерируют все три автоматически.

### Внешнее равенство: IEqualityComparer\<T\>

```csharp
public sealed class CaseInsensitiveSkuComparer : IEqualityComparer<Product>
{
    public static readonly CaseInsensitiveSkuComparer Instance = new();

    public bool Equals(Product? x, Product? y)
        => x is not null && y is not null
           && string.Equals(x.Sku, y.Sku, StringComparison.OrdinalIgnoreCase);

    public int GetHashCode(Product obj) => obj.Sku.GetHashCode(StringComparison.OrdinalIgnoreCase);
}

var unique = products.Distinct(CaseInsensitiveSkuComparer.Instance).ToList();
var index = new Dictionary<Product, int>(CaseInsensitiveSkuComparer.Instance);
```

Компаратор нужен, когда правило сравнения зависит от контекста, а не от типа. Он принимается конструкторами `Dictionary`, `HashSet`, `ConcurrentDictionary` и LINQ-методами `Distinct`, `GroupBy`, `Join`, `Contains`, `Except`, `Intersect`, `ToDictionary`.

Готовые реализации из BCL: `StringComparer.Ordinal`, `StringComparer.OrdinalIgnoreCase`, `StringComparer.CurrentCulture`, `ReferenceEqualityComparer.Instance` (сравнение строго по ссылке), `EqualityComparer<T>.Default`, а также `EqualityComparer<T>.Create(...)` для быстрого создания из лямбд (.NET 8+).

---

## Порядок: IComparable\<T\>

```csharp
public sealed class Version : IComparable<Version>, IComparable
{
    public required int Major { get; init; }
    public required int Minor { get; init; }

    public int CompareTo(Version? other)
    {
        if (other is null) return 1;                     // null считается меньше всего
        int byMajor = Major.CompareTo(other.Major);
        return byMajor != 0 ? byMajor : Minor.CompareTo(other.Minor);
    }

    int IComparable.CompareTo(object? obj) => obj is Version v
        ? CompareTo(v)
        : throw new ArgumentException("Ожидался Version", nameof(obj));
}
```

Контракт `CompareTo`:

| Результат | Значение |
|---|---|
| `< 0` | текущий объект **меньше** other |
| `0` | равны по порядку |
| `> 0` | текущий объект **больше** |

Требования: сравнение должно задавать **полный порядок** — быть антисимметричным (`x.CompareTo(y)` и `y.CompareTo(x)` противоположны по знаку), транзитивным, и `x.CompareTo(null)` должен возвращать положительное число, а не бросать исключение.

Отдельное соглашение: если `CompareTo` возвращает 0, то `Equals` должен возвращать `true`. Нарушение не запрещено, но приводит к странностям: `SortedSet<T>` считает элементы дубликатами по компаратору, а `HashSet<T>` — нет, и одна и та же коллекция ведёт себя по-разному в зависимости от типа.

Не-обобщённый `IComparable` реализуют явно и только для совместимости со старыми API (`ArrayList`, `Array.Sort` без дженериков) — [[Явная реализация интерфейсов]].

### Внешний порядок: IComparer\<T\> и Comparison\<T\>

```csharp
public sealed class OrderByTotalDescending : IComparer<Order>
{
    public int Compare(Order? x, Order? y) => (y?.Total ?? 0).CompareTo(x?.Total ?? 0);
}

orders.Sort(new OrderByTotalDescending());                        // IComparer<T>
orders.Sort((a, b) => b.Total.CompareTo(a.Total));                // Comparison<T> — делегат
orders.Sort(Comparer<Order>.Create((a, b) => b.Total.CompareTo(a.Total)));  // из лямбды

var sortedSet = new SortedSet<Order>(new OrderByTotalDescending());
var sortedDict = new SortedDictionary<string, int>(StringComparer.Ordinal);
```

`Comparer<T>.Default` использует `IComparable<T>` типа, а если его нет — бросает `InvalidOperationException` в момент сравнения, а не при создании коллекции. Это частая причина «неожиданного» исключения при первой вставке в `SortedSet`.

### Составное сравнение

```csharp
// Руками: по статусу, затем по дате убыванию, затем по идентификатору
public sealed class OrderComparer : IComparer<Order>
{
    public int Compare(Order? x, Order? y)
    {
        if (ReferenceEquals(x, y)) return 0;
        if (x is null) return -1;
        if (y is null) return 1;

        int byStatus = x.Status.CompareTo(y.Status);
        if (byStatus != 0) return byStatus;

        int byDate = y.CreatedAt.CompareTo(x.CreatedAt);   // обратный порядок
        return byDate != 0 ? byDate : x.Id.CompareTo(y.Id);
    }
}

// В LINQ то же самое читается лучше
var sorted = orders
    .OrderBy(o => o.Status)
    .ThenByDescending(o => o.CreatedAt)
    .ThenBy(o => o.Id)
    .ToList();
```

Практика: в прикладном коде почти всегда достаточно `OrderBy`/`ThenBy`; отдельный `IComparer<T>` нужен, когда сравнение переиспользуется, передаётся в `SortedSet`/`SortedDictionary` или содержит нетривиальную логику.

---

## Сортировка: что стабильно, а что нет

| Механизм | Алгоритм | Стабильность | Память | Где |
|---|---|---|---|---|
| `Array.Sort`, `List<T>.Sort` | introsort | **нестабильная** | `O(log n)` | на месте |
| `OrderBy`/`ThenBy` | сортировка слиянием | **стабильная** | `O(n)` | новая последовательность |
| `SortedSet<T>`, `SortedDictionary<K,V>` | красно-чёрное дерево | — | `O(n)` | всегда упорядочены |
| `SortedList<K,V>` | отсортированный массив | — | `O(n)` | быстрый поиск, медленная вставка |

Стабильность означает, что элементы с одинаковым ключом сохраняют исходный взаимный порядок. Это заметно в постраничном выводе: нестабильная сортировка по неуникальному полю может выдавать один и тот же элемент на двух страницах подряд. Отсюда правило для API с пагинацией: **всегда добавляй уникальный ключ последним критерием** (`ThenBy(o => o.Id)`) — тогда порядок детерминирован независимо от алгоритма и от того, сортирует ли база или приложение.

Сортировка строк отдельно опасна: по умолчанию `OrderBy(s => s)` использует культуру текущего потока, и результат отличается между машинами. Для машинных данных — `StringComparer.Ordinal`, для интерфейса — культурный компаратор ([[Строки и работа с текстом]]).

---

## Что генерирует record, а что нет

```csharp
public readonly record struct Money(decimal Amount, string Currency);

// Есть: Equals, GetHashCode, IEquatable<Money>, ==, !=, ToString, Deconstruct
// НЕТ: IComparable<Money>, операторов <, >, <=, >=
// Money a = ..., b = ...;  if (a < b) — не скомпилируется
```

`record` закрывает только равенство. Порядок нужно добавлять руками:

```csharp
public readonly record struct Money(decimal Amount, string Currency) : IComparable<Money>
{
    public int CompareTo(Money other) => Currency == other.Currency
        ? Amount.CompareTo(other.Amount)
        : throw new InvalidOperationException($"Нельзя сравнивать {Currency} и {other.Currency}");

    public static bool operator <(Money a, Money b) => a.CompareTo(b) < 0;
    public static bool operator >(Money a, Money b) => a.CompareTo(b) > 0;
    public static bool operator <=(Money a, Money b) => a.CompareTo(b) <= 0;
    public static bool operator >=(Money a, Money b) => a.CompareTo(b) >= 0;
}
```

Кортежи, в отличие от записей, сравниваются и на равенство, и на порядок покомпонентно — это удобный способ получить составное сравнение бесплатно:

```csharp
public int CompareTo(Order? other)
    => (Status, CreatedAt, Id).CompareTo((other!.Status, other.CreatedAt, other.Id));
```

> [!warning] Подводные камни
> - **`CompareTo`, несогласованный с `Equals`.** `SortedSet` схлопнет «равные по порядку» элементы, а `HashSet` — нет; коллекции разойдутся по содержимому.
> - **Отсутствие `IComparable` у ключа `SortedSet`/`SortedDictionary`.** Исключение возникает при первой вставке, а не при создании — легко пропустить в тестах на пустой коллекции.
> - **Нестабильная сортировка в пагинации.** Дубликаты и пропуски между страницами; лечится уникальным ключом в конце сортировки.
> - **Сортировка строк без компаратора.** Зависит от культуры процесса: разные результаты локально и на сервере.
> - **`CompareTo` с вычитанием.** `x.Id - y.Id` для `int` может переполниться и дать неверный знак; всегда `CompareTo`.
> - **`NaN` в сравнениях.** `double.NaN` не равен ничему и не упорядочивается, поэтому нарушает полный порядок и ломает сортировку.
> - **Изменение объекта, лежащего в `SortedSet`.** Позиция в дереве не пересчитывается — элемент становится недостижим, как и в случае с хешем.
> - **`Comparison<T>` из лямбды в горячем цикле.** Замыкание создаёт делегат на каждый вызов; для многократной сортировки лучше статический компаратор-синглтон.

> [!example] Как делают в бою
> Внутри типа реализуют только **одно** сравнение — то, которое естественно и очевидно: у `Money` — по сумме в одной валюте, у `Version` — по номерам, у сущности — равенство по идентификатору. Все остальные варианты («по дате», «по имени без учёта регистра», «сначала VIP») живут снаружи как `IComparer<T>`/`IEqualityComparer<T>` или как выражения `OrderBy` в конкретном запросе.
> Компараторы делают статическими синглтонами без состояния: `public static readonly XComparer Instance = new();` — они переиспользуются и не создают мусора.
> В API с пагинацией сортировка всегда доопределяется уникальным полем, а на стороне БД сортируют средствами SQL с индексом, а не в памяти после `ToList()` — иначе с ростом данных запрос деградирует ([[Индексы: как работают и когда помогают]]).
> Для словарей с ключами-строками почти всегда явно передают `StringComparer.OrdinalIgnoreCase` или `Ordinal`: умолчание ординальное с учётом регистра, и это редко то, что нужно для заголовков, кодов и идентификаторов.

---

## Вопросы с собеседований

> [!question]- Зачем нужен `IEquatable<T>`, если есть `override Equals(object)`?
> Ради типобезопасности и производительности. `EqualityComparer<T>.Default`, который используют `Dictionary`, `HashSet`, `List.Contains`, `Distinct` и другие, проверяет, реализует ли тип `IEquatable<T>`, и если да — вызывает типизированный `Equals(T)` напрямую. Иначе вызывается `Equals(object)`, что для значимых типов означает упаковку **обоих** операндов на каждое сравнение: это аллокации в куче и заметная деградация в горячем коде. Для ссылочных типов выигрыш скромнее — исчезают приведение типа и связанная с ним проверка. При этом `IEquatable<T>` не заменяет переопределение `Equals(object)` и `GetHashCode`: их нужно реализовать всё равно, иначе код, работающий через `object`, получит другое поведение. `record` генерирует все три члена автоматически.

> [!question]- Что должен возвращать `CompareTo` и какие у него требования?
> Отрицательное число, если текущий объект меньше аргумента, ноль при равенстве по порядку и положительное, если больше; конкретные значения (−1/1 против −5/17) не имеют значения. Требования: полный порядок — антисимметричность (знаки при обмене операндов противоположны), транзитивность и согласованность при повторных вызовах; сравнение с `null` возвращает положительное число, потому что `null` считается меньше любого объекта, и исключение бросать нельзя. Отдельное соглашение: если `CompareTo` вернул 0, то `Equals` должен вернуть `true` — иначе `SortedSet` и `HashSet` будут по-разному определять дубликаты. Типичная ошибка реализации — вычитание вместо `CompareTo`: `x.Id - y.Id` для больших `int` переполняется и даёт неправильный знак.

> [!question]- Чем `List.Sort` отличается от `OrderBy`?
> `List<T>.Sort` и `Array.Sort` сортируют **на месте**, используют introsort (гибрид быстрой, пирамидальной и вставками), работают за `O(n log n)` с `O(log n)` дополнительной памяти и **нестабильны** — элементы с равными ключами могут поменяться местами. `OrderBy` из LINQ возвращает новую последовательность, использует стабильную сортировку слиянием, требует `O(n)` дополнительной памяти и выполняется отложенно: сортировка произойдёт при первом перечислении. Практические следствия: если исходный порядок равных элементов важен (пагинация, отображение групп), нужен `OrderBy` либо доопределение сортировки уникальным ключом; если важна память и коллекция уже своя — `Sort`. Ещё одно отличие: `OrderBy` над `IQueryable` транслируется в `ORDER BY` на стороне БД, а `Sort` сначала вытянет всё в память.

> [!question]- Когда сравнение выносят во внешний компаратор?
> Когда правил больше одного или когда правило зависит от контекста, а не от природы типа. У типа может быть только одно естественное сравнение (`Equals`/`CompareTo`), и попытка уместить в него несколько смыслов приводит к тому, что поведение коллекций становится непредсказуемым для читателя. Внешние `IEqualityComparer<T>` и `IComparer<T>` передаются в конструкторы `Dictionary`, `HashSet`, `SortedSet`, `SortedDictionary` и в LINQ-методы `Distinct`, `GroupBy`, `Join`, `OrderBy`, поэтому каждое место использования явно указывает, по какому правилу сравнивает. Дополнительный плюс — компаратор можно тестировать отдельно и переиспользовать. Практическая деталь: компараторы не имеют состояния, поэтому их делают статическими синглтонами, а не создают на каждый вызов.

---

## Задачи

### Задача 1. Сортировка с приоритетом

Отсортируй заказы: сначала VIP-клиенты, затем по дате создания (новые первыми), затем по идентификатору. Сделай двумя способами — LINQ и `IComparer<T>`.

> [!success]- Решение
> ```csharp
> // 1. LINQ — читаемо, годится для разовой сортировки
> var sorted = orders
>     .OrderByDescending(o => o.Customer.IsVip)     // true идёт первым
>     .ThenByDescending(o => o.CreatedAt)
>     .ThenBy(o => o.Id)
>     .ToList();
>
> // 2. Компаратор — если правило переиспользуется или нужно для SortedSet
> public sealed class OrderPriorityComparer : IComparer<Order>
> {
>     public static readonly OrderPriorityComparer Instance = new();
>
>     public int Compare(Order? x, Order? y)
>     {
>         if (ReferenceEquals(x, y)) return 0;
>         if (x is null) return -1;
>         if (y is null) return 1;
>
>         return (!x.Customer.IsVip, DateTimeOffset.MaxValue - x.CreatedAt, x.Id)
>             .CompareTo((!y.Customer.IsVip, DateTimeOffset.MaxValue - y.CreatedAt, y.Id));
>     }
> }
> orders.Sort(OrderPriorityComparer.Instance);
> ```
> Приёмы: инверсия булева значения (`!IsVip`) даёт «сначала VIP» при обычном возрастающем сравнении; вычитание из `DateTimeOffset.MaxValue` разворачивает порядок дат без отдельной ветки; кортеж сравнивается покомпонентно, поэтому составное правило умещается в одно выражение. Завершающее сравнение по `Id` обязательно: оно делает порядок детерминированным, что критично для пагинации, так как `List.Sort` нестабилен.

### Задача 2. Найти ошибку

```csharp
public class Task : IComparable<Task>
{
    public int Priority { get; set; }
    public string Title { get; set; } = "";

    public int CompareTo(Task? other) => Priority - other!.Priority;
    public override bool Equals(object? obj) => obj is Task t && t.Title == Title;
    public override int GetHashCode() => Title.GetHashCode();
}

var set = new SortedSet<Task>();
set.Add(new Task { Priority = 1, Title = "A" });
set.Add(new Task { Priority = 1, Title = "B" });
Console.WriteLine(set.Count);
```

> [!success]- Решение
> Выведет `1`: `SortedSet` определяет дубликаты по компаратору, а `CompareTo` вернул 0 для разных задач с одинаковым приоритетом — вторая просто не добавилась. Это и есть последствие рассогласования `CompareTo` и `Equals`: по равенству объекты различны, по порядку — одинаковы.
> Другие ошибки: `Priority - other.Priority` может переполниться при больших значениях и дать неверный знак; `other!` подавляет предупреждение вместо корректной обработки `null` (по контракту должно возвращаться положительное число); свойства изменяемые, поэтому объект, уже лежащий в `SortedSet` или `HashSet`, «потеряется» при изменении `Priority` или `Title`.
> ```csharp
> public sealed class TaskItem : IComparable<TaskItem>, IEquatable<TaskItem>
> {
>     public required int Priority { get; init; }
>     public required string Title { get; init; }
>
>     public int CompareTo(TaskItem? other)
>     {
>         if (other is null) return 1;
>         int byPriority = Priority.CompareTo(other.Priority);
>         return byPriority != 0
>             ? byPriority
>             : string.Compare(Title, other.Title, StringComparison.Ordinal);   // доопределяем
>     }
>
>     public bool Equals(TaskItem? other)
>         => other is not null && string.Equals(Title, other.Title, StringComparison.Ordinal);
>
>     public override bool Equals(object? obj) => Equals(obj as TaskItem);
>     public override int GetHashCode() => Title.GetHashCode(StringComparison.Ordinal);
> }
> ```
> Теперь `CompareTo` возвращает 0 ровно тогда, когда `Equals` даёт `true` (оба смотрят на `Title` как на завершающий критерий), свойства неизменяемы, а переполнение исключено.

---

## Итог

- Равенство и порядок — разные механизмы: `Equals`/`GetHashCode`/`IEquatable<T>` против `CompareTo`/`IComparable<T>`.
- `IEquatable<T>` избавляет от упаковки при сравнении структур и используется `EqualityComparer<T>.Default`.
- `CompareTo`: отрицательное/ноль/положительное, полный порядок, `null` меньше всего, согласованность с `Equals`.
- Альтернативные правила выносят в `IEqualityComparer<T>`/`IComparer<T>` и передают в коллекцию или LINQ.
- `List.Sort` нестабилен и работает на месте; `OrderBy` стабилен и создаёт новую последовательность.
- В пагинации сортировку всегда доопределяют уникальным ключом.
- `record` генерирует равенство, но не порядок: `IComparable<T>` и операторы сравнения пишутся руками.
- Ключи сортированных и хеш-коллекций должны быть неизменяемыми.

## Связанное

- [[Object: Equals, GetHashCode, ToString]] — контракт равенства и хеша
- [[Записи (record) и структуры]] — что генерируется автоматически
- [[SortedDictionary, SortedSet, SortedList]] · [[Dictionary и HashSet изнутри]]
- [[Обзор коллекций .NET и как выбирать]] · [[LINQ: полный справочник операторов]]
- [[Строки и работа с текстом]] — `StringComparer` и культура при сортировке
- [[Boxing и unboxing]] — почему `IEquatable<T>` важен для структур
- [[Иммутабельность как приём проектирования]]
