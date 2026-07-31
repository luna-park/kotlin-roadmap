# Этап 2. Корутины

**Время:** 2 вечера
**Предыдущий:** [01. Язык Kotlin](01-language.md) · **Следующий:** [02b. Flow](02b-flow.md) · [Обзор](README.md)

---

## Цель

Понимать, **кто** запускает асинхронный код, **где** он выполняется и **когда**
останавливается. Не «уметь написать `launch`», а именно эти три вопроса — из них
растёт всё остальное.

По-прежнему только консоль. UI добавит свои сложности, их лучше отделить.

---

## Зачем это перед сетью и UI

Ktor Client целиком построен на `suspend`-функциях. `ViewModel` в Android — это
класс с корутинной областью. И `Flow`, которому посвящён
[следующий этап](02b-flow.md), тоже целиком стоит на корутинах: без них он
непонятен в принципе.

Пропустить этот этап технически нельзя: без него всё дальнейшее будет магией,
которую копируют, не понимая.

---

## Темы

### 2.1. `suspend fun`

```kotlin
suspend fun loadUser(id: Int): User {
    delay(1000)                 // не блокирует поток
    return User(id, "Иван")
}
```

- `suspend`-функцию можно вызвать **только** из другой `suspend`-функции или из
  корутины. Из обычного `main` — нельзя
- Это не «функция, выполняющаяся в другом потоке». Это функция, которая **умеет
  приостанавливаться**, освобождая поток
- `delay(1000)` ≠ `Thread.sleep(1000)`. Первое отпускает поток, второе его держит
- Компилятор превращает `suspend`-функцию в конечный автомат (CPS-трансформация);
  локальные переменные живут в объекте на куче, а не в стеке потока. Отсюда
  дешевизна: тысячи корутин на одном потоке
- Аналогия из Java: `suspend fun(): T` ≈ `CompletableFuture<T>`, только пишется
  как обычный последовательный код

### 2.2. Структурная конкурентность — ядро модели

Это главная идея, и она важнее синтаксиса.

Любая корутина запускается **внутри области** (`CoroutineScope`). Область владеет
своими дочерними корутинами: пока не завершились дети — не завершена родительская
область; отменили область — отменились все дети.

```kotlin
val scope = CoroutineScope(SupervisorJob() + Dispatchers.Default)

scope.launch { /* задача 1 */ }
scope.launch { /* задача 2 */ }

scope.cancel()     // обе отменены, ничего не течёт
```

Сравнение с тем, что было в Java:

| Java | Kotlin |
|---|---|
| `executor.submit()` → `Future.cancel(true)` руками | отмена области отменяет всё |
| `CompositeDisposable.clear()` в `onDestroy` | автоматически при смерти области |
| забыл отписаться → утечка | забыть невозможно: нет ручной подписки |

Практический вывод: **в реальном коде почти никогда не создают область руками** —
берут готовую (`viewModelScope`, `lifecycleScope`). Свою — только на этапе
[05](05-desktop-project.md), где AndroidX-овой ещё нет.

### 2.3. `Job` и жизненный цикл

- `Job` — хэндл корутины. `launch` возвращает `Job`
- Состояния: New → Active → Completing → Completed, плюс Cancelling → Cancelled
- `job.cancel()`, `job.join()` (дождаться), `job.cancelAndJoin()`
- `job.isActive`, `isCompleted`, `isCancelled`
- Родительско-дочерние отношения: отмена родителя отменяет детей; **необработанное
  исключение в ребёнке по умолчанию убивает родителя и всех братьев**
- `SupervisorJob` меняет последнее: дети изолированы, падение одного не убивает
  остальных. Именно поэтому он нужен в области, привязанной к UI

### 2.4. Билдеры: `launch`, `async`, `runBlocking`

```kotlin
// fire-and-forget, возвращает Job
val job = scope.launch { doWork() }

// с результатом, возвращает Deferred<T>
val deferred = scope.async { loadUser(1) }
val user = deferred.await()

// параллельно — типовой паттерн
suspend fun loadAll() = coroutineScope {
    val users = async { loadUsers() }
    val posts = async { loadPosts() }
    val tags  = async { loadTags() }
    Screen(users.await(), posts.await(), tags.await())    // ~время самого долгого
}
```

- `launch` — когда результат не нужен
- `async` / `await` — когда нужен, особенно для **параллельных** запросов
- `runBlocking` — блокирует текущий поток до завершения. **Только в `main()` и в
  тестах.** В production-коде (и особенно в Android) это ошибка
- `coroutineScope { }` — suspend-функция, создающая вложенную область; возвращает
  управление только когда все дети закончились. Правильный способ «сделать
  несколько дел параллельно и дождаться всех»
- `withContext(ctx) { }` — сменить контекст и вернуть результат
- `supervisorScope { }` — как `coroutineScope`, но с изоляцией падений

Ошибка новичка: `scope.launch { }` внутри `suspend`-функции вместо `coroutineScope { }`.
Первое «отпускает» корутину в чужую область и ломает структурность.

### 2.5. Диспетчеры

```kotlin
withContext(Dispatchers.IO) { readFile() }
```

| Диспетчер | Для чего | Размер пула |
|---|---|---|
| `Dispatchers.Default` | вычисления, парсинг, сортировка | по числу ядер |
| `Dispatchers.IO` | сеть, диск, БД — то, что ждёт | до 64 потоков |
| `Dispatchers.Main` | UI (Android / Compose) | один поток |
| `Dispatchers.Unconfined` | не использовать | — |

- `withContext` — способ сменить поток. Аналог `executor.execute` + возврат результата
- Наследование: корутина без явного диспетчера берёт его у родителя
- **Правило хорошего API:** `suspend`-функция сама выбирает свой диспетчер внутри,
  а не заставляет вызывающего помнить об этом:
  ```kotlin
  suspend fun loadFile(): String = withContext(Dispatchers.IO) { file.readText() }
  ```
- В Ktor и Room этот принцип уже соблюдён — их `suspend`-функции безопасно звать
  из `Dispatchers.Main`. Отсюда: `withContext(Dispatchers.IO)` вокруг вызова
  Ktor — избыточен

### 2.6. Отмена

```kotlin
val job = scope.launch {
    try {
        repeat(1000) { i ->
            delay(100)              // точка отмены
            println(i)
        }
    } finally {
        println("очистка")          // выполнится при отмене
    }
}
delay(500)
job.cancel()
```

- Отмена **кооперативная**: корутина должна её проверять. Проверяют все
  suspend-функции из библиотеки (`delay`, `withContext`, `emit`, вызовы Ktor)
- Цикл без suspend-вызовов **не отменяется**. Прямой аналог потока, который не
  смотрит на `Thread.interrupted()`. Лечится `yield()` или `ensureActive()`
- Отмена реализована через `CancellationException`. Отсюда два правила:
  - **не глотать `CancellationException`** в `catch (e: Exception)`;
    если ловишь широко — пробрось её дальше
  - в `finally` нельзя вызывать suspend-функции (корутина уже отменена);
    если очень нужно — `withContext(NonCancellable)`
- `withTimeout(1000) { }` — бросает `TimeoutCancellationException`;
  `withTimeoutOrNull(1000) { }` — возвращает `null`

### 2.7. Ошибки

```kotlin
// в suspend-коде обычный try/catch работает как обычно
suspend fun safeLoad(): Result<User> = try {
    Result.success(loadUser(1))
} catch (e: CancellationException) {
    throw e                                  // ← обязательно пробросить
} catch (e: Exception) {
    Result.failure(e)
}
```

Главное отличие от колбэков и от Rx: **обычный `try/catch` работает**. Не нужны
`onError`-каналы. Это прямое следствие того, что `suspend`-код — последовательный.

- `launch` — исключение уходит наверх по иерархии Job → к `CoroutineExceptionHandler`
  → к обработчику необработанных исключений потока
- `async` — исключение «хранится» в `Deferred` и бросается при `await()`
- `SupervisorJob` / `supervisorScope` — падение ребёнка не убивает братьев
- `CoroutineExceptionHandler` — знать о существовании; в области, привязанной к UI,
  обычно не нужен, потому что ошибки обрабатываются в репозитории

### 2.8. Типовые паттерны, которые понадобятся дальше

```kotlin
// 1. Отмена предыдущей задачи при новом запуске
private var searchJob: Job? = null

fun search(query: String) {
    searchJob?.cancel()                       // старая задача больше не нужна
    searchJob = scope.launch { /* ... */ }
}
```

Это ручной вариант того, что на [этапе 2b](02b-flow.md) сделает один оператор
`flatMapLatest`. Полезно один раз написать руками, чтобы понимать, что именно
оператор делает за тебя.

```kotlin
// 2. Повтор с задержкой
suspend fun <T> retryIO(times: Int = 3, block: suspend () -> T): T {
    repeat(times - 1) {
        try { return block() } catch (e: IOException) { delay(1000) }
    }
    return block()      // последняя попытка — без перехвата, ошибка уйдёт наверх
}
```

`return` внутри `repeat { }` работает, потому что `repeat` — inline-функция:
её лямбда компилируется прямо в тело вызывающей функции, поэтому `return`
возвращает из `retryIO`, а не из лямбды.

```kotlin
// 3. Ограничение времени
val result = withTimeoutOrNull(5_000) {
    loadSomething()
}
if (result == null) { /* не успели */ }
```

---

## Не учить на этом этапе

- **`Flow` целиком** — ему посвящён [следующий этап](02b-flow.md). Сейчас важно
  разобраться с корутинами; Flow поверх непонятых корутин учить бессмысленно
- `Channel`, `produce`, `actor`
- `select`
- `Mutex`, `Semaphore`
- Кастомные диспетчеры, `asCoroutineDispatcher`, `limitedParallelism`
- `CoroutineExceptionHandler` в деталях
- `runTest`, `TestDispatcher` — тестирование отложено в [приложение](appendix-testing.md)
- Внутренности CPS-трансформации (полезно для понимания, но не для работы)
- `callbackFlow` / `suspendCancellableCoroutine` — понадобятся при обёртке
  Java-колбэков; на Android вернуться к этому

---

## Упражнения

1. **Последовательно vs параллельно.** Три `suspend`-функции по 1 секунде каждая.
   Замерить `System.currentTimeMillis()` для варианта с последовательными вызовами
   и для варианта с `async`. Убедиться: 3 с против 1 с.
2. **Отмена.** Запустить `launch` с бесконечным `while (true) { delay(100) }`,
   отменить через 500 мс, убедиться, что `finally` выполнился.
3. **Неотменяемый цикл.** Заменить `delay` на пустой `while (true) { }` с счётчиком —
   убедиться, что `cancel()` не помогает. Починить через `ensureActive()`.
4. **Диспетчеры.** Напечатать `Thread.currentThread().name` в `Default`, `IO` и
   внутри `withContext`, посмотреть, как меняется.
5. **Ошибка в `async`.** Убедиться, что исключение бросается на `await()`, а не при
   вызове `async`.
6. **`SupervisorJob`.** Две корутины в одной области, одна падает. Сравнить поведение
   с `Job()` и с `SupervisorJob()`.
7. **`coroutineScope` против `launch`.** Написать `suspend fun loadAll()`, которая
   параллельно выполняет три задачи и возвращает результат только когда готовы все.
   Затем заменить `coroutineScope { }` на `scope.launch { }` и увидеть, что функция
   возвращает управление немедленно, не дождавшись ничего.
8. **Отмена предыдущей задачи.** Реализовать паттерн 1 из 2.8: метод `search`,
   отменяющий предыдущий запуск. Вызвать его пять раз подряд с интервалом 100 мс
   при «запросе» длиной 500 мс — убедиться по логам, что до конца дошёл только
   последний.
9. **Итоговое.** Класс `WeatherModel` с методом `refresh()`: имитация загрузки через
   `delay`, случайное исключение, результат в обычное поле (без `Flow` — он на
   [следующем этапе](02b-flow.md)). Своя область корутин, метод `close()`.
   В `main()` вызвать `refresh()` несколько раз, затем `close()` и убедиться, что
   незавершённые задачи отменились.

   Ловушка: `runBlocking` завершится только когда завершатся все запущенные в нём
   корутины. Если `close()` не вызвать, программа повиснет — это и есть наглядная
   демонстрация того, зачем нужна отмена области.

---

## Чекпоинт

- [ ] Понятно, почему `suspend fun` нельзя вызвать из обычной функции
- [ ] Понятно, чем `delay` отличается от `Thread.sleep`
- [ ] Понятно, что означает «отмена области отменяет всех детей» и почему это
      снимает задачу отписки
- [ ] Понятно, когда `launch`, а когда `async`
- [ ] Ясно, что `runBlocking` — только для `main` и тестов
- [ ] `CancellationException` пробрасывается, а не глотается
- [ ] Понятно, зачем `coroutineScope { }`, если есть `launch`
- [ ] Написан класс со своей областью корутин и методом `close()` (упр. 9)
- [ ] Проверено: без отмены области `runBlocking` не завершается
- [ ] Понятно, что `withContext(Dispatchers.IO)` вокруг вызова Ktor — избыточен

---

## Ссылки

- [Coroutines guide](https://kotlinlang.org/docs/coroutines-guide.html)
- [Coroutine basics](https://kotlinlang.org/docs/coroutines-basics.html)
- [Cancellation and timeouts](https://kotlinlang.org/docs/cancellation-and-timeouts.html)
- [Exception handling](https://kotlinlang.org/docs/exception-handling.html)
- [Composing suspending functions](https://kotlinlang.org/docs/composing-suspending-functions.html)
- [Coroutine context and dispatchers](https://kotlinlang.org/docs/coroutine-context-and-dispatchers.html)
- [Coroutines hands-on: intro](https://kotlinlang.org/docs/coroutines-and-channels.html)
- [Глоссарий](glossary.md) — служебные слова Kotlin и термины, если что-то по ходу осталось непонятным

---

**Следующий этап:** [02b. Flow](02b-flow.md)
