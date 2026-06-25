---
aliases:
tags:
  - C_sharp
  - aspnet
  - dotnet
  - architecture
date: 2026-06-22 13:25
status:
---
### Суть (The "What")
**WebApplicationBuilder** — это современный API (представленный в .NET 6), который является легковесной и плоской оберткой над фундаментальным механизмом **Generic Host** (`IHost` / `IHostBuilder`).

Он решает задачу **упрощения инициализации приложения (Bootstrap)**. Старая архитектура с файлом `Startup.cs` требовала сложной иерархии вложенных лямбда-выражений для настройки [[Dependency Injection|DI]] и конфигурации. `WebApplicationBuilder` делает код линейным, позволяя читать конфигурацию и регистрировать зависимости одновременно, без промежуточных сборок контейнера.

---

### Как это работает (The "How" / Nutshell style)

#### 1. Универсальный хост (Generic Host)
Под капотом ASP.NET Core работает не просто веб-сервер, а **Generic Host** (`Microsoft.Extensions.Hosting`). Это ядро, которое управляет:
*   Контейнером внедрения зависимостей (Dependency Injection).
*   Системой конфигурации (`IConfiguration`).
*   Системой логирования (`ILoggerFactory`).
*   Жизненным циклом приложения (`IHostApplicationLifetime`).
Хост может запускать не только веб-серверы (Kestrel), но и фоновые демоны (Worker Services) через `IHostedService`.

#### 2. Магия ConfigurationManager
В старом `IHostBuilder` существовала "проблема двойной сборки". Если вам нужно было прочитать `appsettings.json`, чтобы понять, какую базу данных зарегистрировать в DI, приходилось предварительно вызывать `Build()` для конфигурации.
*   `WebApplicationBuilder` решает это с помощью нового класса `ConfigurationManager`. Он реализует одновременно `IConfigurationBuilder` (для добавления источников) и `IConfigurationRoot` (для чтения данных). 
*   **Низкий уровень:** Когда вы добавляете новый источник JSON, `ConfigurationManager` динамически перестраивает внутренний словарь конфигурации "на лету" (без аллокации новых объектов-провайдеров). Это позволяет читать `builder.Configuration` прямо во время настройки `builder.Services` без потери производительности.

#### 3. Запечатывание контейнера (The Build Phase)
Вызов `builder.Build()` — это точка невозврата.
*   На уровне CLR создается **Root Service Provider** (корневой провайдер служб).
*   Коллекция `IServiceCollection` блокируется (становится доступной только для чтения).
*   Создается объект `WebApplication`, который реализует `IApplicationBuilder` для настройки конвейера Middleware. Попытка добавить новую службу в DI после вызова `Build()` выбросит `InvalidOperationException`.

---

### Пример кода (C# 12)

Современная структура `Program.cs` с использованием плоского API, настройкой хоста и C# 12.

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using System;

// 1. Инициализация WebApplicationBuilder (скрывает IHostBuilder)
var builder = WebApplication.CreateBuilder(args);

// Чтение конфигурации доступно мгновенно благодаря ConfigurationManager
string? dbProvider = builder.Configuration["Database:Provider"];

// 2. Настройка Generic Host (Фоновые задачи, логирование, DI)
builder.Services.AddSingleton<ITimeService, SystemTimeService>();

if (dbProvider == "Postgres")
{
    // builder.Services.AddNpgsql(...);
}

// Добавление фоновой задачи в цикл Generic Host
builder.Services.AddHostedService<CacheWarmerService>();

// 3. Точка невозврата: компиляция контейнера и хоста
var app = builder.Build();

// 4. Настройка конвейера (Middleware Pipeline)
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
}

app.UseRouting();

// Минимальный API
app.MapGet("/time", (ITimeService timeService) => timeService.GetCurrentTime());

// 5. Запуск Generic Host (стартует Kestrel и все IHostedService)
app.Run();

// ------------------------------------------------------------------
// Вспомогательные классы с использованием первичных конструкторов (C# 12)

public interface ITimeService { string GetCurrentTime(); }
public class SystemTimeService : ITimeService { public string GetCurrentTime() => DateTime.Now.ToString("T"); }

public class CacheWarmerService(ITimeService timeService) : BackgroundService
{
    private readonly ITimeService _timeService = timeService;

    protected override Task ExecuteAsync(CancellationToken stoppingToken)
    {
        Console.WriteLine($"Cache warmed up at: {_timeService.GetCurrentTime()}");
        return Task.CompletedTask;
    }
}
```

---

### Ошибки и Best Practices

> [!DANGER] Anti-pattern: Ручная сборка провайдера (BuildServiceProvider)
> Никогда не вызывайте `builder.Services.BuildServiceProvider()` внутри `Program.cs`. Это создаст **второй, скрытый корневой контейнер DI**. Объекты, зарегистрированные как `Singleton`, будут существовать в двух экземплярах (один в вашем временном контейнере, другой — в настоящем хосте), что приведет к рассинхронизации состояния и утечкам памяти. Для доступа к сервисам при инициализации используйте фабрики или `app.Services` после вызова `builder.Build()`.

> [!WARNING] Ошибка: Смешивание builder и app
> Часто разработчики пытаются настроить маршрутизацию или Middleware на этапе `builder` или добавить сервисы в `app.Services`. Помните жесткое разделение: `builder` — это зависимости и настройки среды (DI, Config). `app` — это конвейер HTTP-запросов (Middleware, Endpoints).

**Best Practices:**
1.  **Расширения для чистой архитектуры:** Если код в `Program.cs` разрастается, выносите логику в статические методы расширения: `builder.Services.AddApplicationServices()`. Это сохраняет абстракцию и читаемость (паттерн Composition Root).
2.  **Environment Variables:** Активно используйте `builder.Environment` до сборки приложения, чтобы условно регистрировать зависимости (например, Mock-репозитории для `IsDevelopment()` и реальные для `IsProduction()`).
3.  **Host Options:** Используйте `builder.Host.ConfigureHostOptions(options => ...)`, чтобы настроить системные параметры Generic Host (например, `ShutdownTimeout` — время, которое дается приложению на "изящное" завершение работы перед принудительным убийством процесса ОС).
4.  **IStartupFilter:** Если вы пишете библиотеку (NuGet-пакет) и вам нужно вклиниться в `app` (конвейер Middleware) без изменения файла `Program.cs` конечным пользователем, используйте `IStartupFilter`. Он позволяет библиотеке прозрачно зарегистрировать свои Middleware на этапе сборки хоста.