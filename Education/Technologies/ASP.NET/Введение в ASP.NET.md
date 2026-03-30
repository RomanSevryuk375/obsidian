---
aliases:
tags:
  - C_sharp
  - dotnet
  - api
  - aspnet
  - review
date: 2026-03-25 17:35
status:
---
### Суть (The "What")
**ASP.NET Core** — это кроссплатформенный высокопроизводительный фреймворк с открытым исходным кодом, предназначенный для создания современных облачных приложений, таких как веб-сервисы, API и [[Microservices Architecture|микросервисной архитектуры]].

Он решает проблему **монолитности и зависимости от платформы**, которые были присущи классическому ASP.NET (System.Web). Современный стек полностью модульный, поставляется через NuGet-пакеты и работает на базе легковесного Generic Host.

---

### Как это работает (The "How" / Nutshell style)

#### 1. Хостинг и [[Kestrel]]
В основе приложения лежит сервер **Kestrel** — кроссплатформенный веб-сервер, оптимизированный для асинхронного ввода-вывода (I/O).
*   **Generic Host:** Приложение запускается как обычное консольное приложение. `Host` управляет жизненным циклом, конфигурацией и внедрением зависимостей (DI).
*   **Прослушивание:** Kestrel принимает HTTP-запросы и превращает их в объект `HttpContext`, который содержит данные о запросе (`HttpRequest`) и ответе (`HttpResponse`).

#### 2. Конвейер [[Middleware]] (Middleware Pipeline)
Запрос проходит через цепочку компонентов, называемых **Middleware**.
*   **Модель «Матрешки»:** Каждое Middleware может выполнить логику до и после вызова следующего компонента (`next()`).
*   **Порядок важен:** Авторизация должна идти после аутентификации, а обработка исключений — в самом начале, чтобы поймать ошибки из всех последующих звеньев.

#### 3. Внедрение зависимостей ([[Dependency Injection]])
DI является «гражданином первого класса» в ASP.NET Core. Система типов [[CLR]] используется для разрешения зависимостей через конструктор.
*   **Lifetimes:** 
    *   `Transient`: Создается при каждом запросе сервиса.
    *   `Scoped`: Создается один раз на каждый HTTP-запрос.
    *   `Singleton`: Один экземпляр на всё время жизни приложения.

#### 4. Маршрутизация ([[Routing]])
Современный ASP.NET Core использует **Endpoint Routing**. Система сопоставляет URL запроса с конкретной конечной точкой (Action контроллера или лямбда в Minimal API) еще до начала выполнения основной логики middleware.

> [!INFO] Нюанс: Производительность
> В отличие от старого ASP.NET, здесь нет зависимости от тяжелого модуля `System.Web.dll` и IIS. Это позволяет достигать миллионов запросов в секунду на стандартном железе за счет минимизации аллокаций в куче и использования `Span<T>` в парсере HTTP.

---

### Пример кода (C# 12 - Minimal APIs)

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.Extensions.DependencyInjection;

var builder = WebApplication.CreateBuilder(args);

// 1. Регистрация сервисов (DI Container)
builder.Services.AddScoped<ITimeService, TimeService>();

var app = builder.Build();

// 2. Конфигурация Middleware (Pipeline)
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
}

app.UseHttpsRedirection();

// 3. Определение эндпоинтов (Routing)
app.MapGet("/time", (ITimeService timer) => 
{
    return Results.Ok(new { CurrentTime = timer.GetNow() });
});

app.Run();

// Вспомогательный сервис
public interface ITimeService { string GetNow(); }
public class TimeService : ITimeService 
{
    public string GetNow() => DateTime.UtcNow.ToLongTimeString();
}
```

---

### Ошибки и Best Practices

> [!DANGER] Anti-pattern: Блокирующий код в контроллерах
> Никогда не используйте `.Result` или `.Wait()` внутри контроллеров или middleware. Это приводит к **Thread Pool Starvation** (голоданию пула потоков). Весь стек ASP.NET Core спроектирован как асинхронный (`async/await`).

> [!WARNING] Ошибка: Захват Scoped сервиса в Singleton
> Если Singleton-сервис требует Scoped-сервис (например, контекст БД), вы получите исключение при запуске. Используйте `IServiceScopeFactory` внутри Singleton для создания временного контекста.

**Best Practices:**
1.  **Minimal APIs:** Для простых микросервисов используйте Minimal APIs (как в примере выше) — это снижает накладные расходы на инфраструктуру контроллеров.
2.  **[[CancellationToken]]:** Всегда принимайте `CancellationToken` в методах действий и передавайте его в БД или HttpClient. Это позволяет серверу мгновенно прекратить работу, если пользователь закрыл вкладку браузера.
3.  **Здоровье приложения (Health Checks):** Используйте встроенный механизм `/health`, чтобы системы оркестрации (Kubernetes) могли следить за состоянием вашего приложения.
4.  **Options Pattern:** Не читайте конфигурацию напрямую через `IConfiguration`. Используйте строго типизированные классы и `IOptions<T>`.

> [!TIP] Нюанс: Middleware vs Filters
> Middleware работает на уровне HTTP-запроса (сырые данные). Filters (в MVC) работают в контексте действий контроллера и имеют доступ к метаданным C# (параметрам методов, атрибутам). Выбирайте инструмент в зависимости от того, на каком уровне абстракции вы находитесь.