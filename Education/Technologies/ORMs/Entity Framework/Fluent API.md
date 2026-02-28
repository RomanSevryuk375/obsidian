---
aliases:
tags:
  - ORMs
  - dotnet
date: 2026-02-28 10:41
status:
---
**Fluent API** —  самый гибкий и мощный способ настройки сущностьей в [[Entity Framework]], который позволяет делать вещи, недоступные через атрибуты (например, составные ключи в ранних версиях, сложные связи или специфичные настройки [[SQL]]).

Весь код Fluent API пишется внутри метода **`OnModelCreating`** в классе [[DbContext]].

---

### Почему Fluent API лучше атрибутов?
1.  **[[Clean Architecture]] (Чистая архитектура):** Твои классы моделей (`User`, `Product`) остаются чистыми C# объектами [[POCO (Plain Old CLR Object)]]. Они не знают, что существует база данных, EF Core или SQL.
2.  **Больше возможностей:** Некоторые настройки (например, `OnDelete` [[поведение при удалении]] или [[составные первичные ключи]]) можно сделать **только** через Fluent API.
3.  **Централизация:** Вся конфигурация базы лежит в одном месте, а не размазана по десяткам файлов классов.

---

### Основной синтаксис

Все начинается с `modelBuilder.Entity<T>()`.
Цепочка вызовов читается как предложение на английском:

```csharp
modelBuilder.Entity<User>()      // "Для сущности User..."
    .Property(u => u.Name)       // "...выбери свойство Name..."
    .IsRequired()                // "...сделай его обязательным..."
    .HasMaxLength(200);          // "...и ограничь длину 200 символами."
```

---

### Топ-5 сценариев использования

#### 1. Составной первичный ключ ([[Composite Key]])

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    // Говорим: "Ключ состоит из двух полей"
    modelBuilder.Entity<UserProject>()
        .HasKey(up => new { up.UserId, up.ProjectId });
}
```

#### 2. Тонкая настройка связей ([[Relationships]])

Шаблон: **HasOne** (имеет один) -> **WithMany** (с многими) -> **HasForeignKey** (ключ).

```csharp
modelBuilder.Entity<Order>()
    .HasOne(o => o.Customer)        // У заказа один Клиент
    .WithMany(c => c.Orders)        // У Клиента много Заказов
    .HasForeignKey(o => o.ClientKey) // Используй поле ClientKey внешний ключ
    .OnDelete(DeleteBehavior.Cascade); // Если удалить Клиента, удалятся его Заказы
```

#### 3. Настройка значений по умолчанию (SQL Default)
Если ты хочешь, чтобы база данных (а не C#) подставляла дату создания:

```csharp
modelBuilder.Entity<Post>()
    .Property(p => p.CreatedAt)
    .HasDefaultValueSql("GETDATE()"); // SQL функция (зависит от БД)
```

#### 4. [[Индексы]] (Indexes)

```csharp
modelBuilder.Entity<User>()
    .HasIndex(u => u.Email)
    .IsUnique(); // Гарантирует, что Email не будет дублироваться в базе
```

#### 5. Игнорирование полей

```csharp
modelBuilder.Entity<User>()
    .Ignore(u => u.TemporaryData);
```

---

### Как организовать код (Best Practice)

Если у тебя 50 таблиц, метод `OnModelCreating` превратится в свалку на 1000 строк.
Чтобы этого избежать, используют конфигурационные классы `IEntityTypeConfiguration<T>`.

1. **Создаешь отдельный файл настройки:**
```csharp
public class BlogConfiguration : IEntityTypeConfiguration<Blog>
{
    public void Configure(EntityTypeBuilder<Blog> builder)
    {
        builder.ToTable("Blogs");
        builder.HasKey(x => x.Id);
        builder.Property(x => x.Url).IsRequired();
    }
}
```

2. **Подключаешь его в DbContext:**
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    // Подключаем конкретную конфигурацию
    modelBuilder.ApplyConfiguration(new BlogConfiguration());
    
    // ИЛИ подключаем вообще ВСЕ конфигурации, которые найдутся в проекте 
    modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
}
```
Это самый профессиональный способ работы с Fluent API в крупных проектах.