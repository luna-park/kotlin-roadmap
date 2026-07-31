# Этап 5. 🎯 Desktop-проект

**Время:** 5–7 вечеров
**Предыдущий:** [04. Compose Desktop](04-compose-desktop.md) · **Следующий:** [06. Что изменилось в Android](06-android-changes.md) · [Обзор](README.md)

---

## Цель

Одно работающее приложение, в котором сходятся все четыре предыдущих этапа.
Не «ещё одно упражнение» — именно здесь обнаруживается, что понято не до конца.

Заодно эта кодовая база станет основой для [этапа 8](08-android-project.md):
слои `data` и `domain` переносятся на Android почти дословно.

---

## Постановка задачи

CRUD-клиент к REST API.

**Обязательный функционал:**

- список сущностей с сервера в `LazyColumn`
- экран/диалог создания
- редактирование существующей записи
- удаление с подтверждением
- три состояния экрана: загрузка / данные / ошибка
- кнопка повтора при ошибке
- поиск с `debounce` по введённому тексту

Сущность — любая: посты, пользователи, задачи. Проще всего взять то, что уже
использовалось на [этапе 3](03-network-json.md).

---

## Структура проекта

```
src/main/kotlin/
├── Main.kt                          # точка входа: application { Window { } }
├── di/
│   └── AppContainer.kt              # ручная сборка зависимостей
├── data/
│   ├── remote/
│   │   ├── HttpClientFactory.kt
│   │   ├── PostApi.kt
│   │   └── dto/PostDto.kt
│   ├── mapper/PostMapper.kt
│   └── repository/PostRepositoryImpl.kt
├── domain/
│   ├── model/Post.kt
│   ├── repository/PostRepository.kt  # интерфейс
│   └── ApiResult.kt
└── ui/
    ├── App.kt                       # роутинг по состоянию
    ├── common/
    │   ├── ErrorView.kt
    │   ├── LoadingView.kt
    │   └── ConfirmDialog.kt
    ├── list/
    │   ├── PostsModel.kt
    │   ├── PostsUiState.kt
    │   └── PostsScreen.kt
    └── edit/
        ├── EditModel.kt
        └── EditScreen.kt
```

Пакеты по слоям, внутри UI — по экранам. Это ровно та структура, которую потом
увидишь в Android-проекте.

`domain/repository/PostRepository.kt` — интерфейс, реализация в `data`. Зачем:
чтобы `PostsModel` не знал ни про Ktor, ни про DTO. На [этапе 8](08-android-project.md)
это позволит подменить реализацию на кэширующую, не тронув UI.

---

## Ключевые куски кода

### 5.1. Модель состояния экрана

```kotlin
// ui/list/PostsUiState.kt
data class PostsUiState(
    val posts: List<Post> = emptyList(),
    val query: String = "",
    val isLoading: Boolean = false,      // первая загрузка: показываем спиннер вместо списка
    val isRefreshing: Boolean = false,   // обновление: список остаётся на экране
    val error: String? = null,
    val deletingId: Int? = null
)
```

Два отдельных флага вместо одного — не избыточность. `isLoading` означает
«показывать нечего, рисуем спиннер на весь экран», `isRefreshing` — «данные есть,
идёт обновление поверх них». Слить их в один можно, но тогда при каждом обновлении
список будет мигать на пустой экран. На Android этот же класс переиспользуется без
изменений, и `isRefreshing` там понадобится для pull-to-refresh (см.
[этап 8](08-android-project.md)).

Здесь показан второй подход к состоянию — **один `data class` с флагами** вместо
`sealed class`. Оба валидны:

| | `sealed class` | `data class` с флагами |
|---|---|---|
| Плюс | состояния взаимоисключающие по построению | легко показать список + спиннер поверх |
| Минус | «список есть, но идёт обновление» выразить сложно | возможны бессмысленные комбинации |

**Рекомендация:** попробовать оба. Начать с `sealed class` (проще думать),
и когда потребуется «показать старый список во время обновления» — увидеть его
ограничение своими глазами. Это полезнее, чем прочитать об этом.

### 5.2. Ошибки — в одном месте

Прежде чем писать модели, стоит завести одну функцию, превращающую результат
запроса в текст для человека. Иначе исчерпывающий `when` по `ApiResult` расползётся
по всем моделям, и каждое добавление нового варианта придётся ловить руками.

```kotlin
// ui/common/ErrorMessages.kt
fun ApiResult<*>.toUserMessage(): String? = when (this) {
    is ApiResult.Success -> null
    is ApiResult.NetworkError -> "Нет соединения"
    is ApiResult.HttpError -> "Ошибка сервера: $code"
    is ApiResult.ParseError -> "Неверный формат данных"
}
```

Здесь сходится всё, что разбиралось раньше: `sealed`-тип с
[этапа 1](01-language.md), `when` как выражение, функция-расширение и возврат
`String?`, где `null` означает «ошибки нет».

Исчерпывающий `when` живёт ровно в этом файле, один раз. Добавится пятый вариант —
компилятор укажет сюда, и больше ни на какой файл. По той же причине в моделях
**нигде не появляется `else`**: он бы выключил эту проверку.

Эта же функция без изменений переедет на Android и будет использоваться в
[этапе 7](07-android-basics.md) и [этапе 8](08-android-project.md).

---

### 5.3. Модель списка: ViewModel руками

```kotlin
// ui/list/PostsModel.kt
class PostsModel(private val repository: PostRepository) : Closeable {

    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.Main)

    private val _state = MutableStateFlow(PostsUiState())
    val state: StateFlow<PostsUiState> = _state.asStateFlow()

    private val _events = MutableSharedFlow<String>()
    val events: SharedFlow<String> = _events.asSharedFlow()

    private val queryInput = MutableStateFlow("")
    private var loadJob: Job? = null

    init {
        load()
        observeQuery()
    }

    fun load() {
        loadJob?.cancel()
        loadJob = scope.launch {
            _state.update { it.copy(isLoading = true, error = null) }
            val result = repository.getPosts()
            _state.update {
                it.copy(
                    posts = (result as? ApiResult.Success)?.data ?: it.posts,
                    isLoading = false,
                    error = result.toUserMessage()
                )
            }
        }
    }

    // текст в поле ввода обновляется сразу, запрос — с задержкой
    fun onQueryChange(query: String) {
        queryInput.value = query
        _state.update { it.copy(query = query) }
    }

    @OptIn(FlowPreview::class)
    private fun observeQuery() {
        queryInput
            .debounce(300)               // сначала ждём паузу в наборе
            .distinctUntilChanged()      // потом сравниваем устоявшиеся значения
            .onEach { query -> search(query) }
            .launchIn(scope)
    }

    private suspend fun search(query: String) {
        if (query.isBlank()) {
            load()
            return
        }
        _state.update { it.copy(isRefreshing = true, error = null) }
        val result = repository.searchPosts(query)
        _state.update {
            it.copy(
                posts = (result as? ApiResult.Success)?.data ?: it.posts,
                isRefreshing = false,
                error = result.toUserMessage()
            )
        }
    }

    fun delete(id: Int) {
        scope.launch {
            _state.update { it.copy(deletingId = id) }
            val result = repository.deletePost(id)
            _state.update { s ->
                s.copy(
                    posts = if (result is ApiResult.Success) s.posts.filter { it.id != id }
                            else s.posts,
                    deletingId = null
                )
            }
            _events.emit(result.toUserMessage() ?: "Удалено")
        }
    }

    override fun close() { scope.cancel() }
}
```

Разбор двух неочевидных мест:

- **`debounce` перед `distinctUntilChanged`, а не наоборот.** Обратный порядок
  работает, но хуже: если набрать `a`, дописать `b` и стереть обратно до `a`,
  то `distinctUntilChanged` пропустит всю последовательность (`a` → `ab` → `a`,
  каждое отличается от предыдущего), и запрос по `a` уйдёт второй раз. Сначала
  `debounce` отсекает промежуточные значения, и только устоявшиеся сравниваются
  между собой.
- **Отдельный `queryInput` рядом с `state.query`.** Могло показаться, что можно
  подписаться на `state.map { it.query }` и не заводить второе поле. Технически да,
  но тогда `debounce` будет реагировать на **любое** изменение состояния, включая
  приход данных с сервера, и придётся полагаться на `distinctUntilChanged`, чтобы
  это гасить. Отдельный поток ввода честнее и читается однозначно.

**Это самое важное упражнение этапа.** Написав `ViewModel` руками, ты поймёшь, что
`androidx.lifecycle.ViewModel` — не магия, а тот же класс, у которого:

- `viewModelScope` — это `CoroutineScope(SupervisorJob() + Dispatchers.Main.immediate)`
- `onCleared()` — это вызов `scope.cancel()`
- переживание поворота экрана — работа `ViewModelStore`, а не самого класса

`_state.update { it.copy(...) }` — атомарное обновление. Через `_state.value = ...`
тоже можно, но `update` безопаснее при конкурентных вызовах.

### 5.4. Второй экран: форма

Экран редактирования решает две задачи одним классом: создание (`postId == null`)
и правку существующего (`postId != null`). Это типовая развилка, и заводить под неё
два экрана не нужно.

```kotlin
// ui/edit/EditUiState.kt
data class EditUiState(
    val title: String = "",
    val body: String = "",
    val isLoading: Boolean = false,     // грузим существующий пост
    val isSaving: Boolean = false,
    val error: String? = null,
    val isSaved: Boolean = false        // сигнал экрану: пора закрываться
) {
    // вычисляемое свойство: кнопка активна, только когда есть что сохранять
    val canSave: Boolean get() = title.isNotBlank() && !isSaving && !isLoading
}
```

`canSave` — вычисляемое свойство (`get()` без хранимого значения, см.
[этап 1](01-language.md)). Логика «когда кнопка активна» живёт в состоянии, а не в
разметке: UI остаётся глупым и просто читает флаг.

```kotlin
// ui/edit/EditModel.kt
class EditModel(
    private val repository: PostRepository,
    private val postId: Int?              // null = создание нового
) : Closeable {

    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.Main)

    private val _state = MutableStateFlow(EditUiState())
    val state: StateFlow<EditUiState> = _state.asStateFlow()

    init {
        if (postId != null) loadExisting(postId)
    }

    private fun loadExisting(id: Int) {
        scope.launch {
            _state.update { it.copy(isLoading = true) }
            val result = repository.getPost(id)
            _state.update {
                val post = (result as? ApiResult.Success)?.data
                it.copy(
                    title = post?.title ?: it.title,
                    body = post?.body ?: it.body,
                    isLoading = false,
                    error = result.toUserMessage()
                )
            }
        }
    }

    fun onTitleChange(value: String) = _state.update { it.copy(title = value) }
    fun onBodyChange(value: String) = _state.update { it.copy(body = value) }

    fun save() {
        val current = _state.value
        if (!current.canSave) return

        scope.launch {
            _state.update { it.copy(isSaving = true, error = null) }
            val result = if (postId == null) {
                repository.createPost(current.title, current.body)
            } else {
                repository.updatePost(postId, current.title, current.body)
            }
            _state.update {
                it.copy(
                    isSaving = false,
                    error = result.toUserMessage(),
                    isSaved = result is ApiResult.Success
                )
            }
        }
    }

    override fun close() { scope.cancel() }
}
```

Флаг `isSaved` — приём, который стоит запомнить: модель не умеет и не должна уметь
закрывать экран, она лишь сообщает, что дело сделано. Решение о навигации принимает
UI:

```kotlin
// ui/edit/EditScreen.kt
@Composable
fun EditScreen(model: EditModel, onDone: () -> Unit) {
    val state by model.state.collectAsState()

    // как только сохранение прошло — уходим назад
    LaunchedEffect(state.isSaved) {
        if (state.isSaved) onDone()
    }

    Column(
        modifier = Modifier.fillMaxSize().padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(12.dp)
    ) {
        OutlinedTextField(
            value = state.title,
            onValueChange = model::onTitleChange,
            label = { Text("Заголовок") },
            singleLine = true,
            isError = state.title.isBlank(),
            modifier = Modifier.fillMaxWidth()
        )

        OutlinedTextField(
            value = state.body,
            onValueChange = model::onBodyChange,
            label = { Text("Текст") },
            minLines = 5,
            modifier = Modifier.fillMaxWidth()
        )

        state.error?.let { message ->
            Text(message, color = MaterialTheme.colorScheme.error)
        }

        Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            Button(onClick = model::save, enabled = state.canSave) {
                Text(if (state.isSaving) "Сохранение…" else "Сохранить")
            }
            TextButton(onClick = onDone) { Text("Отмена") }
        }
    }
}
```

`model::save` — ссылка на метод (см. [глоссарий](glossary.md), символ `::`);
короче, чем `{ model.save() }`, и делает то же самое.

---

### 5.5. Общие компоненты

Три состояния экрана — загрузка, ошибка, пусто — повторяются на обоих экранах.
Чтобы не дублировать, они живут в `ui/common/`:

```kotlin
// ui/common/LoadingView.kt
@Composable
fun LoadingView(modifier: Modifier = Modifier) {
    Box(modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
        CircularProgressIndicator()
    }
}

// ui/common/ErrorView.kt
@Composable
fun ErrorView(
    message: String,
    onRetry: () -> Unit,
    modifier: Modifier = Modifier
) {
    Column(
        modifier = modifier.fillMaxSize().padding(24.dp),
        verticalArrangement = Arrangement.spacedBy(12.dp, Alignment.CenterVertically),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text(message, style = MaterialTheme.typography.bodyLarge)
        Button(onClick = onRetry) { Text("Повторить") }
    }
}
```

Оба — stateless и принимают `modifier` первым опциональным параметром, как
требует соглашение из [этапа 4](04-compose-desktop.md). Именно поэтому их можно
переиспользовать без изменений на Android.

---

### 5.6. Ручной DI

```kotlin
// di/AppContainer.kt
class AppContainer {
    private val json = Json { ignoreUnknownKeys = true }

    private val httpClient by lazy { createHttpClient(json) }
    private val api by lazy { PostApi(httpClient) }

    val postRepository: PostRepository by lazy { PostRepositoryImpl(api) }

    fun close() = httpClient.close()
}
```

Hilt/Koin сюда не нужны. Одна ленивая цепочка объектов делает то же самое и
показывает, что DI-фреймворк решает проблему масштаба, а не принципа.

### 5.7. Роутинг без библиотеки

```kotlin
// ui/App.kt
sealed interface Screen {
    data object List : Screen
    data class Edit(val postId: Int?) : Screen     // null = создание
}

@Composable
fun App(container: AppContainer) {
    var screen: Screen by remember { mutableStateOf(Screen.List) }

    when (val s = screen) {
        is Screen.List -> {
            val model = rememberModel { PostsModel(container.postRepository) }
            PostsScreen(
                model = model,
                onAdd = { screen = Screen.Edit(null) },
                onEdit = { id -> screen = Screen.Edit(id) }
            )
        }
        is Screen.Edit -> {
            val model = rememberModel(s.postId) { EditModel(container.postRepository, s.postId) }
            EditScreen(model = model, onDone = { screen = Screen.List })
        }
    }
}
```

### 5.8. Отмена моделей — то, что легко забыть

`remember { PostsModel(...) }` создаёт объект с собственным `CoroutineScope`, но
уничтожать его никто не будет: при уходе с экрана composable покидает композицию,
а модель остаётся жить со своими корутинами. Это ровно та утечка, от которой
избавляет `viewModelScope` на Android — и на desktop её надо закрывать руками.

```kotlin
// ui/common/RememberModel.kt
@Composable
fun <T : Closeable> rememberModel(vararg keys: Any?, factory: () -> T): T {
    val model = remember(*keys) { factory() }
    DisposableEffect(model) {
        onDispose { model.close() }
    }
    return model
}
```

Тогда модели достаточно реализовать `Closeable`:

```kotlin
class PostsModel(private val repository: PostRepository) : Closeable {
    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.Main)
    // ...
    override fun close() { scope.cancel() }
}
```

`DisposableEffect(model)` сработает и при смене ключа: уходя с `Edit(1)` на
`Edit(2)`, старая модель закроется, новая создастся. Проверяется просто — поставить
лог в `close()` и походить между экранами.

Именно этот шаблон на [этапе 7](07-android-basics.md) исчезнет целиком: `ViewModel`
получает `onCleared()` от системы, и `rememberModel` больше не нужен.

Навигационная библиотека на двух экранах — избыточна. На Android
[этап 7](07-android-basics.md) заменит это на Navigation Compose, и разница будет
понятна: там появляется back stack, deep links и системная кнопка «назад».

### 5.9. Точка входа

```kotlin
fun main() = application {
    val container = remember { AppContainer() }

    Window(
        onCloseRequest = {
            container.close()
            exitApplication()
        },
        title = "Posts",
        state = rememberWindowState(width = 900.dp, height = 700.dp)
    ) {
        MaterialTheme { App(container) }
    }
}
```

---

## Опциональный блок: свой сервер на Ktor

**2–3 вечера, но сильно повышает отдачу от этапа.**

Проблема публичных API: у них нельзя вызвать ошибку по требованию.
jsonplaceholder всегда отвечает 200 и всегда быстро. Обработку ошибок толком
не протестировать.

Свой сервер решает это за 50 строк:

```kotlin
// server/src/main/kotlin/Server.kt
fun main() {
    embeddedServer(Netty, port = 8080) {
        install(ContentNegotiation) { json() }

        val posts = mutableListOf(
            PostDto(1, "Первый", "Текст"),
            PostDto(2, "Второй", "Текст")
        )
        var nextId = 3
        var failMode = false

        routing {
            get("/posts") {
                if (failMode) return@get call.respond(HttpStatusCode.InternalServerError)
                delay(500)                              // имитация сети
                call.respond(posts)
            }
            get("/posts/{id}") {
                val id = call.parameters["id"]?.toIntOrNull()
                val post = posts.find { it.id == id }
                if (post == null) call.respond(HttpStatusCode.NotFound)
                else call.respond(post)
            }
            post("/posts") {
                val body = call.receive<PostDto>()
                val created = body.copy(id = nextId++)
                posts += created
                call.respond(HttpStatusCode.Created, created)
            }
            put("/posts/{id}") { /* ... */ }
            delete("/posts/{id}") {
                val id = call.parameters["id"]?.toIntOrNull()
                posts.removeAll { it.id == id }
                call.respond(HttpStatusCode.NoContent)
            }

            // управление поведением для тестов
            post("/debug/fail") { failMode = true;  call.respond(HttpStatusCode.OK) }
            post("/debug/ok")   { failMode = false; call.respond(HttpStatusCode.OK) }
            get("/debug/slow")  { delay(30_000); call.respond("ok") }
        }
    }.start(wait = true)
}
```

Зависимости: `ktor-server-core`, `ktor-server-netty`,
`ktor-server-content-negotiation`, `ktor-serialization-kotlinx-json`.

Что это даёт:

- проверить `NetworkError`, `HttpError(500)`, таймаут — по кнопке
- увидеть, что реально уходит в запросе (логи сервера)
- понять обе стороны контракта: почему сервер отдаёт 201 на create и 204 на delete
- **важно для Android:** на эмуляторе `localhost` — это сам эмулятор.
  Адрес хост-машины — `10.0.2.2`. Об этом на [этапе 8](08-android-project.md)

Общие DTO лучше вынести в отдельный Gradle-модуль `shared`, подключённый и к
клиенту, и к серверу. Тогда изменение контракта сразу ломает компиляцию на обеих
сторонах — это полезно.

---

## Порядок работы (по вечерам)

| Вечер | Что делать | Разделы |
|---|---|---|
| 1 | Скелет: структура пакетов, DI-контейнер, Ktor-клиент, API, DTO, домен-модель, маппер. Проверить репозиторий из `main()` без UI | 5.6 |
| 2 | Состояние и ошибки: `PostsUiState`, `toUserMessage()`, модель списка | 5.1–5.3 |
| 3 | Экран списка: `LazyColumn`, три состояния, `LaunchedEffect`, retry | 5.5 |
| 4 | Экран редактирования: форма, валидация, create и update, возврат к списку | 5.4 |
| 5 | Роутинг, отмена моделей, точка входа, удаление с `AlertDialog`, `Snackbar` через `SharedFlow` | 5.7–5.9 |
| 6 | Поиск с `debounce`; (опц.) свой сервер и проверка всех сценариев ошибок | 5.3 |
| 7 | Рефакторинг: вынести stateless-компоненты, убрать дублирование, `git log` привести в порядок | — |

Порядок «сначала модель, потом экран» (вечера 2 и 3) выбран сознательно: модель
проверяется из `main()` печатью состояний, без UI. Так ошибки логики не смешиваются
с ошибками разметки — это тот же принцип, по которому весь маршрут начинается с
desktop.

Седьмой вечер не пропускать. Первая версия всегда получается со состоянием в
`@Composable` и логикой в UI; переписывание этого своими руками — самая полезная
часть этапа.

---

## Не делать на этом этапе

На интеграционном этапе главный риск — не пропустить тему, а расширить задачу до
того, как заработает основное. Что отложить:

- **DI-фреймворк** (Koin, Hilt). `AppContainer` руками — и есть содержание этапа
- **Навигационную библиотеку.** `when` по `sealed`-типу хватает на два экрана;
  back stack появится на Android вместе с системной кнопкой «назад»
- **Абстракции «на будущее»:** use case на каждое действие, отдельные mapper-классы,
  собственные Result-обёртки поверх `ApiResult`. Три слоя — уже достаточно
- **Многомодульность.** Один модуль, максимум второй (`shared`) под общие DTO,
  если делаешь сервер
- **Тесты.** Не потому, что не нужны, а потому что тестирование корутин и Flow —
  отдельная тема со своим инструментарием. Первый проект пишется без них,
  второй — уже нет. Стартовый набор — в [приложении](appendix-testing.md)
- **Темизацию и полировку UI.** Дефолтный Material 3 выглядит достаточно, чтобы
  не отвлекаться
- **Логирование через библиотеку.** `println` на desktop достаточно

---

## Чекпоинт

Функционал:

- [ ] Список грузится с сервера, показывается спиннер во время загрузки
- [ ] При ошибке — сообщение и работающая кнопка «Повторить»
- [ ] Создание, редактирование, удаление работают, список обновляется
- [ ] Удаление спрашивает подтверждение
- [ ] Поиск с `debounce`, не дёргает сервер на каждую букву
- [ ] Snackbar на успешных операциях

Архитектура:

- [ ] В `@Composable`-функциях **нет** ни `HttpClient`, ни `ApiResult`, ни DTO
- [ ] Все переиспользуемые компоненты stateless, принимают `modifier`
- [ ] `ViewModel`-классы не знают ни про Compose, ни про Ktor
- [ ] Все четыре ветви `ApiResult` обработаны, каждая даёт своё сообщение
- [ ] `HttpClient` создаётся один раз и закрывается при выходе
- [ ] Область корутин каждой модели отменяется при уходе с экрана — проверено
      логом в `close()`, а не «по идее»
- [ ] Область `AppContainer` и `HttpClient` закрываются при закрытии окна
- [ ] Список обновляется через `copy()`, без мутаций

Понимание:

- [ ] Можешь объяснить, что делает `viewModelScope` — потому что написал его аналог
- [ ] Можешь объяснить, почему запрос в теле `@Composable` — ошибка
- [ ] Понятно, чем `StateFlow` отличается от `SharedFlow` и почему snackbar —
      через второй

---

## Типовые проблемы этапа

**Запрос уходит по многу раз.** Загрузка вызвана в теле `@Composable`, а не в
`LaunchedEffect`. Либо `LaunchedEffect(state)` с ключом, который меняется от самой
загрузки — получается цикл.

**UI не обновляется после изменения списка.** Список мутирован, а не пересоздан.
Либо `_state.value` присвоен тот же объект.

**Модель создаётся заново при каждой рекомпозиции.** Забыт `remember` вокруг
`PostsModel(...)`.

**Старые модели продолжают работать после ухода с экрана.** `remember` есть, а
`close()` не вызывается: composable покинул композицию, объект с живым `scope`
остался. Симптом — накапливающиеся запросы и логи от экранов, которых уже нет.
Лечится `rememberModel` из раздела выше. Проверяется логом в `close()`.

**Snackbar показывается повторно при возврате на экран.** События сделаны через
`StateFlow`, а не `SharedFlow`. `StateFlow` отдаёт последнее значение каждому
новому подписчику — для событий это неверно.

**Приложение не закрывается.** Корутины в области с обычным `Job` не отменены,
`HttpClient` не закрыт.

---

## Ссылки

- [Guide to app architecture](https://developer.android.com/topic/architecture) —
  формулирует ровно ту трёхслойную схему, что собирается на этом этапе
- [UI layer / UI state](https://developer.android.com/topic/architecture/ui-layer) —
  про выбор между `sealed class` и `data class` с флагами
- [StateFlow and SharedFlow](https://kotlinlang.org/docs/flow.html#stateflow-and-sharedflow)
- [Compose: state hoisting](https://developer.android.com/develop/ui/compose/state-hoisting)
- [Ktor Server: routing](https://ktor.io/docs/server-routing.html) — для
  опционального блока со своим сервером
- [Ktor Server: quick start](https://ktor.io/docs/server-create-a-new-project.html)
- [Глоссарий](glossary.md) — служебные слова Kotlin и термины, если что-то по ходу осталось непонятным

---

## Что дальше

Кодовая база готова к переносу. На Android поедут почти без изменений:

- `domain/` — целиком
- `data/` — с заменой движка Ktor (`CIO` → `OkHttp`)
- `ui/*Model.kt` — с заменой своего скоупа на `viewModelScope`
- `ui/*Screen.kt` — с заменой `collectAsState` на `collectAsStateWithLifecycle`

Заменится только точка входа, роутинг и сборка.

---

**Следующий этап:** [06. Что изменилось в Android](06-android-changes.md)
