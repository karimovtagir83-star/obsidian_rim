---
tags: [раздел-03, основы, собес, ооп, модель, ревью, агент-ловушка]
aliases: [Equals GetHashCode, Object methods, Равенство объектов, HashCode.Combine, ToString, IEquatable, IComparable, IComparer, IEqualityComparer, Сравнение объектов, Сортировка]
---

# Object: Equals, GetHashCode, ToString

> [!abstract] Коротко
> От `System.Object` наследуются все типы, и три его виртуальных метода определяют поведение объекта в сравнениях, в хеш-таблицах и в логах. Сравнений на самом деле два разных вида: **равенство** (`Equals`/`GetHashCode`, `IEquatable<T>`) и **порядок** (`IComparable<T>`, `CompareTo`). Главное правило собеседований: переопределил `Equals` — обязан переопределить `GetHashCode`, иначе объект «есть в коллекции, но не находится».

## Зачем это знать

```csharp
public class Money { public decimal Amount { get; init; } public string Currency { get; init; } = ""; }

var a = new Money { Amount = 100, Currency = "UZS" };
var b = new Money { Amount = 100, Currency = "UZS" };

Console.WriteLine(a.Equals(b));                       // False — сравнение ссылок
Console.WriteLine(new HashSet<Money> { a, b }.Count);  // 2 — «дубликат» не распознан
Console.WriteLine(a);                                  // MyApp.Money — бесполезно в логе
```

Тип, который по смыслу является значением, ведёт себя как ссылка. Это ломает поиск в коллекциях, `Distinct()`, сравнение в тестах и сортировку — а на ревью такие проблемы почти всегда приходят в сгенерированном коде, где Equals переопределён наполовину.

## Модель

### Ссылочное vs значимое равенство по умолчанию

| Тип | `Equals` по умолчанию | `GetHashCode` по умолчанию |
|---|---|---|
| класс | сравнение ссылок | по идентичности объекта |
| структура (`ValueType`) | сравнение полей | по полям |

Для структуры «быстрый путь» — побитовое сравнение памяти — работает, только если нет ссылочных полей и нет «дырок» выравнивания; иначе поля обходятся **рефлексией**, это медленно и создаёт упаковку. Отсюда правило: структура, участвующая в сравнениях или используемая как ключ, обязана переопределять `Equals`/`GetHashCode` либо быть `record struct`.

### Контракт Equals — обязателен

1. **Рефлексивность:** `x.Equals(x)` — всегда `true`.
2. **Симметричность:** `x.Equals(y) == y.Equals(x)`.
3. **Транзитивность:** из `x.Equals(y)` и `y.Equals(z)` следует `x.Equals(z)`.
4. **Согласованность:** повторные вызовы дают тот же результат, пока объект не менялся.
5. **`x.Equals(null)` — всегда `false`**, без исключений.

Плюс контракт хеша: **равные объекты обязаны иметь одинаковый `GetHashCode`**. Обратное не требуется — коллизии допустимы, `Dictionary` разрешает их сравнением внутри корзины:

```
Dictionary.Add(key, value):
   1. hash = key.GetHashCode()
   2. bucket = hash % capacity     ← корзина выбирается ТОЛЬКО по хешу
   3. внутри корзины ключи сравниваются через Equals
```

Если `Equals` переопределён без `GetHashCode` (или наоборот), два «равных» объекта попадают в разные корзины — поиск не доходит до сравнения. Компилятор предупреждает CS0659, игнорировать нельзя. Дополнительные требования к хешу: **стабильность** (считать только по неизменяемым полям — иначе объект «теряется» после мутации, пока лежит в коллекции) и **хорошее распределение** (`GetHashCode() => 0` формально корректен, но превращает поиск в `O(n)`, см. [[Dictionary и HashSet изнутри]]). Хеш строк рандомизирован между запусками процесса — сохранять его вне процесса нельзя.

### Канонический вариант с IEquatable<T>

```csharp
public sealed class Money : IEquatable<Money>
{
    public required decimal Amount { get; init; }
    public required string Currency { get; init; }

    // Типизированная версия — без упаковки, её находит EqualityComparer<T>.Default
    public bool Equals(Money? other)
        => other is not null
           && (ReferenceEquals(this, other)
               || (Amount == other.Amount && string.Equals(Currency, other.Currency, StringComparison.Ordinal)));

    public override bool Equals(object? obj) => Equals(obj as Money);
    public override int GetHashCode() => HashCode.Combine(Amount, Currency);
    public override string ToString() => $"{Amount:0.##} {Currency}";
}
```

**Зачем `IEquatable<T>`, а не только `Equals(object)`:** `Dictionary`, `HashSet`, `List.Contains`, `Distinct` сравнивают через `EqualityComparer<T>.Default`, который сначала проверяет, реализует ли тип `IEquatable<T>`, и если да — вызывает типизированную версию напрямую. Без него `Equals(object)` **упаковывает оба операнда** при каждом сравнении структуры — аллокации в горячем пути. Для класса выигрыш скромнее (исчезает приведение типа), но интерфейс всё равно не заменяет `Equals(object)`/`GetHashCode` — реализовать нужно всё три.

`record`/`readonly record struct` генерируют `Equals`, `GetHashCode`, `IEquatable<T>`, `==`/`!=` и `ToString` автоматически — для нового кода это основной путь, ручная реализация остаётся для нестандартного сравнения (валюта без учёта регистра) и для сущностей с равенством по идентификатору.

При наследовании симметричность легко сломать: если базовый класс сравнивает через `is`, `base.Equals(derived)` может быть `true`, а `derived.Equals(base)` — `false`. Решения — сравнивать `GetType() == obj.GetType()` вместо `is`, или делать типы `sealed`; `record` решает это сгенерированным `EqualityContract`.

### Порядок: IComparable<T>

Равенство и порядок — разные механизмы. `IComparable<T>.CompareTo` возвращает отрицательное число (меньше), ноль (равен по порядку) или положительное (больше):

```csharp
public int CompareTo(Money? other)
{
    if (other is null) return 1;                 // null меньше всего
    if (Currency != other.Currency)
        throw new InvalidOperationException($"Нельзя сравнивать {Currency} и {other.Currency}");
    return Amount.CompareTo(other.Amount);        // не вычитание — переполнение и NaN
}
```

Контракт: полный порядок (антисимметричность, транзитивность), `CompareTo(null)` не бросает исключение. Отдельное соглашение — **если `CompareTo` вернул 0, `Equals` должен вернуть `true`**: иначе `SortedSet` (сравнивает по компаратору) и `HashSet` (по `Equals`) по-разному считают дубликаты одной и той же коллекции элементов. `record` генерирует равенство, но не `IComparable<T>` — порядок дописывается руками.

### Внешние правила: IEqualityComparer<T> и IComparer<T>

Когда сравнений для типа нужно несколько (по артикулу — в одном месте, по артикулу и складу — в другом), лишний смысл не запихивают в `Equals`/`CompareTo` — его выносят наружу и передают параметром:

```csharp
public sealed class ByWarehouseComparer : IEqualityComparer<Product>
{
    public static readonly ByWarehouseComparer Instance = new();
    public bool Equals(Product? x, Product? y) => x?.WarehouseId == y?.WarehouseId && x?.Sku == y?.Sku;
    public int GetHashCode(Product o) => HashCode.Combine(o.Sku, o.WarehouseId);
}

var set = new HashSet<Product>(ByWarehouseComparer.Instance);
var grouped = products.GroupBy(p => p, ByWarehouseComparer.Instance);
orders.Sort((a, b) => b.Total.CompareTo(a.Total));                          // Comparison<T>
var sorted = orders.OrderBy(o => o.Status).ThenByDescending(o => o.CreatedAt).ThenBy(o => o.Id);
```

`IEqualityComparer<T>`/`IComparer<T>` принимаются конструкторами `Dictionary`, `HashSet`, `SortedSet`, `SortedDictionary` и методами LINQ `Distinct`, `GroupBy`, `Join`, `OrderBy`. Компараторы без состояния делают статическими синглтонами. В пагинации сортировку всегда доопределяют уникальным ключом последним критерием — иначе нестабильная (`List.Sort`, introsort) или даже стабильная (`OrderBy`, но по неуникальному полю) сортировка даёт разный порядок между запросами.

### ToString

Значение по умолчанию — полное имя типа, бесполезно. Переопределяют коротко и информативно (идентификатор + пара ключевых полей), без секретов и персональных данных (строка уедет в логи), без исключений и тяжёлых вычислений. У `record` `ToString` печатает **все** свойства автоматически — для типов с чувствительными полями его переопределяют явно. Для отладчика есть отдельный `[DebuggerDisplay(...)]`, не влияющий на логи ([[Отладка: точки останова, watch, immediate]]).

## Решения

| Ситуация | Что выбрать |
|---|---|
| Value object без нестандартного сравнения | `readonly record struct` — равенство и `ToString` бесплатно |
| Value object с нестандартным сравнением (без учёта регистра, округление) | ручной `Equals`/`GetHashCode` + `IEquatable<T>` |
| Тип используется как ключ `Dictionary`/`HashSet`, особенно структура | обязательно `IEquatable<T>` — иначе упаковка на каждое сравнение |
| Сущность в домене | равенство по `Id` и типу (`GetType()`), не по полям — [[DDD: тактические паттерны]] |
| Нужна сортировка, встроенная в тип | `IComparable<T>`, согласован с `Equals` (`CompareTo == 0` ⇒ `Equals == true`) |
| Сортировка нужна разово или по контексту | `OrderBy`/`ThenBy`, не трогать тип |
| Правило сравнения зависит от места использования | внешний `IEqualityComparer<T>`/`IComparer<T>`, тип не меняется |
| Просто нужны логи и отладка | переопределить только `ToString`, без Equals/GetHashCode |

> [!example] В ServiceShop
> Value objects (деньги, артикул, идентификаторы) в модулях обычно объявляют равенство явно — либо как `record`/`readonly record struct`, либо руками с `IEquatable<T>`, потому что они участвуют в сравнениях и попадают в коллекции и словари при агрегации заказов. Сущности домена (`Order`, `Customer`) сравнивают по идентификатору, а не по набору полей — переопределённый `Equals` по всем свойствам для сущности обычно ошибка ревью, а не осознанное решение. Если сомневаешься в деталях конкретного модуля — общий принцип: value object без переопределённого равенства, который кладут в `HashSet`/`Dictionary` или сравнивают в тестах, почти всегда сигнал недоделанного типа.

> [!warning] Где ошибаются
> - **Агент часто:** переопределяет `Equals`, забывая `GetHashCode` (или наоборот) — классическая рассинхронизация. Компилятор предупреждает CS0659, но предупреждение легко пропустить в большом дифф-патче.
> - **Агент часто:** хеш считает по изменяемому полю (`{ get; set; }`) — объект «теряется» в `HashSet`/`Dictionary` после мутации, хотя формально всё ещё лежит в коллекции.
> - Переопределяет `Equals`, не трогая операторы `==`/`!=` — в одном типе оказываются два разных смысла сравнения.
> - `CompareTo`, не согласованный с `Equals` (например, сравнивает по одному полю, а равенство — по другому) — `SortedSet` и `HashSet` расходятся по числу «уникальных» элементов.
> - Наследование ломает симметричность `Equals` через `is` вместо `GetType()`.
> - `ToString` у `record` печатает все свойства автоматически — персональные данные утекают в логи незаметно.

## Проверка на ревью

- Если переопределён `Equals`, рядом есть `GetHashCode`, и оба используют один и тот же набор полей.
- Поля, участвующие в `GetHashCode`, неизменяемы (`init`/`readonly`), а не `{ get; set; }`.
- Структура или тип-ключ реализует `IEquatable<T>`, а не полагается на `Equals(object)`.
- `CompareTo` не использует вычитание чисел (`a.Id - b.Id`) — только `CompareTo` на полях.
- Value object в проекте — не «голый» класс без равенства, а `record`/`readonly record struct` либо явная реализация.
- `ToString` типа с чувствительными данными не выводит их в незамаскированном виде.

## Проверь себя

> [!question]- Почему при переопределении `Equals` обязательно переопределять `GetHashCode`?
> Хеш-коллекции сначала вычисляют корзину по хешу и только внутри неё сравнивают через `Equals`. Если равные по `Equals` объекты имеют разные хеши, они попадают в разные корзины, и поиск не доходит до сравнения — элемент «есть в словаре, но не находится». Обратное необязательно: разные объекты могут иметь одинаковый хеш, коллизии разрешаются сравнением внутри корзины. Компилятор предупреждает об этом через CS0659.

> [!question]- Зачем нужен `IEquatable<T>`, если есть `override Equals(object)`?
> `EqualityComparer<T>.Default`, который используют `Dictionary`, `HashSet`, `Distinct`, проверяет наличие `IEquatable<T>` и вызывает типизированный метод напрямую. Без него `Equals(object)` для структуры упаковывает оба операнда на каждое сравнение — лишние аллокации в горячем коде. Интерфейс не заменяет `Equals(object)`/`GetHashCode`, а дополняет их.

> [!question]- Как правильно реализовать равенство при наследовании?
> Сравнение через `is` ломает симметричность: `base.Equals(derived)` может быть `true`, а `derived.Equals(base)` — `false`, и поведение коллекций начинает зависеть от порядка операндов. Решения — сравнивать `GetType() == obj.GetType()` вместо `is`, либо делать типы `sealed`. `record` решает это генерацией `EqualityContract`.

> [!question]- Что означает согласованность `CompareTo` и `Equals`, и что будет, если её нарушить?
> Соглашение: если `CompareTo` вернул 0, `Equals` должен вернуть `true`. `SortedSet` определяет дубликаты по компаратору (`CompareTo`), а `HashSet` — по `Equals`; если они рассогласованы, одна и та же коллекция объектов даёт разное число «уникальных» элементов в зависимости от типа коллекции — трудноуловимый баг, который не ловится юнит-тестом на одной структуре данных.

> [!question]- Когда сравнение выносят во внешний компаратор вместо переопределения Equals/CompareTo?
> Когда правил больше одного или правило зависит от контекста использования, а не от природы типа. Тип должен иметь одно естественное сравнение; альтернативы передаются параметром через `IEqualityComparer<T>`/`IComparer<T>` в конструкторы коллекций и в LINQ (`Distinct`, `GroupBy`, `OrderBy` с `IComparer`). Компараторы без состояния делают статическими синглтонами.

## Разбор

**Дан фрагмент — найди проблему:**

```csharp
public class Product
{
    public string Sku { get; set; } = "";
    public string Name { get; set; } = "";

    public override bool Equals(object? obj) => obj is Product p && p.Sku == Sku;
    public override int GetHashCode() => Sku.GetHashCode() + Name.GetHashCode();
}
```

> [!success]- Решение
> `GetHashCode` учитывает `Name`, которого нет в `Equals`, — равные по `Equals` объекты с разными именами получат разные хеши и «потеряются» в словаре. Оба поля изменяемые — хеш меняется после добавления в коллекцию. Сложение хешей даёт плохое распределение. Нет `IEquatable<Product>` — сравнение всегда идёт через упаковку. Починка: `Sku` — единственное поле в обоих методах, `init` вместо `set`, `IEquatable<Product>`, `Sku.GetHashCode(StringComparison.Ordinal)`.

**Дан PR агента — что вернёшь на доработку:**

```csharp
public class Task : IComparable<Task>
{
    public int Priority { get; set; }
    public string Title { get; set; } = "";
    public int CompareTo(Task? other) => Priority - other!.Priority;
    public override bool Equals(object? obj) => obj is Task t && t.Title == Title;
    public override int GetHashCode() => Title.GetHashCode();
}
```

> [!success]- Решение
> Три замечания в ревью: (1) `Priority - other.Priority` переполняется на больших значениях и даёт неверный знак — нужен `Priority.CompareTo(other.Priority)`; (2) `CompareTo` сравнивает по `Priority`, а `Equals` — по `Title`, они рассогласованы: в `SortedSet` задачи с одинаковым приоритетом схлопнутся в одну, хотя `Equals` считает их разными; (3) свойства изменяемые — объект в `SortedSet`/`HashSet` «потеряется» при правке `Priority` или `Title` после вставки. Возврат на доработку: сделать `Priority`/`Title` `init`, согласовать `CompareTo` с `Equals` (оба смотрят на `Title` как завершающий критерий после `Priority`), убрать `other!` в пользу явной проверки на `null`.

## Итог

- Равенство (`Equals`/`GetHashCode`/`IEquatable<T>`) и порядок (`CompareTo`/`IComparable<T>`) — разные механизмы с разными контрактами.
- Переопределил `Equals` — обязан переопределить `GetHashCode`, и оба должны смотреть на одни и те же неизменяемые поля.
- `IEquatable<T>` избавляет от упаковки при сравнении структур; для класса выигрыш скромнее, но реализовать всё равно нужно все три члена.
- `CompareTo == 0` обязан означать `Equals == true` — иначе `SortedSet` и `HashSet` расходятся по числу дубликатов.
- `record`/`readonly record struct` закрывают равенство генерацией, но не порядок — `IComparable<T>` дописывается руками.
- Альтернативные правила сравнения и сортировки выносят в `IEqualityComparer<T>`/`IComparer<T>`, а не множат смыслы внутри `Equals`/`CompareTo`.
- `ToString` — для логов и отладки: коротко, без секретов, без тяжёлых вычислений.

## Связанное

- [[Записи (record) и структуры]] — что генерируется автоматически, а что нет
- [[Dictionary и HashSet изнутри]] — как хеш превращается в корзину
- [[Value types vs Reference types]] — почему структуры сравниваются иначе
- [[Иммутабельность как приём проектирования]] — почему ключи должны быть неизменяемыми
- [[Обзор коллекций .NET и как выбирать]] — где применяются компараторы
- [[DDD: тактические паттерны]] — равенство сущностей и value objects
- [[Отладка: точки останова, watch, immediate]] — `DebuggerDisplay` вместо `ToString`
