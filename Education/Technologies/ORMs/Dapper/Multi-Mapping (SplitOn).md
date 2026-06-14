---
aliases:
tags:
  - C_sharp
  - dotnet
  - ORMs
  - dapper
date: 2026-06-12 20:39
status:
---
### Суть (The "What")
**Multi-Mapping** — это механизм библиотеки [[Dapper]], позволяющий преобразовать одну плоскую строку (row), полученную в результате выполнения [[SQL]]-запроса с `JOIN`, в несколько связанных независимых [[MOC|C#]]-объектов.

Этот механизм решает задачу **объектно-реляционного отображения сложных графов**. Реляционные базы данных возвращают плоские декартовы произведения (дерево таблиц "схлопывается" в таблицу колонок), а Multi-Mapping берет на себя рутину по "нарезке" этих колонок обратно в иерархию объектов, избавляя от ручной работы с `SqlDataReader`.

---

### Как это работает (The "How" / Nutshell style)

#### 1. Чтение слева направо (Left-to-Right Sweep)
При использовании методов вроде `QueryAsync<T1, T2, TReturn>` Dapper сканирует возвращенные колонки строго слева направо. 
*   Сначала он маппит колонки в объект `T1`.
*   Когда он встречает колонку, имя которой совпадает со значением параметра **`splitOn`**, он понимает, что начались данные для объекта `T2`.
*   Процесс продолжается для всех переданных generic-типов (Dapper поддерживает маппинг до 7 типов в одной строке).

#### 2. Параметр splitOn
По умолчанию значение `splitOn` равно `"Id"`. Если ваши первичные ключи называются иначе (например, `UserId` и `RoleId`), вы **обязаны** явно передать параметр `splitOn: "UserId,RoleId"`. Dapper использует разделитель (запятую), чтобы сопоставить точки разреза с generic-типами по порядку.

#### 3. Функция склейки (Map Delegate)
Dapper создает экземпляры `T1` и `T2` в памяти и передает их в предоставленный вами делегат `Func<T1, T2, TReturn>`. Внутри этого делегата разработчик должен указать, как именно объекты связаны (например, вложить `T2` в свойство `T1`), и вернуть итоговый объект.

#### 4. Проблема декартова произведения (1-ко-многим)
SQL-запрос с `INNER JOIN` или `LEFT JOIN` для отношения "Один-ко-многим" (например, 1 Пост имеет 5 Комментариев) вернет 5 строк. Dapper по умолчанию **не знает** об этой связи и создаст 5 *разных* объектов `Post` в куче.
*   **Решение:** Разработчик должен использовать механизм замыкания (closure) — обычно словарь в памяти (`Dictionary`), чтобы дедублицировать родительские объекты.

> [!INFO] Нюанс: IL Генерация
> Dapper динамически генерирует IL-код для маппинга каждого фрагмента строки (нарезанного через `splitOn`). Если в строке SQL значение колонки `splitOn` равно `NULL` (например, при `LEFT JOIN`, когда дочерней записи нет), Dapper передаст в делегат `null` вместо создания пустого объекта `T2`. Вы должны обрабатывать этот `null` в лямбде.

---

### Пример кода ([[MOC|C#]] 12)

Реализация отношений "Один-к-одному" и "Один-ко-многим" с использованием первичных конструкторов и неизменяемых рекордов.

```csharp
using System.Collections.Generic;
using System.Data;
using System.Linq;
using System.Threading.Tasks;
using Dapper;

namespace DataAccess;

// DTO представлены как record для удобства и иммутабельности
public record ProfileDto(int ProfileId, string Bio);
public record UserDto(int UserId, string Name, ProfileDto? Profile = null);

public record CommentDto(int CommentId, string Text);
public class PostDto
{
    public int PostId { get; set; }
    public string Title { get; set; } = string.Empty;
    public List<CommentDto> Comments { get; set; } = [];
}

public class MultiMappingRepository(IDbConnection dbConnection)
{
    private readonly IDbConnection _db = dbConnection;

    // 1. Отношение 1-к-1
    public async Task<IEnumerable<UserDto>> GetUsersWithProfilesAsync()
    {
        const string sql = """
            SELECT u.UserId, u.Name, p.ProfileId, p.Bio
            FROM Users u
            LEFT JOIN Profiles p ON u.UserId = p.UserId
            """;

        // Типы: T1(User), T2(Profile), TReturn(User)
        return await _db.QueryAsync<UserDto, ProfileDto, UserDto>(
            sql,
            map: (user, profile) => user with { Profile = profile },
            splitOn: "ProfileId" // Указываем, где начинаются данные Profile
        );
    }

    // 2. Отношение 1-ко-многим (Дедубликация через Словарь)
    public async Task<IEnumerable<PostDto>> GetPostsWithCommentsAsync()
    {
        const string sql = """
            SELECT p.PostId, p.Title, c.CommentId, c.Text
            FROM Posts p
            LEFT JOIN Comments c ON p.PostId = c.PostId
            """;

        // Словарь-накопитель для дедубликации постов
        var postDictionary = new Dictionary<int, PostDto>();

        await _db.QueryAsync<PostDto, CommentDto, PostDto>(
            sql,
            map: (post, comment) =>
            {
                // Если пост еще не встречался, добавляем в словарь
                if (!postDictionary.TryGetValue(post.PostId, out var currentPost))
                {
                    currentPost = post;
                    postDictionary.Add(currentPost.PostId, currentPost);
                }

                // Обработка LEFT JOIN (если у поста нет комментариев, comment будет null)
                if (comment is not null)
                {
                    currentPost.Comments.Add(comment);
                }

                return currentPost;
            },
            splitOn: "CommentId"
        );

        // ВАЖНО: Возвращаем значения из словаря, а не результат QueryAsync,
        // так как QueryAsync содержит дубликаты родительских объектов.
        return postDictionary.Values;
    }
}
```

---

### Ошибки и Best Practices

> [!DANGER] Anti-pattern: Забытый или неверный splitOn
> Если вы напишете `JOIN`, где дочерняя таблица не имеет колонки `Id`, но забудете передать параметр `splitOn: "CustomId"`, Dapper выбросит исключение `ArgumentException`: *"When using the multi-mapping APIs ensure you set the splitOn param..."*. Dapper просто не поймет, где нужно разрезать строку.

> [!WARNING] Ошибка: Возврат результата Dapper при 1-ко-многим
> В сценарии "Один-ко-многим" метод `QueryAsync` вернет ровно столько элементов, сколько строк вернул SQL. Если один пост имеет 10 комментариев, в `IEnumerable` будет 10 ссылок на один и тот же объект `PostDto`. Вы всегда должны возвращать `dictionary.Values` (или `.Distinct()`), чтобы получить корректный список уникальных постов.

**Best Practices:**
1.  **Порядок в SQL имеет значение:** В вашем SQL-запросе колонки `SELECT` должны идти строго в том же порядке, что и generic-параметры `QueryAsync<T1, T2>`. Сначала все колонки `T1`, затем `splitOn` колонка и все остальные колонки `T2`.
2.  **Использование LEFT JOIN:** Почти всегда при маппинге нужно использовать `LEFT JOIN`. Если использовать `INNER JOIN`, вы потеряете родительские объекты, у которых нет дочерних записей. Dapper корректно маппит отсутствующие дочерние записи в `null` объекты.
3.  **Многоуровневый маппинг:** Dapper поддерживает сигнатуры `QueryAsync` до 7 типов (`T1, T2, T3...`). Строка `splitOn` в этом случае должна передаваться через запятую, например: `splitOn: "CommentId, AuthorId"`.
4.  **[[CQRS]] View Models:** Multi-mapping идеально подходит для построения сложных View Models ([[DTO]] для ответов API) за один сетевой вызов к БД, избегая проблемы "N+1 запросов", которая часто встречается в тяжелых [[ORM]] при ленивой загрузке.

> [!TIP] Нюанс: Псевдонимы колонок (Aliases)
> Если в двух объединяемых таблицах есть колонки с одинаковыми именами (кроме `splitOn`), Dapper автоматически смаппит их в соответствующие объекты `T1` и `T2` в зависимости от их позиции до или после точки разреза. Использовать `AS` алиасы в SQL не обязательно, если порядок колонок правильный.