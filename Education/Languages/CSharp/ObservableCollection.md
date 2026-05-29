---
aliases:
tags:
  - C_sharp
  - dotnet
date: 2026-05-26 14:42
status:
---
### Суть (The "What")
`ObservableCollection<T>` — это обобщенная коллекция из пространства имен `System.Collections.ObjectModel`, которая автоматически генерирует уведомления при добавлении, удалении или очистке элементов, а также при обновлении всего списка.

Она решает задачу **синхронизации данных между моделью и пользовательским интерфейсом (UI)** в паттерне MVVM (WPF, .NET MAUI, Avalonia, WinUI). Привязанный к такой коллекции элемент интерфейса (например, `ListView` или `DataGrid`) мгновенно реагирует на изменения данных без необходимости вручную перерисовывать UI или повторно загружать весь список.

---

### Как это работает (The "How" / Nutshell style)

#### 1. Ключевые интерфейсы и события
`ObservableCollection<T>` наследуется от `Collection<T>` и реализует два фундаментальных интерфейса:
*   **`INotifyCollectionChanged`:** Предоставляет событие `CollectionChanged`. Это событие сообщает подписчикам (обычно UI-контролам), какое именно действие произошло (Add, Remove, Replace, Reset, Move) и какие элементы были затронуты.
*   **`INotifyPropertyChanged`:** Генерирует события изменения свойств `Count` (количество элементов) и `Item[]` (индексатор) при любой модификации.

#### 2. Механика уведомлений (NotifyCollectionChangedEventArgs)
При любом изменении коллекции генерируется объект `NotifyCollectionChangedEventArgs`. Он содержит детальную информацию:
*   `Action`: тип действия (например, `NotifyCollectionChangedAction.Add`).
*   `NewItems` / `OldItems`: списки добавленных или удаленных объектов.
*   `NewStartingIndex` / `OldStartingIndex`: индексы, по которым произошли изменения.

#### 3. Потоковые ограничения (Thread Affinity)
Поскольку `ObservableCollection<T>` тесно связана с UI-компонентами, она имеет критическое ограничение:
*   **Кросс-потоковые исключения:** Если вы попытаетесь добавить или удалить элемент в `ObservableCollection` из фонового потока (например, внутри асинхронного `Task.Run()`), в то время как коллекция привязана к UI, приложение выбросит исключение, сообщающее о невозможности изменения коллекции из другого потока.

---

### Пример кода ([[C#]] 12)

```csharp
using System;
using System.Collections.ObjectModel;
using System.Collections.Specialized;

public record TaskItem(string Title, bool IsCompleted);

public class TodoListViewModel
{
    // Публичное свойство для привязки (UI)
    public ObservableCollection<TaskItem> Tasks { get; } = [];

    public TodoListViewModel()
    {
        // Подписываемся на изменения (имитация того, что делает UI-платформа под капотом)
        Tasks.CollectionChanged += OnTasksCollectionChanged;
    }

    public void AddTask(string title)
    {
        Tasks.Add(new TaskItem(title, false));
    }

    public void RemoveTaskAt(int index)
    {
        if (index >= 0 && index < Tasks.Count)
        {
            Tasks.RemoveAt(index);
        }
    }

    private void OnTasksCollectionChanged(object? sender, NotifyCollectionChangedEventArgs e)
    {
        // Анализ произошедшего изменения
        switch (e.Action)
        {
            case NotifyCollectionChangedAction.Add:
                if (e.NewItems != null)
                {
                    foreach (TaskItem newItem in e.NewItems)
                    {
                        Console.WriteLine($"[Добавлено]: {newItem.Title} на позицию {e.NewStartingIndex}");
                    }
                }
                break;

            case NotifyCollectionChangedAction.Remove:
                if (e.OldItems != null)
                {
                    foreach (TaskItem oldItem in e.OldItems)
                    {
                        Console.WriteLine($"[Удалено]: {oldItem.Title} с позиции {e.OldStartingIndex}");
                    }
                }
                break;
        }
    }
}

public class Program
{
    public static void Main()
    {
        var viewModel = new TodoListViewModel();
        
        viewModel.AddTask("Buy groceries");
        viewModel.AddTask("Clean the house");
        viewModel.RemoveTaskAt(0);
    }
}
```

---

### Ошибки и Best Practices

> [!DANGER] Ошибка: Изменение из фонового потока
> Модификация коллекции из фонового потока разрушит привязку UI. 
> **Решение:** Маршалируйте вызовы изменений в главный/UI поток с помощью диспетчера конкретной платформы (например, `App.Current.Dispatcher.Invoke(...)` в WPF или `MainThread.BeginInvokeOnMainThread(...)` в MAUI).

> [!WARNING] Ошибка: Производительность при массовом добавлении (AddRange)
> Стандартная `ObservableCollection<T>` не имеет метода `AddRange()`. Если вы добавите 1000 элементов в цикле через `.Add()`, событие `CollectionChanged` сработает 1000 раз, заставляя UI перерисовываться 1000 раз. Это приведет к сильным визуальным лагам.
> **Решение:** Создайте наследника от `ObservableCollection<T>` и реализуйте в нем метод `AddRange`, который временно отключает уведомления или вызывает одно финальное уведомление с типом `Reset`.

**Best Practices:**
1.  **Только для UI:** Используйте `ObservableCollection<T>` исключительно в слое представления (ViewModel/UI). Не используйте её в глубокой бизнес-логике, сервисах или репозиториях, так как накладные расходы на постоянную генерацию событий уведомлений очень высоки по сравнению с [[List]]\<T>.
2.  **Обновление свойств элементов:** Помните, что `ObservableCollection` реагирует только на добавление, удаление или замену *самих элементов*. Если вы измените свойство внутри элемента (например, `task.IsCompleted = true`), событие `CollectionChanged` **не сработает**. Для этого сам класс элемента должен реализовывать `INotifyPropertyChanged`.
3.  **Не пересоздавайте коллекцию:** Не пишите код вида `MyCollection = new ObservableCollection<T>(newList)`. Это ломает привязку (binding) в UI. Вместо этого очищайте коллекцию через `.Clear()` и наполняйте заново, либо используйте `with`-выражения для точечного обновления.

> [!TIP] Оптимизация Reset
> Вызов метода `.Clear()` генерирует одно событие `CollectionChanged` с действием `NotifyCollectionChangedAction.Reset`. Это гораздо эффективнее для производительности UI, чем поочередное удаление каждого элемента в цикле.