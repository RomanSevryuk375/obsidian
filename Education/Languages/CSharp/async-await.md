---
aliases:
tags:
  - C_sharp
  - dotnet
date: 2026-08-05 18:46
status:
---
## Суть (The What)
Ключевые слова `async` и `await` — это синтаксический сахар на уровне компилятора поверх [[Task Parallel Library (TPL)]]. 

Главная архитектурная задача — **асинхронность без блокировки потоков**. Они позволяют освободить текущий поток (вернуть его в [[ThreadPool]]) на время ожидания длительной I/O-операции (обращение к БД, сети, диску). Это кардинально повышает пропускную способность (Scalability) серверных приложений и сохраняет отзывчивость интерфейса в десктопных клиентах.

## Как это работает (Under the Hood)

> [!info] Конечный автомат (State Machine)
> Для [[CLR]] методов `async` не существует. На этапе компиляции [[Roslyn]] преобразует каждый `async`-метод в невидимую структуру, реализующую интерфейс `IAsyncStateMachine`. 

- **Разделение метода**: Компилятор "разрезает" тело метода по ключевым словам `await`. Код до первого `await` выполняется синхронно в вызывающем потоке.
- **Освобождение потока**: Когда выполнение доходит до `await` незавершенной задачи, конечный автомат сохраняет свое состояние (локальные переменные) в управляемой куче ([[Managed Heap]]) и возвращает управление вызывающему коду. Поток не блокируется.
- **Аппаратные прерывания (I/O-bound)**: Во время ожидания ответа по сети ни один управляемый поток не простаивает. ОС передает запрос сетевому драйверу, а по завершении использует механизм *I/O Completion Ports (IOCP)* для уведомления среды исполнения.
- **Продолжение (Continuation)**: Когда I/O-операция завершена, планировщик ставит задачу-продолжение (`MoveNext()` конечного автомата) в очередь пула потоков. Свободный поток подхватывает состояние и продолжает выполнение метода со следующей строчки.
- **Контекст синхронизации**: Если в приложении присутствует [[SynchronizationContext]] (как в WPF, WinForms), продолжение будет принудительно отправлено обратно в UI-поток. (В ASP.NET Core контекст синхронизации отсутствует).

## Практический пример

Современный пример (C# 12) создания и потребления асинхронного потока данных через [[IAsyncEnumerable]]. Это позволяет обрабатывать огромные наборы данных по частям, не загружая их целиком в память.

```csharp
using System;
using System.Collections.Generic;
using System.Runtime.CompilerServices;
using System.Threading;
using System.Threading.Tasks;

namespace Infrastructure.Async;

public class DataStreamer(IApiClient apiClient) // C# 12 Primary Constructor
{
    // Генерация асинхронного потока с использованием yield return
    public async IAsyncEnumerable<string> FetchDataChunksAsync(
        [EnumeratorCancellation] CancellationToken cancellationToken = default)
    {
        int page = 1;
        while (!cancellationToken.IsCancellationRequested)
        {
            // Поток возвращается в ThreadPool во время HTTP-запроса
            var chunk = await apiClient.GetPageAsync(page, cancellationToken);
            
            if (chunk.IsEmpty) 
                break;

            // Возвращаем элемент подписчику. Состояние метода сохраняется.
            yield return chunk.Data;
            page++;
        }
    }
}

public class DataProcessor(DataStreamer streamer)
{
    public async Task ProcessAsync(CancellationToken token)
    {
        // Потребление асинхронного потока. 
        // Цикл приостанавливается (await), ожидая каждую новую порцию данных.
        await foreach (var dataChunk in streamer.FetchDataChunksAsync(token))
        {
            Console.WriteLine($"Processed: {dataChunk}");
        }
    }
}
```

## Ошибки и Best Practices

> [!danger] Anti-pattern: async void
> Никогда не используйте `async void`, за исключением обработчиков событий UI (Event Handlers). Метод `async void` не возвращает `Task`, а значит, вызывающий код не может дождаться его завершения (`await`) или перехватить исключения в блоке `try-catch`. Необработанное исключение внутри `async void` обрушит весь процесс приложения.
> **Решение:** Всегда возвращайте `async Task` или `async Task<T>`.

> [!warning] Anti-pattern: Eliding async/await (Пропуск ключевых слов)
> Иногда разработчики убирают `async/await` и просто возвращают `Task`, чтобы сэкономить на аллокации State Machine: `public Task<int> Get() => _repo.GetAsync();`. 
> Это опасно: если `_repo.GetAsync()` выбросит синхронное исключение (до возврата `Task`), или если метод обернут в блок `using`, ресурсы будут освобождены **до** завершения задачи, и приложение упадет с `ObjectDisposedException`. Делайте это только в крайне простых прокси-методах без `using` и `try-catch`.

**Best Practices:**
1. **[[ValueTask]] для горячих путей**: Если метод часто возвращает результат синхронно (например, данные уже есть в in-memory кэше), возвращайте `[[ValueTask]]` вместо [[Task]]. Это предотвратит выделение лишних объектов в куче ([[Garbage Collector]]) при каждом вызове.
2. **Библиотечный код и ConfigureAwait(false)**: Если вы пишете переиспользуемую библиотеку (NuGet-пакет), всегда добавляйте `.ConfigureAwait(false)` к каждому `await`. Это укажет конечной машине не пытаться вернуть выполнение в исходный [[SynchronizationContext]], что предотвратит потенциальные [[Deadlock|взаимоблокировки]] у потребителей вашего кода с UI-потоками.
3. **CPU-bound vs I/O-bound**: Не оборачивайте тяжелые математические вычисления в `async` методы. `async/await` предназначен для I/O. Если нужно освободить поток для CPU-bound задачи, используйте `Task.Run()` на стороне вызывающего кода.

## Связанные темы
- [[Task Parallel Library (TPL)]]
- [[IAsyncEnumerable]]
- [[SynchronizationContext]]
- [[ValueTask]]
- [[Deadlock]]
- [[CancellationToken]]