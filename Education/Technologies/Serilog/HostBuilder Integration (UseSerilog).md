---
aliases:
tags:
  - C_sharp
  - dotnet
  - architecture
  - tools
date: 2026-06-16 09:30
status:
---
### Суть (The "What")
Интеграция через `UseSerilog` — это процесс подмены стандартной подсистемы логирования Microsoft (`Microsoft.Extensions.Logging`) на [[Serilog]] на уровне хоста приложения (.NET Generic Host).

Это решает проблему **разрозненности логов**: вместо того чтобы настраивать каждый провайдер отдельно (консоль, файлы, внешние системы), вы настраиваете единый конвейер Serilog, который перехватывает все сообщения от системных компонентов .NET, сторонних библиотек и вашего собственного кода.

---

### Как это работает (The "How" / Nutshell style)

#### 1. Замещение провайдеров
Метод расширения `UseSerilog` (находящийся в пакете `Serilog.AspNetCore` или `Serilog.Extensions.Hosting`) выполняет критическую низкоуровневую операцию: он очищает коллекцию `ILoggerProvider` в контейнере [[Dependency Injection]] и регистрирует там единственный `SerilogLoggerProvider`. 
*   Все вызовы интерфейса `ILogger<T>` теперь направляются напрямую в ядро Serilog.

#### 2. Два способа инициализации
*   **Статический логгер (Bootstrap):** Инициализируется в самом начале `Program.cs` через `Log.Logger`. Это необходимо для логирования ошибок старта самого хоста (например, если произошла ошибка в `appsettings.json`).
*   **Контейнерный логгер:** Настраивается через `builder.Services.AddSerilog(...)` (в .NET 7/8+) или `builder.Host.UseSerilog(...)`. В этом случае конфигурация может использовать другие службы из DI (например, доступ к `IConfiguration`).

#### 3. Перехват системных событий
При интеграции в Host, Serilog начинает получать события из диагностических источников .NET (DiagnosticSource). 
*   **Нюанс:** Чтобы избежать дублирования и "мусора" от системных компонентов [[ASP.NET MOC|ASP.NET]] Core, часто используется метод `.UseSerilogRequestLogging()`, который заменяет стандартные подробные логи фреймворка на одно структурированное событие для каждого HTTP-запроса.

#### 4. Взаимодействие с [[App Settings and Configurations|IConfiguration]]
Serilog умеет динамически перестраивать конвейер при изменении файла настроек. Пакет `Serilog.Settings.Configuration` позволяет делегировать управление уровнями логирования и фильтрами администраторам системы без перекомпиляции кода.

---

### Пример кода ([[MOC|C#]] 12)

Современная настройка в стиле .NET 8/9 с использованием первичного конструктора и разделением на Bootstrap и основной логгер.

```csharp
using Serilog;
using Serilog.Events;

// 1. Bootstrap-логирование (для ошибок запуска)
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Override("Microsoft", LogEventLevel.Information)
    .Enrich.FromLogContext()
    .WriteTo.Console()
    .CreateBootstrapLogger();

try
{
    var builder = WebApplication.CreateBuilder(args);

    // 2. Интеграция Serilog в HostBuilder
    // Считывает настройки из appsettings.json и позволяет переопределить их в коде
    builder.Services.AddSerilog((services, lc) => lc
        .ReadFrom.Configuration(builder.Configuration)
        .ReadFrom.Services(services)
        .Enrich.WithProperty("ApplicationName", "MyService")
        .WriteTo.Console(outputTemplate: "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj}{NewLine}{Exception}"));

    var app = builder.Build();

    // 3. Оптимизация логирования HTTP-запросов
    app.UseSerilogRequestLogging();

    app.MapGet("/", (ILogger<Program> logger) => {
        logger.LogInformation("Обработка запроса в {Time}", DateTime.Now);
        return "Hello!";
    });

    app.Run();
}
catch (Exception ex)
{
    Log.Fatal(ex, "Приложение аварийно завершилось при запуске");
}
finally
{
    // Гарантированный сброс буферов (особенно важно для файловых Sinks)
    Log.CloseAndFlush();
}
```

---

### Ошибки и Best Practices

> [!DANGER] Anti-pattern: Забытый CloseAndFlush
> Если не вызвать `Log.CloseAndFlush()` в блоке `finally` метода `Main`, последние сообщения (включая критические ошибки падения) могут не успеть записаться в файл или отправиться в сеть (например, в Seq или Elasticsearch), так как асинхронные потоки Serilog будут убиты операционной системой.

> [!WARNING] Ошибка: Повторная регистрация провайдеров
> Если вы вызываете `builder.Logging.AddConsole()` вместе с `UseSerilog`, вы можете получить дублирование логов в консоли. `UseSerilog` по умолчанию пытается подавить другие провайдеры, но явная настройка логирования через `builder.Logging` может привести к конфликтам конфигураций.

**Best Practices:**
1.  **Bootstrap Logger:** Всегда используйте `CreateBootstrapLogger()`. Это гарантирует, что вы увидите логи, даже если конфигурация базы данных или файла настроек повреждена и хост не может подняться.
2.  **Request Logging:** Используйте `app.UseSerilogRequestLogging()`. Это значительно снижает объем хранимых логов в облаке или на диске, объединяя информацию о маршруте, коде ответа и времени выполнения в одну запись.
3.  **DI Integration:** Внедряйте `ILogger<T>` в конструкторы вместо использования статического `Log.Information`. Это делает код тестируемым и позволяет Serilog автоматически добавлять имя класса (SourceContext) к каждому сообщению.
4.  **[[Enrichment & Context|Enrichment]]:** Используйте `Enrich.FromLogContext()`. Это позволяет использовать `LogContext.PushProperty`, чтобы "приклеить" метаданные (например, `TransactionId`) ко всем логам внутри конкретного блока кода.

> [!TIP] Нюанс из Nutshell
> Serilog при интеграции с HostBuilder использует `AsyncLocal<T>` для реализации `LogContext`. Это означает, что свойства контекста корректно передаются между потоками в рамках одной асинхронной операции (`async/await`), что критично для отслеживания цепочек вызовов в микросервисах.