---
tags: [раздел-05, linq, основы, middle, собес, dotnet10]
aliases: [LINQ, Language Integrated Query, Query syntax, Method syntax, Синтаксис запросов LINQ]
---

# LINQ: основы и два синтаксиса

> [!abstract] Коротко
> LINQ (Language Integrated Query) — это не библиотека для баз данных, а набор extension-методов над `IEnumerable<T>`/`IQueryable<T>` плюс встроенный в язык синтаксис запросов. Одни и те же операторы (`Where`, `Select`, `OrderBy`) работают над списками в памяти, таблицами в СУБД, XML-документами и параллельными конвейерами — меняется только провайдер. Есть два равноправных синтаксиса: method syntax (цепочка методов) и query syntax (`from ... where ... select ...`), причём компилятор механически переписывает второй в первый. Всё, что делает LINQ, — строит объект-конвейер; данные не двигаются, пока конвейер не начнут перечислять.

## Зачем это нужно

Задача: из списка заказов выбрать оплаченные, отсортировать по сумме и взять пять самых крупных. Вот как это выглядело в C# 2 — до LINQ:

```csharp
// C# 2: делегаты уже есть, но нет лямбд, нет цепочек, нет анонимных типов
List<Order> paid = orders.FindAll(delegate(Order o) { return o.IsPaid; });
paid.Sort(delegate(Order a, Order b) { return b.Total.CompareTo(a.Total); });
List<Order> top = paid.GetRange(0, Math.Min(5, paid.Count));
```

А в C# 1 и `ArrayList` — ещё хуже: приходилось писать отдельный класс-компаратор:

```csharp
// C# 1: нетипизированная коллекция + отдельный класс ради одной сортировки
public class OrderByTotalDesc : IComparer
{
    public int Compare(object a, object b)
        => ((Order)b).Total.CompareTo(((Order)a).Total);
}

ArrayList list = new ArrayList();
// ...заполнение, потом ручной цикл фильтрации...
list.Sort(new OrderByTotalDesc());
```

Проблемы очевидны: `FindAll` есть у `List<T>`, но нет у массива и словаря; `Sort` мутирует исходную коллекцию; каждый шаг материализует новый список; композиция шагов невозможна. Сегодня то же самое — одна декларативная фраза:

```csharp
var top = orders
    .Where(o => o.IsPaid)
    .OrderByDescending(o => o.Total)
    .Take(5);
```

### Что понадобилось языку, чтобы это стало возможно

LINQ появился в C# 3 (2007) не как библиотека, а как результат нескольких языковых фич, каждая из которых по отдельности выглядела мелкой.

| Фича C# 3 | Что даёт LINQ | Без неё было бы |
|---|---|---|
| Extension-методы | `Where`/`Select` доступны у любого `IEnumerable<T>`, включая массивы и `string` | пришлось бы добавлять методы в сам интерфейс — сломало бы все реализации |
| Лямбда-выражения | `o => o.IsPaid` вместо `delegate(Order o) { return ...; }` | шумный синтаксис `delegate`, запросы нечитаемы |
| Выведение типов лямбд | тип параметра берётся из `Func<TSource, bool>` | пришлось бы писать типы вручную в каждом шаге |
| Анонимные типы | `select new { o.Id, o.Total }` — проекция без объявления класса | класс-DTO на каждую проекцию |
| `var` (неявная типизация) | переменная может держать анонимный тип и `OrderedEnumerable` | анонимные типы были бы бесполезны — их тип нельзя назвать |
| Инициализаторы объектов и коллекций | `select new Dto { Id = o.Id }` внутри выражения | проекция в существующий тип требовала бы конструктора под каждый набор полей |
| Деревья выражений (`Expression<Func<...>>`) | `IQueryable` может *разобрать* лямбду и превратить её в SQL | LINQ работал бы только в памяти |
| Синтаксис запросов (`from...select`) | SQL-подобная запись сложных join и `let` | только цепочки методов |

Про деревья выражений подробно — [[Деревья выражений]], про механику extension-методов — [[Расширяющие методы и extension members]], про лямбды и замыкания — [[Делегаты, события и лямбды]].

## Что такое LINQ и чем он не является

LINQ — это три вещи, соединённые вместе:

1. **Набор стандартных операторов запроса** (standard query operators) — около 60 методов с фиксированными именами и семантикой.
2. **Языковой синтаксис** — ключевые слова `from`, `where`, `select` и остальные.
3. **Соглашение о провайдерах** — любой тип, у которого есть методы с нужными именами, работает с этим синтаксисом.

```
                     ┌──────────────────────────────┐
   query syntax ───► │ переписывание компилятором   │
                     └──────────────┬───────────────┘
                                    ▼
   method syntax ───────► вызовы Where/Select/OrderBy
                                    │
        ┌───────────────────────────┼───────────────────────────┐
        ▼                           ▼                           ▼
  IEnumerable<T>              IQueryable<T>            IAsyncEnumerable<T>
  Enumerable                  Queryable                AsyncEnumerable
  (делегаты, в памяти)        (деревья выражений)      (асинхронный конвейер)
        │                           │
        ▼                           ▼
  LINQ to Objects            LINQ to Entities (EF Core), LINQ to SQL
  ParallelEnumerable         сторонние провайдеры (MongoDB, Elastic, OData)
  (PLINQ)
```

| Провайдер | Класс операторов | Над чем работает | Как исполняется |
|---|---|---|---|
| LINQ to Objects | `System.Linq.Enumerable` | `IEnumerable<T>` | делегаты, цикл в памяти |
| LINQ to Entities | `System.Linq.Queryable` | `IQueryable<T>` | дерево выражений → SQL |
| PLINQ | `System.Linq.ParallelEnumerable` | `ParallelQuery<T>` | пул потоков, разбиение по партициям |
| LINQ to XML | `System.Xml.Linq` + `Enumerable` | `XElement`, `XDocument` | обход дерева в памяти |
| Асинхронный LINQ | `System.Linq.AsyncEnumerable` | `IAsyncEnumerable<T>` | `await foreach` |

Чем LINQ **не** является:

- **Не библиотека для доступа к БД.** LINQ ничего не знает про SQL. SQL генерирует провайдер (EF Core), а LINQ даёт ему только форму запроса.
- **Не оптимизатор.** `Where(...).Where(...)` в LINQ to Objects даст два вложенных итератора и два вызова делегата на элемент; никто их не склеит. Оптимизация — забота провайдера, и `Enumerable` этим почти не занимается. См. [[LINQ: производительность и аллокации]].
- **Не «функциональный язык внутри C#».** Операторы чистые (не мутируют источник), но лямбды никто не проверяет: побочные эффекты внутри `Select` вполне возможны и вполне ломают код при повторном перечислении.
- **Не бесплатная абстракция.** Каждый оператор — минимум один объект-итератор и один делегат.

## Два синтаксиса

Один и тот же запрос, два способа записи:

```csharp
var orders = new[]
{
    new { Id = 1, Customer = "Ali",  Total = 300m, IsPaid = true  },
    new { Id = 2, Customer = "Bob",  Total = 120m, IsPaid = true  },
    new { Id = 3, Customer = "Ali",  Total = 900m, IsPaid = false },
};

// method syntax (fluent)
var m = orders
    .Where(o => o.IsPaid)
    .OrderByDescending(o => o.Total)
    .Select(o => o.Customer);

// query syntax
var q = from o in orders
        where o.IsPaid
        orderby o.Total descending
        select o.Customer;

Console.WriteLine(string.Join(", ", q));   // Вывод: Ali, Bob
```

Разницы в поведении нет никакой: `q` и `m` — объекты одного и того же типа. Query syntax — синтаксический сахар, и об этом важно помнить: он не даёт ни новых возможностей провайдеру, ни другой производительности.

### Таблица соответствия ключевых слов и методов

| Ключевое слово | Во что превращается | Комментарий |
|---|---|---|
| `from x in src` (первое) | сам `src` | задаёт источник, вызова метода нет |
| `from y in x.Items` (второе и далее) | `SelectMany` | плюс анонимный тип, чтобы сохранить `x` и `y` |
| `where` | `Where` | только перегрузка без индекса |
| `select` | `Select` | если проекция тождественная и это единственный шаг — компилятор может опустить `Select` |
| `orderby k` | `OrderBy(x => k)` | |
| `orderby k descending` | `OrderByDescending(x => k)` | |
| `orderby k1, k2` | `OrderBy(k1).ThenBy(k2)` | вторичный ключ — через запятую, а не вторым `orderby` |
| `orderby k1, k2 descending` | `OrderBy(k1).ThenByDescending(k2)` | `descending` действует только на свой ключ |
| `group x by k` | `GroupBy(x => k)` | результат — `IEnumerable<IGrouping<TKey, TSource>>` |
| `group p by k into g` | `GroupBy` + продолжение запроса | `g` — группа, у неё есть `Key` |
| `join b in src2 on a.K equals b.K` | `Join` | только по равенству ключей, только `equals` |
| `join b in src2 on ... into gb` | `GroupJoin` | `gb` — коллекция совпадений (возможно пустая) |
| `let t = expr` | `Select(x => new { x, t = expr })` | вводит вычисленную переменную |
| `into` | продолжение запроса (query continuation) | закрывает текущий запрос и начинает новый над его результатом |
| `equals`, `on`, `by`, `ascending` | часть синтаксиса join/group/orderby | самостоятельных методов не имеют |

### Что есть только в одном синтаксисе

| Возможность | query | method | Почему |
|---|---|---|---|
| `let` — именованное промежуточное значение | да | нет прямого аналога | в method syntax эмулируется `Select` в анонимный тип, читается хуже |
| Прозрачные идентификаторы при нескольких `from` | да | нет | компилятор сам протаскивает переменные через анонимные типы |
| `join ... into` с удобным именем группы | да | `GroupJoin` вручную | в query syntax короче на порядок |
| `Skip` / `Take` / `Chunk` | нет | да | ключевых слов не существует |
| `First` / `Single` / `Last` / `ElementAt` | нет | да | возвращают элемент, а не последовательность — запрос не может ими закончиться |
| `Count` / `Sum` / `Average` / `Aggregate` | нет | да | приходится оборачивать: `(from ... select x).Count()` |
| `Any` / `All` / `Contains` | нет | да | часто пишут внутри `where` как вызов метода |
| `Distinct`, `Union`, `Except`, `Intersect` | нет | да | |
| Всё новое с .NET 6: `DistinctBy`, `MinBy`, `Chunk`, `Order()` | нет | да | новые операторы ключевых слов не получают |
| **`LeftJoin` / `RightJoin` (.NET 10)** | нет | да | ключевого слова нет и не планируется; в query syntax левое соединение по-прежнему пишут через `join ... into` + `DefaultIfEmpty` |

Правило: запрос в query syntax обязан заканчиваться на `select` или `group`. Всё, что возвращает одно значение, дописывают снаружи:

```csharp
var count = (from o in orders where o.IsPaid select o).Count();
```

Именно поэтому в реальном коде query syntax почти всегда смешан с method syntax.

## Механика query syntax: переписывание до разрешения перегрузок

Ключевой факт, который объясняет всё остальное поведение: **компилятор переписывает query syntax в вызовы методов чисто синтаксически, ещё до того, как узнает типы**. Никакой магии, никакого специального знания про `IEnumerable<T>` в этом шаге нет.

Возьмём запрос с несколькими источниками, `let` и сортировкой:

```csharp
var q = from o in orders
        from i in o.Items
        let line = i.Price * i.Quantity
        where line > 100m
        orderby line descending
        select new { o.Id, i.Sku, line };
```

Компилятор превращает его примерно в это (имена анонимных типов и переменных синтетические — я подставил читаемые):

```csharp
var q = orders
    .SelectMany(o => o.Items, (o, i) => new { o, i })          // прозрачный идентификатор №1
    .Select(t1 => new { t1, line = t1.i.Price * t1.i.Quantity }) // прозрачный идентификатор №2
    .Where(t2 => t2.line > 100m)
    .OrderByDescending(t2 => t2.line)
    .Select(t2 => new { t2.t1.o.Id, t2.t1.i.Sku, t2.line });
```

Анонимные типы, которые компилятор создаёт, чтобы «дотащить» `o`, `i` и `line` до финального `select`, и называются **прозрачными идентификаторами** (transparent identifiers): в исходнике их не видно, а в IL они есть. Отсюда два практических следствия:

- Каждый `let` и каждый дополнительный `from` — это ещё одна аллокация анонимного класса **на элемент**. В горячем пути это заметно.
- В стеке исключения и в отладчике вы увидите поля вида `<>h__TransparentIdentifier0`. Это нормально, а не признак поломки.

### Query syntax работает с любым типом, у которого есть нужные методы

Раз переписывание синтаксическое, `from ... select` применим не только к коллекциям. Достаточно, чтобы у типа были методы с правильными именами и подходящими сигнатурами. Классический пример — «опциональное значение»:

```csharp
public readonly struct Option<T>
{
    private readonly T? _value;
    private readonly bool _hasValue;

    private Option(T value)
    {
        _value = value;
        _hasValue = true;
    }

    public static Option<T> Some(T value) => new(value);
    public static Option<T> None => default;

    // именно эти два метода и нужны query syntax
    public Option<TResult> Select<TResult>(Func<T, TResult> selector)
        => _hasValue ? Option<TResult>.Some(selector(_value!)) : Option<TResult>.None;

    public Option<T> Where(Func<T, bool> predicate)
        => _hasValue && predicate(_value!) ? this : None;

    public override string ToString() => _hasValue ? $"Some({_value})" : "None";
}
```

```csharp
var ok = from x in Option<int>.Some(42)
         where x > 10
         select x * 2;
Console.WriteLine(ok);        // Вывод: Some(84)

var no = from x in Option<int>.Some(5)
         where x > 10
         select x * 2;
Console.WriteLine(no);        // Вывод: None
```

`Option<T>` не реализует `IEnumerable<T>` и вообще не является коллекцией — но синтаксис работает, потому что компилятор просто позвал `.Where(...).Select(...)`. Тот же трюк лежит в основе PLINQ (`ParallelQuery<T>`) и `IQueryable<T>`: они не наследуют операторы, а объявляют свои с теми же именами. Как писать такие операторы самому — [[Свои операторы LINQ]].

## Что на самом деле возвращает запрос

Запрос — это **не коллекция**, а объект-конвейер, который знает, откуда брать данные и что с ними делать:

```csharp
var numbers = new List<int> { 1, 2, 3 };
var query = numbers.Where(n => n > 1).Select(n => n * 10);

Console.WriteLine(query.GetType().Name);
// Вывод: SelectEnumerableIterator`2   (внутренний класс System.Linq)

numbers.Add(4);                              // меняем ИСТОЧНИК после создания запроса
Console.WriteLine(string.Join(", ", query)); // Вывод: 20, 30, 40
```

Четвёрка попала в результат, потому что перечисление началось только на `string.Join`. Это отложенное выполнение (deferred/lazy execution) — отдельная большая тема, см. [[Отложенное и немедленное выполнение]]. Для `IQueryable<T>` отложенность ещё важнее: до `ToList()` запрос вообще не превращается в SQL и не уезжает в базу — [[IEnumerable vs IQueryable]].

### Почему `var` часто обязателен

```csharp
var proj = orders.Select(o => new { o.Id, o.Total });
// тип элемента — анонимный, у него нет имени в исходном коде.
// IEnumerable<???> proj = ... написать невозможно
```

Анонимный тип существует в метаданных, но его имя (`<>f__AnonymousType0`2`) невыразимо в C#. Поэтому проекция в анонимный тип и `var` — неразделимая пара. Если тип нужно назвать (вернуть из метода, положить в поле) — берите `record` или кортеж:

```csharp
// кортеж: имя типа выразимо, работает как возвращаемое значение метода
IEnumerable<(int Id, decimal Total)> Pairs(IEnumerable<Order> src)
    => src.Select(o => (o.Id, o.Total));
```

Подробнее про кортежи — [[Записи (record) и структуры]].

## Композиция и переиспользование

Раз запрос — это объект, его можно собирать по частям. Это главный практический приём, ради которого стоит понимать отложенность.

```csharp
IQueryable<Order> BuildQuery(IQueryable<Order> src, OrderFilter f)
{
    var q = src;

    if (f.OnlyPaid)
        q = q.Where(o => o.IsPaid);

    if (f.Customer is not null)
        q = q.Where(o => o.Customer == f.Customer);

    if (f.MinTotal is { } min)
        q = q.Where(o => o.Total >= min);

    return q.OrderByDescending(o => o.CreatedAt);
}
```

Ни один `Where` не обращается к базе: собирается одно дерево выражений, из которого EF Core сделает один `SELECT` с нужным набором условий в `WHERE`. Альтернатива в виде строковой склейки SQL — прямой путь к инъекциям.

Предикаты для LINQ to Objects выносят в статические свойства или методы:

```csharp
public static class OrderPredicates
{
    public static Func<Order, bool> Overdue(DateOnly today)
        => o => !o.IsPaid && o.DueDate < today;
}

var overdue = orders.Where(OrderPredicates.Overdue(DateOnly.FromDateTime(DateTime.UtcNow)));
```

Для `IQueryable` то же самое, но тип должен быть `Expression<Func<Order, bool>>` — иначе провайдер не сможет разобрать предикат и либо упадёт, либо (в старых версиях EF) молча вытянет таблицу в память. Развитие этой идеи — [[Specification pattern]].

## Стиль: что выбирать

| Ситуация | Синтаксис | Причина |
|---|---|---|
| 1–3 оператора в цепочке | method | короче и без `select x` в конце |
| Нужны `Take`, `First`, агрегаты | method | ключевых слов не существует |
| Два и более `join` | query | `on ... equals` читается, вложенные `Join` с анонимными типами — нет |
| Нужны промежуточные вычисления (`let`) | query | в method syntax это цепочка `Select` в анонимные типы |
| Декартово произведение / несколько `from` | query | компилятор сам строит `SelectMany` с прозрачными идентификаторами |
| Группировка с последующей агрегацией | query | `group ... into g` читается ближе к SQL |
| Код в библиотеке, который расширяют | method | легко вставить условный `Where` в середину |

Форматирование длинных цепочек: точка в начале строки, один оператор на строку, лямбды короткие. Так diff в code review показывает изменение одного шага, а не всей строки.

```csharp
var report = orders
    .Where(o => o.IsPaid)
    .GroupBy(o => o.Customer)
    .Select(g => new
    {
        Customer = g.Key,
        Count    = g.Count(),
        Total    = g.Sum(o => o.Total),
    })
    .OrderByDescending(x => x.Total)
    .Take(10)
    .ToList();
```

> [!warning] Подводные камни
> **1. `var q = ...` выглядит как коллекция, но ею не является.** Тип запроса — итератор, а не список. Отсюда три класса ошибок: результат меняется при изменении источника; `q.Count()` каждый раз перечисляет заново; при работе с EF Core запрос уезжает в базу столько раз, сколько раз вы его перечислили. Правило: как только запрос закончен и результат нужен больше одного раза — `ToList()`.
>
> **2. Переменная `for`-цикла в замыкании.** Лямбда захватывает *переменную*, а не её значение. `foreach` в C# 5 исправили — его переменная создаётся заново на каждой итерации, а вот `for` по-прежнему делит один слот:
> ```csharp
> var bad = new List<Func<int>>();
> for (int i = 0; i < 3; i++)
>     bad.Add(() => i);                  // все три лямбды захватили ОДНУ переменную i
> Console.WriteLine(string.Join(",", bad.Select(f => f())));   // Вывод: 3,3,3
>
> var good = new List<Func<int>>();
> for (int i = 0; i < 3; i++)
> {
>     int copy = i;                      // локальная копия на итерацию
>     good.Add(() => copy);
> }
> Console.WriteLine(string.Join(",", good.Select(f => f())));  // Вывод: 0,1,2
> ```
> Почему так: компилятор поднимает захваченную переменную в поле класса-замыкания; у `for` этот класс один на весь цикл. Детали — [[Делегаты, события и лямбды]].
>
> **3. Повторное перечисление с побочными эффектами.** Если внутри `Select` есть логирование, запись в счётчик или обращение к сети, каждое перечисление выполнит их снова. Особенно больно, когда запрос отдан наружу как `IEnumerable<T>`: вызывающий код перечислит его дважды и не поймёт, почему в логах дубли. Возвращайте `IReadOnlyList<T>`, если результат уже материализован.
>
> **4. `Count() > 0` вместо `Any()`.** Для `List<T>` разницы почти нет — `Count()` проверит `ICollection<T>.Count` и вернёт число за O(1). Но для отложенного запроса `Count()` прогонит **весь** конвейер, а `Any()` остановится на первом элементе. На `IQueryable` `Count()` даёт `SELECT COUNT(*)`, а `Any()` — `EXISTS`, который планировщик БД обычно выполняет дешевле. Плюс на бесконечной последовательности `Count()` просто не завершится.
>
> **5. Два `orderby` вместо составного ключа.** `orderby a` и следующей строкой `orderby b` — это `OrderBy(a).OrderBy(b)`, то есть **полная пересортировка**: первый ключ теряет силу и влияет только на порядок равных элементов (сортировка стабильна, поэтому эффект коварно похож на правильный на маленьких данных). Правильно — `orderby a, b` (то есть `OrderBy(a).ThenBy(b)`).
>
> **6. `let` не кеширует между перечислениями.** `let` — это `Select` в анонимный тип, вычисляемый на каждой итерации каждого перечисления. Если внутри `let` тяжёлый вызов, он выполнится столько раз, сколько раз перечислили запрос.

> [!example] Как делают в бою
> - **Слой репозитория возвращает `IQueryable<T>` только внутри сборки данных.** Наружу, за границу слоя, отдают `List<T>`/`IReadOnlyList<T>` или уже спроецированный DTO. Иначе ленивый запрос утекает в контроллер, `DbContext` к этому моменту может быть уже освобождён, и вы получаете `ObjectDisposedException` в рантайме вместо ошибки компиляции.
> - **Проекция в DTO прямо в запросе, а не после.** `Select(o => new OrderDto { ... })` в `IQueryable` превращается в `SELECT` только нужных колонок; `ToList()` перед проекцией тянет все поля и все навигации. Это самая частая причина «почему API отдаёт 200 мс на трёх строках».
> - **Фильтры собирают условно, как в примере с `BuildQuery`.** Это стандарт для endpoint-ов со списком и фасетным поиском.
> - **Query syntax живёт в отчётах и аналитике**, где реально нужны три `join` и `group into`. В прикладном коде сервисов — почти всегда method syntax.
> - **В code review красный флаг — `IEnumerable<T>` в возвращаемом типе публичного метода сервиса.** Он не говорит, материализован результат или нет, а значит вызывающий не знает цену второго `foreach`.
> - **`.Where(x => x.Id == id).FirstOrDefault()` схлопывают в `.FirstOrDefault(x => x.Id == id)`**: на один итератор меньше, и в EF Core генерируется тот же SQL.

## Вопросы с собеседований

> [!question]- LINQ — это библиотека или фича языка?
> И то и другое, но по частям. Библиотечная часть — статические классы `System.Linq.Enumerable` и `System.Linq.Queryable` с extension-методами; они лежат в BCL и работают безо всякой поддержки компилятора. Языковая часть — синтаксис `from ... select` и вспомогательные фичи C# 3 (лямбды, анонимные типы, `var`, extension-методы, деревья выражений). Компилятор знает про query syntax, но не знает про `Enumerable`: он переписывает запрос в вызовы методов по именам, а какие именно методы найдутся — решает обычное разрешение перегрузок. Поэтому query syntax работает над любым типом с методами `Where`/`Select`/`SelectMany`/`Join`/`GroupBy`/`OrderBy`.

> [!question]- Чем отличается query syntax от method syntax по производительности?
> Ничем: query syntax полностью переписывается в вызовы методов на этапе компиляции, в IL следов ключевых слов нет. Единственная реальная разница возникает косвенно — через то, *какие* методы вы в итоге вызываете. Несколько `from` и `let` в query syntax дают дополнительные `SelectMany`/`Select` с анонимными типами (прозрачные идентификаторы), то есть лишнюю аллокацию на элемент. Написав ту же логику вручную в method syntax, вы иногда обойдётесь меньшим числом шагов. Но это следствие структуры запроса, а не синтаксиса.

> [!question]- Что такое прозрачные идентификаторы (transparent identifiers)?
> Это синтетические анонимные типы, которые компилятор вставляет, чтобы протащить переменные из ранних клауз запроса в поздние. Возникают при втором и последующих `from`, при `let`, при `join ... into`. Например `from o in orders from i in o.Items select ...` компилируется в `orders.SelectMany(o => o.Items, (o, i) => new { o, i })`, и дальше все клаузы работают с этим анонимным объектом, обращаясь к `t.o` и `t.i`. Практический эффект: аллокация на элемент на каждый такой шаг и странные имена вида `<>h__TransparentIdentifier0` в отладчике.

> [!question]- Почему у `LeftJoin` из .NET 10 нет ключевого слова в query syntax?
> Потому что набор ключевых слов LINQ зафиксирован в спецификации языка с C# 3, и добавление нового слова — breaking change для любого кода, где это слово используется как идентификатор. Все операторы, добавленные после C# 3 (`Zip`, `Chunk`, `DistinctBy`, `MinBy`, `Order()`, `LeftJoin`/`RightJoin` из .NET 10), доступны только через method syntax. В query syntax левое соединение по-прежнему записывают классически: `join ... into g` (это `GroupJoin`), затем `from x in g.DefaultIfEmpty()`. Сам `LeftJoin` при этом можно дописать снаружи к query-запросу, как и любой другой метод.

> [!question]- Почему для проекции в анонимный тип обязателен `var`?
> Анонимный тип генерируется компилятором, и его имя (`<>f__AnonymousType0`2`) содержит символы, недопустимые в идентификаторах C#. Назвать этот тип в исходнике невозможно, поэтому единственный способ объявить переменную такого типа — вывести тип из инициализатора, то есть `var`. Из этого же следует, что анонимный тип нельзя использовать как возвращаемый тип метода, тип поля или параметра — он живёт только внутри одного метода. Если нужно вынести проекцию за границу метода, берут `record`, `record struct` или кортеж.

> [!question]- Какие языковые фичи C# 3 сделали LINQ возможным и что было бы без каждой?
> Extension-методы — без них операторы пришлось бы объявить в `IEnumerable<T>`, сломав все существующие реализации интерфейса. Лямбды и выведение типов — без них каждый предикат был бы анонимным делегатом с явными типами, и запросы стали бы нечитаемыми. Анонимные типы плюс `var` — без них любая проекция требовала бы объявленного класса. Инициализаторы объектов и коллекций — без них проекция в существующий тип требовала бы конструктора под каждый набор полей. Деревья выражений — без них LINQ работал бы только в памяти, никакой трансляции в SQL не было бы.

> [!question]- Запрос вернул неожиданный результат после того, как изменили исходную коллекцию. Почему?
> Потому что запрос хранит ссылку на источник, а не его снимок. Операторы вроде `Where`/`Select`/`OrderBy` только строят цепочку итераторов; чтение источника происходит в момент перечисления. Если между построением запроса и `foreach` источник изменился, запрос увидит новое состояние. Хуже того, изменение `List<T>` во время активного перечисления бросит `InvalidOperationException` про изменённую коллекцию — потому что `List<T>.Enumerator` проверяет внутренний счётчик версий. Лечится материализацией: `ToList()`/`ToArray()` сразу после построения запроса.

> [!question]- Можно ли применить query syntax к своему типу, не реализуя `IEnumerable<T>`?
> Да. Переписывание запроса — чисто синтаксическая трансформация, выполняемая до разрешения перегрузок, поэтому достаточно объявить у типа обычные (или extension-) методы с нужными именами и сигнатурами: `Select`, `Where`, `SelectMany`, `OrderBy`, `GroupBy`, `Join`, `GroupJoin`. Именно так устроены `IQueryable<T>` (методы в `Queryable`) и PLINQ (`ParallelEnumerable` над `ParallelQuery<T>`). Тот же приём применяют к монадическим типам вроде `Option<T>`/`Result<T>`: пара `Select` + `SelectMany` даёт им поддержку `from ... select`.

## Задачи

### Задача 1. Перевести запрос из method syntax в query syntax

Дан запрос на цепочке методов. Перепишите его в query syntax так, чтобы результат был идентичным, и объясните, какая часть в query syntax записана не будет.

```csharp
var result = employees
    .Join(departments, e => e.DeptId, d => d.Id, (e, d) => new { e, d })
    .Where(x => x.d.City == "Tashkent")
    .Select(x => new { x.e.Name, Dept = x.d.Title, Bonus = x.e.Salary * 0.1m })
    .OrderByDescending(x => x.Bonus)
    .ThenBy(x => x.Name)
    .Take(5)
    .ToList();
```

> [!success]- Решение
> ```csharp
> var query = from e in employees
>             join d in departments on e.DeptId equals d.Id
>             where d.City == "Tashkent"
>             let bonus = e.Salary * 0.1m
>             orderby bonus descending, e.Name
>             select new { e.Name, Dept = d.Title, Bonus = bonus };
>
> var result = query.Take(5).ToList();
> ```
> Что важно:
> - `join ... on ... equals` заменил `Join` вместе с анонимным типом-склейкой: прозрачный идентификатор компилятор создаст сам, поэтому `x.e`/`x.d` превращаются в обычные `e`/`d`.
> - `let bonus = ...` убрал дублирование выражения между `select` и `orderby`.
> - Составная сортировка — через запятую в одном `orderby`. Два отдельных `orderby` дали бы `OrderBy(...).OrderBy(...)` и потеряли первый ключ.
> - `Take` и `ToList` ключевых слов не имеют, поэтому дописаны снаружи. Именно так выглядит нормальный смешанный стиль.

### Задача 2. Найти и объяснить три бага

Код компилируется и на тестовых данных даёт правильный ответ, а в проде ведёт себя не так. Найдите три дефекта.

```csharp
public IEnumerable<string> GetTopCustomers(IEnumerable<Order> orders)
{
    var paid = orders.Where(o =>
    {
        Console.WriteLine($"проверяю {o.Id}");
        return o.IsPaid;
    });

    if (paid.Count() > 0)
        return paid.OrderByDescending(o => o.Total).Select(o => o.Customer);

    return Enumerable.Empty<string>();
}
```

> [!success]- Решение
> ```csharp
> public IReadOnlyList<string> GetTopCustomers(IEnumerable<Order> orders)
> {
>     var paid = orders.Where(o => o.IsPaid).ToList();   // один проход, снимок данных
>
>     if (paid.Count == 0)
>         return [];
>
>     return paid
>         .OrderByDescending(o => o.Total)
>         .Select(o => o.Customer)
>         .ToList();
> }
> ```
> Дефекты:
> 1. **Побочный эффект в предикате.** `Console.WriteLine` выполняется при каждом перечислении, а перечислений здесь минимум два: `Count()` и потом обход результата вызывающим кодом. В логах будут дубли, и это ещё и синхронный I/O внутри горячего цикла.
> 2. **`Count() > 0` вместо `Any()`** — полный проход по источнику ради ответа «непусто ли». Для `IQueryable` это отдельный `SELECT COUNT(*)`. Здесь после материализации проблема исчезает сама: у `List<T>` есть свойство `Count` за O(1).
> 3. **Возврат ленивого `IEnumerable<string>`.** Метод отдаёт наружу конвейер над чужой коллекцией: вызывающий не знает, что второй `foreach` — это второй полный проход, а если источник был запросом к БД, `DbContext` к моменту перечисления может быть уже освобождён. Возврат `IReadOnlyList<T>` делает контракт честным.
>
> Бонус: `return [];` — collection expression, работает начиная с C# 12 и не аллоцирует для пустого массива.

### Задача 3. Дать query syntax своему типу

Есть тип `Result<T>` — «значение или ошибка». Добавьте ему поддержку query syntax так, чтобы работал запрос с `from`, `where` и `select`, а также цепочка из двух `from` (то есть `SelectMany`).

```csharp
public readonly struct Result<T>
{
    public T? Value { get; }
    public string? Error { get; }
    public bool IsOk => Error is null;
    // конструкторы и фабрики Ok/Fail считайте написанными
}
```

> [!success]- Решение
> ```csharp
> public readonly struct Result<T>
> {
>     public T? Value { get; }
>     public string? Error { get; }
>     public bool IsOk => Error is null;
>
>     private Result(T value) { Value = value; Error = null; }
>     private Result(string error) { Value = default; Error = error; }
>
>     public static Result<T> Ok(T value) => new(value);
>     public static Result<T> Fail(string error) => new(error);
>
>     public Result<TR> Select<TR>(Func<T, TR> selector)
>         => IsOk ? Result<TR>.Ok(selector(Value!)) : Result<TR>.Fail(Error!);
>
>     public Result<T> Where(Func<T, bool> predicate)
>         => IsOk && !predicate(Value!) ? Fail("предикат не выполнен") : this;
>
>     // нужен для запроса с двумя from
>     public Result<TR> SelectMany<TMid, TR>(
>         Func<T, Result<TMid>> collectionSelector,
>         Func<T, TMid, TR> resultSelector)
>     {
>         if (!IsOk) return Result<TR>.Fail(Error!);
>         var mid = collectionSelector(Value!);
>         if (!mid.IsOk) return Result<TR>.Fail(mid.Error!);
>         return Result<TR>.Ok(resultSelector(Value!, mid.Value!));
>     }
>
>     public override string ToString() => IsOk ? $"Ok({Value})" : $"Fail({Error})";
> }
> ```
> ```csharp
> static Result<int> ParseInt(string s)
>     => int.TryParse(s, out var n) ? Result<int>.Ok(n) : Result<int>.Fail($"не число: {s}");
>
> var sum = from a in ParseInt("40")
>           from b in ParseInt("2")
>           where a > 0
>           select a + b;
> Console.WriteLine(sum);            // Вывод: Ok(42)
>
> var bad = from a in ParseInt("40")
>           from b in ParseInt("x")
>           select a + b;
> Console.WriteLine(bad);            // Вывод: Fail(не число: x)
> ```
> Почему это работает: компилятор видит `from a in ... from b in ...` и генерирует вызов `SelectMany(a => ParseInt("2"), (a, b) => ...)`. Сигнатура моего `SelectMany` под этот вызов подходит, поэтому разрешение перегрузок проходит. Ни `IEnumerable<T>`, ни `System.Linq` здесь вообще не участвуют — это и есть доказательство того, что query syntax отвязан от коллекций. Обратите внимание: `Where` для `Result<T>` вынужден выдумывать текст ошибки — признак того, что фильтрация для этого типа не самая естественная операция, и в реальном коде её часто не объявляют вовсе.

## Итог

- LINQ = стандартные операторы запроса (`Enumerable`, `Queryable`, `ParallelEnumerable`, `AsyncEnumerable`) + синтаксис запросов + соглашение о провайдерах. К базам данных сам LINQ отношения не имеет.
- Query syntax переписывается компилятором в вызовы методов синтаксически, до разрешения перегрузок. Поэтому он работает над любым типом с методами нужных имён, а не только над коллекциями.
- Method syntax строго мощнее: `Take`, `First`, агрегаты, `Chunk`, `MinBy`, `LeftJoin`/`RightJoin` из .NET 10 ключевых слов не имеют. Query syntax выигрывает на нескольких `join`, `let` и множественных `from` благодаря прозрачным идентификаторам.
- Запрос — объект-конвейер, а не коллекция. Он хранит ссылку на источник и читает его в момент перечисления; каждое перечисление — новый полный проход.
- LINQ стал возможен благодаря extension-методам, лямбдам, анонимным типам, `var`, инициализаторам и деревьям выражений. Убери любую — и синтаксис рассыпается.
- Практика: собирайте `IQueryable` по частям, проецируйте в DTO внутри запроса, материализуйте перед выходом за границу слоя.

## Связанное

- [[LINQ: полный справочник операторов]]
- [[Отложенное и немедленное выполнение]]
- [[IEnumerable vs IQueryable]]
- [[Группировка, соединения, агрегация]]
- [[LINQ: производительность и аллокации]]
- [[Свои операторы LINQ]]
- [[Расширяющие методы и extension members]]
- [[Деревья выражений]]
