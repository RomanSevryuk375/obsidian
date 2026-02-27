Переход на профессиональный уровень владения [[Entity Framework]] Core требует понимания того, как именно ORM преодолевает **Object-Relational Impedance Mismatch** (несоответствие между объектной и реляционной моделями).

Глубокое понимание **Type Mapping (Сопоставления типов)** и **Value Conversions (Преобразования значений)** необходимо для проектирования устойчивых систем, особенно при реализации паттернов [[DDD (Domain-Driven Design)]] или работе с легаси-базами.

---

### 1. Архитектура сопоставления типов (Type Mapping)

По умолчанию EF Core использует провайдеры баз данных ([[SQL Server]], Npgsql и т.д.), которые содержат словарь сопоставления [[CLR (Common Language Runtime)]] типов с типами [[SQL]].

*   `int` $\rightarrow$ `int` / `integer`
*   `string` $\rightarrow$ `nvarchar(max)` / `text`
*   `DateTime` $\rightarrow$ `datetime2` / `timestamp`

Однако в реальных энтерпрайз-системах часто возникают сценарии, когда модель предметной области ([[Domain Model]]) использует типы, не имеющие прямого эквивалента в БД, или же требования к хранению отличаются от стандартных.

Здесь вступает в игру механизм **Value Conversions**.

---

### 2. Механизм Value Converters

`ValueConverter` — это слой абстракции, который перехватывает данные при чтении (Materialization) и записи (Persisting). Он работает на основе двух выражений (Expression Trees):

1.  **`ConvertToProviderExpression`**: Логика трансформации из CLR-типа в тип базы данных (при `SaveChanges`).
2.  **`ConvertFromProviderExpression`**: Логика восстановления из типа базы данных в CLR-тип (при выполнении запросов).

#### Пример: Value Object и Strong Typed IDs
В DDD часто избегают использования примитивов ([[Primitive Obsession]]). Вместо `int Id` используется структура `CustomerId`. База данных о такой структуре не знает, она ожидает `int`.

```csharp
// Конфигурация через Fluent API
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Customer>()
        .Property(c => c.Id)
        .HasConversion(
            id => id.Value,              // Write: CustomerId -> int
            value => new CustomerId(value) // Read: int -> CustomerId
        );
}
```

#### Пример: Легаси схемы (Bool as Char)
Старые базы данных часто хранят булевы значения как `Y`/`N` (char), а не `0`/`1` (bit).

```csharp
var boolConverter = new ValueConverter<bool, string>(
    v => v ? "Y" : "N",    // В базу
    v => v == "Y"          // Из базы
);

modelBuilder.Entity<User>()
    .Property(u => u.IsActive)
    .HasConversion(boolConverter);
```

---

### 3. Работа с Enums (Перечислениями)

В .NET `Enum` — это именованный набор констант (обычно `int`).
В [[SQL Server ]]нет нативного типа `Enum` (в отличие от [[PostgreSQL]]), поэтому есть две стратегии хранения:

#### Стратегия А: Хранение числового значения (По умолчанию)
EF Core кастит `Enum` к `int`.
*   *Плюсы:* Производительность, экономия места.
*   *Минусы:* Если изменить порядок в `Enum` или добавить значение в середину, данные в БД станут некорректными. "Магические числа" в SELECT-запросах.

#### Стратегия Б: Хранение строкового значения
Это Best Practice для систем, где важна читаемость данных и устойчивость к рефакторингу кода.

```csharp
public enum UserRole { Admin, Moderator, User }

modelBuilder.Entity<User>()
    .Property(u => u.Role)
    .HasConversion<string>(); // EF сам сгенерирует конвертер .ToString() / Enum.Parse()
```
**Результат в БД:** Колонка типа `nvarchar` со значением `'Admin'`.

---

### 4. Сериализация сложных объектов (JSON)

До появления нативной поддержки JSON в EF Core 7/8, стандартным паттерном для хранения списков или сложных объектов внутри одной ячейки была конвертация в JSON-строку.

```csharp
// Свойство в модели: public List<string> Tags { get; set; }

modelBuilder.Entity<Post>()
    .Property(p => p.Tags)
    .HasConversion(
        v => JsonSerializer.Serialize(v, (JsonSerializerOptions)null), // List<string> -> JSON string
        v => JsonSerializer.Deserialize<List<string>>(v, (JsonSerializerOptions)null) // JSON string -> List<string>
    );
```

---

### 5. Критический аспект: Value Comparers (Сравнение значений)

Это "подводный камень", на котором спотыкаются многие разработчики уровня Middle+.

**Проблема:**
EF Core использует *[[Change Tracker]]* (отслеживание изменений), чтобы понять, нужно ли делать SQL UPDATE. Для примитивов это просто (5 != 4).
Но для ссылочных типов, проходящих через конвертер (как `List<string>` в примере выше), EF Core по умолчанию сравнивает **ссылки**.

Если вы загрузили список, добавили в него элемент, но ссылка на сам объект списка не изменилась — **EF Core не увидит изменений и не сохранит их в базу.**

**Решение:**
Необходимо явно задать `ValueComparer`.

```csharp
var listComparer = new ValueComparer<List<string>>(
    // 1. Выражение для сравнения (Equals)
    (c1, c2) => c1.SequenceEqual(c2),
    
    // 2. Выражение для получения хэш-кода
    c => c.Aggregate(0, (a, v) => HashCode.Combine(a, v.GetHashCode())),
    
    // 3. Выражение для создания глубокой копии (Snapshotting)
    c => c.ToList()
);

modelBuilder.Entity<Post>()
    .Property(p => p.Tags)
    .HasConversion(...) // Конвертер из пункта 4
    .Metadata.SetValueComparer(listComparer); // Подключаем компаратор
```

Теперь EF Core будет сравнивать содержимое списков, а не их ссылки в памяти.

---

### 6. Ограничения и влияние на производительность

Использование `HasConversion` имеет фундаментальное ограничение: **SQL-трансляция**.

1.  **Client-Side Evaluation:** Конвертация происходит на стороне клиента (в памяти приложения). База данных не знает вашей логики C#.
2.  **Проблема с [[LINQ]]:** Если вы напишете `.Where(u => u.CustomerId == new CustomerId(5))`, EF Core сможет это транслировать (преобразует 5 в int и отправит запрос).
    Но если конвертер сложный (например, шифрование данных), то поиск (`WHERE`), сортировка (`ORDER BY`) и группировка по этому полю на стороне БД станут невозможными или приведут к загрузке всей таблицы в память, что недопустимо.

### Резюме для архитектора:
Используйте `ValueConversions` для:
*   Strongly Typed IDs.
*   Enums (String mapping).
*   Простых трансформаций форматов.
*   Шифрования отдельных колонок (с пониманием потери возможности поиска).

Для хранения сложных вложенных структур лучше рассмотреть **[[Owned Types]]** или нативную поддержку **[[JSON Columns]]** (доступно в EF Core 7+), так как они обеспечивают лучшую интеграцию с возможностями движка БД.