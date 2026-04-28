# Архитектура приложения ВОЛС Mobile

## Общая структура

Приложение построено на принципах **Clean Architecture** с разделением на три слоя:

```
┌─────────────────────────────────────────┐
│           UI Layer (Screens)            │  Composable-функции, Navigation
├─────────────────────────────────────────┤
│       Presentation Layer (ViewModels)   │  BlocViewModel, State/Event/Action
├─────────────────────────────────────────┤
│         Domain Layer (Use Cases)        │  Бизнес-логика, интерфейсы репозиториев
├─────────────────────────────────────────┤
│     Data Layer (Repositories, DB, Net)  │  Room, Ktor, KVault, FileStorage
└─────────────────────────────────────────┘
```

## Структура модулей Gradle

```
vols-mobile/
├── composeApp/          — единственный KMP модуль (shared + platform код)
│   └── src/
│       ├── commonMain/  — платформо-независимый код (вся бизнес-логика)
│       ├── androidMain/ — Android-специфичный код (HTTP engine, Entry point)
│       └── iosMain/     — iOS-специфичный код (HTTP engine)
└── iosApp/              — Xcode project, Swift UI entry point
```

## Структура пакетов (commonMain)

```
ru.meket.volsmobile/
├── core/
│   ├── data/
│   │   ├── database/        # Room DB, entities, DAOs, converters
│   │   ├── network/         # Ktor clients, HTTP config, DTO models
│   │   ├── repository/      # Реализации репозиториев (8 шт.)
│   │   ├── storage/         # UserStorage (KVault), FileStorage
│   │   └── manager/         # UserManager, SyncManager, PushManager
│   ├── domain/
│   │   ├── models/          # Доменные модели (чистые data class)
│   │   ├── repository/      # Интерфейсы репозиториев
│   │   └── usecase/         # Интерфейсы use-case
│   ├── ui/
│   │   ├── bloc/            # BlocViewModel базовый класс
│   │   ├── components/      # Переиспользуемые UI-компоненты
│   │   └── theme/           # Material3: цвета, типографика
│   ├── di/                  # VolsSingleton.kt — Service Locator
│   └── utils/               # Утилиты и расширения
└── screens/                 # Все экраны приложения
    ├── login/
    ├── map/
    ├── agreement/
    ├── changePassword/
    ├── details/
    │   ├── addVols/
    │   ├── choosePillarPoint/
    │   ├── pillarInfo/
    │   ├── previewPhoto/
    │   ├── selectPowerLine/
    │   ├── selectTask/
    │   └── startTask/
    └── license/
```

## Dependency Injection — VolsSingleton

**Паттерн**: Service Locator (кастомная реализация, не Hilt/Koin/Dagger)  
**Файл**: `core/di/VolsSingleton.kt`

Синглтон с lazy-инициализацией всех зависимостей. Содержит:

| Тип | Объект | Назначение |
|-----|--------|-----------|
| Database | `AppDatabase` | Room БД |
| Network | `VolsClient`, `LoginClient` | HTTP клиенты |
| Repositories | 8 репозиториев | Доступ к данным |
| Use Cases | `LoginUseCase`, `ChangePasswordUseCase`, `AcceptAgreementUseCase` | Бизнес-операции |
| Storage | `UserStorage`, `FileStorage` | Персистентность |
| Managers | `UserManager`, `SyncManager`, `PushManager` | Оркестраторы |
| Utils | `Grant` (permissions), `DeviceIdProvider` | Платформенные утилиты |

```kotlin
// Пример использования в ViewModel
object VolsSingleton {
    val appDatabase: AppDatabase by lazy { ... }
    val volsClient: VolsClient by lazy { ... }
    val pillarRepository: PillarRepository by lazy { ... }
    val loginUseCase: LoginUseCase by lazy { ... }
    val userManager: UserManager by lazy { ... }
}
```

## BlocViewModel — паттерн управления состоянием

**Файл**: `core/ui/bloc/BlocViewModel.kt`

Каждый экран состоит из 5 файлов:

| Файл | Тип | Назначение |
|------|-----|-----------|
| `XxxScreen.kt` | `@Composable` | UI-отображение |
| `XxxViewModel.kt` | `BlocViewModel<S,E,A>` | Бизнес-логика экрана |
| `XxxState.kt` | `data class` | Полное состояние UI |
| `XxxEvent.kt` | `sealed class` | События от пользователя |
| `XxxAction.kt` | `sealed class` | Одноразовые эффекты (навигация, toast) |

```kotlin
abstract class BlocViewModel<STATE, EVENT, ACTION>(
    savedStateHandle: SavedStateHandle
) : ViewModel() {

    val state: StateFlow<STATE>        // UI наблюдает реактивно
    val actions: SharedFlow<ACTION>    // Одноразовые side-effects

    abstract fun obtainEvent(event: EVENT)

    fun runInIOScope(block: suspend CoroutineScope.() -> Unit) { ... }
    fun runInMainScope(block: suspend CoroutineScope.() -> Unit) { ... }
}
```

**Поток данных на экране:**
```
Composable                  ViewModel
   │                           │
   ├──obtainEvent(event)───►  obtainEvent()
   │                           │
   │                           ├── setState { ... }
   │                           │
   ◄──state.collectAsState()──►│
   │                           │
   │                           ├── sendAction(action)
   │                           │
   ◄──actions.collect { }─────►│
```

## Навигация

**Библиотека**: AndroidX Navigation Compose (Multiplatform) v2.9.2  
**Файл**: `screens/App.kt`

**Типизированные маршруты** (`MainPages.kt`):

```kotlin
sealed class MainPages {
    @Serializable object Loading
    @Serializable object Map
    @Serializable data class Auth(val isReLogin: Boolean)
    @Serializable data class ChangePassword(val userId: Long, val oldPassword: String)
    @Serializable data class AcceptAgreement(val userId: Long)
}
```

**Стартовая логика** (App.kt при инициализации):
1. `VolsSingleton.deviceIdProvider.init()` — инициализация Device ID
2. `VolsSingleton.userManager.checkForAppStart()` — определение стартового экрана
3. Переход на нужный маршрут по результату `StartCheckResult`

## Платформенные различия

| Компонент | Android | iOS |
|-----------|---------|-----|
| HTTP Engine | OkHttp | Darwin |
| Entry Point | `MainActivity.kt` | `iOSApp.swift` |
| БД | Room + SQLite bundled | Room + SQLite bundled |
| File Storage | Android FS | iOS FS |
| Permissions | Grant library | Grant library |

## Принципы разработки

- Вся бизнес-логика — в `commonMain` (Kotlin Multiplatform)
- Платформо-зависимый код — только там, где API недоступен в KMP
- Reactive: данные через `Flow<T>`, обновление UI через `StateFlow`
- Offline-first: все данные в локальной БД, сеть — только для sync
- Транзакционность: комплексные операции с БД — через `runTransaction {}`
