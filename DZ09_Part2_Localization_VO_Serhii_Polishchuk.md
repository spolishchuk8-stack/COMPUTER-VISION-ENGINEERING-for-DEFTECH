# Документ 08 — Part 2  
## Інтерфейс локалізації, опорна VO, дрейф і деградація

**Курс:** Computer Vision Engineering for DefenceTech  
**Автор:** Serhii Polishchuk  
**Дата:** 11.11.2026  
**Ноутбук:** `09_VO_TUM_fr1_xyz_Serhii_Polishchuk.ipynb`  
**Пов’язано:** Документ 08 Part 1 (I/O, фрейми, матриця відмов), CV-пайплайн (детектор + трекер + change events)

Part 1 і Part 2 читаються як **один** документ. Карта системи доповнюється сервісом локалізації.

---

## 1. Прогін опорного VO

| Параметр | Значення |
|----------|----------|
| Датасет | TUM RGB-D `rgbd_dataset_freiburg1_xyz` |
| Метод | OpenCV RGB-D odometry + fallback ORB+PnP (depth) |
| Кадрів / поз | 794 poses (ok=793, fail=0) |
| Intrinsics (fr1) | fx=517.3, fy=516.5, cx=318.6, cy=255.3 |
| Depth scale | 5000 |
| Артефакти | `traj_est_baseline.txt`, `calib_fr1.txt`, `run_config_baseline.yaml` |

**Команда відтворення** (Colab / локально після відкриття ноутбука):

```bash
# Execute notebook top-to-bottom, or:
# cells: Setup → Download → Associate → run_vo(baseline) → evo
jupyter nbconvert --to notebook --execute 09_VO_TUM_fr1_xyz_Serhii_Polishchuk.ipynb
```

Артефакти: `dz09_out/traj_est_baseline.txt`, `calib_fr1.txt`, `run_config_*.yaml`, `ape_*.zip`, plots.

---

## 2. Порівняння з GT (ATE / RPE)

### Параметри порівняння (обов’язкові)

| Параметр | Значення |
|----------|----------|
| Тип вирівнювання | **Sim(3)** Umeyama (`evo -as`) на основному прогоні; для RGB-D метрики також доречний **SE(3)** (`evo -a`) |
| Допуск асоціації часу | **t_max_diff = 0.02 s** |
| Конвенція осей | TUM RGB-D (timestamp + tx ty tz qx qy qz qw) |

### Baseline (чисті RGB-D кадри)

| Метрика | Значення |
|---------|----------|
| **ATE RMSE** | **0.099 m** |
| ATE mean / median | 0.091 / 0.081 m |
| ATE max | 0.214 m |
| **RPE mean (δ = 1.0 m)** | **0.061 m** |
| RPE RMSE (δ = 1.0 m) | 0.076 m |
| **RPE mean (δ = 0.5 m)** | **0.090 m** |
| RPE RMSE (δ = 0.5 m) | 0.105 m |

Для frame-to-frame VO без глобальної оптимізації на fr1/xyz це прийнятний рівень (дециметрний ATE).

---

## 3. Аналіз дрейфу

1. **Форма траєкторії.** fr1/xyz — короткий «хрест» уздовж осей столу; GT і VO до alignment лежать у різних початкових фреймах (очікувано). Після Umeyama ATE RMSE ≈ 0.10 m.
2. **Де розходиться.** Помилка накопичується вздовж сегментів руху; max ATE 0.21 m — локальні піки, не повний зрив треку (fail=0 на 793 кроках).
3. **Що домінує.** За RPE: на коротшому інтервалі (0.5 m) відносна помилка вища (0.09 m), ніж на 1.0 m (0.06 m) у mean — типовий шум кроку; **масштаб** у RGB-D тримається глибиною, домінує **накопичення орієнтації / зсуву**, не «вибух» scale як у чистому моно VO.
4. **Накладення.** `overlay_xy_baseline.png` — raw (до SE3); `ape_baseline_map.png` — після alignment у evo.
5. **Inliers / track status.** У прогоні ok=793, fail=0 → трек не губився; окрема медіана ORB-inliers у fallback не логувалась стабільно по кадрах (залежить від гілки RgbdOdometry vs PnP).

---

## 4. Деградований сценарій

| Параметр | Значення |
|----------|----------|
| Тип | Gaussian blur RGB, **ksize = 15** |
| Сегмент | той самий fr1/xyz |
| Конфіг | `run_config_degraded_blur15.yaml` |
| Траєкторія | `traj_est_degraded_blur15.txt` |

### Метрики degraded vs baseline

| | Baseline | Degraded blur15 |
|--|----------|-----------------|
| **ATE RMSE** | 0.099 m | **0.089 m** |
| ATE mean | 0.091 m | 0.081 m |
| **RPE mean δ=1 m** | 0.061 m | **0.067 m** |
| RPE RMSE δ=1 m | 0.076 m | 0.076 m |

**Висновок по деградації.** На текстурованому fr1/xyz помірний blur майже не руйнує RGB-D VO: глибина зберігає метрику, ATE після Sim(3) навіть трохи нижчий; RPE mean трохи гірший (шумніші кроки). Статус треку: прогін завершено без масових fail. Для жорсткішої деградації (сильніший blur / ↓resolution) очікується ріст RPE і зриви PnP — кандидат у сценарні тести теми 10.

---

## 5. Інтерфейс локалізації

### 5.1 Контракт пози (вихід VO / localization service)

```yaml
localization_pose:
  timestamp: float   # s, той самий clock що stamp_cam де можливо
  frame_id: string   # "odom" | "map"
  pose:
    # TUM / ROS-friendly
    position: {x: m, y: m, z: m}
    orientation: {qx, qy, qz, qw}  # unit quaternion
  uncertainty:       # optional
    type: diagonal_cov  # or "none"
    values: [σx, σy, σz, ...]
  status: enum [ok, degraded, lost]
```

Споживачі:
- **трекер / геоприв’язка подій** з Part 1: `change_events` + `tracks` з bbox у пікселях → за наявності depth/pose — у `base_link` / `odom`;
- логіка застосунку: фільтрація подій за районом карти.

### 5.2 Ролі у sensor fusion (архітектура, без коду EKF)

| Джерело | Роль | Predict / Update |
|---------|------|------------------|
| **VO** | відносна поза, висока частота | predict (інтеграція кроків) + update, коли track ok |
| **IMU** | висока частота ω, a | predict між кадрами камери |
| **GPS/GNSS** | абсолют (низька частота) | update `map`←`odom` при фіксі |

**GPS-denied:** лишаються VO+IMU; `frame_id` лишається `odom`; `status=degraded` при рості uncertainty; події bbox лишаються валідними в пікселях (Part 1).

### 5.3 Залежності калібрування

| ID | Навіщо для VO / pose |
|----|----------------------|
| `intrinsics_id` | K камери (обов’язково) |
| `depth_scale` / depth calib | метри RGB-D |
| `extrinsics_id` | camera→base_link (метри корпусу) |
| timesync cam↔IMU | якщо IMU в fusion |

Застаріле калібрування (`calibration stale`) → систематичний bias пози / неправильний scale.

### 5.4 Профіль зображення

- **Локалізація (VO)** бачить **повний кадр** (сірий + depth) для odometry.  
- **Детектор рамок** бачить той самий кадр (або ROI), але інший «профіль» обробки: RGB/IR → detections/tracks (Part 1).  
Одним реченням: *локалізація споживає geometry-stream кадру, детектор — appearance-stream; час і frame_id спільні.*

### 5.5 Нові рядки матриці відмов (до Part 1)

| Відмова | Симптом | Вплив на вихід | Поведінка | Перевірка |
|---------|---------|----------------|-----------|-----------|
| **VO drift** | ATE/RPE ростуть уздовж траєкторії; розходження з GT | pose `degraded`; геоприв’язка подій зсувається | обмежити TTL пози; не оновлювати map без GPS | власний прогін fr1 + ATE |
| **calibration stale** | систематичний зсув / кривий scale | метричний вихід ненадійний | `degraded`; pixel-tracks з Part 1 лишаються | змінити K або extrinsics у конфігу |
| **GPS-denied** | немає GNSS fix | тільки odom; дрейф не компенсується глобально | VO+IMU predict; status degraded | вимкнути GPS на стенді |

---

## 6. Оновлення карти системи (фрагмент)

| Компонент | Зараз |
|-----------|--------|
| Локалізація | VO (OpenCV RGB-D / ORB+PnP) → `localization_pose` у `odom` |
| Вихід локалізації | timestamp, frame_id, pose, status |
| Споживач | трекер + геоприв’язка change_events / tracks (Part 1) |
| Відмови | + VO drift, calibration stale, GPS-denied |

### Журнал рішень (уривок)

| Дата | Рішення | Чому |
|------|---------|------|
| 11.11.2026 | Опорний VO = OpenCV RGB-D + ORB/PnP fallback на TUM fr1/xyz | немає бінарника курсу; відтворюваний прогін |
| 11.11.2026 | Eval: evo APE/RPE, t_max_diff=0.02, Sim(3)/SE(3) | вимога завдання; RGB-D метричний |
| 11.11.2026 | Деградація = Gaussian blur 15 | контрольована ↓текстура, той самий сегмент |
| 11.11.2026 | Контракт пози → споживачі bbox/events | єдиний час і frame_id з Part 1 |

---

## 7. Висновок

1. **Контракт пози** для подій із bounding boxes і треків: `timestamp`, `frame_id` (`odom`/`map`), `pose` (t + q), `status` / uncertainty — той самий clock, що `stamp_cam` у Part 1.  
2. **Дрейф найнебезпечніший** для геоприв’язки подій при довгій роботі **без GPS** (накопичення в `odom`): локально на fr1 ATE ~0.1 m, на довгій місії без update map зсув подій на метри.  
3. **Blur** на цьому сегменті дав помірний ефект (RPE трохи ↑, ATE порівнянний) — RGB-D тримається глибиною; сильніша втрата текстури / depth holes — окремий сценарій теми 10.  
4. **Не перевірено без стенду IMU/GNSS:** predict IMU між кадрами, реальний GPS-denied у полі, stale extrinsics на залізі, timesync cam–IMU.  
5. Part 2 готовий як вхід у **тему 10** (сценарні тести) і повне ДЗ 4: інтерфейс локалізації, degraded run, три нові рядки матриці відмов.

---

## Додаток — Зведена таблиця метрик

| Run | ATE RMSE | RPE mean δ=1 m | RPE mean δ=0.5 m |
|-----|----------|----------------|------------------|
| baseline | 0.099 m | 0.061 m | 0.090 m |
| degraded blur15 | 0.089 m | 0.067 m | — |
