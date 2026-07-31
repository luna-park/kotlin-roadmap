# Этап 6. Что изменилось в Android

**Время:** 2 вечера на основной материал + чтение behavior changes по мере
надобности (см. ниже). Кода писать не нужно.
**Предыдущий:** [05. Desktop-проект](05-desktop-project.md) · **Следующий:** [07. Android-минимум](07-android-basics.md) · [Обзор](README.md)

---

## Цель

Этот этап — не про изучение нового, а про **выгрузку устаревшего**. Задача:
понять, какие привычки «старого стиля» больше не применяются и почему, чтобы не
писать современный проект старыми паттернами.

Кода писать не нужно. Это чтение с ручкой.

---

## Таблица замен

| Было | Стало | Почему |
|---|---|---|
| `AsyncTask` | корутины + `viewModelScope` | deprecated с API 30; утечки, привязка к Activity |
| `Loader`, `CursorLoader` | `Flow` из Room | deprecated |
| `Handler` + `postDelayed` | `delay` в корутине | не нужно снимать `removeCallbacks` |
| `Thread` + `runOnUiThread` | `withContext(Dispatchers.Main)` | структурная отмена |
| `findViewById` | Compose / ViewBinding | нет runtime-ошибок и приведений типов |
| `Activity` на каждый экран | одна `Activity` + навигация внутри | быстрее переходы, общее состояние |
| `Fragment` + `FragmentManager` | `@Composable` + Navigation Compose | без транзакций и жизненного цикла фрагмента |
| `LiveData` | `StateFlow` | не привязана к Android, есть операторы |
| `RecyclerView` + `Adapter` + `ViewHolder` + `DiffUtil` | `LazyColumn` с `key` | ~200 строк boilerplate исчезают |
| `SQLiteOpenHelper` + `Cursor` | Room | компиляционная проверка SQL |
| `ContentObserver` для инвалидации | `Flow<List<T>>` из Room | автоматическая эмиссия при изменении таблицы |
| `SharedPreferences` | DataStore | `suspend`-API, нет `commit()` на главном потоке |
| `startActivityForResult` | Activity Result API | deprecated; типизированные контракты |
| `Picasso`, старый Glide | Coil | Kotlin-first, `AsyncImage` одной строкой |
| `AsyncTask` для фона + `Service` | WorkManager | системные ограничения фона |
| `onSaveInstanceState` для всего | `ViewModel` + `SavedStateHandle` | ViewModel переживает поворот сам |
| `AppCompatActivity` + XML темы | `ComponentActivity` + `MaterialTheme` | тема в Kotlin, не в XML |
| Butterknife, Dagger 2 руками | ViewBinding / Hilt | кодогенерация без ручной обвязки |

---

## Ключевые смены концепций

### 1. Activity больше не «экран»

Раньше: одна `Activity` = один экран, состояние в полях `Activity`, при повороте
всё пересоздаётся, спасаемся `onSaveInstanceState`.

Теперь: **одна `Activity` на всё приложение**, экраны — это `@Composable`-функции
(или фрагменты в XML-проектах), состояние живёт в `ViewModel`, которая поворот
переживает.

`Activity` осталась только как:

- точка входа из системы (launcher, intent, deep link)
- владелец окна
- поставщик системных API (permissions, activity result)

Практический вывод: жизненный цикл `Activity` теперь нужно знать **не для того,
чтобы в нём хранить состояние**, а чтобы понимать, почему состояние в нём хранить
нельзя.

### 2. Состояние поднялось из UI в ViewModel

Это ровно тот же state hoisting, что на [этапе 4](04-compose-desktop.md), но на
уровне архитектуры. `ViewModel` создаётся при первом обращении и живёт, пока живёт
её владелец (`Activity` / composable-назначение навигации) — **включая поворот
экрана и смену конфигурации**.

Механизм: `ViewModelStore` сохраняется системой отдельно от `Activity`, поэтому
пересоздание `Activity` не задевает `ViewModel`.

Чего это не покрывает: убийство процесса системой при нехватке памяти.
Для этого — `SavedStateHandle`, но на большинстве экранов достаточно перезагрузки
данных.

### 3. Асинхронность стала структурной

Всё, что было выучено на этапах [2](02-coroutines.md) и [2b](02b-flow.md), в Android получает
готовые области:

| Область | Живёт | Диспетчер |
|---|---|---|
| `viewModelScope` | до `onCleared()` ViewModel | `Main.immediate` |
| `lifecycleScope` | до уничтожения `Activity`/`Fragment` | `Main.immediate` |
| `rememberCoroutineScope()` | пока composable в композиции | `Main` |

Правило: **фоновая работа — в `viewModelScope`**, потому что он переживает поворот.
`lifecycleScope` — только для того, что связано именно с UI (подписка на состояние).

### 4. UI стал декларативным

Уже пройдено на [этапе 4](04-compose-desktop.md), здесь только про переход:
Compose и Views **совместимы**. В существующем проекте можно:

- вставить Compose в XML через `ComposeView`
- вставить View в Compose через `AndroidView`

Поэтому миграция реального проекта делается по экранам, а не целиком. Это важно
знать, если проект команды на XML.

---

## Платформенные изменения, которые ломают старые проекты

Это то, что нужно прочитать внимательно — здесь чаще всего возникают
«у меня всё работало, а на новом Android нет».

### Runtime permissions (API 23+, но постоянно ужесточаются)

Разрешение в манифесте больше не даёт доступ. Нужно спрашивать в рантайме, и
пользователь может отказать, отозвать позже или выбрать «только один раз».
Плюс scoped-варианты: доступ к медиа по типам, приблизительная геолокация,
`POST_NOTIFICATIONS` как отдельное разрешение (API 33+).

Для этого маршрута нужен только `INTERNET` — он не runtime, спрашивать не надо.
Но знать про механизм необходимо.

### Ограничения фоновой работы

Начиная с API 26 нельзя просто запустить `Service` и работать. Что действует:

- **Doze / App Standby** — система усыпляет приложение
- **Background execution limits** — фоновые сервисы останавливаются
- **Foreground service types** (API 34+) — обязательное объявление типа и
  обоснование
- **Exact alarms** требуют отдельного разрешения (API 31+)

Вывод для приложения из этого маршрута: любая работа «когда приложение закрыто» —
через **WorkManager**, и только он. Прямой `Thread` в фоне не выживет.

### Scoped storage (API 29+)

Прямого доступа к `/sdcard` больше нет. Свои файлы — во внутреннем хранилище
приложения (`context.filesDir`, разрешения не нужны). Чужие файлы — через
`MediaStore` или системный picker (`ActivityResultContracts.OpenDocument`).

`WRITE_EXTERNAL_STORAGE` фактически не работает.

### `targetSdk` — это переключатель поведения

Ключевая вещь, которую часто понимают неправильно:

- `compileSdk` — против какого API компилируешь (можно ставить последний всегда)
- `minSdk` — минимальная версия ОС
- **`targetSdk` — заявление «я протестирован на этой версии»**. Система включает
  для приложения все поведенческие изменения до этой версии включительно

Поднять `targetSdk` — значит принять новые ограничения. Именно поэтому в старых
проектах его боятся поднимать, а Google Play требует. Перед подъёмом читается
раздел «Behavior changes» для каждой версии.

### Edge-to-edge (API 35+)

Приложение рисуется под системными панелями по умолчанию. Нужно обрабатывать
insets, иначе контент уедет под статус-бар. В Compose — `WindowInsets` и
`Modifier.safeDrawingPadding()`, `Scaffold` делает это сам.

### Прочее, что стоит знать

- **`PendingIntent`** требует явного флага mutability (API 31+)
- **Exported components** — `android:exported` обязателен для компонентов с
  intent-filter (API 31+)
- **Package visibility** (API 30+) — нельзя увидеть список установленных
  приложений без `<queries>`
- **Photo picker** — рекомендованный способ выбора медиа без разрешений

---

## Что читать (порядок)

1. **[Guide to app architecture](https://developer.android.com/topic/architecture)** —
   официальная рекомендация Google. Это ровно та архитектура, что была собрана на
   [этапе 5](05-desktop-project.md): UI → ViewModel → Repository → data source.
   Прочитать целиком, включая раздел про UI layer и UI state
2. **[Behavior changes](https://developer.android.com/about/versions)** — по одной
   странице на версию, от текущего `targetSdk` проекта до последней. Читать разделы
   «Behavior changes: apps targeting API N».

   Честно про объём: если проект стоит на API 28, это семь-восемь плотных страниц,
   и целиком за вечер их не прочитать. Рабочий подход — сначала пробежать заголовки
   всех версий и отметить только то, что касается твоего проекта (обычно это сеть,
   хранилище, фон и разрешения), а подробно читать уже отмеченное. Остальное —
   справочник, к которому возвращаешься при подъёме `targetSdk`.
3. **[Coroutines on Android](https://developer.android.com/kotlin/coroutines)** —
   привязка знаний с этапов [2](02-coroutines.md) и [2b](02b-flow.md) к платформе
4. **[Migrate from LiveData to Flow](https://developer.android.com/kotlin/flow/stateflow-and-sharedflow)** —
   если в проекте команды есть LiveData
5. **[Compose и Views вместе](https://developer.android.com/develop/ui/compose/migrate/interoperability-apis)** —
   если проект гибридный

---

## Не учить на этом этапе

Этап целиком про выгрузку устаревшего, и соблазн параллельно начать изучать всё
новое велик. Отложить:

- **Подробности API, которые не нужны для клиент-серверного приложения:**
  Services, BroadcastReceiver, ContentProvider, AIDL, Notifications, CameraX
- **WorkManager в деталях.** Достаточно знать, что фоновая работа идёт только через
  него; API понадобится, когда появится задача синхронизации
- **Runtime permissions в деталях.** На этом маршруте нужен только `INTERNET`,
  который не runtime. Механизм понять — да, API учить — нет
- **Миграцию существующего проекта.** Соблазн начать переписывать рабочий код после
  чтения этого этапа высок; сначала стоит собрать своё приложение с нуля
  (этапы 7–8), а уже потом трогать чужое
- **Jetpack-библиотеки, которых нет в маршруте:** Paging, Startup, Baseline
  Profiles, App Search

---

## Упражнения (без кода)

1. **Инвентаризация.** Выписать из проекта команды (или из своего старого) всё,
   что попадает в левую колонку таблицы замен. Для каждого пункта — во что
   превращается.
2. **`targetSdk`.** Найти текущий `targetSdk` проекта, открыть behavior changes
   для следующей версии, выписать, что сломается при подъёме.
3. **Мысленный перенос.** Взять любой экран старого проекта и расписать на бумаге:
   что станет `UiState`, что станет `ViewModel`, что станет `@Composable`.
   Где сейчас лежит состояние и куда оно переедет.
4. **Проверка понимания ViewModel.** Ответить себе: почему `ViewModel` нельзя
   давать ссылку на `Activity` или `View`? Что произойдёт при повороте, если дать?

---

## Чекпоинт

- [ ] Понятно, почему `Activity` больше не «экран»
- [ ] Понятно, чем `ViewModel` переживает поворот и чего не переживает
- [ ] Понятно различие `compileSdk` / `minSdk` / `targetSdk`
- [ ] Понятно, почему фоновая работа — только через WorkManager
- [ ] Прочитан Guide to app architecture, и видно, что архитектура этапа 5 — она же
- [ ] Составлен список того, что из старых привычек больше не применяется
- [ ] Понятно, что Compose и Views совместимы, миграция возможна по экранам

---

## Ссылки

- [Guide to app architecture](https://developer.android.com/topic/architecture)
- [Android versions and behavior changes](https://developer.android.com/about/versions)
- [Coroutines on Android](https://developer.android.com/kotlin/coroutines)
- [StateFlow and SharedFlow](https://developer.android.com/kotlin/flow/stateflow-and-sharedflow)
- [ViewModel overview](https://developer.android.com/topic/libraries/architecture/viewmodel)
- [Background work overview](https://developer.android.com/develop/background-work/background-tasks)
- [Storage updates / scoped storage](https://developer.android.com/about/versions/11/privacy/storage)
- [Compose ↔ Views interoperability](https://developer.android.com/develop/ui/compose/migrate/interoperability-apis)
- [Глоссарий](glossary.md) — служебные слова Kotlin и термины, если что-то по ходу осталось непонятным

---

**Следующий этап:** [07. Android-минимум](07-android-basics.md)
