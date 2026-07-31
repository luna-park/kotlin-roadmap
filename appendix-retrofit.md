# Приложение: Retrofit

**Время:** 1 вечер
**Когда читать:** после [этапа 3](03-network-json.md), а лучше — после
[этапа 8](08-android-project.md) · [Обзор](README.md)

---

## Зачем это приложение

Маршрут построен на Ktor Client, потому что он одинаково работает на desktop и на
Android. Но **в существующих Android-проектах почти всегда стоит Retrofit** — он
появился раньше, и переписывать рабочий сетевой слой никто не будет.

Поэтому Retrofit нужно уметь читать и править, даже если новое пишешь на Ktor.
Концепции те же: `suspend`-функции, конвертер JSON, перехватчики, обработка ошибок.
Меняется только способ описания эндпоинтов.

Отдельно: Retrofit **не устарел**. Это зрелая и активно используемая библиотека,
и выбор в пользу Ktor на этом маршруте связан с переносимостью, а не с качеством.

---

## Разница в одной таблице

| | Ktor Client | Retrofit |
|---|---|---|
| Описание запросов | обычные функции с вызовами `client.get(...)` | интерфейс с аннотациями |
| Реализация | пишешь сам | генерируется библиотекой |
| Платформы | JVM, Android, iOS, JS, Native | только JVM/Android |
| HTTP-движок | сменный (`CIO`, `OkHttp`, …) | всегда OkHttp |
| JSON | плагин `ContentNegotiation` | конвертер-фабрика |
| Ошибки 4xx/5xx | `expectSuccess = true` → исключение | исключение только для `suspend`-методов; либо `Response<T>` |
| Перехватчики | плагины Ktor | интерцепторы OkHttp |

---

## Зависимости

```kotlin
dependencies {
    implementation("com.squareup.retrofit2:retrofit:<версия>")
    implementation("com.squareup.okhttp3:logging-interceptor:<версия>")

    // JSON: kotlinx.serialization — тот же, что на этапе 3
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:<версия>")
    implementation("com.jakewharton.retrofit:retrofit2-kotlinx-serialization-converter:<версия>")
}
```

Конвертер можно взять и на Gson или Moshi — они чаще встречаются в старых проектах.
Но если проект уже собран по этому маршруту, `@Serializable`-классы из
[этапа 3](03-network-json.md) переиспользуются как есть, и менять DTO не нужно.

---

## Описание API

Вместо класса с функциями — интерфейс с аннотациями. Реализацию Retrofit
сгенерирует сам.

```kotlin
interface PostApi {

    @GET("posts")
    suspend fun getPosts(): List<PostDto>

    @GET("posts/{id}")
    suspend fun getPost(@Path("id") id: Int): PostDto

    @GET("posts/search")
    suspend fun searchPosts(@Query("q") query: String): List<PostDto>

    @POST("posts")
    suspend fun createPost(@Body post: PostDto): PostDto

    @PUT("posts/{id}")
    suspend fun updatePost(@Path("id") id: Int, @Body post: PostDto): PostDto

    @DELETE("posts/{id}")
    suspend fun deletePost(@Path("id") id: Int)

    @GET("profile")
    suspend fun profile(@Header("Authorization") token: String): ProfileDto
}
```

Соответствие с Ktor построчное:

| Ktor | Retrofit |
|---|---|
| `client.get("posts")` | `@GET("posts")` |
| `client.get("posts/$id")` | `@GET("posts/{id}")` + `@Path("id")` |
| `parameter("q", query)` | `@Query("q") query: String` |
| `setBody(post)` | `@Body post: PostDto` |
| `header("Authorization", t)` | `@Header("Authorization") token: String` |

Ключевое: **`suspend` в сигнатуре** — именно он превращает вызов в обычную
корутинную функцию. Без него метод должен возвращать `Call<T>`, и это старый
колбэчный стиль, который в новом коде не нужен.

---

## Сборка клиента

```kotlin
private val json = Json { ignoreUnknownKeys = true }

private val okHttp = OkHttpClient.Builder()
    .addInterceptor(HttpLoggingInterceptor().apply {
        level = HttpLoggingInterceptor.Level.BODY      // в release — NONE
    })
    .connectTimeout(10, TimeUnit.SECONDS)
    .readTimeout(15, TimeUnit.SECONDS)
    .build()

val api: PostApi = Retrofit.Builder()
    .baseUrl("https://jsonplaceholder.typicode.com/")   // ← слэш в конце обязателен
    .client(okHttp)
    .addConverterFactory(json.asConverterFactory("application/json".toMediaType()))
    .build()
    .create(PostApi::class.java)
```

Две вещи, на которых спотыкаются:

- **`baseUrl` обязан заканчиваться слэшем**, иначе Retrofit бросит исключение при
  сборке. Пути в аннотациях, наоборот, пишутся **без** ведущего слэша: ведущий
  слэш означает «от корня домена» и отбросит путь из `baseUrl`
- **Таймауты и логирование живут в OkHttp**, а не в Retrofit. Retrofit — это
  только слой поверх; всё транспортное настраивается в `OkHttpClient.Builder`

---

## Ошибки

Здесь отличие от Ktor существенное, и его важно понять.

`suspend`-метод, возвращающий `T`, бросает исключение на любом ответе, кроме 2xx:

| Исключение | Причина |
|---|---|
| `HttpException` | код ответа не 2xx; код доступен как `e.code()` |
| `IOException` и наследники | нет сети, таймаут, обрыв |
| `SerializationException` | JSON не подошёл под модель |

То есть `safeApiCall` с [этапа 3](03-network-json.md) переписывается почти
дословно — меняются только два типа:

```kotlin
suspend fun <T> safeApiCall(block: suspend () -> T): ApiResult<T> = try {
    ApiResult.Success(block())
} catch (e: CancellationException) {
    throw e
} catch (e: HttpException) {                       // ← вместо ClientRequest/ServerResponse
    ApiResult.HttpError(e.code(), e.message())
} catch (e: IOException) {                         // ← покрывает всё транспортное
    ApiResult.NetworkError
} catch (e: SerializationException) {
    ApiResult.ParseError(e.message ?: "Неверный формат ответа")
} catch (e: Exception) {
    ApiResult.NetworkError
}
```

Ветвь `UnresolvedAddressException` не нужна: движок всегда OkHttp, а он бросает
наследников `IOException`.

**Альтернатива — `Response<T>`.** Если объявить метод как
`suspend fun getPosts(): Response<List<PostDto>>`, исключения на 4xx/5xx не будет,
а статус придётся проверять вручную:

```kotlin
val response = api.getPosts()
if (response.isSuccessful) {
    val body = response.body()
} else {
    val code = response.code()
    val errorText = response.errorBody()?.string()    // ← читается ровно один раз
}
```

Это нужно, когда сервер возвращает осмысленное тело ошибки, которое надо разобрать.
В остальных случаях версия с исключениями короче.

---

## Что не меняется вообще

Это главная мысль приложения. При замене Ktor на Retrofit **остаётся нетронутым**:

- `ApiResult` и вся модель ошибок
- домен-модели и мапперы `PostDto.toDomain()`
- интерфейс `PostRepository` и его реализация — меняется только тип поля `api`
- `toUserMessage()`
- `ViewModel`, `UiState`, весь UI

Меняются два файла: описание API и сборка клиента. Именно ради этого на
[этапе 3](03-network-json.md) заводился слой репозитория — чтобы смена сетевой
библиотеки не доходила до экрана.

---

## Упражнения

1. Переписать `PostApi` с [этапа 3](03-network-json.md) на Retrofit-интерфейс,
   не трогая ничего выше репозитория.
2. Собрать клиент, проверить, что все пять методов работают.
3. Обратиться к `httpbin.org/status/404` и убедиться, что `HttpException` даёт
   код 404 в `ApiResult.HttpError`.
4. Убрать `suspend` из одного метода и посмотреть, что предложит компилятор.
5. Убрать завершающий слэш из `baseUrl` — увидеть исключение и запомнить его текст.
6. Переписать один метод на `Response<T>` и прочитать `errorBody()`. Прочитать
   дважды — убедиться, что второй раз тело пустое.

---

## Чекпоинт

- [ ] Понятно, чем `suspend fun getPosts(): List<T>` отличается от версии с `Call<T>`
- [ ] Понятно, что таймауты и логирование настраиваются в OkHttp, а не в Retrofit
- [ ] Понятно правило слэшей в `baseUrl` и путях
- [ ] `safeApiCall` переписан на `HttpException` + `IOException`
- [ ] Проверено: репозиторий, домен и UI не изменились ни на строку

---

## Ссылки

- [Retrofit](https://square.github.io/retrofit/)
- [OkHttp: interceptors](https://square.github.io/okhttp/features/interceptors/)
- [kotlinx.serialization converter](https://github.com/JakeWharton/retrofit2-kotlinx-serialization-converter)
- [Глоссарий](glossary.md)

---

[Обзор маршрута](README.md)
