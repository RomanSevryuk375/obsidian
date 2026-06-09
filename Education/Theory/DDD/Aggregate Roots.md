---
aliases:
tags:
  - C_sharp
  - dotnet
  - ddd
  - architecture
date: 2026-06-09 14:55
status:
---
### Суть (The "What")
**Корень агрегации (Aggregate Root, AR)** — это главная сущность ([[DDD Entities|Entity]]) внутри кластера объектов (Агрегата), которая выступает в качестве **единственной глобально доступной точки входа** для взаимодействия с этим кластером. 

Он решает задачу **защиты инвариантов (бизнес-правил)**. Если разрешить внешнему коду произвольно менять дочерние элементы агрегата, система быстро придет в несогласованное состояние. Корень агрегации берет на себя роль "фасада" и "стража", гарантируя, что любая мутация данных внутри его границ пройдет все необходимые проверки.

---

### Как это работает (The "How" / Nutshell style)

#### 1. Единственная точка взаимодействия (Single Point of Entry)
Внешний код (сервисы приложения, обработчики команд) имеет право удерживать ссылки только на Корень агрегации.
*   Корень имеет глобальную идентичность (Global Identity), например, `Guid`, который используется для поиска в БД.
*   Внутренние сущности (дочерние) могут иметь локальную идентичность (Local Identity), которая имеет смысл только внутри данного агрегата. Внешний мир не может запросить внутреннюю сущность из базы данных напрямую.

#### 2. Связь с Репозиториями
В архитектуре [[DDD (Domain-Driven Design)|DDD]] существует строгое правило: **один Корень агрегации = один Репозиторий**. 
Вы не имеете права создавать `IOrderLineRepository` для изменения строк заказа. Вы должны запросить через `IOrderRepository` сам `Order` (Корень), вызвать у него метод изменения строк, а затем сохранить весь `Order` целиком.

#### 3. Накопитель доменных событий ([[Domain Events]] Container)
В реализациях DDD на платформе .NET Корень агрегации обычно наследуется от специального базового класса, который содержит коллекцию `DomainEvents`. 
*   **Механика:** Когда внутри агрегата происходит важное изменение, Корень не отправляет событие в шину сразу. Он сохраняет его в свою внутреннюю коллекцию в памяти (в куче).
*   **Инфраструктура:** При вызове `DbContext.SaveChanges()` в Entity Framework Core, инфраструктурный код извлекает эти события из Корня и публикует их (например, через [[MediatR]]), обеспечивая атомарность транзакции БД и рассылки событий.

#### 4. [[CLR]] и Ссылочная целостность
На уровне CLR Корень агрегации владеет графом объектов. Он удерживает **сильные ссылки (strong references)** на дочерние объекты-значения и сущности. Как только Корень теряет связь с корнями приложения ([[GC Roots]]), весь агрегат атомарно помечается как мусор и уничтожается сборщиком мусора (Garbage Collector).

---

### Пример кода ([[C#]] 12)

Реализация Корня агрегации "Корзина покупателя" с использованием маркерных интерфейсов и механизмов работы с доменными событиями.

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

// 1. Маркерный интерфейс для ограничения Репозиториев
public interface IAggregateRoot { }

// 2. Инфраструктурный базовый класс для корней агрегации
public abstract class AggregateRoot : IAggregateRoot
{
    public Guid Id { get; protected set; }
    
    // Коллекция для хранения бизнес-событий в памяти до фиксации транзакции
    private readonly List<object> _domainEvents = [];
    public IReadOnlyCollection<object> DomainEvents => _domainEvents.AsReadOnly();

    protected void AddDomainEvent(object domainEvent) => _domainEvents.Add(domainEvent);
    public void ClearDomainEvents() => _domainEvents.Clear();
}

// 3. Внутренняя сущность (не доступна снаружи для прямого создания)
public class CartItem(Guid productId, decimal price, int quantity)
{
    public Guid ProductId { get; } = productId;
    public decimal Price { get; } = price;
    public int Quantity { get; private set; } = quantity;

    internal void UpdateQuantity(int quantity) => Quantity = quantity;
}

// 4. Сам Корень агрегации
public class ShoppingCart : AggregateRoot
{
    private readonly List<CartItem> _items = [];
    public Guid CustomerId { get; private set; }
    
    public IReadOnlyCollection<CartItem> Items => _items.AsReadOnly();
    
    public decimal TotalPrice => _items.Sum(i => i.Price * i.Quantity);

    private ShoppingCart(Guid id, Guid customerId)
    {
        Id = id;
        CustomerId = customerId;
    }

    // Фабричный метод
    public static ShoppingCart Create(Guid customerId)
    {
        var cart = new ShoppingCart(Guid.NewGuid(), customerId);
        // Записываем событие в коллекцию Корня
        cart.AddDomainEvent(new { EventName = "CartCreated", CustomerId = customerId });
        return cart;
    }

    // Внешний код просит Корень изменить состояние.
    // Корень сам находит дочерний элемент и меняет его.
    public void ChangeItemQuantity(Guid productId, int newQuantity)
    {
        if (newQuantity <= 0) 
            throw new ArgumentException("Количество должно быть больше нуля");

        var item = _items.SingleOrDefault(i => i.ProductId == productId) 
            ?? throw new InvalidOperationException("Товар не найден в корзине");

        item.UpdateQuantity(newQuantity);

        // Инвариант: оповещаем систему, если корзина стала слишком дорогой
        if (TotalPrice > 100_000m)
        {
            AddDomainEvent(new { EventName = "HighValueCartDetected", CartId = Id });
        }
    }
}
```

---

### Ошибки и Best Practices

> [!DANGER] Anti-pattern: Repository для дочерних сущностей
> Создание `ICartItemRepository` и попытка изменить `CartItem` напрямую в обход `ShoppingCart`. Это уничтожает саму суть Корня агрегации: он больше не может контролировать общую сумму корзины (`TotalPrice`) и гарантировать выполнение инвариантов. Используйте репозитории **только** для классов, реализующих `IAggregateRoot`.

> [!WARNING] Ошибка: Раздутый Корень (Bloated Root)
> Если в вашем Корне агрегации 50 методов и 20 списков дочерних сущностей, вы, вероятно, смоделировали половину системы как один Агрегат. При параллельных запросах такой Корень будет постоянно вызывать исключения параллелизма (Concurrency Exceptions) в базе данных. Корень должен быть максимально компактным.

**Best Practices:**
1.  **Ссылки по ID:** Корень агрегации не должен хранить прямые ссылки (указатели) на другие Корни агрегации. Если Корзине (`ShoppingCart`) нужен Клиент (`Customer`), она хранит только `CustomerId` (`Guid`), а не `public Customer Customer { get; }`.
2.  **Оптимистичная блокировка:** Всегда добавляйте в Корень агрегации поле для контроля версий (например, `[Timestamp] byte[] RowVersion` для EF Core). Это защитит данные при одновременной попытке изменения одного и того же Корня из разных потоков.
3.  **Инкапсуляция коллекций:** Корень всегда должен возвращать `IReadOnlyCollection<T>` или `IEnumerable<T>`, чтобы внешний код не мог вызвать метод `.Add()` у коллекции в обход бизнес-логики Корня.
4.  **Удаление через Корень:** Если вам нужно удалить сущность из системы, вы загружаете Корень, удаляете дочерний элемент через его метод, а репозиторий при сохранении (EF Core Cascade Delete) удаляет запись из базы данных.