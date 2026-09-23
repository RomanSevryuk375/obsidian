---
aliases:
tags:
  - C_sharp
  - dotnet
date: 2026-08-05 18:55
status:
---
## Суть (The What)
**CancellationToken** — это механизм кооперативной отмены асинхронных или длительных синхронных операций в .NET. 

Архитектурная задача: безопасное прерывание работы без использования агрессивных методов ОС (вроде принудительного убийства потоков). Позволяет сэкономить ресурсы процессора, сети и памяти, отбрасывая вычисления, результат которых больше не нужен вызывающей стороне (например, пользователь закрыл страницу браузера или отменил загрузку).

## Как это работает (Under the Hood)

> [!info] Паттерн "Издатель-Подписчик"
> Механизм разделен на два типа для соблюдения инкапсуляции:
> 1. `CancellationTokenSource` (CTS) — класс ([[Reference Types]]), который владеет состоянием и имеет право инициировать отмену (вызвать `.Cancel()`).
> 2. `CancellationToken` — легковесная структура ([[Value Types]]), которая является "read-only" слепком состояния. Передается рабочим методам.

- **Кооперативность**: Среда выполнения (CLR) не прерывает ваш код автоматически. Ваш код обязан сам проверять состояние токена и прекращать работу. Если метод не принимает токен, отменить его извне безопасно невозможно.
- **Два пути отмены**:
  1. *Опрос (Polling)*: Регулярный вызов `token.ThrowIfCancellationRequested()` внутри тяжелых циклов обработки (CPU-bound). Выбрасывает `OperationCanceledException`.
  2. *Колбэки (Push)*: Метод `token.Register(...)` позволяет подписать делегат, который выполнится в момент вызова `Cancel()`. Используется для интеграции с устаревшим кодом или нативными API.
- **IL-код и память**: Структура `CancellationToken` занимает минимум памяти и копируется по значению. Если отмена не запрашивалась (используется `CancellationToken.None`), накладные расходы равны нулю (отсутствуют аллокации в [[Managed Heap]]).

## Практический пример

Реализация сервиса, который объединяет внешний токен (например, от клиента API) и внутренний таймаут, используя `CreateLinkedTokenSource`. 

```csharp
using System;
using System.Net.Http;
using System.Runtime.CompilerServices;
using System.Threading;
using System.Threading.Tasks;

namespace Infrastructure.Processing;

// Использование Primary Constructor (C# 12)
public class DataAggregator(HttpClient httpClient)
{
    // CancellationToken всегда передается последним аргументом со значением по умолчанию
    public async Task<string> AggregateDataAsync(string endpoint, CancellationToken externalToken = default)
    {
        // Создаем локальный источник отмены по таймауту (5 секунд)
        // и связываем его с внешним токеном. Отмена произойдет, если сработает любой из них.
        using var timeoutCts = CancellationTokenSource.CreateLinkedTokenSource(externalToken);
        timeoutCts.CancelAfter(TimeSpan.FromSeconds(5));

        // 1. I/O операция: передаем токен инфраструктурному методу
        using var response = await httpClient.GetAsync(endpoint, timeoutCts.Token);
        response.EnsureSuccessStatusCode();
        var rawData = await response.Content.ReadAsStringAsync(timeoutCts.Token);

        // 2. CPU-bound операция: ручной опрос токена
        return ProcessData(rawData, timeoutCts.Token);
    }

    private string ProcessData(string data, CancellationToken token)
    {
        var span = data.AsSpan(); // Избегаем аллокаций
        var result = new System.Text.StringBuilder();

        for (int i = 0; i < span.Length; i++)
        {
            // В тяжелом цикле проверяем токен каждые N итераций (для микрооптимизации)
            if (i % 100 == 0)
            {
                // Если произошла отмена, выбросит OperationCanceledException
                token.ThrowIfCancellationRequested(); 
            }
            
            // Бизнес-логика обработки...
            result.Append(span[i]);
        }

        return result.ToString();
    }
}
```

## Ошибки и Best Practices

> [!danger] Anti-pattern: Перехват всех исключений (Swallowing Cancellation)
> Если вы используете глобальный `catch (Exception ex)`, вы случайно перехватите `OperationCanceledException` (или производный `TaskCanceledException`). Это сломает механизм отмены — метод вернет пустой результат или `null` вместо корректного прерывания потока выполнения.
> **Решение:** Делайте явный `catch (OperationCanceledException)` и пробрасывайте его дальше (`throw;`), либо фильтруйте: `catch (Exception ex) when (ex is not OperationCanceledException)`.

> [!warning] Ошибка: Утечка памяти при CancelAfter или Linked Tokens
> Если вы создаете `CancellationTokenSource` с таймером (`CancelAfter`) или связываете его с долгоживущим токеном (`CreateLinkedTokenSource`), под капотом регистрируются таймеры ОС или колбэки. Если не вызвать `Dispose()` у такого CTS, объекты зависнут в памяти до срабатывания глобальной отмены.
> **Решение:** Всегда оборачивайте кастомные `CancellationTokenSource` в блок [[Оператор using|using]].

**Best Practices:**
1. **Прокидывание "до талого"**: Передавайте `CancellationToken` во **все** методы цепочки вызовов, от контроллера базы данных (`Entity Framework` или `Dapper`) до сетевого сокета.
2. **Параметр по умолчанию**: В сигнатурах методов делайте токен опциональным: `CancellationToken token = default`. Под капотом `default` для структуры развернется в `CancellationToken.None`.
3. **Не используйте `IsCancellationRequested` для выхода**: Не пишите `if (token.IsCancellationRequested) return null;`. Это нарушает контракт `Task`, ожидающий исключение при отмене. Используйте `ThrowIfCancellationRequested()`.
4. **Интеграция с ASP.NET Core**: Во всех эндпоинтах контроллеров или Minimal API добавляйте токен в параметры. ASP.NET автоматически привяжет его к состоянию HTTP-запроса (если клиент разорвет TCP-соединение, токен отменится).

## Связанные темы
- [[Task Parallel Library (TPL)]]
- [[async-await]]
- [[Оператор using|using]] (Управление ресурсами)
- [[IAsyncEnumerable]] (и атрибут `[EnumeratorCancellation]`)