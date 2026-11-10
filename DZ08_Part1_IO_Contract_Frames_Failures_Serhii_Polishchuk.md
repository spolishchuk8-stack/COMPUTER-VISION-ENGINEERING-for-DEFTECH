# Документ 08 — Part 1  
## CV-компонент у мультисенсорній системі: ролі, I/O-контракт, фрейми, відмови

**Курс:** Computer Vision Engineering for DefenceTech  
**Автор:** Serhii Polishchuk  
**Дата:** 10.11.2026  
**Пов’язані артефакти:** карта системи (ДЗ1), Note 07 (change detection), Note 07 Domain Shift (A+B), пайплайн детеції/трекінгу (ДЗ2–5)

---

## 0. Use case (контекст)

Наземний / повітряний носій з **камерою** (RGB або IR), **далекоміром (range)**, **IMU** та **GNSS**.  
CV-компонент:

1. детекція об’єктів (Person / Car / Bicycle та суміжні класи VisDrone);  
2. трекінг (ByteTrack);  
3. пошук змін між кадрами (align → absdiff → events, Note 07).

**Sensor fusion у коді не реалізується** — лише архітектурний опис і контракти.  
Вихід CV іде в модуль локалізації / логіку застосунку (Part 2, ДЗ4).

---

## 1. Ролі сенсорів і data flow

### 1.1 Таблиця ролей

| Сенсор | Що вимірює | Чого не вимірює | Одиниці | Частота (типова) | Головна відмова |
|--------|------------|-----------------|---------|------------------|-----------------|
| **Camera** (RGB/IR) | інтенсивність зображення, текстура сцени | абсолютна дальність, орієнтація корпусу | px; timestamp ms | 15–30 Hz | засвітка / IR-зсув / втрата кадрів |
| **Range** (ToF / LiDAR / stereo depth proxy) | відстань до поверхонь у секторі | клас об’єкта, семантика | m | 10–20 Hz | відблиски, max-range, сліпа зона |
| **IMU** | кутова швидкість, прискорення | абсолютна позиція, семантика | rad/s, m/s² | 100–200 Hz | дрейф bias, насичення |
| **GNSS** | геопозиція (і швидкість) | локальна сцена, орієнтація з високою частотою | deg / m (ENU) | 1–10 Hz | urban canyon, втрата фіксу |

CV-компонент **споживає** переважно camera (+ опційно range для оцінки масштабу); IMU/GNSS — для часу, одометрії та прив’язки подій до map/odom (локалізація поза CV).

### 1.2 Data flow (логічний)

```
[Camera] --frames+ts--> [Preprocess] --> [Detector YOLO] --> [Tracker ByteTrack]
                              |                                    |
                              v                                    v
                    [Change Detection]                      [Tracks: id, bbox, cls, conf]
                              |                                    |
                              v                                    v
                      [Change Events] --------------------> [CV Output Bus]
[Range]  --range+ts--> (опційно scale / gate)                    |
[IMU]    --ω,a+ts----> (час / orientation hint)                  v
[GNSS]   --fix+ts---> (map anchor)                    [Localization / App Logic]
```

Узгодження з картою системи (ДЗ1): джерело → preprocess → CV-модель → постобробка/трекер → логіка (події змін + треки) → вихід.

---

## 2. I/O-контракт CV-компонента і стратегія часу

### 2.1 Входи

| Порт | Тип | Одиниці / формат | Примітка |
|------|-----|------------------|----------|
| `image` | HxWxC uint8 | px | RGB або 3-канальний IR |
| `stamp_cam` | int64 | ms (або ROS time) | час кадру |
| `range` (opt) | float / cloud | m | для scale / ROI |
| `stamp_range` | int64 | ms | |
| `imu` (opt) | ω, a | rad/s, m/s² | лише для sync / diagnostics |
| `gnss` (opt) | lat, lon, alt | deg, m | прив’язка події |
| `calib` | ids | — | `intrinsics_id`, `extrinsics_id` |

### 2.2 Виходи

| Порт | Поля | Опис |
|------|------|------|
| **tracks[]** | `track_id`, `cls`, `conf`, `bbox_xyxy`, `frame`, `t` | треки з bounding boxes у координатах **повного кадру** |
| **change_events[]** | `pair_id` / `frame`, `t`, `change_score`, `area`, `bbox_xyxy` або `xyxy`, `centroid` | події теми 7 (Note 07) |
| **quality** | `n_tracks`, `n_events`, `latency_ms`, `drop_flag` | поля якості |
| **status** | `ok` / `degraded` / `fail` | режим CV |

Бюджет затримки CV (ціль): **≤ 100 ms** на кадр на робочому залозі (детектор+трекер); change detection може йти рідше (на ключових парах).

### 2.3 Фрагмент контракту (YAML)

```yaml
cv_io_contract:
  version: "0.1"
  inputs:
    image: {type: image_bgr8, frame: camera_optical}
    stamp_cam: {type: time, unit: ms}
    range: {type: optional, unit: m, frame: range_finder}
  outputs:
    tracks:
      item:
        track_id: int
        cls: string   # Person|Car|Bicycle|...
        conf: float
        bbox_xyxy: [x1, y1, x2, y2]  # full-frame pixels
        frame: int
        t: int        # ms
    change_events:
      item:
        frame: int
        t: int
        change_score: float
        area: int
        bbox_xyxy: [x1, y1, x2, y2]
        centroid: [cx, cy]
    quality:
      latency_ms: float
      n_tracks: int
      n_events: int
      drop_flag: bool
  latency_budget_ms: 100
  time_policy: see_section_2_4
```

### 2.4 Стратегія часу (камера 30 Hz vs range 10 Hz)

| Параметр | Значення |
|----------|----------|
| **Master-час** | `stamp_cam` (камера — ведучий для CV) |
| **Association** | **nearest** у вікні; при `|t_cam − t_range| > max_gap_ms` — range **не** підмішується |
| **max_gap_ms** | 50 ms |
| **interpolate** | ні для range (дискретні виміри); IMU можна інтегрувати окремо в localization |
| **Буфер** | кільцевий на 3–5 range-семплів |
| **При miss range** | CV працює **тільки по camera**; `quality.drop_flag=false`, статус може лишатись `ok` (деградація масштабу) |
| **Ціна в затримці** | очікування nearest ≤ 1 період range (~100 ms) **не** блокує видачу tracks; range-асоціація best-effort |

---

## 3. Карта фреймів і калібрування

### 3.1 Coordinate frames (≥5)

```
map (static, world / ENU)
 └─[dynamic, GNSS+odom]─ odom
      └─[dynamic, IMU+odom]─ base_link          # корпус носія
           ├─[static extrinsics]─ camera_optical  # оптична вісь камери
           └─[static extrinsics]─ range_finder    # вісь далекоміра
```

| Ребро | Тип | Зміст |
|-------|-----|--------|
| map → odom | dynamic | дрейф одометрії / GNSS jump |
| odom → base_link | dynamic | поза / orientation корпусу |
| base_link → camera_optical | **static** | монтаж камери (extrinsics) |
| base_link → range_finder | **static** | монтаж range |
| camera_optical → image plane | static | intrinsics (K, D) |

**Шлях від центру рамки в кадрі до метрів корпусу:**  
`pixel (u,v)` → (K⁻¹, depth/range) → точка в `camera_optical` → `T_base_camera` → `base_link` (метри корпусу) → за потреби `odom` / `map`.

### 3.2 Calibration dependencies (для старту)

| ID | Призначення | Обов’язково для CV start |
|----|-------------|---------------------------|
| `intrinsics_id` | K, D камери | **так** (або дефолт + warning) |
| `extrinsics_id` | T_base_camera | так для метрів корпусу; для pure image tracks — ні |
| `extrinsics_range_id` | T_base_range | лише якщо range у pipeline |
| `timesync_id` | offset cam↔range | бажано |

Без валідних intrinsics CV може віддавати лише pixel-tracks; статус `degraded`.

---

## 4. Матриця відмов (≥4 режими)

| # | Відмова | Симптом | Вплив на вихід CV | Поведінка системи | Спосіб перевірки |
|---|---------|---------|-------------------|-------------------|------------------|
| 1 | **Втрата кадрів камери** / FPS↓ | `stamp` gaps > 2/FPS; порожній буфер | tracks порожні або застарілі; events не оновлюються | `status=fail` або `degraded`; остання валідна рамка з TTL; алерт | штучний drop кадрів; watchdog timeout |
| 2 | **IR / domain shift** (модель RGB на IR) | низький conf, mAP≈0 (див. zero-shot ДЗ07) | мало tracks; false negatives | використовувати ваги **A+B**; або `degraded` + лог domain | прогін HIT val; порівняння zero vs A+B |
| 3 | **Немає / зіпсований range** | `max_gap` постійно перевищено | tracks лишаються в px; немає метрів корпусу | CV **продовжує** pixel-tracks/events; scale вимкнено | відключити range на стенді |
| 4 | **Збій трекера / ID switch** | стрибки `track_id`, розриви | нестабільні треки; помилкова прив’язка change↔track | скинути трекер; публікувати detections без id; `degraded` | synthetic occlusion; метрики IDSW |
| 5 | **Помилка калібрування extrinsics** | систематичний зсув bbox→base_link | події «зміщені» в метрах корпусу | pixel-контракт валідний; метричний вихід `degraded` | known target на відомій відстані |

Рядки матриці читаються як заготовки сценаріїв теми 10.

---

## 5. Висновок

1. У I/O-контракт далі йде **формат події зміни** з Note 07: `frame`, `t`, `change_score` / area, `bbox_xyxy` у координатах повного кадру, опційно `track_id` через IoU.  
2. Паралельно контракт віддає **tracks** з bbox — єдиний вихідний інтерфейс для логіки й локалізації.  
3. При частковій відмові (немає range, слабкий GNSS) CV **залишається в pixel-домені** і не блокує видачу; `status=degraded`.  
4. Повна втрата камери → `fail`, без вигаданих треків.  
5. Domain shift RGB→IR закриваємо вагами **A+B** (ДЗ07), не fusion.  
6. На стенді з кількома каналами лишаються перевірки: timesync cam↔range, extrinsics base→camera, watchdog FPS, прогін failure-рядків 1–4.  
7. Карта фреймів і матриця відмов готові як вхід у ДЗ4 / Part 2 (локалізація) і сценарні тести теми 10.

---

## Додаток A. Оновлений фрагмент карти системи (CV)

| Компонент | Зараз |
|-----------|--------|
| CV-модель | YOLOv8n, ваги **A+B** (HIT-adapted) після domain-shift ДЗ07 |
| Постобробка | conf=0.25, iou=0.7; ByteTrack |
| Логіка | change events (τ*=50, Note 07) + tracks bus |
| Вихід | JSON/YAML tracks + change_events (контракт §2) |

## Додаток B. Запис у журнал рішень (уривок)

| Дата | Рішення | Чому |
|------|---------|------|
| 10.11.2026 | Master-час = camera; range nearest, max_gap=50 ms | CV прив’язаний до кадрів; range best-effort |
| 10.11.2026 | Ваги A+B як default для IR | ДЗ07: mAP50 0.23 vs ~0 zero |
| 10.11.2026 | При miss range не блокувати tracks | доступність важливіша за метричний scale |
