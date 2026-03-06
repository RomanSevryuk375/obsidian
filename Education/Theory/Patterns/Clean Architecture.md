---
aliases:
tags:
  - architecture
  - DesignPatterns
  - dotnet
date: 2026-03-02 16:07
status:
---
## Концепция

**Clean Architecture** — это подход к проектированию, который ставит **Бизнес-правила (Domain)** в центр системы, изолируя их от деталей реализации (UI, Базы данных, Фреймворков).

### Основные принципы (The Dependency Rule)
1.  **Центрирование:** Ядро системы — это [[Domain Model]] (Сущности и Правила). Оно не зависит ни от чего.
2.  **Направление зависимостей:** Все зависимости указывают **внутрь** круга. Внешние круги (Инфраструктура) знают о внутренних (Домен), но внутренние ничего не знают о внешних.
3.  **Абстракции через границы:** Взаимодействие между слоями происходит через интерфейсы ([[Ports]]), определенные во внутреннем слое.

**Какую проблему решает?**
*   **Тестируемость:** Можно протестировать бизнес-логику на 100% без базы данных, Web API или интернета.
*   **Устойчивость к изменениям:** Замена UI (React на Angular) или БД (SQL Server на PostgreSQL) не требует переписывания бизнес-правил.

---

## 🔑 Ключевое отличие: N-Layer vs Clean

Самый частый вопрос на собеседованиях.

| Характеристика | **N-Layer (Слоистая)** | **Clean Architecture** |
| :--- | :--- | :--- |
| **Центр системы** | База данных (Database Driven). | Доменная модель (Domain Driven). |
| **Зависимости** | `UI -> BLL -> DAL -> DB` | `UI -> Core` и `Infra -> Core` |
| **Интерфейс репозитория** | Лежит в DAL (или BLL). | Лежит в **Application/Domain** (Core). |
| **Реализация репозитория** | Лежит в DAL. | Лежит в **Infrastructure**. |

> [!NOTE] Суть инверсии
> В N-Layer слой логики *зависит* от слоя данных. В Clean Architecture слой данных (Infrastructure) *зависит* от слоя логики (Core), реализуя его интерфейсы. Это и есть [[Dependency Inversion Principle]].

---

## 🏗️ Структура решения (.NET)

Стандартная структура папок для Modern .NET Solutions:

```text
MySolution.sln
├── 1. Core (Ядро - никаких ссылок на EntityFramework или ASP.NET!)
│   ├── MyProject.Domain        (Enterprise Business Rules)
│   │   ├── Entities            (Order, Product)
│   │   ├── ValueObjects        (Address, Money)
│   │   └── Exceptions          (DomainException)
│   │
│   └── MyProject.Application   (Application Business Rules)
│       ├── Interfaces          (IOrderRepository, IEmailService)
│       ├── UseCases/Services   (CreateOrderHandler)
│       └── DTOs                (OrderDto)
│
├── 2. Infrastructure (Детали реализации)
│   ├── MyProject.Infrastructure (Внешний мир)
│   │   ├── Persistence         (AppDbContext : DbContext)
│   │   ├── Repositories        (OrderRepository : IOrderRepository)
│   │   └── Services            (SmtpEmailService : IEmailService)
│   │
│   └── MyProject.API           (Presentation / Entry Point)
│       ├── Controllers
│       └── Program.cs          (DI Composition Root)
```

**Пример кода (C#):**

```csharp
// --- Core/Application/Interfaces/IOrderRepository.cs ---
// Интерфейс принадлежит слою Application!
public interface IOrderRepository {
    Task SaveAsync(Order order);
}

// --- Infrastructure/Repositories/OrderRepository.cs ---
// Реализация зависит от EF Core и Application
public class OrderRepository : IOrderRepository {
    private readonly AppDbContext _context;
    public Task SaveAsync(Order order) {
        _context.Orders.Add(order); // Маппинг Domain Entity -> DB Row
        return _context.SaveChangesAsync();
    }
}
```

---

## 📊 Диаграмма зависимостей

Диаграмма показывает "поток управления" (Flow of Control) против "потока зависимостей" (Source Code Dependencies).

![[YsN6twE3-4Q4OYpgxoModmx29I8zthQ3f0OR.jfif]]

---

##  Плюсы и Минусы (Trade-offs)

### ✅ Плюсы 
1.  **Независимость от фреймворков:** Бизнес-логика — это "чистый" C# код. Фреймворки (ASP.NET Core, EF Core) вытеснены на периферию.
2.  **Тестируемость:** `UseCases` тестируются юнит-тестами молниеносно, так как вместо реальной базы подсовывается Mock.
3.  **Гибкость командной разработки:** Разные команды могут параллельно писать `Core` (бизнес) и `Infrastructure` (интеграции).

### ❌ Минусы 
1.  **Boilerplate (Многословность):** Вам придется писать много "перекладывания данных". DTO -> Domain Entity -> DB Entity -> DTO.
2.  **Сложность навигации:** Чтобы понять, что делает метод `Save`, нужно нажать "Go to Implementation", так как вы видите только интерфейс.
3.  **Иллюзия заменяемости БД:** Аргумент "мы сможем легко сменить SQL Server на Mongo" на практике срабатывает редко. Обычно это огромная боль, несмотря на архитектуру.
4.  **Performance:** Абстракции имеют цену. Использование ORM через репозитории часто мешает использовать специфичные фичи БД (например, Bulk Insert или сложные CTE запросы).

> [!WARNING] Золотое правило
> Если ваш проект — это простой CRUD (создать, прочитать, обновить, удалить) без сложной бизнес-логики, **Clean Architecture — это over-engineering**. Используйте N-Layer.

---

## Связь с другими паттернами

*   **[[CQRS]]:** Идеально встраивается в слой **Application**. Команды (`Commands`) и Запросы (`Queries`) становятся Use Cases.
*   **[[Microservices Architecture]]:** Каждый микросервис внутри себя обычно построен по Clean Architecture.
*   **[[Mediator Pattern]] (MediatR):** Часто используется в слое **Application** для связывания Controller'ов и Use Cases без прямой зависимости.

---

## 🛒 Практический кейс: Эволюция E-commerce

Представим проект **ShopNet** (интернет-магазин), который вырос из N-Layer.

### 1. Старт (Monolith N-Layer)
У нас были "толстые" контроллеры и сервисы, смешанные с EF Core.
*Боль:* Тестировать расчет скидок невозможно без поднятия базы данных. Логика размазана.

### 2. Рефакторинг (Внедрение Clean Architecture)
**Задача:** Изолировать логику расчета корзины.
**Действия:**
1.  Создаем проект `ShopNet.Domain`. Переносим туда сущности `Order`, `OrderItem`. Добавляем методы `CalculateTotal()` прямо в сущность (Rich Domain Model).
2.  Создаем проект `ShopNet.Application`. Определяем интерфейс `ICartRepository`.
3.  Создаем UseCase `CheckoutService`, который берет данные из репозитория, вызывает метод домена и сохраняет результат.
4.  Старый `ShopNet.Data` (EF Core) переименовываем в `Infrastructure` и реализуем там `ICartRepository`.

**Результат:** Логика скидок покрыта Unit-тестами на 100%.

### 3. Оптимизация (Clean + [[CQRS]])
**Проблема:** Экран "История заказов" грузится медленно из-за сложного маппинга Entities в DTO.
**Решение:** Для *чтения* мы обходим Domain слой.
Контроллер -> Application (Query) -> Dapper (Infrastructure) -> DTO.
Мы нарушаем правила Clean Architecture *осознанно* для ветки чтения (Fast Track), но сохраняем чистоту для ветки записи (Commands).

### 4. Масштабирование
Выделяем модуль `Delivery` в отдельный микросервис. Благодаря Clean Architecture, мы просто берем папку `Delivery` из домена и приложения и переносим в новый Solution. Код почти не меняется, меняется только слой Infrastructure (вместо вызова метода — отправка сообщения в RabbitMQ).

**Связи:** [[Dependency Inversion Principle]], [[Domain-Driven Design (DDD)]], [[Repository Pattern]], [[Unit Testing]]
