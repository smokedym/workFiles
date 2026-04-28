# Управление объектами ВОЛС

## Обзор

Модуль отвечает за создание и редактирование записей об оборудовании ВОЛС (волоконно-оптических линий связи), размещённом на осматриваемой опоре. Каждая запись (группа ВОЛС) принадлежит оператору связи и содержит перечень устройств по типам с количеством.

Файлы:
- `screens/details/addVols/AddVolsViewModel.kt`
- `screens/details/addVols/AddVolsScreen.kt`
- `core/data/repository/PillarInspectionRepositoryImpl.kt` — `addVols()`, `updateVols()`
- `core/data/repository/VolsRepositoryImpl.kt`
- `core/domain/models/vols/` — доменные модели

---

## 1. Концепция ВОЛС (бизнес-смысл)

На каждой опоре линии электропередачи может быть размещено оборудование одного или нескольких операторов связи. Каждый **оператор** имеет свою **группу ВОЛС** с набором устройств:

```
Опора №42
  └── Группа ВОЛС (оператор: МТС)
  │     ├── Оптический делитель — 2 шт.
  │     ├── Кронштейн — 3 шт.
  │     └── Отпайка в дом — 1 шт.
  │
  └── Группа ВОЛС (оператор: Мегафон)
        └── УПМК с муфтой и запасом кабеля — 1 шт.
```

---

## 2. Состояние AddVolsState

| Поле | Тип | Назначение |
|------|-----|-----------|
| `addVolsInfoItem` | `Vols?` | Редактируемые данные группы ВОЛС |
| `operators` | `List<OperatorItem>` | Список операторов для выбора |
| `volsTypes` | `List<VolsType>` | Доступные типы устройств |
| `showCameraPicker` | Boolean | Показать выбор источника фото |
| `isLoading` | Boolean | Флаг загрузки/сохранения |
| `isEditable` | Boolean | Режим редактирования (false = только чтение) |
| `pillarName` | String | Имя опоры (для заголовка) |
| `parentPowerLineName` | String | Имя линии (для заголовка) |

---

## 3. Доменная модель Vols

```kotlin
data class Vols(
    val id: UUID,                        // суррогатный UUID (генерируется на устройстве)
    val items: List<VolsItem>,           // перечень устройств
    val operator: OperatorItem?,         // оператор (провайдер)
    val dateOfPlacing: LocalDateTime?,   // дата установки оборудования
    val images: List<PhotoCaptureItem>,  // фотографии группы ВОЛС
    val notes: String,                   // примечания
    val hasMark: Boolean                 // наличие маркировки кабеля
)

data class VolsItem(
    val id: UUID,
    val count: Int,       // количество единиц
    val type: VolsType    // тип устройства
)

data class VolsType(
    val id: Long,
    val name: String
)
```

### Типы устройств ВОЛС (VolsType)

Загружаются с сервера и хранятся локально в `VolsTypeEntity`:

| Типичное название | Описание |
|-----------------|---------|
| Оптический делитель | Пассивный делитель оптического сигнала |
| Кронштейн | Крепёжный элемент на опоре |
| Кроссмуфта | Соединительная муфта для кабелей |
| УПМК с муфтой и запасом кабеля | Устройство хранения муфты с запасом кабеля |
| Отпайка в дом | Ответвление кабеля к жилому зданию |
| Шкаф ШРМ | Распределительный шкаф |

---

## 4. События AddVolsEvent

| Событие | Описание |
|---------|---------|
| `Init(taskPillarId, volsId?)` | Загрузить данные (null volsId = новый ВОЛС) |
| `ClickAddPhoto` | Открыть выбор источника фото |
| `AddPhoto(PhotoResult)` | Сохранить фото к группе ВОЛС |
| `DeletePhoto(photoId)` | Удалить фото из группы |
| `OnSelectOperator(operator)` | Выбрать оператора |
| `OnVolsItemCountEdit(itemId, count)` | Изменить количество устройства |
| `OnVolsItemSelected(type)` | Добавить тип устройства в список |
| `SetPlacingDate(date)` | Установить дату монтажа |
| `OnHasMarkChecked(checked)` | Переключить флаг маркировки |
| `OnNotesTextEdit(text)` | Обновить примечания |
| `OnClickSave` | Сохранить группу ВОЛС |
| `CloseWithoutSave` | Закрыть без изменений |

---

## 5. Инициализация экрана

### Новая группа ВОЛС (volsId = null)

```
Init(taskPillarId, volsId=null)
    │
    ▼
operatorRepository.getCompanies() → List<OperatorItem>
volsTypeRepository.getVolsTypes() → List<VolsType>
    │
    ▼
addVolsInfoItem = Vols(
    id = UUID.randomUUID(),
    items = emptyList(),
    operator = null,
    dateOfPlacing = null,
    images = emptyList(),
    notes = "",
    hasMark = false
)
isEditable = true
```

### Редактирование существующего ВОЛС (volsId != null)

```
Init(taskPillarId, volsId)
    │
    ▼
volsRepository.getVolsInfo(volsId) → Vols
operatorRepository.getCompanies() → List<OperatorItem>
volsTypeRepository.getVolsTypes() → List<VolsType>
    │
    ▼
addVolsInfoItem = existingVols
isEditable = (pushStatus in [NOT_PUSHED, WAIT_PUSH, DENIED])
```

---

## 6. Управление устройствами (VolsItem)

### Добавление типа устройства

```
OnVolsItemSelected(type: VolsType)
    │
    ▼
Если type.id уже есть в items → не добавлять (дедупликация)
Иначе:
    items += VolsItem(
        id = UUID.randomUUID(),
        count = 1,
        type = type
    )
```

### Изменение количества

```
OnVolsItemCountEdit(itemId, count)
    │
    ▼
Если count == 0 → удалить элемент из items
Иначе → items[itemId].count = count
```

---

## 7. Сохранение группы ВОЛС

### Новая группа (создание)

```
OnClickSave (addVolsInfoItem.id не существует в БД)
    │
    ▼
isLoading = true
    │
    ▼
pillarInspectionRepository.addVols(vols, taskPillarId)
    │
    ├── [1] Получить или создать PillarInspectionEntity
    ├── [2] Вставить VolsEntity {
    │           id = vols.id (UUID),
    │           inspectionId,
    │           hasMark, notes, placingDate,
    │           operatorOwnerId,
    │           createdAt = now(), updatedAt = now()
    │       }
    ├── [3] Вставить VolsItemEntity[] для каждого item
    ├── [4] Вставить PhotoCaptureEntity[] для каждого images
    ├── [5] Вставить VolsCaptureEntity[] (связь фото ↔ vols)
    ├── [6] recalculateWorkStatus(inspection)
    └── [7] appDatabase.runTransaction { всё атомарно }
    │
    ▼
isLoading = false
→ NavigateBack
```

### Существующая группа (обновление)

```
OnClickSave (addVolsInfoItem.id уже есть в БД)
    │
    ▼
pillarInspectionRepository.updateVols(newVols, oldVols)
    │
    ├── [1] VolsEntity.update {
    │       hasMark, notes, placingDate,
    │       operatorOwnerId,
    │       updatedAt = now()
    │   }
    │
    ├── [2] UpdateObjectsUtils.updateObjects(
    │       oldItems = oldVols.items,
    │       newItems = newVols.items
    │   )
    │   ├── Удалённые items → DeletePushEntity (если serverId != null)
    │   │                  → VolsItemEntity.delete()
    │   ├── Новые items     → VolsItemEntity.insert()
    │   └── Изменённые items→ VolsItemEntity.update(count)
    │
    ├── [3] UpdateObjectsUtils.updateObjects(
    │       oldImages = oldVols.images,
    │       newImages = newVols.images
    │   )
    │   ├── Удалённые фото → DeletePushEntity (если serverId != null)
    │   │                 → PhotoCaptureEntity.delete()
    │   ├── Новые фото     → PhotoCaptureEntity.insert()
    │   │                 → VolsCaptureEntity.insert()
    │   └── (фото не редактируются, только добавление/удаление)
    │
    ├── [4] recalculateWorkStatus(inspection)
    └── [5] appDatabase.runTransaction { всё атомарно }
    │
    ▼
→ NavigateBack
```

---

## 8. UpdateObjectsUtils

**Утилита** для вычисления дельты между старой и новой версиями списка объектов:

```kotlin
object UpdateObjectsUtils {
    fun <T : HasId> updateObjects(
        old: List<T>,
        new: List<T>,
        onDelete: (T) -> Unit,
        onInsert: (T) -> Unit,
        onUpdate: (old: T, new: T) -> Unit
    ) {
        val oldIds = old.map { it.id }.toSet()
        val newIds = new.map { it.id }.toSet()

        // Удалённые (были, нет в новом)
        old.filter { it.id !in newIds }.forEach { onDelete(it) }

        // Новые (не было в старом)
        new.filter { it.id !in oldIds }.forEach { onInsert(it) }

        // Изменённые (есть в обоих)
        new.filter { it.id in oldIds }.forEach { newItem ->
            val oldItem = old.first { it.id == newItem.id }
            if (oldItem != newItem) onUpdate(oldItem, newItem)
        }
    }
}
```

---

## 9. Фотографии группы ВОЛС

Отличие от фото инспекции (опоры):

| Характеристика | Фото инспекции | Фото ВОЛС |
|---------------|---------------|-----------|
| Привязка | К `PillarInspection` | К `Vols` (группе) |
| Количество | Ровно 1 (обязательно) | Несколько (необязательно) |
| Таблица | `PillarInspectionEntity.photoId` | `VolsCaptureEntity` |
| GPS при съёмке | Да | Да |

### Добавление фото к ВОЛС

```
AddPhoto(PhotoResult)
    │
    ▼
captureRepository.addCaptureForVols(path, lat, lon, date) → photoId: UUID
    │
    ▼
VolsCaptureEntity.insert(volsId, captureId = photoId)
    │
    ▼
vols.images += PhotoCaptureItem(id=photoId, path, lat, lon, date)
```

---

## 10. VolsRepository

```kotlin
interface VolsRepository {
    suspend fun getVolsInfo(volsId: UUID): Vols
    suspend fun getVolsStatsForPillar(inspectionId: UUID): List<VolsTypeStats>
}
```

`getVolsStatsForPillar` используется для отображения сводной таблицы типов ВОЛС на экране опоры:

```kotlin
data class VolsTypeStats(
    val typeId: Long,
    val typeName: String,
    val totalCount: Int   // сумма по всем группам ВОЛС на данной опоре
)
```
