---
aliases:
tags:
  - C_sharp
  - dotnet
  - architecture
  - aspnet
date: 2026-06-22 12:51
status:
---
### Суть (The "What")
**Хостинг в ASP.NET Core** — это инфраструктурный механизм (Generic Host), который инициализирует приложение, настраивает встроенный веб-сервер (Kestrel), систему конфигурации, логирование и контейнер внедрения зависимостей ([[Dependency Injection|DI]]), а затем управляет их работой от старта до завершения.

Он решает задачу **инкапсуляции низкоуровневой работы с ОС и сетью**. Вместо того чтобы вручную открывать сокеты и управлять пулами потоков, разработчик работает с единой точкой сборки (`WebApplicationBuilder`), которая гарантирует корректный старт и безопасное (graceful) завершение работы всех компонентов системы.

---

### Как это работает (The "How" / Nutshell style)

#### 1. Архитектура Generic Host
Современный ASP.NET Core построен на базе универсального хоста (`IHost`). Метод `WebApplication.CreateBuilder()` под капотом собирает три основных абстракции:
*   `IConfiguration` (настройки из JSON, переменных окружения).
*   `IServiceCollection` (реестр зависимостей).
*   `ILoggingBuilder` (система логирования).

#### 2. Фазы запуска (Startup Phases)
Жизненный цикл старта строго детерминирован:
1.  **Фаза конфигурации:** Загружаются настройки, регистрируются зависимости (Services).
2.  **Фаза сборки (Build):** Контейнер DI "запечатывается". Создается корневой провайдер служб (Root Service Provider). После вызова `builder.Build()` добавлять новые зависимости нельзя.
3.  **Фаза конвейера (Pipeline):** Вызовы `app.Use...` регистрируют [[Middleware]].
4.  **Фаза запуска (Run):** Сервер Kestrel привязывается к указанным портам (Socket Binding) и начинает слушать входящие HTTP-запросы.

#### 3. IHostApplicationLifetime
[[CLR]] и ASP.NET Core предоставляют интерфейс `IHostApplicationLifetime` для отслеживания состояний приложения. Он содержит три свойства типа [[CancellationToken]]:
*   `ApplicationStarted`: Срабатывает, когда хост полностью загрузился и готов принимать запросы.
*   `ApplicationStopping`: Срабатывает при получении сигнала завершения (например, `Ctrl+C` или `SIGTERM` от Kubernetes). Начинается процесс изящного завершения.
*   `ApplicationStopped`: Срабатывает, когда хост полностью завершил свою работу.

#### 4. Изящное завершение (Graceful Shutdown)
Когда ОС посылает сигнал остановки процесса, хост не "убивает" потоки мгновенно. 
*   Kestrel перестает принимать *новые* соединения.
*   Существующим запросам дается время на завершение (по умолчанию 5 секунд).
*   Вызываются все зарегистрированные делегаты на токен `ApplicationStopping`.
*   У всех Singleton-сервисов, реализующих `IDisposable`, вызывается метод `Dispose()`.

> [!INFO] Нюанс: Root Scope
> При сборке хоста создается корневая область видимости DI (Root Scope). Singleton-сервисы живут в ней. Transient и Scoped-сервисы создаются в дочерних областях. Важно: никогда не создавайте Scoped-сервисы вручную из корневого провайдера вне HTTP-запроса, иначе они останутся в памяти до остановки всего приложения (утечка памяти).

---

### Пример кода ([[MOC|C#]] 12)

Пример регистрации сервиса, который интегрируется в жизненный цикл приложения для выполнения задач при старте и остановке.

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using System.Threading;
using System.Threading.Tasks;

var builder = WebApplication.CreateBuilder(args);

// Регистрация IHostedService (выполняется в фоне вместе с хостом)
builder.Services.AddHostedService<LifecycleManagerService>();

var app = builder.Build();

app.MapGet("/", () => "Host is running");

app.Run();

// Сервис, использующий первичный конструктор (C# 12) для инжекта зависимостей
public class LifecycleManagerService(
    IHostApplicationLifetime appLifetime, 
    ILogger<LifecycleManagerService> logger) : IHostedService
{
    private readonly IHostApplicationLifetime _appLifetime = appLifetime;
    private readonly ILogger<LifecycleManagerService> _logger = logger;

    public Task StartAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("1. StartAsync: Подготовка к запуску фоновых задач.");

        // Подписка на события жизненного цикла
        _appLifetime.ApplicationStarted.Register(() => 
            _logger.LogInformation("2. Приложение полностью запущено и принимает запросы."));

        _appLifetime.ApplicationStopping.Register(() => 
            _logger.LogInformation("3. Получен сигнал остановки (SIGTERM/Ctrl+C). Закрываем соединения."));

        _appLifetime.ApplicationStopped.Register(() => 
            _logger.LogInformation("4. Приложение полностью остановлено. Очистка ресурсов."));

        return Task.CompletedTask;
    }

    public Task StopAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("5. StopAsync: Принудительное завершение фоновых задач.");
        return Task.CompletedTask;
    }
}
```

---

### Ошибки и Best Practices

> [!DANGER] Anti-pattern: Блокирующий код при старте
> Выполнение тяжелых синхронных операций (например, долгие миграции базы данных или загрузка гигабайтных файлов в кэш) прямо в методе `Program.cs` до вызова `app.Run()`. Это заблокирует старт приложения. В облачных средах (Kubernetes) оркестратор может посчитать контейнер "мертвым" (провал Liveness Probe) и убить его до того, как он запустится.

> [!WARNING] Ошибка: Игнорирование токена ApplicationStopping
> Если ваши фоновые задачи (`BackgroundService`) содержат бесконечные циклы `while(true)` и не проверяют токен отмены, приложение никогда не сможет корректно завершиться. ОС будет вынуждена принудительно убить процесс (SIGKILL), что приведет к потере данных, которые находились в памяти.

**Best Practices:**
1.  **IHostedService для старта:** Всю логику инициализации (разогрев кэша, запуск слушателей очередей [[RabbitMQ]]) выносите в классы, реализующие `IHostedService` или `BackgroundService`.
2.  **Миграции БД:** Не выполняйте миграции Entity Framework при старте веб-сервера (`context.Database.Migrate()`) в production-среде. Если стартуют сразу несколько реплик вашего сервиса, они могут заблокировать друг друга (Deadlock в БД). Миграции должны применяться отдельным процессом в CI/CD пайплайне (Init Container).
3.  **Graceful Shutdown Timeout:** В конфигурации хоста можно изменить время, которое дается на изящное завершение (по умолчанию 5 секунд). Если ваши фоновые задачи требуют больше времени на сохранение состояния, увеличьте `ShutdownTimeout` в `HostOptions`.
4.  **Environment Variables:** Полагайтесь на `builder.Environment.IsDevelopment()` для настройки жизненного цикла и логирования. Разделяйте конфигурацию для разработки (где ошибки могут выводиться в UI) и продакшена.

> [!TIP] Нюанс: IStartupFilter (Legacy)
> В старых версиях ASP.NET Core использовался интерфейс `IStartupFilter` для вклинивания в конвейер до старта сервера. В современном .NET (начиная с 6.0 и `WebApplication`) этот паттерн потерял актуальность, так как все Middleware можно настраивать прямо в `Program.cs` последовательно и прозрачно.