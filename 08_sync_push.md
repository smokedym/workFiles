# Синхронизация и выгрузка данных (Sync & Push)

## Обзор

Система синхронизации обеспечивает двустороннее обновление данных между устройством и сервером. Реализована стратегия **offline-first**: устройство работает автономно, синхронизация выполняется вручную.

Файлы:
- `core/data/manager/sync/SyncManager.kt` — оркестратор синхронизации
- `core/data/manager/sync/RemoteSyncManager.kt` — загрузка данных с сервера
- `core/data/manager/sync/TemplateSyncManager.kt` — заполнение тестовыми данными
- `core/data/manager/sync/remote/executors/` — исполнители по типам сущностей
- `core/data/manager/push/PillarInspectionPushExecutor.kt` — выгрузка инспекций

---

## 1. Архитектура синхронизации

```
Пользователь нажимает "Синхронизировать"
           │
           ▼
    SyncManager.sync()
    ┌──────────────────────────────┐
    │  [1] PushManager.push()      │  ← Сначала выгрузить локальные данные
    │  [2] RemoteSyncManager.sync()│  ← Затем загрузить с сервера
    └──────────────────────────────┘
```

**Принцип**: Push всегда предшествует Sync, чтобы локальные данные не были перезаписаны серверными до их выгрузки.

---

## 2. Push — выгрузка инспекций на сервер

**Файл**: `core/data/manager/push/PillarInspectionPushExecutor.kt`

### Что выгружается

Выгружаются только инспекции со статусом `pushStatus = WAIT_PUSH`.

### Полная последовательность выгрузки одной инспекции

```
Для каждой инспекции со статусом WAIT_PUSH:
│
├── [ШАГ 1] Загрузить фото инспекции на сервер
│   ├── Если photo.serverId == null:
│   │     volsClient.sendPhotoFile(localPath) → serverPhotoId
│   │     pushDao.setServerIdForPillarCapture(photoId, serverPhotoId)
│   └── Иначе: пропустить (уже загружено)
│
├── [ШАГ 2] Создать/обновить запись инспекции
│   ├── Если inspection.serverId == null (новая):
│   │     volsClient.postPillarInspection(PillarInspectionPost {
│   │         mobileId: UUID,
│   │         pillarId, taskId,
│   │         photoId: serverPhotoId,
│   │         mobileCreatedAt, mobileUpdatedAt
│   │     }) → serverInspectionId
│   │     pushDao.setServerDataForNewInspection(inspectionId, serverInspectionId)
│   │
│   └── Если inspection.serverId != null (обновление):
│         volsClient.putInspection(PillarInspectionPut {
│             id: serverInspectionId,
│             mobileId, photoId,
│             mobileCreatedAt, mobileUpdatedAt
│         })
│
├── [ШАГ 3] Удалить помеченные объекты (DeletePushEntity)
│   ├── Для VOLS → volsClient.deleteVols([serverIds])
│   ├── Для VOLS_ITEMS → volsClient.deleteVolsItems([serverIds])
│   ├── Для VOLS_CAPTURE → volsClient.deleteVolsPhotos([serverIds])
│   ├── Для PILLAR_CAPTURE → volsClient.deletePillarPhoto(serverId)
│   └── После успеха → DeletePushEntity.delete()
│       (500 от сервера = принято, удаляем из очереди)
│
├── [ШАГ 4] Загрузить/обновить группы ВОЛС
│   ├── Разделить на новые (serverId == null) и обновлённые:
│   │     новые:      volsClient.postVols(VolsPost[]) → serverIds
│   │     обновлённые: volsClient.putVols(VolsPutModel[])
│   └── pushDao.setServerIdsForVols(mapping)
│
├── [ШАГ 5] Загрузить/обновить позиции ВОЛС (VolsItem)
│   ├── Разделить аналогично на новые и обновлённые
│   │     новые:      volsClient.postVolsItems(VolsItemPost[])
│   │     обновлённые: volsClient.putVolsItems(VolsItemPutModel[])
│   └── pushDao.setServerIdsForVolsItems(mapping)
│
├── [ШАГ 6] Загрузить фотографии ВОЛС
│   ├── Для каждого фото без serverId:
│   │     volsClient.sendPhotoFile(localPath) → serverPhotoId
│   │     volsClient.postVolsPhoto(VolsPhotoPost {
│   │         mobileSurrogateUid, volsId: serverVolsId,
│   │         photoId: serverPhotoId, mobileCreatedAt
│   │     })
│   └── pushDao.setServerIdForVolsCapture(captureId, serverPhotoId)
│
└── [ШАГ 7] Обновить статус инспекции
      pushDao.setPushStatusForInspection(inspectionId, WAIT_VERIFY)
```

### Логика разделения объектов на новые/обновлённые

```kotlin
fun groupedObjectsByPushTime(
    objects: List<T>,
    getServerUpdatedAt: (T) -> Long?,
    getUpdatedAt: (T) -> Long
): Pair<List<T>, List<T>> {
    val created = objects.filter { getServerUpdatedAt(it) == null }
    val updated = objects.filter { obj ->
        val serverTime = getServerUpdatedAt(obj)
        serverTime != null && serverTime <= getUpdatedAt(obj)
    }
    return Pair(created, updated)
}
```

| Условие | Категория |
|---------|-----------|
| `serverUpdatedAt == null` | Новый — POST |
| `serverUpdatedAt <= localUpdatedAt` | Обновлённый — PUT |
| `serverUpdatedAt > localUpdatedAt` | Не изменялся — пропустить |

### Обработка ошибок при Push

| Ситуация | Поведение |
|----------|----------|
| HTTP 500 при удалении | Считается успехом (сервер уже не видит объект) |
| HTTP 4xx / сетевая ошибка | Объект остаётся в очереди |
| Частичный успех | `pushStatus = PUSH_NOT_COMPLETE` |
| Полный успех | `pushStatus = WAIT_VERIFY` |

---

## 3. Sync — загрузка данных с сервера

**Файл**: `core/data/manager/sync/RemoteSyncManager.kt`

### Фазы синхронизации

```
RemoteSyncManager.sync()
│
├── [ФАЗА 1] Параллельная загрузка справочников:
│   ┌─────────────────────────────────────────────┐
│   │ async { volsTypeExecutor.sync()  }           │
│   │ async { taskExecutor.sync()      }           │
│   │ async { powerLineExecutor.sync() }           │
│   │ async { operatorsExecutor.sync() }           │
│   └─────────────────────────────────────────────┘
│              awaitAll()
│
├── [ФАЗА 2] Структурные данные (последовательно):
│   pillarsExecutor.sync()
│   (зависит от powerLine данных из фазы 1)
│
├── [ФАЗА 3] Привязки задач:
│   taskPillarsExecutor.sync()
│   (зависит от tasks + pillars из предыдущих фаз)
│   │
│   └── Получить все taskPillarIds из БД
│
└── [ФАЗА 4] Содержательные данные:
    contentExecutor.sync(taskPillarKeys)
    → inspections, vols, vols items, photos
    → newInspectionsIds: List<UUID>
    │
    ▼
    pillarInspectionRepository
        .recalculateWorkStatusForInspectionsFromSync(newInspectionsIds)
    │
    ▼
    userStorage.setLastSyncRaw(newSyncTime)
```

### Executors (исполнители синхронизации)

Расположение: `core/data/manager/sync/remote/executors/`

| Executor | Синхронизируемые данные |
|----------|------------------------|
| `VolsTypeExecutor` | Типы ВОЛС (`VolsTypeEntity`) |
| `TaskExecutor` | Задания (`TaskEntity`) |
| `PowerLineExecutor` | Линии ЭП (`PowerLineEntity`, `TaskPowerLineEntity`) |
| `OperatorsExecutor` | Операторы/провайдеры (`OperatorEntity`) |
| `PillarsExecutor` | Опоры (`PillarEntity`) |
| `TaskPillarsExecutor` | Связи задание↔опора (`TaskPillarEntity`) |
| `ContentExecutor` | Инспекции, ВОЛС, фото (`PillarInspectionEntity`, `VolsEntity`, etc.) |

### Стратегия синхронизации каждого executor

```
Executor.sync()
    │
    ▼
volsClient.get[EntityType](since=lastSyncTime)
    → List<EntityJson>
    │
    ▼
Для каждого объекта:
    ├── Существует в БД? → UPDATE
    └── Не существует?  → INSERT
    │
    ▼
Удалённые на сервере объекты:
    (если сервер поддерживает) → DELETE из БД
```

### recalculateWorkStatusForInspectionsFromSync

После загрузки данных с сервера статусы `workStatus` пересчитываются:

```kotlin
suspend fun recalculateWorkStatusForInspectionsFromSync(
    inspections: List<UUID>
) {
    inspections.forEach { inspectionId ->
        val hasPhoto = captureDao.getByInspectionId(inspectionId) != null
        val hasVols  = volsDao.countByInspectionId(inspectionId) > 0
        val newStatus = when {
            hasPhoto && hasVols  -> WITH_VOLS
            hasPhoto && !hasVols -> WITHOUT_VOLS
            !hasPhoto && hasVols -> WITHOUT_CAPTURE
            else                 -> NOT_STARTED
        }
        inspectionDao.updateWorkStatus(inspectionId, newStatus)
    }
}
```

---

## 4. TemplateSyncManager

**Файл**: `core/data/manager/sync/TemplateSyncManager.kt`

Используется для заполнения БД тестовыми данными (демо-режим / разработка).

### Тестовые данные

```
Операторы: МТС, Мегафон, Ростелеком
Задания: Задание 1, Задание 2
Типы ВОЛС: 6 типов (сплиттеры, кронштейны, муфты и т.д.)
Линии ЭП: 8 тестовых линий
Опоры: 32 тестовые опоры с реальными GPS-координатами
Привязки: TaskPowerLine, TaskPillar
```

Данные вставляются только если БД пуста (UPSERT-стратегия).

---

## 5. DeletePushEntity — очередь удалений

При локальном удалении объекта (ВОЛС, фото и т.д.) его нельзя сразу удалить на сервере (нет соединения). Объект добавляется в таблицу `DeletePushEntity`:

```kotlin
@Entity
data class DeletePushEntity(
    val id: Long,                            // автоинкремент
    val objectType: DeletePushObjectType,    // тип объекта
    val serverId: Long,                      // ID на сервере
    val parentId: UUID                       // ID родительской инспекции
)

enum class DeletePushObjectType {
    VOLS,           // группа ВОЛС
    VOLS_ITEMS,     // позиция ВОЛС
    VOLS_CAPTURE,   // фото ВОЛС
    PILLAR_CAPTURE  // фото инспекции
}
```

При следующем Push эти объекты будут отправлены на удаление, после чего запись в `DeletePushEntity` удаляется.

**Уникальный индекс**: `(serverId, objectType)` — предотвращает дублирование запросов на удаление.

---

## 6. Диаграмма потоков данных

```
УСТРОЙСТВО                              СЕРВЕР
──────────                              ──────
LocalDB                                 Remote API
   │                                         │
   │  ←────── Sync (загрузка) ──────────────│
   │           Tasks, PowerLines,            │
   │           Pillars, VOLS types,          │
   │           Operators, Inspections        │
   │                                         │
   │  ──────── Push (выгрузка) ────────────►│
   │           Photos, Inspections,          │
   │           VOLS, VOLS Items,             │
   │           Delete requests               │
   │                                         │
   │  WAIT_PUSH → WAIT_VERIFY               │
   │                                         │
   │  ←────── Review result ────────────────│
   │           ACCEPTED / DENIED             │
   │           (через следующий Sync)        │
```

---

## 7. Отсутствие фоновой синхронизации

**Важно**: В приложении **нет** автоматической/фоновой синхронизации:
- Нет WorkManager задач
- Нет периодических Push notifications для запуска sync
- Нет WebSocket соединения
- Sync запускается **только вручную** через кнопку на экране SelectTask

Это обусловлено полевыми условиями работы: нестабильное соединение, экономия батареи.
