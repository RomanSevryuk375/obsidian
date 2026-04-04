---
aliases:
tags:
  - C_sharp
  - dotnet
  - solid
  - review
date: 2026-03-12 16:09
status:
sr-due: 2026-04-12
sr-interval: 13
sr-ease: 270
---
### Суть (The "What")
**SOLID** — это аббревиатура пяти основных принципов [[ООП|объектно-ориентированного проектирования]], сформулированных Робертом Мартином. Они представляют собой набор правил для создания гибкой, масштабируемой и поддерживаемой архитектуры программного обеспечения.

SOLID решает проблему **"гнилого кода" (code rot)**: жесткости (сложно изменить), хрупкости (одно изменение ломает всё) и неподвижности (невозможно переиспользовать компоненты). Применение этих принципов снижает связность (coupling) и повышает зацепление (cohesion) внутри системы.

---

### Как это работает (The "How" / Nutshell style)

#### 1. S: [[Single Responsibility Principle|Single Responsibility]] (Принцип единственной ответственности)
У класса должна быть только одна причина для изменения.
*   **Механика:** Класс должен отвечать за одну конкретную бизнес-задачу или функциональную область. Если класс занимается и валидацией, и сохранением в БД, и логированием — его нужно разделить.

#### 2. O: [[Open Closed Principle|Open/Closed]] (Принцип открытости/закрытости)
Программные сущности должны быть открыты для расширения, но закрыты для модификации.
*   **Механика:** Используются абстрактные классы и интерфейсы. Новое поведение добавляется через создание новых классов-наследников, а не через изменение существующего, уже протестированного кода.

#### 3. L: [[Liskov Substitution Principle|Liskov Substitution]] (Принцип подстановки Лисков)
Объекты в программе должны быть заменяемыми на экземпляры их подтипов без изменения правильности работы программы.
*   **Механика:** Наследник не должен нарушать "контракт" базового класса (не должен выбрасывать `NotImplementedException` для методов родителя или менять логику предусловий).

#### 4. I: [[Interface Segregation Principle|Interface Segregation]] (Принцип разделения интерфейса)
Клиенты не должны зависеть от методов, которые они не используют.
*   **Механика:** Вместо одного "толстого" интерфейса создаются несколько специализированных. Это предотвращает ситуацию, когда класс вынужден реализовывать пустые методы "для галочки".

#### 5. D: [[Dependency Inversion Principle|Dependency Inversion]] (Принцип инверсии зависимостей)
Зависимости внутри системы должны строиться на основе абстракций, а не конкретных реализаций.
*   **Механика:** Высокоуровневые модули не зависят от низкоуровневых. Оба зависят от интерфейсов. В .NET это реализуется через **Dependency Injection (DI)** контейнеры.

> [!INFO] Нюанс: Полиморфизм и VMT
> Принципы O, L и D опираются на механизм динамического полиморфизма. Вызов метода через интерфейс или абстрактный класс проходит через таблицу виртуальных методов (VMT). Это добавляет микроскопическую задержку (индирекцию), которая многократно окупается гибкостью архитектуры.

---

### Пример кода ([[MOC#/MOC|C#]] 12)

Ниже пример системы уведомлений, соблюдающей SOLID.

```csharp
using System;
using System.Collections.Generic;

// 1. Interface Segregation: Маленькие интерфейсы под конкретные задачи
public interface IMessage
{
    string Content { get; }
}

public interface INotificationSender
{
    void Send(IMessage message);
}

// 2. Single Responsibility: Класс только хранит данные сообщения
public record EmailMessage(string Content, string Subject) : IMessage;

// 3. Open/Closed: Добавление нового способа отправки не меняет существующий код
public class EmailSender : INotificationSender
{
    public void Send(IMessage message) => 
        Console.WriteLine($"Email sent: {message.Content}");
}

public class SmsSender : INotificationSender
{
    public void Send(IMessage message) => 
        Console.WriteLine($"SMS sent: {message.Content}");
}

// 4. Dependency Inversion: Сервис зависит от интерфейса, а не от конкретного класса
public class NotificationService(IEnumerable<INotificationSender> senders)
{
    private readonly IEnumerable<INotificationSender> _senders = senders;

    public void NotifyAll(IMessage message)
    {
        foreach (var sender in _senders)
        {
            // 5. Liskov Substitution: Любой INotificationSender корректно отработает метод Send
            sender.Send(message);
        }
    }
}

public class Program
{
    public static void Main()
    {
        // Использование (имитация DI контейнера)
        var senders = new List<INotificationSender> { new EmailSender(), new SmsSender() };
        var service = new NotificationService(senders);
        
        service.NotifyAll(new EmailMessage("System Alert", "Warning"));
    }
}
```

---

### Ошибки и Best Practices

> [!DANGER] Anti-pattern: Over-engineering (Чрезмерное проектирование)
> Не применяйте все принципы SOLID к крошечным утилитам или коду, который никогда не будет расширяться. Создание 10 интерфейсов для одного метода `Main` — это трата ресурсов и усложнение навигации.

> [!WARNING] Ошибка: Нарушение LSP (NotImplementedException)
> Если вы наследуетесь от `List<T>`, чтобы создать `ReadOnlyList`, и выбрасываете исключение в методе `Add`, вы нарушаете принцип Лисков. Лучше использовать композицию или реализовать только `IEnumerable`.

**Best Practices:**
1.  **Проектируйте от интерфейсов:** Начинайте создание нового сервиса с определения его контракта (`interface`). Это автоматически подталкивает к соблюдению DIP.
2.  **Маленькие методы:** SRP применим не только к классам, но и к методам. Метод более 20 строк — кандидат на разделение.
3.  **Используйте встроенный DI:** В .NET 6/8+ используйте `IServiceCollection`. Это стандартный способ реализации принципа инверсии зависимостей.
4.  **Composition over Inheritance:** Вместо глубоких иерархий классов (нарушающих OCP и LSP) используйте агрегацию объектов, реализующих нужные интерфейсы.

> [!TIP] Нюанс: DRY vs SOLID
> Иногда принцип Single Responsibility может конфликтовать с принципом [[
]] (Don't Repeat Yourself). Если разделение на разные классы требует дублирования мелкой логики, иногда разумнее оставить это дублирование ради чистоты архитектуры и изоляции изменений.