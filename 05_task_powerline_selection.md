# Выбор задания и линии электропередачи

## Обзор

Два последовательных экрана для навигации к конкретной линии электропередачи в рамках задания инспектора. Второй экран также является точкой запуска синхронизации данных.

Файлы:
- `screens/details/selectTask/SelectTaskViewModel.kt`
- `screens/details/selectPowerLine/SelectPowerLineViewModel.kt`
- `screens/details/choosePillarPoint/ChoosePillarViewModel.kt`

---

## 1. Экран выбора задания (SelectTask)

**Файл**: `screens/details/selectTask/SelectTaskViewModel.kt`

### Состояние SelectTaskState

| Поле | Тип | Назначение |
|------|-----|-----------|
| `tasks` | `List<TaskItem>` | Полный список заданий |
| `filteredTasks` | `List<TaskItem>` | Отфильтрованный список |
| `searchText` | String | Строка поиска |
| `isLoading` | Boolean | Флаг загрузки/синхронизации |

### События SelectTaskEvent

| Событие | Описание |
|---------|---------|
| `Init` | Загрузить список заданий из БД |
| `ClickTaskItem(taskId)` | Выбрать задание → перейти к линиям |
| `SearchTextChanged(text)` | Обновить фильтр |
| `Sync` | Запустить синхронизацию |
| `ClickBack` | Вернуться назад |

### Загрузка заданий

```
Init event
    │
    ▼
taskRepository.getTasks() → Flow<List<TaskItem>>
    │
    ▼
state.tasks = items
state.filteredTasks = items (без фильтра)
```

### Фильтрация по поиску

```
SearchTextChanged(text)
    │
    ▼
filteredTasks = tasks.filter { task ->
    task.name.contains(text, ignoreCase = true)
}
searchText = text
```

### Процесс синхронизации

Синхронизация выполняется в строго последовательном порядке: сначала **выгрузка** локальных данных (Push), затем **загрузка** с сервера (Sync).

```
Sync event
    │
    ▼
isLoading = true
    │
    ▼
[1] pushManager.push()
    │   Выгружает локально сохранённые инспекции на сервер
    │   └── Ошибка? → isLoading = false, showError
    │
    ▼ [Успех Push]
[2] syncManager.sync()
    │   Загружает актуальные данные с сервера
    │   └── Ошибка? → isLoading = false, showError
    │
    ▼ [Успех Sync]
isLoading = false
Список задач обновляется автоматически (Flow)
```

**Правило**: синхронизация всегда запускается вручную пользователем — нет фоновой автоматической синхронизации.

### TaskItem (доменная модель)

```kotlin
data class TaskItem(
    val id: Long,
    val name: String,
    val isAccepted: Boolean?
)
```

---

## 2. Экран выбора линии электропередачи (SelectPowerLine)

**Файл**: `screens/details/selectPowerLine/SelectPowerLineViewModel.kt`

### Состояние SelectPowerLineState

| Поле | Тип | Назначение |
|------|-----|-----------|
| `powerLines` | `List<PowerLineListItem>` | Все линии для задания |
| `filteredPowerLines` | `List<PowerLineListItem>` | Отфильтрованный список |
| `searchText` | String | Строка поиска |
| `isLoading` | Boolean | Флаг загрузки |

### Загрузка линий для задания

```
Init(taskId)
    │
    ▼
powerLineRepository.getPowerLinesForTask(taskId)
    → Flow<List<PowerLineListItem>>
    │
    ▼
state.powerLines = items
state.filteredPowerLines = items
```

### Фильтрация

Аналогично SelectTask:
```kotlin
filteredPowerLines = powerLines.filter { line ->
    line.name.contains(searchText, ignoreCase = true)
}
```

### Выбор линии → к опорам

```
ClickPowerLine(taskPowerLineId)
    │
    ▼
→ NavigateToChoosePillarPoint(taskPowerLineIds = listOf(taskPowerLineId))
```

**Примечание**: передаётся список `taskPowerLineIds` (а не одно значение) — архитектурно предусмотрена возможность показа опор нескольких линий одновременно.

### PowerLineListItem (доменная модель)

```kotlin
data class PowerLineListItem(
    val taskPowerLineId: Long,
    val powerLineId: Long,
    val name: String
)
```

---

## 3. Экран выбора точки опоры (ChoosePillarPoint)

**Файл**: `screens/details/choosePillarPoint/ChoosePillarViewModel.kt`

Этот экран **не показывает отдельный список** — он передаёт данные об опорах на карту и выступает навигационным «входом» к инспекции конкретной опоры.

### Состояние ChoosePillarPointState

| Поле | Тип | Назначение |
|------|-----|-----------|
| `currentPowerLine` | `PowerLineItem?` | Информация о выбранной линии |

### Поток работы

```
Init(taskPowerLineIds)
    │
    ▼
powerLineRepository.getPowerLinesInfo(taskPowerLineIds)
    → Flow<List<PowerLineItem>>
    │
    ├── currentPowerLine = powerLines.first()
    │
    ▼
pillarsRepository.getPillarsOfPowerLine(taskPowerLineIds)
    → Flow<List<PillarListItem>>
    │
    ▼ На каждое обновление:
callback.onPillarsChanged(pillars)  ← передаёт в MapViewModel
    │
    ▼
MapViewModel получает: ShowPillarPoints(pillars)
Карта показывает маркеры опор данной линии
    │
    ▼
Пользователь тапает маркер на карте
    │
    ▼
MapViewModel → OpenPillarInfo(taskPillarId)
    │
    ▼
→ NavigateToPillarInfo(taskPillarId)
```

### PowerLineItem (доменная модель)

```kotlin
data class PowerLineItem(
    val taskPowerLineId: Long,
    val powerLineId: Long,
    val name: String,
    val taskId: Long
)
```

---

## 4. Иерархия навигации

```
SelectTask
    └── [выбрать задание]
         ▼
    SelectPowerLine (taskId)
         └── [выбрать линию]
              ▼
         ChoosePillarPoint (taskPowerLineIds)
              │
              ├── [опоры → карта через callback]
              │
              └── [тап на маркер]
                   ▼
              PillarInfo (taskPillarId)
```

---

## 5. Связанные репозитории

### TaskRepository

```kotlin
interface TaskRepository {
    suspend fun getTasks(): Flow<List<TaskItem>>
}
```

Данные поступают из таблицы `TaskEntity` (БД), обновляются при синхронизации.

### PowerLineRepository

```kotlin
interface PowerLineRepository {
    suspend fun getPowerLinesForTask(taskId: Long): Flow<List<PowerLineListItem>>
    suspend fun getPowerLinesInfo(taskPowerLineIds: List<Long>): Flow<List<PowerLineItem>>
}
```

Использует связь через `TaskPowerLineEntity` (M2M: Task ↔ PowerLine).

### PillarRepository

```kotlin
interface PillarRepository {
    suspend fun getPillarsOfPowerLine(taskPowerLineIds: List<Long>): Flow<List<PillarListItem>>
    suspend fun getPillarInfo(taskPillarId: Long): Flow<PillarInfoItem>
}
```

`PillarListItem` включает текущие статусы инспекции — позволяет карте показывать актуальный цвет маркера.
