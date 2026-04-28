# Модуль аутентификации

## Обзор

Модуль охватывает три связанных процесса:
1. **Вход в систему** (Login) — первичная и повторная аутентификация
2. **Смена пароля** (ChangePassword) — принудительная смена при первом входе или истечении срока
3. **Принятие соглашения** (AcceptAgreement) — подписание пользовательского соглашения

Файлы:
- `screens/login/` — экран входа
- `screens/changePassword/` — экран смены пароля
- `screens/agreement/` — экран соглашения
- `core/domain/usecase/LoginUseCase.kt`
- `core/data/usecase/LoginToTokenUseCaseImpl.kt`
- `core/data/usecase/ChangePasswordUseCaseImpl.kt`
- `core/data/usecase/AcceptAgreementUseCaseImpl.kt`

---

## 1. Вход в систему (Login)

### Экран и ViewModel

**Файл**: `screens/login/LoginViewModel.kt`

#### Состояние UI (LoginState)

| Поле | Тип | Назначение |
|------|-----|-----------|
| `loginHolder` | String | Вводимый логин |
| `passwordHolder` | String | Вводимый пароль |
| `isLoading` | Boolean | Флаг загрузки (блокирует форму) |
| `isReLogin` | Boolean | Повторный вход (сессия истекла) |
| `passwordIsVisible` | Boolean | Видимость пароля |

#### События (LoginEvent)

| Событие | Описание |
|---------|---------|
| `Init(isReLogin)` | Инициализация экрана |
| `RequestLogin` | Нажатие кнопки «Войти» |
| `OnLoginInput(value)` | Изменение поля логина |
| `OnPasswordInput(value)` | Изменение поля пароля |
| `OnPasswordVisibleChanged` | Переключение видимости пароля |

#### Действия (LoginAction)

| Действие | Навигация |
|---------|-----------|
| `NavigateToMap` | → MainPages.Map |
| `NavigateToChangePassword(userId, oldPassword)` | → MainPages.ChangePassword |
| `NavigateToAcceptAgreement(userId)` | → MainPages.AcceptAgreement |
| `ShowError(message)` | Toast/Snackbar с ошибкой |

### Поток выполнения входа

```
Пользователь вводит логин + пароль
        │
        ▼
RequestLogin event
        │
        ▼
isLoading = true
        │
        ▼
LoginUseCase.login(login.trim(), password.trim(), isReLogin)
        │
        ├── isReLogin = true?
        │       ├── logout(oldRefreshToken)  — завершить старую сессию
        │       └── clearDeviceId()          — сброс Device ID
        │
        ▼
LoginClient.login(credentials) → TokenPackage {
    accessToken, refreshToken,
    expiresIn, refreshExpiresIn,
    mustChangePassword
}
        │
        ▼
LoginClient.getUserInfo(accessToken) → UserInfo {
    id, username, mustChangePassword,
    needAcceptAgreement, roles
}
        │
        ▼
UserStorage.saveTokenPackage(tokenPackage, timestamp)
UserStorage.saveUserInfo(userInfo)
        │
        ▼
        ├── userInfo.mustChangePassword == true
        │       └── → LoginResult.NeedChangePassword(userId, userInfo)
        │               → Action: NavigateToChangePassword
        │
        ├── userInfo.needAcceptAgreement == true
        │       └── → LoginResult.NeedAcceptAgreement(userId, userInfo)
        │               → Action: NavigateToAcceptAgreement
        │
        └── иначе → LoginResult.Success(userId, userInfo)
                        → Action: NavigateToMap
```

---

## 2. Смена пароля (ChangePassword)

### Экран и ViewModel

**Файл**: `screens/changePassword/ChangePasswordViewModel.kt`

Экран вызывается принудительно: когда сервер указывает `mustChangePassword = true`.

#### Состояние UI (ChangePasswordState)

| Поле | Тип | Назначение |
|------|-----|-----------|
| `password` | String | Новый пароль |
| `confirmPassword` | String | Подтверждение пароля |
| `isPasswordVisible` | Boolean | Видимость нового пароля |
| `isConfirmPasswordVisible` | Boolean | Видимость подтверждения |
| `isPasswordValid` | Boolean | Пароль соответствует требованиям |
| `doPasswordsMatch` | Boolean | Пароли совпадают |
| `canSubmit` | Boolean | Можно отправить форму |
| `isLoading` | Boolean | Флаг загрузки |

#### Требования к паролю

| Критерий | Правило |
|---------|---------|
| Длина | Минимум 12 символов |
| Верхний регистр | Минимум 1 символ `[A-Z]` |
| Нижний регистр | Минимум 1 символ `[a-z]` |
| Цифра | Минимум 1 цифра `[0-9]` |
| Спецсимвол | Минимум 1 из: `!@#$%^&*()_+-=[]{}|;:,.<>?/~` |

`canSubmit = isPasswordValid && doPasswordsMatch && !isLoading`

#### Поток смены пароля

```
Пользователь вводит новый пароль + подтверждение
        │
        ▼
Валидация в реальном времени (на каждый символ):
  - isPasswordValid  ← проверка всех 5 критериев
  - doPasswordsMatch ← новый == подтверждение
        │
        ▼
Нажатие «Сохранить» (canSubmit = true)
        │
        ▼
ChangePasswordUseCase.changePassword(
    userId,        ← из навигационного маршрута
    oldPassword,   ← из навигационного маршрута
    newPassword
) → Result<Unit>
        │
        ▼ На успех:
UserStorage: tokenPackage.mustChangePassword = false
UserStorage: userInfo.mustChangePassword = false
Пересоздать VolsClient с обновлёнными токенами
Очистить oldPassword из UserStorage
        │
        ▼
userInfo.needAcceptAgreement?
  ├── true  → NavigateToAcceptAgreement(userId)
  └── false → NavigateToMap
```

---

## 3. Принятие соглашения (AcceptAgreement)

### Экран и ViewModel

**Файл**: `screens/agreement/AcceptAgreementViewModel.kt`

Показывается когда сервер возвращает `needAcceptAgreement = true`.

#### Состояние UI (AcceptAgreementState)

| Поле | Тип | Назначение |
|------|-----|-----------|
| `agreementText` | String | Текст соглашения |
| `isAgreed` | Boolean | Пользователь принял |
| `isLoading` | Boolean | Загрузка текста соглашения |
| `isSubmitting` | Boolean | Отправка подписи |
| `hasScrolledToEnd` | Boolean | Прокрутил до конца |

**Правило**: кнопка «Принять» активна только если `hasScrolledToEnd = true`.

#### Поток принятия соглашения

```
Инициализация экрана
        │
        ▼
AcceptAgreementUseCase.getRequiredAgreements()
→ List<Agreement> (берётся первое из списка)
        │
        ▼
Отображение текста соглашения
Пользователь прокручивает до конца → hasScrolledToEnd = true
Кнопка «Принять» становится активной
        │
        ▼
[Принять]                    [Отклонить]
    │                              │
    ▼                              ▼
AcceptAgreementUseCase         loginClient.logout()
    .signAgreement(id)         UserStorage.clearAgreementInProgress()
    .refreshToken()            → NavigateToAuth
    .getUserInfo()
    │
    ▼
userInfo.needAcceptAgreement = false
UserStorage.saveUpdatedUserInfo()
    │
    ▼
→ NavigateToMap
```

#### Обработка прерывания процесса

В `UserStorage` хранится флаг `agreement_in_progress`:
- Устанавливается в `true` при начале процесса принятия соглашения
- Сбрасывается при успешном принятии или отклонении
- При старте приложения: если `agreement_in_progress = true` → возобновить процесс принятия

---

## 4. Use Cases аутентификации

### LoginUseCase / LoginToTokenUseCaseImpl

```kotlin
interface LoginUseCase {
    suspend fun login(
        login: String,
        password: String,
        isReLogin: Boolean
    ): LoginResult
}

sealed class LoginResult {
    data class Success(val userId: Long, val userInfo: UserInfo) : LoginResult()
    data class NeedChangePassword(val userId: Long, val userInfo: UserInfo) : LoginResult()
    data class NeedAcceptAgreement(val userId: Long, val userInfo: UserInfo) : LoginResult()
    object Error : LoginResult()
}
```

### ChangePasswordUseCase / ChangePasswordUseCaseImpl

```kotlin
interface ChangePasswordUseCase {
    suspend fun changePassword(
        userId: Long,
        oldPassword: String,
        newPassword: String
    ): Result<Unit>
}
```

### AcceptAgreementUseCase / AcceptAgreementUseCaseImpl

```kotlin
interface AcceptAgreementUseCase {
    suspend fun getRequiredAgreements(): List<Agreement>
    suspend fun signAgreement(agreementId: Long): Result<Unit>
    suspend fun refreshToken(): Result<Unit>
}
```

---

## 5. Диаграмма состояний аутентификации

```
              [App Start]
                   │
                   ▼
          checkForAppStart()
                   │
        ┌──────────┴──────────┐
        │                     │
   needReLogin?          токен валиден?
        │                     │
        ▼                     ▼
   Auth(isReLogin=true)  checkAgreements()
        │                     │
        │             ┌───────┴────────┐
        │         needAgr?         mustChangePwd?
        │             │                 │
        │             ▼                 ▼
        │     AcceptAgreement    ChangePassword
        │             │                 │
        └─────────────┴────────┬────────┘
                               │
                               ▼
                           Map (Главный экран)
```
