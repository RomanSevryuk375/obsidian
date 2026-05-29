---
aliases:
tags:
  - aspnet
  - C_sharp
  - dotnet
date: 2026-05-29 18:52
status:
---
# Время жизни служб (Service Lifetimes) в .NET

### Суть (The "What")
**Service Lifetimes (время жизни служб)** — это набор правил, определяющих, как встроенный [[Inversion of Control|IoC]]-контейнер .NET управляет жизненным циклом создаваемых им объектов: когда создавать новый экземпляр класса, как долго удерживать его в памяти и когда уничтожать.

Этот механизм решает задачу **эффективного управления ресурсами**. Он гарантирует, что тяжелые и потенциально небезопасные для многопоточности объекты (например, соединения с базой данных) будут вовремя закрываться и освобождаться, а глобальные утилиты не будут плодить лишние копии в куче, экономя память.

---

### Как это работает (The "How" / Nutshell style)

В .NET [[Dependency Injection|DI]]-контейнер предлагает три основных времени жизни для регистрируемых служб:
#### 1. Transient (Временный)
*   **Регистрация:** `builder.Services.AddTransient<IService, Service>();`
*   **Механика:** Новый экземпляр создается **каждый раз**, когда он запрашивается из контейнера (внедряется в конструктор другого класса или извлекается вручную).
*   **Применение:** Легковесные службы без внутреннего состояния (stateless), такие как хелперы форматирования, валидаторы.
#### 2. Scoped (В рамках области)
*   **Регистрация:** `builder.Services.AddScoped<IService, Service>();`
*   **Механика:** Один экземпляр создается **один раз на область видимости (Scope)**. В ASP.NET Core область видимости создается автоматически для каждого входящего HTTP-запроса и уничтожается после отправки ответа клиенту. Все классы, участвующие в обработке этого [[HTTP]]-запроса, получат одну и ту же ссылку.
*   **Применение:** Службы, привязанные к состоянию транзакции или запроса: `DbContext` в Entity Framework Core, репозитории, контекст текущего пользователя.
#### 3. Singleton (Одиночка)
*   **Регистрация:** `builder.Services.AddSingleton<IService, Service>();`
*   **Механика:** Один экземпляр создается при первом обращении (или принудительно при старте приложения) и **живет все время работы процесса**. Все запросы ко всем классам получат одну и ту же ссылку.
*   **Применение:** Службы с глобальным состоянием или тяжелой инициализацией: кэш-менеджеры, пулы [[TCP]]-соединений, конфигурации.
#### 4. Автоматическая очистка (Dispose)
IoC-контейнер отслеживает все созданные им объекты. Если тип реализует `IDisposable` или `IAsyncDisposable`:
*   Для `Transient` и `Scoped` контейнер автоматически вызывает `Dispose()` в конце жизни их области видимости (для Scoped — при завершении HTTP-запроса).
*   Для `Singleton` метод `Dispose()` вызывается при остановке всего приложения.

> [!DANGER] Улавливание зависимости (Captive Dependency)
> Типичная ошибка: внедрение Scoped-службы (например, репозитория, работающего с БД) в Singleton-службу. Singleton живет вечно, поэтому он "схватит" Scoped-объект и никогда его не отпустит. Это сломает границы запроса, приведет к утечкам памяти и ошибкам многопоточного доступа внутри не-потокобезопасного `DbContext`.

---

### Структура папок и пример кода ([[C#]] 12)

Реализация фоновой службы (`Singleton`), которая должна безопасно обращаться к контексту базы данных (`Scoped`) с использованием фабрики областей видимости:

```text
MyProject.API/
├── Services/
│   ├── IDbService.cs            # Scoped зависимость
│   └── QueueProcessor.cs        # Singleton (IHostedService)
└── Program.cs
```

#### 1. Реализация фонового воркера (C# 12 Primary Constructors)
Фоновые службы регистрируются как Singleton. Мы не можем внедрить Scoped-сервис напрямую, поэтому создаем `Scope` вручную через `IServiceScopeFactory`:

```csharp
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using System;
using System.Threading;
using System.Threading.Tasks;

namespace MyProject.API.Services;

public interface IDbService { Task SaveMetricsAsync(); }

// Наследуем BackgroundService (всегда регистрируется как Singleton)
public class QueueProcessor(IServiceScopeFactory scopeFactory) : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory = scopeFactory;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            // Создаем ручной Scope для безопасного извлечения Scoped-зависимостей
            using (var scope = _scopeFactory.CreateScope())
            {
                // Разрешаем Scoped-службу внутри созданной области видимости
                var dbService = scope.ServiceProvider.GetRequiredService<IDbService>();
                await dbService.SaveMetricsAsync();
            }

            await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
        }
    }
}
```

#### 2. Регистрация в Program.cs
```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Регистрация с правильными временами жизни
builder.Services.AddScoped<IDbService, SqlDbService>(); // Scoped (БД)
builder.Services.AddHostedService<QueueProcessor>();     // Singleton (Фоновая задача)

var app = builder.Build();
app.Run();
```

---

### Ошибки и Best Practices

> [!DANGER] Ошибка: Transient-утечки в Singleton-объектах
> Если вы внедряете `Transient` объект, реализующий `IDisposable`, в `Singleton` объект, то метод `Dispose` у временного объекта никогда не вызовется до завершения работы приложения. Это гарантированно вызовет утечку памяти. Избегайте внедрения `Disposable-Transient` объектов в синглтоны.

> [!WARNING] Ошибка: Отключение валидации областей
> В .NET Core по умолчанию в среде `Development` включена валидация областей видимости (`ValidateScopes = true`). Никогда не отключайте её принудительно. Она защищает вас от деплоя кода с "плененными зависимостями" (Captive Dependencies) на продакшн.

**Best Practices:**
1.  **Потокобезопасность синглтонов:** Любая служба, зарегистрированная как `Singleton`, обязана быть абсолютно потокобезопасной (`thread-safe`), так как к ней будут обращаться сотни параллельных HTTP-запросов одновременно.
2.  **Неявный вызов Dispose:** Никогда не вызывайте метод `Dispose()` вручную на объектах, которые вы получили через DI-конструктор. Контейнер создал их — контейнер сам их и утилизирует.
3.  **Scoped для транзакционных данных:** Все, что работает с хранилищем данных на чтение/запись в рамках одного бизнес-транзакционного шага, регистрируйте как `Scoped`.
4.  **Использование IServiceScopeFactory:** В фоновых задачах, синглтон-обработчиках очередей или SignalR-хабах всегда создавайте явный `using var scope = scopeFactory.CreateScope()` для работы со Scoped-ресурсами.

> [!TIP] Нюанс: Поведение HttpClient в DI
> Регистрация `HttpClient` напрямую как `Transient` или `Singleton` — плохая практика (вызывает исчерпание сокетов ОС или игнорирование изменений DNS). Используйте `builder.Services.AddHttpClient()`, который регистрирует `IHttpClientFactory` как `Singleton`, а сами клиенты создает во временном scope с правильным управлением временем жизни базовых сокетов.