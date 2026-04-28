# Сетевой слой и API-модели

## Обзор

Сетевой слой реализован на Ktor 3.4.0. Два HTTP-клиента: `LoginClient` (аутентификация) и `VolsClient` (все данные). Все модели сериализуются через Kotlinx Serialization.

Файлы:
- `core/data/network/LoginClient.kt`
- `core/data/network/VolsClient.kt`
- `core/data/network/HttpClientConfig.kt`
- `core/data/network/models/get/` — Response DTOs
- `core/data/network/models/post/` — Request DTOs (создание)
- `core/data/network/models/put/` — Request DTOs (обновление)

---

## 1. HTTP Клиенты

### LoginClient

Отвечает за аутентификационные операции.

```kotlin
class LoginClient(private val httpClient: HttpClient) {

    // Вход (получение токенов)
    suspend fun login(
        username: String,
        password: String,
        deviceId: String
    ): TokenPackage

    // Получение информации о текущем пользователе
    suspend fun getUserInfo(accessToken: String): UserInfo

    // Обновление access token
    suspend fun refreshToken(
        refreshToken: String,
        deviceId: String
    ): TokenPackage

    // Выход (инвалидация токена на сервере)
    suspend fun logout(refreshToken: String)

    // Смена пароля
    suspend fun changePassword(
        userId: Long,
        oldPassword: String,
        newPassword: String,
        accessToken: String
    ): Result<Unit>

    // Получить список непринятых соглашений
    suspend fun getRequiredAgreements(accessToken: String): List<AgreementJson>

    // Подписать соглашение
    suspend fun signAgreement(
        agreementId: Long,
        accessToken: String
    ): Result<Unit>
}
```

### VolsClient

Основной клиент для работы с данными ВОЛС. Использует токенную авторизацию (токен из `UserStorage`).

```kotlin
class VolsClient(private val httpClient: HttpClient) {

    // Справочники
    suspend fun getVolsTypes(since: Long?): List<VolsTypeJson>
    suspend fun getTasks(since: Long?): List<TaskJson>
    suspend fun getPowerLines(since: Long?): List<PowerLineJson>
    suspend fun getOperators(since: Long?): List<OperatorJson>
    suspend fun getPillars(since: Long?): List<PillarJson>
    suspend fun getTaskPillars(since: Long?): List<TaskPillarJson>

    // Инспекции
    suspend fun getInspections(taskPillarIds: List<Long>): List<PillarInspectionJson>
    suspend fun postPillarInspection(body: PillarInspectionPost): Long    // → serverInspectionId
    suspend fun putInspection(body: PillarInspectionPut)

    // ВОЛС
    suspend fun getVols(taskPillarIds: List<Long>): List<VolsJson>
    suspend fun postVols(bodies: List<VolsPost>): List<IdMapping>
    suspend fun putVols(bodies: List<VolsPutModel>)
    suspend fun deleteVols(serverIds: List<Long>)

    // Позиции ВОЛС
    suspend fun postVolsItems(bodies: List<VolsItemPost>): List<IdMapping>
    suspend fun putVolsItems(bodies: List<VolsItemPutModel>)
    suspend fun deleteVolsItems(serverIds: List<Long>)

    // Фотографии
    suspend fun sendPhotoFile(localPath: String): Long    // → serverPhotoId
    suspend fun postVolsPhoto(body: VolsPhotoPost): Long
    suspend fun deleteVolsPhotos(serverIds: List<Long>)
    suspend fun deletePillarPhoto(serverId: Long)
}
```

### HttpClientConfig

Конфигурация Ktor клиентов:

```kotlin
// Плагины:
install(ContentNegotiation) {
    json(Json { ignoreUnknownKeys = true })
}
install(Auth) {
    bearer {
        loadTokens { /* из UserStorage */ }
        refreshTokens { /* LoginClient.refreshToken() */ }
    }
}
install(Logging) {
    level = LogLevel.BODY  // в debug режиме
}
```

### Платформенные движки

| Платформа | Engine | Особенности |
|-----------|--------|------------|
| Android | OkHttp | HTTP/2, connection pooling |
| iOS | Darwin | NSURLSession |
| Fallback | CIO | Kotlin coroutines, только для тестов |

---

## 2. GET-модели (Server → Client)

### TokenPackage

```kotlin
@Serializable
data class TokenPackage(
    @SerialName("access_token")  val accessToken: String,
    @SerialName("refresh_token") val refreshToken: String,
    @SerialName("expires_in")    val expiresIn: Long,          // секунды
    @SerialName("refresh_expires_in") val refreshExpiresIn: Long,
    @SerialName("must_change_password") val mustChangePassword: Boolean
)
```

### UserInfo

```kotlin
@Serializable
data class UserInfo(
    val id: Long,
    val username: String,
    @SerialName("first_name")  val firstName: String,
    @SerialName("last_name")   val lastName: String,
    val email: String,
    @SerialName("must_change_password")  val mustChangePassword: Boolean,
    @SerialName("need_accept_agreement") val needAcceptAgreement: Boolean,
    val roles: List<UserRole>?,
    @SerialName("is_superuser") val isSuperuser: Boolean
)
```

### AgreementJson

```kotlin
@Serializable
data class AgreementJson(
    val id: Long,
    val title: String,
    val text: String,           // HTML-контент соглашения
    @SerialName("created_at") val createdAt: String
)
```

### PillarJson

```kotlin
@Serializable
data class PillarJson(
    val id: Long,
    val name: String,
    @SerialName("power_line_id") val powerLineId: Long,
    val longitude: Double,
    val latitude: Double
)
```

### TaskJson

```kotlin
@Serializable
data class TaskJson(
    val id: Long,
    val name: String,
    @SerialName("is_accepted") val isAccepted: Boolean?
)
```

### PowerLineJson

```kotlin
@Serializable
data class PowerLineJson(
    val id: Long,
    val name: String,
    @SerialName("task_id") val taskId: Long
)
```

### TaskPillarJson

```kotlin
@Serializable
data class TaskPillarJson(
    val id: Long,                          // taskPillarId
    @SerialName("pillar_id") val pillarId: Long,
    @SerialName("task_power_line_id") val taskPowerLineId: Long
)
```

### PillarInspectionJson

```kotlin
@Serializable
data class PillarInspectionJson(
    val id: Long,                                        // serverInspectionId
    @SerialName("mobile_id")   val mobileId: UUID,
    @SerialName("pillar_id")   val pillarId: Long,
    @SerialName("task_id")     val taskId: Long,
    @SerialName("photo_id")    val photoId: Long,
    @SerialName("mobile_created_at")  val mobileCreatedAt: LocalDateTime,
    @SerialName("mobile_updated_at")  val mobileUpdatedAt: LocalDateTime?,
    @SerialName("created_at")         val createdAt: LocalDateTime,
    @SerialName("updated_at")         val updatedAt: LocalDateTime?,
    @SerialName("is_accepted")        val isAccepted: Boolean?,
    @SerialName("denied_reason")      val deniedReason: String?
)
```

`isAccepted` определяет `pushStatus`:
- `null` → `WAIT_VERIFY`
- `true` → `ACCEPTED`
- `false` → `DENIED`

### VolsJson

```kotlin
@Serializable
data class VolsJson(
    val id: Long,
    @SerialName("mobile_id")     val mobileId: UUID,
    @SerialName("pillar_id")     val pillarId: Long,
    @SerialName("task_id")       val taskId: Long,
    @SerialName("operator_id")   val operatorId: Long?,
    val notes: String,
    @SerialName("date_placement") val datePlacement: LocalDate?,
    @SerialName("has_mark")      val hasMark: Boolean,
    @SerialName("mobile_created_at") val mobileCreatedAt: LocalDateTime,
    @SerialName("mobile_updated_at") val mobileUpdatedAt: LocalDateTime?,
    @SerialName("created_at")    val createdAt: LocalDateTime,
    @SerialName("updated_at")    val updatedAt: LocalDateTime?,
    @SerialName("is_success")    val isSuccess: Boolean?
)
```

### OperatorJson

```kotlin
@Serializable
data class OperatorJson(
    val id: Long,
    val name: String
)
```

### VolsTypeJson

```kotlin
@Serializable
data class VolsTypeJson(
    val id: Long,
    val name: String
)
```

---

## 3. POST-модели (Client → Server, создание)

### PillarInspectionPost

```kotlin
@Serializable
data class PillarInspectionPost(
    @SerialName("mobile_id")    val mobileId: String,      // UUID как String
    @SerialName("pillar_id")    val pillarId: Long,
    @SerialName("photo_id")     val photoId: Long,         // serverPhotoId
    @SerialName("task_id")      val taskId: Long,
    @SerialName("mobile_created_at") val mobileCreatedAt: LocalDateTime,
    @SerialName("mobile_updated_at") val mobileUpdatedAt: LocalDateTime?
)
```

### VolsPost

```kotlin
@Serializable
data class VolsPost(
    @SerialName("mobile_surrogate_uid") val mobileSurrogateUid: String,
    @SerialName("task_id")              val taskId: Long,
    @SerialName("pillar_id")            val pillarId: Long,
    @SerialName("vols_operator_id")     val volsOperatorId: Long?,
    val description: String,
    @SerialName("date_placement")       val datePlacement: LocalDate?,
    @SerialName("has_mark")             val hasMark: Boolean,
    @SerialName("mobile_created_at")    val mobileCreatedAt: LocalDateTime,
    @SerialName("mobile_updated_at")    val mobileUpdatedAt: LocalDateTime?
)
```

### VolsItemPost

```kotlin
@Serializable
data class VolsItemPost(
    @SerialName("mobile_surrogate_uid") val mobileSurrogateUid: UUID,
    @SerialName("vols_group_vols_id")   val volsGroupVolsId: Long,  // serverVolsId
    @SerialName("vols_type_vols_id")    val volsTypeVolsId: Long,
    val total: Int,
    @SerialName("mobile_created_at")    val mobileCreatedAt: LocalDateTime,
    @SerialName("mobile_updated_at")    val mobileUpdatedAt: LocalDateTime?
)
```

### VolsPhotoPost

```kotlin
@Serializable
data class VolsPhotoPost(
    @SerialName("mobile_surrogate_uid") val mobileSurrogateUid: String,
    @SerialName("vols_id")  val volsId: Long,    // serverVolsId
    @SerialName("photo_id") val photoId: Long,   // serverPhotoId
    @SerialName("mobile_created_at") val mobileCreatedAt: LocalDateTime
)
```

---

## 4. PUT-модели (Client → Server, обновление)

### PillarInspectionPut

```kotlin
@Serializable
data class PillarInspectionPut(
    val id: Long,                                          // serverInspectionId
    @SerialName("mobile_id")    val mobileId: String,
    @SerialName("photo_id")     val photoId: Long,
    @SerialName("mobile_created_at") val mobileCreatedAt: LocalDateTime,
    @SerialName("mobile_updated_at") val mobileUpdatedAt: LocalDateTime?
)
```

### VolsPutModel

```kotlin
@Serializable
data class VolsPutModel(
    val id: Long,                                          // serverVolsId
    @SerialName("mobile_surrogate_uid") val mobileSurrogateUid: String,
    @SerialName("vols_operator_id")     val volsOperatorId: Long?,
    val description: String,
    @SerialName("date_placement")       val datePlacement: LocalDate?,
    @SerialName("has_mark")             val hasMark: Boolean,
    @SerialName("mobile_created_at")    val mobileCreatedAt: LocalDateTime,
    @SerialName("mobile_updated_at")    val mobileUpdatedAt: LocalDateTime?
)
```

### VolsItemPutModel

```kotlin
@Serializable
data class VolsItemPutModel(
    val id: Long,                                           // serverItemId
    @SerialName("mobile_surrogate_uid") val mobileSurrogateUid: UUID,
    val total: Int,
    @SerialName("mobile_created_at")    val mobileCreatedAt: LocalDateTime,
    @SerialName("mobile_updated_at")    val mobileUpdatedAt: LocalDateTime?
)
```

---

## 5. IdMapping — ответ сервера на создание

При POST-запросах сервер возвращает маппинг мобильных UUID → серверных ID:

```kotlin
@Serializable
data class IdMapping(
    @SerialName("mobile_id") val mobileId: String,   // UUID с устройства
    @SerialName("server_id") val serverId: Long       // ID на сервере
)
```

Используется для сохранения `serverId` в локальную БД после успешной выгрузки.

---

## 6. Загрузка файлов (multipart)

Фотографии загружаются через multipart/form-data:

```kotlin
volsClient.sendPhotoFile(localPath: String): Long {
    httpClient.submitFormWithBinaryData(
        url = "$BASE_URL/photos/upload",
        formData = formData {
            append("file", File(localPath).readBytes(),
                   Headers.build {
                       append(HttpHeaders.ContentType, "image/jpeg")
                       append(HttpHeaders.ContentDisposition, "filename=photo.jpg")
                   })
        }
    )
    // Ответ: { "photo_id": 12345 }
}
```

---

## 7. Обработка ошибок

| HTTP код | Поведение |
|----------|----------|
| 200-299 | Успех |
| 401 | Auto-refresh токена через Auth плагин |
| 404 | Объект не найден — логировать |
| 500 | При удалении — считать успехом; иначе — ошибка |
| Таймаут | Ошибка сети — ретрай при следующей синхронизации |
| Нет соединения | `IOException` — предупредить пользователя |
