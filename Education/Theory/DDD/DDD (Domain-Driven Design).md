---
aliases:
tags:
  - C_sharp
  - architecture
  - dotnet
  - oop
  - ddd
date: 2026-06-09 13:58
status:
---
### Суть (The "What")
**Domain-Driven Design (проблемно-ориентированное проектирование)** — это методология проектирования сложного программного обеспечения, основанная на глубоком моделировании предметной области и тесном сотрудничестве технических специалистов с экспертами бизнеса (Domain Experts).

DDD решает задачу **управления сложностью бизнес-логики**. Он переносит фокус с технологий (баз данных, API, фреймворков) на саму бизнес-модель. С помощью построения **Единого языка (Ubiquitous Language)** DDD устраняет разрыв между тем, как бизнес формулирует задачи, и тем, как они реализованы в коде.

[[Правила DDD]]

---

### Как это работает (The "How" / Nutshell style)

#### 1. Стратегическое проектирование
*   **[[Bounded Context]] (Ограниченный контекст):** Разделение большой предметной области на изолированные подсистемы. Внутри каждого контекста действует своя модель данных и свой Единый язык. Например, понятие `Product` в контексте Продаж — это цена и описание, а в контексте Доставки — это вес и габариты.
*   **[[Context Mapping]] (Карта контекстов):** Описание связей и способов интеграции между ограниченными контекстами.

#### 2. Тактическое проектирование (Кодовые паттерны в .NET)
*   **[[DDD Entities]] (Сущности):** Объекты, обладающие уникальным идентификатором (Identity), который не меняется на протяжении всей их жизни (например, `Order` с `Guid Id`). Сравнение сущностей идет строго по их ID.
*   **[[Value Objects]] (Объекты-значения):** Неизменяемые (immutable) объекты без идентификатора, определяемые исключительно набором своих свойств (например, `Address` или `Money`). Сравнение идет по значениям полей. 
*   **[[Aggregates]] & [[Aggregate Roots]] (Агрегаты и корни агрегации):** Кластер связанных сущностей и объектов-значений, рассматриваемых как единое целое. Корень агрегации (Aggregate Root) — это "страж" (gatekeeper) границ агрегата. Внешний код не имеет права менять элементы внутри агрегата напрямую, только через методы Корня, который контролирует доменные инварианты (бизнес-правила целостности).
*   **[[Domain Events]] (Доменные события):** Факты, произошедшие в домене, важные для бизнеса (например, `OrderPaidDomainEvent`). Используются для синхронизации состояния между разными агрегатами без создания жестких связей.

> [!INFO] Архитектурное правило: Чистый Domain
> Доменный слой должен быть абсолютно чистым от инфраструктурных зависимостей (Entity Framework, базы данных, [[HTTP]]-клиенты, сериализаторы). Сущности должны быть чистыми C# классами ([[POCO]] — Plain Old CLR Objects), содержащими только бизнес-логику.

---

### Структура папок и пример реализации ([[MOC|C#]] 12)

Рекомендуемая структура папок для доменного слоя (Domain Layer) микросервиса:

```text
Ordering.Domain/
├── Aggregates/
│   └── OrderAggregate/
│       ├── Order.cs                 # Корень агрегации (Entity)
│       ├── OrderLine.cs             # Внутренняя сущность агрегата
│       └── Address.cs               # Объект-значение (Value Object)
├── Events/
│   └── OrderPlacedDomainEvent.cs    # Доменное событие (Record)
└── Exceptions/
    └── InvalidOrderAmountException.cs
```

#### 1. Объект-значение и Доменное событие (C# 12)
Используем `readonly record struct` для объектов-значений — это дает нулевые аллокации в куче и автоматическое сравнение по значениям полей:

```csharp
using System;
using MediatR;

namespace Ordering.Domain.Aggregates.OrderAggregate;

// Value Object: абсолютно неизменяем, сравнивается по значению полей
public readonly record struct Address(string Street, string City, string ZipCode);

// Domain Event: маркерный контракт для сигнализации в памяти приложения
public record OrderPlacedDomainEvent(Guid OrderId, Guid CustomerId, decimal TotalAmount) : INotification;
```

#### 2. Корень агрегации (Order.cs)
Реализует **богатую модель поведения (Rich Domain Model)**: приватные сеттеры, валидация при создании, инкапсуляция коллекции и публикация доменных событий:

```csharp
using System;
using System.Collections.Generic;
using Ordering.Domain.Events;

namespace Ordering.Domain.Aggregates.OrderAggregate;

public class Order : Entity
{
    private readonly List<OrderLine> _orderLines = [];
    private readonly List<INotification> _domainEvents = [];

    // Приватные сеттеры защищают поля от неконтролируемого изменения извне
    public Guid CustomerId { get; private set; }
    public Address ShippingAddress { get; private set; }
    public decimal TotalAmount { get; private set; }
    public IReadOnlyCollection<OrderLine> OrderLines => _orderLines.AsReadOnly();
    public IReadOnlyCollection<INotification> DomainEvents => _domainEvents.AsReadOnly();

    // Приватный конструктор заставляет использовать фабричный метод
    private Order(Guid id, Guid customerId, Address shippingAddress)
    {
        Id = id;
        CustomerId = customerId;
        ShippingAddress = shippingAddress;
    }

    // Фабричный метод гарантирует, что объект невозможно создать в невалидном состоянии
    public static Order Create(Guid customerId, Address shippingAddress)
    {
        if (customerId == Guid.Empty) 
            throw new ArgumentException("Customer ID cannot be empty", nameof(customerId));

        var order = new Order(Guid.NewGuid(), customerId, shippingAddress);

        // Фиксируем доменное событие
        order.AddDomainEvent(new OrderPlacedDomainEvent(order.Id, customerId, 0));

        return order;
    }

    public void AddOrderLine(Guid productId, decimal price, int quantity)
    {
        // Контроль инварианта: мы не можем добавить товар с отрицательной ценой
        if (price <= 0) throw new ArgumentOutOfRangeException(nameof(price), "Price must be positive");
        if (quantity <= 0) throw new ArgumentOutOfRangeException(nameof(quantity), "Quantity must be positive");

        var line = new OrderLine(Guid.NewGuid(), productId, price, quantity);
        _orderLines.Add(line);

        // Пересчет общей суммы контролируется исключительно внутри агрегата
        TotalAmount += price * quantity;
    }

    private void AddDomainEvent(INotification domainEvent) => _domainEvents.Add(domainEvent);
    public void ClearDomainEvents() => _domainEvents.Clear();
}

// Базовый класс для всех сущностей
public abstract class Entity
{
    public Guid Id { get; protected set; }
}

// Заглушка внутренней сущности для компиляции
public class OrderLine(Guid id, Guid productId, decimal price, int quantity) : Entity
{
    public Guid ProductId { get; } = productId;
    public decimal Price { get; } = price;
    public int Quantity { get; } = quantity;
}
```

---

### Ошибки и Best Practices

> [!DANGER] Anti-pattern: Анемичная модель данных (Anemic Domain Model)
> Ситуация, когда сущности домена содержат только автосвойства `{ get; set; }` и не несут никакой логики, а все бизнес-правила и проверки размазаны по внешним сервисам. Это превращает ООП в процедурный стиль. В DDD сущности домена должны обладать "богатым" поведением (Rich Domain Model), инкапсулируя свои данные и методы работы с ними.

> [!WARNING] Ошибка: Прямая работа с внутренностями агрегата
> Внешний код (например, репозиторий или контроллер) никогда не должен напрямую добавлять элементы в коллекцию `OrderLines` или менять свойства `OrderLine`. Все изменения дочерних элементов агрегата должны выполняться строго через методы Корня агрегации (`Order.AddOrderLine()`).

**Best Practices:**
1.  **Value Objects для всего:** Не используйте примитивные типы (`string`, `decimal`) для бизнес-понятий. Если у вас есть номер телефона — создайте `Phone` (Value Object) с валидацией формата. Если есть деньги — создайте `Money` (Value Object), хранящий сумму и валюту. Это защищает от некорректных присвоений.
2.  **Защита конструкторов:** Делайте конструкторы сущностей закрытыми (`private` или `protected`). Для создания объектов предоставляйте статические фабричные методы (`Create()`), которые возвращают полностью валидный объект.
3.  **Доменные события вместо вызова сервисов:** Если при создании заказа нужно отправить письмо, не внедряйте `IEmailSender` в сущность `Order`. Сущность должна опубликовать событие `OrderPlacedDomainEvent`, которое перехватит и обработает инфраструктурный уровень.
4.  **Репозитории только для Агрегатов:** В слое данных репозитории должны существовать только для Корней агрегации (`Order`). Не создавайте отдельные репозитории для `OrderLine`.

> [!TIP] Оптимизация EF Core и Value Objects
> При использовании Entity Framework Core объекты-значения (`Value Objects`), реализованные как `readonly record struct`, можно маппить в ту же таблицу, что и сущность, с помощью метода `.OwnsOne()` в конфигурациях Fluent API. Это позволяет сохранить красивую архитектуру в коде, не создавая лишних связей на уровне таблиц БД.