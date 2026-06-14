---
aliases:
tags:
  - C_sharp
  - dotnet
  - ORMs
  - tools
date: 2026-06-12 19:04
status:
---
### Суть (The "What")
**Dapper** — это высокопроизводительный микро-[[ORM]] (Object-Relational Mapper) для .NET, разработанный командой Stack Exchange. Он представляет собой набор методов расширения для интерфейса `IDbConnection` из базового ADO.NET.

Dapper решает задачу **устранения накладных расходов тяжелых ORM** (таких как [[Entity Framework|Entity Framework Core]]), предоставляя скорость выполнения запросов, практически идентичную сырому `SqlDataReader`, но избавляя разработчика от рутинного кода (boilerplate) по ручному маппингу колонок базы данных в свойства [[MOC|C#]]-объектов.

---

### Как это работает (The "How" / Nutshell style)

#### 1. Механика материализации и Reflection.Emit
В отличие от старых ORM или наивного ручного маппинга, Dapper **не использует рефлексию при каждом запросе** для сопоставления колонок и свойств.
*   **Генерация [[IL]]-кода:** При первом выполнении запроса Dapper использует `Reflection.Emit` для динамической генерации оптимального IL-кода (в виде делегата), который читает данные из `IDataReader` и присваивает их свойствам конкретного объекта.
*   **Кэширование делегатов:** Этот сгенерированный делегат кэшируется в статической памяти (ConcurrentDictionary). Ключом кэша выступает сигнатура (текст [[SQL]]-запроса + типы параметров).
*   При повторном вызове того же запроса Dapper мгновенно берет готовый делегат из кэша, что обеспечивает скорость маппинга, равную скорости нативного скомпилированного кода.

#### 2. Отсутствие [[Change Tracker]]
Фундаментальное отличие Dapper от EF Core заключается в полном отсутствии механизмов отслеживания состояний (Change Tracker) и Identity Map.
*   **Влияние на кучу (Heap):** Dapper просто создает новые объекты ([[POCO]]) и отдает их вам. Он не сохраняет ссылки на эти объекты у себя внутри. Это кардинально снижает нагрузку на [[GC|Garbage Collector]] и потребление оперативной памяти.
*   **Следствие:** Вы не можете просто изменить свойство объекта и вызвать `SaveChanges()`. Для обновления данных вы обязаны написать явный SQL-запрос `UPDATE`.

#### 3. Управление соединениями (Connection Management)
Dapper автоматически открывает `IDbConnection`, если оно закрыто, и закрывает его по завершении выполнения метода. Однако, если вы передаете уже открытое соединение (или используете транзакцию), Dapper оставляет управление состоянием за вами.

> [!INFO] Нюанс: Буферизация (Buffering)
> По умолчанию метод `Query<T>` читает все данные из `IDataReader` и загружает их в `List<T>` (буферизация включена). Если вы читаете миллионы строк, это вызовет OutOfMemory. Передача параметра `buffered: false` заставляет Dapper использовать `yield return`, возвращая элементы потоково по мере их чтения из сокета БД.

---

### Пример кода (C# 12)

Современное использование Dapper с первичными конструкторами, неизменяемыми рекордами и сырыми строковыми литералами (Raw String Literals).

```csharp
using System;
using System.Collections.Generic;
using System.Data;
using System.Threading.Tasks;
using Dapper;

// DTO в виде неизменяемого рекорда
public record UserDto(int Id, string Email, DateTime CreatedAt);

// Primary Constructor для внедрения IDbConnection (например, NpgsqlConnection)
public class UserRepository(IDbConnection dbConnection)
{
    private readonly IDbConnection _db = dbConnection;

    public async Task<UserDto?> GetUserByIdAsync(int userId)
    {
        // Использование Raw String Literals (""") делает SQL читаемым и позволяет копипастить его в консоль БД
        const string sql = """
            SELECT Id, Email, CreatedAt 
            FROM Users 
            WHERE Id = @Id
            """;

        // Анонимный объект передается для безопасной подстановки параметров
        return await _db.QueryFirstOrDefaultAsync<UserDto>(sql, new { Id = userId });
    }

    public async Task<IEnumerable<UserDto>> GetActiveUsersAsync()
    {
        const string sql = """
            SELECT Id, Email, CreatedAt 
            FROM Users 
            WHERE IsActive = true
            """;

        // Возвращает IEnumerable<T> (по умолчанию буферизированный в список)
        return await _db.QueryAsync<UserDto>(sql);
    }

    public async Task<int> UpdateUserEmailAsync(int userId, string newEmail)
    {
        const string sql = """
            UPDATE Users 
            SET Email = @Email 
            WHERE Id = @Id
            """;

        // ExecuteAsync возвращает количество затронутых строк
        return await _db.ExecuteAsync(sql, new { Id = userId, Email = newEmail });
    }
}
```

---

### Ошибки и Best Practices

> [!DANGER] Anti-pattern: SQL-инъекции через интерполяцию
> Никогда не используйте интерполяцию C# `$"SELECT * FROM Users WHERE Name = '{name}'"` в методах Dapper. Dapper кэширует делегаты по строке запроса. Использование интерполяции не только открывает прямую уязвимость для SQL-инъекций, но и "взрывает" внутренний кэш Dapper, так как каждый запрос с новым именем будет считаться уникальным, вызывая утечку памяти. Всегда передавайте параметры через анонимные объекты: `new { Name = name }`.

> [!WARNING] Ошибка: Сложный маппинг 1-ко-многим
> Dapper поддерживает Multi-mapping (когда один SQL-запрос с `JOIN` возвращает сразу несколько объектов), но ручная склейка связи "Один ко многим" (например, Заказ и его 10 Строк) требует написания словаря-накопителя в памяти. Это громоздко и чревато ошибками. Для глубоких иерархий графов объектов Entity Framework Core подходит лучше.

**Best Practices:**
1.  **Dapper в [[CQRS]]:** Идеальный архитектурный паттерн — использовать Entity Framework Core для Команд (Commands — сложная бизнес-логика изменения состояния), а Dapper — для Запросов (Queries — сверхбыстрое чтение денормализованных плоских DTO напрямую из таблиц или View).
2.  **Raw String Literals:** В C# 11/12 всегда используйте `"""` для написания SQL. Это позволяет сохранить форматирование и избежать проблем с экранированием кавычек.
3.  **Имена параметров:** Имена свойств в анонимном объекте параметров должны в точности совпадать с именами параметров в SQL-коде (без символа `@`).
4.  **Асинхронность:** Всегда используйте методы с суффиксом `Async` (`QueryAsync`, `ExecuteAsync`). Блокирующие вызовы к базе данных — главная причина снижения пропускной способности ASP.NET приложений.

> [!TIP] Нюанс: Dapper и snake_case
> По умолчанию Dapper ожидает, что названия колонок в БД совпадают со свойствами в C# (без учета регистра). Если ваша БД использует `snake_case` (например, `first_name`), а C# использует `PascalCase` (`FirstName`), вы можете настроить глобальный маппер Dapper при старте приложения: `DefaultTypeMap.MatchNamesWithUnderscores = true;`.