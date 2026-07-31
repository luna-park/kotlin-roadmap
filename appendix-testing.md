# Приложение: Тестирование

**Время:** 2–3 вечера
**Когда читать:** после [этапа 8](08-android-project.md) · [Обзор](README.md)

---

## Почему это приложение, а не этап

По всему маршруту тестирование стоит в списках «не учить». Это осознанно: чтобы
писать тесты, нужно сначала иметь что тестировать, а инструментарий для корутин и
Flow — отдельная тема со своими правилами, которая на старте только отвлекает.

Но откладывать бесконечно тоже нельзя: код, который живёт дольше недели, без тестов
начинает ломаться при каждой правке. Это приложение — точка входа: минимальный
набор, которого хватает, чтобы покрыть архитектуру с [этапа 5](05-desktop-project.md).

Хорошая новость: **эта архитектура тестируется легко**. Репозиторий за интерфейсом,
модели без зависимостей от UI, состояние в `StateFlow` — всё это писалось не ради
тестов, но именно они делают тесты возможными.

---

## Что тестировать, а что нет

| Слой | Тестировать | Чем |
|---|---|---|
| Мапперы `Dto → Domain` | да, это чистые функции | обычный JUnit |
| `safeApiCall` и обработка ошибок | да | MockEngine / MockWebServer |
| Репозиторий | да | подменённый API |
| `ViewModel` / модели состояния | **да, в первую очередь** | `runTest` + Turbine |
| `@Composable`-функции | по необходимости | Compose UI Test |
| Room DAO | если есть нетривиальный SQL | instrumentation-тест |
| Сам Ktor, Compose, Room | нет | это чужие библиотеки |

Начинать стоит с моделей состояния: там живёт логика, там больше всего ветвлений,
и тестируются они быстрее всего.

---

## Зависимости

```kotlin
dependencies {
    testImplementation(kotlin("test"))
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:<версия>")
    testImplementation("app.cash.turbine:turbine:<версия>")
    testImplementation("io.ktor:ktor-client-mock:<версия>")
}
```

---

## Корутины в тестах: `runTest`

Главная проблема тестов с корутинами — время. Если код содержит `delay(300)`,
честно ждать эти 300 мс в каждом тесте нельзя.

`runTest` решает это **виртуальным временем**: `delay` возвращается мгновенно, но
порядок выполнения сохраняется.

```kotlin
@Test
fun `debounce отсекает промежуточный ввод`() = runTest {
    val model = SearchModel(fakeRepository)

    model.onQueryChange("k")
    model.onQueryChange("ko")
    model.onQueryChange("kotlin")

    advanceTimeBy(301)          // виртуально проматываем debounce
    runCurrent()

    assertEquals(1, fakeRepository.searchCallCount)   // запрос ушёл один раз
}
```

Что нужно знать:

- `runTest { }` — замена `runBlocking` в тестах. Внутри работает виртуальное время
- `advanceTimeBy(ms)` — промотать время вперёд
- `advanceUntilIdle()` — домотать до момента, когда все корутины отработали
- `runCurrent()` — выполнить задачи, запланированные на текущий момент

**Подмена `Dispatchers.Main`.** Модели с [этапа 5](05-desktop-project.md) создают
область на `Dispatchers.Main`, которого в JVM-тестах нет. Лечится правилом:

```kotlin
class MainDispatcherRule : TestWatcher() {
    private val dispatcher = StandardTestDispatcher()

    override fun starting(description: Description) {
        Dispatchers.setMain(dispatcher)
    }

    override fun finished(description: Description) {
        Dispatchers.resetMain()
    }
}
```

Более чистая альтернатива — **не хардкодить диспетчер**, а передавать его в
конструктор:

```kotlin
class PostsModel(
    private val repository: PostRepository,
    dispatcher: CoroutineDispatcher = Dispatchers.Main
) : Closeable {
    private val scope = CoroutineScope(SupervisorJob() + dispatcher)
    // ...
}
```

Тогда тест просто передаёт `StandardTestDispatcher()`, и никаких правил не нужно.
Это тот случай, когда тестируемость улучшает и сам код.

---

## Flow в тестах: Turbine

Проверять `StateFlow` через `collect` неудобно: он не завершается, и тест повиснет
(ровно та ловушка из упражнения 9 [этапа 2](02-coroutines.md)). Turbine делает это
за тебя:

```kotlin
@Test
fun `при ошибке сети состояние показывает сообщение`() = runTest {
    val repository = FakePostRepository(result = ApiResult.NetworkError)
    val model = PostsModel(repository, StandardTestDispatcher(testScheduler))

    model.state.test {
        assertTrue(awaitItem().isLoading)          // первое состояние — загрузка

        val error = awaitItem()
        assertFalse(error.isLoading)
        assertEquals("Нет соединения", error.error)

        cancelAndIgnoreRemainingEvents()
    }
}
```

- `.test { }` — подписаться на Flow внутри блока
- `awaitItem()` — дождаться следующего значения
- `awaitComplete()` / `awaitError()` — для завершающихся потоков
- `cancelAndIgnoreRemainingEvents()` — закончить, не проверяя остаток
- Если по окончании блока остались непрочитанные значения, Turbine **уронит тест** —
  это защита от «проверил первое, а дальше как-нибудь»

---

## Подмена зависимостей: фейки вместо моков

Для архитектуры этого маршрута мок-библиотека (Mockito, MockK) не нужна.
Репозиторий за интерфейсом — достаточно написать ручную реализацию:

```kotlin
class FakePostRepository(
    private var result: ApiResult<List<Post>> = ApiResult.Success(emptyList())
) : PostRepository {

    var searchCallCount = 0
        private set

    override suspend fun getPosts() = result
    override suspend fun searchPosts(query: String): ApiResult<List<Post>> {
        searchCallCount++
        return result
    }
    override suspend fun getPost(id: Int) = ApiResult.NetworkError
    override suspend fun createPost(title: String, body: String) = ApiResult.NetworkError
    override suspend fun updatePost(id: Int, title: String, body: String) = ApiResult.NetworkError
    override suspend fun deletePost(id: Int) = ApiResult.Success(Unit)

    fun setResult(value: ApiResult<List<Post>>) { result = value }
}
```

Многословнее мока на одну строку, но читается без знания DSL, не ломается при
рефакторинге и работает быстрее. Мок-библиотеки оправданы, когда интерфейс большой
или когда важен порядок вызовов.

---

## Сеть: MockEngine

Ktor даёт подменный движок — реальных запросов не будет, а весь остальной код
(сериализация, `expectSuccess`, `safeApiCall`) работает по-настоящему.

```kotlin
@Test
fun `ответ 500 превращается в HttpError`() = runTest {
    val engine = MockEngine { request ->
        respond(
            content = "",
            status = HttpStatusCode.InternalServerError
        )
    }
    val repository = PostRepositoryImpl(PostApi(createHttpClient(engine)))

    val result = repository.getPosts()

    assertTrue(result is ApiResult.HttpError)
    assertEquals(500, (result as ApiResult.HttpError).code)
}
```

Это самый дешёвый способ проверить всю таблицу ошибок из
[этапа 3](03-network-json.md): 404, 500, битый JSON, таймаут. Для Retrofit
эквивалент — `MockWebServer` из OkHttp.

---

## Именование и структура теста

```kotlin
@Test
fun `пустой заголовок блокирует кнопку сохранения`() = runTest {
    // Arrange — подготовка
    val model = EditModel(FakePostRepository(), postId = null)

    // Act — действие
    model.onTitleChange("   ")

    // Assert — проверка
    assertFalse(model.state.value.canSave)
}
```

Имена в обратных кавычках — особенность Kotlin: в них можно писать пробелы и
кириллицу. Для тестов это удобно, отчёт читается как список требований.

Три части (Arrange–Act–Assert) стоит разделять явно: тест, где непонятно, где
кончается подготовка и начинается проверка, читать тяжело.

---

## С чего начать на практике

Порядок по убыванию отдачи:

1. **Мапперы и `toUserMessage()`** — чистые функции, тесты пишутся за минуты
2. **Модель состояния одного экрана** — все ветви `ApiResult` до сообщения в UI
3. **`safeApiCall` через MockEngine** — вся таблица ошибок
4. **Логика формы** — валидация, `canSave`, режимы создания и правки
5. **Compose UI-тесты** — в последнюю очередь и только для нетривиальных экранов

Не гнаться за процентом покрытия. Пять тестов на логику ветвлений полезнее
пятидесяти на геттеры.

---

## Чего здесь нет

- Compose UI Test (`createComposeRule`, `onNodeWithText`) — отдельная тема
- Instrumentation-тесты и Espresso
- Тестирование Room с `inMemoryDatabaseBuilder`
- Screenshot-тесты
- Моки через MockK и их DSL
- Property-based тестирование
- CI и запуск тестов на сборке

---

## Ссылки

- [Testing coroutines](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-test/)
- [Turbine](https://github.com/cashapp/turbine)
- [Ktor: testing a client](https://ktor.io/docs/client-testing.html)
- [Android: test your app](https://developer.android.com/training/testing)
- [Compose testing](https://developer.android.com/develop/ui/compose/testing)
- [Глоссарий](glossary.md)

---

[Обзор маршрута](README.md)
