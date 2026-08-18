---
tags: [раздел-93, рабочий-проект, архитектура, onboarding, middle]
aliases: [Alif ServiceShop, ServiceShop, Рабочий проект, alifshop.backend.service-shop, Обзор рабочего проекта]
---

# 93 — Рабочий проект: Alif ServiceShop (обзор раздела)

> [!abstract] Коротко
> Раздел разбирает реальный боевой код, с которым идёт работа каждый день: `alifshop.backend.service-shop` — бэкенд маркетплейса Alif Shop. Это **модульный монолит на DDD + CQRS + event-driven**: одно ASP.NET Core-приложение, внутри пять почти независимых модулей, обмен между ними только через таблицы `outbox_messages`/`inbox_messages` и Kafka.
> Ценность раздела не в описании продукта, а в том, что почти каждая теоретическая заметка базы здесь имеет живой пример: [[Clean Architecture]], [[CQRS]], [[Паттерн Transactional Outbox]], [[Доменные события]], [[Идемпотентность]] — всё это в проекте не абстракции, а конкретные файлы, которые можно открыть.

## Зачем этот раздел

Теория без боевого кода запоминается плохо, а боевой код без теории читается как случайный набор решений. Раздел закрывает разрыв в обе стороны:

- когда в проекте встречается непонятная конструкция — тут написано, какой паттерн за ней стоит и почему выбран именно он;
- когда в базе изучается паттерн — тут показано, как он выглядит, когда его применили к 300 тысячам строк, пяти модулям и десятку внешних интеграций.

Отдельная польза: разбор чужого большого кода — это навык уровня Middle, см. [[Работа с легаси]]. ServiceShop не легаси, но объём тот же, и подход к чтению такой же: не читать всё, а находить точки входа и идти по одному сценарию до конца.

---

## Что это за продукт

Маркетплейс: витрина товаров, корзина, оформление заказа, оплата, модерация товаров продавцов, пуш-уведомления. Уникальная для рынка часть — **оплата в рассрочку**: интеграция с кредитными продуктами Alif (Nasiya BNPL, BML), поэтому оформление заказа — не «списали деньги с карты», а «создали кредитную заявку, дождались решения скоринга, синхронизировали статусы».

Именно из-за этого модуль заказов — самая сложная часть системы: у него не один флоу оплаты, а пять, и каждый со своими статусами и внешними сервисами.

---

## Стек

| Слой | Технология | Заметка базы |
|---|---|---|
| Рантайм | .NET 8 (`net8.0`) | [[Файл проекта .csproj изнутри]] |
| Веб | ASP.NET Core, контроллеры MVC | [[MVC и контроллеры]] |
| DI | **Autofac**, не встроенный контейнер | [[Dependency Injection: контейнер ASP.NET Core]] |
| Медиатор | MediatR 12 | [[MediatR и альтернативы]] |
| ORM (запись) | EF Core 8 + Pomelo для MySQL | [[EF Core: введение и DbContext]] |
| ORM (чтение) | Dapper | [[Dapper: когда микро-ORM лучше]] |
| Спецификации | Ardalis.Specification | [[Specification pattern]] |
| Миграции | **FluentMigrator**, не EF Migrations | [[EF Core: миграции]] |
| Валидация | FluentValidation | [[FluentValidation]] |
| Брокер | Kafka через KafkaFlow 4 | [[Apache Kafka: основы]] |
| Планировщик | Quartz | [[Планировщики задач: Hangfire, Quartz]] |
| БД | MySQL 8, MongoDB (только корзина), Redis | [[NoSQL: когда и зачем]], [[Redis и стратегии кеширования]] |
| Логи | **Serilog**, `Microsoft.Extensions.Logging` запрещён | [[Serilog]] |
| Телеметрия | OpenTelemetry + Sentry + Prometheus | [[OpenTelemetry в .NET]] |
| Файлы | MinIO, Firebase (пуши) | |
| Тесты | xUnit + FluentAssertions + NSubstitute/Moq + NetArchTest | [[xUnit: основы]], [[Моки: NSubstitute и Moq]] |
| Деплой | Docker, Kubernetes, Werf, Helm | [[Kubernetes: минимум для разработчика]] |

> [!warning] Проект на .NET 8, а база учит .NET 10
> База сознательно написана под .NET 10 / C# 14 (см. `_Шаблоны/СПЕЦИФИКАЦИЯ.md`), проект — на `net8.0`. Практические следствия: в коде нет `field`-ключевого слова, новых extension members и `?.=`; в EF Core 8 нет именованных фильтров запросов и complex types в JSON. Значит, часть приёмов из базы в этом коде применить нельзя — не потому что «плохо написано», а потому что версия старше.
> Полезная привычка: встретив в базе фичу .NET 10, проверять, доступна ли она в `net8.0`, прежде чем предлагать в ревью.

Ещё две настройки уровня решения, которые бьют по рукам сразу (`Directory.Build.props`):

- `TreatWarningsAsErrors` — любое предупреждение компилятора ломает сборку. Плюс StyleCop-анализаторы. Сборка падает из-за форматирования — это норма проекта, а не поломка.
- `Nullable enable` — строгий режим nullable по всей кодовой базе, см. [[Nullable reference types]].
- `ManagePackageVersionsCentrally` — версии пакетов только в `Directory.Packages.props`, в `.csproj` их не пишут.

---

## Пять модулей

| Модуль | Ответственность | Ключевые агрегаты |
|---|---|---|
| **Orders** | весь жизненный цикл заказа, оплата, кредитные заявки, доставка | `Order`, `Purchase`, `BuyerCard`, `PartnerClone`, `OrderPromoCodeUsage` |
| **Catalog** | товары, офферы продавцов, категории, бренды, атрибуты, витрины, промо | `ModeratedOffer`, `Offer`, `Model`, `Category`, `Brand`, `Promo` |
| **Cart** | корзина, шаренная корзина, вишлист, синхронизация с каталогом | `Cart`, `SharedCart`, `Wishlist` |
| **Marketing** | клиенты, устройства, промокоды, пуш-кампании, уведомления | `Client`, `Device`, `PromoCode`, `PushNotification` |
| **Moderation** | апрув офферов продавца перед публикацией | `OfferDemand`, `PartnerClone` |

Каждый модуль — четыре проекта: `Domain`, `Application`, `Infrastructure`, `IntegrationEvents`, плюс `*.UnitTests`. Разбор изоляции — [[ServiceShop: модульный монолит и изоляция модулей]].

### Бизнес-словарь, без которого код не читается

| Термин | Значение |
|---|---|
| **Offer** | предложение продавца: конкретный товар по конкретной цене у конкретного партнёра |
| **ModeratedOffer** | оффер, прошедший модерацию; именно он виден в витрине и попадает в корзину |
| **Partner / Store** | продавец на маркетплейсе |
| **FBO / FBS** | Fulfilled by Operator (товар на складе маркетплейса) / by Seller (на складе продавца) — влияет на доставку и на то, кто может менять состав заказа |
| **Nasiya** | кредитная система Alif; BNPL — buy now pay later, рассрочка |
| **BML** | Buy Me Later — отдельный кредитный продукт со своими лимитами |
| **AlifMobi Native / Acquiring** | оплата внутри приложения AlifMobi: нативная и через эквайринг |
| **Hold** | удержание суммы на карте до подтверждения заказа |

---

## Три идеи, которые объясняют почти весь код

1. **DDD** — бизнес-логика внутри доменных объектов (`Order.ApproveAsync()`), а не в сервисах и не в контроллерах → [[ServiceShop: домен и тактический DDD]]
2. **CQRS** — запись и чтение идут разными путями и разными библиотеками (EF Core против Dapper) → [[ServiceShop: CQRS и путь запроса]]
3. **Event-driven через Outbox/Inbox** — модули не вызывают друг друга, а обмениваются событиями через таблицы БД → [[ServiceShop: Outbox, Inbox и обмен событиями]]

Плюс инфраструктурный слой: хранилища, миграции, автотесты архитектуры → [[ServiceShop: данные, миграции и тесты]].

---

## Карта путей: где что искать

Имя проекта — это адрес: `Alif.ServiceShop.<Область>.<Имя>.<Слой>`.

```
Alif.ServiceShop.API                          ← единственный запускаемый проект
├── Modules/<Модуль>/**/*Controller.cs         ← все HTTP-эндпоинты
├── Startup.cs                                 ← инициализация всех модулей
└── Configuration/                             ← Swagger, OpenTelemetry, Sentry, ProblemDetails

Alif.ServiceShop.BuildingBlocks.Domain         ← Entity, ValueObject, IBusinessRule, IDomainEvent
Alif.ServiceShop.BuildingBlocks.Application    ← Outbox, DomainEventNotification, исключения
Alif.ServiceShop.BuildingBlocks.Infrastructure ← DomainEventsDispatcher, EventBus, репозитории
Alif.ServiceShop.BuildingBlocks.EventBus       ← in-memory шина

Alif.ServiceShop.Modules.<M>.Domain            ← агрегаты, правила, доменные события
Alif.ServiceShop.Modules.<M>.Application       ← Commands/, Queries/, Contracts/, Services/
Alif.ServiceShop.Modules.<M>.Infrastructure    ← Configuration/ (DI, Quartz, Kafka), Migrations/
Alif.ServiceShop.Modules.<M>.IntegrationEvents ← публичные контракты событий модуля

ArchTests                                      ← тесты, проверяющие саму архитектуру
docs/adr/                                       ← ADR: почему решили так
```

### Шпаргалка «где искать, если…»

| Вопрос | Куда смотреть |
|---|---|
| Откуда взялся эндпоинт? | `API/Modules/<Модуль>/**/*Controller.cs` |
| Где бизнес-правило X? | `<M>.Domain/<Агрегат>/BusinessRules/` — имена файлов читаются как требования |
| Почему заказ не меняет статус? | метод агрегата в `Order.cs` + правило, которое он проверяет |
| Кто зарегистрировал этот сервис? | `<M>.Infrastructure/Configuration/<M>Startup.cs` |
| Событие не дошло до модуля? | `EventsBusStartup` получателя → таблица `inbox_messages` → `ProcessInboxJob` |
| Событие не ушло наружу? | карта `domainNotificationsMap` в `<M>Startup.cs` → `outbox_messages` → `ProcessOutboxJob` |
| Что за фоновая задача? | `<M>.Infrastructure/Configuration/Quartz/` и `Processing/*Job.cs` |
| Где SQL для списка? | `<M>.Application/Queries/**/*QueryHandler.cs` (Dapper) |
| Как поменять схему БД? | скилл `create-migration`, файл ляжет в `<M>.Infrastructure/Migrations/` |

---

## Правила проекта, которые нельзя нарушать

1. Логирование — **только Serilog** (`using ILogger = Serilog.ILogger;`).
2. DI — **только Autofac**, регистрации в `*Startup.cs` модуля.
3. Конструкторы агрегатов — **только `private`**, создание через `static Create(...)`.
4. Командный хендлер называется `*CommandHandler` и реализует `ICommandHandler<,>`.
5. Любое изменение схемы БД — **только через скилл `create-migration`**, даже одна колонка.
6. Межмодульное общение — только через Integration Event или фасад модуля.
7. Интеграционные события публикуются через Outbox, не синхронно.
8. Сложные запросы — через Ardalis.Specification, а не сырой SQL в хендлере.

Пункты 3, 4 и часть 6 проверяются автотестами — нарушение не доедет до ревью, упадёт сборка. Подробно: [[ServiceShop: данные, миграции и тесты]].

---

## Маршрут чтения кода на первую неделю

**День 1 — базовые кирпичи.** Файлы маленькие, читаются за час:
`BuildingBlocks.Domain/Entity.cs` → `IBusinessRule.cs` → `IDomainEvent.cs` → `ValueObject.cs` → `Orders.Application/Contracts/ICommand.cs` → `Configuration/Commands/ICommandHandler.cs`.

**День 2 — один сквозной сценарий на самом простом модуле (Cart).** `POST /api/cart/moderated-items`:
`API/Modules/Cart/WebAndMobile/Cart/CartController.cs` → `AddModeratedItemToCartCommand.cs` → `AddModeratedItemToCartCommandHandler.cs` → `Cart.Domain/Aggregates/Cart/Cart.cs` → `BusinessRules/ModeratedOfferMustBeAvailableRule.cs`.

**День 3 — инфраструктура и события.**
`Orders.Infrastructure/Configuration/OrdersStartup.cs` → `BuildingBlocks.Infrastructure/DomainEventsDispatching/DomainEventsDispatcher.cs` → `Processing/Outbox/ProcessOutboxCommandHandler.cs` → `Cart.Infrastructure/Configuration/EventsBus/`.

**День 4 — сложный домен.**
`Orders.Domain/Order/AggregateRoot/Order.cs` (метод `ApproveAsync` как эталон) → пробежаться по именам всех файлов в `Order/BusinessRules/` → `Order/OrderStatus.cs`.

**День 5 — контекст решений.** `docs/adr/` (три ADR: unified reserve/release events, alif card promotion toggle, actor audit moderated offer) и `readme.md` с обзором продукта. Про формат ADR — [[Документация: ADR и RFC]].

> [!tip] Как не утонуть
> Не читать модуль «целиком». Всегда идти по одному сценарию сверху вниз: эндпоинт → команда → хендлер → агрегат → репозиторий. Один пройденный до конца сценарий даёт больше, чем пятьдесят открытых наугад файлов — потому что закрепляется не факт, а схема, по которой построены все остальные сценарии.

---

## Чек-лист: как добавить фичу

```
1. Определить модуль по ответственности
2. Domain:
   - новый тип идентификатора → Value Object
   - новое ограничение → класс в BusinessRules/, реализует IBusinessRule
   - метод агрегата: CheckRuleAsync(...) → изменить состояние → AddDomainEvent(...)
3. Application:
   - <Действие>Command.cs        (наследует CommandBase / CommandBase<TResult>)
   - <Действие>CommandHandler.cs (реализует ICommandHandler<,>)
   - при необходимости валидатор FluentValidation
4. Infrastructure:
   - репозиторий / конфигурация EF, если появились поля
   - регистрация сервисов в <M>Startup.cs
   - миграция → только через скилл create-migration
5. API: тонкий метод контроллера → команда → module.ExecuteCommandAsync
6. Нужно оповестить другой модуль?
   - IntegrationEvent в <M>.IntegrationEvents
   - IDomainEventNotification + запись в domainNotificationsMap
   - подписка SubscribeToIntegrationEvent<T> в EventsBusStartup получателя
   - обработчик INotificationHandler<T> → ICommandsScheduler.EnqueueAsync(...)
7. Тесты: юнит-тест на правило + dotnet test ArchTests/
```

---

## Итог

- ServiceShop — модульный монолит: один процесс, пять изолированных модулей, у каждого свой DI-контейнер и своя БД.
- Три несущие идеи: DDD в домене, CQRS в приложении, обмен событиями через Outbox/Inbox между модулями.
- Проект на `net8.0` — часть приёмов из базы (`.NET 10`, C# 14, EF Core 10) здесь недоступна; сверяться перед применением.
- Миграции — FluentMigrator и только через скилл `create-migration`; выдуманный таймстемп ломает порядок накатки у всей команды.
- Архитектурные правила защищены не соглашениями, а автотестами в `ArchTests` — они падают до ревью.
- Читать код только сквозными сценариями, начиная с модуля Cart как самого простого.

## Связанное

- [[ServiceShop: модульный монолит и изоляция модулей]]
- [[ServiceShop: домен и тактический DDD]]
- [[ServiceShop: CQRS и путь запроса]]
- [[ServiceShop: Outbox, Inbox и обмен событиями]]
- [[ServiceShop: данные, миграции и тесты]]
- [[Clean Architecture]]
- [[Монолит vs микросервисы: как решать]]
- [[Работа с легаси]]
