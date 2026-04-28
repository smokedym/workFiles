# Управление состоянием (BlocViewModel)

## Обзор

Приложение использует кастомный паттерн **BlocViewModel** — унифицированный способ управления состоянием экранов. Паттерн объединяет идеи BLoC (Business Logic Component) из Flutter и архитектурных компонентов Android (ViewModel + StateFlow).

**Файл**: `core/ui/bloc/BlocViewModel.kt`

---

## 1. Базовый класс BlocViewModel

```kotlin
abstract class BlocViewModel<STATE, EVENT, ACTION>(
    savedStateHandle: SavedStateHandle
) : ViewModel() {

    // Реактивное состояние — UI наблюдает через collectAsState()
    private val _state = MutableStateFlow(initialState())
    val state: StateFlow<STATE> = _state.asStateFlow()

    // Одноразовые эффекты — навигация, toast, диалоги
    private val _actions = MutableSharedFlow<ACTION>(
        extraBufferCapacity = 10,
        onBufferOverflow = DROP_OLDEST
    )
    val actions: SharedFlow<ACTION> = _actions.asSharedFlow()

    // Точка входа для всех пользовательских действий
    abstract fun obtainEvent(event: EVENT)

    // Защищённые методы для ViewModel-наследников:
    protected fun setState(reducer: STATE.() -> STATE) {
        _state.value = _state.value.reducer()
    }

    protected fun sendAction(action: ACTION) {
        viewModelScope.launch { _actions.emit(action) }
    }

    protected fun runInIOScope(block: suspend CoroutineScope.() -> Unit) {
        viewModelScope.launch(Dispatchers.IO) { block() }
    }

    protected fun runInMainScope(block: suspend CoroutineScope.() -> Unit) {
        viewModelScope.launch(Dispatchers.Main) { block() }
    }

    // Начальное состояние — реализуется в наследнике
    protected abstract fun initialState(): STATE
}
```

---

## 2. Три типа данных на каждый экран

### State — состояние UI

`data class` с полным снимком состояния экрана. UI рендерится на основе State.

```kotlin
data class LoginState(
    val loginHolder: String = "",
    val passwordHolder: String = "",
    val isLoading: Boolean = false,
    val isReLogin: Boolean = false,
    val passwordIsVisible: Boolean = false
)
```

**Правила:**
- Всегда `data class` с дефолтными значениями
- Иммутабельный — обновляется через `copy()`
- Содержит ВСЁ необходимое для рендеринга UI

### Event — пользовательские события

`sealed class` всех возможных действий пользователя на экране.

```kotlin
sealed class LoginEvent {
    data class Init(val isReLogin: Boolean) : LoginEvent()
    object RequestLogin : LoginEvent()
    data class OnLoginInput(val value: String) : LoginEvent()
    data class OnPasswordInput(val value: String) : LoginEvent()
    object OnPasswordVisibleChanged : LoginEvent()
}
```

**Правила:**
- Только факты действий, без логики
- Параметры несут данные (текст поля, ID объекта)
- `object` для событий без параметров

### Action — побочные эффекты

`sealed class` для одноразовых эффектов (навигация, toast, диалоги).

```kotlin
sealed class LoginAction {
    object NavigateToMap : LoginAction()
    data class NavigateToChangePassword(
        val userId: Long,
        val oldPassword: String
    ) : LoginAction()
    data class NavigateToAcceptAgreement(val userId: Long) : LoginAction()
    data class ShowError(val message: String) : LoginAction()
}
```

**Отличие от State:**
- `SharedFlow` (не `StateFlow`) — события не повторяются при пересоздании Composable
- Одноразовые: навигация срабатывает ровно один раз

---

## 3. Паттерн использования в Composable

```kotlin
@Composable
fun LoginScreen(
    viewModel: LoginViewModel = viewModel(),
    navController: NavController
) {
    // Наблюдение за состоянием (обновляет UI при изменении)
    val state by viewModel.state.collectAsStateWithLifecycle()

    // Обработка одноразовых эффектов
    LaunchedEffect(Unit) {
        viewModel.actions.collect { action ->
            when (action) {
                is LoginAction.NavigateToMap ->
                    navController.navigate(MainPages.Map)
                is LoginAction.NavigateToChangePassword ->
                    navController.navigate(
                        MainPages.ChangePassword(action.userId, action.oldPassword)
                    )
                is LoginAction.ShowError ->
                    snackbarHostState.showSnackbar(action.message)
            }
        }
    }

    // Инициализация при первом показе
    LaunchedEffect(Unit) {
        viewModel.obtainEvent(LoginEvent.Init(isReLogin))
    }

    // UI на основе состояния
    Column {
        TextField(
            value = state.loginHolder,
            onValueChange = { viewModel.obtainEvent(LoginEvent.OnLoginInput(it)) }
        )
        Button(
            onClick = { viewModel.obtainEvent(LoginEvent.RequestLogin) },
            enabled = !state.isLoading
        ) { Text("Войти") }
    }
}
```

---

## 4. Паттерн реализации ViewModel

```kotlin
class LoginViewModel(
    savedStateHandle: SavedStateHandle
) : BlocViewModel<LoginState, LoginEvent, LoginAction>(savedStateHandle) {

    private val loginUseCase = VolsSingleton.loginUseCase

    override fun initialState() = LoginState()

    override fun obtainEvent(event: LoginEvent) {
        when (event) {
            is LoginEvent.Init ->
                setState { copy(isReLogin = event.isReLogin) }

            is LoginEvent.OnLoginInput ->
                setState { copy(loginHolder = event.value) }

            LoginEvent.RequestLogin ->
                runInIOScope { performLogin() }

            LoginEvent.OnPasswordVisibleChanged ->
                setState { copy(passwordIsVisible = !passwordIsVisible) }
        }
    }

    private suspend fun performLogin() {
        setState { copy(isLoading = true) }

        val result = loginUseCase.login(
            login = state.value.loginHolder.trim(),
            password = state.value.passwordHolder.trim(),
            isReLogin = state.value.isReLogin
        )

        setState { copy(isLoading = false) }

        when (result) {
            is LoginResult.Success ->
                sendAction(LoginAction.NavigateToMap)
            is LoginResult.NeedChangePassword ->
                sendAction(LoginAction.NavigateToChangePassword(
                    result.userId, state.value.passwordHolder
                ))
            is LoginResult.Error ->
                sendAction(LoginAction.ShowError("Ошибка входа"))
        }
    }
}
```

---

## 5. Работа с корутинами

### runInIOScope — для фоновых операций

```kotlin
runInIOScope {
    // Здесь: сетевые запросы, операции с БД
    val data = repository.getItems()
    // Переключение на Main для обновления состояния
    runInMainScope {
        setState { copy(items = data) }
    }
}
```

### runInMainScope — для UI-операций

```kotlin
runInMainScope {
    // Здесь: обновление состояния, отправка actions
    setState { copy(isLoading = false) }
    sendAction(MyAction.ShowDialog)
}
```

### Реактивные Flow в ViewModel

```kotlin
override fun obtainEvent(event: MyEvent) {
    when (event) {
        MyEvent.Init -> runInIOScope {
            // Flow автоматически обновляет state при изменении БД
            repository.getItems().collect { items ->
                runInMainScope {
                    setState { copy(items = items) }
                }
            }
        }
    }
}
```

---

## 6. SavedStateHandle

`SavedStateHandle` передаётся в `BlocViewModel` для восстановления состояния после смерти процесса (process death). Навигационные параметры также читаются через него:

```kotlin
class PillarInfoViewModel(
    savedStateHandle: SavedStateHandle
) : BlocViewModel<...>(savedStateHandle) {

    // Навигационный параметр из маршрута
    private val taskPillarId: Long =
        savedStateHandle["taskPillarId"] ?: 0L
}
```

---

## 7. Преимущества паттерна

| Характеристика | Описание |
|---------------|---------|
| **Однонаправленный поток** | Event → ViewModel → State/Action → UI |
| **Предсказуемость** | Состояние изменяется только через `setState {}` |
| **Тестируемость** | ViewModel тестируется без UI |
| **Реактивность** | StateFlow обновляет UI автоматически |
| **Разделение ответственности** | UI не знает о бизнес-логике |
| **Process death** | SavedStateHandle сохраняет состояние |
