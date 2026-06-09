---
aliases:
tags:
  - C_sharp
  - dotnet
  - architecture
  - ddd
date: 2026-06-09 14:44
status:
---
### Суть (The "What")
**Агрегат (Aggregate)** — это кластер логически связанных сущностей ([[DDD Entities|Entities]]) и объектов-значений ([[Value Objects]]), которые рассматриваются как единое целое с точки зрения изменения данных. Каждый агрегат имеет одну главную сущность — **Корень агрегации (Aggregate Root)**.

Агрегаты решают задачу **обеспечения согласованности данных (Data Consistency)** и защиты бизнес-правил (инвариантов). Они определяют строгие транзакционные границы: любые изменения внутри агрегата должны происходить и сохраняться в базу данных атомарно, как одна операция.

---

### Как это работает (The "How" / Nutshell style)

#### 1. Инкапсуляция и [[Aggregate Roots]]
Корень агрегации выполняет роль "стража" (Gatekeeper) для всех своих внутренних элементов.
*   Внешний код (контроллеры, сервисы, другие агрегаты) имеет право хранить ссылки (указатели в памяти) **только на Корень агрегации**. 
*   Внешний код не может напрямую обращаться к внутренним сущностям агрегата для их изменения. Все запросы на мутацию должны проходить через публичные методы Корня.

#### 2. Транзакционные границы
На уровне инфраструктуры (например, при использовании Entity Framework Core) один агрегат = одна транзакция БД. 
Если вам нужно изменить данные в двух разных агрегатах, вы не должны делать это в одной синхронной транзакции (в чистом [[DDD (Domain-Driven Design)||DDD]]). Вместо этого первый агрегат публикует **Доменное событие ([[Domain Event]])**, а второй агрегат асинхронно реагирует на него (Eventual Consistency).

#### 3. Память и Жизненный цикл ([[CLR]])
С точки зрения управляемой кучи ([[Managed Heap]]) .NET, агрегат — это граф объектов. 
*   Корень агрегации удерживает сильные ссылки (strong references) на свои дочерние элементы. 
*   Если Корень агрегации становится недоступным для корней приложения ([[GC]] Roots), сборщик мусора (Garbage Collector) уничтожит весь агрегат целиком. Это гарантирует, что "осиротевшие" дочерние сущности не останутся висеть в памяти.

#### 4. Оптимистичная блокировка (Concurrency Control)
Поскольку агрегат меняется как единое целое, конфликты параллельного доступа (когда два потока пытаются изменить один агрегат) обычно разрешаются на уровне Корня. В .NET это реализуется через поле `RowVersion` (или `ConcurrencyToken`), которое инкрементируется при любом изменении внутреннего состояния агрегата.

---

### Пример кода ([[MOC|C#]] 12)

Реализация агрегата "Заказ", который инкапсулирует коллекцию "Строки заказа" и защищает бизнес-инвариант (ограничение максимальной суммы).

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

namespace Domain.Aggregates.OrderAggregate;

// 1. Внутренняя сущность агрегата.
// Используем Primary Constructor (C# 12).
// Обратите внимание: модификатор доступа internal (или private/protected), 
// чтобы внешний код не мог создавать эти объекты напрямую.
public class OrderLine(Guid id, Guid productId, decimal price, int quantity)
{
    public Guid Id { get; } = id;
    public Guid ProductId { get; } = productId;
    public decimal Price { get; private set; } = price;
    public int Quantity { get; private set; } = quantity;

    internal void AddQuantity(int additionalQuantity) => Quantity += additionalQuantity;
}

// 2. Корень агрегации (Aggregate Root)
public class Order
{
    // Защищенное хранилище дочерних сущностей
    private readonly List<OrderLine> _orderLines = [];

    public Guid Id { get; private set; }
    public Guid CustomerId { get; private set; }
    
    // Публичный доступ только для чтения
    public IReadOnlyCollection<OrderLine> OrderLines => _orderLines.AsReadOnly();

    public decimal TotalAmount => _orderLines.Sum(x => x.Price * x.Quantity);

    // Бизнес-правило (Инвариант)
    private const decimal MaxOrderTotal = 100_000m;

    private Order(Guid id, Guid customerId)
    {
        Id = id;
        CustomerId = customerId;
    }

    public static Order Create(Guid customerId)
    {
        ArgumentException.ThrowIfNullOrEmpty(customerId.ToString());
        return new Order(Guid.NewGuid(), customerId);
    }

    // Единственный способ добавить товар в заказ - через Корень агрегации
    public void AddProduct(Guid productId, decimal price, int quantity)
    {
        if (price <= 0) throw new ArgumentException("Цена должна быть больше нуля.");
        
        var potentialNewTotal = TotalAmount + (price * quantity);
        if (potentialNewTotal > MaxOrderTotal)
        {
            throw new InvalidOperationException($"Сумма заказа не может превышать {MaxOrderTotal}.");
        }

        var existingLine = _orderLines.SingleOrDefault(x => x.ProductId == productId);
        if (existingLine != null)
        {
            // Изменение внутренней сущности скрыто внутри Корня
            existingLine.AddQuantity(quantity);
        }
        else
        {
            _orderLines.Add(new OrderLine(Guid.NewGuid(), productId, price, quantity));
        }
    }
}
```

---

### Ошибки и Best Practices

> [!DANGER] Anti-pattern: Связь через прямые ссылки
> Никогда не храните в одном агрегате ссылку на другой агрегат (например, `public Customer Customer { get; }` внутри класса `Order`). Это размывает границы транзакций и заставляет ORM выгружать из базы половину системы. Храните **только идентификаторы** других агрегатов (например, `public Guid CustomerId { get; }`).

> [!WARNING] Ошибка: Гигантские агрегаты (God Aggregate)
> Попытка засунуть в один агрегат `User`, `Posts`, `Comments` и `Likes`. При любом изменении комментария вам придется блокировать всего пользователя, что убьет производительность и вызовет массу исключений параллельного доступа (Concurrency Exceptions). Агрегат должен быть минимально возможного размера, достаточного для сохранения инвариантов.

**Best Practices:**
1.  **Защита коллекций:** Всегда прячьте `List<T>` за `IReadOnlyCollection<T>`. Если вы отдадите наружу `IList<T>`, внешний код сможет вызвать `Order.OrderLines.Clear()`, обойдя вашу бизнес-логику и нарушив правила агрегата.
2.  **Удаление через корень:** Если нужно удалить строку заказа, метод также должен находиться в Корне: `order.RemoveProduct(productId)`.
3.  **Маркерные интерфейсы:** Используйте пустой интерфейс `IAggregateRoot` (или базовый класс), чтобы помечать корни агрегации. Настройте архитектуру (например, паттерн Repository) так, чтобы Репозитории могли работать **только** с классами, реализующими `IAggregateRoot`.
4.  **Справочные данные (Reference Data):** Допустимо хранить ссылки на неизменяемые объекты (справочники, Value Objects) из других агрегатов, но не на сущности, состояние которых может измениться.

> [!TIP] Нюанс: EF Core и Агрегаты
> В Entity Framework Core для маппинга скрытых полей (как `_orderLines`) используется Fluent API: `builder.Metadata.FindNavigation(nameof(Order.OrderLines)).SetPropertyAccessMode(PropertyAccessMode.Field);`. Это позволяет ORM материализовать данные из БД прямо в приватное поле, минуя ваши бизнес-проверки в момент загрузки из базы.