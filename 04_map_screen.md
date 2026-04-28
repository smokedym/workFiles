# Экран карты (Map Screen)

## Обзор

Главный экран приложения. Отображает карту с маркерами опор ВОЛС, управляет навигацией к детальным экранам, показывает текущую выборку опор для выбранной линии.

Файлы:
- `screens/map/MapScreen.kt` — основной Composable
- `screens/map/viewmodel/MapViewModel.kt` — логика экрана
- `screens/map/mapView/` — компоненты карты (MapLibre)
- `screens/map/ui/` — UI-компоненты (кнопки, overlays)
- `screens/map/utils/MapConfig.kt` — конфигурация карты
- `core/domain/usecase/MapCameraUseCase.kt` — расчёт bounding box

---

## 1. Состояние MapState

| Поле | Тип | Назначение |
|------|-----|-----------|
| `isLoading` | Boolean | Загрузка тайлов карты |
| `hasLocationAccess` | Boolean | Разрешение на геолокацию |
| `pillars` | `MapPillars` | Текущий набор маркеров опор |
| `cameraState` | `MapLibreCamera` | Внешнее состояние камеры (позиция, zoom) |

```kotlin
data class MapPillars(
    val pillarItems: List<PillarListItem>,
    val currentPillarId: Long?   // выбранная (подсвеченная) опора
)
```

---

## 2. События MapEvent

| Событие | Описание |
|---------|---------|
| `Init` | Запрос разрешений, инициализация карты |
| `MapLoadingFinished` | Тайлы карты загружены |
| `OnClickPillarPoint(taskPillarId)` | Tap по маркеру опоры |
| `OnZoomIn` | Кнопка увеличения масштаба |
| `OnZoomOut` | Кнопка уменьшения масштаба |
| `ShowPillarPoints(pillars)` | Отобразить список опор на карте |
| `ResetCurrentPillar` | Снять выделение с опоры |
| `MoveToCluster(minLat,maxLat,minLon,maxLon)` | Вписать кластер в экран |

---

## 3. Действия MapAction

| Действие | Эффект |
|---------|--------|
| `OpenPillarInfo(taskPillarId)` | Открыть боковую панель с деталями опоры |

---

## 4. Логика работы

### Инициализация (Init)

```
Init event
    │
    ▼
VolsSingleton.appGrant.LOCATION.request()
    │
    ├── Отказано → hasLocationAccess = false
    │               Открыть настройки устройства
    │
    └── Разрешено → hasLocationAccess = true
                    GPS service доступен?
                    │
                    └── Нет → предложить включить GPS
```

### Загрузка опор (ShowPillarPoints)

Вызывается из дочернего экрана `ChoosePillarPoint` через callback:

```
ChoosePillarViewModel наблюдает за БД
    │
    ▼
pillarsRepository.getPillarsOfPowerLine(taskPowerLineIds) → Flow<List<PillarListItem>>
    │
    ▼
Callback → MapViewModel.obtainEvent(ShowPillarPoints(pillars))
    │
    ▼
MapState.pillars = MapPillars(pillarItems = pillars, currentPillarId = null)
```

### Выбор опоры на карте

```
Пользователь нажимает на маркер опоры
    │
    ▼
OnClickPillarPoint(taskPillarId)
    │
    ▼
Анимация камеры к координатам опоры
cameraState.animateTo(pillar.coordinates, duration=1000ms)
    │
    ▼
pillars.currentPillarId = taskPillarId  (маркер подсвечивается)
    │
    ▼
sendAction(MapAction.OpenPillarInfo(taskPillarId))
    │
    ▼
Открывается панель PillarInfoScreen (Bottom Sheet или навигация)
```

### Управление масштабом

```kotlin
OnZoomIn:
    cameraState.zoom = min(currentZoom + 1.5, MAX_ZOOM)
    анимировать переход (duration = 1000ms)

OnZoomOut:
    cameraState.zoom = max(currentZoom - 1.5, MIN_ZOOM)
    анимировать переход (duration = 1000ms)
```

**Параметры камеры**:
- `MIN_ZOOM = 0.0`
- `MAX_ZOOM = 25.5`
- `ZOOM_STEP = 1.5`
- `ANIMATION_DURATION = 1000ms`

### Вписывание кластера (MoveToCluster)

```
MoveToCluster(minLat, maxLat, minLon, maxLon)
    │
    ▼
MapCameraUseCase.zoomBoxForPowerlinePoints(pillars)
    → BoundingBox(north, south, east, west)
    │
    ▼
cameraState.fitBounds(boundingBox, padding=50dp)
```

**Когда вызывается**: при выборе линии электропередачи — камера автоматически показывает все опоры этой линии.

---

## 5. Отображение маркеров опор

### Цветовая кодировка по статусам

Каждый `PillarListItem` содержит `workStatus` и `pushStatus`, на основе которых выбирается цвет маркера:

| Комбинация статусов | Цвет маркера | Значение |
|--------------------|--------------|---------|
| `workStatus = null` (не начата) | Серый | Опора не осмотрена |
| `NOT_STARTED` | Серый | Осмотр не начат |
| `WITHOUT_VOLS` | Жёлтый | Фото есть, ВОЛС нет |
| `WITHOUT_CAPTURE` | Оранжевый | ВОЛС есть, фото нет |
| `WITH_VOLS` + `NOT_PUSHED` | Зелёный | Готово, не отправлено |
| `WITH_VOLS` + `WAIT_PUSH` | Синий | Ожидает отправки |
| `WITH_VOLS` + `WAIT_VERIFY` | Голубой | На проверке |
| `WITH_VOLS` + `ACCEPTED` | Тёмно-зелёный | Принято |
| `WITH_VOLS` + `DENIED` | Красный | Отклонено |
| `currentPillarId == taskPillarId` | + пульсирующий контур | Выбранная опора |

### Кластеризация

При малом масштабе (zoom < 12) близко расположенные маркеры группируются в кластеры — показывается количество опор в группе. MapLibre автоматически управляет кластеризацией.

---

## 6. Конфигурация карты (MapConfig)

```kotlin
object MapConfig {
    val INITIAL_BOUNDS = BoundingBox(
        north = 77.0,   // Россия — северная граница
        south = 41.0,   // Россия — южная граница
        east = 190.0,   // Россия — восточная граница
        west = 19.0     // Россия — западная граница
    )
    val TILE_SOURCE = "..."  // URL тайл-сервера
}
```

---

## 7. MapCameraUseCase

**Файл**: `core/domain/usecase/MapCameraUseCase.kt`

```kotlin
object MapCameraUseCase {
    fun zoomBoxForPowerlinePoints(pillars: List<PillarListItem>): BoundingBox? {
        if (pillars.isEmpty()) return null
        return BoundingBox(
            north = pillars.maxOf { it.coordinates.latitude } + PADDING,
            south = pillars.minOf { it.coordinates.latitude } - PADDING,
            east  = pillars.maxOf { it.coordinates.longitude } + PADDING,
            west  = pillars.minOf { it.coordinates.longitude } - PADDING
        )
    }
}
```

**Назначение**: вычислить оптимальные границы камеры чтобы все опоры выбранной линии поместились в экран.

---

## 8. Интеграция с дочерними экранами

Экран карты является «хостом» для деталей. Дочерние экраны (ChoosePillarPoint, PillarInfo, AddVols) открываются поверх карты (Bottom Sheet или Navigation Component), сохраняя видимость карты на заднем плане.

```
MapScreen (хост)
    ├── MapView (MapLibre карта — всегда visible)
    ├── ZoomControls
    ├── LocationButton
    │
    └── BottomSheet / NavigationContainer:
          ├── SelectTask
          ├── SelectPowerLine
          ├── ChoosePillarPoint ──── callback → ShowPillarPoints ──► MapViewModel
          ├── PillarInfo
          ├── AddVols
          └── PreviewPhoto
```
