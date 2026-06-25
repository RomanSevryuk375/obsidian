---
aliases:
tags:
  - C_sharp
  - dotnet
  - architecture
  - tools
date: 2026-06-16 09:07
status:
---
### Суть (The "What")
**Serilog** — это библиотека для структурированного логирования в экосистеме .NET. В отличие от традиционных логгеров, которые сохраняют данные в виде плоских текстовых строк, Serilog изначально спроектирован для записи семантических свойств и полноценных объектов (обычно в формате JSON).

Библиотека решает задачу **анализа и поиска в логах**. Сохраняя переменные как отдельные структурированные поля, она позволяет в системах агрегации ([[Seq]], [[Elasticsearch]], [[Kibana]]) делать сложные [[SQL]]-подобные запросы: находить все ошибки конкретного пользователя (`UserId == 123`) или фильтровать события по времени выполнения (`ElapsedMs > 500`), не прибегая к парсингу текста с помощью регулярных выражений.

---

### Как это работает (The "How" / Nutshell style)

#### 1. Шаблоны сообщений (Message Templates)
Serilog использует собственный механизм форматирования строк, не зависящий от стандартного `string.Format`. Когда вы пишете `Log.Information("User {UserId} logged in", id)`, Serilog не просто склеивает строку. Он создает объект `LogEvent`, который содержит:
*   Оригинальный шаблон сообщения (`"User {UserId} logged in"`).
*   Словарь свойств (Dictionary), где ключу `UserId` сопоставлено значение переменной `id`.

#### 2. Деструктуризация (Destructuring)
Это ключевая фича Serilog на низком уровне. Если передать в лог сложный объект, по умолчанию Serilog вызовет у него `ToString()`. Чтобы заставить библиотеку разобрать объект по полям, используется оператор `@`.
*   `{@User}` — Сериализует объект (с помощью рефлексии и внутреннего кэширования), сохраняя все его публичные свойства в структуру JSON.
*   `{$User}` — Принудительно вызывает `ToString()`.

#### 3. Архитектура: [[Sinks]] и [[Enrichment & Context|Enrichers]]
*   **Sinks (Приемники):** Это целевые точки назначения логов (Console, File, Seq, SQL Server). Каждый Sink реализует интерфейс `ILogEventSink` и обрабатывает поток готовых объектов `LogEvent`.
*   **Enrichers (Обогатители):** Механизм внедрения контекста. Позволяет автоматически добавлять определенные свойства ко всем логам (например, `ThreadId`, `MachineName`, `CorrelationId`), модифицируя `LogEvent` до того, как он попадет в Sink.

#### 4. Контекст выполнения (LogContext)
Serilog использует `AsyncLocal<T>` для управления контекстом. Это позволяет "пробрасывать" данные через всю цепочку асинхронных вызовов (`async/await`) без необходимости передавать их в параметрах методов.

> [!INFO] Нюанс: Управление памятью
> Формирование `LogEvent`, парсинг шаблонов и деструктуризация объектов — это ресурсозатратные операции, требующие аллокаций в куче. Если уровень логирования (LogEventLevel) отключен (например, стоит `Warning`, а вызывается `Debug`), Serilog мгновенно прерывает выполнение, не вычисляя аргументы и не выделяя память.

---

### Пример кода ([[MOC|C#]] 12)

Настройка и использование Serilog в современном приложении ASP.NET Core с использованием первичных конструкторов.

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;
using Serilog;
using Serilog.Context;
using System;

// 1. Инициализация Serilog при старте (Program.cs)
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .Enrich.FromLogContext() // Включаем поддержку контекста
    .Enrich.WithThreadId()
    .WriteTo.Console()
    .WriteTo.Async(a => a.File("logs/app.txt", rollingInterval: RollingInterval.Day)) // Асинхронная запись
    .CreateLogger();

try
{
    var builder = WebApplication.CreateBuilder(args);
    
    // Подменяем стандартный логгер .NET на Serilog
    builder.Host.UseSerilog(); 
    
    builder.Services.AddTransient<OrderService>();

    var app = builder.Build();
    
    // Добавляем встроенное логирование HTTP-запросов от Serilog
    app.UseSerilogRequestLogging(); 

    app.Run();
}
finally
{
    // Обязательно для сброса буферов при завершении приложения
    Log.CloseAndFlush(); 
}

// 2. Использование в бизнес-логике
public record OrderDto(Guid OrderId, decimal Amount, string Currency);

// Использование первичного конструктора C# 12 для DI
public class OrderService(ILogger<OrderService> logger)
{
    private readonly ILogger<OrderService> _logger = logger;

    public void ProcessOrder(OrderDto order)
    {
        // Добавление контекста для всех логов в рамках этого блока using
        using (LogContext.PushProperty("CorrelationId", Guid.NewGuid()))
        {
            _logger.LogInformation("Processing started for {OrderId}", order.OrderId);

            try
            {
                // Имитация бизнес-логики
                if (order.Amount < 0) throw new InvalidOperationException("Negative amount");

                // Использование оператора '@' для деструктуризации объекта в JSON
                _logger.LogInformation("Order processed successfully. Details: {@OrderDetails}", order);
            }
            catch (Exception ex)
            {
                // Передача исключения первым аргументом
                _logger.LogError(ex, "Failed to process order {OrderId}", order.OrderId);
            }
        }
    }
}
```

---

### Ошибки и Best Practices

> [!DANGER] Anti-pattern: Интерполяция строк в логах
> Никогда не используйте интерполяцию C# в шаблонах сообщений: `_logger.LogInformation($"User {userId} logged in");`. 
> Это убивает весь смысл структурированного логирования: Serilog получит готовую строку, не сможет извлечь переменную `userId` в отдельное поле для поиска, а система агрегации "захлебнется", так как будет считать каждое такое сообщение уникальным шаблоном (Memory Leak в кэше шаблонов Serilog).

> [!WARNING] Ошибка: Блокирующий ввод-вывод
> Прямая запись в файл или в сеть (Seq, Elastic) из бизнес-потока может заблокировать приложение при пиковых нагрузках или проблемах с диском/сетью. Всегда оборачивайте медленные приемники в `.WriteTo.Async(...)`. Это создает фоновый поток и `Channel<T>` в памяти для буферизации `LogEvent`.

**Best Practices:**
1.  **CloseAndFlush:** Всегда вызывайте `Log.CloseAndFlush()` в блоке `finally` метода `Main`. Если приложение упадет или завершится, фоновые потоки асинхронных приемников не успеют сбросить последние логи на диск, и вы потеряете информацию о причине падения.
2.  **Защита персональных данных (PII):** При деструктуризации объектов (через `@`) следите за тем, чтобы в лог не попали пароли, номера карт или токены. Используйте атрибуты `[NotLogged]` (через пакет `Destructurama.Attributed`) для маскировки чувствительных полей.
3.  **UseSerilogRequestLogging:** В ASP.NET Core используйте этот middleware. Он заменяет десятки стандартных мелких логов от контроллеров на один компактный и емкий лог-событие для каждого HTTP-запроса, значительно экономя ресурсы и место на диске.
4.  **Семантические имена:** Имена свойств в шаблонах должны быть в PascalCase и быть уникальными в рамках системы (например, `OrderId`, а не просто `Id`), чтобы избежать конфликтов типов полей в Elasticsearch.

> [!TIP] Нюанс: ILOGGER vs Log.Static
> В современном .NET рекомендуется всегда внедрять `Microsoft.Extensions.Logging.ILogger<T>` через DI-контейнер, а не использовать глобальный статический класс `Log.Information(...)`. Это облегчает модульное тестирование и соблюдает принцип инверсии зависимостей ([[Dependency Inversion Principle|DIP]]). Serilog идеально интегрируется в стандартный интерфейс .NET.