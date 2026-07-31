# Этап 0. Окружение и сборка

**Время:** 1 вечер
**Предыдущий:** — · **Следующий:** [01. Язык Kotlin](01-language.md) · [Обзор](README.md)

---

## Цель

Есть проект, который собирается Gradle и запускает `fun main()`. Понятно, где что
лежит и как добавляется зависимость. Android Studio пока не нужен.

---

## Что установить

| Что | Зачем |
|---|---|
| **IntelliJ IDEA Community** | Kotlin + Gradle + Compose Desktop работают в бесплатной версии |
| **JDK 17** | LTS, требуется современным Android-тулингом; ставится через IDEA (Project Structure → SDKs → Download) |
| Git | уже есть |

Android Studio ставить на этапе 0–5 не нужно и даже вредно: он тянет за собой
эмулятор и Android-специфичные шаблоны, которые пока только отвлекают.

---

## Gradle: сразу Kotlin DSL

Файлы называются `build.gradle.kts`, не `build.gradle`. Это принципиально: Android
Studio генерирует именно `.kts`, вся современная документация — на нём. Учить
Groovy-синтаксис, чтобы потом переучиваться, — потерянное время.

Разница на глаз:

```groovy
// Groovy — старое
implementation 'io.ktor:ktor-client-cio:2.3.12'
```

```kotlin
// Kotlin DSL — то, что нужно
implementation("io.ktor:ktor-client-cio:2.3.12")
```

Kotlin DSL — это обычный Kotlin-код, поэтому IDE даёт автодополнение и переход к
исходникам. Взамен — более медленная первая сборка. Терпимо.

---

## Version catalog

Второй современный стандарт, который стоит взять сразу. Версии зависимостей живут
в отдельном файле `gradle/libs.versions.toml`, а не размазаны по модулям.

```toml
[versions]
kotlin = "2.0.20"
coroutines = "1.8.1"
ktor = "2.3.12"

[libraries]
coroutines-core = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-core", version.ref = "coroutines" }
ktor-client-cio = { module = "io.ktor:ktor-client-cio", version.ref = "ktor" }

[plugins]
kotlin-jvm = { id = "org.jetbrains.kotlin.jvm", version.ref = "kotlin" }
```

Использование:

```kotlin
plugins {
    alias(libs.plugins.kotlin.jvm)
}

dependencies {
    implementation(libs.coroutines.core)
    implementation(libs.ktor.client.cio)
}
```

Зачем это на одномодульном учебном проекте: потому что в реальном Android-проекте
модулей будет много, и catalog там обязателен. Привычка ставится сейчас бесплатно.

> Версии в примерах выше приведены для формы. Актуальные смотри на
> [kotlinlang.org](https://kotlinlang.org/docs/releases.html) и
> [ktor.io](https://ktor.io/docs/welcome.html) — Kotlin и Ktor обновляются часто.

**Порядок в этом документе.** Пример «минимальный проект целиком» ниже намеренно
написан с версиями прямо в строках — так короче и видно, что откуда берётся.
Упражнение 2 просит перенести его на catalog: разобраться в механизме своими руками
полезнее, чем скопировать готовый `libs.versions.toml`. Дальше по маршруту
(этапы 3–8) зависимости тоже приводятся в виде
`implementation("группа:артефакт:версия")` — так однозначнее читается, какой именно
артефакт нужен. В своём проекте они должны лежать в catalog.

---

## Минимальный проект целиком

Структура:

```
kotlin-playground/
├── gradle/
│   ├── libs.versions.toml
│   └── wrapper/
├── src/main/kotlin/
│   └── Main.kt
├── build.gradle.kts
├── settings.gradle.kts
└── gradlew
```

`settings.gradle.kts`:

```kotlin
rootProject.name = "kotlin-playground"
```

`build.gradle.kts`:

```kotlin
plugins {
    kotlin("jvm") version "2.0.20"
    application
}

repositories {
    mavenCentral()
}

dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.8.1")
    testImplementation(kotlin("test"))
}

kotlin {
    jvmToolchain(17)
}

application {
    mainClass.set("MainKt")   // ← Main.kt превращается в класс MainKt
}
```

`src/main/kotlin/Main.kt`:

```kotlin
fun main() {
    println("работает")
}
```

Запуск: `./gradlew run` или зелёный треугольник рядом с `main` в IDE.

---

## Что заметить сразу

**Файл без класса.** В Kotlin функция может лежать прямо в файле, на верхнем уровне.
Класс-обёртка `MainKt` создаётся компилятором автоматически — отсюда `mainClass.set("MainKt")`.

**`package` не обязан совпадать с путём.** Компилятор не проверяет это, как javac.
Совпадение всё равно стоит соблюдать — для людей, не для компилятора.

**Один файл — много классов верхнего уровня.** Ограничения «один public-класс на файл»
нет. Это меняет привычку организации кода: мелкие связанные `data class` живут в одном
файле рядом с тем, что их использует.

---

## Scratch files — главный инструмент этапа 1

`Ctrl+Alt+Shift+Insert` (или File → New → Scratch File → Kotlin) — файл-черновик вне
проекта, который запускается по кнопке и показывает результат каждой строки справа.

Для изучения языка это удобнее, чем создавать `main` под каждый эксперимент. На этапе 1
почти вся работа пойдёт именно там.

Альтернатива без установки вообще: [play.kotlinlang.org](https://play.kotlinlang.org/) —
онлайн-песочница с примерами.

---

## Не делать на этом этапе

Настройка окружения — то место, где легко потратить вечер на инструменты вместо
языка. Отложить:

- **Android Studio.** Понадобится только на [этапе 7](07-android-basics.md).
  Сейчас он тянет за собой эмулятор и Android-специфичные шаблоны, которые
  отвлекают
- **Многомодульность.** Один модуль до [этапа 5](05-desktop-project.md), где
  максимум появится второй под общие DTO
- **CI, линтеры, форматтеры, pre-commit хуки.** Полезно в командном проекте,
  бессмысленно в учебном
- **Настройку тем, шрифтов и плагинов IDE** сверх дефолтных
- **Kotlin Multiplatform-шаблон.** Обычный JVM-проект проще и делает то же самое

---

## Упражнения

1. Создать проект по структуре выше, добиться `./gradlew run`.
2. Добавить зависимость на `kotlinx-coroutines-core` через version catalog (не строкой).
3. Создать второй файл `Utils.kt` с функцией верхнего уровня, вызвать её из `Main.kt`.
4. Создать scratch file, выполнить в нём `println((1..10).map { it * it })`.
5. Инициализировать git-репозиторий, добавить `.gitignore` для Gradle-проекта
   (`.gradle/`, `build/`, `.idea/`).

---

## Чекпоинт

- [ ] `./gradlew run` печатает текст в консоль
- [ ] Зависимость добавлена через `libs.versions.toml`, а не строкой
- [ ] Понятно, откуда взялось имя `MainKt`
- [ ] Работает scratch file
- [ ] Проект под git

---

## Ссылки

- [Kotlin: Get started](https://kotlinlang.org/docs/getting-started.html)
- [Gradle Kotlin DSL Primer](https://docs.gradle.org/current/userguide/kotlin_dsl.html)
- [Version catalogs](https://docs.gradle.org/current/userguide/version_catalogs.html)
- [Kotlin Playground](https://play.kotlinlang.org/)
- [Глоссарий](glossary.md) — служебные слова Kotlin и термины, если что-то по ходу осталось непонятным

---

**Следующий этап:** [01. Язык Kotlin](01-language.md)
