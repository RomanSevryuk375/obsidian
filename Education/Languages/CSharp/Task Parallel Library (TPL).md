---
aliases:
tags:
  - C_sharp
  - dotnet
date: 2026-08-05 18:40
status:
---
## Суть (The What)
**Task Parallel Library (TPL)** — это высокоуровневый набор API (в пространстве имен `System.Threading.Tasks`), который абстрагирует разработчика от ручного управления потоками, вводя концепцию Задачи (`Task` и `Task<TResult>`). 

Архитектурно TPL решает проблему сложной оркестрации многопоточного кода: библиотека берет на себя распределение задач по потокам, обработку очередей, балансировку нагрузки, отмену операций и агрегацию исключений, позволяя разработчику сосредоточиться на бизнес-логике, а не на синхронизации.

## Как это работает (Under the Hood)

> [!info] Task — это не Thread
> `Task` (Задача) — это просто объект-обещание (Future/Promise) в [[Managed Heap|управляемой куче]]. Он представляет собой абстрактную операцию, которая может выполниться в будущем. Сама по себе задача **не потребляет** ресурсы процессора, пока не будет передана на выполнение.

- **TaskScheduler и Work-Stealing Queues**: За распределение задач отвечает `TaskScheduler`. По умолчанию он отправляет задачи в [[ThreadPool]]. В пуле реализован алгоритм "кражи работы" (Work-Stealing): каждый рабочий поток имеет свою локальную очередь задач. Если поток освобождается, а его очередь пуста, он "крадет" задачи из хвоста очередей других потоков. Это минимизирует блокировки (lock contention) при доступе к глобальной очереди пула.
- **Типы параллелизма**:
  1. *Task Parallelism* (Параллелизм задач): Независимый запуск асинхронных операций (через `Task.Run` или композицию `Task.WhenAll`).
  2. *Data Parallelism* (Параллелизм данных): Одновременная обработка элементов коллекции (через классы `Parallel` или [[PLINQ]]).
- **Агрегация исключений**: Если при параллельном выполнении нескольких задач возникают ошибки, TPL оборачивает их в единый тип `AggregateException`. Распаковать его можно через метод `Flatten()`.
- **ValueTask (Оптимизация)**: Для микрооптимизаций памяти существует `ValueTask<T>`, представляющий собой [[Value Types|структуру]]. Используется в сценариях, где результат операции часто возвращается синхронно (например, из кэша), чтобы избежать лишних аллокаций объектов `Task` в куче и снизить нагрузку на [[Garbage Collector]].

## Практический пример

Реализация параллельной загрузки данных из нескольких источников с ограничением степени параллелизма (Throttling) — классическая задача из реального продакшена.

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;

namespace Infrastructure.Concurrency;

// Используем Primary Constructor (C# 12)
public class ExternalDataFetcher(IHttpClientFactory httpClientFactory)
{
    public async Task<List<string>> FetchDataInParallelAsync(
        IEnumerable<string> urls, 
        CancellationToken cancellationToken = default)
    {
        // Ограничиваем одновременное выполнение до 4 потоков (Throttling)
        // SemaphoreSlim реализует легковесный механизм синхронизации
        using var semaphore = new SemaphoreSlim(4);
        
        var client = httpClientFactory.CreateClient();

        // 1. Формируем коллекцию ненаблюдаемых (холодных) задач
        var tasks = urls.Select(async url =>
        {
            // Ждем, пока семафор пустит нас дальше
            await semaphore.WaitAsync(cancellationToken);
            try
            {
                // Эмуляция сетевого вызова
                var response = await client.GetAsync(url, cancellationToken);
                response.EnsureSuccessStatusCode();
                return await response.Content.ReadAsStringAsync(cancellationToken);
            }
            finally
            {
                // Гарантированно освобождаем место для следующей задачи
                semaphore.Release();
            }
        });

        // 2. Task Parallelism: запускаем все задачи конкурентно и ждем их завершения.
        // Task.WhenAll не блокирует текущий поток, а возвращает новую Task.
        string[] results = await Task.WhenAll(tasks);
        
        // Collection expression (C# 12) для возврата результата
        return [.. results];
    }
}
```

## Ошибки и Best Practices

> [!danger] Anti-pattern: Sync-over-Async (Deadlock)
> Вызов `.Result` или `.Wait()` у задачи в синхронном коде (особенно в старом ASP.NET или WPF) приводит к взаимной блокировке потоков (Deadlock). Основной поток ждет завершения задачи, а задача не может завершиться, так как контекст синхронизации (SynchronizationContext) занят ожидающим основным потоком.
> **Решение:** Если начали использовать `async/await` (надстройку над TPL), делайте это "до самого верха" (Async all the way).

> [!warning] Ошибка: Task.Run для I/O операций
> Вызов `Task.Run(async () => await DownloadFileAsync())` — это бессмысленная трата потока из пула. `Task.Run` предназначен **исключительно** для выгрузки тяжелых CPU-bound вычислений. Для I/O-операций (сеть, диск, БД) просто вызывайте асинхронный метод напрямую, используя механизм прерываний ОС ([[async-await]]).

> [!tip] Best Practices
> - **Отмена операций:** Всегда прокидывайте `[[CancellationToken]]` по всей цепочке вызовов TPL. Задачи в `ThreadPool` не могут быть убиты принудительно, они должны завершаться кооперативно.
> - **Parallel vs Task.WhenAll:** Используйте `Parallel.ForEach` (или `Parallel.ForEachAsync` из .NET 6+) для применения одной CPU-bound операции к набору данных (Data Parallelism). Используйте `Task.WhenAll` для ожидания множества разнородных I/O-операций (Task Parallelism).
> - **Забытые таски (Fire-and-Forget):** Не вызывайте метод, возвращающий `Task`, без `await` или сохранения ссылки. Если такая задача упадет с исключением, оно будет "проглочено" или (в старых версиях .NET) обрушит весь процесс в методе финализатора `TaskScheduler.UnobservedTaskException`.

## Связанные темы
- [[Многопоточность]] (Базовые концепции потоков)
- [[async-await]] (Синтаксический сахар над TPL)
- [[ThreadPool]] (Пул потоков CLR)
- [[CancellationToken]] (Паттерн кооперативной отмены)
- [[PLINQ]] (Параллельный LINQ)