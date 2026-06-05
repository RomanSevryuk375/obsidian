---
aliases:
tags:
  - C_sharp
  - dotnet
  - design-patterns
  - oop
date: 2026-05-30 21:20
status:
---
### Суть (The "What")
**Pipeline Behaviors (поведения конвейера)** — это механизм в MediatR, реализующий концепцию аспектно-ориентированного программирования (AOP) через паттерны «Декоратор» и «Цепочка обязанностей». Они позволяют перехватывать входящие запросы (`IRequest<T>`) до того, как они попадут в соответствующий обработчик (`IRequestHandler`), и выполнять сквозную логику до, после или вместо его работы.

Этот паттерн решает задачу **очистки обработчиков от инфраструктурного кода (Boilerplate)**. Вместо того чтобы в каждом обработчике Use Case писать код для логирования, валидации входных параметров, кэширования или управления транзакциями БД, вся эта сквозная логика выносится в изолированные "поведения", работающие для всех запросов автоматически.

---

### Как это работает (The "How" / Nutshell style)

#### 1. Интерфейс IPipelineBehavior\<TRequest, TResponse\>
Каждое поведение реализует этот интерфейс и определяет метод `Handle`:
*   `TRequest request`: Перехватываемый объект запроса.
*   `RequestHandlerDelegate<TResponse> next`: Делегат, указывающий на следующий шаг конвейера (это может быть следующее поведение или сам финальный обработчик).
*   `CancellationToken cancellationToken`: [[Cancellation Token|Токен отмены операции]].

#### 2. Открытые обобщения (Open Generics)
Поведения регистрируются как открытые обобщенные типы (например, `LoggingBehavior<,>`). Это позволяет им автоматически перехватывать абсолютно любые запросы в приложении. 
С помощью [[Generics|generic]]-ограничений (`where TRequest : ...`) можно сузить область действия поведения. Например, применять транзакционное поведение только к командам изменения данных, реализующим маркерный интерфейс `ICommand`.

#### 3. Короткое замыкание (Short-Circuiting)
Если поведение принимает решение не вызывать `await next()`, выполнение конвейера прерывается. 
*   **Пример с кэшированием:** Поведение проверяет наличие данных в Redis. Если данные найдены, оно сразу возвращает их, а финальный обработчик (запрос к БД) даже не запускается.
*   **Пример с валидацией:** Если данные не прошли проверку, поведение выбрасывает исключение, не допуская запуск обработчика с некорректными параметрами.

---

### Структура папок и пример реализации (C# 12)

Реализация автоматического поведения валидации запросов с использованием популярной библиотеки [[FluentValidation]] и первичных конструкторов [[C#]] 12:

```text
Ordering.Application/
├── Common/
│   └── Behaviors/
│       └── ValidationBehavior.cs    # Перехватчик (Primary Constructor)
└── Orders/
    └── Commands/
        └── CreateOrder/
            ├── CreateOrderCommand.cs
            ├── CreateOrderCommandValidator.cs
            └── CreateOrderCommandHandler.cs
```

#### 1. Реализация Pipeline Behavior (ValidationBehavior.cs)
Поведение перехватывает запросы, находит для них зарегистрированные валидаторы и выполняет проверку до запуска обработчика:

```csharp
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;
using FluentValidation;
using MediatR;

namespace Ordering.Application.Common.Behaviors;

// Открытый дженерик. Перехватывает любые запросы TRequest, возвращающие TResponse
public class ValidationBehavior<TRequest, TResponse>(IEnumerable<IValidator<TRequest>> validators) 
    : IPipelineBehavior<TRequest, TResponse> where TRequest : notnull
{
    private readonly IEnumerable<IValidator<TRequest>> _validators = validators;

    public async Task<TResponse> Handle(
        TRequest request, 
        RequestHandlerDelegate<TResponse> next, 
        CancellationToken cancellationToken)
    {
        if (_validators.Any())
        {
            var context = new ValidationContext<TRequest>(request);
            
            // Запуск всех валидаторов для данного запроса параллельно
            var validationResults = await Task.WhenAll(
                _validators.Select(v => v.ValidateAsync(context, cancellationToken))
            );

            var failures = validationResults
                .SelectMany(r => r.Errors)
                .Where(f => f != null)
                .ToList();

            if (failures.Count != 0)
            {
                // Прерываем конвейер выбросом исключения. Код обработчика не выполнится.
                throw new ValidationException(failures);
            }
        }

        // Передаем управление дальше по конвейеру (к следующему поведению или обработчику)
        return await next();
    }
}
```

#### 2. Регистрация в Program.cs (C# 12 / MediatR 12+)
В современных версиях MediatR регистрация открытых поведений выполняется через метод `AddOpenBehavior`:

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddMediatR(cfg => {
    cfg.RegisterServicesFromAssembly(typeof(Program).Assembly);
    
    // Регистрация поведения. Порядок вызова AddOpenBehavior определяет порядок в конвейере!
    cfg.AddOpenBehavior(typeof(ValidationBehavior<,>));
});

// Регистрация всех валидаторов FluentValidation из сборки
builder.Services.AddValidatorsFromAssembly(typeof(Program).Assembly);
```

---

### Ошибки и Best Practices

> [!DANGER] Anti-pattern: Нарушение порядка регистрации (Order Violation)
> Порядок, в котором вы регистрируете поведения, определяет порядок их выполнения. Ошибка в очередности может сломать логику. 
> **Правильный порядок:** 
> 1. Логирование (должно быть самым внешним, чтобы логировать ошибки других слоев). 
> 2. Обработка ошибок (try-catch поведение). 
> 3. Валидация (должна проверить данные до открытия транзакции). 
> 4. Транзакции (открывает Unit of Work непосредственно перед бизнес-логикой).

> [!WARNING] Ошибка: Чрезмерная нагрузка на кэширование / логирование
> Будьте осторожны с логированием всего тела запроса в `LoggingBehavior` на открытых дженериках. Запросы могут содержать конфиденциальные данные (пароли, номера карт) или тяжелые бинарные файлы (фотографии), логирование которых забьет диски сервера и нарушит правила безопасности (GDPR).

**Best Practices:**
1.  **Ограничение поведений (Interface Constraints):** Не запускайте транзакционное поведение для запросов на чтение (Queries). Создайте интерфейс-маркер `ICommand` и ограничьте поведение: `where TRequest : ICommand`.
2.  **Асинхронность:** Всегда делайте вызовы внутри поведений асинхронными и пробрасывайте `cancellationToken`.
3.  **Избегайте изменения запроса:** Поведения не должны модифицировать данные внутри объекта `request`. Запрос должен оставаться иммутабельным.
4.  **Короткое замыкание для кэша:** При реализации поведения кэширования возвращайте закэшированный результат напрямую, не вызывая `next()`.

> [!TIP] Нюанс: Производительность
> Каждый зарегистрированный `IPipelineBehavior` добавляет уровень вложенности в вызове методов (кадр стека). Если в приложении настроено 10-15 поведений на каждый запрос, это может незначительно увеличить накладные расходы на работу со стеком в критических к производительности участках. Старайтесь держать конвейер компактным (3-5 ключевых поведений).