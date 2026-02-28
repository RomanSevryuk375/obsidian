---
aliases:
  - Выполнение сырого SQL
tags:
  - EF_Core
  - ORMs
  - SQL
  - dotnet
date: 2026-02-28 12:57
status:
---
Хотя [[LINQ]] — мощный инструмент, иногда он генерирует неоптимальный [[SQL]], или тебе нужно использовать специфические фичи [[БД]] ([[CTE]], [[оконные функции]], [[Хранимые процедуры]]), которые LINQ не поддерживает.

В таких случаях [[Entity Framework|EF Core]] позволяет тебе "спуститься на уровень ниже" и выполнить **Сырой SQL (Raw SQL)**.

Есть два основных сценария использования:
1.  **Получение данных (`FromSql...`)**: Когда ты хочешь получить объекты из базы.
2.  **Выполнение команд (`ExecuteSql...`)**: Когда нужно что-то изменить (INSERT, UPDATE, DELETE) или запустить процедуру.

---

### 1. Получение сущностей ([[FromSql]])

Используется, когда ты хочешь выполнить `SELECT` и превратить результат в свои классы (Entities).

#### Основные методы:
*   `FromSqlRaw("SQL строка", параметры)` — для сырых строк и параметров.
*   `FromSqlInterpolated($"SQL {переменная}")` — (или просто `FromSql` в EF Core 7+) безопасный вариант с интерполяцией строк.

**Пример:**
```csharp
var minAmount = 5000;

// Безопасный вызов с параметрами
var bigBills = context.Bills
    .FromSqlInterpolated($"SELECT * FROM bills WHERE Amount > {minAmount}")
    .ToList();
```

#### Крутая фишка: Компонуемость ([[Composability]])
Если твой [[SQL]]-запрос является валидным подзапросом (начинается с `SELECT`), ты можешь продолжать навешивать на него [[LINQ]]!

```csharp
// 1. Пишем основу на SQL (например, используем специфичный хинт базы данных)
var query = context.Bills
    .FromSqlRaw("SELECT * FROM bills WITH (NOLOCK)"); 

// 2. Добавляем фильтрацию уже через C# LINQ
// EF Core обернет твой SQL в подзапрос: SELECT * FROM (Твой SQL) WHERE StatusId = 1
var result = query
    .Where(b => b.StatusId == 1)
    .OrderBy(b => b.CreatedAt)
    .ToList();
```

**Ограничения `FromSql`:**
1.  SQL должен возвращать **все** колонки, которые есть в Entity. Нельзя вернуть только `Id`, если [[Entity Framework|EF]] ожидает `Id, Amount, Date...`.
2.  Имена колонок в SQL должны совпадать с именами свойств (или их маппингом).

---

### 2. Выполнение команд ([[ExecuteSql]])

Используется для `INSERT`, `UPDATE`, `DELETE`, вызова хранимых процедур или [[DDL]] команд (например, `TRUNCATE TABLE`). Эти методы не возвращают данные, они возвращают `int` (количество затронутых строк).

Вызываются не через `DbSet`, а через `context.Database`.

**Пример:**
```csharp
var statusId = 1;

// Выполняем массовое обновление (до EF Core 7 это был единственный способ сделать это быстро)
int rowsAffected = await context.Database
    .ExecuteSqlInterpolatedAsync($"UPDATE bills SET Amount = Amount + 10 WHERE StatusId = {statusId}");
```

Или вызов хранимой процедуры:
```csharp
await context.Database.ExecuteSqlRawAsync("EXEC CleanUpOldLogs @DaysOld", new SqlParameter("@DaysOld", 30));
```

---

### 3. Получение простых типов ([[SqlQuery]]) — EF Core 7/8+

Раньше `FromSql` работал только для сущностей (`DbSet`). Если ты хотел получить просто список `Id` или скалярное значение, приходилось танцевать с бубном.

Теперь (в EF Core 8) есть метод `SqlQuery`:

```csharp
// Возвращаем не сущности Bill, а просто список чисел (Id)
var ids = await context.Database
    .SqlQuery<int>($"SELECT Id FROM bills WHERE Amount > 1000")
    .ToListAsync();

// Или DTO (если это не Entity)
var stats = await context.Database
    .SqlQuery<BillStatDto>($"SELECT StatusId, COUNT(*) as Count FROM bills GROUP BY StatusId")
    .ToListAsync();
```

---

### 4. БЕЗОПАСНОСТЬ ([[SQL Injection]])

Это самый важный пункт. При использовании сырого [[SQL]] ты берешь ответственность за безопасность на себя.

**❌ СМЕРТЕЛЬНО ОПАСНО:**
Никогда не склеивай строки с данными от пользователя через `+` или `string.Format`.
```csharp
string name = userInput; // Пользователь ввел: "'; DROP TABLE bills; --"
// Взлом базы данных:
var users = context.Users.FromSqlRaw("SELECT * FROM Users WHERE Name = '" + name + "'");
```

**✅ БЕЗОПАСНО:**
1.  **Интерполяция (рекомендуется):**
    EF Core **не** подставляет значение просто как строку. Он создает `DbParameter` (@p0), что делает инъекцию невозможной.
    ```csharp
    context.Users.FromSqlInterpolated($"SELECT * FROM Users WHERE Name = {name}");
    ```
2.  **Явные параметры:**
    ```csharp
    context.Users.FromSqlRaw("SELECT * FROM Users WHERE Name = {0}", name);
    ```

---

### 5. Когда использовать?

1.  **Сложные отчеты:** Когда [[LINQ]] генерирует запрос на 5 страниц, который тормозит, а ты можешь написать оптимальный [[SQL]] на 5 строк.
2.  **[[Оконные функции]]:** `ROW_NUMBER() OVER (...)`, `RANK()` (хотя [[Entity Framework|EF Core]] 8 уже начал их поддерживать, но не все).
3.  **[[CTE]] (Common Table Expressions):** Рекурсивные запросы (иерархии).
4.  **[[Bulk Operations]]:** Массовое обновление/удаление (хотя сейчас лучше использовать [[ExecuteUpdate]] / [[ExecuteDelete]]).
5.  **[[Хранимые процедуры]]:** Использование логики, зашитой в [[БД]].

### Резюме

*   `FromSql` — когда нужен результат в виде **объектов Entity**. Можно комбинировать с LINQ (`.Where`).
*   `SqlQuery` — (EF Core 8+) когда нужны **DTO** или **простые типы** (int, string).
*   `ExecuteSql` — когда нужно просто выполнить команду (**UPDATE, DELETE, EXEC**) и забыть.
*   Всегда используй **параметры** (интерполяцию), чтобы избежать SQL-инъекций.