# SPEC: дневник тренировок, MVP

Рабочее название приложения: **Lift**. Bundle ID `ru.nikita.lift` и App Group `group.ru.nikita.lift` заглушки, владелец заменит их в T00.

## 1. Цель MVP

Проверить одно: можно ли провести целую тренировку, почти не открывая приложение.

Главная метрика: среднее число действий на записанный подход. Цель не больше 2. Действие = любое касание кнопки в приложении или в Live Activity и любой вызов интента (кнопка Action, Siri).

## 2. Что входит и что нет

Входит:

1. Справочник упражнений: 60-80 базовых плюс свои.
2. Одна активная программа: дни, упражнения, целевые подходы, повторы, отдых.
3. Экран «Сегодня» с тренировкой дня и кнопкой «Начать».
4. Экран тренировки: предложенный подход, кнопки шага веса и повторов, «Готово», таймер отдыха, список сделанных подходов.
5. Live Activity на локскрине и в Dynamic Island с кнопками «−шаг», «+шаг», «−1», «+1», «Готово», «Пропустить отдых».
6. Интент «Подход сделан» для кнопки Action и Siri.
7. История тренировок и карточка упражнения с графиком рабочего веса.
8. Экспорт всей истории в CSV.
9. Локальный подсчёт метрики действий на подход.

Не входит: Apple Watch, голос, синхронизация, аккаунты, соцсеть, генерация программ, RPE, суперсеты, разминочные подходы как отдельный тип ввода, светлая тема, Android.

## 3. Ключевые сценарии

| Сценарий | Шаги пользователя | Действий |
| --- | --- | --- |
| Подход как в прошлый раз | «Готово» на локскрине или кнопка Action | 1 |
| Подход на повтор меньше | «−1», «Готово» | 2 |
| Начать тренировку | Открыть приложение, «Начать» | 1 (+ открытие) |
| Пропустить отдых | «Пропустить» | 1 |
| Посмотреть прогресс | История, упражнение | 2 |

## 4. Архитектура

```
Lift (app target) ──┐
                    ├── Packages/UI: Features ── Session ── Intents
LiftWidgets (ext) ──┘          │            │
                               DesignSystem  LiveActivityShared
                                             │
                    Packages/Core: WorkoutEngine ── Storage ── Domain
```

Правило слоёв: зависимости идут только вниз. `Core` не знает про UI, ActivityKit и WidgetKit и тестируется на macOS через `swift test`.

### Пакет `Packages/Core` (платформы iOS 17, macOS 14)

| Модуль | Отвечает за | Зависит от |
| --- | --- | --- |
| `Domain` | Value-типы предметной области, без логики хранения | нет |
| `Storage` | SwiftData-схема, репозитории, засев справочника, in-memory реализации для тестов и превью | `Domain` |
| `WorkoutEngine` | Сессия тренировки: предложение подхода, шаг, запись, отдых, переход к следующему упражнению, восстановление после перезапуска | `Domain`, `Storage` (только протоколы) |

### Пакет `Packages/UI` (платформа iOS 17)

| Модуль | Отвечает за | Зависит от |
| --- | --- | --- |
| `DesignSystem` | Токены, шрифты, `WeightFormatter`, базовые компоненты | нет |
| `LiveActivityShared` | `WorkoutActivityAttributes` и вьюхи Live Activity. Линкуется и в приложение, и в расширение | `DesignSystem`, `Intents` |
| `Session` | `SessionCoordinator`: обёртка над движком для UI, управление Live Activity, уведомление о конце отдыха, счётчик действий | `WorkoutEngine`, `Storage`, `LiveActivityShared`, `Intents` |
| `Intents` | `LogSetIntent`, `AdjustSetIntent`, `SkipRestIntent`, `LiftShortcuts` | `Domain` + протокол `SessionCommandHandler` (см. 6.5) |
| `TodayFeature`, `WorkoutFeature`, `ProgramFeature`, `HistoryFeature` | Экраны | `Session`, `DesignSystem`, `Storage` |

Интенты зависят не от `Session`, а от протокола `SessionCommandHandler`. Приложение при старте регистрирует `SessionCoordinator` как реализацию. Так `Intents` можно слинковать в расширение без всего движка.

### Таргеты (`project.yml`, XcodeGen)

- `Lift`: приложение. Точка входа, `AppDependencies` (сборка графа), регистрация `SessionCommandHandler`, `TabView`.
- `LiftWidgets`: Widget Extension. Только `ActivityConfiguration` из `LiveActivityShared`.
- Оба таргета в App Group `group.ru.nikita.lift`. `NSSupportsLiveActivities = YES` в Info.plist приложения.

### Важные технические решения

1. **Состояние сессии живёт в базе.** `LiveActivityIntent.perform()` выполняется в процессе приложения, система может поднять его из фона с нуля. Поэтому движок после каждой операции сохраняет активную тренировку, а `resume()` полностью восстанавливает сессию из `WorkoutRepository.activeWorkout()`.
2. **Таймер отдыха без пушей.** В `ContentState` лежит `restEndsAt: Date?`, вьюха рисует `Text(timerInterval:countsDown:)`. Обновлять Live Activity каждую секунду не нужно. Локальное уведомление на `restEndsAt` ставит `Session`.
3. **Отдых истекает сам.** Движок не держит таймеров: `RestState` вычисляется от `now()` при каждом снапшоте.
4. **Время инъектируется.** Движок получает `now: @Sendable () -> Date`, тесты подставляют фиксированное время.

## 5. Модель данных

Доменные типы в `Domain` (структуры, `Sendable`, `Hashable`, `Codable`). `Storage` хранит их в `@Model`-классах и маппит туда и обратно. Наружу из `Storage` `@Model`-классы не выходят.

```swift
public enum Equipment: String, Codable, Sendable, CaseIterable {
    case barbell, dumbbell, machine, cable, bodyweight
}

public struct Exercise: Identifiable, Hashable, Codable, Sendable {
    public var id: UUID
    public var name: String
    public var equipment: Equipment
    public var weightStep: Decimal   // кг; штанга 2.5, гантели 2, тренажёр и блок 5, свой вес 0
    public var isCustom: Bool
}

public struct PlannedExercise: Hashable, Codable, Sendable {
    public var exerciseID: UUID
    public var targetSets: Int       // 1...10
    public var targetReps: Int       // 1...50
    public var restSeconds: Int      // 30...600, шаг 15
}

public struct ProgramDay: Identifiable, Hashable, Codable, Sendable {
    public var id: UUID
    public var name: String          // «Грудь и трицепс»
    public var weekday: Int?         // 1 = понедельник ... 7 = воскресенье, nil = без дня
    public var exercises: [PlannedExercise]   // порядок важен
}

public struct Program: Identifiable, Hashable, Codable, Sendable {
    public var id: UUID
    public var name: String
    public var isActive: Bool
    public var days: [ProgramDay]    // порядок важен
}

public struct SetEntry: Identifiable, Hashable, Codable, Sendable {
    public var id: UUID
    public var exerciseID: UUID
    public var weight: Decimal       // кг, >= 0
    public var reps: Int             // >= 1
    public var completedAt: Date
    public var order: Int            // порядковый номер подхода внутри упражнения, с 1
    public var inputActions: Int     // сколько действий ушло на этот подход, для метрики
}

public struct Workout: Identifiable, Hashable, Codable, Sendable {
    public var id: UUID
    public var programDayID: UUID?
    public var dayName: String
    public var startedAt: Date
    public var finishedAt: Date?     // nil = активная тренировка
    public var currentExerciseIndex: Int
    public var plan: [PlannedExercise]   // копия плана дня на момент старта
    public var sets: [SetEntry]
}

public enum Adjustment: String, Codable, Sendable, CaseIterable {
    case weightUp, weightDown, repsUp, repsDown
}

public enum SessionCommand: Hashable, Sendable {
    case adjust(Adjustment), logSet, skipRest, skipExercise
}

public enum CommandSource: String, Sendable {
    case app, liveActivity, shortcut
}
```

`Adjustment`, `SessionCommand` и `CommandSource` лежат в `Domain`, чтобы модуль `Intents` не зависел от движка.

Инварианты:

- Активная тренировка одна или ни одной (`finishedAt == nil`).
- `Workout.plan` копируется при старте, поэтому правка программы не ломает идущую тренировку и историю.
- Вес хранится как `Decimal`, сравнения и шаги без `Double`.

## 6. Публичные контракты

Это замороженные интерфейсы. Задачи реализуют их и пишут код против них параллельно. Менять только через `docs/proposals/`.

### 6.1 Storage

```swift
public protocol ExerciseRepository: Sendable {
    func all() async throws -> [Exercise]
    func exercise(id: UUID) async throws -> Exercise?
    func save(_ exercise: Exercise) async throws
}

public protocol ProgramRepository: Sendable {
    func activeProgram() async throws -> Program?
    func save(_ program: Program) async throws
}

public protocol WorkoutRepository: Sendable {
    func activeWorkout() async throws -> Workout?
    func save(_ workout: Workout) async throws
    func delete(workoutID: UUID) async throws
    /// Подходы этого упражнения из последней ЗАВЕРШЁННОЙ тренировки, где оно было, по order.
    func lastSets(exerciseID: UUID) async throws -> [SetEntry]
    /// Завершённые тренировки, новые первыми.
    func finishedWorkouts(limit: Int) async throws -> [Workout]
    /// Завершённые тренировки, где есть это упражнение, новые первыми.
    func workouts(containing exerciseID: UUID, limit: Int) async throws -> [Workout]
}

public struct StorageFactory {
    public static func live(appGroup: String) throws -> Repositories
    public static func inMemory(seed: Bool) -> Repositories
}

public struct Repositories: Sendable {
    public let exercises: any ExerciseRepository
    public let programs: any ProgramRepository
    public let workouts: any WorkoutRepository
}

/// Читает Resources/exercises.json из Bundle.module.
public enum ExerciseSeed {
    public static func exercises() throws -> [Exercise]
}

/// Пример программы «Пн грудь, Ср спина, Пт ноги» из упражнений справочника.
/// Для превью и кнопки «Взять пример».
public enum DemoProgram {
    public static func make(from exercises: [Exercise]) -> Program
}
```

`live` при первом запуске засевает справочник из `exercises.json` (T02). Повторный запуск дублей не создаёт.

### 6.2 WorkoutEngine

```swift
public struct SetProposal: Equatable, Sendable {
    public var exerciseID: UUID
    public var exerciseName: String
    public var setNumber: Int        // с 1
    public var totalSets: Int
    public var weight: Decimal
    public var reps: Int
    public var weightStep: Decimal
    public var previousWeight: Decimal?   // прошлый раз, этот же номер подхода
    public var previousReps: Int?
}

public enum RestState: Equatable, Sendable {
    case idle
    case resting(endsAt: Date)
}

public struct SessionSnapshot: Equatable, Sendable {
    public var workoutID: UUID
    public var dayName: String
    public var startedAt: Date
    public var exerciseIndex: Int    // с 0
    public var exerciseCount: Int
    public var proposal: SetProposal?     // nil, если тренировка закончена
    public var completedSets: [SetEntry]  // подходы текущего упражнения
    public var rest: RestState
    public var nextExerciseName: String?
    public var isFinished: Bool
}

public enum EngineError: Error, Equatable {
    case noActiveSession
    case sessionAlreadyActive
    case dayNotFound
}

public actor WorkoutEngine {
    public init(repositories: Repositories, now: @escaping @Sendable () -> Date = Date.init)
    public func start(dayID: UUID) async throws -> SessionSnapshot
    public func resume() async throws -> SessionSnapshot?
    public func adjust(_ adjustment: Adjustment) async throws -> SessionSnapshot
    public func logSet(inputActions: Int) async throws -> SessionSnapshot
    public func skipRest() async throws -> SessionSnapshot
    public func skipExercise() async throws -> SessionSnapshot
    public func finish() async throws -> Workout
    public func discard() async throws
}
```

Правила предложения подхода (каждое покрывается тестом):

1. **Вес** для подхода N: последний записанный вес этого упражнения в текущей тренировке; если его нет, вес подхода N из `lastSets`; если в прошлый раз подходов было меньше N, вес последнего из них; если истории нет, 0.
2. **Повторы** для подхода N: повторы подхода N из `lastSets`; если нет, последние записанные повторы в текущей тренировке; если нет, `targetReps`.
3. `adjust` меняет только текущее предложение: вес на `±weightStep`, не ниже 0; повторы на `±1`, не ниже 1. Для `weightStep == 0` кнопки веса ничего не делают.
4. `logSet` записывает подход из текущего предложения, сохраняет тренировку и запускает отдых `restSeconds` текущего упражнения. Если это был последний целевой подход упражнения, `exerciseIndex` переходит к следующему. После последнего подхода последнего упражнения `isFinished = true`, отдых не запускается.
5. Отдых вычисляется от `now()`: если `now() >= endsAt`, состояние `idle`.
6. Любой метод кроме `start` и `resume` без активной сессии бросает `noActiveSession`. `start` при активной сессии бросает `sessionAlreadyActive`.
7. После каждой операции состояние сохранено в `WorkoutRepository`. Новый экземпляр движка через `resume()` возвращает тот же снапшот.

### 6.3 LiveActivityShared

```swift
public struct WorkoutActivityAttributes: ActivityAttributes {
    public struct ContentState: Codable, Hashable {
        public var exerciseName: String
        public var setNumber: Int
        public var totalSets: Int
        public var weight: Decimal
        public var reps: Int
        public var weightStep: Decimal
        public var restEndsAt: Date?
        public var isFinished: Bool
    }
    public var workoutID: UUID
    public var dayName: String
}

public struct WorkoutActivityWidget: Widget   // ActivityConfiguration: локскрин + Dynamic Island
```

`LiveActivityShared` не видит `WorkoutEngine`, поэтому маппинг `SessionSnapshot → ContentState` живёт в `Session`:

```swift
// модуль Session
extension WorkoutActivityAttributes.ContentState {
    init(_ snapshot: SessionSnapshot)
}
```

### 6.4 Session

```swift
@MainActor @Observable
public final class SessionCoordinator: SessionCommandHandler {
    public private(set) var snapshot: SessionSnapshot?
    public private(set) var lastError: EngineError?

    public init(engine: WorkoutEngine, notifications: any RestNotifier, activities: any ActivityPresenter)
    public func restore() async
    public func start(dayID: UUID) async
    public func perform(_ command: SessionCommand, source: CommandSource) async
    public func finish() async -> Workout?
    public func discard() async
}

public protocol RestNotifier: Sendable {          // локальное уведомление о конце отдыха
    func schedule(at date: Date, exerciseName: String) async
    func cancel() async
}

public protocol ActivityPresenter: Sendable {     // обёртка над ActivityKit, мок в тестах
    func start(attributes: WorkoutActivityAttributes, state: WorkoutActivityAttributes.ContentState) async
    func update(_ state: WorkoutActivityAttributes.ContentState) async
    func end(_ state: WorkoutActivityAttributes.ContentState) async
}
```

Счётчик действий: каждый `perform` увеличивает счётчик на 1. При `.logSet` значение счётчика (включая сам `logSet`) уходит в `engine.logSet(inputActions:)` и сбрасывается.

### 6.5 Intents

```swift
public protocol SessionCommandHandler: AnyObject, Sendable {
    func perform(_ command: SessionCommand, source: CommandSource) async
}

public enum SessionCommandRegistry {
    @MainActor public static var handler: (any SessionCommandHandler)?
}

public struct LogSetIntent: LiveActivityIntent          // «Подход сделан»
public struct AdjustSetIntent: LiveActivityIntent {     // параметр kind: AdjustmentKind (AppEnum)
    @Parameter var kind: AdjustmentKind
}
public struct SkipRestIntent: LiveActivityIntent
public struct LiftShortcuts: AppShortcutsProvider       // фразы для LogSetIntent
```

`AdjustmentKind` это `AppEnum`-зеркало `Adjustment` из `Domain` с русскими названиями для Shortcuts.

Если `SessionCommandRegistry.handler == nil` или сессии нет, интент возвращает диалог «Тренировка не начата» и ничего не меняет.

## 7. Дизайн

Макеты лежат в `docs/design/` как HTML: `today.dc.html`, `workout.dc.html`, `live-activity.dc.html`, `history.dc.html`. Это разметка с inline-стилями, читай её как текст: цвета, размеры, отступы и тексты там точные. Открывать в браузере не нужно (подключаемый `support.js` в репозитории отсутствует, это нормально). Пиксели во вьюхах переводи в токены ниже.

### Токены

| Токен | Значение |
| --- | --- |
| `background` | `#0E0F11` |
| `surface` | `#1A1C20` |
| `surfaceRaised` | `#23262B` |
| `control` | `#33373E` |
| `divider` | `#2A2D33` |
| `textPrimary` | `#F2F2F0` |
| `textSecondary` | `#9A9DA3` |
| `accent` | `#C8F04A` |
| `onAccent` | `#0E0F11` |
| `accentSoft` | `#2B3317` |

- Тема только тёмная: `.preferredColorScheme(.dark)` на корне.
- Шрифты: Unbounded 500/700 для заголовков и больших чисел, Onest 400-700 для текста. Оба OFL, файлы в `Resources/Fonts`, регистрация через `UIAppFonts`. Если файлов нет, фолбэк `.system(design: .rounded)`, приложение не падает.
- Числа всегда `.monospacedDigit()`.
- `Spacing`: 4, 8, 12, 16, 20, 24, 32. `Radius`: 12, 14, 18, 24.
- Кнопки не ниже 44 pt, главная «Готово» 64 pt в приложении и 52 pt в Live Activity.
- Вес показываем с запятой: `100`, `97,5`, `32`. Без «.0».

### Экраны

1. **Сегодня.** Дата, заголовок, карточка дня программы на сегодня: название, список упражнений с «подходы × вес из прошлого раза», кнопка «Начать тренировку». Если тренировка уже идёт, вместо кнопки «Продолжить». Ниже неделя: 7 кружков, обведён сегодняшний, закрашены дни с тренировками. Если на сегодня дня нет, показываем ближайший следующий и кнопку «Начать всё равно».
2. **Тренировка.** Шапка: свернуть, название дня и время с начала, «Завершить». Заголовок упражнения, «Упражнение N из M». Карточка: «Подход N из M», «В прошлый раз 100 × 8», две плитки (вес и повторы) с кнопками шага, «Готово» или плашка отдыха с «Пропустить». Под карточкой сделанные подходы. Внизу «Дальше: <упражнение>».
3. **Live Activity.** Локскрин: упражнение и номер подхода, «100 кг × 8», справа отдых. Ряд из 4 кнопок шага и «Готово» на всю ширину. Dynamic Island compact: иконка слева, таймер отдыха или «3/4» справа. Expanded: как локскрин, без ряда шага. Minimal: таймер.
4. **Программа.** Список дней. День: упражнения, у каждого подходы, повторы, отдых. Добавление упражнения из справочника с поиском, своё упражнение.
5. **История.** Список тренировок, новые сверху. Строка метрики «В среднем X действий на подход за 30 дней». Карточка упражнения: рабочий вес, лучший подход, график рабочего веса (Swift Charts), последние тренировки. Кнопка экспорта CSV в тулбаре.

Рабочий вес тренировки = максимальный вес среди подходов этого упражнения в ней. Лучший подход = максимальный вес, при равенстве больше повторов.

## 8. CSV

Одна строка на подход, UTF-8 с BOM, разделитель `;`, десятичная запятая (открывается в Excel и Numbers с русской локалью).

```
date;workout;exercise;set;weight_kg;reps
2026-10-06;Грудь и трицепс;Жим лёжа;1;100;8
```

## 9. Открытые решения (владелец)

- Рынок старта: русскоязычный по умолчанию, английские строки во второй очереди.
- Минимальная iOS 17.0. Поднять до 18 можно позже без изменений контрактов.
