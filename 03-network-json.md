# Этап 3. Сеть и JSON

**Время:** 3 вечера
**Предыдущий:** [02b. Flow](02b-flow.md) · **Следующий:** [04. Compose Desktop](04-compose-desktop.md) · [Обзор](README.md)

---

## Цель

Рабочий слой данных: `Repository` с `suspend`-методами, который скрывает от
остального кода HTTP, JSON и исключения, а наружу отдаёт домен-модели или
`sealed class` с результатом.

Знания по HTTP переносятся один в один — учить нужно только синтаксис.

---

## Почему Ktor, а не Retrofit

| | Ktor Client | Retrofit |
|---|---|---|
| Платформы | JVM, Android, iOS, JS, Native | только JVM/Android |
| API | функции + `suspend` | аннотации на интерфейсе |
| Подходит для desktop-этапа | ✅ | ✅, но нетипично |
| Распространённость в Android | растёт | стандарт де-факто |

Ktor выбран потому, что на этом маршруте один и тот же код должен работать на
desktop и на Android. Retrofit тоже работает на desktop, но там его почти никто
не использует, и документация вся Android-ориентированная.

**Retrofit придётся выучить позже** — он стоит в большинстве существующих
Android-проектов. Это займёт один вечер: концепции те же (`suspend`-функции,
конвертер JSON, интерцепторы), меняется только способ описания эндпоинтов.
Разбор с построчным сопоставлением — в [приложении](appendix-retrofit.md);
читать его имеет смысл после этого этапа, когда Ktor-версия уже работает.

---

## Зависимости

```kotlin
plugins {
    kotlin("jvm")
    kotlin("plugin.serialization") version "2.0.20"   // ← отдельный компилятор-плагин
}

dependencies {
    implementation("io.ktor:ktor-client-core:$ktor")
    implementation("io.ktor:ktor-client-cio:$ktor")                    // движок для JVM
    implementation("io.ktor:ktor-client-content-negotiation:$ktor")    // авто JSON
    implementation("io.ktor:ktor-serialization-kotlinx-json:$ktor")
    implementation("io.ktor:ktor-client-logging:$ktor")

    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.7.1")
}
```

Три вещи, которые часто пропускают:

1. **`plugin.serialization` — это плагин компилятора.** Без него `@Serializable`
   не сгенерирует сериализатор и упадёт в рантайме. Одна из самых частых ошибок
2. **Движок (engine) отделён от ядра.** На JVM/desktop — `cio` (чистый Kotlin) или
   `okhttp`/`apache`. На Android позже поставим `okhttp`, поменяв одну строку
3. `ContentNegotiation` — отдельный плагин, без него `body<User>()` не заработает

---

## Темы

Три блока: модели данных и их разбор (3.1–3.5), сам HTTP-клиент (3.6–3.10) и
слой, который прячет и то, и другое от остального приложения (3.11–3.14).

### 3.1. Модели данных: `@Serializable`

```kotlin
@Serializable
data class PostDto(
    val id: Int,
    val userId: Int,
    val title: String,
    val body: String,
    @SerialName("created_at") val createdAt: String? = null,  // snake_case в JSON
    val coverUrl: String? = null                              // nullable + default
)
```

Одна и та же сущность `PostDto` используется до конца этапа — и в примерах
сериализации, и в запросах, и в репозитории.

- `@Serializable` на `data class` — этого достаточно
- `@SerialName` — когда имя в JSON не совпадает с полем Kotlin
- **Nullable-поле и поле со значением по умолчанию — разные вещи.** Если поле
  может отсутствовать в JSON, нужно значение по умолчанию; если может прийти
  `null` — нужен `?`. Часто нужно и то, и то: `val x: String? = null`
- `@Transient` — не сериализовать поле
- `@EncodeDefault` — управление тем, писать ли значения по умолчанию при сериализации

### 3.2. Настройка парсера

```kotlin
val json = Json {
    ignoreUnknownKeys = true      // ← обязательно: сервер добавит поле — не упадём
    isLenient = false
    encodeDefaults = false
    explicitNulls = false         // не писать null-поля при сериализации
    prettyPrint = false
}
```

`ignoreUnknownKeys = true` — практически всегда нужно. По умолчанию `false`, и
любое неизвестное поле в ответе — исключение. Это ловушка номер один.

### 3.3. Ручное использование

```kotlin
val post: PostDto = json.decodeFromString(jsonString)
val text: String = json.encodeToString(post)

val posts: List<PostDto> = json.decodeFromString(jsonArrayString)
```

### 3.4. Что понадобится реже

- Вложенные объекты и списки — работают автоматически, ничего делать не нужно
- Полиморфизм: `sealed class` + `@Serializable` + дискриминатор `type` —
  когда сервер отдаёт разнотипные элементы в одном массиве
- `JsonElement` / `JsonObject` — для навигации по неизвестной структуре
- Кастомные сериализаторы (`KSerializer`) — например, для даты. **Отложить:**
  проще принять дату строкой и распарсить в маппере

### 3.5. Отличие от Gson / Jackson

- Работает через кодогенерацию, **не через рефлексию** → быстрее и не ломается
  ProGuard/R8 на Android
- Требует `@Serializable`: нельзя сериализовать произвольный чужой класс
- Понимает nullability Kotlin: `val name: String` и пришедший `null` — это
  исключение, а не тихое `null` в non-null поле (Gson именно так и делает)
- Понимает значения по умолчанию конструктора (Gson — нет)

---

### 3.6. Создание клиента

```kotlin
val client = HttpClient(CIO) {
    expectSuccess = true          // ← 4xx/5xx станут исключениями; см. «Обработка ошибок»

    install(ContentNegotiation) {
        json(Json { ignoreUnknownKeys = true })
    }
    install(Logging) {
        level = LogLevel.BODY        // на этапе изучения — BODY, потом HEADERS
    }
    install(HttpTimeout) {
        requestTimeoutMillis = 15_000
        connectTimeoutMillis = 10_000
        socketTimeoutMillis = 15_000
    }
    defaultRequest {
        url("https://jsonplaceholder.typicode.com/")
        header("Accept", "application/json")
    }
}
```

`HttpClient` — **тяжёлый объект**. Один на приложение, не создавать на каждый
запрос. Закрывать через `client.close()` при завершении.

### 3.7. Запросы

Все запросы удобно собрать в один класс — это и есть тот `PostApi`, который дальше
получает репозиторий:

```kotlin
// data/remote/PostApi.kt
class PostApi(private val client: HttpClient) {

    // GET
    suspend fun getPosts(): List<PostDto> =
        client.get("posts").body()

    // GET с параметрами и путём
    suspend fun getPost(id: Int): PostDto =
        client.get("posts/$id") {
            parameter("expand", "author")
        }.body()

    // POST с телом
    suspend fun createPost(post: PostDto): PostDto =
        client.post("posts") {
            contentType(ContentType.Application.Json)
            setBody(post)                                // сериализуется автоматически
        }.body()

    // PUT
    suspend fun updatePost(id: Int, post: PostDto): PostDto =
        client.put("posts/$id") {
            contentType(ContentType.Application.Json)
            setBody(post)
        }.body()

    // DELETE
    suspend fun deletePost(id: Int) {
        client.delete("posts/$id")
    }

    // поиск; если API его не умеет — см. оговорку в конце 3.14
    suspend fun searchPosts(query: String): List<PostDto> =
        client.get("posts/search") {
            parameter("q", query)
        }.body()
}
```

Что заметить:

- Все методы — `suspend`. Никаких `Call`, `enqueue`, колбэков
- `.body<T>()` — десериализация. Тип берётся из контекста (`: List<PostDto>`)
  или указывается явно: `.body<List<PostDto>>()`
- `contentType(...)` нужен явно для тела; можно вынести в `defaultRequest`
- Ktor сам выбирает диспетчер, **`withContext(Dispatchers.IO)` не нужен**

### 3.8. Заголовки и авторизация

```kotlin
client.get("profile") {
    header("Authorization", "Bearer $token")
    headers {
        append("X-Request-Id", uuid)
        append(HttpHeaders.AcceptLanguage, "ru")
    }
}
```

Для постоянных заголовков — `defaultRequest { }` при создании клиента.
Плагин `Auth` с refresh-токенами — **отложить**, это отдельная тема.

### 3.9. Ответ целиком

```kotlin
val response: HttpResponse = client.get("posts")

response.status                    // HttpStatusCode
response.status.value              // Int
response.status.isSuccess()        // Boolean
response.headers["Content-Type"]
response.bodyAsText()              // сырой текст — полезно при отладке
```

### 3.10. Обработка ошибок

Ktor по умолчанию **не бросает** исключение на 4xx/5xx: `client.get(...)` вернёт
объект ответа со статусом 500, и `.body<T>()` попытается разобрать тело ошибки как
вашу модель. Проверять приходится вручную:

```kotlin
val response = client.get("posts")
if (!response.status.isSuccess()) { /* обработка */ }
```

Именно поэтому в конфигурации клиента выше стоит **`expectSuccess = true`** — тогда
4xx превращается в `ClientRequestException`, 5xx в `ServerResponseException`, и
обработка ошибок собирается в одном месте (`safeApiCall`, см. 3.12). Без этого
флага `safeApiCall` вернёт `Success` на ответе 500 — это самая частая причина,
по которой «обработка ошибок написана, но не срабатывает».

Дерево исключений делится на две части, и это различие важнее, чем кажется.

**Не зависят от движка** — их даёт само ядро Ktor, они одинаковы на CIO, OkHttp и
любом другом движке:

| Исключение | Причина |
|---|---|
| `ClientRequestException` | 4xx (при `expectSuccess = true`) |
| `ServerResponseException` | 5xx |
| `RedirectResponseException` | 3xx без автоследования |
| `HttpRequestTimeoutException` | таймаут запроса, задан в `HttpTimeout` |
| `SerializationException` | JSON не соответствует модели (это kotlinx, не Ktor) |

**Зависят от движка** — «сети нет» каждый движок сообщает своими исключениями,
потому что под капотом это разные транспорты:

| Движок | Что прилетает при отсутствии сети |
|---|---|
| CIO (JVM, desktop) | `UnresolvedAddressException` (`java.nio.channels`), `ConnectException` |
| OkHttp (Android) | `UnknownHostException`, `ConnectException`, `SocketTimeoutException` |

> **Это единственное место, где data-слой не переносится на Android дословно.**
> На [этапе 8](08-android-project.md) движок меняется на OkHttp, и написанный под
> CIO `catch (e: UnresolvedAddressException)` перестанет срабатывать — молча.
> Симптом: в режиме полёта приложение показывает не «нет соединения», а общую
> ошибку из последней ветви `catch`.

Чтобы не переписывать `safeApiCall` при переходе, стоит сразу ловить общего
предка вместо конкретных классов:

```kotlin
catch (e: IOException) {          // java.io.IOException
    ApiResult.NetworkError        // покрывает UnknownHost, Connect, SocketTimeout
}
```

`UnknownHostException`, `ConnectException` и `SocketTimeoutException` — все
наследники `IOException`. Исключение из правила — `UnresolvedAddressException`:
оно наследует `IllegalArgumentException`, а не `IOException`, поэтому для CIO
нужна отдельная ветвь. Итоговый вариант в 3.12 учитывает и то, и другое.

---


Это то, ради чего этап. Правило: **DTO не выходят за пределы репозитория**,
исключения не выходят наружу.

### 3.11. Модель результата

```kotlin
sealed interface ApiResult<out T> {
    data class Success<T>(val data: T) : ApiResult<T>
    data class HttpError(val code: Int, val message: String) : ApiResult<Nothing>
    data object NetworkError : ApiResult<Nothing>
    data class ParseError(val message: String) : ApiResult<Nothing>
}
```

Альтернатива — встроенный `kotlin.Result<T>`. Он проще, но не различает виды ошибок,
а UI обычно нужно показать «нет сети» иначе, чем «сервер вернул 500». Свой
`sealed interface` для этого удобнее.

### 3.12. Обёртка вызовов

```kotlin
suspend fun <T> safeApiCall(block: suspend () -> T): ApiResult<T> = try {
    ApiResult.Success(block())
} catch (e: CancellationException) {
    throw e                                       // ← не глотать отмену
} catch (e: ClientRequestException) {
    ApiResult.HttpError(e.response.status.value, e.message)
} catch (e: ServerResponseException) {
    ApiResult.HttpError(e.response.status.value, "Ошибка сервера")
} catch (e: HttpRequestTimeoutException) {
    ApiResult.NetworkError
} catch (e: UnresolvedAddressException) {
    ApiResult.NetworkError                        // движок CIO: нет DNS
} catch (e: IOException) {
    ApiResult.NetworkError                        // OkHttp и всё остальное транспортное
} catch (e: SerializationException) {
    ApiResult.ParseError(e.message ?: "Неверный формат ответа")
} catch (e: Exception) {
    ApiResult.NetworkError
}
```

Порядок ветвей — не случайный, `catch` проверяет их сверху вниз:

- `CancellationException` **строго первой**, иначе её проглотит любая ветвь ниже
  и отмена корутины превратится в «ошибку сети»
- `ClientRequestException` и `ServerResponseException` до `IOException`: они его
  наследники, и общая ветвь перехватила бы их первой, потеряв код ответа
- `UnresolvedAddressException` отдельно от `IOException`, потому что не наследует
  его (см. выше)
- `IOException` покрывает `UnknownHostException`, `ConnectException`,
  `SocketTimeoutException` — то есть весь Android-набор без перечисления
- финальный `catch (e: Exception)` — страховка, чтобы наружу не улетело ничего
  неожиданного

Благодаря двум ветвям — `UnresolvedAddressException` и `IOException` — этот файл
работает и на CIO, и на OkHttp без правок. Это и есть та оговорка, из-за которой
data-слой всё-таки переносится на Android дословно.

### 3.13. Домен-модель и маппер

```kotlin
// domain
data class Post(val id: Int, val title: String, val body: String)

// mapper — extension-функция на DTO
fun PostDto.toDomain() = Post(
    id = id,
    title = title.trim(),
    body = body.ifBlank { "(нет текста)" }
)
```

Зачем отдельная домен-модель, если поля почти те же: чтобы изменение формата на
сервере правилось в одном файле, а не по всему UI. На учебном проекте это может
показаться лишним — но именно эта граница делает возможным добавление кэша на
[этапе 7](07-android-basics.md) без переписывания UI.

### 3.14. Репозиторий целиком

Репозиторий разделён на интерфейс и реализацию. Интерфейс живёт в `domain` и не
знает ни про Ktor, ни про DTO; реализация — в `data`:

```kotlin
// domain/repository/PostRepository.kt
interface PostRepository {
    suspend fun getPosts(): ApiResult<List<Post>>
    suspend fun getPost(id: Int): ApiResult<Post>
    suspend fun createPost(title: String, body: String): ApiResult<Post>
    suspend fun updatePost(id: Int, title: String, body: String): ApiResult<Post>
    suspend fun deletePost(id: Int): ApiResult<Unit>
    suspend fun searchPosts(query: String): ApiResult<List<Post>>
}
```

```kotlin
// data/repository/PostRepositoryImpl.kt
class PostRepositoryImpl(private val api: PostApi) : PostRepository {

    override suspend fun getPosts(): ApiResult<List<Post>> = safeApiCall {
        api.getPosts().map { it.toDomain() }
    }

    override suspend fun getPost(id: Int): ApiResult<Post> = safeApiCall {
        api.getPost(id).toDomain()
    }

    override suspend fun createPost(title: String, body: String): ApiResult<Post> = safeApiCall {
        api.createPost(PostDto(id = 0, userId = 1, title = title, body = body)).toDomain()
    }

    override suspend fun updatePost(id: Int, title: String, body: String): ApiResult<Post> =
        safeApiCall {
            api.updatePost(id, PostDto(id = id, userId = 1, title = title, body = body)).toDomain()
        }

    override suspend fun deletePost(id: Int): ApiResult<Unit> = safeApiCall {
        api.deletePost(id)
    }

    override suspend fun searchPosts(query: String): ApiResult<List<Post>> = safeApiCall {
        api.searchPosts(query).map { it.toDomain() }
    }
}
```

Наружу — только `suspend`-функции с `ApiResult`. Ни `HttpClient`, ни DTO,
ни исключений.

Разделение на интерфейс и реализацию на этом этапе выглядит избыточным: реализация
одна, подменять нечем. Оправдается оно на [этапе 7](07-android-basics.md), где
появится вторая реализация — с кэшем в Room, — и UI об этой замене не узнает.
Структура пакетов, в которой всё это лежит, разобрана на
[этапе 5](05-desktop-project.md).

> **Про `searchPosts`.** У многих тестовых API поиска нет: `jsonplaceholder` его не
> умеет, `dummyjson.com` умеет (`/posts/search?q=...`). Если выбранный API поиска не
> поддерживает — оставь метод в интерфейсе, а в реализации фильтруй локально по уже
> загруженному списку. Для [этапа 5](05-desktop-project.md) важно не откуда приходят
> результаты, а что поиск идёт через `debounce` и не блокирует ввод.

---

## Не учить на этом этапе

- Retrofit (пока) — разбор в [приложении](appendix-retrofit.md); OkHttp напрямую
- Плагин `Auth`, refresh-токены, OAuth
- WebSocket, Server-Sent Events
- Multipart / загрузка файлов
- Кэширование на уровне HTTP, `HttpCache`
- Свои плагины Ktor, интерцепторы
- Кастомные `KSerializer`, полиморфная сериализация
- Protobuf, CBOR
- MockEngine и тестирование сети — [приложение](appendix-testing.md)

---

## Тестовые API

| API | Особенности |
|---|---|
| `jsonplaceholder.typicode.com` | полный CRUD, без ключа, но записи не сохраняются |
| `reqres.in` | пагинация, задержки (`?delay=3`), реалистичные ошибки |
| `dummyjson.com` | много сущностей, поиск, авторизация |
| `httpbin.org` | эхо-сервис: посмотреть, что именно ты отправил |

`httpbin.org/post` особенно полезен на первом часу: он возвращает твой запрос
обратно, и сразу видно, правильно ли сформированы заголовки и тело.

Для проверки обработки ошибок: `httpbin.org/status/500`, `reqres.in/api/users?delay=10`.

---

## Упражнения

1. **httpbin.** Отправить POST с телом и кастомным заголовком на
   `httpbin.org/post`, распечатать ответ через `bodyAsText()`. Убедиться, что
   отправилось именно то, что задумано.
2. **Десериализация.** Получить `List<PostDto>` с jsonplaceholder. Затем удалить
   `ignoreUnknownKeys` и добавить в DTO лишнее обязательное поле — посмотреть на
   оба типа ошибок.
3. **CRUD.** Все четыре метода для `posts`, каждый — отдельная `suspend`-функция.
4. **Ошибки.** Обратиться к `httpbin.org/status/404` и `/status/500`,
   убедиться, что `safeApiCall` возвращает `HttpError` с правильным кодом.
   Отключить Wi-Fi — получить `NetworkError`.
5. **Таймаут.** `reqres.in/api/users?delay=10` при `requestTimeoutMillis = 2000`.
6. **Параллельно.** Через `async` загрузить posts, users и comments одновременно;
   сравнить время с последовательной загрузкой (связка с
   [этапом 2](02-coroutines.md)).
7. **Итоговое.** Консольный CRUD-клиент: меню в цикле (`readLine()`), пункты
   «список», «показать по id», «создать», «изменить», «удалить», «выход».
   Архитектура: `Api` → `Repository` → `main`. Все результаты через `ApiResult`,
   каждая ошибка — с внятным сообщением на русском.

---

## Чекпоинт

- [ ] Плагин `kotlin("plugin.serialization")` подключён, и понятно, зачем
- [ ] `ignoreUnknownKeys = true` стоит, и понятно, что будет без него
- [ ] `expectSuccess = true` стоит, и проверено: ответ 500 доходит до `HttpError`
- [ ] Все четыре HTTP-метода работают
- [ ] `HttpClient` создаётся один раз, не на каждый запрос
- [ ] Ни одно исключение не выходит за пределы репозитория
- [ ] `CancellationException` пробрасывается в `safeApiCall`
- [ ] DTO и домен-модель разделены, есть маппер
- [ ] Понятно, почему `withContext(Dispatchers.IO)` вокруг Ktor не нужен
- [ ] Упражнение 7 работает целиком

---

## Ссылки

- [Ktor Client: create and configure](https://ktor.io/docs/client-create-and-configure.html)
- [Ktor: making requests](https://ktor.io/docs/client-requests.html)
- [Ktor: response validation](https://ktor.io/docs/client-response-validation.html)
- [Ktor: content negotiation and serialization](https://ktor.io/docs/client-serialization.html)
- [kotlinx.serialization guide](https://github.com/Kotlin/kotlinx.serialization/blob/master/docs/serialization-guide.md)
- [kotlinx.serialization: JSON features](https://github.com/Kotlin/kotlinx.serialization/blob/master/docs/json.md)
- [Глоссарий](glossary.md) — служебные слова Kotlin и термины, если что-то по ходу осталось непонятным

---

**Следующий этап:** [04. Compose Desktop](04-compose-desktop.md)
