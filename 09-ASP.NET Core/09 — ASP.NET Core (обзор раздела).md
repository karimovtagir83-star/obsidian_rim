---
tags: [раздел-09, обзор, moc, middle, dotnet10]
aliases: [ASP.NET Core обзор, ASP.NET Core MOC, Web API overview]
---

# 09 — ASP.NET Core (обзор раздела)

> [!abstract] Коротко
> ASP.NET Core — кроссплатформенный веб-фреймворк .NET: сервер Kestrel, конвейер middleware, DI-контейнер из коробки, роутинг и два стиля описания эндпоинтов (Minimal API и MVC-контроллеры). Это самый большой раздел базы: именно здесь живёт 80 % повседневной работы backend-разработчика на .NET. Целевая версия — **ASP.NET Core 10 / .NET 10 LTS**.

## Как читать раздел

Раздел большой, но у него есть чёткий порядок. Идти строго сверху вниз не обязательно, но пропускать блок «Фундамент» нельзя — на нём висит всё остальное.

```
HTTP / REST                 ← что мы вообще передаём по сети
     │
Устройство хоста            ← кто принимает байты и куда их отдаёт
     │
Middleware + Роутинг        ← как запрос доходит до вашего кода
     │
DI                          ← как ваш код получает зависимости
     │
Minimal API / MVC           ← как вы описываете эндпоинты
     │
Валидация, ошибки, OpenAPI  ← как API становится пригодным для людей
     │
Инфраструктура              ← аутентификация, логи, кеш, фоновые задачи
```

## 0. База: сеть и контракт

- [[HTTP: методы, коды, заголовки, кеширование]] — семантика методов, идемпотентность, ETag, `Cache-Control`, HTTP/1.1 vs 2 vs 3
- [[REST: принципы и проектирование API]] — ограничения REST, уровни Ричардсона, пагинация, PATCH, HATEOAS
- [[Шпаргалка — HTTP коды и заголовки]] — быстрый справочник

## 1. Фундамент фреймворка

- [[Введение и устройство ASP.NET Core]] — путь запроса от сокета до эндпоинта, `WebApplication`, шаблон .NET 10
- [[Host, конфигурация и окружения]] — Generic Host, порядок источников конфигурации, `IOptions`, graceful shutdown
- [[Kestrel, reverse proxy и хостинг]] — лимиты, HTTPS, Nginx/YARP, `ForwardedHeaders`, Docker
- [[Middleware и конвейер обработки запроса]] — **ключевая заметка**: порядок, `Use`/`Run`/`Map`, своё middleware
- [[Роутинг]] — endpoint routing, шаблоны, constraints, `MapGroup`, `LinkGenerator`

## 2. Dependency Injection

Классика собеседований целиком живёт здесь.

- [[Dependency Injection: контейнер ASP.NET Core]] — **ключевая**: регистрация, `TryAdd*`, декораторы, `IEnumerable<T>`, Scrutor
- [[Жизненные циклы сервисов: Singleton, Scoped, Transient]] — **ключевая**: что такое scope, кто вызывает `Dispose`
- [[Captive dependency и типичные ошибки DI]] — Scoped в Singleton, Service Locator, `DbContext` в Singleton
- [[Options pattern и конфигурация сервисов]] — `IOptions`/`IOptionsSnapshot`/`IOptionsMonitor`, валидация, `AddXxx`
- [[Keyed services и продвинутая регистрация]] — keyed services (.NET 8+), фабрики, открытые дженерики

Теория за этим — в [[Инверсия зависимостей на практике]] и [[SOLID]].

## 3. Описание эндпоинтов

- [[Minimal API]] — **основной подход в 2026**: параметры, `TypedResults`, `MapGroup`, endpoint filters, организация кода
- [[MVC и контроллеры]] — `[ApiController]`, `ActionResult<T>`, что встретишь в легаси
- [[Model binding и валидация]] — источники привязки + **новая валидация .NET 10** (`AddValidation()`, source generator)
- [[FluentValidation]] — когда встроенной валидации не хватает
- [[Фильтры и endpoint filters]] — `IEndpointFilter` vs фильтры MVC vs middleware
- [[Загрузка файлов и работа с формами]] — `IFormFile`, multipart, стриминг без буферизации
- [[Server-Sent Events и стриминг ответов]] — **новое в .NET 10**: `TypedResults.ServerSentEvents`

## 4. Качество контракта

- [[Обработка ошибок и ProblemDetails]] — `IExceptionHandler`, RFC 9457, что нельзя показывать клиенту
- [[Версионирование API]] — стратегии, `Asp.Versioning`, депрекация
- [[OpenAPI и Swagger в .NET 10]] — Swashbuckle ушёл из шаблонов, `AddOpenApi`, OpenAPI 3.1, Scalar

## 5. Безопасность

- [[Аутентификация: обзор схем]]
- [[JWT и Bearer-аутентификация]]
- [[Авторизация: роли, политики, claims]]
- [[ASP.NET Core Identity]]
- [[OAuth 2.0 и OpenID Connect]]
- [[Passkeys и WebAuthn в .NET 10]]
- [[CORS]]
- [[Защита данных (Data Protection API)]]

Смежное: [[OWASP Top 10 для .NET]], [[HTTPS, TLS и сертификаты]], [[XSS, CSRF и защита от них]].

## 6. Эксплуатация

- [[Логирование и структурные логи]]
- [[Serilog]]
- [[Кеширование: Output, Response, HybridCache]]
- [[Rate limiting]]
- [[Health checks]]
- [[Конфигурация и секреты]]
- [[Background services и IHostedService]]
- [[Планировщики задач: Hangfire, Quartz]]

## 7. Исходящие вызовы и интеграции

- [[HttpClient и IHttpClientFactory]]
- [[Устойчивость: retry, circuit breaker, Polly]]
- [[SignalR]]
- [[gRPC в .NET]]

## 8. Что нового

- [[ASP.NET Core 10: что нового]] — сводка изменений .NET 10 в одном месте

## Что должен уметь Middle 2

| Навык | Где в разделе |
|---|---|
| Объяснить путь запроса от TCP-соединения до вашего лямбда-выражения | [[Введение и устройство ASP.NET Core]], [[Middleware и конвейер обработки запроса]] |
| Расставить middleware в правильном порядке и объяснить каждый | [[Middleware и конвейер обработки запроса]] |
| Выбрать lifetime сервиса и не создать captive dependency | [[Жизненные циклы сервисов: Singleton, Scoped, Transient]], [[Captive dependency и типичные ошибки DI]] |
| Спроектировать REST-контракт с пагинацией, ошибками и версиями | [[REST: принципы и проектирование API]], [[Версионирование API]] |
| Настроить валидацию и единый формат ошибок | [[Model binding и валидация]], [[Обработка ошибок и ProblemDetails]] |
| Выкатить сервис за reverse proxy и не потерять реальный IP клиента | [[Kestrel, reverse proxy и хостинг]] |
| Отдать OpenAPI-документ, по которому фронт сгенерирует клиент | [[OpenAPI и Swagger в .NET 10]] |
| Написать интеграционный тест на эндпоинт | [[Интеграционные тесты и WebApplicationFactory]] |

## Порядок изучения (практический)

1. **Неделя 1.** HTTP + REST + [[Введение и устройство ASP.NET Core]] + [[Middleware и конвейер обработки запроса]]. Поднять «Hello world» и своё middleware с логированием времени ответа.
2. **Неделя 2.** DI целиком (4 заметки блока 2). Написать сервис с тремя lifetime и посмотреть, что происходит в фоновом сервисе.
3. **Неделя 3.** [[Minimal API]] + [[Model binding и валидация]] + [[Обработка ошибок и ProblemDetails]] + [[OpenAPI и Swagger в .NET 10]]. Собрать CRUD с валидацией и ProblemDetails.
4. **Неделя 4.** Аутентификация/авторизация + [[Кеширование: Output, Response, HybridCache]] + [[Health checks]] + [[Логирование и структурные логи]].
5. **Дальше.** [[Проект 3 — REST API «Блог»]] и [[Проект 4 — Fintech: сервис платежей]].

> [!warning] Частая ошибка новичка в разделе
> Учить ASP.NET Core как набор рецептов («чтобы сделать X, вызови `AddX()`»). На собеседовании это видно моментально: вопрос «а почему `UseAuthentication` должен идти до `UseAuthorization`» превращает рецепт в тупик. Всегда доходите до механики — конвейер, scope, метаданные эндпоинта.

## Связанное

- [[🗺️ Роадмап (полный)]]
- [[Вопросы — ASP.NET Core]]
- [[Задачи — ASP.NET Core]]
- [[08 — Данные и EF Core (обзор раздела)]]
- [[10 — Архитектура и паттерны (обзор раздела)]]
- [[11 — Тестирование (обзор раздела)]]
- [[Чек-лист готовности к Middle 2]]
