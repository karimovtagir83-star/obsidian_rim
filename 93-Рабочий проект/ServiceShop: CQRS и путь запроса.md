---
tags: [раздел-93, рабочий-проект, cqrs, mediatr, декораторы, middle, собес]
aliases: [CQRS в ServiceShop, Путь запроса ServiceShop, Command Handler ServiceShop, Декораторы хендлеров, UnitOfWork ServiceShop]
---

# ServiceShop: CQRS и путь запроса

> [!abstract] Коротко
> Запись и чтение в проекте разведены полностью: команда идёт через MediatR → декораторы → хендлер → агрегат → EF Core, запрос идёт через MediatR → хендлер → Dapper → сырой SQL, минуя домен вообще. Это [[CQRS]] в лёгком варианте — одна база, два пути доступа, без отдельного read-хранилища.
> Второе, что нужно понять: в хендлере **никогда не увидишь `SaveChanges` или `BeginTransaction`**. Логирование, валидация и транзакция вынесены в три декоратора, которые Autofac навешивает автоматически. Хендлер содержит только бизнес-сценарий.

## Зачем это нужно

У чтения и записи разные требования, и попытка обслужить их одним кодом проигрывает обоим.

**Записи** нужны инварианты, транзакция и точность: загрузить агрегат целиком, проверить правила, изменить, сохранить. Объём данных мал — один заказ.

**Чтению** нужна скорость и произвольная форма: список заказов с именем партнёра, суммой, статусом доставки и превью первых трёх товаров. Прогонять это через агрегаты означает загрузить сотню полных объектов ради десяти полей, а дальше — либо N+1, либо `Include` на пол-базы.

CQRS признаёт, что это две разные задачи:

| | Command | Query |
|---|---|---|
| Смысл | «сделай» | «покажи» |
| Возвращает | ничего, id или ViewModel | данные |
| Путь | Handler → агрегат → Repository → EF Core | Handler → Dapper → SQL |
| Транзакция | да, `UnitOfWork` | нет |
| Проходит через домен | да | **нет, специально** |
| Валидация | FluentValidation в декораторе | обычно нет |

Правило «запросы не ходят через домен» форсируется автотестом `QueryHandlersShouldNotInjectDomainRepositoriesTest` — с XML-комментарием прямо в коде: «Query handlers must read via Dapper, not domain repositories». В тесте есть список унаследованных нарушений (~20 хендлеров, в основном Cart) и комментарий «не добавляй новые, а эти переводи на Dapper». Это хороший приём: правило включено сразу, старый код зафиксирован как долг, новый уже не может его увеличить.

---

## Контракты

У каждого модуля свои копии контрактов — следствие изоляции (см. [[ServiceShop: модульный монолит и изоляция модулей]]). `Orders.Application/Contracts/`:

```csharp
public interface ICommand : IRequest                        { Guid Id { get; } }
public interface ICommand<out TResult> : IRequest<TResult>   { Guid Id { get; } }

public interface IQuery<out TResult> : IRequest<TResult> { }   // без Id
```

`IRequest` — из MediatR ([[MediatR и альтернативы]]).

**Почему у команды есть `Id`, а у запроса нет.** `Id` нужен для идемпотентности и трассировки: по нему отложенная команда находится в таблице `internal_commands` и помечается обработанной. Запрос ничего не меняет, повторять его безопасно, идентифицировать незачем.

Базовый класс генерирует `Id` сам:

```csharp
public abstract class CommandBase<TResult> : ICommand<TResult>
{
    public Guid Id { get; }

    protected CommandBase() => Id = Guid.NewGuid();       // новая команда
    protected CommandBase(Guid id) => Id = id;            // восстановление из internal_commands
}
```

Два конструктора — не дублирование: первый для создания в контроллере, второй для восстановления команды из таблицы, где `Id` уже есть. Если бы конструктор был один, при повторном чтении из БД команда получила бы новый `Id` и потеряла связь со своей записью — то есть исполнилась бы дважды.

Хендлеры:

```csharp
public interface ICommandHandler<in TCommand>
    : IRequestHandler<TCommand> where TCommand : ICommand;

public interface ICommandHandler<in TCommand, TResult>
    : IRequestHandler<TCommand, TResult> where TCommand : ICommand<TResult>;

public interface IQueryHandler<in TQuery, TResult>
    : IRequestHandler<TQuery, TResult> where TQuery : IQuery<TResult> { }
```

Технически это просто `IRequestHandler` из MediatR с ограничением на тип. Зачем свои интерфейсы, если можно было использовать MediatR напрямую:

1. **Декораторы навешиваются на свой интерфейс.** Autofac оборачивает `ICommandHandler<,>`, а не `IRequestHandler<,>` — иначе транзакция открывалась бы и на запросах тоже.
2. **Тип обработчика виден из имени.** Ограничение `where TCommand : ICommand` не даст объявить `ICommandHandler` для запроса.
3. **ArchTests могут проверить правило** «хендлер называется `*CommandHandler` и реализует `ICommandHandler<,>`» — по интерфейсу MediatR так не различить.

> [!info] Про `in` и `out` в объявлениях
> `in TCommand` — контравариантность, `out TResult` — ковариантность. Практический смысл: `ICommand<Derived>` можно передать туда, где ждут `ICommand<Base>`. Именно это позволяет фасаду принимать `ICommand<TResult>` любого конкретного типа. Подробно — [[Дженерики: ограничения и вариантность]].

---

## Полный путь команды

Разбор `POST /api/cart/moderated-items` — «добавить товар в корзину».

```
 1. HTTP POST /api/cart/moderated-items
        │  body: { moderated_offer_id, quantity, condition_id }
        ▼
 2. CartController.AddModeratedItemToCartAsync
        │  AddModeratedItemToCartRequest → AddModeratedItemToCartCommand
        │  cartModule.ExecuteCommandAsync(command)
        ▼
 3. CartModule → CommandsExecutor.Execute(command)
        │  using var scope = CartCompositionRoot.BeginLifetimeScope();
        │  mediator.Send(command)
        ▼
 4. MediatR резолвит хендлер — но получает его в обёртках:
        ┌──────────────────────────────────────────────┐
        │ LoggingCommandHandlerWithResultDecorator     │  лог входа/выхода
        │  ┌────────────────────────────────────────┐  │
        │  │ ValidationCommandHandlerWithResult...  │  │  FluentValidation
        │  │  ┌──────────────────────────────────┐  │  │
        │  │  │ UnitOfWorkCommandHandlerWith...  │  │  │  транзакция
        │  │  │  ┌────────────────────────────┐  │  │  │
        │  │  │  │ AddModeratedItemToCart     │  │  │  │  ← бизнес-сценарий
        │  │  │  │ CommandHandler             │  │  │  │
        │  │  │  └────────────────────────────┘  │  │  │
        │  │  └──────────────────────────────────┘  │  │
        │  └────────────────────────────────────────┘  │
        └──────────────────────────────────────────────┘
        ▼
 5. Handler.Handle(...)
        │  buyer   = await _buyerProviderFactory.GetBuyerIdProvider().GetBuyer()
        │  itemDto = await _itemProvider.GetModeratedItemDto(...)
        │  cart    = await _cartRepository.GetByBuyerAsync(buyer) ?? Cart.Create(buyer)
        │  await cart.AddModeratedCartItem(...)   ← домен: правила + событие
        │  await _cartRepository.SaveAsync(cart)
        │  return ViewCartVm.FromEntity(cart)
        ▼
 6. UnitOfWork.CommitAsync (в декораторе)
        │  DomainEventsDispatcher.DispatchEventsAsync()
        │     ├── MediatR.Publish(доменные события) → внутренние обработчики
        │     └── сериализация → outbox_messages
        │  SaveChanges — одна транзакция на всё
        ▼
 7. ViewCartVm → 200 OK
```

Ключевой момент — **шаг 6**: доменные события и бизнес-данные попадают в базу **одной транзакцией**. Разбор дальнейшей судьбы события — [[ServiceShop: Outbox, Inbox и обмен событиями]].

### Как выглядит хендлер

`Cart.Application/Commands/ModeratedCartItems/AddModeratedItemToCart/AddModeratedItemToCartCommandHandler.cs`:

```csharp
public class AddModeratedItemToCartCommandHandler
    : ICommandHandler<AddModeratedItemToCartCommand, ViewCartVm>
{
    private readonly ICartRepository _cartRepository;
    private readonly IBuyerProviderFactory _buyerProviderFactory;
    private readonly IItemProvider _itemProvider;
    private readonly ICheckOfferPurchaseLimitForBuyer _checkOfferPurchaseLimitForBuyer;

    // ...конструктор с DI...

    public async Task<ViewCartVm> Handle(
        AddModeratedItemToCartCommand request, CancellationToken cancellationToken)
    {
        var buyer   = await _buyerProviderFactory.GetBuyerIdProvider().GetBuyer();
        var itemDto = await _itemProvider.GetModeratedItemDto(request.ModeratedOfferId, request.ConditionId);

        var cart = await _cartRepository.GetByBuyerAsync(buyer) ?? CartAggregate.Create(buyer);

        await cart.AddModeratedCartItem(
            itemDto.ModeratedOfferId, itemDto.ItemId, /* ...много параметров... */,
            _checkOfferPurchaseLimitForBuyer);

        await _cartRepository.SaveAsync(cart);

        return ViewCartVm.FromEntity(cart);
    }
}
```

Роль хендлера — **оркестрация, не логика**:

1. собрать данные, которых у агрегата нет (кто покупатель, что за товар);
2. загрузить или создать агрегат;
3. вызвать **один** доменный метод;
4. сохранить;
5. смапить в ViewModel.

Ни одной бизнес-проверки: все они внутри `AddModeratedCartItem`. Если в хендлере появился `if` про бизнес-условие — правило утекло из домена, см. [[ServiceShop: домен и тактический DDD]].

Заметь `_checkOfferPurchaseLimitForBuyer` — интерфейс, объявленный в домене и переданный **в доменный метод параметром**. Так домен получает доступ к внешней информации (лимиты покупателя), не зная про инфраструктуру. Приём называется «внедрение зависимости в метод» и применяется, когда тащить сервис в саму сущность нельзя — EF Core и Mongo-драйвер создают объекты без DI.

---

## Три декоратора

**Декоратор** — объект, реализующий тот же интерфейс, что и обёрнутый, и добавляющий поведение до/после. Теория — [[Паттерны GoF: структурные]].

Каждый существует в двух версиях — с результатом и без (`...Decorator` и `...WithResultDecorator`), потому что `ICommandHandler<T>` и `ICommandHandler<T, TResult>` — разные интерфейсы. Дублирование неизбежное: в C# нельзя написать один дженерик, покрывающий оба случая.

### Порядок применения важен

```
Logging → Validation → UnitOfWork → сам хендлер
```

Почему именно так:

- **Logging снаружи** — чтобы в лог попали и падения валидации;
- **Validation до UnitOfWork** — чтобы не открывать транзакцию под заведомо невалидную команду;
- **UnitOfWork ближе всех к хендлеру** — чтобы коммит происходил сразу после бизнес-логики, а не после чего-то ещё.

### Validation

```csharp
internal class ValidationCommandHandlerWithResultDecorator<T, TResult> : ICommandHandler<T, TResult>
    where T : ICommand<TResult>
{
    private readonly IList<IValidator<T>> _validators;
    private readonly ICommandHandler<T, TResult> _decorated;

    public Task<TResult> Handle(T command, CancellationToken cancellationToken)
    {
        var errors = _validators
            .Select(v => v.Validate(command))
            .SelectMany(result => result.Errors)
            .Where(error => error != null)
            .ToList();

        if (errors.Any())
            throw new InvalidCommandException(errors.Select(x => x.ErrorMessage).ToList());

        return _decorated.Handle(command, cancellationToken);
    }
}
```

Валидаторы приходят списком: `IList<IValidator<T>>`. Если валидатора для команды нет — список пуст, декоратор просто пропускает вызов дальше. Валидируется **форма** команды (обязательные поля, диапазоны), а не бизнес-правила — те в домене. Про FluentValidation — [[FluentValidation]].

`InvalidCommandException` маппится в `Startup.cs` в `InvalidCommandProblemDetails` → HTTP 400 с машиночитаемым телом ([[Обработка ошибок и ProblemDetails]]).

### UnitOfWork

```csharp
public async Task<TResult> Handle(T command, CancellationToken cancellationToken)
{
    var result = await _decorated.Handle(command, cancellationToken);

    if (command is InternalCommandBase<TResult>)
    {
        var internalCommand = await _ordersContext.InternalCommands
            .FirstOrDefaultAsync(x => x.Id == command.Id, cancellationToken);

        if (internalCommand != null)
            internalCommand.ProcessedDate = DateTime.UtcNow;
    }

    await _unitOfWork.CommitAsync(cancellationToken);

    return result;
}
```

Здесь два дела в одном месте, и это осмысленно:

1. **Отметка отложенной команды обработанной.** Если команда пришла из таблицы `internal_commands`, её `ProcessedDate` ставится **в той же транзакции**, что и результат работы. Иначе возможно: работа выполнена, а отметка не поставлена → команда исполнится повторно.
2. **Единственный коммит.** Один `CommitAsync` на всю команду. Отсюда и следует, что в хендлерах нет `SaveChanges`.

Про паттерн вообще — [[Repository и Unit of Work: нужны ли поверх EF Core]].

> [!warning] Транзакция шире, чем кажется
> `CommitAsync` внутри дёргает `DomainEventsDispatcher`, который публикует доменные события через MediatR **до** `SaveChanges`. Значит внутренний обработчик доменного события выполняется внутри той же транзакции и может добавить в неё свои изменения. Это даёт атомарность, но и создаёт риск: тяжёлый обработчик или внешний HTTP-вызов внутри него удлиняет транзакцию и держит блокировки в MySQL. Внутренние обработчики доменных событий должны быть быстрыми; всё тяжёлое — через `ICommandsScheduler` в `internal_commands`.

---

## Запросы

Хендлер запроса не касается домена вовсе:

```csharp
public class GetOrdersQueryHandler : IQueryHandler<GetOrdersQuery, IEnumerable<OrderListItemVm>>
{
    private readonly ISqlConnectionFactory _sqlConnectionFactory;

    public async Task<IEnumerable<OrderListItemVm>> Handle(
        GetOrdersQuery query, CancellationToken cancellationToken)
    {
        var connection = _sqlConnectionFactory.GetOpenConnection();

        const string sql = "select o.id, o.total_amount, o.status_id ... from orders o where ...";

        return await connection.QueryAsync<OrderListItemVm>(sql, new { query.ClientId });
    }
}
```

Три отличия от команды: `ISqlConnectionFactory` вместо репозитория, сырой SQL вместо LINQ, ViewModel вместо агрегата. Никакой транзакции, никакого change tracking. Про Dapper — [[Dapper: когда микро-ORM лучше]].

Ветвление запросов по клиентам видно в структуре папок Orders: `Queries/Web/`, `Queries/Mobile/`, `Queries/MiniApp/`, `Queries/AlifshopAdminPanel/`, `Queries/MerchantAdminPanel/`. Одни и те же данные, разная форма и права — и это правильное применение CQRS: под каждый экран свой запрос, вместо одного универсального с двадцатью опциональными параметрами.

Про типы, которые Dapper не умеет мапить сам (`MultiLang`, `ImagePaths`, `Diff<T>`), в `Startup.cs` зарегистрированы кастомные `SqlMapper.AddTypeHandler(...)` — около двадцати штук.

---

## Отложенные команды

`ICommandsScheduler` — четвёртый способ запустить команду, помимо контроллера, обработчика события и фоновой джобы:

```csharp
await _commandsScheduler.EnqueueAsync(new ChangeModeratedItemPriceCommand(
    Guid.NewGuid(), notification.ModeratedOfferId, notification.Price));
```

Команда сериализуется в таблицу `internal_commands` и выполняется позже джобой `ProcessInternalCommandsJob`. Зачем: вынести тяжёлую работу из текущей транзакции, сохранив гарантию, что она будет выполнена — запись в таблицу идёт в той же транзакции, что и всё остальное.

Классический случай — изменилась цена товара, надо обновить её в тысячах корзин. Обработчик события кладёт команду и завершается за миллисекунды; работу делает фон.

---

> [!example] Как делают в бою
> Куда смотреть, когда «команда не сработала, а ошибки нет»:
> 1. **Валидатор** — `InvalidCommandException` до входа в хендлер. В логах есть, в теле ответа 400 тоже, но при вызове из фоновой джобы легко пропустить.
> 2. **Правило в домене** — `BusinessRuleValidationException`. Смотреть `Message` правила, там обычно указаны конкретные значения.
> 3. **Идемпотентный `return`** — метод агрегата тихо вышел, потому что состояние уже целевое. Самый коварный случай: ни ошибки, ни изменений.
> 4. **Команда не доехала** — если шла через `internal_commands`: проверить `processed_date` в таблице и живость `ProcessInternalCommandsJob`.
>
> Порядок именно такой — от самого раннего барьера к самому позднему.

> [!warning] Подводные камни
> - **Нет `SaveChanges` в хендлере — это не забыли.** Попытка «добавить на всякий случай» ломает атомарность: часть изменений уедет в базу до диспатча доменных событий.
> - **`CancellationToken` часто игнорируется.** В `ValidationCommandHandlerWithResultDecorator` он даже не пробрасывается в валидаторы (используется синхронный `Validate`). При отмене запроса работа продолжится.
> - **Валидатор не заменяет бизнес-правило.** Валидация — форма команды (не пусто, в диапазоне). Бизнес-условие в валидаторе будет обойдено, когда тот же агрегат меняется другим сценарием.
> - **`internal_commands` — ещё один барьер отладки.** Между «команда поставлена» и «команда выполнена» проходит интервал джобы. В локальной разработке легко решить, что код не работает, не дождавшись тика.
> - **Запрос через доменный репозиторий** технически соберётся, но упадёт ArchTest — если только хендлер не в списке унаследованных исключений. Список расширять нельзя.
> - **Дублирование контрактов между модулями.** `ICommand`, `CommandBase`, все три декоратора существуют в пяти копиях. Правя баг в декораторе, надо помнить, что таких файлов пять — grep по имени класса обязателен.

---

## Вопросы с собеседований

> [!question]- Что такое CQRS и обязательно ли для него две базы данных?
> CQRS — разделение ответственности между операциями чтения и записи: у них разные модели, разные пути и разные требования. Две базы **не обязательны** — это отдельное решение (CQRS с раздельными хранилищами и синхронизацией через события), и оно приносит eventual consistency со всеми её сложностями. ServiceShop использует лёгкий вариант: одна MySQL, но запись идёт через агрегаты и EF Core, а чтение — через Dapper и сырой SQL прямо в ViewModel. Этого достаточно, чтобы получить главную выгоду — не тащить домен туда, где нужен быстрый селект.

> [!question]- Зачем декораторы вокруг хендлеров, если есть pipeline behaviors в MediatR?
> `IPipelineBehavior` из MediatR решает ту же задачу и в новом проекте был бы естественнее. Декораторы через Autofac дают два отличия: во-первых, они навешиваются на **свой** интерфейс `ICommandHandler<,>`, поэтому транзакция не открывается на запросах — с behaviors пришлось бы проверять тип внутри; во-вторых, порядок задаётся явно в регистрации контейнера, а не порядком добавления в MediatR. Плюс историческая причина: эта схема пришла из образцового проекта Kamil Grzybek, на который ориентирован ServiceShop. Практический вывод: важен не выбор механизма, а то, что сквозная логика вынесена из хендлеров.

> [!question]- Почему у команды есть `Guid Id`, а у запроса нет?
> `Id` команды нужен для идемпотентности: команда может быть сохранена в `internal_commands` и выполнена позже, и по этому `Id` в том же коммите ставится `processed_date` — иначе при перезапуске она исполнится дважды. Плюс `Id` служит корреляцией в логах. Запрос ничего не меняет, повторить его безопасно, поэтому идентифицировать незачем. Показательная деталь: `CommandBase` имеет два конструктора — один генерирует новый `Id`, второй принимает существующий, именно для восстановления команды из таблицы.

> [!question]- Что произойдёт, если в хендлере вызвать `SaveChanges` вручную?
> Часть изменений уедет в базу до того, как `DomainEventsDispatcher` соберёт доменные события и запишет их в `outbox_messages`. Атомарность «данные + событие» ломается: возможен коммит бизнес-данных без события — и тогда другие модули никогда не узнают о произошедшем, а восстановить это автоматически нечем. Ровно эту проблему Outbox и решает, а ручной `SaveChanges` её возвращает.

> [!question]- Чем валидация в декораторе отличается от бизнес-правила в домене?
> Валидация проверяет **форму команды**: поле не пустое, количество больше нуля, guid не пустой. Она не знает состояния системы и работает до загрузки чего-либо из базы. Бизнес-правило проверяет **состояние домена**: «заказ в статусе, из которого можно отменить», «на складе хватает товара». Ключевое различие в надёжности: валидатор привязан к конкретной команде, и если тот же агрегат меняется другим сценарием — валидатор не сработает. Правило внутри агрегата сработает всегда, потому что находится на пути любого изменения.

> [!question]- Зачем `ICommandHandler`, если можно использовать `IRequestHandler` из MediatR?
> Три причины. Декораторы навешиваются на свой интерфейс — обёртывание `IRequestHandler` затянуло бы в транзакцию и запросы. Ограничение `where TCommand : ICommand` не даёт по ошибке объявить командный хендлер для запроса. И ArchTests могут проверить конвенцию «`*CommandHandler` реализует `ICommandHandler<,>`» — по общему интерфейсу MediatR команду от запроса не отличить.

---

## Задачи

### Задача 1. Проследить путь команды

Дана команда `CancelOrderCommand`, вызванная из контроллера. Перечислить по порядку все объекты, через которые пройдёт вызов до строчки, меняющей статус в памяти.

> [!success]- Решение
> ```
> OrdersController                                    HTTP → команда
> IOrdersModule (OrdersModule)                        фасад модуля
> CommandsExecutor.Execute                            открывает LifetimeScope
> OrdersCompositionRoot.BeginLifetimeScope            свой контейнер модуля
> IMediator.Send                                      резолв хендлера
> LoggingCommandHandlerDecorator                      лог входа
> ValidationCommandHandlerDecorator                   FluentValidation
> UnitOfWorkCommandHandlerDecorator                   до вызова — ничего
> CancelOrderCommandHandler.Handle                    оркестрация
> IOrderRepository.GetByIdAsync                       загрузка агрегата
> Order.CancelAsync                                   ← здесь меняется статус
> ```
> Дальше на обратном пути: `UnitOfWork.CommitAsync` → `DomainEventsDispatcher.DispatchEventsAsync` → `MediatR.Publish` внутренних событий → запись в `outbox_messages` → `SaveChanges`.
>
> Одиннадцать звеньев до бизнес-логики. Это цена изоляции и сквозной обвязки: каждое звено делает ровно одно дело, но при отладке приходится помнить всю цепочку.

### Задача 2. Куда положить проверку

В задаче: «нельзя добавить в корзину товар, если у покупателя уже 50 позиций». Куда положить проверку — в валидатор команды, в хендлер или в домен? Обосновать.

> [!success]- Решение
> **В домен**, правилом `CartItemsCountMustNotExceedLimitRule`, вызываемым из `Cart.AddModeratedCartItem`.
>
> Почему не валидатор: валидатор видит только команду, а количество позиций — это состояние агрегата, которое ещё не загружено. Валидатор в принципе не может это проверить.
>
> Почему не хендлер: технически возможно (агрегат уже загружен), но правило окажется привязано к одному сценарию. Товар попадает в корзину минимум тремя путями — `AddModeratedItemToCart`, `AddItemsToCart` (bulk), `MergeCarts` при авторизации. Проверка в одном хендлере оставит две дыры, и найдут их в проде.
>
> В домене правило стоит на пути любого изменения — все три сценария в итоге вызывают доменный метод.

### Задача 3. Найти проблему в хендлере

```csharp
public async Task<Unit> Handle(ApproveOrderCommand request, CancellationToken ct)
{
    var order = await _orderRepository.GetByIdAsync(request.OrderId);

    if (order.Status.Id != OrderStatus.Reviewing.Id)
        throw new InvalidOperationException("Wrong status");

    await order.ApproveAsync();
    await _orderRepository.SaveAsync(order);
    await _dbContext.SaveChangesAsync(ct);

    await _notificationService.SendPushAsync(order.ClientId, "Заказ одобрен");

    return Unit.Value;
}
```

Найти три проблемы.

> [!success]- Решение
> **1. Бизнес-проверка в хендлере.** `if (order.Status.Id != OrderStatus.Reviewing.Id)` — это правило `OrderMustBeReviewingToApproveRule`, и оно должно быть внутри `ApproveAsync`. В хендлере оно не сработает при одобрении из другого сценария, а тип исключения (`InvalidOperationException` вместо `BusinessRuleValidationException`) даст клиенту 500 вместо осмысленного 400.
>
> **2. Ручной `SaveChangesAsync`.** Коммит произойдёт до того, как декоратор `UnitOfWork` вызовет `DomainEventsDispatcher`. Доменное событие `OrderApprovedEvent` не попадёт в `outbox_messages` в той же транзакции — атомарность «данные + событие» потеряна.
>
> **3. Отправка пуша прямо в хендлере.** Побочный эффект вне транзакции: если коммит откатится, пуш уже улетел, и клиент получит уведомление об одобрении несуществующего заказа. Правильно — доменное событие `OrderApprovedEvent` → Outbox → подписчик в Marketing.
>
> Исправленный хендлер — четыре строки: загрузить, вызвать доменный метод, сохранить, вернуть. Всё остальное делают декоратор и подписчики событий.

---

## Итог

- Команда идёт через агрегаты и EF Core, запрос — напрямую через Dapper, минуя домен; это проверяется ArchTest.
- Свои `ICommand`/`ICommandHandler` поверх MediatR нужны, чтобы декораторы и архитектурные тесты могли различать команды и запросы.
- Три декоратора в порядке Logging → Validation → UnitOfWork; ровно поэтому в хендлерах нет `SaveChanges`.
- `UnitOfWork` делает две вещи в одной транзакции: помечает отложенную команду обработанной и коммитит всё, включая записи Outbox.
- Хендлер — оркестрация: собрать данные, загрузить агрегат, вызвать один доменный метод, сохранить, смапить в VM.
- `Guid Id` у команды существует ради идемпотентности и связи с `internal_commands`.
- Тяжёлая работа выносится через `ICommandsScheduler` в `internal_commands`, а не делается в текущей транзакции.

## Связанное

- [[93 — Рабочий проект: Alif ServiceShop (обзор раздела)]]
- [[ServiceShop: домен и тактический DDD]]
- [[ServiceShop: Outbox, Inbox и обмен событиями]]
- [[ServiceShop: модульный монолит и изоляция модулей]]
- [[CQRS]]
- [[MediatR и альтернативы]]
- [[Паттерны GoF: структурные]]
- [[Repository и Unit of Work: нужны ли поверх EF Core]]
- [[Dapper: когда микро-ORM лучше]]
- [[FluentValidation]]
- [[Обработка ошибок и ProblemDetails]]
