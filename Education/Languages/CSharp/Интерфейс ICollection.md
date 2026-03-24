---
aliases:
tags:
  - C_sharp
  - dotnet
  - review
date: 2026-03-15 20:59
status:
---
### Суть (The "What")
`ICollection<T>` — это базовый [[Интерфейсы|интерфейс]] для всех изменяемых (mutable) [[Коллекции|коллекций]] в .NET. Он расширяет `IEnumerable<T>`, добавляя методы для манипуляции данными (добавление, удаление, очистка) и свойства для получения метаданных (размер коллекции).

Этот интерфейс решает задачу **унификации управления данными**. Если `IEnumerable` отвечает только за перебор, то `ICollection` определяет контракт для контейнера, размер которого можно узнать и содержимое которого можно модифицировать.

---

### Как это работает (The "How" / Nutshell style)

#### 1. Иерархия и контракты
*   `ICollection<T>` наследуется от `IEnumerable<T>`.
*   Все стандартные коллекции (`List<T>`, `Dictionary<K,V>`, `HashSet<T>`) реализуют этот интерфейс.
*   Существует также нетипизированный `ICollection` (legacy), который содержит свойства для синхронизации потоков (`SyncRoot`, `IsSynchronized`), отсутствующие в generic-версии.

#### 2. Свойство Count vs [[LINQ]] .Count()
В `ICollection<T>` свойство `Count` всегда возвращает текущее количество элементов.
*   **Производительность:** Обращение к свойству `Count` обычно имеет сложность **O(1)**, так как значение хранится в поле класса.
*   **Нюанс:** Метод расширения LINQ `.Count()` сначала пытается привести объект к `ICollection<T>`. Если это удается, он вызывает свойство. Если нет — он перебирает всю последовательность за **O(n)**.

#### 3. Модификация и IsReadOnly
Интерфейс содержит свойство `IsReadOnly`. 
*   Если оно возвращает `true`, методы `Add`, `Remove` и `Clear` должны выбрасывать `NotSupportedException`.
*   Это позволяет использовать один и тот же интерфейс как для обычных списков, так и для неизменяемых оберток (например, `ReadOnlyCollection<T>`).

#### 4. Метод CopyTo
`CopyTo(T[] array, int arrayIndex)` — низкоуровневый метод для быстрого копирования содержимого коллекции в массив. Часто используется внутри [[CLR]] для высокопроизводительных операций и при конвертации коллекций.

> [!INFO] Нюанс: Explicit Implementation
> Многие коллекции реализуют методы `ICollection` (нетипизированного) явно (explicitly), чтобы не засорять публичный API устаревшими свойствами вроде `SyncRoot`.

---

### Пример кода ([[MOC|C#]] 12)

```csharp
using System;
using System.Collections.Generic;

public class MyDataBuffer<T> : ICollection<T>
{
    private readonly List<T> _internalStorage = [];

    // Реализация свойств
    public int Count => _internalStorage.Count;
    public bool IsReadOnly => false;

    // Реализация методов манипуляции
    public void Add(T item) => _internalStorage.Add(item);
    public void Clear() => _internalStorage.Clear();
    public bool Contains(T item) => _internalStorage.Contains(item);
    public bool Remove(T item) => _internalStorage.Remove(item);

    public void CopyTo(T[] array, int arrayIndex) => 
        _internalStorage.CopyTo(array, arrayIndex);

    // Итератор для поддержки IEnumerable
    public IEnumerator<T> GetEnumerator() => _internalStorage.GetEnumerator();
    System.Collections.IEnumerator System.Collections.IEnumerable.GetEnumerator() => GetEnumerator();
}

public class Program
{
    public static void Main()
    {
        // Использование коллекции (C# 12 collection expression)
        ICollection<string> collection = new MyDataBuffer<string> { "A", "B", "C" };
        
        if (!collection.IsReadOnly)
        {
            collection.Add("D");
        }

        Console.WriteLine($"Количество элементов: {collection.Count}");
    }
}
```

---

### Ошибки и Best Practices

> [!DANGER] Anti-pattern: Использование ICollection для передачи данных "только для чтения"
> Если метод должен только принимать данные для чтения, используйте `IEnumerable<T>` или `IReadOnlyCollection<T>`. Использование `ICollection<T>` вводит вызывающий код в заблуждение, предполагая, что метод может изменить коллекцию.

> [!WARNING] Ошибка: Потокобезопасность
> Свойство `Count` в стандартных generic-коллекциях не является потокобезопасным. Если один поток добавляет элементы, а другой читает `Count`, может возникнуть состояние гонки (Race Condition). Для многопоточности используйте `System.Collections.Concurrent`.

**Best Practices:**
1.  **Проверка IsReadOnly:** Если вы пишете универсальный метод, принимающий `ICollection<T>` и модифицирующий его, всегда проверяйте свойство `IsReadOnly` перед вызовом `Add` или `Remove`.
2.  **Initial Capacity:** При реализации своих коллекций помните, что `ICollection` не обязывает иметь начальную емкость, но для производительности её стоит добавить в конструктор (как в `List<T>`).
3.  **Использование CopyTo:** Если вам нужно эффективно передать данные в нативный код или массив, используйте `CopyTo`. Это быстрее, чем ручной перебор через `foreach`.

> [!TIP] Нюанс: ICollection и Массивы
> Массивы в C# (`T[]`) реализуют `ICollection<T>`, но их свойство `IsReadOnly` возвращает `true` для методов изменения размера (Add/Remove), так как размер массива фиксирован в куче в момент аллокации.