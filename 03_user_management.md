# Управление пользователем и токенами

## Обзор

Модуль отвечает за:
- Проверку состояния при старте приложения
- Жизненный цикл токенов (хранение, проверка истечения, обновление)
- Хранение пользовательских данных в защищённом хранилище

Файлы:
- `core/data/manager/UserManager.kt` — оркестратор стартовой проверки
- `core/data/storage/UserStorage.kt` — защищённое key-value хранилище (KVault)

---

## 1. UserManager

**Файл**: `core/data/manager/UserManager.kt`

Основной компонент, определяющий с какого экрана запустить приложение.

### checkForAppStart(): StartCheckResult

Вызывается в `App.kt` при каждом запуске приложения.

```
checkForAppStart()
        │
        ▼ [1] Проверка offline-режима (FLAVOR)
        │       offline-mode? → StartCheckResult.MAIN
        │
        ▼ [2] needReLogin?
        │       true → StartCheckResult.LOGIN (isReLogin=false)
        │
        ▼ [3] Авторизован ли пользователь?
        │       нет → StartCheckResult.LOGIN (isReLogin=false)
        │
        ▼ [4] Смена пароля в процессе?
        │       true → StartCheckResult.RE_LOGIN
        │              (нужно переавторизоваться для продолжения смены пароля)
        │
        ▼ [5] Принятие соглашения в процессе?
        │       true → StartCheckResult.AGREEMENT(userId)
        │
        ▼ [6] Access token истёк?
        │       true → refreshToken(refreshToken, deviceId)
        │               │
        │               ├── Успех → сохранить новые токены
        │               ├── Сетевая ошибка → StartCheckResult.MAIN (офлайн)
        │               └── Другая ошибка → StartCheckResult.LOGIN
        │
        ▼ [7] Есть непринятые соглашения?
        │       true → StartCheckResult.AGREEMENT(userId)
        │
        ▼ [8] Нужна смена пароля?
        │       true → StartCheckResult.CHANGE_PASSWORD(userId, token)
        │
        └──────► StartCheckResult.MAIN
```

### StartCheckResult (перечисление)

| Результат | Действие |
|-----------|---------|
| `LOGIN` | Открыть экран входа (isReLogin=false) |
| `RE_LOGIN` | Открыть экран входа (isReLogin=true) |
| `CHANGE_PASSWORD(userId, oldToken)` | Открыть экран смены пароля |
| `AGREEMENT(userId)` | Открыть экран соглашения |
| `MAIN` | Открыть главный экран карты |

---

## 2. UserStorage

**Файл**: `core/data/storage/UserStorage.kt`  
**Технология**: KVault 1.12.0 (платформо-независимое защищённое хранилище)

### Хранимые данные

| Ключ | Тип значения | Содержимое |
|------|------------|-----------|
| `TOKEN_RAW_KEY` | JSON String | Сериализованный `TokenPackage` |
| `TOKEN_RECEIVE_TIME_KEY` | Long (ms) | Временная метка получения токена |
| `LOGIN_KEY` | String | Текущий логин пользователя |
| `DEVICE_ID_KEY` | String | Уникальный ID устройства |
| `USER_ID_KEY` | Long | ID пользователя на сервере |
| `USER_INFO_KEY` | JSON String | Сериализованный `UserInfo` |
| `NEEDS_RELOGIN_KEY` | Boolean | Флаг необходимости повторного входа |
| `PASSWORD_CHANGE_IN_PROGRESS_KEY` | Boolean | Флаг процесса смены пароля |
| `AGREEMENT_IN_PROGRESS_KEY` | Boolean | Флаг процесса принятия соглашения |
| `LAST_SYNC_KEY` | Long (ms) | Время последней синхронизации |
| `OLD_PASSWORD_KEY` | String | Старый пароль (временно, для смены) |

### TokenPackage

```kotlin
@Serializable
data class TokenPackage(
    val accessToken: String,
    val refreshToken: String,
    val expiresIn: Long,        // секунды до истечения access token
    val refreshExpiresIn: Long, // секунды до истечения refresh token
    val mustChangePassword: Boolean
)
```

### Логика проверки истечения токенов

```kotlin
val tokenReceiveTime: Long = storage.getTokenReceiveTime()
val now: Long = currentTimeMillis()

val accessTokenExpired: Boolean =
    (tokenReceiveTime + expiresIn * 1000) <= now

val refreshTokenExpired: Boolean =
    (tokenReceiveTime + refreshExpiresIn * 1000) <= now
```

**Важно**: `expiresIn` хранится в секундах, время в миллисекундах — поэтому умножение на 1000.

### UserInfo

```kotlin
@Serializable
data class UserInfo(
    val id: Long,
    val username: String,
    val firstName: String,
    val lastName: String,
    val email: String,
    val mustChangePassword: Boolean,
    val needAcceptAgreement: Boolean,
    val roles: List<UserRole>?,
    val isSuperuser: Boolean
)
```

---

## 3. Жизненный цикл токенов

### Получение токенов (при входе)

```
loginClient.login(login, password, deviceId)
    │
    ▼
TokenPackage → UserStorage.saveTokenPackage(pkg, currentTimeMillis())
```

### Обновление токенов (refresh)

```
UserStorage.getRefreshToken()
UserStorage.getDeviceId()
    │
    ▼
loginClient.refreshToken(refreshToken, deviceId)
    │
    ▼
новый TokenPackage → UserStorage.saveTokenPackage(pkg, currentTimeMillis())
```

### Обновление токенов после смены пароля

При смене пароля сервер инвалидирует все токены. Необходимо:
1. Пересоздать `VolsClient` с новыми учётными данными
2. Обновить `mustChangePassword = false` в `UserStorage`
3. Получить обновлённый `UserInfo`

### Обновление токенов после принятия соглашения

После подписи соглашения:
1. `AcceptAgreementUseCase.refreshToken()` — принудительное обновление
2. `loginClient.getUserInfo()` — получить актуальный `UserInfo`
3. Проверить `needAcceptAgreement` — должно стать `false`

---

## 4. Device ID

**Назначение**: Уникальный идентификатор устройства, передаётся при входе и обновлении токенов для серверной идентификации сессии.

**Провайдер**: `DeviceIdProvider` (платформо-зависимая реализация)

**Жизненный цикл**:
- Инициализируется при первом запуске приложения
- Хранится в `UserStorage` (KVault)
- Очищается при повторном входе (`isReLogin = true`) — для сброса серверной сессии
- Заново генерируется при следующем входе

---

## 5. Связь компонентов

```
App.kt
  │
  └── UserManager.checkForAppStart()
        │
        ├── UserStorage (read: токены, флаги, userInfo)
        ├── LoginClient (refresh token если нужно)
        └── UserStorage (write: обновлённые токены)
              │
              ▼
        StartCheckResult → NavController.navigate(route)
```
