# Инспекция опоры (Pillar Inspection)

## Обзор

Центральный бизнес-процесс приложения. Инспектор осматривает опору ВОЛС, фиксирует её состояние фотографией и добавляет сведения о размещённых объектах ВОЛС. Результат отправляется на сервер для проверки.

Файлы:
- `screens/details/pillarInfo/PillarInfoViewModel.kt`
- `screens/details/pillarInfo/PillarInfoScreen.kt`
- `screens/details/previewPhoto/PreviewPhotoViewModel.kt`
- `core/data/repository/PillarInspectionRepositoryImpl.kt`
- `core/domain/repository/PillarInspectionRepository.kt`

---

## 1. Состояние PillarInfoState

| Поле | Тип | Назначение |
|------|-----|-----------|
| `info` | `PillarInfoItem?` | Данные опоры + текущая инспекция |
| `showCameraPicker` | Boolean | Показать выбор источника фото |
| `pushStatus` | `PillarInspectionPushStatus` | Статус выгрузки |
| `workStatus` | `PillarInspectionWorkStatus` | Статус выполнения |
| `pagerInitialPage` | Int | Начальная страница (0=данные, 1=замечания) |

**Вычисляемые свойства:**
```kotlin
val isEditable: Boolean
    get() = pushStatus in listOf(NOT_PUSHED, WAIT_PUSH, DENIED)

val saveButtonText: String
    get() = when (pushStatus) {
        DENIED -> "Отправить повторно"
        else   -> "Завершить осмотр"
    }

val isSaveEnabled: Boolean
    get() = info?.inspection?.image != null  // фото обязательно
```

---

## 2. События PillarInfoEvent

| Событие | Описание |
|---------|---------|
| `Init(taskPillarId)` | Загрузить данные опоры |
| `ClickAddPhoto` | Открыть выбор источника фото |
| `AddPhoto(PhotoResult)` | Сохранить сделанное фото |
| `DeletePhoto` | Удалить фото инспекции |
| `ClickAddVols` | Перейти к добавлению ВОЛС |
| `OpenVolsEditor(volsGroup)` | Редактировать существующий ВОЛС |
| `DeleteVols(volsGroup)` | Удалить группу ВОЛС |
| `FinishPillarView` | Завершить осмотр → поставить в очередь на выгрузку |
| `CloseWithoutSave` | Закрыть без изменений |

---

## 3. Загрузка данных опоры

```
Init(taskPillarId)
    │
    ▼
pillarRepository.getPillarInfo(taskPillarId) → Flow<PillarInfoItem>
    │
    ▼ Реактивное обновление при каждом изменении в БД:
state.info = PillarInfoItem(
    pillar = PillarItem(name, coordinates, powerLineName),
    inspection = PillarInspection?(
        id, workStatus, pushStatus,
        deniedReason?,
        image: PhotoCaptureItem?,
        volsGroups: List<VolsGroup>,
        volsStats: List<VolsTypeStats>
    )
)
    │
    ▼
pagerInitialPage = if (deniedReason != null) 1 else 0
    (при отклонении — сразу показать вкладку с замечанием)
```

---

## 4. Фотофиксация опоры

### Добавление фото

```
ClickAddPhoto
    │
    ▼
showCameraPicker = true
(показывается Bottom Sheet: Камера / Галерея)
    │
    ▼
Пользователь делает/выбирает фото
    │
    ▼
AddPhoto(PhotoResult { path, ... })
    │
    ▼
isLoading = true
    │
    ▼
PillarInspectionRepository.setPillarPhoto(taskPillarId, tempPath)
    │
    ├── [1] Geolocator.mobile().getLocation(HighAccuracy)
    │           → latitude, longitude (GPS)
    │
    ├── [2] captureRepository.addCaptureForPillar(
    │           path = tempPath,
    │           latitude, longitude,
    │           captureDate = LocalDateTime.now()
    │       ) → photoId: UUID
    │
    ├── [3] Получить или создать PillarInspectionEntity:
    │       getOrCreate by taskPillarId
    │
    ├── [4] inspection.photoId = photoId
    │       inspection.updatedAt = now()
    │
    ├── [5] recalculateWorkStatus(inspection)
    │
    └── [6] appDatabase.runTransaction { ... }
              (всё атомарно)
    │
    ▼
isLoading = false
state обновляется автоматически через Flow
```

### Удаление фото

```
DeletePhoto
    │
    ▼
pillarInspectionRepository.removePillarPhoto(inspectionId)
    │
    ├── [1] Добавить в DeletePushEntity если serverId != null
    │       (сервер нужно уведомить об удалении)
    │
    ├── [2] PhotoCaptureEntity.delete(photoId)
    │
    ├── [3] inspection.photoId = null
    │
    └── [4] recalculateWorkStatus(inspection)
```

---

## 5. Работа с ВОЛС на экране опоры

### Просмотр сводки ВОЛС

В `PillarInfoState.info.inspection` содержится:

```kotlin
data class PillarInspection(
    ...
    val volsGroups: List<VolsGroup>,   // группы ВОЛС (по оператору)
    val volsStats: List<VolsTypeStats> // итоговая сводка по типам
)

data class VolsGroup(
    val volsId: UUID,
    val name: String?   // имя оператора или null
)

data class VolsTypeStats(
    val typeId: Long,
    val typeName: String,
    val totalCount: Int  // суммарное количество по всем группам
)
```

### Переход к добавлению/редактированию ВОЛС

```
ClickAddVols
    │
    └─► NavigateToAddVols(taskPillarId, volsId=null)
         (создание нового ВОЛС)

OpenVolsEditor(volsGroup)
    │
    └─► NavigateToAddVols(taskPillarId, volsId=volsGroup.volsId)
         (редактирование существующего)
```

### Удаление группы ВОЛС

```
DeleteVols(volsGroup)
    │
    ▼
pillarInspectionRepository.deleteVolsGroup(inspectionId, volsId)
    │
    ├── [1] VolsEntity.delete(volsId)
    ├── [2] VolsItemEntity.deleteByParentId(volsId)
    ├── [3] VolsCaptureEntity.deleteByVolsId(volsId)
    ├── [4] Добавить в DeletePushEntity если serverId != null
    └── [5] recalculateWorkStatus(inspection)
```

---

## 6. Завершение осмотра

```
FinishPillarView
    │
    ▼
Проверка: inspection.image != null?
  └── Нет → isSaveEnabled = false (кнопка неактивна, не выполняется)
    │
    ▼
pillarInspectionRepository.setPillarToPush(inspectionId)
    │
    ▼
inspection.pushStatus = WAIT_PUSH
inspection.updatedAt = now()
    │
    ▼
→ NavigateBack
```

После установки `WAIT_PUSH` инспекция попадает в очередь Push-менеджера и будет загружена на сервер при следующей синхронизации.

---

## 7. Статусы инспекции

### WorkStatus — статус выполнения

| Значение | Условие | Описание |
|---------|---------|---------|
| `NOT_STARTED` | нет фото, нет ВОЛС | Не начата |
| `WITHOUT_CAPTURE` | нет фото, есть ВОЛС | Нет фотофиксации |
| `WITHOUT_VOLS` | есть фото, нет ВОЛС | Нет объектов ВОЛС |
| `WITH_VOLS` | есть фото + ВОЛС | Полностью заполнена |

### PushStatus — статус выгрузки

| Значение | Условие | Редактируемость |
|---------|---------|----------------|
| `NOT_PUSHED` | Создана локально | ✅ Редактируется |
| `WAIT_PUSH` | Помечена к выгрузке | ✅ Редактируется |
| `PUSH_NOT_COMPLETE` | Частичная выгрузка | ❌ Только чтение |
| `WAIT_VERIFY` | Выгружена, ожидает проверки | ❌ Только чтение |
| `ACCEPTED` | Принята сервером | ❌ Только чтение |
| `DENIED` | Отклонена сервером | ✅ Редактируется (повторная отправка) |

### Автоматический пересчёт WorkStatus

```kotlin
fun recalculateWorkStatus(inspection: PillarInspectionEntity) {
    val hasPhoto = inspection.photoId != null
    val hasVols  = volsDao.countByInspectionId(inspection.id) > 0

    inspection.workStatus = when {
        hasPhoto && hasVols  -> WITH_VOLS
        hasPhoto && !hasVols -> WITHOUT_VOLS
        !hasPhoto && hasVols -> WITHOUT_CAPTURE
        else                 -> NOT_STARTED
    }
}
```

Вызывается после каждой операции: добавление/удаление фото, добавление/удаление ВОЛС.

---

## 8. Предпросмотр фото (PreviewPhoto)

**Файл**: `screens/details/previewPhoto/PreviewPhotoViewModel.kt`

Простой экран для полноэкранного просмотра сделанного фото.

```kotlin
data class PreviewPhotoState(
    val photoPath: String,
    val latitude: Double,
    val longitude: Double,
    val captureDate: LocalDateTime
)
```

Навигация: `NavigateToPreviewPhoto(photoPath, latitude, longitude, captureDate)`

---

## 9. PillarInspectionRepository — ключевые операции

**Файл**: `core/data/repository/PillarInspectionRepositoryImpl.kt`

```kotlin
interface PillarInspectionRepository {
    // Установить фото инспекции
    suspend fun setPillarPhoto(taskPillarId: Long, tempPath: String)

    // Удалить фото инспекции
    suspend fun removePillarPhoto(inspectionId: UUID)

    // Удалить группу ВОЛС
    suspend fun deleteVolsGroup(inspectionId: UUID, volsId: UUID)

    // Поставить в очередь на выгрузку
    suspend fun setPillarToPush(inspectionId: UUID)

    // Добавить новую группу ВОЛС
    suspend fun addVols(vols: Vols, taskPillarId: Long)

    // Обновить группу ВОЛС
    suspend fun updateVols(newVols: Vols, oldVols: Vols)

    // Получить текущую инспекцию (nullable)
    suspend fun getPillarInspection(taskPillarId: Long): PillarInspectionEntity?

    // Пересчитать WorkStatus после синхронизации с сервера
    suspend fun recalculateWorkStatusForInspectionsFromSync(inspections: List<UUID>)
}
```
