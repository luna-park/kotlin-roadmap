# Этап 1. Язык Kotlin

**Время:** 4 вечера
**Предыдущий:** [00. Окружение](00-environment.md) · **Следующий:** [02. Корутины](02-coroutines.md) · [Обзор](README.md)

---

## Цель

Читать любой Kotlin-код без спотыканий. Писать свой без оглядки на Java-привычки:
не создавать геттеры руками, не писать `if (x != null)` там, где есть `?.`,
не заводить класс ради одной функции.

Только консоль и scratch files. Никакого UI и никакой асинхронности.

---

## Главная установка этапа

Kotlin читается настолько легко, что создаёт ложное ощущение «я это уже знаю».
Оно рассыпается при первой попытке написать самому. Поэтому: **на каждую тему —
сразу мелкая задача**, а не чтение подряд.

Второе: не пытаться писать «Java с другим синтаксисом». Признаки, что это происходит:
классы с явными геттерами, `for (i in 0 until list.size)`, `!!` вместо `?.`,
циклы вместо `map`/`filter`, отдельный файл на каждый мелкий класс.

> **Если по ходу встретится незнакомое служебное слово или символ** — `?.`, `?:`,
> `!!`, `it`, `by`, `field`, `::`, `->` и прочие — они собраны с короткими
> пояснениями в [глоссарии](glossary.md). Все они объясняются и здесь, по ходу
> темы, но глоссарий удобнее, когда слово встретилось раньше, чем объяснено.

---

## Темы

### 1.1. Переменные и типы

- `val` (аналог `final`) против `var`. **По умолчанию всегда `val`**, `var` — когда
  доказано, что нужно
- Вывод типов: `val x = 5` — тип есть, писать его не нужно
- Числовые типы: нет неявных расширяющих преобразований. `val l: Long = 5` работает,
  а `val l: Long = intValue` — нет, нужен `intValue.toLong()`. Отличие от Java
- `Int` vs `Int?` — разные типы, и это ядро всей системы (см. 1.2)
- `Any`, `Unit`, `Nothing`: `Any` ≈ `Object`, `Unit` ≈ `void` но это реальный объект,
  `Nothing` — тип выражения, которое никогда не возвращает управление

### 1.2. Null safety — самое важное отличие

Это главное, что даёт Kotlin, и главное, где будут ошибки на первых неделях.

```kotlin
var a: String = "text"
a = null            // не скомпилируется

var b: String? = "text"
b = null            // ок

val len = b.length          // не скомпилируется — b может быть null
val len = b?.length         // Int? — null, если b == null
val len = b?.length ?: 0    // Int — elvis-оператор даёт значение по умолчанию
val len = b!!.length        // NPE, если null. Использовать почти никогда
```

Разобрать:

- `?.` — safe call, цепочки: `user?.address?.city?.length`
- `?:` — elvis; правая часть может быть `return` или `throw`:
  `val name = input ?: return`
- `!!` — не-null утверждение. **Правило: каждое `!!` в своём коде — повод
  переписать.** Исключения бывают, но редко
- `as?` — безопасное приведение типа, даёт `null` вместо исключения
- `lateinit var` — для полей, которые точно будут проинициализированы позже
  (в Android — во `onCreate`). Только для не-null ссылочных типов
- `by lazy` — ленивая инициализация `val`, потокобезопасная по умолчанию
- **Smart cast**: после `if (b != null)` внутри блока `b` имеет тип `String`,
  вызывать `b.length` можно напрямую
- `requireNotNull(x)`, `checkNotNull(x)` — проверки с осмысленным сообщением
- `x?.let { ... }` — выполнить блок только если не null

**Platform types.** При вызове Java-кода без аннотаций Kotlin не знает про
nullability и подставляет `String!` — тип, который позволяет всё и не проверяется.
Это единственная дыра в системе, и через неё в Kotlin приходит NPE. В коде команды
на Java стоит расставить `@Nullable` / `@NonNull` — тогда Kotlin увидит их и
превратит в `String?` / `String`.

### 1.3. Функции

```kotlin
// expression body — тело из одного выражения, тип выводится
fun double(x: Int) = x * 2

// block body
fun greet(name: String): String {
    return "Привет, $name"
}

// параметры по умолчанию + именованные аргументы
fun connect(host: String, port: Int = 8080, timeout: Long = 5_000, retry: Boolean = false) { }

connect("example.com")
connect("example.com", retry = true)     // порядок не важен, читаемо
```

- Значения по умолчанию **заменяют перегрузки**. Пять `@Overload`-версий метода
  превращаются в одну функцию
- Именованные аргументы — обязательная привычка для булевых параметров:
  `connect(host, retry = true)` вместо `connect(host, true)`
- Функции верхнего уровня — вместо `static` в utility-классах
- `vararg` + spread-оператор `*array`
- Локальные функции — функция внутри функции, видит её переменные

### 1.4. Классы и свойства

```kotlin
// primary constructor прямо в объявлении; val создаёт свойство
class User(val name: String, var age: Int)

// data class — equals/hashCode/toString/copy/componentN бесплатно
data class Point(val x: Int, val y: Int)

val p1 = Point(1, 2)
val p2 = p1.copy(y = 5)      // Point(1, 5)

// object — синглтон
object Config {
    const val BASE_URL = "https://api.example.com"
}

// companion object — вместо static-членов
class Repo private constructor() {
    companion object {
        fun create() = Repo()
    }
}
```

- **Properties вместо полей.** `user.name` — это уже вызов геттера. Свои
  геттеры/сеттеры пишутся так:
  ```kotlin
  class Rect(val w: Int, val h: Int) {
      val area: Int get() = w * h          // вычисляется при обращении

      var scale: Int = 1
          set(value) { field = value.coerceAtLeast(1) }
  }
  ```
  `field` — служебное имя backing field, доступное только внутри аксессора.
  Обращаться в сеттере к `scale` вместо `field` — бесконечная рекурсия
- `init { }` — блок инициализации, аналог кода в конструкторе
- Secondary constructors: `constructor(...) : this(...)` — нужны редко, обычно
  хватает значений по умолчанию
- `data class` — для моделей данных, DTO, UI-состояний. `copy()` понадобится
  постоянно при работе с immutable state
- `object` — синглтон без boilerplate. `object : SomeInterface { }` — анонимный объект,
  аналог анонимного класса Java
- Наследование: классы **`final` по умолчанию**, нужен `open`. Методы тоже.
  `override` — обязательное ключевое слово, не аннотация
- Видимость: `public` (по умолчанию), `private`, `protected`, **`internal`**
  (видно внутри модуля — то есть внутри одной единицы компиляции Gradle).
  Прямого аналога в Java нет: package-private ограничивает пакетом, а не модулем,
  а `exports` в JPMS решает похожую задачу, но на другом уровне и почти не
  используется в Android. Практически `internal` — это способ сказать «публичное
  для моего модуля, но не часть его API»
- Вложенные классы: `class Inner` по умолчанию **статический**; для доступа к
  внешнему нужен `inner class`. Обратно к Java

### 1.5. sealed class — понадобится сразу

Ограниченная иерархия: компилятор знает все наследники, потому что они в том же модуле.

```kotlin
sealed interface UiState<out T> {
    data object Loading : UiState<Nothing>
    data class Success<T>(val data: T) : UiState<T>
    data class Error(val message: String, val cause: Throwable? = null) : UiState<Nothing>
}
```

`data object` требует Kotlin 1.9 или новее. На более старых версиях — просто
`object`, разница только в сгенерированном `toString()`.

Зачем это здесь, на первом же этапе: именно так на этапах 4–8 будет описываться
состояние экрана. Три состояния — загрузка, данные, ошибка — и `when`, который
компилятор заставит обработать все три.

Отличие от `enum`: у `enum` фиксированный набор **экземпляров**, у `sealed` —
фиксированный набор **типов**, каждый со своими полями.

> **Про имена, чтобы не путаться дальше по маршруту.** В этих документах
> используются ровно два таких типа, и они принадлежат разным слоям:
>
> | Тип | Слой | Что описывает |
> |---|---|---|
> | `ApiResult<T>` | data (вводится на [этапе 3](03-network-json.md)) | чем закончился запрос: успех, HTTP-ошибка, нет сети, кривой JSON |
> | `UiState` | UI (этапы 4–8) | что сейчас рисовать на экране |
>
> Разделение не формальное: у них разный набор вариантов. `ApiResult` различает
> четыре вида неудачи, потому что репозиторию нужна точность; UI все четыре
> сводит к одному сообщению для человека. Плюс `UiState` знает про вещи, которых
> нет в сети, — например, «список пуст» или «идёт обновление поверх старых данных».
>
> Начиная с [этапа 5](05-desktop-project.md) `UiState` встретится и во втором виде —
> как один `data class` с флагами вместо `sealed`. Там же разобрано, когда какой
> вариант удобнее.

### 1.6. `when` — не просто switch

```kotlin
// как выражение (возвращает значение)
val label = when (state) {
    is UiState.Loading -> "Загрузка…"
    is UiState.Success -> "Найдено: ${state.data}"   // smart cast!
    is UiState.Error   -> "Ошибка: ${state.message}"
}
// else не нужен: sealed + все ветки покрыты

// без аргумента — вместо цепочки if-else
val category = when {
    age < 13 -> "ребёнок"
    age < 18 -> "подросток"
    else -> "взрослый"
}

// диапазоны и наборы
when (code) {
    in 200..299 -> "успех"
    in 400..499 -> "ошибка клиента"
    301, 302, 307 -> "редирект"
    else -> "прочее"
}
```

Ключевое: `when` как **выражение** обязан быть исчерпывающим. Для `sealed` это значит,
что при добавлении нового наследника компилятор укажет все места, где нужно дописать
ветку. В Java этого не было — там `switch` без `default` просто молча ничего не делал.

### 1.7. Extension functions

Функция, которая выглядит как метод чужого класса.

```kotlin
fun String.isValidEmail(): Boolean = contains("@") && contains(".")

"test@mail.ru".isValidEmail()     // true

fun <T> List<T>.secondOrNull(): T? = if (size >= 2) this[1] else null
```

- Внутри `this` — объект-получатель, можно опустить
- Разрешаются **статически**, по объявленному типу, — это не полиморфизм.
  Вызывать `override` через extension нельзя
- Не имеют доступа к `private` членам класса
- Extension properties: `val String.firstChar: Char get() = this[0]`
- Компилируются в статический метод с получателем первым аргументом — из Java
  вызывается как `StringUtilsKt.isValidEmail(str)`

Почему это важно: в Kotlin-коде extension-функций **очень много**. Вся стандартная
библиотека коллекций — extensions. Без понимания механизма код нечитаем.

### 1.8. Лямбды и функции высшего порядка

```kotlin
// тип функции как тип значения
val op: (Int, Int) -> Int = { a, b -> a + b }

fun repeatAction(times: Int, action: (Int) -> Unit) {
    for (i in 0 until times) action(i)
}

// trailing lambda: последний аргумент-лямбда выносится за скобки
repeatAction(3) { println(it) }      // it — неявное имя единственного параметра
```

- Trailing lambda syntax — причина, по которой Compose-код выглядит как разметка,
  а не как вызовы функций. Понять это нужно **до** этапа 4
- `it` — только для одного параметра. Для двух — явные имена
- Ссылки на функции: `::println`, `String::length`, `list.map(::transform)`
- SAM-конверсия при вызове Java-интерфейсов: `button.setOnClickListener { }`
- `return` внутри лямбды — тема, которую можно отложить, но знать про
  labeled return (`return@forEach`) полезно

### 1.9. Scope-функции

Минимум, который нужен: `let` и `apply`. Остальные — распознавать при чтении.

```kotlin
// let — выполнить блок с объектом, вернуть результат блока
val length = nullable?.let { transform(it) } ?: 0

// apply — настроить объект, вернуть сам объект
val client = HttpClient().apply {
    timeout = 5_000
    retries = 3
}

// also — побочный эффект в цепочке, вернуть объект
val result = compute().also { log("получено: $it") }

// run — как let, но объект доступен как this
// with — как run, но объект передаётся аргументом
```

Шпаргалка:

| Функция | Объект внутри | Возвращает | Типичное применение |
|---|---|---|---|
| `let` | `it` | результат блока | null-безопасное преобразование |
| `apply` | `this` | сам объект | настройка при создании |
| `also` | `it` | сам объект | логирование в цепочке |
| `run` | `this` | результат блока | блок вычислений |
| `with` | `this` | результат блока | много обращений к одному объекту |

**Не насаждать в своём коде.** Вложенные `let` внутри `apply` внутри `run` — частая
болезнь новичков в Kotlin. Обычный `if` и обычная переменная часто читаются лучше.

### 1.10. Коллекции

```kotlin
val list = listOf(1, 2, 3)                    // immutable (read-only интерфейс)
val mutable = mutableListOf(1, 2, 3)
val map = mapOf("a" to 1, "b" to 2)
val set = setOf(1, 2, 3)
```

Главное отличие от Java: **операции вызываются прямо на коллекции, без `.stream()`
и без `.collect(Collectors.toList())`**.

```kotlin
// Java
list.stream().filter(x -> x > 2).map(x -> x * 10).collect(Collectors.toList());

// Kotlin
list.filter { it > 2 }.map { it * 10 }
```

Минимально нужный набор: `map`, `filter`, `filterNotNull`, `first` / `firstOrNull`,
`find`, `any` / `all` / `none`, `count`, `sumOf`, `maxByOrNull`, `sortedBy`,
`groupBy`, `associateBy`, `flatMap`, `distinct`, `take` / `drop`, `zip`,
`joinToString`, `partition`.

Что важно понимать:

- Kotlin-коллекции **eager** по умолчанию: каждая операция создаёт новый список.
  Для ленивости — `.asSequence()`, что и есть аналог `Stream`. На списках до
  тысяч элементов разница несущественна
- `List` vs `MutableList` — это интерфейсы, а не разные классы. `listOf` возвращает
  read-only **представление**; иммутабельности на уровне JVM нет
- Разделение read-only / mutable — главный приём для API: возвращать `List`,
  внутри держать `MutableList`

### 1.11. Мелочи, которые нужны постоянно

- **String templates:** `"Пользователь $name, возраст ${user.age}"`
- **Raw strings:** `"""многострочный текст"""` + `.trimIndent()`
- **Диапазоны:** `1..10`, `1 until 10`, `10 downTo 1`, `1..10 step 2`,
  `for (c in 'a'..'z')`
- **Деструктуризация:** `val (name, age) = user` (работает для `data class`,
  `Pair`, `Map.Entry`)
- **`Pair` / `Triple`** и инфиксный `to`
- **`typealias`** — знать о существовании
- **`if` как выражение:** `val max = if (a > b) a else b` (тернарного оператора нет)
- **`try` как выражение:** `val n = try { s.toInt() } catch (e: Exception) { 0 }`
- **`toIntOrNull()`**, `toLongOrNull()` — парсинг без исключений
- **`require` / `check` / `error`** — быстрые проверки аргументов и состояния

---

## Не учить на этом этапе

Всё это встретится, но не сейчас:

- `inline`, `noinline`, `crossinline`, `reified`
- Делегаты свойств кроме `by lazy` (`Delegates.observable`, свои делегаты, `by map`)
- Перегрузка операторов, `operator fun`
- DSL-билдеры и функции с receiver'ом (`A.() -> Unit`) — **кроме** понимания,
  что это такое, потому что `apply` и `flow { }` на этом построены
- Вариантность `in` / `out` в своём коде (в чужом — распознавать)
- `value class` / `inline class`
- Аннотации взаимодействия с Java: `@JvmStatic`, `@JvmOverloads`, `@JvmField`,
  `@Throws` — понадобятся, когда придётся звать Kotlin из Java в проекте команды
- Контракты, `Nothing` в сложных сценариях
- Рефлексия, `KClass`

---

## Упражнения

1. **Null safety.** Дана функция `fun findUser(id: Int): User?`. Написать функцию,
   которая по списку id возвращает список имён найденных пользователей, без `!!`
   и без явных `if (x != null)`.
2. **data class + copy.** Класс `Task(id, title, done)`. Функция, которая в
   `List<Task>` помечает задачу с заданным id выполненной, **не мутируя** исходный
   список.
3. **sealed class.** Описать `ApiResult<T>` с вариантами `Success`, `HttpError(code)`,
   `NetworkError`, `ParseError` — это тот самый тип из data-слоя, который
   понадобится на [этапе 3](03-network-json.md), так что писать его стоит сразу
   в финальном виде. Написать `when`, превращающий его в текст для пользователя.
   Убедиться, что без ветки компилятор ругается. Затем добавить пятый вариант и
   посмотреть, на какие места укажет компилятор.
4. **Extension.** `fun String.toSlug(): String` — нижний регистр, пробелы в дефисы,
   всё не-буквенно-цифровое убрать.
5. **Коллекции.** Дан `List<Employee(name, department, salary)>`. Одним выражением
   получить `Map<String, Employee>` — самый высокооплачиваемый в каждом отделе.
6. **Свойства.** Класс `Temperature` с полем в Цельсиях и вычисляемыми
   свойствами `fahrenheit`, `kelvin` (только геттеры).
7. **Итоговое.** Консольная утилита: читает JSON-файл со списком объектов
   (парсить руками, регуляркой или простым сплитом — библиотеки будут на этапе 3),
   печатает статистику: количество, группировку по полю, минимум/максимум.
   Результат возвращать через `sealed class`, аргументы командной строки
   валидировать через `require`.

---

## Чекпоинт

- [ ] В написанном коде нет ни одного `!!`
- [ ] Нет циклов `for` там, где подошёл бы `map` / `filter`
- [ ] `sealed class` + `when` используются естественно, не по инструкции
- [ ] Понятно, что `user.name` — это вызов геттера
- [ ] Понятно, почему `class` по умолчанию `final`
- [ ] Понятно, что такое trailing lambda и почему `list.map { }` пишется без скобок
- [ ] Написана хотя бы одна extension-функция, которая реально пригодилась
- [ ] Задача 7 работает

---

## Ссылки

- [Kotlin Docs: Basic syntax](https://kotlinlang.org/docs/basic-syntax.html)
- [Null safety](https://kotlinlang.org/docs/null-safety.html)
- [Sealed classes](https://kotlinlang.org/docs/sealed-classes.html)
- [Extensions](https://kotlinlang.org/docs/extensions.html)
- [Scope functions](https://kotlinlang.org/docs/scope-functions.html)
- [Collections overview](https://kotlinlang.org/docs/collections-overview.html)
- [Java to Kotlin idioms](https://kotlinlang.org/docs/java-to-kotlin-idioms-strings.html)
- [Kotlin Koans](https://play.kotlinlang.org/koans/overview) — интерактивные задачи,
  ровно под этот этап
- [Глоссарий](glossary.md) — служебные слова Kotlin и термины, если что-то по ходу осталось непонятным

---

**Следующий этап:** [02. Корутины](02-coroutines.md)
