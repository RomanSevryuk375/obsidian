---
aliases:
tags:
  - C_sharp
  - dotnet
  - aspnet
date: 2026-05-29 17:57
status:
---
### Суть (The "What")
**Атрибуты** в [[C#]] — это декларативные метаданные, которые можно добавлять к классам, методам или параметрам. 
**Фильтры** в ASP.NET Core — это компоненты, которые позволяют выполнять код до или после определенных этапов конвейера обработки запроса (Action Invocation Pipeline).

В связке они решают задачу реализации **сквозной функциональности (cross-cutting concerns)**. Вместо дублирования кода авторизации, кэширования, валидации или обработки ошибок в каждом контроллере, эта логика выносится в фильтры и декларативно применяется с помощью атрибутов.

---

### Как это работает (The "How" / Nutshell style)

#### 1. Место в конвейере ASP.NET Core
Фильтры работают внутри конвейера действий MVC/API. Важно понимать разницу между Middleware (промежуточным ПО) и Фильтрами:
*   **[[Middleware]]:** обрабатывает сырой [[HTTP]]-запрос на уровне `HttpContext`. Оно ничего не знает о маршрутизации, контроллерах, параметрах действий или состоянии валидации (`ModelState`).
*   **Фильтры:** запускаются после того, как маршрутизатор выбрал нужное действие (Action). У них есть полный доступ к контексту MVC (дескрипторы действий, аргументы, результаты).

#### 2. Жизненный цикл фильтров (Типы фильтров)
Конвейер фильтров состоит из пяти последовательных стадий:
1.  **Authorization Filters (`IAuthorizationFilter`):** Выполняются первыми. Определяют, разрешен ли доступ к действию.
2.  **Resource Filters (`IResourceFilter`):** Выполняются до привязки модели (model binding). Полезны для реализации кэширования (могут завершить обработку запроса досрочно).
3.  **Action Filters (`IActionFilter`):** Выполняются непосредственно перед вызовом метода действия и сразу после него. Позволяют читать и изменять аргументы метода или результат.
4.  **Exception Filters (`IExceptionFilter`):** Выполняются только в том случае, если в методе действия или на этапе Result Filters произошло необработанное исключение.
5.  **Result Filters (`IResultFilter`):** Выполняются до и после генерации конечного результата действия (например, рендеринг JSON или Razor-представления).

#### 3. Внедрение зависимостей ([[Dependency Injection]]) в Атрибуты
Атрибуты инициализируются статически во время загрузки метаданных сборки, поэтому в них нельзя напрямую внедрить зависимости через конструктор. Для решения этой проблемы используются следующие подходы:
*   **`TypeFilterAttribute`:** Динамически создает экземпляр фильтра в рантайме, используя DI-контейнер для разрешения его зависимостей.
*   **`ServiceFilterAttribute`:** Требует явной регистрации класса фильтра в DI-контейнере (как Scoped или Transient).
*   **`IFilterFactory`:** Интерфейс, который позволяет атрибуту самому выступать в роли фабрики для создания фильтра.

> [!INFO] Очередность выполнения (Scope & Order)
> По умолчанию фильтры выполняются в порядке расширения их области видимости: Global (для всего приложения) $\to$ Controller (для класса) $\to$ Action (для метода). Если нужно принудительно изменить этот порядок, фильтры должны реализовывать интерфейс `IOrderedFilter`.

---

### Структура папок и пример реализации

Типичная организация кода фильтров в проекте ASP.NET Core:

```text
MyProject.API/
└── Filters/
    ├── ExecutionTimeFilter.cs       # Логика асинхронного фильтра действия
    └── TrackExecutionAttribute.cs   # Атрибут-маркер (фабрика)
```

#### 1. Реализация фильтра действия (Action Filter)
Используем асинхронную версию интерфейса и первичные конструкторы C# 12 для логирования:

```csharp
using System.Diagnostics;
using Microsoft.AspNetCore.Mvc.Filters;
using Microsoft.Extensions.Logging;

namespace MyProject.API.Filters;

// Фильтр требует логгер, поэтому мы будем внедрять его через DI
public class ExecutionTimeFilter(ILogger<ExecutionTimeFilter> logger) : IAsyncActionFilter
{
    private readonly ILogger<ExecutionTimeFilter> _logger = logger;

    public async Task OnActionExecutionAsync(ActionExecutingContext context, ActionExecutionDelegate next)
    {
        // Логика ДО выполнения метода действия
        var stopwatch = Stopwatch.StartNew();
        var actionName = context.ActionDescriptor.DisplayName;

        _logger.LogInformation("Starting execution of {ActionName}", actionName);

        // Передача управления следующему фильтру или самому методу действия
        var resultContext = await next();

        // Логика ПОСЛЕ выполнения метода действия
        stopwatch.Stop();
        _logger.LogInformation("Finished execution of {ActionName} in {ElapsedMs}ms", actionName, stopwatch.ElapsedMilliseconds);
    }
}
```

#### 2. Атрибут-фабрика для применения фильтра
Реализуем `IFilterFactory`, чтобы избавиться от громоздкого синтаксиса `[TypeFilter(typeof(ExecutionTimeFilter))]`:

```csharp
using Microsoft.AspNetCore.Mvc.Filters;

namespace MyProject.API.Filters;

[AttributeUsage(AttributeTargets.Method | AttributeTargets.Class)]
public class TrackExecutionAttribute : Attribute, IFilterFactory
{
    // Указываем, что фильтр может повторно использоваться между запросами
    public bool IsReusable => true;

    public IFilterMetadata CreateInstance(IServiceProvider serviceProvider)
    {
        // Разрешаем зависимости фильтра из DI-контейнера
        return serviceProvider.GetRequiredService<ExecutionTimeFilter>();
    }
}
```

#### 3. Регистрация и применение
Регистрируем фильтр в `Program.cs` и применяем атрибут в контроллере:

```csharp
// Program.cs
builder.Services.AddScoped<ExecutionTimeFilter>(); // Обязательно для IFilterFactory/ServiceFilter

// MyController.cs
[ApiController]
[Route("[controller]")]
public class UsersController : ControllerBase
{
    [HttpGet]
    [TrackExecution] // Декларативное применение нашего фильтра
    public IActionResult GetUsers() => Ok(new string[] { "Alice", "Bob" });
}
```

---

### Ошибки и Best Practices

> [!DANGER] Anti-pattern: Синхронные фильтры для I/O операций
> Никогда не используйте синхронные версии интерфейсов (например, `IActionFilter`) для задач, включающих работу с БД или сетью. Блокировка потоков внутри синхронного метода `OnActionExecuting` приводит к быстрому исчерпанию пула потоков (Thread Pool Starvation). Используйте только `IAsyncActionFilter`.

> [!WARNING] Ошибка: Избыточность глобальных фильтров
> Добавление сложной логики (например, тяжелых проверок прав или запросов к БД) в глобальные фильтры замедляет абсолютно каждый запрос в приложении. Для специфических проверок применяйте фильтры точечно через атрибуты.

**Best Practices:**
1.  **Используйте правильный тип фильтра:**
    *   Для кэширования ответов — `IResourceFilter` (экономит время на привязке модели).
    *   Для централизованной обработки ошибок — `IExceptionFilter` (хотя в современном ASP.NET Core для этого чаще рекомендуют использовать специализированный Middleware `UseExceptionHandler`).
    *   Для валидации входных данных — `IAsyncActionFilter`.
2.  **Защита от повторного выполнения:** Если фильтр применяется и глобально, и на уровне контроллера, он может выполниться дважды. Проверяйте наличие флагов в `context.HttpContext.Items` или используйте уникальные ключи для предотвращения повторной логики.
3.  **Идемпотентность фильтров авторизации:** Фильтры авторизации должны быстро принимать решение на основе метаданных и токенов, не задействуя тяжелые бизнес-вычисления.
4.  **Короткое замыкание (Short-circuiting):** Если фильтр понимает, что дальнейшая обработка не нужна (например, проверка прав провалена или данные взяты из кэша), он должен установить свойство `context.Result` (например, `context.Result = new UnauthorizedResult()`). Это немедленно прервет выполнение и вернет ответ клиенту, минуя сам метод контроллера.