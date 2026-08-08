---
tags: [раздел-05, итераторы, ienumerable, yield, middle, собес, подводный-камень]
aliases: [IEnumerable, IEnumerator, yield return, Iterators, Итераторы, Энумераторы, Iterator state machine]
---

# IEnumerable, IEnumerator и yield return

> [!abstract] Коротко
> `IEnumerable<T>` — это «я умею выдать перечислитель», `IEnumerator<T>` — сам курсор с методами `MoveNext()` и свойством `Current`. `foreach` — синтаксический сахар: компилятор разворачивает его в `GetEnumerator()` + `while (MoveNext())` + `try/finally` с `Dispose()`, причём интерфейсы ему для этого не нужны — достаточно подходящих по форме методов (duck typing).
> `yield return` заставляет компилятор сгенерировать за вас класс-конечный автомат (state machine): тело метода разрезается на куски по точкам `yield`, локальные переменные становятся полями, а вызов метода лишь создаёт объект автомата — код не выполняется до первого `MoveNext()`. Отсюда все свойства итераторов: ленивость, отложенная валидация аргументов и `finally`, который срабатывает через `Dispose()`.

## Зачем это нужно

В C# 1.0 (2002) итератора не было. Чтобы по вашему типу работал `foreach`, вы писали два класса руками:

```csharp
// Как это выглядело до C# 2. Именно так до сих пор выглядят энумераторы в BCL.
public sealed class Countdown : IEnumerable
{
    private readonly int _from;
    public Countdown(int from) => _from = from;

    public IEnumerator GetEnumerator() => new CountdownEnumerator(_from);

    private sealed class CountdownEnumerator : IEnumerator
    {
        private readonly int _from;
        private int _current;
        private bool _started;

        public CountdownEnumerator(int from) { _from = from; _current = from + 1; }

        public bool MoveNext()
        {
            if (!_started) { _started = true; _current = _from; return _from >= 0; }
            _current--;
            return _current >= 0;
        }

        public object Current => _current;   // boxing на каждом элементе
        public void Reset() { _started = false; _current = _from + 1; }
    }
}
```

Двадцать строк инфраструктуры на три строки логики. В C# 2.0 (2005) появился `yield return`, и то же самое стало таким:

```csharp
public static IEnumerable<int> Countdown(int from)
{
    for (var i = from; i >= 0; i--)
        yield return i;
}
```

Компилятор пишет тот скучный класс за вас. Весь LINQ (`Where`, `Select`, `Take`) построен ровно на этом механизме — см. [[Отложенное и немедленное выполнение]].

## foreach — синтаксический сахар

### Точное разворачивание

Код:

```csharp
foreach (var x in src)
{
    Use(x);
}
```

Компилятор превращает в (для `src` статического типа `IEnumerable<int>`):

```csharp
{
    IEnumerator<int> e = src.GetEnumerator();
    try
    {
        while (e.MoveNext())
        {
            int x = e.Current;       // x — новая переменная на каждой итерации
            Use(x);
        }
    }
    finally
    {
        if (e is not null) e.Dispose();   // IEnumerator<T> : IDisposable, известно статически
    }
}
```

Для негенерик-случая (`src` типа `IEnumerable`) — вариант с проверкой в рантайме, потому что `IEnumerator` **не** наследует `IDisposable`:

```csharp
{
    IEnumerator e = src.GetEnumerator();
    try
    {
        while (e.MoveNext())
        {
            object x = e.Current;
            Use(x);
        }
    }
    finally
    {
        (e as IDisposable)?.Dispose();   // может и не быть IDisposable
    }
}
```

Как именно строится `finally` — зависит от статического типа энумератора:

| Статический тип энумератора | Что в `finally` | Аллокации |
|---|---|---|
| `IEnumerator<T>` | `e.Dispose()` | зависит от энумератора |
| `IEnumerator` (негенерик) | `(e as IDisposable)?.Dispose()` | приведение типа в рантайме |
| struct, реализующая `IDisposable` | `e.Dispose()` напрямую по значению | нет boxing |
| struct/class **без** `IDisposable` (duck typing) | `finally` не генерируется вообще | нет |

Три вывода, которые часто спрашивают:
1. Переменная цикла (`x`) — **новая на каждой итерации** (важно для замыканий, см. [[Локальные функции и замыкания]]).
2. `Current` читается один раз за итерацию. Если ваш `Current` считает что-то тяжёлое — это будет один вызов, но кэшировать его никто не станет.
3. `Dispose()` вызывается **всегда**: и при нормальном завершении, и при `break`, и при `return` из середины цикла, и при исключении.

### Duck typing в foreach

Компилятору `IEnumerable` не нужен. Правило языка: тип пригоден для `foreach`, если у него есть **доступный** метод `GetEnumerator()` без параметров, возвращающий тип, у которого есть метод `bool MoveNext()` и свойство `Current` с get-аксессором. Всё. Никаких интерфейсов.

```csharp
// Тип, по которому работает foreach, но который не реализует IEnumerable
public sealed class Bits
{
    private readonly uint _value;
    public Bits(uint value) => _value = value;

    public Enumerator GetEnumerator() => new(_value);

    public struct Enumerator
    {
        private uint _rest;
        private int _index;
        public Enumerator(uint value) { _rest = value; _index = -1; }

        public bool MoveNext()
        {
            if (_index >= 31) return false;
            _index++;
            Current = (_rest & (1u << _index)) != 0;
            return true;
        }

        public bool Current { get; private set; }
    }
}

var ones = 0;
foreach (var bit in new Bits(0b1011))   // компилируется: форма подходит
    if (bit) ones++;
Console.WriteLine(ones);
// Вывод: 3
```

Зачем это нужно на практике? Ровно за этим сделан `Span<T>`:

```csharp
Span<int> span = stackalloc int[4] { 1, 2, 3, 4 };
var sum = 0;
foreach (var x in span) sum += x;       // ноль аллокаций, ноль виртуальных вызовов
Console.WriteLine(sum);
// Вывод: 10
```

`Span<T>` — это `ref struct`, он **физически не может** реализовать `IEnumerable<T>` (ref struct до .NET 9 вообще не мог реализовывать интерфейсы, а боксить его нельзя никогда). Duck typing — единственная причина, по которой `foreach` по спану вообще работает. Подробнее — [[Span, ReadOnlySpan и Memory]].

> [!info] `GetEnumerator` как extension-метод (C# 9)
> С C# 9 `foreach` находит `GetEnumerator`, объявленный как расширяющий метод. Это позволяет «научить» `foreach` чужому типу, который вы не можете править:
> ```csharp
> public static class TupleRangeExtensions
> {
>     // Позволяет: foreach (var i in (1, 5))
>     public static IEnumerator<int> GetEnumerator(this (int start, int end) range)
>     {
>         for (var i = range.start; i <= range.end; i++)
>             yield return i;
>     }
> }
> ```
> В .NET 9+ в BCL этим приёмом добавлен `GetEnumerator` для `Range`, так что `foreach (var i in ..10)` работает. См. [[Расширяющие методы и extension members]] и [[Индексаторы, Index и Range]].

## Иерархия интерфейсов

| Интерфейс | Члены | Особенности |
|---|---|---|
| `IEnumerable` | `IEnumerator GetEnumerator()` | .NET 1.0, `System.Collections`. Основной путь — #устарело |
| `IEnumerator` | `bool MoveNext()`, `object Current`, `void Reset()` | `Current` типа `object` → boxing для value types |
| `IEnumerable<T>` | `IEnumerator<T> GetEnumerator()` | Ковариантен: `out T` |
| `IEnumerator<T>` | `T Current` + всё от `IEnumerator` | Наследует `IDisposable`. `Current` тоже `out T`-ковариантен по факту, но интерфейс инвариантен по `T`... нет, `IEnumerator<out T>` тоже ковариантен |
| `IReadOnlyCollection<T>` | `+ int Count` | Знает размер, не даёт мутировать |
| `IAsyncEnumerable<T>` | `IAsyncEnumerator<T> GetAsyncEnumerator(CancellationToken)` | `MoveNextAsync()` возвращает `ValueTask<bool>` |

Ковариантность `IEnumerable<out T>` — причина того, что это работает:

```csharp
IEnumerable<string> strings = new[] { "a", "b" };
IEnumerable<object> objects = strings;      // OK: ковариантность
Console.WriteLine(objects.Count());
// Вывод: 2
```

Мутабельный `IList<T>` так делать не даёт — и правильно. Механику см. в [[Дженерики: ограничения и вариантность]].

### Почему `Reset()` никто не реализует

`Reset()` попал в `IEnumerator` в .NET 1.0 ради COM-совместимости (`IEnumVARIANT::Reset`). На практике:

- Сгенерированные компилятором итераторы бросают `NotSupportedException`.
- LINQ-операторы его не поддерживают.
- Даже документация BCL прямо говорит: реализация не обязательна.

```csharp
IEnumerator<int> e = Countdown(3).GetEnumerator();
e.MoveNext();
e.Reset();   // System.NotSupportedException
```

Правильный способ «начать заново» — взять новый энумератор: `GetEnumerator()` ещё раз. Никогда не пишите код, который зависит от `Reset()`. #устарело

### Негенерик-версия и boxing

```csharp
IEnumerable list = new List<int> { 1, 2, 3 };
var sum = 0;
foreach (int x in list)     // компилятор вставляет unbox: (int)e.Current
    sum += x;
```

Каждый `Current` здесь — это `object`, то есть `int` уже упакован в куче. Три элемента — три аллокации. Именно поэтому негенерик-коллекции (`ArrayList`, `Hashtable`) не используют в новом коде. См. [[Boxing и unboxing]].

### Асинхронный вариант

`IAsyncEnumerable<T>` — тот же принцип, но `MoveNextAsync()` возвращает `ValueTask<bool>`, а итератор пишется как `async IAsyncEnumerable<T>` + `await foreach`. Всё, что ниже про конечный автомат, справедливо и там (только автомат объединён с автоматом `async`). Подробно — [[IAsyncEnumerable и асинхронные потоки]].

## yield return: во что разворачивается итератор

Это главный раздел заметки. Почти все «странности» итераторов объясняются одной картинкой.

### Исходный метод

```csharp
public static class Demo
{
    public static IEnumerable<int> CountTo(int n)
    {
        for (var i = 0; i < n; i++)
            yield return i;
    }
}
```

### Что генерирует компилятор

Roslyn заменяет тело метода на создание объекта и генерирует вложенный класс с «непроизносимым» именем (`<CountTo>d__0` — такой идентификатор нельзя написать в C#, поэтому конфликта имён не бывает):

```csharp
public static class Demo
{
    // Тело метода целиком заменено: НИКАКОЙ логики, только создание автомата
    public static IEnumerable<int> CountTo(int n)
    {
        var machine = new <CountTo>d__0(-2);
        machine.<>3__n = n;
        return machine;
    }

    [CompilerGenerated]
    private sealed class <CountTo>d__0
        : IEnumerable<int>, IEnumerable, IEnumerator<int>, IEnumerator, IDisposable
    {
        private int <>1__state;             // номер состояния автомата
        private int <>2__current;           // значение для Current
        private int <>l__initialThreadId;   // поток, создавший объект
        private int n;                      // параметр — рабочая копия
        public int <>3__n;                  // параметр — эталон для копий
        private int <i>5__1;                // поднятая локальная переменная i

        public <CountTo>d__0(int state)
        {
            <>1__state = state;
            <>l__initialThreadId = Environment.CurrentManagedThreadId;
        }

        private bool MoveNext()
        {
            switch (<>1__state)
            {
                default:                 // -1 (завершён) и -2 (не запущен/Dispose)
                    return false;

                case 0:                  // первый вход: выполняем начало тела
                    <>1__state = -1;     // «выполняюсь прямо сейчас»
                    <i>5__1 = 0;         // var i = 0
                    break;

                case 1:                  // возобновление после yield return
                    <>1__state = -1;
                    <i>5__1++;           // i++ из шага цикла
                    break;
            }

            if (<i>5__1 < n)             // условие цикла
            {
                <>2__current = <i>5__1;  // yield return i
                <>1__state = 1;          // «приостановлен на yield №1»
                return true;             // выход из MoveNext
            }
            return false;                // цикл кончился — последовательность исчерпана
        }

        int IEnumerator<int>.Current => <>2__current;
        object IEnumerator.Current => <>2__current;          // boxing
        void IEnumerator.Reset() => throw new NotSupportedException();
        void IDisposable.Dispose() { }                        // нет try/finally — пусто

        IEnumerator<int> IEnumerable<int>.GetEnumerator()
        {
            <CountTo>d__0 result;
            if (<>1__state == -2 && <>l__initialThreadId == Environment.CurrentManagedThreadId)
            {
                <>1__state = 0;
                result = this;            // первый раз из «своего» потока — переиспользуем себя
            }
            else
            {
                result = new <CountTo>d__0(0);   // иначе — свежая копия
            }
            result.n = <>3__n;            // копируем параметр из эталона
            return result;
        }

        IEnumerator IEnumerable.GetEnumerator() => ((IEnumerable<int>)this).GetEnumerator();
    }
}
```

### Таблица состояний

```
 <>1__state │ смысл
────────────┼──────────────────────────────────────────────────────────────────
     -2     │ объект создан итератор-методом, перечисление ещё НЕ начиналось
            │ (это форма «IEnumerable»). Сюда же Dispose() переводит автомат,
            │ если в итераторе есть try/finally: «мёртв, повторно не запустить»
     -1     │ MoveNext() прямо сейчас исполняет тело метода;
            │ он же — терминальное состояние после нормального завершения
     -3,-4… │ исполняется внутри try/finally-региона (по одному значению
            │ на регион) — нужно, чтобы Dispose() знал, какие finally догнать
      0     │ готов к работе: GetEnumerator() выдал энумератор,
            │ но ни одного MoveNext() ещё не было
    1..n    │ приостановлен на yield return №n; следующий MoveNext()
            │ через switch прыгнет ровно за этот yield
```

> [!tip]
> Хотите увидеть это своими глазами — вставьте итератор в sharplab.io и переключите вывод в «C#» (Roslyn декомпилирует lowering) или в IL. Читать сгенерированный код полезнее, чем читать про него.

### Почему тело не выполняется до первого MoveNext

Посмотрите на `CountTo` после lowering: в нём **нет ни одной строки** из вашего тела. Только `new` и присваивание поля. Значит:

```csharp
public static IEnumerable<int> Boom(int n)
{
    Console.WriteLine("вход в метод");
    if (n < 0) throw new ArgumentOutOfRangeException(nameof(n));
    for (var i = 0; i < n; i++) yield return i;
}

var seq = Boom(-5);            // ничего не напечатано, исключения нет
Console.WriteLine("метод вызван");
foreach (var x in seq) { }     // вот ЗДЕСЬ печатается «вход в метод» и летит исключение

// Вывод:
// метод вызван
// вход в метод
// Unhandled exception. System.ArgumentOutOfRangeException
```

Это и есть ленивость (lazy evaluation). Ни магии, ни планировщика — просто ваш код физически переехал в `MoveNext()`.

### Один класс — и IEnumerable, и IEnumerator

Зачем компилятору так извращаться? Экономия аллокации в самом частом случае: `foreach (var x in CountTo(10))` создаёт **один** объект, а не два (фабрику + курсор).

Логика `GetEnumerator()`:

| Условие | Результат |
|---|---|
| `state == -2` и вызов из того же потока, где создан | `state = 0`, вернуть `this` |
| `state != -2` (уже перечисляли) | новый объект в состоянии 0 |
| Другой поток | новый объект в состоянии 0 |

Следствия:
1. **Повторное перечисление одного и того же `IEnumerable` работает и начинается с начала.** Второй `foreach` попадёт в ветку «создать копию», получит свежий автомат с `n`, скопированным из `<>3__n`, и всё тело выполнится заново. Именно поэтому двойной `foreach` по LINQ-запросу дважды ходит в базу.
2. **Два одновременных перечисления не мешают друг другу** — у каждого свой автомат со своим `i`.
3. Проверка потока нужна, чтобы два потока не получили один и тот же `this` как курсор. Но это **не** делает итератор потокобезопасным: два `MoveNext()` из разных потоков на *одном энумераторе* по-прежнему ломают состояние.

```csharp
var seq = Demo.CountTo(3);
Console.WriteLine(string.Join(",", seq));   // 0,1,2
Console.WriteLine(string.Join(",", seq));   // 0,1,2 — заново, а не пусто
```

### Локальные переменные становятся полями

Любая локальная переменная, чьё значение должно пережить `yield return`, поднимается (hoisting) в поле автомата с именем вида `<i>5__1`. Это объясняет:

- почему у итераторов нет «потери состояния» между вызовами `MoveNext`;
- почему в итераторе нельзя иметь `ref`-локальные переменные, пересекающие `yield` (поле не может быть `ref`);
- почему в итераторе с большим числом локальных переменных объект автомата может быть неожиданно жирным (все живут одновременно, даже если в исходнике они в разных областях видимости).

Локальные переменные, которые **не** пересекают `yield`, остаются настоящими локальными переменными в `MoveNext` — Roslyn это оптимизирует.

### Dispose() и try/finally внутри итератора

Если в итераторе есть `try/finally` (в том числе `using`), компилятор выносит тело `finally` в отдельный метод `<>m__Finally1()` и вызывает его из `Dispose()`:

```csharp
public static IEnumerable<string> ReadLines(string path)
{
    using var reader = new StreamReader(path);      // раскроется в try/finally
    string? line;
    while ((line = reader.ReadLine()) is not null)
        yield return line;
}
```

Схематично сгенерированный `Dispose`:

```csharp
void IDisposable.Dispose()
{
    switch (<>1__state)
    {
        case -3:      // выполняемся внутри try
        case 1:       // приостановлены на yield ВНУТРИ try
            try { }
            finally
            {
                <>1__state = -2;
                <>m__Finally1();     // reader?.Dispose()
            }
            break;
    }
}
```

Теперь смотрим, что происходит при досрочном выходе:

```csharp
foreach (var line in ReadLines("big.log"))
{
    if (line.Contains("ERROR"))
        break;              // выходим на 5-й строке из миллиона
}
// break → выход из while → finally блока foreach → e.Dispose()
//       → <>m__Finally1() → reader.Dispose() → файл закрыт
```

**Это ключевая причина, по которой `foreach` оборачивает цикл в `try/finally`.** Без этого любой `break` по итератору с `using` оставлял бы открытый файл до сборки мусора и финализатора. Если вы перечисляете вручную — обязаны сами вызвать `Dispose()`, иначе `finally` в итераторе **никогда не выполнится**:

```csharp
var e = ReadLines("big.log").GetEnumerator();
e.MoveNext();
// забыли e.Dispose() → StreamReader жив, файл заблокирован
```

> [!danger] Почему `yield return` запрещён в `try` с `catch`
> Ошибка компилятора **CS1626**: *Cannot yield a value in the body of a try block with a catch clause*.
> Причина механическая. `yield return` — это `return` из `MoveNext()`. Управление уходит вызывающему коду, а исключение, которое там случится, к вашему `try` уже никакого отношения не имеет: кадр стека `MoveNext` давно снят. Обработчик `catch` физически не может «поймать» исключение, брошенное между двумя `MoveNext`.
> С `finally` компилятор смог схитрить: он вырезал `finally` в отдельный метод и повесил его на `Dispose()`, потому что момент «перечисление кончилось» определён однозначно. Для `catch` такой хитрости нет — ловить нечего и негде.
> Практический обход: разделите на два метода — внешний с `try/catch` вокруг `foreach` по внутреннему итератору.

## Ограничения итератор-методов

| Ограничение | Диагностика | Актуальность |
|---|---|---|
| `ref`/`in`/`out` параметры запрещены | CS1623 | всегда |
| Нельзя `return значение` в итераторе | CS1622 | всегда |
| Тип возврата — только `IEnumerable`, `IEnumerable<T>`, `IEnumerator`, `IEnumerator<T>` (для `async` — `IAsyncEnumerable<T>`/`IAsyncEnumerator<T>`) | CS1624 | всегда |
| `yield` в `finally` | CS1625 | всегда |
| `yield return` в `try` с `catch` | CS1626 | всегда |
| `yield` в `catch` | CS1631 | всегда |
| Параметры и yield-тип не могут быть unsafe (указатели) | CS1637 | всегда |
| Нельзя возвращать по ссылке (`ref T`) | CS8154 | всегда |
| `yield return` внутри `unsafe`-блока | CS9238 | всегда |
| Оператор `&` (взять адрес) на параметрах/локальных | CS9239 | всегда |
| `unsafe`-код в итераторе целиком | CS1629 | **только до C# 13**; с C# 13 разрешён, если сам `yield return` в safe-контексте |
| `ref`-локальные и `ref struct`-локальные | CS8176 / CS4013 | **только до C# 13**; с C# 13 можно, если они не живут через `yield return` |

Про `lock` — отдельная тонкость:

```csharp
private readonly object _gate = new();
private readonly System.Threading.Lock _newGate = new();

public IEnumerable<int> Old()
{
    lock (_gate)          // компилируется! Monitor.Enter/Exit + try/finally
        yield return 1;   // но лок держится между MoveNext — почти всегда баг
}

public IEnumerable<int> New()
{
    lock (_newGate)       // ОШИБКА компиляции
        yield return 1;   // lock над System.Threading.Lock == using над ref struct
}
```

Классический `lock (object)` с `yield return` компилятор пропускает, хотя это ловушка: монитор захвачен, пока потребитель делает что угодно между итерациями, и освобождается только на `Dispose()`. `lock` над `System.Threading.Lock` (C# 13+) разворачивается в `using (gate.EnterScope())`, а `EnterScope()` возвращает `ref struct` — `yield return` внутри такого `using` запрещён языком, и компилятор ошибку выдаст. Правильный подход: снимать снапшот под локом и перечислять уже его. См. [[Примитивы синхронизации: lock, Monitor]].

## Отложенная валидация аргументов и как её лечить

Классический баг, который в проде вылезает через месяц:

```csharp
// ПЛОХО
public static IEnumerable<T> TakeEvery<T>(this IEnumerable<T> source, int step)
{
    ArgumentNullException.ThrowIfNull(source);
    ArgumentOutOfRangeException.ThrowIfNegativeOrZero(step);

    var i = 0;
    foreach (var item in source)
        if (i++ % step == 0) yield return item;
}

var q = ((IEnumerable<int>)null!).TakeEvery(2);   // исключения НЕТ
// ...двадцать строк построения конвейера...
var list = q.ToList();                            // ArgumentNullException здесь
```

Стектрейс укажет на `ToList()`, а не на строку с ошибкой. Отладка превращается в квест.

Лечение — паттерн «публичная обёртка + приватный итератор». Обёртка **не** содержит `yield`, значит это обычный метод: проверки исполняются немедленно, а итератор создаётся уже отдельным вызовом.

```csharp
// ХОРОШО
public static IEnumerable<T> TakeEvery<T>(this IEnumerable<T> source, int step)
{
    ArgumentNullException.ThrowIfNull(source);
    ArgumentOutOfRangeException.ThrowIfNegativeOrZero(step);
    return Iterate(source, step);          // обычный return — метод не итератор

    static IEnumerable<T> Iterate(IEnumerable<T> src, int st)
    {
        var i = 0;
        foreach (var item in src)
            if (i++ % st == 0) yield return item;
    }
}

((IEnumerable<int>)null!).TakeEvery(2);   // ArgumentNullException сразу, на этой строке
```

Локальная функция здесь удобнее приватного метода: не засоряет тип и видна только там, где нужна. Именно так устроен весь LINQ в BCL (`Enumerable.Where` → `WhereIterator`). Подробнее в [[Свои операторы LINQ]] и [[Защитное программирование и guard clauses]].

## Struct-энумераторы и аллокации

`List<T>.Enumerator` — это **структура**, а не класс. Поэтому:

```csharp
var list = new List<int> { 1, 2, 3 };

// Вариант A: статический тип — List<int>
foreach (var x in list) { }
// GetEnumerator() возвращает List<int>.Enumerator (struct) → 0 аллокаций,
// MoveNext/Current вызываются напрямую и инлайнятся JIT-ом

// Вариант B: тот же список, но через интерфейс
IEnumerable<int> seq = list;
foreach (var x in seq) { }
// IEnumerable<int>.GetEnumerator() реализован явно и возвращает IEnumerator<int>
// → структура боксится → 1 аллокация в куче + все вызовы виртуальные (callvirt)
```

Разница видна на бенчмарках: вариант B в горячем цикле — это аллокация на каждый вызов и невозможность инлайна. Приём BCL, который стоит скопировать в свои коллекции:

```csharp
public struct Enumerator { /* ... */ }                       // публичная структура

public Enumerator GetEnumerator() => new(this);              // foreach найдёт ЭТОТ метод
IEnumerator<T> IEnumerable<T>.GetEnumerator() => GetEnumerator();   // здесь boxing, но
IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();         // только при работе через интерфейс
```

Явная реализация интерфейсов (см. [[Явная реализация интерфейсов]]) убирает интерфейсные версии из публичного API типа, и перегрузка `foreach` всегда выбирает структуру.

> [!warning] `List<T>.Enumerator` нельзя копировать
> Это **мутабельная структура**: её поля `_index`, `_current`, `_version` меняются в `MoveNext()`. Копирование по значению даёт независимый курсор:
> ```csharp
> var list = new List<int> { 1, 2, 3 };
> var e = list.GetEnumerator();
> var copy = e;              // копия структуры со своей позицией
> e.MoveNext();              // e.Current == 1
> Console.WriteLine(copy.MoveNext() ? copy.Current : -1);
> // Вывод: 1  — копия начала заново, а не продолжила
> ```
> Отсюда правила: не передавайте struct-энумератор в метод по значению (передавайте `ref`), не кладите его в поле типа `IEnumerator<T>` (заboxится и превратится в общий изменяемый объект), не помечайте `readonly` переменную с энумератором — иначе каждый вызов `MoveNext` будет работать с defensive copy и цикл станет бесконечным.

## IEnumerable<T> как контракт API

`IEnumerable<T>` — самый слабый и самый честный контракт: «я выдам элементы по одному, когда попросишь». Именно слабость делает его и полезным, и опасным.

| Возвращать `IEnumerable<T>` | Возвращать `IReadOnlyList<T>` / `T[]` |
|---|---|
| Стриминг: миллион строк из файла, курсор БД, SSE-поток | Данные уже в памяти, их немного |
| Потребитель может остановиться на середине (`Take`, `First`) | Потребителю нужен `Count`, индекс, повторные проходы |
| Ленивая композиция конвейера (LINQ) | Результат зафиксирован, побочных эффектов больше не будет |
| Потребление ровно один раз | Возможно многократное перечисление |

Чем `IEnumerable<T>` из публичного метода стреляет в ногу:

```csharp
public IEnumerable<Order> GetPendingOrders() =>
    _db.Orders.Where(o => o.Status == Status.Pending);   // IQueryable, но тип метода — IEnumerable

var orders = service.GetPendingOrders();
if (orders.Any())                        // запрос №1 к БД
    Log(orders.Count());                 // запрос №2
foreach (var o in orders) Process(o);    // запрос №3
```

Три обращения к базе там, где программист видел «одну переменную со списком». Плюс: между запросами данные могли измениться, и `Any() == true` не гарантирует, что `foreach` что-то выдаст. Про это же — [[IEnumerable vs IQueryable]] и [[EF Core: запросы и загрузка связанных данных]].

Правило, которое стоит держать в голове: **если вы уже материализовали данные — верните материализованный тип.** `IEnumerable<T>` возвращают, когда лень — часть контракта, а не побочный эффект реализации.

> [!warning] Подводные камни
> **1. Аргументы валидируются при первом `MoveNext()`, а не при вызове.** Причина: тело итератор-метода целиком переехало в `MoveNext()` сгенерированного класса, в самом методе остался только `new`. Стектрейс укажет на `ToList()`/`foreach`, а не на виновника. Лечится обёрткой + приватным итератором.
>
> **2. Повторное перечисление незаметно повторяет всю работу.** `GetEnumerator()` при втором вызове создаёт новый автомат, тело выполняется заново: снова читается файл, снова уходит SQL-запрос, снова вызывается HTTP. Особенно больно, когда в конвейере есть побочные эффекты или счётчики. Лечение: `ToList()` там, где данные нужны больше одного раза (см. [[Отложенное и немедленное выполнение]]).
>
> **3. `finally` в итераторе выполняется только через `Dispose()`.** Если вы перечисляете вручную (`GetEnumerator()` + `MoveNext()`) и не вызвали `Dispose()`, `using` внутри итератора не сработает: файл останется открытым, соединение — незакрытым. `foreach` вызывает `Dispose()` за вас, ручной цикл — нет. Всегда `using var e = ...`.
>
> **4. Боксинг энумератора при работе через интерфейс.** `foreach (var x in list)` — ноль аллокаций, `foreach (var x in (IEnumerable<int>)list)` — аллокация на каждый цикл, потому что `List<T>.Enumerator` — структура, и явная реализация `IEnumerable<T>.GetEnumerator()` её упаковывает. В горячем пути принимайте конкретный тип или `ReadOnlySpan<T>` вместо `IEnumerable<T>`.
>
> **5. Захват изменяемого состояния в итераторе.** Параметры копируются в поля автомата один раз (в момент вызова метода), а вот поля объекта-владельца читаются лениво, во время `MoveNext`. Итератор, читающий `this._items`, увидит то состояние, которое будет на момент перечисления, а не на момент вызова.
>
> **6. Модификация коллекции во время перечисления.** `List<T>` бросит `InvalidOperationException: Collection was modified` — это делает поле `_version`, проверяемое в `MoveNext`. Ваши собственные коллекции по умолчанию так не делают, и вместо честной ошибки вы получите тихо пропущенные или продублированные элементы.

> [!example] Как делают в бою
> **Стриминг больших файлов.** Разбор гигабайтного CSV делают итератором: `IEnumerable<Row> Parse(string path)` с `using var reader` внутри. Память — константная, `Take(100)` в тестах не читает весь файл, `break` закрывает файл детерминированно.
>
> **Пагинация чужого API.** Итератор скрывает continuation token: снаружи это просто `IEnumerable<Item>` (или, честнее, `IAsyncEnumerable<Item>`), внутри — цикл «запросил страницу, отдал элементы, взял следующий токен». Потребитель, которому нужны первые 10 элементов, дёрнет ровно одну страницу.
>
> **Не отдавайте `IEnumerable<T>` из репозитория.** Общее правило команд: методы уровня данных возвращают `Task<List<T>>` или `IReadOnlyList<T>` — тогда невозможно случайно утащить ленивый `IQueryable` за границу scope `DbContext` и получить `ObjectDisposedException` в контроллере.
>
> **`IAsyncEnumerable<T>` для IO.** Как только внутри итератора появляется `await` (БД, HTTP, файл), синхронный `IEnumerable<T>` становится вредным: он блокирует поток. Переход на `async IAsyncEnumerable<T>` + `await foreach` — стандартная практика в ASP.NET Core 10 (в том числе для `TypedResults.ServerSentEvents`).
>
> **Диагностика в тестах.** Отложенность проверяют источником-счётчиком: итератор, инкрементирующий счётчик на каждом `MoveNext`. Тест утверждает, что после построения конвейера счётчик равен нулю, а после `Take(3).ToList()` — трём.

## Вопросы с собеседований

> [!question]- Почему тело итератор-метода не выполняется в момент вызова?
> Потому что после компиляции тела в методе не осталось. Roslyn генерирует вложенный класс — конечный автомат, переносит весь код в его метод `MoveNext()`, а в исходном методе оставляет только создание объекта этого класса и копирование параметров в его поля. Вызов `CountTo(10)` — это буквально `new <CountTo>d__0(-2) { <>3__n = 10 }`. Первая строка вашего кода исполнится, когда потребитель вызовет `MoveNext()`, то есть при первом заходе `foreach`. Отсюда и ленивость, и то, что проверки аргументов «опаздывают».

> [!question]- Что произойдёт, если перечислить один и тот же `IEnumerable<T>`, полученный из итератор-метода, дважды?
> Всё выполнится заново, с начала. `GetEnumerator()` первый раз (состояние `-2`, тот же поток) возвращает `this`, переведя состояние в `0`. Второй вызов видит состояние не равное `-2` и создаёт новый объект автомата, скопировав в него параметры из полей-эталонов (`<>3__n`). Поэтому двойной `foreach` по LINQ-запросу к EF Core даёт два SQL-запроса, а двойной проход по итератору чтения файла дважды читает файл. Если данные нужны больше одного раза — материализуйте: `ToList()`.

> [!question]- Обязательно ли реализовывать `IEnumerable`, чтобы работал `foreach`?
> Нет. `foreach` разрешается по форме (duck typing): нужен доступный `GetEnumerator()` без параметров, у результата — `bool MoveNext()` и свойство `Current` с геттером. С C# 9 `GetEnumerator` может быть даже расширяющим методом. Ровно на этом работает `foreach` по `Span<T>`: `ref struct` не может реализовать `IEnumerable<T>` и не может быть упакован, но `foreach` по нему компилируется без интерфейсов, без boxing и без аллокаций. Интерфейсы нужны, чтобы работал LINQ и чтобы тип можно было передать как `IEnumerable<T>`, — но не для `foreach`.

> [!question]- Почему `foreach` по `List<T>` не аллоцирует, а по `IEnumerable<T>` — аллоцирует?
> `List<T>.GetEnumerator()` возвращает `List<T>.Enumerator` — структуру. `foreach` по переменной статического типа `List<T>` выбирает этот публичный метод, структура живёт на стеке, вызовы `MoveNext`/`Current` невиртуальные и инлайнятся. Если статический тип переменной — `IEnumerable<T>`, компилятор обязан вызвать явную реализацию `IEnumerable<T>.GetEnumerator()`, которая возвращает `IEnumerator<T>`; структура при этом упаковывается в куче (одна аллокация на цикл), а все обращения идут через `callvirt`. Тот же список, разный статический тип — разная стоимость.

> [!question]- Что делает `Dispose()` у сгенерированного итератора и когда он вызывается?
> Если в итераторе есть `try/finally` (в том числе `using`), Roslyn выносит код `finally` в отдельный метод `<>m__Finally1()` и генерирует `Dispose()`, который по текущему состоянию решает, какие `finally` нужно догнать, вызывает их и переводит состояние в `-2`. Если `try/finally` нет — `Dispose()` пустой. Вызывает его `foreach`: он оборачивает цикл в `try/finally { e.Dispose(); }`, поэтому `break`, `return` из середины цикла и исключение одинаково приводят к освобождению ресурсов. При ручном перечислении `Dispose()` — ваша обязанность, иначе `finally` внутри итератора не выполнится никогда.

> [!question]- Почему `yield return` нельзя писать внутри `try` с `catch`?
> Ошибка CS1626. `yield return` компилируется в `return` из `MoveNext()` — кадр стека снимается, управление уходит потребителю. Исключение, случившееся в теле `foreach` между двумя `MoveNext`, происходит в совершенно другом кадре, и `catch` внутри итератора его поймать физически не может: защищённый регион в этот момент не активен. Для `finally` компилятор нашёл обходной путь — вырезал его в отдельный метод и повесил на `Dispose()`, так как момент «перечисление закончилось» определён однозначно. Для `catch` аналога нет. `yield` в самом `catch` и в `finally` тоже запрещён (CS1631, CS1625).

> [!question]- Почему `IEnumerator.Reset()` практически никогда не реализуют?
> Он попал в интерфейс в .NET 1.0 ради совместимости с COM-энумераторами (`IEnumVARIANT::Reset`) и практической ценности не имеет: чтобы начать заново, достаточно взять новый энумератор через `GetEnumerator()`. Сгенерированные компилятором итераторы бросают `NotSupportedException`, LINQ-операторы — тоже, документация BCL прямо разрешает не реализовывать. Считайте его историческим артефактом и никогда не пишите логику, которая на него опирается.

> [!question]- Чем `IEnumerable<T>` отличается от `IReadOnlyCollection<T>` как возвращаемый тип, и что выбрать?
> `IEnumerable<T>` обещает только «элементы можно перебрать», без `Count`, без индексации, без гарантии, что повторный перебор даст то же самое или вообще будет дешёвым. `IReadOnlyCollection<T>` добавляет `Count` и тем самым неявно обещает, что данные уже есть в памяти. Выбор простой: если внутри метода вы вызвали `ToList()`, возвращайте `IReadOnlyList<T>` — иначе вы скрываете от вызывающего важный факт и провоцируете его на `Count()`/`Any()` поверх лени. `IEnumerable<T>` уместен, когда лень принципиальна: стриминг больших данных, бесконечные последовательности, конвейеры LINQ.

## Задачи

### Задача 1. Interleave с честной валидацией

Напишите оператор `Interleave<T>(this IEnumerable<T> first, IEnumerable<T> second)`, который выдаёт элементы поочерёдно (`a1, b1, a2, b2, ...`), а когда одна последовательность кончилась — доливает остаток другой. Требования: `null`-аргументы должны бросать исключение **в момент вызова**, источники перечисляются ровно один раз и лениво.

> [!success]- Решение
> ```csharp
> public static class SequenceExtensions
> {
>     public static IEnumerable<T> Interleave<T>(this IEnumerable<T> first, IEnumerable<T> second)
>     {
>         ArgumentNullException.ThrowIfNull(first);
>         ArgumentNullException.ThrowIfNull(second);
>         return Iterate(first, second);
>
>         static IEnumerable<T> Iterate(IEnumerable<T> a, IEnumerable<T> b)
>         {
>             // using обязателен: иначе энумераторы источников не будут освобождены
>             using var ea = a.GetEnumerator();
>             using var eb = b.GetEnumerator();
>
>             bool hasA = ea.MoveNext(), hasB = eb.MoveNext();
>             while (hasA || hasB)
>             {
>                 if (hasA) { yield return ea.Current; hasA = ea.MoveNext(); }
>                 if (hasB) { yield return eb.Current; hasB = eb.MoveNext(); }
>             }
>         }
>     }
> }
>
> var r = new[] { 1, 3, 5, 7 }.Interleave(new[] { 2, 4 });
> Console.WriteLine(string.Join(",", r));
> // Вывод: 1,2,3,4,5,7
> ```
> Ключевые моменты. Публичный метод не содержит `yield`, значит это обычный метод и проверки срабатывают немедленно. Внутри мы работаем с энумераторами вручную, потому что двумя `foreach` поочерёдный обход не выразить. `using var` на энумераторах критичен: `yield return` внутри развернётся в `try/finally`, и при `break` у потребителя `Dispose()` итератора освободит оба источника.

### Задача 2. Докажите, что итератор ленив

Напишите тестовый источник, который считает, сколько раз у него дёрнули `MoveNext()`, и с его помощью покажите: (а) построение конвейера `Where(...).Select(...)` не читает источник вообще; (б) `Take(2).ToList()` читает ровно столько элементов, сколько нужно.

> [!success]- Решение
> ```csharp
> public sealed class CountingSource : IEnumerable<int>
> {
>     private readonly int _upTo;
>     public int MoveNextCalls { get; private set; }
>     public CountingSource(int upTo) => _upTo = upTo;
>
>     public IEnumerator<int> GetEnumerator() => Iterate();
>     IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
>
>     private IEnumerator<int> Iterate()
>     {
>         for (var i = 0; i < _upTo; i++)
>         {
>             MoveNextCalls++;      // считаем ФАКТИЧЕСКИЕ шаги источника
>             yield return i;
>         }
>     }
> }
>
> var src = new CountingSource(1_000_000);
>
> var pipeline = src.Where(x => x % 2 == 0).Select(x => x * 10);
> Console.WriteLine(src.MoveNextCalls);          // Вывод: 0 — источник не тронут
>
> var firstTwo = pipeline.Take(2).ToList();
> Console.WriteLine(string.Join(",", firstTwo)); // Вывод: 0,20
> Console.WriteLine(src.MoveNextCalls);          // Вывод: 3 — прочитано 0,1,2 и всё
> ```
> Почему именно 3: `Take(2)` останавливает конвейер, как только получил второй элемент. Чтобы `Where` выдал второй чётный элемент (`2`), источнику пришлось отдать `0`, `1`, `2` — три шага, а не миллион. Это и есть практическая польза лени: `Take`/`First`/`Any` не тянут всё. Обратите внимание, что счётчик живёт в поле объекта-владельца, а не в поле автомата, поэтому его видно снаружи.

### Задача 3. Найдите утечку

Этот код в проде постепенно съедает файловые дескрипторы, пока сервис не падает с `IOException: Too many open files`. Найдите причину и исправьте.

```csharp
public static IEnumerable<string> ReadLines(string path)
{
    using var reader = new StreamReader(path);
    string? line;
    while ((line = reader.ReadLine()) is not null)
        yield return line;
}

public static string? FindFirstError(string path)
{
    var e = ReadLines(path).GetEnumerator();
    while (e.MoveNext())
        if (e.Current.Contains("ERROR"))
            return e.Current;        // <-- выходим, не тронув e
    return null;
}
```

> [!success]- Решение
> ```csharp
> public static string? FindFirstError(string path)
> {
>     // Вариант 1: foreach сам обернёт цикл в try/finally { e.Dispose(); }
>     foreach (var line in ReadLines(path))
>         if (line.Contains("ERROR"))
>             return line;
>     return null;
> }
>
> public static string? FindFirstErrorManual(string path)
> {
>     // Вариант 2: если нужен ручной цикл — using на энумераторе
>     using var e = ReadLines(path).GetEnumerator();
>     while (e.MoveNext())
>         if (e.Current.Contains("ERROR"))
>             return e.Current;
>     return null;
> }
>
> // Вариант 3: короче всего, и лень сохраняется
> public static string? FindFirstErrorLinq(string path) =>
>     ReadLines(path).FirstOrDefault(l => l.Contains("ERROR"));
> ```
> Причина утечки. `using var reader` внутри итератора компилируется в `try/finally`, а этот `finally` вызывается **только** из `Dispose()` автомата. `return` из середины ручного `while` не вызывает `Dispose()` — итератор навсегда остаётся в состоянии «приостановлен на yield внутри try», а `StreamReader` и его `FileStream` живут до сборки мусора и финализатора `SafeFileHandle`. При сотнях вызовов в секунду дескрипторы кончаются раньше, чем GC доберётся до объектов. Все три варианта исправления делают одно и то же — гарантируют вызов `Dispose()`. См. [[IDisposable, using и паттерн Dispose]].

## Итог

- `foreach` — сахар над `GetEnumerator()` + `while (MoveNext())` + `try/finally { Dispose() }`. Интерфейсы не обязательны: работает duck typing по форме, поэтому `foreach` по `Span<T>` не аллоцирует ничего.
- `yield return` заставляет Roslyn сгенерировать класс-конечный автомат: тело метода переезжает в `MoveNext()`, локальные переменные становятся полями, состояние хранится в `<>1__state`. Из этого механически следуют ленивость и отложенная валидация аргументов.
- Один сгенерированный класс играет и `IEnumerable<T>`, и `IEnumerator<T>`: первый `GetEnumerator()` из создавшего потока возвращает `this`, остальные — копию. Поэтому повторное перечисление работает и начинается заново, повторяя всю работу.
- `finally`/`using` внутри итератора выполняется через `Dispose()`. `foreach` вызывает его сам; ручной цикл без `using` — гарантированная утечка ресурсов.
- Валидируйте аргументы в публичной обёртке, а `yield` держите в приватном итераторе — иначе исключение прилетит не там, где ошибка.
- `IEnumerable<T>` возвращайте только тогда, когда лень — часть контракта. Материализовали данные — возвращайте `IReadOnlyList<T>`.

## Связанное

- [[Свои коллекции и итераторы]]
- [[Свои операторы LINQ]]
- [[Отложенное и немедленное выполнение]]
- [[IEnumerable vs IQueryable]]
- [[IAsyncEnumerable и асинхронные потоки]]
- [[Span, ReadOnlySpan и Memory]]
- [[Boxing и unboxing]]
- [[IDisposable, using и паттерн Dispose]]
