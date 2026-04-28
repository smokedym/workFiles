# База данных (Room Database)

## Обзор

**ORM**: AndroidX Room (Multiplatform)  
**Версия схемы**: 2  
**Файл**: `core/data/database/AppDatabase.kt`  
**Схемы миграций**: `composeApp/schemas/`

Локальная SQLite БД — основное хранилище данных приложения. Все бизнес-данные хранятся локально, синхронизируются с сервером по запросу.

---

## 1. Сущности (Entities)

### Иерархия зависимостей

```
TaskEntity (1)
    └── TaskPowerLineEntity (M) ── PowerLineEntity (1)
                                       └── PillarEntity (M)
                                               └── TaskPillarEntity (M)
                                                       └── PillarInspectionEntity (1)
                                                               ├── PhotoCaptureEntity (1)
                                                               └── VolsEntity (M)
                                                                       ├── VolsItemEntity (M) ── VolsTypeEntity
                                                                       └── VolsCaptureEntity (M) ── PhotoCaptureEntity
```

---

### TaskEntity — Задание инспектора

```kotlin
@Entity(tableName = "tasks")
data class TaskEntity(
    @PrimaryKey val id: Long,
    val name: String,
    val isAccepted: Boolean?
)
```

| Поле | Тип | Описание |
|------|-----|---------|
| `id` | Long (PK) | Серверный ID задания |
| `name` | String | Наименование задания |
| `isAccepted` | Boolean? | Статус принятия (null = неизвестно) |

---

### PowerLineEntity — Линия электропередачи

```kotlin
@Entity(tableName = "power_lines")
data class PowerLineEntity(
    @PrimaryKey val id: Long,
    val name: String
)
```

---

### TaskPowerLineEntity — Связь Задание ↔ Линия (M2M)

```kotlin
@Entity(tableName = "task_power_lines")
data class TaskPowerLineEntity(
    @PrimaryKey val id: Long,
    val taskId: Long,       // FK → TaskEntity
    val powerLineId: Long   // FK → PowerLineEntity
)
```

---

### PillarEntity — Опора ВОЛС

```kotlin
@Entity(tableName = "pillars")
data class PillarEntity(
    @PrimaryKey val id: Long,
    val name: String,
    val powerLineId: Long,    // FK → PowerLineEntity (CASCADE DELETE)
    val longitude: Double,    // GPS долгота
    val latitude: Double      // GPS широта
)
```

| Поле | Описание |
|------|---------|
| `id` | Серверный ID опоры |
| `name` | Номер/наименование опоры (например «ОП-42») |
| `powerLineId` | Принадлежность к линии |
| `longitude`, `latitude` | Координаты для отображения на карте |

---

### TaskPillarEntity — Связь Задание ↔ Опора (M2M)

```kotlin
@Entity(tableName = "task_pillars")
data class TaskPillarEntity(
    @PrimaryKey val id: Long,         // taskPillarId (ключ в задании)
    val pillarId: Long,               // FK → PillarEntity
    val taskPowerLineId: Long         // FK → TaskPowerLineEntity
)
```

`taskPillarId` — основной ключ для работы с инспекциями. Используется повсеместно вместо `pillarId`.

---

### PillarInspectionEntity — Инспекция опоры

```kotlin
@Entity(tableName = "pillar_inspections",
        indices = [Index("taskPillarId")])
data class PillarInspectionEntity(
    @PrimaryKey val id: UUID,              // мобильный UUID (суррогатный)
    val taskPillarId: Long,                // FK → TaskPillarEntity
    val serverId: Long?,                   // ID на сервере (null = не выгружено)
    val photoId: UUID?,                    // FK → PhotoCaptureEntity
    val workStatus: PillarInspectionWorkStatus,
    val pushStatus: PillarInspectionPushStatus,
    val deniedReason: String?,             // причина отклонения сервером
    val createdAt: Long,                   // ms, device time
    val updatedAt: Long,                   // ms, device time
    val serverUpdatedAt: Long?             // ms, server time
)
```

**Ключевые особенности:**
- `id` — UUID генерируется на устройстве, используется как `mobileId` при выгрузке
- Индекс на `taskPillarId` для быстрого поиска инспекции по опоре
- Два временны́х штампа: устройство (`createdAt`, `updatedAt`) и сервер (`serverUpdatedAt`)

---

### PhotoCaptureEntity — Фотофиксация

```kotlin
@Entity(tableName = "photo_captures")
data class PhotoCaptureEntity(
    @PrimaryKey val id: UUID,
    val path: String,          // локальный путь к файлу
    val captureDate: Long,     // ms, время съёмки
    val longitude: Double,     // GPS при съёмке
    val latitude: Double,      // GPS при съёмке
    val serverId: Long?        // ID на сервере после выгрузки
)
```

Используется как для фото инспекции опоры, так и для фото групп ВОЛС (через `VolsCaptureEntity`).

---

### VolsEntity — Группа ВОЛС

```kotlin
@Entity(tableName = "vols")
data class VolsEntity(
    @PrimaryKey val id: UUID,              // мобильный UUID
    val inspectionId: UUID,               // FK → PillarInspectionEntity
    val serverId: Long?,                   // ID на сервере
    val hasMark: Boolean,                  // наличие маркировки
    val notes: String,                     // примечания
    val placingDate: Long?,               // дата установки (ms, только дата)
    val operatorOwnerId: Long?,            // FK → OperatorEntity
    val createdAt: Long,
    val updatedAt: Long,
    val serverUpdatedAt: Long?
)
```

---

### VolsItemEntity — Позиция ВОЛС (тип устройства + количество)

```kotlin
@Entity(tableName = "vols_items",
        foreignKeys = [ForeignKey(
            entity = VolsEntity::class,
            onDelete = CASCADE
        )])
data class VolsItemEntity(
    @PrimaryKey val id: UUID,
    val parentId: UUID,         // FK → VolsEntity (CASCADE DELETE)
    val typeId: Long,           // FK → VolsTypeEntity
    val count: Int,             // количество единиц
    val serverId: Long?,
    val createdAt: Long,
    val updatedAt: Long,
    val serverUpdatedAt: Long?
)
```

---

### VolsTypeEntity — Тип устройства ВОЛС

```kotlin
@Entity(tableName = "vols_types")
data class VolsTypeEntity(
    @PrimaryKey val id: Long,
    val name: String
)
```

Справочник. Загружается с сервера при синхронизации.

---

### VolsCaptureEntity — Связь ВОЛС ↔ Фото (M2M)

```kotlin
@Entity(tableName = "vols_captures")
data class VolsCaptureEntity(
    @PrimaryKey val id: UUID,
    val volsId: UUID,      // FK → VolsEntity
    val captureId: UUID    // FK → PhotoCaptureEntity
)
```

---

### OperatorEntity — Оператор связи

```kotlin
@Entity(tableName = "operators")
data class OperatorEntity(
    @PrimaryKey val id: Long,
    val name: String       // Например: «МТС», «Мегафон», «Ростелеком»
)
```

---

### PushEntity — Push-уведомление

```kotlin
@Entity(tableName = "push_notifications")
data class PushEntity(
    @PrimaryKey val id: UUID,
    // ... поля push-уведомления
)
```

---

### DeletePushEntity — Очередь удалений

```kotlin
@Entity(tableName = "delete_push",
        indices = [UniqueIndex("serverId", "objectType")])
data class DeletePushEntity(
    @PrimaryKey(autoGenerate = true) val id: Long,
    val objectType: DeletePushObjectType,
    val serverId: Long,
    val parentId: UUID    // inspectionId для контекста
)
```

---

## 2. DAOs (Data Access Objects)

Расположение: `core/data/database/dao/`

### PillarDao

```kotlin
@Dao
interface PillarDao {
    @Query("SELECT * FROM pillars WHERE id = :pillarId")
    suspend fun getPillarById(pillarId: Long): PillarEntity?

    @Query("SELECT p.* FROM pillars p
            JOIN task_pillars tp ON tp.pillarId = p.id
            WHERE tp.id IN (:taskPillarIds)")
    fun getPillarsByTaskPillarIds(taskPillarIds: List<Long>): Flow<List<PillarWithStatus>>

    @Upsert
    suspend fun upsertPillars(pillars: List<PillarEntity>)
}
```

### PillarInspectionDao

```kotlin
@Dao
interface PillarInspectionDao {
    @Query("SELECT * FROM pillar_inspections WHERE taskPillarId = :taskPillarId")
    suspend fun getByTaskPillarId(taskPillarId: Long): PillarInspectionEntity?

    @Query("SELECT * FROM pillar_inspections WHERE pushStatus = 'WAIT_PUSH'")
    suspend fun getAllInspectionsForPush(): List<PillarInspectionEntity>

    @Update
    suspend fun updateInspection(entity: PillarInspectionEntity)
}
```

### TaskDao, PowerLineDao

CRUD + Flow-запросы для реактивного обновления UI.

### VolsDao

```kotlin
@Dao
interface VolsDao {
    @Query("SELECT COUNT(*) FROM vols WHERE inspectionId = :inspectionId")
    suspend fun countByInspectionId(inspectionId: UUID): Int

    @Query("SELECT * FROM vols WHERE id = :volsId")
    suspend fun getVolsById(volsId: UUID): VolsEntity?
}
```

### CaptureDao

Управление `PhotoCaptureEntity` и `VolsCaptureEntity`.

### DeletePushDao

```kotlin
@Dao
interface DeletePushDao {
    @Insert(onConflict = IGNORE)  // уникальный индекс = ignore дублей
    suspend fun insert(entity: DeletePushEntity)

    @Query("DELETE FROM delete_push WHERE id = :id")
    suspend fun deleteById(id: Long)
}
```

---

## 3. Type Converters

Расположение: `core/data/database/converters/`

| Конвертер | Преобразование |
|-----------|--------------|
| `UuidConverter` | `UUID ↔ String` |
| `PillarStatusConverter` | `PillarStatus enum ↔ String` |
| `DeletePushObjectTypeConverter` | `DeletePushObjectType enum ↔ String` |
| `LocalDateTimeConverter` | `LocalDateTime ↔ Long (ms)` |

---

## 4. Миграции схемы

**Версия 1 → 2**: добавлена таблица `delete_push` для очереди удалений.

Файлы миграций: `composeApp/schemas/ru.meket.volsmobile.AppDatabase/`

---

## 5. Транзакционность

Все комплексные операции выполняются в транзакции:

```kotlin
appDatabase.withTransaction {
    // Атомарная группа операций
    inspectionDao.update(inspection)
    captureDao.insert(photo)
    inspectionDao.update(inspection.copy(photoId = photo.id))
}
```

Это гарантирует консистентность данных при сбоях (crash, OOM).
