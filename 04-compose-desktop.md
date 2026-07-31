# Этап 4. UI: Compose Desktop

**Время:** 3–4 вечера
**Предыдущий:** [03. Сеть и JSON](03-network-json.md) · **Следующий:** [05. Desktop-проект](05-desktop-project.md) · [Обзор](README.md)

---

## Цель

Собирать экраны из стандартных компонентов Material 3, управлять состоянием и
понимать, почему UI перерисовывается. Тот же код почти без изменений заработает
на Android на [этапе 7](07-android-basics.md).

---

## Главный сдвиг в мышлении

В классическом Android/Swing UI — это **объекты, которые ты мутируешь**:

```java
textView.setText("Загрузка");
progressBar.setVisibility(View.VISIBLE);
button.setEnabled(false);
```

Проблема известная: состояние размазано по виджетам, и его легко рассинхронизировать.
Забыл спрятать `progressBar` в одной из четырёх ветвей — увидел баг.

В Compose UI — это **функция от состояния**:

```kotlin
@Composable
fun Screen(state: UiState) {
    when (state) {
        is UiState.Loading -> CircularProgressIndicator()
        is UiState.Success -> ItemList(state.items)
        is UiState.Error   -> ErrorMessage(state.message)
    }
}
```

Ты не описываешь **переходы**, ты описываешь **как выглядит каждое состояние**.
Меняется состояние — Compose сам пересобирает нужную часть дерева (рекомпозиция).
Рассинхронизация невозможна по построению.

**Если приходилось работать с Flutter — здесь фора:** это тот же принцип, что
`build()`. `@Composable`-функция ≈ `Widget.build()`, `remember` ≈ поле `State`,
`LazyColumn` ≈ `ListView.builder`. Если нет — ничего страшного, аналогия не
обязательна, дальше всё объясняется с нуля.

---

## Настройка

```kotlin
plugins {
    kotlin("jvm")
    id("org.jetbrains.compose") version "<актуальная>"
    id("org.jetbrains.kotlin.plugin.compose") version "<версия Kotlin>"
}

dependencies {
    implementation(compose.desktop.currentOs)
    implementation(compose.material3)
}

compose.desktop {
    application {
        mainClass = "MainKt"
    }
}
```

Минимальное приложение:

```kotlin
fun main() = application {
    Window(onCloseRequest = ::exitApplication, title = "Демо") {
        MaterialTheme {
            App()
        }
    }
}

@Composable
fun App() {
    Text("Привет")
}
```

> Версии Compose Desktop и Kotlin связаны: плагин `plugin.compose` должен
> соответствовать версии Kotlin. Актуальную матрицу смотри в
> [Compose Multiplatform releases](https://github.com/JetBrains/compose-multiplatform/releases).

Запуск: `./gradlew run`.

Про **hot reload** стоит сразу настроить ожидания: в Compose Desktop он есть, но
это отдельный механизм, который нужно включать, и работает он не на любое
изменение — правки внутри тела `@Composable` подхватываются, изменения сигнатур,
классов и состояния обычно требуют перезапуска. Актуальный способ включения смотри
в [документации Compose Multiplatform](https://github.com/JetBrains/compose-multiplatform/blob/master/tutorials/Hot_reload/README.md);
он менялся между версиями. На время изучения проще исходить из того, что после
правки нужен перезапуск — благо на desktop это 2–3 секунды, а не сборка APK.
Это, собственно, и есть главная причина начинать с desktop.

---

## Темы

### 4.1. `@Composable` функции

```kotlin
@Composable
fun Greeting(name: String, onClick: () -> Unit) {
    Column {
        Text("Привет, $name")
        Button(onClick = onClick) { Text("Нажми") }
    }
}
```

- Обычная функция с аннотацией. Возвращает `Unit`, **не** объект View
- Вызывать можно только из другой `@Composable` (как `suspend` — только из `suspend`)
- Имя с большой буквы — соглашение
- **Не должна иметь побочных эффектов.** Может быть вызвана многократно,
  в любом порядке, на любом потоке, пропущена или вызвана заново.
  Не писать в файл, не делать запросы, не менять внешние переменные напрямую
- Параметры — вход, лямбды-колбэки — выход. Это весь контракт

### 4.2. Стандартные компоненты Material 3

Ровно то, что нужно для клиент-серверного приложения:

| Компонент | Назначение |
|---|---|
| `Text` | текст |
| `Button`, `OutlinedButton`, `TextButton`, `IconButton` | кнопки |
| `TextField`, `OutlinedTextField` | ввод |
| `Checkbox`, `Switch`, `RadioButton` | переключатели |
| `Slider` | ползунок |
| `DropdownMenu`, `ExposedDropdownMenuBox` | выпадающий список |
| `Card`, `Surface` | контейнеры |
| `Divider` / `HorizontalDivider` | разделитель |
| `CircularProgressIndicator`, `LinearProgressIndicator` | загрузка |
| `AlertDialog` | диалог |
| `Scaffold`, `TopAppBar`, `SnackbarHost` | каркас экрана |
| `Icon` + `Icons.Default.*` | иконки |

Этого набора достаточно. Кастомных компонентов на этом маршруте не нужно.

### 4.3. Layout и `Modifier`

```kotlin
Column(
    modifier = Modifier
        .fillMaxSize()
        .padding(16.dp)
        .verticalScroll(rememberScrollState()),
    verticalArrangement = Arrangement.spacedBy(8.dp),
    horizontalAlignment = Alignment.CenterHorizontally
) {
    Text("Заголовок", style = MaterialTheme.typography.headlineMedium)

    Row(
        modifier = Modifier.fillMaxWidth(),
        horizontalArrangement = Arrangement.SpaceBetween,
        verticalAlignment = Alignment.CenterVertically
    ) {
        Text("Слева", modifier = Modifier.weight(1f))
        Button(onClick = {}) { Text("Справа") }
    }
}
```

- `Column` — вертикально, `Row` — горизонтально, `Box` — наложением
- `Modifier` — цепочка, **порядок важен**: `.padding().background()` и
  `.background().padding()` дают разный результат
- Нужный минимум модификаторов: `fillMaxSize`, `fillMaxWidth`, `size`, `width`,
  `height`, `padding`, `weight`, `background`, `clickable`, `border`, `align`
- Единицы: `dp` для размеров, `sp` для шрифтов. Импорт `androidx.compose.ui.unit.dp`
- `Arrangement.spacedBy(8.dp)` — вместо `Spacer` между каждым элементом
- `Spacer(Modifier.height(16.dp))` — когда нужен один отступ

Аналогии с XML: `Column` ≈ вертикальный `LinearLayout`, `Row` ≈ горизонтальный,
`Box` ≈ `FrameLayout`, `Modifier.weight` ≈ `layout_weight`. `ConstraintLayout`
в Compose есть, но на стандартных экранах почти не нужен.

### 4.4. Списки: `LazyColumn`

```kotlin
LazyColumn(
    modifier = Modifier.fillMaxSize(),
    contentPadding = PaddingValues(16.dp),
    verticalArrangement = Arrangement.spacedBy(8.dp)
) {
    items(
        items = posts,
        key = { it.id }              // ← важно: стабильный ключ
    ) { post ->
        PostCard(post = post, onClick = { onPostClick(post.id) })
    }
}
```

Это замена `RecyclerView`. **Полностью исчезают:** `Adapter`, `ViewHolder`,
`onCreateViewHolder`, `onBindViewHolder`, `DiffUtil`, `notifyItemChanged`,
`getItemViewType`. Вместо всего этого — лямбда и `key`.

- `items(list)` / `itemsIndexed(list)` / `item { }` для одиночного элемента
- `key` — нужен, чтобы Compose корректно переиспользовал элементы при изменении
  списка. Без него будут визуальные артефакты при удалении/вставке
- `LazyRow` — горизонтальный, `LazyVerticalGrid` — сетка
- `rememberLazyListState()` + `state.firstVisibleItemIndex` — если нужно узнать
  позицию прокрутки

Различие, которое стоит запомнить: `Column` рендерит **все** элементы,
`LazyColumn` — только видимые. Для списка из 20 элементов разницы нет; для
данных с сервера всегда `LazyColumn`.

### 4.5. Состояние — ключевая тема этапа

```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }     // ← by, не =

    Button(onClick = { count++ }) {
        Text("Нажато: $count")
    }
}
```

Три части, каждая обязательна:

- `mutableStateOf(0)` — **наблюдаемое** значение. Compose отслеживает, кто его
  читает, и перерисовывает только этих читателей
- `remember { }` — сохранить между рекомпозициями. Без него значение сбрасывалось
  бы при каждой перерисовке
- `by` — делегат, чтобы писать `count` вместо `count.value`

Что бывает при пропуске:

| Ошибка | Результат |
|---|---|
| `var count = 0` | UI не обновится: значение не наблюдаемое |
| `var count = mutableStateOf(0)` без `remember` | сбрасывается при каждой рекомпозиции |
| `var count by remember { ... }` без импортов | не компилируется (см. ниже) |
| мутация `MutableList` вместо создания нового списка | UI не заметит изменения |

**Про `by` и импорты — ступор №1 у начинающих.** Делегат `by` для `MutableState`
требует двух импортов, которые IDE не всегда подставляет сама:

```kotlin
import androidx.compose.runtime.getValue
import androidx.compose.runtime.setValue
```

Без них строка `var count by remember { mutableStateOf(0) }` не компилируется, а
сообщение об ошибке говорит про отсутствующий оператор `getValue`, что мало
помогает. Альтернатива без импортов — работать через `.value`:

```kotlin
val count = remember { mutableStateOf(0) }
Text("${count.value}")
count.value++
```

Оба варианта равноценны; `by` просто короче и потому встречается чаще.

Про последнее: Compose сравнивает **ссылки**. Изменение содержимого обычного
списка невидимо. Правильно — либо `mutableStateListOf`, либо (лучше) создавать
новый список:

```kotlin
items = items.map { if (it.id == id) it.copy(done = true) else it }
```

Вот здесь и выстреливает `data class` + `copy()` из [этапа 1](01-language.md).

### 4.6. State hoisting

Без этого принципа Compose-код разваливается. Правило: **состояние поднимается
наверх, вниз идёт только значение и колбэк**.

```kotlin
// ❌ состояние внутри — компонент неуправляем, нетестируем, значение не достать
@Composable
fun SearchField() {
    var text by remember { mutableStateOf("") }
    TextField(value = text, onValueChange = { text = it })
}

// ✅ stateless: значение снаружи, изменение — наружу
@Composable
fun SearchField(
    value: String,
    onValueChange: (String) -> Unit,
    modifier: Modifier = Modifier
) {
    OutlinedTextField(
        value = value,
        onValueChange = onValueChange,
        modifier = modifier,
        label = { Text("Поиск") },
        singleLine = true
    )
}

// вызов: состояние живёт выше — здесь в родителе
@Composable
fun SearchScreen() {
    var query by remember { mutableStateOf("") }

    Column {
        SearchField(value = query, onValueChange = { query = it })
        Text("Ищем: $query")          // ← родитель тоже видит значение
    }
}
```

Разница принципиальная: в первом варианте узнать введённый текст снаружи
невозможно, во втором — он доступен всем, кому нужен. Позже состояние переедет
ещё выше, в класс-держатель (см. 4.7), но принцип тот же.

Соглашения, которые стоит соблюдать сразу:

- `modifier: Modifier = Modifier` — первый опциональный параметр любого
  переиспользуемого компонента
- Разделять stateless-компоненты (принимают данные) и stateful-обёртки
  (держат состояние или ViewModel)
- Локальный `remember` — только для UI-состояния, которое никому больше не нужно:
  раскрыт ли аккордеон, открыто ли меню

### 4.7. Связь со `StateFlow`

Здесь стыкуются [этап 2b](02b-flow.md) и UI. Класс-держатель состояния — это тот
самый шаблон с `_state` / `asStateFlow()` из 2b.13; как его собрать целиком под
реальный экран, разбирается на [этапе 5](05-desktop-project.md).

```kotlin
// PostsModel — обычный класс с областью корутин и StateFlow<UiState>
@Composable
fun PostsScreen(model: PostsModel) {
    val state by model.state.collectAsState()

    when (val s = state) {
        is UiState.Loading -> Box(Modifier.fillMaxSize(), Alignment.Center) {
            CircularProgressIndicator()
        }
        is UiState.Success -> PostList(posts = s.posts, onDelete = model::delete)
        is UiState.Error -> ErrorView(message = s.message, onRetry = model::load)
    }
}
```

- `collectAsState()` — подписка на `StateFlow`, автоматически отписывается при
  уходе компонента из композиции
- На Android будет `collectAsStateWithLifecycle()` — то же плюс учёт lifecycle
- `when (val s = state)` — присваивание в `when` даёт smart cast внутри ветвей

### 4.8. Эффекты

```kotlin
// запустить suspend-код при появлении компонента
@Composable
fun PostsScreen(model: PostsModel) {
    LaunchedEffect(Unit) {          // Unit = один раз при входе в композицию
        model.load()
    }
    // ...
}

// перезапуск при изменении ключа
LaunchedEffect(postId) {
    model.loadPost(postId)
}

// показать snackbar по событию
LaunchedEffect(Unit) {
    model.events.collect { message ->
        snackbarHostState.showSnackbar(message)
    }
}

// очистка при уходе
DisposableEffect(Unit) {
    val listener = registerSomething()
    onDispose { listener.unregister() }
}
```

- `LaunchedEffect(keys)` — корутина, привязанная к композиции; отменяется при
  уходе компонента, перезапускается при изменении ключа
- **Почему нельзя просто вызвать `model.load()` в теле `@Composable`:** тело
  выполняется при каждой рекомпозиции, то есть запрос уйдёт десятки раз
- `rememberCoroutineScope()` — область для запуска корутин из колбэков
  (например, `onClick`)
- `DisposableEffect` — когда нужна явная очистка

### 4.9. `Scaffold` и каркас экрана

```kotlin
@Composable
fun AppScreen() {
    val snackbarHostState = remember { SnackbarHostState() }

    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("Посты") },
                actions = {
                    IconButton(onClick = onRefresh) {
                        Icon(Icons.Default.Refresh, contentDescription = "Обновить")
                    }
                }
            )
        },
        snackbarHost = { SnackbarHost(snackbarHostState) },
        floatingActionButton = {
            FloatingActionButton(onClick = onAdd) {
                Icon(Icons.Default.Add, contentDescription = "Добавить")
            }
        }
    ) { innerPadding ->
        Content(modifier = Modifier.padding(innerPadding))   // ← не забыть padding
    }
}
```

Типовая ошибка: проигнорировать `innerPadding` — контент уедет под `TopAppBar`.

---

## Не учить на этом этапе

- Анимации (`animate*AsState`, `AnimatedVisibility`, transitions)
- `Canvas`, рисование, `drawBehind`
- Кастомный `Layout`, `SubcomposeLayout`, `intrinsic` размеры
- Темизация за пределами дефолтной `MaterialTheme` (свои цвета/шрифты)
- `derivedStateOf`, `snapshotFlow`, `produceState`, `SideEffect`
- `CompositionLocal` и свои провайдеры
- `Modifier.Node`, свои модификаторы
- Навигация (на desktop хватит `when` по состоянию — см.
  [этап 5](05-desktop-project.md))
- Оптимизация рекомпозиции, `@Stable` / `@Immutable`, Layout Inspector
- Тестирование Compose

---

## Упражнения

1. **Счётчик.** `remember` + `mutableStateOf`, две кнопки (+/−) и текст.
   Затем убрать `remember` и посмотреть, что произойдёт. Затем заменить
   `mutableStateOf(0)` на обычную переменную — посмотреть снова.
2. **Форма.** Три `OutlinedTextField` и кнопка «Отправить», неактивная, пока поля
   пусты (`enabled = ...`). Валидация email с показом `isError` и `supportingText`.
3. **State hoisting.** Переписать форму из упр. 2 так, чтобы все `TextField` были
   stateless, а состояние жило в одном `data class FormState` уровнем выше.
4. **Список.** `LazyColumn` из 100 сгенерированных элементов в `Card`, с `key`.
   Каждый элемент — `Row` с текстом слева и `IconButton` удаления справа.
5. **Иммутабельность.** Реализовать удаление и переключение галочки в списке
   **через `copy()`**, без мутации. Затем попробовать мутировать `MutableList`
   напрямую и убедиться, что UI не реагирует.
6. **Три состояния.** `sealed class UiState` + `when`, кнопки для переключения
   между Loading / Success / Error вручную (без сети).
7. **`LaunchedEffect`.** Имитировать загрузку: `LaunchedEffect(Unit) { delay(2000);
   state = Success(...) }`. Затем перенести вызов в тело `@Composable` и посмотреть
   в логе, сколько раз он выполнится.
8. **Scaffold.** Собрать каркас: `TopAppBar` с кнопкой обновления, FAB, `Snackbar`
   по нажатию FAB, диалог подтверждения при удалении.
9. **Связка с корутинами.** Подключить `WeatherModel` из упражнения 9
   [этапа 2b](02b-flow.md) — вывести его `StateFlow` в UI через
   `collectAsState()`. Ничего в модели не менять.

---

## Чекпоинт

- [ ] Понятно, почему `@Composable` не должна иметь побочных эффектов
- [ ] Понятно, зачем нужны все три части: `remember`, `mutableStateOf`, `by`
- [ ] Понятно, почему мутация списка не вызывает перерисовку
- [ ] Все переиспользуемые компоненты stateless, принимают `modifier`
- [ ] Загрузка данных — в `LaunchedEffect`, не в теле функции
- [ ] `LazyColumn` с `key`
- [ ] Три состояния экрана рисуются из `sealed class`
- [ ] `StateFlow` из класса-модели выводится в UI без изменений в модели

---

## Ссылки

- [Compose Multiplatform: Desktop tutorials](https://github.com/JetBrains/compose-multiplatform/tree/master/tutorials)
- [Thinking in Compose](https://developer.android.com/develop/ui/compose/mental-model)
- [State and Jetpack Compose](https://developer.android.com/develop/ui/compose/state)
- [Lists and grids](https://developer.android.com/develop/ui/compose/lists)
- [Side-effects in Compose](https://developer.android.com/develop/ui/compose/side-effects)
- [Compose Material 3 components](https://developer.android.com/develop/ui/compose/components)
- [Compose layout basics](https://developer.android.com/develop/ui/compose/layouts/basics)
- [Глоссарий](glossary.md) — служебные слова Kotlin и термины, если что-то по ходу осталось непонятным

> Документация Android по Compose применима к Compose Desktop почти целиком:
> различается только точка входа (`Window` вместо `Activity`) и отсутствие
> Android-специфичных API.

---

**Следующий этап:** [05. Desktop-проект](05-desktop-project.md)
