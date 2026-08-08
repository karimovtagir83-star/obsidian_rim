---
tags: [раздел-04, tfm, мультитаргетинг, netstandard, msbuild, middle, собес]
aliases: [Target Framework Moniker, TFM, Multitargeting, .NET Standard, Мультитаргетинг]
---

# Target Framework Moniker и мультитаргетинг

> [!abstract] Коротко
> TFM (Target Framework Moniker) — короткая строка вида `net10.0`, которая говорит компилятору и NuGet, под какой набор API и какой рантайм собирается проект. Он определяет доступную часть BCL, версию C# по умолчанию, правила выбора папок внутри NuGet-пакетов и то, кто вообще сможет вашу сборку потребить. Мультитаргетинг — сборка одного проекта сразу под несколько TFM, чтобы библиотека работала и в .NET 10, и в древнем .NET Framework. TFM отвечает на вопрос «какие API», RID — на вопрос «под какую ОС и процессор».

## Зачем это нужно

Три ситуации, где непонимание TFM стоит рабочего дня:

1. Вы ставите NuGet-пакет, и он падает в рантайме с `MissingMethodException`, хотя компилировался. Причина — `AssetTargetFallback` подсунул сборку от другого фреймворка.
2. Вы написали библиотеку на `net10.0`, а коллега из соседней команды не может её подключить: у него `net472`, и NuGet честно говорит «несовместимо».
3. Вы добавили `<TargetFrameworks>net10.0;netstandard2.0</TargetFrameworks>`, и половина кода перестала компилироваться, потому что в `netstandard2.0` нет ни `Span<T>`, ни `record`, ни `required`.

TFM — это контракт «какой поверхностью API я пользуюсь». Всё остальное в системе сборки (см. [[Файл проекта .csproj изнутри]]) и в пакетном менеджере (см. [[NuGet: пакеты, версии, транзитивные зависимости]]) пляшет от него.

## Теория

### Анатомия современного TFM

Формат для .NET 5 и новее:

```
net<версия>[-<платформа>[<версия платформы>]]
 │      │        │              │
 │      │        │              └── 10.0.19041.0 — минимальный/целевой API уровня ОС
 │      │        └───────────────── windows | android | ios | macos | maccatalyst | tvos | browser
 │      └────────────────────────── 10.0, 9.0, 8.0 ...
 └───────────────────────────────── идентификатор семейства
```

Примеры и что они значат:

| TFM | Что означает | Типичное применение |
|---|---|---|
| `net10.0` | Только кроссплатформенный API. Компилятор не даст вызвать `Registry` или WinForms | Библиотеки, ASP.NET Core, консоль, сервисы |
| `net10.0-windows` | Плюс Windows-специфика. Версия платформы по умолчанию `7.0` | Библиотека, дергающая Win32/COM |
| `net10.0-windows10.0.19041.0` | Плюс WinRT/Windows App SDK API уровня Windows 10 2004 | WinUI, WinForms/WPF с современным API |
| `net10.0-android` | Android BCL + биндинги Java. Требует workload `android` | MAUI/Android-приложения |
| `net10.0-ios` | iOS BCL + биндинги Objective-C. Обязателен Native AOT-подобный режим | iOS-приложения |
| `net10.0-browser` | WASM-специфичный API (`JSImport`/`JSExport`) | Библиотеки для Blazor WebAssembly и wasm-хостов |

MSBuild разбирает TFM на свойства, которые видно в билде:

| Свойство | Значение для `net10.0-windows10.0.19041.0` |
|---|---|
| `$(TargetFramework)` | `net10.0-windows10.0.19041.0` |
| `$(TargetFrameworkIdentifier)` | `.NETCoreApp` |
| `$(TargetFrameworkVersion)` | `v10.0` |
| `$(TargetPlatformIdentifier)` | `windows` |
| `$(TargetPlatformVersion)` | `10.0.19041.0` |
| `$(SupportedOSPlatformVersion)` | по умолчанию равно `TargetPlatformVersion` |

Обратите внимание на пару `TargetPlatformVersion` / `SupportedOSPlatformVersion`. Первая — «против каких API я компилируюсь» (потолок). Вторая — «на какой минимальной версии ОС я обещаю работать» (пол). Их можно развести:

```xml
<PropertyGroup>
  <TargetFramework>net10.0-windows10.0.19041.0</TargetFramework>
  <!-- компилируемся против API 2004, но запускаться готовы на 1809 -->
  <SupportedOSPlatformVersion>10.0.17763.0</SupportedOSPlatformVersion>
</PropertyGroup>
```

После этого анализатор совместимости платформ **CA1416** начинает ругаться на каждый вызов API, помеченного `[SupportedOSPlatform("windows10.0.19041.0")]`, если вы не проверили версию через `OperatingSystem.IsWindowsVersionAtLeast(10, 0, 19041)`. Это ровно та защита, ради которой платформенный суффикс и придумали.

### Атрибуты платформенной совместимости

Живут в `System.Runtime.Versioning` и являются обычными атрибутами (см. [[Атрибуты]]), которые читает анализатор, а не рантайм:

| Атрибут | Смысл |
|---|---|
| `[SupportedOSPlatform("windows")]` | API есть только на этой платформе |
| `[SupportedOSPlatform("ios15.0")]` | API появился с этой версии платформы |
| `[UnsupportedOSPlatform("browser")]` | API заведомо не работает здесь (типично для WASM: потоки, файловая система) |
| `[ObsoletedOSPlatform("macos13.0")]` | API устарел с этой версии ОС |

Правило проверки, которое понимает CA1416: `OperatingSystem.IsWindows()`, `OperatingSystem.IsWindowsVersionAtLeast(...)`, `OperatingSystem.IsAndroidVersionAtLeast(...)`, `RuntimeInformation.IsOSPlatform(...)` в некоторых случаях. Обычный `if (Environment.OSVersion.Platform == ...)` анализатор не распознаёт — это классическая причина «я же проверил, а он всё равно ругается».

### Старые TFM: что встретится в легаси

| TFM | Что это | Статус |
|---|---|---|
| `net48`, `net472`, `net462` | .NET Framework (только Windows, ставится в систему) | Жив в энтерпрайзе, новых фич не будет |
| `netcoreapp3.1`, `netcoreapp2.1` | .NET Core до унификации имён | #устарело, EOL |
| `netstandard2.0` | Спецификация API, не рантайм | Жив как «общий знаменатель» |
| `netstandard2.1` | То же, но шире | Почти бесполезен, см. ниже |
| `net5.0` … `net9.0` | Унифицированный .NET | `net9.0` EOL с 12.05.2026 — не начинать на нём новое |
| `portable-net45+win8+wpa81` | PCL (Portable Class Library) — профили из пересечений | #устарело, мертво окончательно |
| `sl5`, `wp8`, `xamarin.ios` | Silverlight, Windows Phone, старые Xamarin-TFM | #устарело |

PCL — поучительная страница истории. Идея была: перечисляешь платформы, а инструмент вычисляет пересечение их API. Проблема — комбинаторный взрыв: каждая новая платформа множила число «профилей», и авторы библиотек не понимали, какой профиль выбрать. .NET Standard заменил пересечение на **фиксированный список версий**: одна ось вместо решётки.

### .NET Standard: зачем был и почему умер

.NET Standard — это не рантайм. Это набор reference-сборок с сигнатурами методов и телами `throw null;`. Вы компилируетесь против контракта, а в рантайме `System.Runtime.dll` через type-forwarding перенаправляет типы в реальную реализацию конкретной платформы (подробнее про механику форвардинга — в [[Сборки, метаданные и IL]]).

Кто что поддерживает:

| .NET Standard | .NET Framework | .NET Core / .NET | Mono / Unity |
|---|---|---|---|
| 1.x | 4.5+ (частично) | всё | да |
| **2.0** | **4.6.1+ (реально стабильно с 4.7.2)** | Core 2.0+, .NET 5+ | Mono 5.4+, Unity 2018.1+ |
| **2.1** | **никогда** | Core 3.0+, .NET 5+ | Mono 6.4+, Unity 2021+ |

Почему умер: .NET 5 объединил Core, Mono и Xamarin в одну кодовую базу с одним BCL. Абстракция «спецификация поверх N реализаций» перестала быть нужна — реализация стала одна. Microsoft явно зафиксировала: `netstandard2.1` — последняя версия, дальнейшего развития не будет.

Почему `netstandard2.1` почти бесполезен: единственная причина брать .NET Standard вместо `net10.0` — потребители на .NET Framework. Но `netstandard2.1` .NET Framework не поддерживает **никогда** и **принципиально** (там нет `Span<T>` как ref struct на уровне рантайма, нет default interface methods). Значит `netstandard2.1` покрывает ровно тех же потребителей, что и `net8.0`/`net10.0`, но с урезанным API. Выбор всегда бинарный: либо `netstandard2.0` (нужен .NET Framework/Unity), либо современный `net10.0`.

Когда `netstandard2.0` ещё оправдан в 2026:

- Библиотека-SDK, которую вы отдаёте наружу неизвестным клиентам (клиент банка, партнёрская интеграция).
- Потребители на .NET Framework 4.6.2+ — куча внутренних сервисов и плагинов к Office/AutoCAD/1С-коннекторам.
- Unity: скриптовый рантайм там по-прежнему Mono-совместимый и понимает `netstandard2.0`/`netstandard2.1`.
- Анализаторы и Source Generators — Roslyn грузит их в свой процесс, и требование строгое: `netstandard2.0` (см. [[Source Generators]]).

Во всех остальных случаях `netstandard2.0` — самострел: вы теряете `Span<T>`, `IAsyncEnumerable<T>`, `record`, `required`, `init`, nullable-аннотации BCL и половину перформанс-API.

### Мультитаргетинг

Множественное число в имени свойства — это всё, что нужно:

```xml
<PropertyGroup>
  <!-- ВАЖНО: TargetFrameworks, не TargetFramework -->
  <TargetFrameworks>net10.0;netstandard2.0</TargetFrameworks>
</PropertyGroup>
```

MSBuild запускает сборку N раз (по разу на TFM), каждый со своим `obj/<tfm>/` и `bin/<config>/<tfm>/`. `dotnet pack` кладёт результат в `lib/net10.0/` и `lib/netstandard2.0/` внутри одного nupkg.

Автоматические символы препроцессора для каждого прохода:

| TFM | Определены символы |
|---|---|
| `net10.0` | `NET`, `NET10_0`, `NET10_0_OR_GREATER`, `NET5_0_OR_GREATER` … `NET9_0_OR_GREATER`, `NETCOREAPP`, `NETCOREAPP1_0_OR_GREATER` … `NETCOREAPP3_1_OR_GREATER` |
| `net10.0-windows` | всё выше + `WINDOWS`, `NET10_0_WINDOWS`, `NET10_0_WINDOWS_OR_GREATER` |
| `netstandard2.0` | `NETSTANDARD`, `NETSTANDARD2_0`, `NETSTANDARD2_0_OR_GREATER`, `NETSTANDARD1_0_OR_GREATER` … |
| `net48` | `NETFRAMEWORK`, `NET48`, `NET48_OR_GREATER`, `NET20_OR_GREATER` … |

Практическое правило: пишите `#if NET10_0_OR_GREATER`, а не `#if NET10_0`. Первое переживёт апгрейд на `net11.0`, второе тихо отвалится в ветку `else`.

### Совместимость: кто кого может съесть

| Проект (TFM) | Может потребить сборку/пакет для |
|---|---|
| `net10.0` | `net10.0` … `net5.0`, `netcoreapp*`, `netstandard2.1`, `netstandard2.0`, `netstandard1.x` |
| `net10.0-windows` | всё, что `net10.0`, плюс `net10.0-windows` с версией платформы не выше своей |
| `netstandard2.0` | только `netstandard2.0` и `netstandard1.x` |
| `net48` | `net4x` (<= своей), `netstandard2.0`, `netstandard1.x`. **Не** `netstandard2.1`, **не** `net5.0+` |
| `netcoreapp3.1` | `netcoreapp3.1` и ниже, `netstandard2.1` и ниже |

Стрелка совместимости однонаправленная: **более широкий и более новый TFM потребляет более узкий и более старый**. `net10.0` съест `netstandard2.0`-пакет, потому что весь контракт .NET Standard 2.0 реализован в .NET 10. Обратно — нет: библиотека под `netstandard2.0` не имеет права зависеть от типов, которых нет в её контракте.

`AssetTargetFallback` — это костыль совместимости. По умолчанию для .NET Core/.NET SDK туда прописан `net461` (исторически) — то есть, если пакет не имеет ни одной подходящей папки, restore берёт `.NET Framework`-сборку и печатает:

```
warning NU1701: Package 'Foo 1.0.0' was restored using '.NETFramework,Version=v4.6.1'
instead of the project target framework 'net10.0'. This package may not be fully
compatible with your project.
```

NU1701 — красный флаг, а не шум. Restore буквально говорит: «я подсунул сборку от другого фреймворка, там могут быть P/Invoke в `user32.dll`, `AppDomain.CreateDomain`, `System.Configuration` и прочее, чего в .NET 10 нет». Компиляция пройдёт, а упадёт в проде на первой строчке. Правильные реакции: найти современную версию пакета, найти замену, либо (осознанно, с тестом) оставить и задокументировать. Заглушать `<NoWarn>NU1701</NoWarn>` без проверки — прямая дорога к `PlatformNotSupportedException`.

### LangVersion живёт своей жизнью

Версия языка C# по умолчанию выводится из TFM:

| TFM | C# по умолчанию |
|---|---|
| `net10.0` | 14 |
| `net9.0` | 13 |
| `net8.0` | 12 |
| `netstandard2.1`, `netcoreapp3.1` | 8.0 |
| `netstandard2.0`, `net48` | 7.3 |

Можно поднять принудительно: `<LangVersion>latest</LangVersion>` или `<LangVersion>14.0</LangVersion>`. Но фичи C# делятся на три категории:

1. **Чистый синтаксический сахар** — работает везде. Например, `switch`-выражения, target-typed `new`, интерполяция, локальные функции, `nameof(List<>)`.
2. **Нужен тип из BCL, которого нет** — компилятор скажет «predefined type is not defined». Лечится полифилом: свой `internal static class IsExternalInit` включает `init`-аксессоры и `record` на `netstandard2.0`; `RequiredMemberAttribute` + `CompilerFeatureRequiredAttribute` + `SetsRequiredMembersAttribute` включают `required`.
3. **Нужна поддержка рантайма** — не лечится ничем. Default interface methods, `ref struct`-семантика `Span<T>`, статические абстрактные члены интерфейсов, `ref` поля. На .NET Framework это не заработает никогда.

Именно поэтому «`latest` на `netstandard2.0` работает частично»: категория 1 полностью, категория 2 через полифилы, категория 3 никак.

### Полифилы

| Инструмент | Что даёт |
|---|---|
| **PolySharp** | Source generator, добавляет десятки compiler-attribute'ов (`IsExternalInit`, `RequiredMember`, `CallerArgumentExpression`, nullable-атрибуты) без единой строчки вашего кода. Подключается с `PrivateAssets="all"` |
| `System.Memory` | `Span<T>`, `ReadOnlySpan<T>`, `Memory<T>` для `netstandard2.0` — но это **медленная** эмуляция без поддержки JIT |
| `Microsoft.Bcl.AsyncInterfaces` | `IAsyncEnumerable<T>`, `IAsyncDisposable` |
| `Microsoft.Bcl.HashCode` | `System.HashCode` |
| `Microsoft.Bcl.TimeProvider` | `TimeProvider` для тестируемого времени |
| Ручной `IsExternalInit` | Минимальный вариант, если не хочется зависимости |

### RID: другая ось координат

RID (Runtime Identifier) отвечает не на «какие API», а на «под какую ОС и архитектуру раскладывать нативные бинарники».

| | TFM | RID |
|---|---|---|
| Отвечает на вопрос | какие API доступны | под какую ОС/CPU публикуемся |
| Пример | `net10.0` | `linux-x64` |
| Влияет на | компиляцию, выбор `lib/` в пакете | выбор `runtimes/{rid}/native`, self-contained publish |
| Задаётся | `TargetFramework(s)` | `RuntimeIdentifier(s)` или `-r` у `dotnet publish` |
| Обязателен | всегда | только для self-contained / AOT / нативных зависимостей |

Формат RID: `<os>[.<version>]-<arch>[-<extra>]`. Практический набор: `win-x64`, `win-arm64`, `linux-x64`, `linux-arm64`, `linux-musl-x64` (Alpine), `osx-x64`, `osx-arm64`, `browser-wasm`, `android-arm64`, `ios-arm64`.

```xml
<PropertyGroup>
  <!-- одиночный: публикуем только сюда -->
  <RuntimeIdentifier>linux-x64</RuntimeIdentifier>
</PropertyGroup>

<PropertyGroup>
  <!-- множественный: restore тянет нативные ассеты под все, publish выбирает -r -->
  <RuntimeIdentifiers>linux-x64;linux-arm64;win-x64</RuntimeIdentifiers>
</PropertyGroup>
```

**RID graph** — историческая структура: огромный `runtime.json` с деревом наследования (`ubuntu.20.04-x64` → `ubuntu-x64` → `linux-x64` → `linux` → `any`). NuGet шёл по этому графу вверх, пока не находил подходящую папку `runtimes/`. Проблема: граф надо было обновлять при выходе каждого дистрибутива, а пакеты со специфичными RID устаревали мгновенно.

Начиная с .NET 8 граф по умолчанию не используется — работает плоский набор портабельных RID (`linux-x64`, а не `ubuntu.22.04-x64`). Вернуть старое поведение можно свойством:

```xml
<PropertyGroup>
  <!-- нужно, только если старый пакет публикует ассеты под дистрибутив-специфичный RID -->
  <UseRidGraph>true</UseRidGraph>
</PropertyGroup>
```

## Код

Полноценный мультитаргет-проект библиотеки:

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFrameworks>net10.0;netstandard2.0</TargetFrameworks>
    <LangVersion>latest</LangVersion>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <!-- явный запрет тихого фолбэка на .NET Framework -->
    <AssetTargetFallback></AssetTargetFallback>
  </PropertyGroup>

  <!-- Полифилы нужны ТОЛЬКО старому таргету -->
  <ItemGroup Condition="'$(TargetFramework)' == 'netstandard2.0'">
    <PackageReference Include="PolySharp" Version="1.15.0" PrivateAssets="all" />
    <PackageReference Include="System.Memory" Version="4.6.0" />
    <PackageReference Include="Microsoft.Bcl.AsyncInterfaces" Version="10.0.0" />
  </ItemGroup>

  <!-- Файл с быстрой реализацией компилируется только в современном таргете -->
  <ItemGroup Condition="'$(TargetFramework)' != 'net10.0'">
    <Compile Remove="Fast/VectorizedSearch.cs" />
  </ItemGroup>

</Project>
```

Условная компиляция внутри одного файла:

```csharp
using System;

namespace Alif.Text;

public static class Hex
{
    // Публичная сигнатура одинакова во всех таргетах — это важно для потребителей
    public static string Encode(byte[] data)
    {
#if NET10_0_OR_GREATER
        // Convert.ToHexStringLower появился в .NET 9, доступен в .NET 10
        return Convert.ToHexStringLower(data);
#else
        // Медленный, но совместимый путь для netstandard2.0
        var sb = new System.Text.StringBuilder(data.Length * 2);
        foreach (byte b in data)
        {
            sb.Append(b.ToString("x2"));
        }
        return sb.ToString();
#endif
    }
}
```

Разведение целых файлов вместо `#if` — обычно чище:

```
src/Alif.Text/
├── Hex.cs                    // общая часть, публичный контракт
├── Hex.Modern.cs             // partial, компилируется только для net10.0
└── Hex.Legacy.cs             // partial, только для netstandard2.0
```

```xml
<ItemGroup>
  <Compile Remove="Hex.Modern.cs" />
  <Compile Remove="Hex.Legacy.cs" />
</ItemGroup>
<ItemGroup Condition="'$(TargetFramework)' == 'net10.0'">
  <Compile Include="Hex.Modern.cs" />
</ItemGroup>
<ItemGroup Condition="'$(TargetFramework)' == 'netstandard2.0'">
  <Compile Include="Hex.Legacy.cs" />
</ItemGroup>
```

Платформенный код с корректной защитой от CA1416:

```csharp
using System.Runtime.Versioning;

public static class MachineName
{
    public static string Describe()
    {
        // Анализатор понимает именно OperatingSystem.Is*
        if (OperatingSystem.IsWindowsVersionAtLeast(10, 0, 19041))
        {
            return ReadFromRegistry();
        }

        if (OperatingSystem.IsLinux())
        {
            return System.IO.File.ReadAllText("/etc/hostname").Trim();
        }

        return Environment.MachineName;
    }

    // Метод помечен — вызывать его без проверки нельзя, CA1416 не пропустит
    [SupportedOSPlatform("windows10.0.19041.0")]
    private static string ReadFromRegistry()
    {
        using var key = Microsoft.Win32.Registry.LocalMachine
            .OpenSubKey(@"SYSTEM\CurrentControlSet\Control\ComputerName\ComputerName");
        return key?.GetValue("ComputerName") as string ?? Environment.MachineName;
    }
}
```

Полифил `init`-аксессоров вручную (то, что PolySharp генерирует автоматически):

```csharp
#if !NET5_0_OR_GREATER
namespace System.Runtime.CompilerServices
{
    // Компилятору достаточно самого факта существования типа с этим именем.
    // Без него на netstandard2.0 не работают init-свойства и record.
    internal static class IsExternalInit { }
}
#endif
```

> [!warning] Подводные камни
> **`TargetFramework` вместо `TargetFrameworks`.** Опечатка на одну букву. MSBuild не ругается — он просто соберёт под один таргет, а вы будете гадать, почему в nupkg только одна папка `lib/`. Проверка: `dotnet build -v:n` покажет, сколько раз выполнился `CoreCompile`.
>
> **Разная публичная поверхность в разных таргетах.** Если под `net10.0` метод принимает `ReadOnlySpan<char>`, а под `netstandard2.0` — `string`, то потребитель, который сам мультитаргетится, получит невозможный к написанию код. Держите публичный API идентичным, различия прячьте в реализацию.
>
> **`System.Memory` на `netstandard2.0` — не тот `Span<T>`.** Там это обычная структура с полем-массивом и смещением, без поддержки JIT и без `ref`-полей. Ноль преимуществ по производительности, иногда медленнее массива. Не пишите «оптимизацию через Span» в коде, который реально исполняется под .NET Framework (подробнее — [[Span, ReadOnlySpan и Memory]]).
>
> **`NU1701` заглушённый в `Directory.Build.props`.** Кто-то однажды добавил `<NoWarn>$(NoWarn);NU1701</NoWarn>`, чтобы билд был зелёным. Через полгода никто не знает, какие пакеты в решении на самом деле от .NET Framework. Заглушать надо точечно, на конкретном `PackageReference` через `NoWarn="NU1701"`.
>
> **Платформенный TFM без нужды.** `net10.0-windows` на библиотеке, которая просто читает файлы, отрезает Linux-потребителей и Docker-сборки. Ставьте платформенный суффикс, только если реально дёргаете платформенный API.
>
> **`net9.0` в шаблонах.** .NET 9 закончил поддержку 12.05.2026. Если `global.json` или CI-образ всё ещё указывает на 9-ку, вы собираете на непатченном рантайме.

> [!example] Как делают в бою
> **Продуктовые сервисы и приложения** — всегда один TFM, всегда LTS: `net10.0`. Мультитаргетинг приложения не нужен никогда: у приложения ровно один рантайм.
>
> **Внутренние библиотеки в монорепозитории** — тоже один TFM `net10.0`. Мультитаргетинг стоит времени сборки (кратно числу таргетов) и когнитивной нагрузки; если все потребители внутри и все на .NET 10, платить не за что.
>
> **Публичные/партнёрские библиотеки** — `net10.0;netstandard2.0`. Ровно два таргета: современный быстрый и совместимый медленный. Не `net10.0;net8.0;netstandard2.1;netstandard2.0;net472` — каждый лишний таргет это отдельная матрица тестов.
>
> **Анализаторы и Source Generators** — строго `netstandard2.0` и отдельный проект. Rider/VS/`csc` грузят их в свой процесс, у которого свой рантайм.
>
> TFM выносят в `Directory.Build.props` на весь репозиторий:
> ```xml
> <PropertyGroup>
>   <TargetFramework>net10.0</TargetFramework>
>   <LangVersion>14.0</LangVersion>
>   <Nullable>enable</Nullable>
> </PropertyGroup>
> ```
> Тогда апгрейд на .NET 11 — правка одной строки, а не сорока `.csproj`. Проекты, которым нужно иначе, переопределяют локально: последнее присваивание свойства побеждает.
>
> В CI на мультитаргет-библиотеке гоняют тесты **под каждый TFM** отдельно. Иначе `netstandard2.0`-ветка кода не исполняется никогда и тихо гниёт.

## Вопросы с собеседований

> [!question]- Чем TFM отличается от RID? Можно ли обойтись одним?
> TFM (`net10.0`) отвечает на вопрос «какой набор API мне доступен на этапе компиляции» — он определяет reference-сборки, версию C# по умолчанию и то, какую папку `lib/` NuGet возьмёт из пакета. RID (`linux-x64`) отвечает на вопрос «под какую ОС и архитектуру я публикуюсь» — он определяет, какие нативные библиотеки из `runtimes/{rid}/native` попадут в выход и какой рантайм положить при self-contained. TFM обязателен всегда, RID — только при self-contained-публикации, Native AOT или наличии нативных зависимостей. Обойтись одним нельзя: `net10.0` без RID соберёт framework-dependent приложение, которое запустится на любой ОС с установленным рантаймом, а RID без TFM просто не имеет смысла — компилятору нечего дать.

> [!question]- Почему проект на `net48` не может подключить библиотеку под `netstandard2.1`?
> .NET Standard — это спецификация, реализовать которую должен рантайм. В 2.1 попали вещи, требующие изменений именно в рантайме: `Span<T>` как настоящий `ref struct` с verification-правилами, default interface methods (нужна поддержка в загрузчике типов и в диспетчеризации интерфейсов), `IAsyncEnumerable<T>`. .NET Framework заморожен как продукт — Microsoft явно заявила, что новых рантайм-фич там не будет, поэтому 2.1 туда не портирован и не будет. Практическое следствие: если среди ваших потребителей есть .NET Framework, единственный вариант совместимости — `netstandard2.0`. А если .NET Framework среди потребителей нет, то `netstandard2.1` даёт ровно то же покрытие, что и `net10.0`, но с меньшим API — то есть смысла не имеет.

> [!question]- Что означает предупреждение NU1701 и почему его опасно глушить?
> NU1701 говорит: «для этого пакета не нашлось ассетов под ваш TFM, поэтому по правилу `AssetTargetFallback` я взял сборку, собранную под .NET Framework». Компиляция пройдёт, потому что публичные сигнатуры совпадают. А в рантайме сборка может дёрнуть `AppDomain.CreateDomain`, `System.Configuration.ConfigurationManager`, `System.Drawing` поверх GDI+, P/Invoke в `user32.dll` или Remoting — всего этого в .NET 10 нет, и вы получите `PlatformNotSupportedException` или `TypeLoadException` в самый неудачный момент. Правильно: проверить, есть ли современная версия пакета, поискать замену, а если пакет действительно нужен — прогнать интеграционный тест по всем используемым путям кода и заглушить предупреждение точечно, прямо на этом `PackageReference` через `NoWarn="NU1701"`, с комментарием.

> [!question]- Как заставить C# 14 работать на `netstandard2.0` и где предел?
> Ставите `<LangVersion>latest</LangVersion>` — компилятор Roslyn один и тот же, он умеет разбирать новый синтаксис независимо от TFM. Дальше всё упирается в три категории фич. Чистый сахар (switch-выражения, target-typed `new`, `nameof(List<>)`, коллекционные выражения над массивом) работает сразу. Фичи, которым нужен просто тип-маркер из BCL (`init`, `record`, `required`, `CallerArgumentExpression`), включаются полифилом: достаточно объявить `internal` тип с правильным полным именем, либо подключить PolySharp, который генерирует всё это автоматически. Фичи, которым нужен рантайм (default interface methods, статические абстрактные члены, `ref` поля, настоящий `Span<T>`), не заработают ни при каких настройках — там нужны изменения в загрузчике типов и в JIT.

> [!question]- Что делает суффикс платформы в `net10.0-windows10.0.19041.0` и как связаны TargetPlatformVersion и SupportedOSPlatformVersion?
> Суффикс `-windows` подключает reference-сборки Windows-специфичного API и выставляет `$(TargetPlatformIdentifier)`. Число `10.0.19041.0` — `TargetPlatformVersion`, то есть уровень API, против которого идёт компиляция: это «потолок» того, что вам разрешено вызывать. `SupportedOSPlatformVersion` — «пол», минимальная версия ОС, на которой вы обещаете работать; по умолчанию она равна `TargetPlatformVersion`. Если опустить пол ниже потолка, включается анализатор CA1416: каждый вызов API, помеченного `[SupportedOSPlatform("windows10.0.19041.0")]`, потребует явной проверки через `OperatingSystem.IsWindowsVersionAtLeast(10, 0, 19041)`. Это статическая гарантия, что вы не вызовете отсутствующий на старой ОС API.

> [!question]- Зачем нужен RID graph и почему его отключили в .NET 8?
> RID graph — дерево наследования идентификаторов рантайма (`ubuntu.22.04-x64` наследует `ubuntu-x64`, тот `linux-x64`, тот `linux`, тот `any`). NuGet шёл по цепочке вверх, пока не находил в пакете подходящую папку `runtimes/{rid}/`. Схема была нужна, когда пакеты публиковали дистрибутив-специфичные бинарники. Проблемы: граф был статическим файлом, который надо было обновлять при каждом новом релизе Ubuntu/Fedora/Alpine, размер его рос, а разрешение занимало заметное время restore. С .NET 8 по умолчанию используется плоский набор портабельных RID (`linux-x64`, `linux-musl-x64`, `osx-arm64`), а старое поведение включается свойством `<UseRidGraph>true</UseRidGraph>` — оно нужно только если вы зависите от древнего пакета с дистрибутив-специфичными ассетами.

> [!question]- Проект мультитаргетится на `net10.0;netstandard2.0`. Тесты гоняются только под `net10.0`. В чём риск?
> В том, что вся `netstandard2.0`-ветка кода никогда не исполняется. Полифилы, `#else`-блоки, медленные реализации — всё это компилируется, но не проверяется. Классический сценарий: кто-то правит общий метод, `#if`-ветка под `net10.0` обновлена, `#else`-ветка забыта, и потребитель на .NET Framework получает старое поведение. Решение: тестовый проект тоже мультитаргетится (`<TargetFrameworks>net10.0;net472</TargetFrameworks>`), и `dotnet test` прогоняет обе матрицы. Если .NET Framework в CI недоступен — как минимум прогонять `net10.0` с искусственно включённым `netstandard2.0`-путём через отдельную сборку.

## Задачи

### Задача 1. Разложить пакет по таргетам

Есть nupkg со следующим содержимым:

```
lib/netstandard2.0/Foo.dll
lib/net8.0/Foo.dll
lib/net10.0/Foo.dll
```

Какую сборку получит проект под `net10.0`, `net9.0`, `net8.0`, `netstandard2.1`, `net472`, `netcoreapp3.1`? Что произойдёт, если удалить папку `lib/netstandard2.0`?

> [!success]- Решение
> | Проект | Выбранная папка | Почему |
> |---|---|---|
> | `net10.0` | `lib/net10.0` | точное совпадение, самый близкий |
> | `net9.0` | `lib/net8.0` | `net10.0` новее проекта — нельзя; ближайший подходящий — `net8.0` |
> | `net8.0` | `lib/net8.0` | точное совпадение |
> | `netstandard2.1` | `lib/netstandard2.0` | проект-библиотека под .NET Standard может брать только .NET Standard |
> | `net472` | `lib/netstandard2.0` | .NET Framework 4.7.2 реализует .NET Standard 2.0 |
> | `netcoreapp3.1` | `lib/netstandard2.0` | `net8.0` новее — нельзя; .NET Standard 2.1 в пакете нет |
>
> Если удалить `lib/netstandard2.0`, то `net472` и `netcoreapp3.1` потеряют совместимость. `netcoreapp3.1` получит ошибку NU1202 («package is not compatible»). `net472` при включённом `AssetTargetFallback` может ничего не найти и тоже упасть с NU1202 — фолбэк работает в другую сторону (для .NET Core-проектов, тянущих .NET Framework-пакеты), а не наоборот.
>
> Ключевой принцип выбора: NuGet берёт **наиболее близкий совместимый** TFM, никогда не берёт более новый, чем у проекта.

### Задача 2. Починить мультитаргет-проект

Дан `.csproj`, который не собирается:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net10.0;netstandard2.0</TargetFramework>
    <Nullable>enable</Nullable>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="System.Text.Json" Version="10.0.0" />
  </ItemGroup>
</Project>
```

В коде используются `record`, `init`-свойства и `IAsyncEnumerable<T>`. Найдите все проблемы и напишите исправленный файл.

> [!success]- Решение
> Проблемы:
> 1. `TargetFramework` (единственное число) со списком через `;` — MSBuild воспримет это как одно кривое имя фреймворка. Нужно `TargetFrameworks`.
> 2. `LangVersion` не задан: для `netstandard2.0` по умолчанию C# 7.3, там нет ни `record`, ни `init`.
> 3. `IsExternalInit` отсутствует → `init` и `record` не скомпилируются даже с `LangVersion=latest`.
> 4. `IAsyncEnumerable<T>` отсутствует в `netstandard2.0` → нужен `Microsoft.Bcl.AsyncInterfaces`.
> 5. `System.Text.Json` подтянут для обоих таргетов, хотя в `net10.0` он часть рантайма — лишняя транзитивная зависимость (см. framework pruning в [[NuGet: пакеты, версии, транзитивные зависимости]]).
>
> ```xml
> <Project Sdk="Microsoft.NET.Sdk">
>
>   <PropertyGroup>
>     <TargetFrameworks>net10.0;netstandard2.0</TargetFrameworks>
>     <LangVersion>latest</LangVersion>
>     <Nullable>enable</Nullable>
>   </PropertyGroup>
>
>   <ItemGroup Condition="'$(TargetFramework)' == 'netstandard2.0'">
>     <!-- IsExternalInit, RequiredMember, nullable-атрибуты и прочее -->
>     <PackageReference Include="PolySharp" Version="1.15.0" PrivateAssets="all" />
>     <PackageReference Include="Microsoft.Bcl.AsyncInterfaces" Version="10.0.0" />
>     <PackageReference Include="System.Text.Json" Version="10.0.0" />
>   </ItemGroup>
>
> </Project>
> ```
>
> `Nullable=enable` на `netstandard2.0` работает: сами аннотации `?` — это метаданные, а атрибуты вроде `NotNullWhen` даёт PolySharp. Но BCL там без аннотаций, поэтому предупреждения будут отличаться между таргетами — это нормально (см. [[Nullable reference types]]).

### Задача 3. Найти платформенную мину

Библиотека собрана под `net10.0`, работает в разработке на Windows, падает в Docker на Linux:

```csharp
public static string GetConfigPath()
{
    var root = Environment.GetFolderPath(Environment.SpecialFolder.CommonApplicationData);
    return root + "\\myapp\\config.json";
}
```

Что не так и как поймать это на этапе компиляции?

> [!success]- Решение
> Две ошибки. Явная — жёстко зашитый разделитель `\`, на Linux путь превратится в один файл с обратными слэшами в имени. Неявная — `SpecialFolder.CommonApplicationData` на Linux возвращает `/usr/share`, куда процесс в контейнере обычно не имеет права писать, а иногда возвращает пустую строку.
>
> ```csharp
> public static string GetConfigPath()
> {
>     var root = Environment.GetFolderPath(Environment.SpecialFolder.CommonApplicationData);
>
>     if (string.IsNullOrEmpty(root))
>     {
>         // На части Unix-конфигураций SpecialFolder возвращает пустую строку
>         root = OperatingSystem.IsWindows() ? @"C:\ProgramData" : "/var/lib";
>     }
>
>     // Path.Combine сам подставит '/' или '\'
>     return Path.Combine(root, "myapp", "config.json");
> }
> ```
>
> Как поймать компилятором: анализатор CA1416 здесь не поможет — `SpecialFolder` не помечен платформенными атрибутами, потому что формально работает везде. Помогают:
> - `dotnet build -p:EnableNETAnalyzers=true -p:AnalysisLevel=latest-all` — включит CA1307/CA1310 и родственные правила по строкам и путям;
> - правило CI: сборка и тесты обязаны прогоняться на Linux-раннере, а не только на dev-машине;
> - код-ревью-эвристика — литералы `"\\"`, `":"` в путях и `"C:\"` не должны попадать в кроссплатформенный код.

## Итог

- TFM определяет **доступный API и версию C# по умолчанию**; RID определяет **ОС и архитектуру публикации**. Это ортогональные оси.
- Платформенный суффикс (`-windows`, `-android`) включает платформенные reference-сборки и анализатор CA1416; пара `TargetPlatformVersion` / `SupportedOSPlatformVersion` задаёт потолок и пол.
- .NET Standard умер вместе с унификацией в .NET 5. `netstandard2.0` остаётся живым только как мост к .NET Framework 4.6.2+, Unity и Roslyn-анализаторам. `netstandard2.1` практического смысла не имеет.
- Мультитаргетинг — это `<TargetFrameworks>`, автоматические символы `NET10_0_OR_GREATER` / `NETSTANDARD2_0`, условные `ItemGroup` и полифилы. Каждый лишний таргет умножает время сборки и матрицу тестов.
- Совместимость односторонняя: новый и широкий TFM потребляет старый и узкий. Обратное невозможно. `AssetTargetFallback` и NU1701 — костыль, а не решение.
- Для приложений — один TFM, всегда LTS (`net10.0`). Для публичных библиотек — максимум два.

## Связанное

- [[04 — Типы, память, CLR (обзор раздела)]]
- [[Устройство .NET: CLR, BCL, SDK, рантаймы]]
- [[Сборки, метаданные и IL]]
- [[NuGet: пакеты, версии, транзитивные зависимости]]
- [[Файл проекта .csproj изнутри]]
- [[dotnet CLI: полный обзор]]
- [[JIT, Tiered Compilation, ReadyToRun, Native AOT]]
- [[Native AOT: когда стоит]]
- [[Span, ReadOnlySpan и Memory]]
- [[Nullable reference types]]
- [[Source Generators]]
- [[Атрибуты]]
- [[Новое в C# 12, 13, 14]]
- [[Что впереди: .NET 11 и C# 15]]
- [[Docker: образы для .NET]]
- [[Настройка окружения]]
- [[FAQ и мифы о .NET]]
