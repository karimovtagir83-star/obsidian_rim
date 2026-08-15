---
tags: [раздел-04, основы, инструменты, dotnet10]
aliases: [dotnet CLI, dotnet команды, global.json, dotnet tool, dotnet watch]
---

# dotnet CLI: полный обзор

> [!abstract] Коротко
> `dotnet` — единая точка входа ко всему: создание проектов, восстановление зависимостей, сборка, запуск, тесты, публикация, упаковка, инструменты и диагностика. IDE не делает ничего, чего нельзя сделать этими командами, — она их и вызывает. Знать CLI обязательно, потому что CI, Docker и продовая диагностика живут только в терминале. Минимальный рабочий набор — десяток команд; остальное смотрится по `--help`.

## Зачем это нужно

Три ситуации, где без CLI не обойтись: сборка в CI (там нет IDE), Dockerfile (`dotnet restore` и `dotnet publish` — обычные шаги), диагностика на сервере (`dotnet-counters`, `dotnet-dump`). Плюс локально команды часто быстрее, чем меню: `dotnet watch` перезапускает приложение при изменении файла, `dotnet format` приводит код к `.editorconfig`, `dotnet test --filter` гоняет один тест.

---

## Проект и решение

```bash
dotnet new list                        # все доступные шаблоны
dotnet new console -o Sandbox           # консольное приложение
dotnet new web -o Api                   # минимальное веб-приложение
dotnet new webapi -o Api                # API с контроллерами
dotnet new webapiaot -o Api             # API под Native AOT
dotnet new classlib -o Domain
dotnet new xunit -o Domain.Tests
dotnet new gitignore                    # .gitignore под .NET
dotnet new editorconfig                 # заготовка .editorconfig
dotnet new globaljson --sdk-version 10.0.100

# Решение
dotnet new sln -n Shop
dotnet sln add src/**/*.csproj
dotnet sln list

# Ссылки между проектами
dotnet add src/Api/Api.csproj reference src/Domain/Domain.csproj
dotnet list src/Api/Api.csproj reference
```

Шаблоны расширяемы: `dotnet new install <PackageId>` ставит чужие (например, шаблоны Clean Architecture), `dotnet new uninstall` удаляет.

### global.json: закрепление версии SDK

```json
{
  "sdk": {
    "version": "10.0.100",
    "rollForward": "latestFeature"
  }
}
```

Файл в корне репозитория фиксирует версию SDK для всех, кто собирает проект: разработчик с более новым SDK получит ту же сборку, что и CI. `rollForward` управляет тем, насколько свободно можно подниматься по версиям (`patch`, `feature`, `minor`, `major`, `disable`). Без `global.json` берётся самый свежий установленный SDK, и поведение может отличаться между машинами.

---

## Пакеты

```bash
dotnet add package Serilog.AspNetCore                    # последняя стабильная
dotnet add package Npgsql --version 10.0.0
dotnet remove package Newtonsoft.Json

dotnet restore                                            # восстановить зависимости
dotnet restore --locked-mode                              # строго по packages.lock.json

dotnet list package                                       # прямые зависимости
dotnet list package --include-transitive                  # весь граф
dotnet list package --vulnerable --include-transitive     # уязвимости
dotnet list package --outdated                            # доступные обновления
dotnet nuget why Api System.Text.Json                     # откуда взялся пакет

dotnet nuget locals all --clear                           # очистить кеш пакетов
```

Подробно про версии и конфликты — [[NuGet: пакеты, версии, транзитивные зависимости]].

---

## Сборка и запуск

```bash
dotnet build                        # сборка, по умолчанию Debug
dotnet build -c Release
dotnet build -f net10.0             # конкретный TFM при мультитаргетинге
dotnet build --no-restore           # если restore уже был (важно для CI)
dotnet build -warnaserror           # предупреждения как ошибки разово

dotnet run                          # сборка + запуск
dotnet run -c Release
dotnet run --project src/Api        # из корня решения
dotnet run -- --arg value           # всё после -- уходит в приложение
dotnet run --launch-profile Https   # профиль из launchSettings.json

dotnet watch                        # перезапуск при изменении файлов
dotnet watch test                   # тот же режим для тестов

dotnet clean
```

Передача свойств MSBuild — через `-p:`:

```bash
dotnet build -c Release -p:Version=1.2.3 -p:TreatWarningsAsErrors=true
dotnet publish -p:PublishReadyToRun=true -p:PublishSingleFile=true
```

### Файловые приложения (.NET 10)

```bash
dotnet run script.cs          # запуск одного файла без проекта
```

```csharp
#!/usr/bin/env dotnet
#:package Humanizer@2.14.1

Console.WriteLine("Скрипт без csproj");
```

Удобно для утилит, экспериментов и обучения: попробовать конструкцию языка теперь можно без создания проекта ([[Новое в C# 12, 13, 14]]).

---

## Тесты

```bash
dotnet test
dotnet test -c Release --no-build
dotnet test --filter "FullyQualifiedName~OrderTests"     # по имени
dotnet test --filter "Category=Integration"               # по трейту
dotnet test --logger "trx;LogFileName=results.trx"        # отчёт для CI
dotnet test --collect:"XPlat Code Coverage"               # покрытие
dotnet test -- xunit.parallelizeAssembly=true             # параметры раннера
```

Код возврата `dotnet test` ненулевой при падении тестов — на этом строятся шаги CI ([[CI/CD: GitHub Actions]]).

---

## Публикация

```bash
# Framework-dependent: нужен установленный рантайм, минимальный размер
dotnet publish -c Release -o out

# Self-contained: рантайм внутри, ничего ставить не нужно
dotnet publish -c Release -r linux-x64 --self-contained -o out

# Ускорение старта предкомпиляцией
dotnet publish -c Release -r linux-x64 -p:PublishReadyToRun=true

# Один файл
dotnet publish -c Release -r win-x64 --self-contained -p:PublishSingleFile=true

# Native AOT
dotnet publish -c Release -r osx-arm64 -p:PublishAot=true
```

Что выбирать в каком случае — [[JIT, Tiered Compilation, ReadyToRun, Native AOT]].

```bash
# Библиотека в пакет
dotnet pack -c Release
dotnet nuget push bin/Release/*.nupkg -s https://api.nuget.org/v3/index.json -k $KEY
```

---

## Инструменты (tools)

```bash
# Глобальные — доступны из любой папки
dotnet tool install -g dotnet-ef
dotnet tool install -g dotnet-counters
dotnet tool list -g
dotnet tool update -g dotnet-ef

# Локальные — версии зафиксированы в репозитории (.config/dotnet-tools.json)
dotnet new tool-manifest
dotnet tool install dotnet-ef
dotnet tool restore                    # на новой машине или в CI
dotnet ef migrations add Init          # запуск локального инструмента
```

Локальные инструменты предпочтительнее: манифест коммитится, и у всей команды одинаковые версии. Для глобальных нужно, чтобы `~/.dotnet/tools` был в `PATH` — частая причина ошибки «command not found» после успешной установки ([[Командная строка Linux для .NET-разработчика]]).

Полезные инструменты:

| Инструмент | Зачем |
|---|---|
| `dotnet-ef` | миграции и работа с моделью EF Core |
| `dotnet-counters` | живые метрики процесса: GC, пул потоков, запросы |
| `dotnet-trace` | профиль исполнения без остановки процесса |
| `dotnet-dump` | снятие и анализ дампов памяти |
| `dotnet-gcdump` | лёгкий дамп только графа объектов |
| `dotnet-monitor` | сбор диагностики в контейнере по HTTP |
| `dotnet-outdated` | обновление зависимостей (сторонний) |

---

## Форматирование и секреты

```bash
dotnet format                             # исправить по .editorconfig
dotnet format --verify-no-changes         # проверка без изменений (шаг CI)
dotnet format style --severity warn

# Секреты для разработки: хранятся вне репозитория, в профиле пользователя
dotnet user-secrets init
dotnet user-secrets set "ConnectionStrings:Default" "Host=localhost;..."
dotnet user-secrets list

# HTTPS-сертификат для локальной разработки
dotnet dev-certs https --trust
```

`user-secrets` — правильный способ хранить локальные строки подключения и ключи: файл лежит в профиле пользователя, а не в репозитории, и подхватывается конфигурацией автоматически в окружении Development ([[Конфигурация и секреты]]).

---

## Диагностика окружения

```bash
dotnet --info                    # версии SDK и рантаймов, RID, пути
dotnet --list-sdks
dotnet --list-runtimes
dotnet --version                 # версия SDK, выбранная с учётом global.json

dotnet build -v detailed         # уровни: q[uiet], m[inimal], n[ormal], d[etailed], diag[nostic]
dotnet build -bl                 # бинарный лог MSBuild для разбора в MSBuild Structured Log Viewer
```

`dotnet build -bl` — главный инструмент, когда сборка ведёт себя необъяснимо: лог содержит все цели, свойства и порядок их вычисления.

> [!warning] Подводные камни
> - **`dotnet run` в проде.** Он собирает проект; на сервере запускают `dotnet MyApp.dll` или готовый исполняемый файл.
> - **Забытый `--no-restore`/`--no-build` в CI.** Каждая команда заново восстанавливает и собирает — минуты впустую.
> - **Отсутствие `global.json`.** Разные версии SDK у разработчиков и в CI дают разное поведение сборки.
> - **Глобальные инструменты не в `PATH`.** `dotnet-ef: command not found` после успешной установки.
> - **`dotnet publish` без `-c Release`.** По умолчанию Debug: медленнее и с включёнными проверками.
> - **Аргументы приложения без `--`.** `dotnet run --verbose` попадёт в CLI, а не в программу.
> - **Publish в непустую папку.** Остаются файлы прошлого деплоя, что даёт конфликты версий сборок.
> - **`dotnet test` без кода возврата в скрипте.** Шаг CI «зелёный» при упавших тестах, если результат не проверяется.

> [!example] Как делают в бою
> Типовой конвейер CI состоит из четырёх шагов, и все они на CLI: `dotnet restore --locked-mode`, `dotnet format --verify-no-changes`, `dotnet build -c Release --no-restore` (с предупреждениями-ошибками из `Directory.Build.props`), `dotnet test -c Release --no-build --logger trx`. Порядок важен: дешёвые проверки раньше дорогих.
> В Dockerfile сборка идёт в два этапа: на образе SDK делают `restore` и `publish`, в финальный образ рантайма копируют только результат. Файлы проектов копируют до остального кода, чтобы слой восстановления кешировался ([[Docker: образы для .NET]]).
> Локально ежедневно используются `dotnet watch` (перезапуск при правках), `dotnet test --filter` (один тест вместо всех), `dotnet ef migrations add` и `dotnet user-secrets`. Инструменты держат локальными через манифест — тогда у всей команды и у CI одинаковые версии.
> На проде из CLI остаётся только диагностика: `dotnet-counters` для быстрой оценки состояния и `dotnet-dump`/`dotnet-gcdump` для разбора памяти ([[Диагностика в проде: дампы и dotnet-tools]]).

---

## Вопросы с собеседований

> [!question]- Чем `dotnet run` отличается от `dotnet MyApp.dll`?
> `dotnet run` — команда SDK для разработки: она восстанавливает зависимости (если нужно), собирает проект и запускает результат, ориентируясь на файл проекта и `launchSettings.json`. Она требует установленного SDK и предполагает наличие исходников. `dotnet MyApp.dll` — запуск уже собранного приложения хостом рантайма: SDK не нужен, достаточно установленного рантайма, никакой сборки не происходит. В продакшене используют второй вариант или самодостаточный исполняемый файл, созданный при публикации; `dotnet run` там неуместен, потому что тянет за собой SDK, увеличивает время старта и создаёт риск непреднамеренной пересборки. Отдельная деталь: аргументы приложению при `dotnet run` передаются после `--`, иначе их разберёт сам CLI.

> [!question]- Зачем нужен `global.json`?
> Он закрепляет версию .NET SDK для репозитория. Без него используется самая свежая установленная версия, и это приводит к расхождениям: разработчик с более новым SDK может получить другое поведение анализаторов, другие умолчания шаблонов или новые предупреждения, которых нет в CI. `global.json` в корне решения фиксирует нужную версию, а свойство `rollForward` определяет, насколько свободно можно подниматься — от `patch` (только исправления) до `disable` (строго указанная версия). Практика: закреплять мажорно-минорную версию и разрешать подъём по патчам, а обновлять её осознанно, вместе с обновлением образов CI. Отдельно стоит помнить, что `global.json` управляет версией SDK, а не рантайма — целевая платформа задаётся в файле проекта через TFM.

> [!question]- Какие шаги CLI обычно составляют шаг сборки в CI?
> Восстановление с фиксацией графа (`dotnet restore --locked-mode`), проверка форматирования (`dotnet format --verify-no-changes`), сборка в Release без повторного восстановления (`dotnet build -c Release --no-restore`) и тесты без повторной сборки (`dotnet test -c Release --no-build`) с формированием отчёта. Флаги `--no-restore` и `--no-build` важны не только для скорости: они гарантируют, что тестируется ровно то, что собрано на предыдущем шаге. Предупреждения-ошибки задают не флагом, а свойством `TreatWarningsAsErrors` в `Directory.Build.props`, чтобы правило действовало и локально. Дополнительно добавляют проверку уязвимостей зависимостей и публикацию артефактов. Каждый шаг опирается на код возврата: ненулевой результат любой из команд валит конвейер.

---

## Задачи

### Задача 1. Собрать конвейер

Опиши последовательность команд для CI: проверка стиля, сборка, тесты с покрытием, публикация артефакта.

> [!success]- Решение
> ```bash
> dotnet restore --locked-mode
> dotnet format --verify-no-changes --severity warn
> dotnet build -c Release --no-restore
> dotnet test -c Release --no-build --logger "trx;LogFileName=tests.trx" \
>              --collect:"XPlat Code Coverage"
> dotnet publish src/Api/Api.csproj -c Release --no-build -o artifacts
> ```
> Логика порядка: сначала самое дешёвое и быстро падающее (восстановление и стиль), потом сборка, потом тесты, потом публикация. `--locked-mode` гарантирует, что версии зависимостей те же, что зафиксированы в `packages.lock.json`. `--no-restore` и `--no-build` исключают повторную работу и гарантируют, что тестируются и публикуются именно собранные артефакты, а не пересобранные заново.
> Один нюанс: `dotnet publish --no-build` работает, только если сборка выполнялась с той же конфигурацией и TFM; при мультитаргетинге понадобится указать `-f`. Предупреждения как ошибки задаются в `Directory.Build.props`, а не флагом, чтобы правило работало и на машине разработчика.

### Задача 2. Разобраться с окружением

Коллега жалуется: `dotnet build` падает у него с ошибкой, которой нет у остальных. С чего начать?

> [!success]- Решение
> ```bash
> dotnet --info          # версии SDK и рантаймов, архитектура, RID
> dotnet --list-sdks     # какие SDK вообще установлены
> cat global.json        # какая версия закреплена для репозитория
> ```
> Самая частая причина — другая версия SDK: без `global.json` берётся самая новая установленная, а новые версии добавляют анализаторы и предупреждения; при `TreatWarningsAsErrors` это ломает сборку. Решение — закрепить версию в `global.json` и, если нужно, зафиксировать `AnalysisLevel` в свойствах проекта.
> Вторая по частоте причина — архитектура: на Apple Silicon случайно установленный x64-SDK работает через эмуляцию и может вести себя иначе; `dotnet --info` покажет `RID: osx-x64` вместо `osx-arm64`.
> Дальше по убыванию: старый кеш пакетов (`dotnet nuget locals all --clear`), остатки `bin`/`obj` от другой версии (удалить), различия в регистре имён файлов (актуально при сравнении с Linux-контейнером). Если ничего не помогает — `dotnet build -bl` и разбор бинарного лога: он показывает, какие цели и с какими свойствами выполнялись.

---

## Итог

- `dotnet` покрывает весь цикл: создание, восстановление, сборку, запуск, тесты, публикацию, упаковку, инструменты.
- `global.json` закрепляет версию SDK для репозитория — иначе поведение сборки различается между машинами.
- В CI используют `--locked-mode`, `--no-restore`, `--no-build`: это и скорость, и гарантия, что тестируется собранное.
- Аргументы приложению передаются после `--`; свойства MSBuild — через `-p:`.
- Публикация: framework-dependent по умолчанию, self-contained и ReadyToRun по флагам, Native AOT свойством.
- Инструменты лучше локальные (манифест в репозитории), чем глобальные.
- `dotnet format`, `dotnet user-secrets`, `dotnet dev-certs`, `dotnet watch` — ежедневный набор разработчика.
- При необъяснимом поведении сборки: `dotnet --info` и бинарный лог `-bl`.

## Связанное

- [[Файл проекта .csproj изнутри]] — что означают свойства, передаваемые через `-p:`
- [[NuGet: пакеты, версии, транзитивные зависимости]] · [[Target Framework Moniker и мультитаргетинг]]
- [[JIT, Tiered Compilation, ReadyToRun, Native AOT]] — варианты публикации
- [[Командная строка Linux для .NET-разработчика]] · [[Настройка окружения]]
- [[CI/CD: GitHub Actions]] · [[Docker: образы для .NET]]
- [[Диагностика в проде: дампы и dotnet-tools]] · [[Конфигурация и секреты]]
- [[Шпаргалка — dotnet CLI]]
