---
aliases:
  - Data Annotations
tags:
  - ORMs
  - dotnet
date: 2026-02-28 10:18
status:
---
**Data Annotations (Атрибуты данных)** — это способ настройки базы данных и валидации путем добавления специальных тегов (атрибутов) прямо над свойствами твоих классов.

Это альтернатива [[Fluent API]]. Если Fluent API пишется в [[DbContext]] (отдельно от класса), то аннотации живут прямо внутри класса модели.

### Подключение пространств имен
Чтобы использовать их, нужно подключить:
1.  `System.ComponentModel.DataAnnotations` (для валидации: `Required`, `MaxLength`)
2.  `System.ComponentModel.DataAnnotations.Schema` (для структуры [[БД]]: `Table`, `Column`)

---

### Основные атрибуты

#### 1. Управление ключами и таблицами

*   **`[Key]`**
    Указывает, что это свойство — **Первичный ключ ([[Primary Key]])**.
    ```csharp
    [Key]
    public string LicenseNumber { get; set; }
    ```

*   **`[Table("Name")]`**
    Меняет имя таблицы в базе данных.
    ```csharp
    [Table("sys_customers", Schema = "admin")]
    public class Customer { ... }
    ```

*   **`[Column("Name", TypeName = "...")]`**
    ```csharp
    [Column("created_at", TypeName = "date")] // Только дата, без времени
    public DateTime CreationDate { get; set; }
    ```

#### 2. Ограничения данных (Валидация + Схема)

Эти атрибуты влияют и на структуру [[БД]] (создают [[ограничения]]), и на валидацию в [[ASP.NET MOC]][[Web API]] (возвращают 400 Bad Request, если данные неверны).

*   **`[Required]`**
    Делает поле **обязательным** (`NOT NULL` в базе).
    *Примечание:* Типы вроде `int`, `bool`, `DateTime` и так не могут быть null, поэтому для них атрибут не обязателен, но полезен для семантики.
    ```csharp
    [Required] 
    public string Name { get; set; }
    ```

*   **`[MaxLength(100)]` / `[StringLength(100)]`**
    *В базе:* `nvarchar(100)` вместо `nvarchar(max)`.
    *В [[API]]:* Ошибка валидации, если строка длиннее.
    ```csharp
    [MaxLength(200)]
    public string Description { get; set; }
    ```

#### 3. Связи ([[Relationships]])

*   **`[ForeignKey("Name")]`**
    Указывает, какое свойство является внешним ключом для навигационного свойства. Это нужно, если вы отошли от стандартных имен.
    ```csharp
    public class Order
    {
        public int Id { get; set; }
        
        public int UserKey { get; set; } // Мы назвали ключ нестандартно "UserKey"

        [ForeignKey("UserKey")] // Говорим EF: свяжи это свойство с UserKey
        public User User { get; set; }
    }
    ```

#### 4. Исключение из базы

*   **`[NotMapped]`**
    Говорит [[Entity Framework|EF Core]] **игнорировать** это свойство. Оно не создаст колонку в [[базе данных]].
    *Зачем нужно:* Для вычисляемых полей или данных, которые нужны только в коде.
    ```csharp
    public string FirstName { get; set; }
    public string LastName { get; set; }

    [NotMapped]
    public string FullName => $"{FirstName} {LastName}";
    ```

#### 5. Генерация значений

*   **`[DatabaseGenerated(...)]`**
    Управляет тем, как создаются значения ([[автоинкремент]] и т.д.).
    *   `DatabaseGeneratedOption.Identity` — база сама генерирует (обычно ID).
    *   `DatabaseGeneratedOption.None` — ты сам должен задать ID (полезно, если ID приходят из внешней системы).
    *   `DatabaseGeneratedOption.Computed` — значение вычисляется базой (например, дата создания по умолчанию).

---
### Data Annotations vs [[Fluent API]]

| Data Annotations                                                                                                                      | [[Fluent API]] (`OnModelCreating`)                                                                                    |
| :------------------------------------------------------------------------------------------------------------------------------------ | :-------------------------------------------------------------------------------------------------------------------- |
| **Простота:** Видно сразу в классе модели.                                                                                            | **Мощь:** Можно настроить всё, что угодно.                                                                            |
| **Валидация:** Работает и для [[БД]], и для UI ([[API]]).                                                                             | **Чистота:** Модели остаются чистыми ([[POCO (Plain Old CLR Object)]]), без зависимостей от [[Entity Framework\|EF]]. |
| **Ограничения:** Нельзя настроить составные ключи (до [[Entity Framework\|EF Core]] 5+) или сложные связи Many-to-Many с доп. полями. | **Гибкость:** Позволяет переопределять настройки для разных баз данных.                                               |

#### Рекомендация:
Используй **Data Annotations** для простых вещей (`[Required]`, `[MaxLength]`, `[Table]`), так как это сокращает код и сразу добавляет валидацию.
Используй [[Fluent API]] для сложных связей, настройки ключей, индексов и специфичных [[SQL]]-типов, чтобы не захламлять класс модели.