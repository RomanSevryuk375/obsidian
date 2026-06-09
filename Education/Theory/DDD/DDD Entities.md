---
aliases:
tags:
  - C_sharp
  - architecture
  - dotnet
  - ddd
date: 2026-06-09 14:18
status:
---
### Суть (The "What")
**Сущность (Entity)** — это объект предметной области, который определяется своим уникальным идентификатором (Identity), а не набором значений его атрибутов. 

Сущности решают задачу **отслеживания жизненного цикла бизнес-объекта во времени**. Даже если у клиента (Customer) изменятся имя, адрес и номер телефона, с точки зрения системы это останется тот же самый клиент, пока неизменен его идентификатор (например, `Guid`).

---

### Как это работает (The "How" / Nutshell style)

#### 1. Идентичность и переопределение эквивалентности
По умолчанию ссылочные типы (`class`) в [[MOC|C#]] сравниваются по ссылочной идентичности (адресу в управляемой куче). В [[DDD (Domain-Driven Design)||DDD]] логика сравнения сущностей должна базироваться **исключительно на их идентификаторах**.
*   **Механика:** Разработчик должен переопределить виртуальные методы `Equals(object)` и `GetHashCode()`, а также операторы `==` и `!=`. 
*   **Оптимизация:** При переопределении `Equals` на низком уровне сначала используется `object.ReferenceEquals(this, other)` (быстрая проверка указателей), и только если объекты в разных участках памяти, происходит сравнение их ID.

#### 2. Богатая модель поведения (Rich Domain Model)
С точки зрения [[ООП]] сущность инкапсулирует и данные (состояние), и поведение (бизнес-логику). 
*   Данные защищены модификатором `private set` или `protected set`.
*   Состояние изменяется только через вызов бизнес-методов, имена которых соответствуют **Единому языку (Ubiquitous Language)** (например, `ActivateAccount()`, а не `IsActive = true`).

#### 3. Базовый абстрактный класс
Для устранения дублирования ([[DRY]]) инфраструктурного кода (сравнение ID, хранение доменных событий) в проектах на .NET всегда создается базовый класс `Entity`.

> [!INFO] Нюанс: Тип идентификатора
> В высоконагруженных распределенных системах для `Id` предпочтительнее использовать `Guid` (или более современные `Ulid` / `Snowflake` для упорядочивания в БД), генерируемые прямо в коде приложения. Использование автоинкремента БД (`int IDENTITY`) заставляет домен зависеть от инфраструктурного слоя, так как объект не получает свой ID до вызова `SaveChanges()`.

---

### Пример кода ([[MOC|C#]] 12)

```csharp
using System;
using System.Collections.Generic;

// 1. Базовый класс для всех сущностей домена
public abstract class Entity : IEquatable<Entity>
{
    public Guid Id { get; protected init; }

    public override bool Equals(object? obj) => obj is Entity entity && Equals(entity);

    public bool Equals(Entity? other)
    {
        if (other is null) return false;
        if (ReferenceEquals(this, other)) return true; // Оптимизация CLR
        if (GetType() != other.GetType()) return false;
        
        return Id == other.Id;
    }

    public override int GetHashCode() => Id.GetHashCode();

    public static bool operator ==(Entity? left, Entity? right) => left?.Equals(right) ?? right is null;
    public static bool operator !=(Entity? left, Entity? right) => !(left == right);
}

// 2. Конкретная сущность с инкапсулированной логикой
public class Customer : Entity
{
    // Инкапсуляция коллекции (защита от прямого добавления через List.Add)
    private readonly List<string> _deliveryAddresses = [];

    // Приватные сеттеры защищают инварианты
    public string FullName { get; private set; }
    public bool IsActive { get; private set; }

    // Публичный доступ к коллекции только для чтения
    public IReadOnlyCollection<string> DeliveryAddresses => _deliveryAddresses.AsReadOnly();

    // Защищенный конструктор без параметров необходим для ORM (например, EF Core)
    protected Customer() { }

    private Customer(Guid id, string fullName)
    {
        Id = id;
        FullName = fullName;
        IsActive = true;
    }

    // Фабричный метод (Factory Method) для контролируемого создания
    public static Customer Create(string fullName)
    {
        // Использование C# ArgumentException helpers
        ArgumentException.ThrowIfNullOrWhiteSpace(fullName);

        return new Customer(Guid.NewGuid(), fullName);
    }

    // Бизнес-метод (Поведение)
    public void Deactivate(string reason)
    {
        if (!IsActive) throw new InvalidOperationException("Customer is already inactive.");
        
        ArgumentException.ThrowIfNullOrWhiteSpace(reason);
        IsActive = false;
        
        // Здесь можно опубликовать доменное событие: CustomerDeactivatedDomainEvent
    }

    public void AddAddress(string address)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(address);
        _deliveryAddresses.Add(address);
    }
}
```

---

### Ошибки и Best Practices

> [!DANGER] Anti-pattern: Анемичная модель (Anemic Domain Model)
> Это самая распространенная ошибка при переходе на DDD. Сущность, состоящая исключительно из публичных свойств `{ get; set; }` без методов бизнес-логики, является не доменной моделью, а простым DTO. Вся логика валидации и изменения состояния в таком случае "утекает" в сервисы (Services), превращая ООП в процедурное программирование.

> [!WARNING] Ошибка: Сравнение транзитных (Transient) сущностей
> Если сущность только создана и еще не получила ID (например, ID равен 0 при использовании базы данных), сравнение `entityA == entityB` даст `true`, даже если это разные объекты. Базовый класс `Entity` должен учитывать этот нюанс, либо вы должны генерировать `Guid` сразу в конструкторе (рекомендуемый путь).

**Best Practices:**
1.  **Контроль инвариантов:** Сущность никогда не должна находиться в невалидном состоянии. Все проверки должны осуществляться внутри нее самой (в конструкторе или методах). Если вызывается `Customer.Create("")`, должно немедленно вылететь исключение.
2.  **Защита коллекций:** Никогда не выставляйте `List<T>` наружу через публичные свойства. Внешний код может вызвать `.Clear()`, разрушив состояние агрегата. Используйте резервное поле (backing field) и возвращайте `IReadOnlyCollection<T>`.
3.  **ORM-независимость:** Не добавляйте в сущности атрибуты Entity Framework Core (например, `[Table]`, `[Column]`). Настройка маппинга базы данных должна происходить в инфраструктурном слое через Fluent API (интерфейс `IEntityTypeConfiguration<T>`).
4.  **Фабричные методы:** Используйте статические методы `Create()` вместо публичных конструкторов, особенно если создание объекта требует сложной валидации или генерации доменных событий.

> [!TIP] Нюанс: Ссылки между сущностями
> В чистом DDD сущности из разных агрегатов ([[Aggregates]]) не должны хранить прямые ссылки на объекты друг друга (например, `public Order LastOrder { get; }`). Они должны ссылаться друг на друга строго по идентификаторам (`public Guid LastOrderId { get; }`). Это уменьшает связность и упрощает сохранение в базу данных.