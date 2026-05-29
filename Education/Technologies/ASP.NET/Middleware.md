---
aliases:
tags:
  - C_sharp
  - dotnet
  - aspnet
date: 2026-05-29 18:15
status:
---
### Суть (The "What")
**Middleware** — это программные компоненты, которые выстраиваются в цепочку (конвейер, pipeline) для обработки входящих [[HTTP]]-запросов и исходящих ответов. Каждый компонент может выполнить код до и после следующего компонента, а также принять решение: передать запрос дальше по конвейеру (`next`) или прервать выполнение (короткое замыкание / short-circuiting).

Middleware решает задачу **сквозной низкоуровневой обработки HTTP-трафика**. Оно используется для логирования, аутентификации, авторизации, раздачи статических файлов, обработки ошибок, CORS, сжатия ответов и маршрутизации.

---

### Как это работает (The "How" / Nutshell style)

#### 1. Паттерн "Цепочка обязанностей" (Chain of Responsibility)
Конвейер middleware является двунаправленным. Запрос заходит в приложение, последовательно проходит через все компоненты "туда", достигает конечной точки (например, контроллера) и возвращается "обратно", проходя те же компоненты в обратном порядке. Это позволяет middleware модифицировать не только запрос, но и итоговый HTTP-ответ.

#### 2. Порядок регистрации определяет порядок выполнения
Порядок вызова методов `app.Use...` в файле `Program.cs` имеет критическое значение.
*   Например, `UseStaticFiles()` должен вызываться до `UseRouting()`, чтобы запросы к физическим файлам (картинкам, скриптам) обрабатывались сразу, не запуская тяжелый механизм маршрутизации.

#### 3. Типы реализации Middleware
В .NET Core существует два способа создания пользовательского middleware:

*   **По соглашению (Convention-based):**
    *   Регистрируется как **Singleton** при старте приложения (создается один раз).
    *   Должно содержать конструктор, принимающий `RequestDelegate next`.
    *   Должно иметь метод `Invoke` или `InvokeAsync`, принимающий `HttpContext` первым параметром.

*   **На основе фабрики (Factory-based / `IMiddleware`):**
    *   Реализует интерфейс `IMiddleware`.
    *   Регистрируется в [[Dependency Injection|DI]]-контейнере и может создаваться как **Scoped** или **Transient** (на каждый запрос или при каждом обращении).
    *   Позволяет безопасно внедрять зависимости в конструктор.

> [!DANGER] Утечка зависимостей (Captive Dependency)
> Поскольку Convention-based middleware является Singleton-объектом, вы **не имеете права** внедрять в его конструктор короткоживущие зависимости (например, `DbContext` или репозитории с временем жизни Scoped). Это приведет к удержанию зависимости на все время жизни приложения. Внедрять такие зависимости нужно непосредственно в параметры метода `InvokeAsync`.

---

### Структура папок и пример реализации

Рекомендуемая организация кода для кастомного middleware в проекте:

```text
MyProject.API/
└── Middleware/
    ├── RequestLoggingMiddleware.cs  # Логика обработки запроса (IMiddleware)
    └── MiddlewareExtensions.cs      # Методы расширения для регистрации в Program.cs
```

#### 1. Реализация Middleware через `IMiddleware` ([[C#]] 12)
Используем первичный конструктор для безопасного внедрения логгера (Scoped/Transient время жизни):

```csharp
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Logging;
using System.Diagnostics;
using System.Threading.Tasks;

namespace MyProject.API.Middleware;

// Использование первичного конструктора C# 12 для DI
public class RequestLoggingMiddleware(ILogger<RequestLoggingMiddleware> logger) : IMiddleware
{
    private readonly ILogger<RequestLoggingMiddleware> _logger = logger;

    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        var stopwatch = Stopwatch.StartNew();
        var requestPath = context.Request.Path;

        _logger.LogInformation("HTTP Request started: {Path}", requestPath);

        // Передача управления следующему компоненту в цепочке
        await next(context);

        stopwatch.Stop();
        var statusCode = context.Response.StatusCode;

        _logger.LogInformation("HTTP Request finished: {Path} with status {StatusCode} in {ElapsedMs}ms", 
            requestPath, statusCode, stopwatch.ElapsedMilliseconds);
    }
}
```

#### 2. Создание метода расширения для регистрации
```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.Extensions.DependencyInjection;

namespace MyProject.API.Middleware;

public static class MiddlewareExtensions
{
    public static IServiceCollection AddCustomMiddleware(this IServiceCollection services)
    {
        // IMiddleware обязательно требует регистрации в DI-контейнере
        services.AddTransient<RequestLoggingMiddleware>();
        return services;
    }

    public static IApplicationBuilder UseRequestLogging(this IApplicationBuilder app)
    {
        return app.UseMiddleware<RequestLoggingMiddleware>();
    }
}
```

#### 3. Подключение в Program.cs
```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Регистрация в DI
builder.Services.AddCustomMiddleware();

var app = builder.Build();

// Подключение к конвейеру (порядок критичен!)
app.UseRequestLogging(); 

app.MapGet("/", () => "Hello World!");
app.Run();
```

---

### Ошибки и Best Practices

> [!WARNING] Ошибка: Чтение тела запроса (Request Body)
> Если вам нужно прочитать тело запроса в middleware (например, для логирования payload), помните, что поток тела запроса (`context.Request.Body`) является однонаправленным (forward-only). Чтобы прочитать его без поломки последующих этапов (привязки модели), необходимо предварительно включить буферизацию: `context.Request.EnableBuffering()`, прочитать поток, а затем сбросить его позицию: `context.Request.Body.Position = 0;`.

> [!CAUTION] Ошибка: Модификация заголовков после отправки ответа
> Не пытайтесь изменить заголовки ответа (`context.Response.Headers`) после того, как управление вернулось от вызова `await next(context)`. В этот момент тело ответа уже могло начать отправляться клиенту, и любая попытка изменить заголовки выбросит `InvalidOperationException`. Если нужно изменить ответ, делайте это до вызова `next` или используйте событие `context.Response.OnStarting()`.

**Best Practices:**
1.  **Используйте IMiddleware:** Для кастомных компонентов отдавайте предпочтение фабричному `IMiddleware` — это избавляет от архитектурных ошибок со временем жизни зависимостей.
2.  **Короткое замыкание:** Если middleware прерывает запрос (например, при проверке API-ключа), не вызывайте `await next(context)`. Вместо этого установите код статуса (например, 401) и запишите сообщение напрямую в `context.Response.WriteAsync()`.
3.  **Не усложняйте бизнес-логикой:** Middleware должно заниматься исключительно инфраструктурными задачами (транспорт, безопасность, логирование). Бизнес-логика должна жить в сервисах и контроллерах.
4.  **Разделяйте конвейеры:** Используйте метод `app.Map()` или `app.MapWhen()`, чтобы создавать ветвления конвейера. Это позволяет применять определенные middleware только к специфическим URL-адресам.