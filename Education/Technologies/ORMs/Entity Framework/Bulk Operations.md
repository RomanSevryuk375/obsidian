---
aliases:
tags:
  - dotnet
  - EF_Core
  - ORMs
  - SQL
date: 2026-02-28 13:18
status:
---
**Bulk Updates (Массовое обновление)** и **Bulk Deletes (Массовое удаление)** в [[Entity Framework|EF Core]] 7+ — это одна из самых долгожданных фич, которая кардинально меняет производительность при работе с большими объемами данных.

До выхода EF Core 7, чтобы удалить или обновить 1000 записей, нам приходилось использовать сторонние библиотеки (типа `EFCore.BulkExtensions`) или писать "сырой" [[SQL]]. Теперь это нативная часть фреймворка.

---

### 1. Как было РАНЬШЕ (Проблема)

До EF Core 7 процесс выглядел так:
1.  **Загрузить** все 1000 записей в память (`SELECT`).
2.  Пройти циклом и изменить/удалить объекты.
3.  Вызвать `SaveChanges()`.
4.  EF Core генерировал **1000 отдельных SQL команд** (или батчи, но все равно передавал кучу параметров).

**Минусы:**
*   Огромный расход памяти (все объекты в RAM).
*   Медленно (материализация объектов, [[Change Tracker]]).
*   Много трафика (гоняем данные туда-сюда).

---

### 2. Как стало ТЕПЕРЬ (Решение)

Методы `ExecuteUpdate` и `ExecuteDelete` транслируются **напрямую** в одну [[SQL]]-команду (`UPDATE ... WHERE` или `DELETE ... WHERE`).
Данные в память **не загружаются**.

---

### 3. ExecuteDelete (Массовое удаление)

Самый простой сценарий. Допустим, нужно удалить все неоплаченные счета, которые старше 1 года.

```csharp
// Генерирует ОДИН запрос: DELETE FROM bills WHERE CreatedAt < ... AND StatusId = 2
await context.Bills
    .Where(b => b.CreatedAt < DateTime.UtcNow.AddYears(-1) && b.StatusId == (int)BillStatusEnum.Unpaid)
    .ExecuteDeleteAsync();
```

*   **Возвращает:** `int` — количество удаленных строк.
*   **[[SaveChanges]] не нужен:** Запрос выполняется немедленно.

---

### 4. ExecuteUpdate (Массовое обновление)

Здесь синтаксис чуть сложнее, так как нужно сказать "какую колонку на что меняем". Используется метод `SetProperty`.

**Сценарий:** Увеличить сумму всех неоплаченных счетов на 10% (штраф) и обновить дату изменения.

```csharp
await context.Bills
    .Where(b => b.StatusId == (int)BillStatusEnum.Unpaid)
    .ExecuteUpdateAsync(s => s
        // 1. Устанавливаем новое значение для Amount
        .SetProperty(b => b.Amount, b => b.Amount * 1.1m) 
        // 2. Устанавливаем новое значение для LastBillDate
        .SetProperty(b => b.ActualBillDate, DateOnly.FromDateTime(DateTime.UtcNow))
    );
```

**[[SQL]], который получится:**
```sql
UPDATE bills
SET Amount = Amount * 1.1, ActualBillDate = GETUTCDATE()
WHERE StatusId = 2
```

Это **атомарная** операция на уровне базы данных.

---

### 5. Важные особенности и "Подводные камни" 

#### А. [[Change Tracker]] ничего не знает
Это самое главное отличие. Поскольку данные меняются напрямую в базе, `DbContext` (который живет в памяти) об этом **не узнает**.

```csharp
// 1. Загрузили счет в память (Amount = 100)
var bill = await context.Bills.FirstAsync(b => b.Id == 1);

// 2. Сделали массовое обновление в базе (Amount стало 200)
await context.Bills.Where(b => b.Id == 1)
      .ExecuteUpdateAsync(s => s.SetProperty(b => b.Amount, 200));

// 3. В памяти (в переменной bill) Amount всё еще 100!
Console.WriteLine(bill.Amount); // Выведет 100
```
**Вывод:** Если ты используешь эти методы, делай это либо в начале запроса, либо создавай новый контекст, либо сбрасывай ChangeTracker (`context.ChangeTracker.Clear()`), если планируешь дальше работать с этими же данными.

#### Б. [[SaveChanges]] не работает
Методы `ExecuteUpdate/Delete` выполняются **сразу же** при вызове (как `ToList`).
Если ты напишешь:
```csharp
context.Bills.ExecuteDeleteAsync(); // Удалилось сразу!
// ... ошибка в коде ...
await context.SaveChangesAsync(); // Это уже не имеет значения
```
Чтобы откатить изменения в случае ошибки, нужно явно использовать **[[Транзакции]]**:

```csharp
using var transaction = await context.Database.BeginTransactionAsync();
try 
{
    await context.Bills.Where(...).ExecuteDeleteAsync();
    // ... другая логика ...
    await transaction.CommitAsync();
}
catch 
{
    await transaction.RollbackAsync();
}
```

#### В. Ограничения наследования [[TPH (Table Per Hierarchy)]]
Если у тебя сложная иерархия наследования (Table Per Hierarchy), `ExecuteUpdate` может не сработать, если нужно обновить таблицы, разбросанные по разным [[JOIN]]'ам. Но для стандартных плоских таблиц это работает идеально.

---

### Резюме

Используй `ExecuteUpdate` и `ExecuteDelete`, когда:
1.  Нужно изменить/удалить много записей (архивация логов, начисление процентов всем пользователям).
2.  Тебе не нужно валидировать каждую запись отдельно в C# коде.
3.  Ты хочешь максимальной производительности.

Это современная замена старым подходам с перебором в цикле.