---
aliases:
  - using
tags:
  - C_sharp
  - dotnet
date: 2026-08-02 21:45
status:
---
# Оператор using (Управление ресурсами)

### Суть (The "What")
**Оператор `using`** — это синтаксический сахар для блока `try-finally`, который гарантирует детерминированное освобождение неуправляемых ресурсов (файловых дескрипторов, сетевых сокетов, подключений к БД) сразу после завершения работы с ними. 

Он решает задачу предотвращения утечек памяти и ресурсов, автоматически вызывая метод `Dispose()` (или `DisposeAsync()`) у объектов, реализующих интерфейс `[[IDisposable]]` (или `IAsyncDisposable`), даже если в процессе работы возникло исключение.

---

### Как это работает (The "How" / Nutshell style)

#### 1. Трансляция в [[IL]]
На уровне компилятора [[Roslyn]] блок `using` полностью исчезает. Компилятор разворачивает его в надежную конструкцию `try-finally`. 
```csharp
// Исходный код:
using (var stream = new MemoryStream()) { /* работа */ }

// То, во что это превращается (упрощенно):
MemoryStream stream = new MemoryStream();
try { /* работа */ }
finally { 
    if (stream != null) ((IDisposable)stream).Dispose(); 
}
```

#### 2. Интеграция со Сборщиком мусора ([[GC]])
Если не вызвать `Dispose()` у объекта, владеющего неуправляемыми ресурсами, он все равно будет очищен, но через механизм финализации (Finalizer/Destructor). 
*   Объекты с финализаторами при первой сборке мусора попадают в очередь финализации (Finalization Queue) и переживают сборку, переходя в следующее поколение (Generation 1 или 2).
*   Это задерживает освобождение памяти и создает "плавающий" момент закрытия ресурса. Правильная реализация `Dispose()` обычно вызывает `GC.SuppressFinalize(this)`, что снимает объект с очереди финализации, снижая нагрузку на [[GC]].

#### 3. Область видимости (using declarations)
Начиная с C# 8, доступно объявление `using` без фигурных скобок. Объект уничтожается немедленно в момент выхода потока выполнения из текущей области видимости (блока кода, метода или цикла).

#### 4. Асинхронное освобождение (`await using`)
При работе с I/O операциями (сеть, файловая система) синхронный вызов `Dispose()` может заблокировать вызывающий поток (Thread Blocking), пока ОС закрывает хендл. Интерфейс `IAsyncDisposable` и конструкция `await using` позволяют освобождать ресурсы асинхронно, возвращая поток в пул ([[ThreadPool]]).

---

### Пример кода ([[MOC|C#]] 12)

Пример использования современных конструкций `using` при чтении данных из базы и записи их в файл.

```csharp
using System.IO;
using System.Data.SqlClient;
using System.Threading.Tasks;

namespace Infrastructure.Data;

public class DataExportService
{
    // Использование псевдонимов любых типов (C# 12) для сокращения сигнатур
    using ExportResult = (int RowsExported, long BytesWritten);

    public async Task<ExportResult> ExportUsersAsync(string connectionString, string filePath)
    {
        // 1. Асинхронный using без скобок. 
        // Поток файла будет закрыт при выходе из метода ExportUsersAsync.
        await using var fileStream = new FileStream(filePath, FileMode.Create, FileAccess.Write, FileShare.None, 4096, useAsync: true);
        
        // 2. Синхронный using без скобок для StreamWriter.
        // Буфер запишется на диск при уничтожении объекта.
        using var writer = new StreamWriter(fileStream);

        // 3. Классический using для подключения к БД
        await using var connection = new SqlConnection(connectionString);
        await connection.OpenAsync();

        await using var command = new SqlCommand("SELECT Id, Name FROM Users", connection);
        
        // SqlDataReader реализует IAsyncDisposable
        await using var reader = await command.ExecuteReaderAsync();

        int rowCount = 0;
        while (await reader.ReadAsync())
        {
            var id = reader.GetInt32(0);
            var name = reader.GetString(1);
            
            await writer.WriteLineAsync($"{id},{name}");
            rowCount++;
        }

        // Объекты уничтожаются в обратном порядке:
        // reader -> command -> connection -> writer -> fileStream
        
        return (rowCount, fileStream.Position);
    }
}
```

---

### Ошибки и Best Practices

> [!DANGER] Anti-pattern: ObjectDisposedException при отложенном выполнении (yield return)
> Никогда не возвращайте `IEnumerable<T>` (через `yield return`) или `IQueryable<T>` из метода, если источником данных является ресурс, обернутый в `using`.
> ```csharp
> // ПЛОХО: Подключение закроется до того, как вызывающий код начнет перечислять результат.
> public IEnumerable<string> GetNames() {
>     using var db = new DbContext();
>     return db.Users.Select(u => u.Name); // Исключение при эвалюации запроса
> }
> ```
> **Решение:** Материализуйте данные в память (`.ToList()`) внутри блока `using`, либо передавайте ответственность за `IDisposable` на уровень выше.

> [!WARNING] Ошибка: using для HttpClient
> Инстанцирование и оборачивание `HttpClient` в `using` на каждый запрос — классическая ошибка. Это приводит к исчерпанию сетевых сокетов (Socket Exhaustion), так как ОС удерживает закрытые сокеты в состоянии `TIME_WAIT`. 
> **Решение:** Используйте `IHttpClientFactory` или держите `HttpClient` как синглтон.

**Best Practices:**
1.  **Сокращение вложенности:** Всегда предпочитайте `using declarations` (объявление с `var` без фигурных скобок), если ресурс нужен до конца текущего блока кода. Это предотвращает "стрелочный код" (Arrow Anti-Pattern).
2.  **Порядок имеет значение:** При каскадном создании ресурсов (например, `Stream` передается в `StreamReader`), оборачивайте в `using` все `IDisposable` объекты. Внутренний объект может перехватить исключение в конструкторе обертки, и если вы не использовали `using` для базового потока, произойдет утечка.
3.  **Nullable check:** Вручную проверять объект на `null` перед блоком `using` не нужно. Компилятор в IL-коде сам генерирует проверку `if (obj != null)`.

> [!TIP] Нюанс: Директивы C# 10/12 (Global & Alias)
> Не путайте *оператор* `using` (управление ресурсами) с *директивами* (импорт пространств имен). 
> В C# 10 появились `global using` (глобальные импорты для всего проекта, обычно хранятся в `GlobalUsings.cs`), а в C# 12 разрешили использовать `using alias = Type;` не только для классов, но и для любых типов, включая кортежи (`using UserData = (string Name, int Age);`), массивы и указатели.