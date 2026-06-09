---
aliases:
tags:
  - C_sharp
  - dotnet
  - ddd
  - architecture
date: 2026-06-09 15:08
status:
---
### Суть (The "What")
**Доменное событие (Domain Event)** — это объект, фиксирующий факт того, что в предметной области (внутри конкретного Ограниченного контекста) произошло что-то важное для бизнеса.

Они решают задачу **снижения связности (decoupling) при реализации побочных эффектов**. Если при изменении статуса заказа нужно обновить статистику клиента и зарезервировать товар на складе, Корень агрегации "Заказ" не должен вызывать сервисы статистики и склада напрямую. Он просто порождает событие `OrderConfirmedDomainEvent`, на которое реагируют независимые обработчики.

---

### Как это работает (The "How" / Nutshell style)

#### 1. Доменные против Интеграционных
Критически важно разделять эти два типа событий:
*   **Доменные события (Domain Events):** Публикуются и обрабатываются **в памяти одного процесса** (In-Memory). Обработчики обычно выполняются в той же транзакции базы данных.
*   **Интеграционные события ([[Integration Events]]):** Покидают процесс и уходят в брокер сообщений ([[RabbitMQ]], Kafka) для уведомления *других* микросервисов. Они гарантируют итоговую согласованность (Eventual Consistency).

#### 2. Накопление в памяти ([[CLR]])
В чистой архитектуре [[DDD (Domain-Driven Design)|DDD]] сущности домена ничего не знают о [[MediatR]] или базах данных. 
*   С точки зрения среды выполнения (CLR), доменное событие — это просто объект в куче.
*   Корень агрегации ([[Aggregate Roots]]) содержит приватную коллекцию (`List<INotification>`), куда он складывает эти объекты при изменении своего состояния.

#### 3. Инфраструктурная диспетчеризация (Dispatching)
"Высвобождение" событий происходит на уровне инфраструктуры. Стандартный паттерн в .NET:
1.  Вызывается метод `SaveChangesAsync` в Entity Framework Core.
2.  Перехватчик (Interceptor) или переопределенный метод `SaveChanges` сканирует `ChangeTracker` на наличие измененных сущностей.
3.  Из этих сущностей извлекаются все доменные события (и сразу удаляются из списков).
4.  События публикуются через внутрипроцессную шину (например, `IMediator.Publish`).
5.  Только после завершения всех обработчиков транзакция фиксируется в БД (Commit).

> [!INFO] Нюанс: Границы транзакции
> Поскольку диспетчеризация происходит *до* фиксации транзакции в БД, любые изменения, сделанные обработчиками доменных событий (например, изменение другого агрегата в той же базе), будут сохранены атомарно вместе с оригинальным агрегатом.

---

### Пример кода ([[MOC|C#]] 12)

Реализация паттерна с использованием `record`, базового класса сущности и диспетчеризации через EF Core.

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;
using MediatR;
using Microsoft.EntityFrameworkCore;

// 1. Контракт доменного события (Record для иммутабельности)
public record OrderCanceledDomainEvent(Guid OrderId, string Reason) : INotification;

// 2. Базовый класс для сущностей
public abstract class Entity
{
    private readonly List<INotification> _domainEvents = [];
    public IReadOnlyCollection<INotification> DomainEvents => _domainEvents.AsReadOnly();

    protected void AddDomainEvent(INotification eventItem) => _domainEvents.Add(eventItem);
    public void ClearDomainEvents() => _domainEvents.Clear();
}

// 3. Корень агрегации
public class Order(Guid id) : Entity
{
    public Guid Id { get; } = id;
    public bool IsCanceled { get; private set; }

    public void Cancel(string reason)
    {
        if (IsCanceled) return;

        IsCanceled = true;
        // Сущность НЕ вызывает сервисы, она просто фиксирует факт
        AddDomainEvent(new OrderCanceledDomainEvent(Id, reason));
    }
}

// 4. Инфраструктурный слой: Диспетчеризация перед сохранением
public class AppDbContext(DbContextOptions<AppDbContext> options, IMediator mediator) : DbContext(options)
{
    private readonly IMediator _mediator = mediator;

    public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        // Извлекаем все сущности с событиями
        var domainEntities = ChangeTracker
            .Entries<Entity>()
            .Where(x => x.Entity.DomainEvents.Count != 0)
            .ToList();

        var domainEvents = domainEntities
            .SelectMany(x => x.Entity.DomainEvents)
            .ToList();

        // Очищаем события, чтобы не отправить их дважды
        foreach (var entity in domainEntities)
            entity.Entity.ClearDomainEvents();

        // Публикуем события ДО сохранения в БД
        foreach (var domainEvent in domainEvents)
            await _mediator.Publish(domainEvent, cancellationToken);

        // Сохраняем изменения (исходного агрегата и тех, что изменили обработчики)
        return await base.SaveChangesAsync(cancellationToken);
    }
}
```

---

### Ошибки и Best Practices

> [!DANGER] Anti-pattern: Внедрение IMediator в Сущности
> Никогда не передавайте `IMediator` или `IServiceProvider` в методы или конструкторы доменных сущностей (Корней агрегации) для немедленной отправки событий. Это нарушает принцип чистоты домена (Domain Purity). Домен должен только описывать правила, а не управлять инфраструктурой доставки сообщений.

> [!WARNING] Ошибка: Тяжелые операции в обработчиках доменных событий
> Если обработчик `OrderCanceledDomainEvent` будет делать долгий HTTP-запрос к внешнему API или отправлять email синхронно, весь вызов `SaveChangesAsync` будет заблокирован. Если HTTP-запрос упадет с ошибкой, транзакция БД откатится, и заказ не будет отменен. 
> **Решение:** Внешние вызовы должны выполняться через *Интеграционные события* (после успешного коммита транзакции) или через паттерн Outbox.

**Best Practices:**
1.  **Именование в прошедшем времени:** Доменное событие — это свершившийся факт. Используйте прошедшее время: `OrderCreated`, `PaymentFailed`, а не `CreateOrder` (это Команда).
2.  **Immutability:** Используйте `readonly record struct` или `record class` для доменных событий. Данные события не должны меняться ни при каких обстоятельствах.
3.  **Минимализм данных:** Передавайте в событии только необходимые данные (ID агрегата и измененные параметры). Не передавайте весь граф объектов, так как к моменту обработки события состояние агрегата могло измениться.
4.  **Разделение логики:** Используйте обработчики доменных событий только для изменения состояния других агрегатов внутри того же микросервиса (в рамках одной транзакции БД).