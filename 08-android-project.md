# Этап 8. 🎯 Android-проект

**Время:** 5–7 вечеров
**Предыдущий:** [07. Android-минимум](07-android-basics.md) · **Следующий:** — · [Обзор](README.md)

---

## Цель

То же приложение, что на [этапе 5](05-desktop-project.md), но на Android — со
всеми платформенными вещами, которых на desktop не было: поворот экрана,
отсутствие сети, back stack, release-сборка.

Финальный критерий: **APK, который можно поставить на телефон и отдать другому
человеку**, и он не сломается.

---

## Постановка задачи

**Обязательно:**

- два экрана: список и деталь/редактирование, через Navigation Compose
- `ViewModel` со `StateFlow<UiState>`
- тот же `Repository` с [этапа 3](03-network-json.md)
- обработка отсутствия сети с внятным сообщением и повтором
- сохранение состояния при повороте экрана
- pull-to-refresh
- поиск с `debounce`
- подписанная release-сборка

**Опционально, но сильно повышает реалистичность:**

- Room как кэш (работа офлайн)
- индикатор «нет сети»
- пустое состояние (список пуст — это не ошибка и не загрузка)

---

## Порядок переноса

Не начинать с нуля. Порядок такой, чтобы на каждом шаге приложение работало.

### Вечер 1. Скелет и слой данных

1. Новый проект (Empty Activity)
2. **Скопировать `domain/` целиком** — изменений не требуется
3. **Скопировать `data/`**, поменять движок Ktor: `CIO` → `OkHttp`
4. `INTERNET` в манифест
5. `Application` + `AppContainer`
6. Проверить репозиторий, ещё без UI: вызвать из `MainActivity.onCreate` в
   `lifecycleScope.launch` и посмотреть результат в Logcat
7. **Включить режим полёта и убедиться, что приходит именно `NetworkError`.**
   Смена движка меняет типы транспортных исключений (см.
   [этап 3](03-network-json.md)): под CIO это `UnresolvedAddressException`, под
   OkHttp — `UnknownHostException`. Если в `safeApiCall` есть ветвь
   `catch (e: IOException)`, всё сработает само; если её нет — «нет сети»
   провалится в общую ветвь и пользователь увидит невнятное сообщение.
   Проверять это надо сейчас, а не на вечере 4, иначе причина будет неочевидна

Ключевая мысль этого вечера: слой данных переносится **дословно**. Если он не
перенёсся — значит, на [этапе 5](05-desktop-project.md) в него протекло что-то
лишнее, и это стоит починить.

### Вечер 2. Экран списка

1. Скопировать `PostsUiState` — без изменений. Оба флага (`isLoading` и
   `isRefreshing`) уже на месте: первый для первой загрузки, второй понадобится
   для pull-to-refresh на вечере 4
2. `PostsModel` → `PostsViewModel : ViewModel()`, свой scope → `viewModelScope`,
   убрать `close()`
3. Фабрика ViewModel
4. Скопировать `PostsScreen`, заменить `collectAsState()` →
   `collectAsStateWithLifecycle()`
5. Три состояния, retry
6. **Проверить поворот экрана** — данные не должны перезагружаться

### Вечер 3. Навигация и второй экран

1. `NavHost`, маршруты как `@Serializable`-классы
2. Переход в редактирование с `postId`
3. `EditViewModel`, загрузка по id, сохранение
4. `popBackStack()` после сохранения
5. Проверить системную кнопку «назад» и жест назад

### Вечер 4. Платформенные особенности

1. Pull-to-refresh
2. `Snackbar` через `SharedFlow` + `SnackbarHostState`
3. Удаление с `AlertDialog`
4. Обработка отсутствия сети: включить режим полёта, проверить сообщение
5. Insets: `Scaffold`, `enableEdgeToEdge()`, проверить на устройстве с вырезом
6. Тёмная тема — проверить, что читается (Material 3 делает это сам, если не
   хардкодить цвета)

### Вечер 5. Room (опционально)

1. `PostEntity`, `PostDao`, `AppDatabase`
2. Переделать `PostRepositoryImpl` на single source of truth:
   `observePosts(): Flow<List<Post>>` из БД, `refresh()` в сеть
3. `PostsViewModel` подписывается на `observePosts()` через `stateIn`
4. **Проверить офлайн:** загрузить данные, включить режим полёта, перезапустить
   приложение — данные на месте

### Вечер 6. Release-сборка

1. `keystore` через Android Studio (Build → Generate Signed App Bundle / APK)
2. `signingConfigs` в `build.gradle.kts`, ключи — в `local.properties` (не в git!)
3. `isMinifyEnabled = true`, `isShrinkResources = true`
4. **Собрать release и проверить на устройстве** — не debug
5. Убедиться, что сеть и JSON работают после обфускации

### Вечер 7. Разбор

1. Пройти чек-лист ниже
2. Убрать дублирование, вынести общие компоненты
3. Прогнать по типовым проблемам из раздела ниже

---

## Ключевые куски, которых не было на desktop

### stateIn — холодный Flow из Room в горячее состояние

> **Этот подраздел относится к варианту с Room (вечер 5).** Если Room не
> подключался, `PostsViewModel` остаётся таким, каким получился на вечере 2 —
> копией модели с [этапа 5](05-desktop-project.md) с заменой `scope` на
> `viewModelScope`. Ничего доделывать не нужно, и остальные подразделы
> (pull-to-refresh, `Snackbar`, пустое состояние) работают одинаково в обоих
> вариантах. Прочитать всё равно стоит: `stateIn` — самый частый способ строить
> `ViewModel` в реальных Android-проектах, потому что реальные проекты почти всегда
> имеют локальный кэш.

Разница между двумя вариантами — в том, **кто владеет списком**:

| | Без Room (вечер 2) | С Room (вечер 5) |
|---|---|---|
| Источник списка | поле в `PostsUiState` | `Flow` из БД |
| Кто пишет список | `load()` присваивает | БД, после `refresh()` |
| Состояние собирается | `_state.update { copy(...) }` | `combine` + `stateIn` |
| Работает офлайн | нет | да |

```kotlin
class PostsViewModel(
    private val repository: PostRepository
) : ViewModel() {

    private val query = MutableStateFlow("")
    private val isRefreshing = MutableStateFlow(false)
    private val errorMessage = MutableStateFlow<String?>(null)

    val state: StateFlow<PostsUiState> = combine(
        repository.observePosts(),      // холодный Flow из Room
        query,
        isRefreshing,
        errorMessage
    ) { posts, q, refreshing, error ->
        PostsUiState(
            posts = posts.filter { it.title.contains(q, ignoreCase = true) },
            query = q,
            isRefreshing = refreshing,
            error = error
        )
    }.stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5_000),
        initialValue = PostsUiState(isLoading = true)
    )

    init { refresh() }

    fun onQueryChange(value: String) { query.value = value }

    fun refresh() {
        viewModelScope.launch {
            isRefreshing.value = true
            errorMessage.value = null
            // при успехе toUserMessage() даёт null, и данные придут сами через Flow из Room
            errorMessage.value = repository.refresh().toUserMessage()
            isRefreshing.value = false
        }
    }
}
```

Обрати внимание, что `debounce` из [этапа 5](05-desktop-project.md) здесь исчез.
Он был нужен, чтобы не дёргать сервер на каждую букву; здесь фильтрация идёт по
уже загруженному списку в памяти, и задержка только раздражала бы. Если поиск
всё-таки должен уходить на сервер — `debounce` возвращается, но на отдельный поток
ввода, как было на этапе 5, а не внутрь `combine`.

Разбор:

- `combine` собирает несколько источников в одно состояние экрана
- `stateIn` превращает холодный Flow в `StateFlow`: **один запуск на всех
  подписчиков** вместо перезапуска на каждого
- `WhileSubscribed(5_000)` — держать подписку 5 секунд после ухода последнего
  подписчика. Это нужно именно для Android: при повороте экрана подписчик исчезает
  и появляется заново, и без таймаута (`WhileSubscribed()`) запрос к БД перезапустится
  зря. Альтернатива `SharingStarted.Lazily` тоже убирает перезапуск, но никогда не
  останавливается — подписка работает и когда экран в фоне
- Обрати внимание: `refresh()` не присваивает данные напрямую. Он обновляет БД,
  а данные приходят в UI сами через `Flow` из Room. Это и есть single source of truth

### Pull-to-refresh

> API pull-to-refresh в Material 3 **экспериментален и уже трижды менял имя**:
> `Modifier.pullRefresh` → `PullToRefreshContainer` → `PullToRefreshBox`. Ниже —
> актуальный на момент написания вариант. Если он не находится, проверь версию
> `material3` и текущую сигнатуру в
> [документации компонента](https://developer.android.com/develop/ui/compose/components/pull-to-refresh) —
> смысл параметров (`isRefreshing` + `onRefresh`) сохраняется во всех вариантах,
> меняется обёртка.

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun PostsScreen(viewModel: PostsViewModel) {
    val state by viewModel.state.collectAsStateWithLifecycle()

    PullToRefreshBox(
        isRefreshing = state.isRefreshing,
        onRefresh = viewModel::refresh
    ) {
        LazyColumn(Modifier.fillMaxSize()) {
            items(state.posts, key = { it.id }) { post ->
                PostCard(post = post, onClick = { /* ... */ })
            }
        }
    }
}
```

### Snackbar из SharedFlow

```kotlin
@Composable
fun PostsScreen(viewModel: PostsViewModel) {
    val snackbarHostState = remember { SnackbarHostState() }

    LaunchedEffect(Unit) {
        viewModel.events.collect { message ->
            snackbarHostState.showSnackbar(message)
        }
    }

    Scaffold(snackbarHost = { SnackbarHost(snackbarHostState) }) { padding ->
        // ...
    }
}
```

Почему `SharedFlow`, а не `StateFlow`: `StateFlow` отдаёт последнее значение
каждому новому подписчику. При повороте экрана подписчик новый — snackbar
покажется повторно. Для событий нужен `SharedFlow` без replay.

### Пустое состояние

Три состояния — мало. Реальных четыре:

```kotlin
when {
    state.isLoading -> LoadingView()
    state.error != null -> ErrorView(state.error, onRetry = viewModel::refresh)
    state.posts.isEmpty() -> EmptyView(
        message = if (state.query.isBlank()) "Пока ничего нет"
                  else "По запросу «${state.query}» ничего не найдено"
    )
    else -> PostList(state.posts)
}
```

Пустой список — не ошибка. И «пусто, потому что нет данных» отличается от
«пусто, потому что фильтр ничего не нашёл». Это то, что забывают в 90% учебных
проектов.

### Эмулятор и локальный сервер

Если на [этапе 5](05-desktop-project.md) был свой Ktor-сервер:

| Откуда | Адрес хост-машины |
|---|---|
| Эмулятор Android | `10.0.2.2` |
| Реальное устройство в той же Wi-Fi | локальный IP машины (`192.168.x.x`) |
| Genymotion | `10.0.3.2` |

`localhost` на эмуляторе — это сам эмулятор, а не компьютер. Плюс для HTTP (не
HTTPS) на API 28+ нужно явно разрешить cleartext:

```xml
<application android:usesCleartextTraffic="true" ...>
```

В release-сборке это, разумеется, убрать.

---

## Release-сборка

```kotlin
// app/build.gradle.kts
import java.util.Properties      // ← в самом начале файла, до блока plugins

android {
    signingConfigs {
        create("release") {
            val props = Properties().apply {
                val f = rootProject.file("local.properties")
                if (f.exists()) load(f.inputStream())
            }
            storeFile = props.getProperty("storeFile")?.let { file(it) }
            storePassword = props.getProperty("storePassword")
            keyAlias = props.getProperty("keyAlias")
            keyPassword = props.getProperty("keyPassword")
        }
    }

    buildTypes {
        release {
            isMinifyEnabled = true
            isShrinkResources = true
            signingConfig = signingConfigs.getByName("release")
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
}
```

`local.properties` и `*.jks` — **в `.gitignore`**. Пароли в репозитории — самая
частая ошибка первого релиза.

Что проверить в release-сборке отдельно (в debug это не проявляется):

- сеть работает — R8 не выбросил нужные классы
- JSON парсится — kotlinx.serialization не требует правил, но если где-то
  используется рефлексия, оно сломается именно здесь
- нет `usesCleartextTraffic`
- логи не сыпят чувствительными данными (`LogLevel.BODY` в Ktor убрать)

---

## Не делать на этом этапе

Финальный этап особенно провоцирует «доделать по-нормальному». Что отложить до
следующего проекта:

- **Hilt.** Фабрик ViewModel здесь две-три; `AppContainer` справляется
- **Многомодульность.** Пакеты по слоям в одном модуле — правильный масштаб
- **Тесты.** Отдельная тема — [приложение](appendix-testing.md)
- **CI, Firebase, аналитику, Crashlytics**
- **Анимации и кастомную темизацию.** Проверить читаемость тёмной темы — да,
  рисовать свою палитру — нет
- **WorkManager, Paging, deep links, виджеты, уведомления**
- **Публикацию в Google Play.** Собрать подписанный APK и поставить на телефон —
  входит в задачу; проходить review, заполнять карточку и настраивать
  App Bundle — нет
- **Оптимизацию.** Baseline Profiles, R8-тонкости, Layout Inspector для
  рекомпозиций — только если приложение реально тормозит

---

## Итоговый чек-лист

**Функционал:**

- [ ] Список грузится, показывается индикатор
- [ ] Pull-to-refresh работает
- [ ] Поиск с `debounce`
- [ ] Создание, редактирование, удаление
- [ ] Удаление с подтверждением
- [ ] Snackbar на успех, не повторяется при повороте
- [ ] Пустое состояние отличается от загрузки и от ошибки
- [ ] «Нет данных» и «ничего не найдено» — разные сообщения

**Платформа:**

- [ ] Поворот экрана: данные не перезагружаются, ввод в форме не теряется
- [ ] Режим полёта: внятное сообщение, кнопка повтора работает
- [ ] Системная кнопка «назад» и жест назад работают корректно
- [ ] Тёмная тема читается
- [ ] Контент не уезжает под статус-бар и навигационную панель
- [ ] Нет ANR при быстрых нажатиях
- [ ] (опц.) Приложение открывается офлайн и показывает кэш

**Архитектура:**

- [ ] `ViewModel` не знает ни про Compose, ни про `Context`, ни про Ktor
- [ ] `@Composable`-функции не знают про `ApiResult` и DTO
- [ ] Слои `domain` и `data` перенеслись с этапа 5 без правок логики
- [ ] Ошибки маппятся в текст в одном месте (`toUserMessage()` с этапа 5),
      исчерпывающий `when` по `ApiResult` не продублирован по моделям
- [ ] Проверено, что смена движка Ktor не сломала определение «нет сети»
- [ ] Строки в `strings.xml`
- [ ] Есть `@Preview` хотя бы для основных компонентов

**Сборка:**

- [ ] Release-APK подписан и установлен на реальное устройство
- [ ] Keystore и пароли не в git
- [ ] `isMinifyEnabled = true`, и после обфускации всё работает
- [ ] `usesCleartextTraffic` убран
- [ ] Логирование тела запросов отключено в release

---

## Типовые проблемы этапа

**Данные перезагружаются при повороте.** Загрузка вызывается в `LaunchedEffect`
внутри `@Composable` вместо `init` во `ViewModel`, либо ViewModel создаётся не
через `viewModel()`, а напрямую конструктором.

**Snackbar показывается повторно после поворота.** События через `StateFlow`.
Нужен `SharedFlow` без replay.

**Ввод в форме теряется при повороте.** Состояние формы в
`remember { mutableStateOf() }` — оно не переживает пересоздание `Activity`.
Форма, которую жалко потерять, должна жить во `ViewModel`; для мелкого UI-состояния
хватит `rememberSaveable` (таблица «где что хранить» — в
[этапе 7](07-android-basics.md), раздел 7.5).

**Запрос к БД перезапускается при каждом повороте.** Либо `stateIn` нет вовсе
(тогда холодный Flow перезапускается на каждого нового подписчика), либо он есть,
но с `started = SharingStarted.WhileSubscribed()` **без таймаута**: при повороте
подписчик исчезает, подписка сразу останавливается, новый подписчик запускает всё
заново. Лечится таймаутом: `WhileSubscribed(5_000)`.

`SharingStarted.Lazily` этой проблемы не даёт — он стартует один раз и не
останавливается никогда. Но у него обратная беда: подписка живёт, пока жива
`ViewModel`, то есть работа продолжается, даже когда экран в фоне и никто не
слушает. Для данных из БД это терпимо, для сетевого поллинга — нет.

**Приложение падает при обращении к сети.** Нет `INTERNET` в манифесте, либо
HTTP без `usesCleartextTraffic`, либо `10.0.2.2` вместо `localhost` не прописан.

**Release-сборка падает, debug работает.** R8 выбросил классы. Смотреть
`mapping.txt` и стектрейс; для kotlinx.serialization правила обычно не нужны, но
для рефлексивных библиотек — нужны.

**ANR при старте.** Что-то тяжёлое в `Application.onCreate` или в
`Activity.onCreate` синхронно. `AppContainer` должен быть ленивым (`by lazy`).

**Утечка памяти.** `ViewModel` держит `Context` или `View`. Проверяется LeakCanary
(`debugImplementation("com.squareup.leakcanary:leakcanary-android:...")`) — стоит
подключить на один вечер, чтобы посмотреть, что он найдёт.

---

## Ссылки

- [Guide to app architecture](https://developer.android.com/topic/architecture)
- [StateFlow: stateIn and SharingStarted](https://developer.android.com/kotlin/flow/stateflow-and-sharedflow)
- [Pull to refresh](https://developer.android.com/develop/ui/compose/components/pull-to-refresh)
- [Save UI state](https://developer.android.com/topic/libraries/architecture/saving-states)
- [Room: single source of truth](https://developer.android.com/training/data-storage/room/async-queries)
- [Sign your app](https://developer.android.com/studio/publish/app-signing)
- [Shrink, obfuscate and optimize](https://developer.android.com/build/shrink-code)
- [Connect to the network from the emulator](https://developer.android.com/studio/run/emulator-networking)
- [Network security configuration](https://developer.android.com/privacy-and-security/security-config)
- [Глоссарий](glossary.md) — служебные слова Kotlin и термины, если что-то по ходу осталось непонятным

---

## Что делать после

Маршрут закончен: есть работающее приложение и понимание всего стека. Дальше —
по потребности, а не «всё подряд».

**Ближайшие полезные шаги, в порядке отдачи:**

1. **[Retrofit](appendix-retrofit.md)** — один вечер. Он стоит в большинстве
   существующих проектов, и его нужно уметь читать. Концепции те же
2. **[Тестирование](appendix-testing.md)** — `runTest`, Turbine, MockEngine.
   Первое, что стоит добавить, если код будет жить дольше недели
3. **Hilt** — когда фабрик ViewModel станет больше пяти
4. **WorkManager** — когда понадобится синхронизация в фоне
5. **Paging 3** — когда список перестанет влезать в один запрос
6. **Многомодульность** — когда проект перестанет собираться быстро

**Что читать как справочник:**

- [Now in Android](https://github.com/android/nowinandroid) — эталонный проект
  Google: тот же стек, но с многомодульностью, тестами и CI. Полезно смотреть
  выборочно, целиком копировать структуру на маленьком проекте не нужно
- [Android Developers Blog](https://android-developers.googleblog.com/) — что
  меняется в платформе
- [Kotlin roadmap](https://kotlinlang.org/docs/roadmap.html) — что идёт в языке

**Чего избегать:** браться за Kotlin Multiplatform, MVI-фреймворки или
многомодульность до второго-третьего реального проекта. Каждая из этих вещей
решает проблему, которой на маленьком проекте ещё нет, и без ощущения проблемы
выглядит как случайная сложность.

---

**Обзор маршрута:** [README.md](README.md)
