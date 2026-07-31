# Этап 7. Android-минимум

**Время:** 5–7 вечеров
**Предыдущий:** [06. Что изменилось в Android](06-android-changes.md) · **Следующий:** [08. Android-проект](08-android-project.md) · [Обзор](README.md)

---

## Цель

Добрать то, чего нет на desktop: `Activity`, `ViewModel` из AndroidX, навигацию,
lifecycle-aware подписки, локальное хранение. Язык, корутины, сеть и Compose уже
известны — их учить не нужно.

---

## Что физически меняется при переходе

Приятная часть этапа: список очень короткий.

| Было на desktop | Стало на Android |
|---|---|
| `fun main() = application { Window { } }` | `class MainActivity : ComponentActivity()` |
| свой `CoroutineScope` в модели | `viewModelScope` |
| `collectAsState()` | `collectAsStateWithLifecycle()` |
| `HttpClient(CIO)` | `HttpClient(OkHttp)` |
| `when (screen)` для роутинга | `NavHost` |
| `AppContainer` руками | тот же `AppContainer` (или Hilt позже) |
| — | `AndroidManifest.xml`, permissions |

Всё остальное — `domain`, `data`, `Repository`, `UiState`, `@Composable`-экраны —
переносится дословно.

---

## Темы

### 7.1. Проект и сборка

**Android Studio** — теперь нужен: эмулятор, Logcat, Layout Inspector, App
Inspection, профилировщик.

Создание: New Project → **Empty Activity** (это шаблон с Compose, не путать с
«Empty Views Activity»).

`app/build.gradle.kts`, минимально нужное:

```kotlin
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("org.jetbrains.kotlin.plugin.compose")
    id("org.jetbrains.kotlin.plugin.serialization")
}

android {
    namespace = "com.example.posts"
    compileSdk = 35

    defaultConfig {
        applicationId = "com.example.posts"
        minSdk = 24
        targetSdk = 35
        versionCode = 1
        versionName = "1.0"
    }

    buildFeatures { compose = true }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
    kotlinOptions { jvmTarget = "17" }
}

dependencies {
    val composeBom = platform("androidx.compose:compose-bom:<версия>")
    implementation(composeBom)

    implementation("androidx.core:core-ktx:<версия>")
    implementation("androidx.activity:activity-compose:<версия>")
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:<версия>")
    implementation("androidx.lifecycle:lifecycle-runtime-compose:<версия>")   // collectAsStateWithLifecycle
    implementation("androidx.navigation:navigation-compose:<версия>")

    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.ui:ui-graphics")
    implementation("androidx.compose.ui:ui-tooling-preview")   // сама аннотация @Preview
    implementation("androidx.compose.material3:material3")
    implementation("androidx.compose.material:material-icons-extended")
    debugImplementation("androidx.compose.ui:ui-tooling")       // рендер превью в IDE

    implementation("io.ktor:ktor-client-okhttp:<версия>")     // ← вместо cio
    implementation("io.ktor:ktor-client-content-negotiation:<версия>")
    implementation("io.ktor:ktor-serialization-kotlinx-json:<версия>")
}
```

Что заметить:

- **BOM** (`compose-bom`) — согласует версии всех Compose-артефактов, поэтому у
  них не указывается версия
- `minSdk = 24` — разумный минимум сегодня; ниже начинаются проблемы с
  десугарингом и библиотеками
- `namespace` заменил `package` из манифеста

`AndroidManifest.xml`:

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <uses-permission android:name="android.permission.INTERNET" />

    <application
        android:name=".PostsApp"
        android:label="@string/app_name"
        android:theme="@style/Theme.Posts">

        <activity
            android:name=".MainActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```

`INTERNET` — не runtime-разрешение, спрашивать не нужно. Без него запросы падают
с `SecurityException`.

---

### 7.2. Эмулятор и Logcat

Два инструмента, которые нужны с первой минуты и которых не было на desktop.

#### Эмулятор

Создаётся через Device Manager (иконка телефона справа или Tools → Device Manager
→ Create Device).

- **Образ:** брать вариант с пометкой **Google APIs** (не «Google Play») — он
  быстрее, и в нём доступен root для отладки. «Google Play» нужен, только если
  тестируешь покупки или сервисы Google
- **API level:** ставить равным `targetSdk` проекта. Отдельно полезно завести
  второе устройство с `minSdk` — именно там всплывают проблемы совместимости
- **ABI:** `x86_64` на Intel/AMD, `arm64-v8a` на Apple Silicon. Несовпадение с
  архитектурой хоста означает эмуляцию процессора и работу в разы медленнее

Полезные мелочи: холодный старт эмулятора долгий, поэтому его не закрывают между
запусками; поворот экрана — `Ctrl+←` / `Ctrl+→`; режим полёта и слабая сеть
задаются в расширенных настройках (три точки на панели эмулятора → Cellular).

#### Logcat

Основной инструмент отладки: системный лог всех процессов. Открывается вкладкой
Logcat внизу окна.

```kotlin
private const val TAG = "PostsApp"

Log.d(TAG, "загрузка началась")                  // debug — рабочая отладка
Log.i(TAG, "получено ${posts.size} записей")     // info — важные события
Log.w(TAG, "пустой ответ")                       // warning
Log.e(TAG, "ошибка загрузки", throwable)         // error + стектрейс
```

Что нужно уметь сразу:

- **Фильтр по своему приложению.** По умолчанию видны логи всей системы, а это
  тысячи строк в секунду. В строке фильтра: `package:mine` — только своё
  приложение
- **Фильтр по тегу:** `tag:PostsApp`. Комбинируется: `package:mine tag:PostsApp`
- **Уровень:** выпадающий список слева; `Warn` и выше отсекает шум
- **Стектрейс исключения** — единственная строка с `E` и длинным «хвостом» с
  `at com.example...`. Читать снизу вверх: первая строка с твоим пакетом и есть
  место, где всё сломалось
- **`Log.e(TAG, "текст", throwable)`** — третий параметр печатает стектрейс
  целиком. Без него в логе окажется только сообщение, и причина потеряется

**Логи не должны попадать в release.** `Log.d` в собранном приложении будет
работать и может печатать чувствительные данные. Простой способ — обёртка:

```kotlin
fun logd(message: String) {
    if (BuildConfig.DEBUG) Log.d(TAG, message)
}
```

#### Когда запрос не уходит

Типовая ситуация первого дня: код вроде правильный, но данных нет. Порядок
проверки:

1. **Включить логирование Ktor** — плагин `Logging` с `level = LogLevel.BODY`
   (см. [этап 3](03-network-json.md)). В Logcat появятся URL, заголовки и тело.
   Если их нет — запрос вообще не отправлялся, проблема в коде выше
2. **`INTERNET` в манифесте.** Без него — `SecurityException` в логе
3. **HTTP вместо HTTPS** на API 28+ блокируется молча. Ищи в логе
   `CLEARTEXT communication ... not permitted`
4. **`localhost` на эмуляторе** — это сам эмулятор. Адрес хост-машины `10.0.2.2`
   (подробно на [этапе 8](08-android-project.md))
5. **Исключение проглочено.** Если `safeApiCall` возвращает общую ошибку, добавь в
   ветку `catch` временный `Log.e(TAG, "api", e)` — увидишь настоящий тип исключения

---

### 7.3. Activity как точка входа

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContent {
            PostsTheme {
                AppNavHost()
            }
        }
    }
}
```

Это весь код `Activity` в Compose-проекте. Ни `findViewById`, ни
`setContentView(R.layout...)`, ни ссылок на View.

- `ComponentActivity`, а не `AppCompatActivity` — второй нужен только для
  XML-совместимости
- `setContent { }` — мост из Android в Compose
- `enableEdgeToEdge()` — рисование под системными панелями (см.
  [этап 6](06-android-changes.md))

**Жизненный цикл** нужно освежить, но теперь по другой причине: не «где хранить
состояние», а «почему здесь его хранить нельзя».

```
onCreate → onStart → onResume → [работа] → onPause → onStop → onDestroy
```

Что важно на практике:

- поворот экрана = `onDestroy` + `onCreate` заново
- `onStop` — приложение ушло в фон; здесь останавливаются подписки на UI
- **`ViewModel` этот цикл переживает** — она уничтожается только при
  окончательном завершении, не при пересоздании

#### Application

```kotlin
class PostsApp : Application() {
    lateinit var container: AppContainer
        private set

    override fun onCreate() {
        super.onCreate()
        container = AppContainer()
    }
}
```

Единственная причина завести свой `Application` — держать DI-контейнер из
[этапа 5](05-desktop-project.md). Больше ничего в него класть не надо.

---

### 7.4. ViewModel

```kotlin
class PostsViewModel(
    private val repository: PostRepository
) : ViewModel() {

    private val _state = MutableStateFlow(PostsUiState())
    val state: StateFlow<PostsUiState> = _state.asStateFlow()

    init { load() }

    fun load() {
        viewModelScope.launch {                  // ← вместо своего scope
            _state.update { it.copy(isLoading = true, error = null) }
            val result = repository.getPosts()
            _state.update {
                it.copy(
                    posts = (result as? ApiResult.Success)?.data ?: it.posts,
                    isLoading = false,
                    error = result.toUserMessage()     // тот же маппер с этапа 5
                )
            }
        }
    }
}
```

Разница с классом из [этапа 5](05-desktop-project.md) — **буквально одна строка**:
`scope.launch` заменён на `viewModelScope.launch`. Поля `_state`, `_events`,
`queryInput`, методы `load`, `onQueryChange`, `observeQuery`, `search`, `delete`
и маппер `toUserMessage()` переносятся дословно. Убирается только `Closeable` и
`close()` — их роль берёт на себя `onCleared()`.

Это и есть главный ответ на вопрос «что такое `ViewModel`»: не новый инструмент,
а тот же класс с готовой областью корутин и точкой уничтожения от системы.

- `viewModelScope` = `CoroutineScope(SupervisorJob() + Dispatchers.Main.immediate)`
  с автоматической отменой в `onCleared()`
- Правила: **никогда** не давать `ViewModel` ссылку на `Context`, `Activity` или
  `View`. Если нужен контекст приложения — `AndroidViewModel` (лучше избегать)
- `SavedStateHandle` — переживание убийства процесса. Знать о существовании,
  на этом маршруте не требуется

#### Factory

`ViewModel` с параметрами конструктора требует фабрику:

```kotlin
class PostsViewModel(private val repository: PostRepository) : ViewModel() {
    companion object {
        fun factory(repository: PostRepository) = viewModelFactory {
            initializer { PostsViewModel(repository) }
        }
    }
}
```

Использование:

```kotlin
@Composable
fun PostsScreen(container: AppContainer) {
    val viewModel: PostsViewModel = viewModel(
        factory = PostsViewModel.factory(container.postRepository)
    )
    // ...
}
```

Это то место, где Hilt избавляет от шаблона. Но сначала — руками: тогда видно,
какую именно проблему Hilt решает.

---

### 7.5. Compose на Android

Тот же Compose, что на [этапе 4](04-compose-desktop.md). Отличия:

```kotlin
@Composable
fun PostsScreen(viewModel: PostsViewModel) {
    val state by viewModel.state.collectAsStateWithLifecycle()   // ← не collectAsState
    // ...
}
```

`collectAsStateWithLifecycle()` дополнительно останавливает сбор, когда экран в
фоне (`onStop`), и возобновляет при возврате. `collectAsState()` тоже работает,
но продолжает подписку в фоне — лишний расход батареи.

Прочее, чего не было на desktop:

- **Ресурсы:** `stringResource(R.string.title)`, `painterResource(R.drawable.ic)`,
  `dimensionResource(R.dimen.padding)`
- **Строки в `strings.xml`**, а не в коде — иначе не локализуется
- **`Modifier.clickable`**, `Modifier.combinedClickable` — тач-обработка
- **`@Preview`** — просмотр компонента в IDE без запуска приложения:
  ```kotlin
  @Preview(showBackground = true)
  @Composable
  private fun PostCardPreview() {
      PostsTheme { PostCard(post = Post(1, "Заголовок", "Текст"), onClick = {}) }
  }
  ```
  Работает только для stateless-компонентов — ещё одна причина соблюдать state
  hoisting.

  Артефакта нужно **два**, и их часто путают: `ui-tooling-preview` как
  `implementation` даёт саму аннотацию (без него не скомпилируется), а
  `ui-tooling` как `debugImplementation` — рендер в IDE. Второй специально только
  в debug: тянуть инструментарий в release не нужно
- **`rememberSaveable`** — то же, что `remember`, но переживает пересоздание
  `Activity`. На desktop его нет смысла (окно не пересоздаётся), на Android он
  закрывает разрыв между `remember` и `ViewModel`:

  ```kotlin
  var isExpanded by rememberSaveable { mutableStateOf(false) }
  ```

  | Где хранить | Переживает поворот | Переживает убийство процесса | Для чего |
  |---|:---:|:---:|---|
  | `remember` | нет | нет | раскрыт ли аккордеон, открыто ли меню |
  | `rememberSaveable` | да | да | позиция таба, черновик короткого поля |
  | `ViewModel` | да | нет | состояние экрана, данные, флаги загрузки |
  | `SavedStateHandle` | да | да | то, без чего экран не восстановить |

  Хранит только то, что укладывается в `Bundle`: примитивы, строки, `Parcelable`.
  Для своих типов нужен `Saver`, и это уже повод перенести состояние в `ViewModel`.
  Практическое правило: форма, которую жалко потерять, живёт во `ViewModel`;
  мелкое UI-состояние — в `rememberSaveable`
- **Тема** генерируется шаблоном в `ui/theme/`: `Color.kt`, `Type.kt`, `Theme.kt`.
  Трогать не обязательно, но полезно понимать, что `MaterialTheme.colorScheme`
  берётся оттуда
- **Insets:** `Scaffold` учитывает их сам; вручную — `Modifier.safeDrawingPadding()`

---

### 7.6. Навигация

```kotlin
@Serializable
sealed interface Route {
    @Serializable data object PostList : Route
    @Serializable data class PostEdit(val postId: Int? = null) : Route
}

@Composable
fun AppNavHost(container: AppContainer) {
    val navController = rememberNavController()

    NavHost(navController = navController, startDestination = Route.PostList) {

        composable<Route.PostList> {
            PostsScreen(
                viewModel = viewModel(factory = PostsViewModel.factory(container.postRepository)),
                onAddClick = { navController.navigate(Route.PostEdit()) },
                onPostClick = { id -> navController.navigate(Route.PostEdit(id)) }
            )
        }

        composable<Route.PostEdit> { backStackEntry ->
            val route: Route.PostEdit = backStackEntry.toRoute()
            EditScreen(
                postId = route.postId,
                onDone = { navController.popBackStack() }
            )
        }
    }
}
```

Показан **type-safe вариант** (Navigation Compose 2.8+): маршруты — это
`@Serializable`-классы, аргументы типизированы. Старый вариант со строками
(`"edit/{postId}"`) ещё встречается в документации и проектах — его нужно уметь
читать, но писать лучше новый.

Что нужно знать:

- `navController.navigate(route)` — переход вперёд
- `popBackStack()` — назад; системная кнопка «Назад» работает автоматически
- `navigate(route) { popUpTo(...) { inclusive = true } }` — управление стеком
  (например, после логина не возвращаться на экран логина)
- **`ViewModel` привязана к назначению (destination)**, не к `Activity`:
  у каждого экрана своя, и она уничтожается при уходе с экрана
- Передавать через маршрут только **идентификаторы**, не объекты. Данные экран
  загружает сам по id

Разница с `when (screen)` из [этапа 5](05-desktop-project.md): появляется back
stack, системная кнопка «назад», привязка ViewModel к экрану, deep links.

---

### 7.7. Локальное хранение (по необходимости)

#### Room — если нужен офлайн-кэш

```kotlin
@Entity(tableName = "posts")
data class PostEntity(
    @PrimaryKey val id: Int,
    val title: String,
    val body: String
)

@Dao
interface PostDao {
    @Query("SELECT * FROM posts ORDER BY id DESC")
    fun observeAll(): Flow<List<PostEntity>>          // ← эмитит при изменении таблицы

    @Query("SELECT * FROM posts WHERE id = :id")
    suspend fun getById(id: Int): PostEntity?

    @Upsert
    suspend fun upsertAll(posts: List<PostEntity>)

    @Query("DELETE FROM posts WHERE id = :id")
    suspend fun deleteById(id: Int)
}

@Database(entities = [PostEntity::class], version = 1)
abstract class AppDatabase : RoomDatabase() {
    abstract fun postDao(): PostDao
}
```

- `Flow<List<T>>` из `@Query` — Room сам эмитит новое значение при любом изменении
  таблицы. Это то место, где Flow с [этапа 2b](02b-flow.md) окупается
  полностью: `ContentObserver` и ручная инвалидация не нужны
- SQL проверяется **на этапе компиляции** — опечатка не доживёт до рантайма
- `suspend`-методы сами уходят на фоновый диспетчер
- Миграции: `version` + `Migration` или `fallbackToDestructiveMigration()` на
  время разработки
- Требует KSP-плагин (`com.google.devtools.ksp`) и `ksp("androidx.room:room-compiler:...")`

Паттерн «single source of truth»:

```kotlin
class PostRepositoryImpl(
    private val api: PostApi,
    private val dao: PostDao
) : PostRepository {

    // UI подписывается на БД, а не на сеть
    override fun observePosts(): Flow<List<Post>> =
        dao.observeAll().map { list -> list.map { it.toDomain() } }

    // сеть только обновляет БД
    override suspend fun refresh(): ApiResult<Unit> = safeApiCall {
        dao.upsertAll(api.getPosts().map { it.toEntity() })
    }
}
```

Приложение работает без сети, обновление данных приходит в UI автоматически.
Ради этой конструкции и нужен слой `domain` с интерфейсом репозитория.

#### DataStore — вместо SharedPreferences

```kotlin
val Context.settings: DataStore<Preferences> by preferencesDataStore(name = "settings")

private val KEY_TOKEN = stringPreferencesKey("token")

suspend fun saveToken(token: String) {
    context.settings.edit { it[KEY_TOKEN] = token }
}

val token: Flow<String?> = context.settings.data.map { it[KEY_TOKEN] }
```

`suspend`-API, никаких `commit()` на главном потоке, чтение как `Flow`.

#### Coil — картинки

```kotlin
AsyncImage(
    model = post.imageUrl,
    contentDescription = null,
    modifier = Modifier.size(64.dp).clip(CircleShape),
    contentScale = ContentScale.Crop
)
```

Одна зависимость (`io.coil-kt:coil-compose`), одна строка кода. Заменяет Picasso
и ручную работу с `Bitmap`.

---

### 7.8. DI: когда пора

Порядок такой:

1. **Сначала руками** — `AppContainer` из [этапа 5](05-desktop-project.md),
   доступный через `Application`:
   ```kotlin
   @Composable
   fun rememberContainer(): AppContainer {
       val context = LocalContext.current
       return (context.applicationContext as PostsApp).container
   }
   ```
2. **Hilt — когда фабрик ViewModel станет больше пяти** и от передачи зависимостей
   через параметры начнёт болеть. Тогда `@HiltViewModel` + `@Inject constructor`
   убирают фабрики целиком

Начинать с Hilt на этом маршруте не стоит: пока не почувствуешь проблему, он
выглядит как немотивированная сложность с непонятными аннотациями.

---

## Не учить на этом этапе

- WorkManager (пока не нужна фоновая работа)
- Paging 3
- Services, BroadcastReceiver, ContentProvider
- CameraX, Media3
- Кастомные View, `Canvas`
- Notifications
- Compose Multiplatform
- Baseline Profiles, оптимизация запуска
- R8/ProGuard-правила (кроме понимания, что kotlinx.serialization их не требует)
- Firebase, аналитика, crashlytics
- Тестирование (unit, Compose UI, instrumentation) — до реального проекта

---

## Если проект команды на XML

Один вечер, чтобы **читать и править** существующий код. Новое всё равно писать
на Compose.

- **ViewBinding** вместо `findViewById`:
  ```kotlin
  private var _binding: FragmentPostsBinding? = null
  private val binding get() = _binding!!

  override fun onCreateView(...) : View {
      _binding = FragmentPostsBinding.inflate(inflater, container, false)
      return binding.root
  }

  override fun onDestroyView() {
      _binding = null            // ← обязательно, иначе утечка
      super.onDestroyView()
  }
  ```
- **`Fragment`** и его жизненный цикл; `viewLifecycleOwner` — не то же самое, что
  сам фрагмент
- **`RecyclerView` + `ListAdapter` + `DiffUtil`** — современный способ, без
  `notifyDataSetChanged()`
- **`repeatOnLifecycle`** вместо `collectAsStateWithLifecycle`:
  ```kotlin
  viewLifecycleOwner.lifecycleScope.launch {
      viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
          viewModel.state.collect { render(it) }
      }
  }
  ```
  Без `repeatOnLifecycle` подписка продолжит работать в фоне — классическая ошибка
- **`ComposeView`** в XML-разметке — способ вставлять новые экраны в старый проект

---

## Упражнения

1. **Пустой проект.** Создать, запустить на эмуляторе, изменить текст, пересобрать.
   Заодно попробовать **Live Edit** (включается в настройках Android Studio) и
   посмотреть на его границы: правки внутри `@Composable` подхватываются на лету,
   изменение сигнатур или добавление классов требует полной пересборки. Это не
   замена перезапуску, а ускорение мелких итераций.
2. **`@Preview`.** Написать stateless-компонент карточки и preview к нему.
   Затем добавить внутрь `remember` со состоянием — посмотреть, как preview
   ломается.
3. **ViewModel и поворот.** ViewModel со счётчиком, увеличиваемым по кнопке.
   Повернуть экран — значение сохранилось. Затем перенести счётчик в
   `remember { mutableStateOf() }` внутри `@Composable` — повернуть снова.
4. **Перенос модели.** Взять `PostsModel` с [этапа 5](05-desktop-project.md),
   унаследовать от `ViewModel`, заменить свой scope на `viewModelScope`.
   Убедиться, что больше ничего менять не пришлось.
5. **Сеть на Android.** Подключить репозиторий с [этапа 3](03-network-json.md),
   заменив движок на `OkHttp`. Проверить, что без `INTERNET` в манифесте всё
   падает — и найти это в Logcat.
6. **Lifecycle-подписка.** Собрать `StateFlow` через `collectAsState()`, свернуть
   приложение, посмотреть в Logcat, что подписка активна. Заменить на
   `collectAsStateWithLifecycle()` — убедиться, что она останавливается.
7. **Навигация.** Два экрана, переход с аргументом, возврат через `popBackStack()`
   и через системную кнопку. Убедиться, что ViewModel второго экрана уничтожается
   при уходе.
8. **Room.** Одна таблица, `Flow<List<T>>` из `@Query`. Вставить запись из другого
   места приложения — убедиться, что UI обновился сам.
9. **Эмулятор и localhost.** Если делал сервер на [этапе 5](05-desktop-project.md) —
   подключиться к нему с эмулятора по адресу `10.0.2.2:8080`. Понять, почему не
   `localhost`.

---

## Чекпоинт

- [ ] Проект собирается и запускается на эмуляторе и на устройстве
- [ ] Модель с этапа 5 перенесена почти без изменений
- [ ] Понятно, что `viewModelScope` — это тот же scope, что писался руками
- [ ] Проверено на практике: ViewModel переживает поворот, `remember` — нет
- [ ] Используется `collectAsStateWithLifecycle()`, и понятно, зачем
- [ ] Навигация между двумя экранами с аргументом работает, «назад» работает
- [ ] Есть хотя бы один `@Preview`
- [ ] Строки в `strings.xml`, а не в коде
- [ ] Понятно, когда понадобится Hilt и почему пока не нужен

---

## Ссылки

- [Android developers: Compose pathway](https://developer.android.com/courses/pathways/compose)
- [Guide to app architecture](https://developer.android.com/topic/architecture)
- [ViewModel overview](https://developer.android.com/topic/libraries/architecture/viewmodel)
- [Navigation Compose](https://developer.android.com/guide/navigation)
- [Type-safe navigation](https://developer.android.com/guide/navigation/design/type-safety)
- [Room](https://developer.android.com/training/data-storage/room)
- [DataStore](https://developer.android.com/topic/libraries/architecture/datastore)
- [Coil](https://coil-kt.github.io/coil/compose/)
- [Lifecycle-aware collection](https://developer.android.com/topic/libraries/architecture/coroutines#lifecycle-aware)
- [Now in Android](https://github.com/android/nowinandroid) — эталонный проект
  Google на этом же стеке; полезно смотреть как справочник, но он значительно
  сложнее, чем нужно для старта
- [Глоссарий](glossary.md) — служебные слова Kotlin и термины, если что-то по ходу осталось непонятным

---

**Следующий этап:** [08. Android-проект](08-android-project.md)
