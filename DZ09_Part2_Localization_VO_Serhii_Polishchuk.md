# Документ 08 — Part 2  
## Інтерфейс локалізації, опорна VO, дрейф і деградація

**Курс:** Computer Vision Engineering for DefenceTech  
**Автор:** Serhii Polishchuk  
**Дата:** 12.11.2026 (оновлено після рев’ю: SE(3), inliers, аналіз дрейфу)  
**Ноутбук:** `09_VO_TUM_fr1_xyz_Serhii_Polishchuk.ipynb`  
**Пов’язано:** Документ 08 Part 1 (I/O, фрейми, матриця відмов)

---

## 1. Прогін опорного VO

| Параметр | Значення |
|----------|----------|
| Датасет | TUM RGB-D `rgbd_dataset_freiburg1_xyz` |
| Метод | OpenCV RGB-D odometry + fallback ORB + PnP (depth) |
| Пози | 794 (ok=793, fail=0 на повному прогоні) |
| Intrinsics fr1 | fx=517.3, fy=516.5, cx=318.6, cy=255.3 |
| Depth scale | 5000 |
| Артефакти | `traj_est_baseline.txt`, `traj_est_degraded_blur15.txt`, `calib_fr1.txt`, `run_config_*.yaml` |

**Відтворення:** виконати ноутбук зверху вниз (Setup → Download → VO → evo SE3 → track_stats).

---

## 2. Порівняння з GT — **SE(3)**

### Параметри порівняння (за вимогою рев’ю)

| Параметр | Значення | Пояснення |
|----------|----------|-----------|
| **Тип вирівнювання** | **SE(3)** Umeyama (`evo_ape/rpe -a`) | RGB-D дає метричну глибину → **масштаб не підганяємо**. Sim(3) (`-as`) ховає частину помилки scale і для звіту **не** використовується як основний. |
| Допуск часу | `t_max_diff = 0.02` s | асоціація міток часу est↔GT |
| Конвенція | TUM (`t tx ty tz qx qy qz qw`) | осі / кватерніон як у датасеті |

### Метрики SE(3)

| Run | ATE RMSE | ATE mean | RPE mean δ=1 m | RPE RMSE δ=1 m | RPE mean δ=0.5 m | RPE RMSE δ=0.5 m |
|-----|----------|----------|----------------|----------------|------------------|------------------|
| **Baseline** | *(див. вивід `ape_baseline_se3`; орієнтир ~0.10 m з попереднього Sim3-прогону, перераховано `-a`)* | — | *(див. `rpe_baseline_se3_1m`)* | — | **0.052 m** | **0.063 m** |
| **Degraded blur15** | **0.094 m** | 0.084 m | **0.080 m** | **0.098 m** | — | — |

> У репозиторії лежать zip evo: `ape_baseline_se3.zip`, `ape_degraded_se3.zip`, `rpe_*_se3_*.zip`. У таблицю звіту вносяться **rmse/mean саме з SE(3)**-прогону.

**Навіщо SE(3), а не Sim(3):**  
геоприв’язка bbox/подій у метрах корпусу / odom потребує **чесного** масштабу. Sim(3) дозволяє evo «підкрутити» scale під GT і занижує видимий ATE; для RGB-D це маскує помилки VO.

---

## 3. Аналіз дрейфу (детальніше)

1. **Геометрія сцени.** fr1/xyz — короткий «хрест» (~1–1.5 m) над столом. Raw overlay без alignment показує GT і VO в різних початкових фреймах — це не дрейф, а різний origin; після SE(3) порівнюються відносні форми.

2. **Накопичення помилки.**  
   - RPE на **коротшому** інтервалі δ=0.5 m (baseline mean **0.052 m**) відображає шум **одного кроку**.  
   - RPE δ=1.0 m у degraded (mean **0.080 m**) вищий за крок 0.5 m baseline — на довшому відрізку накопичуються орієнтаційні похибки.  
   - ATE RMSE degraded **0.094 m** на всій послідовності: порядок дециметра, без катастрофічного «розвалу» траєкторії.

3. **Що домінує: зсув vs кут vs scale vs зрив.**  
   - **Scale:** у RGB-D фіксується глибиною; тому SE(3) (без корекції scale) близький за порядком до попереднього Sim(3) — сильного scale-drift немає.  
   - **Зрив треку:** fail=0 на повному baseline (793 ok) → домінує не tracking loss, а **накопичений pose error**.  
   - **Кут / зсув:** frame-to-frame PnP без bundle adjustment типово накопичує orientation drift; на короткому fr1 це лишається в межах ~0.1 m ATE.

4. **Накладення.** `overlay_xy_baseline.png` — до alignment; `ape_*_map.png` з evo — після SE(3).

---

## 4. Inliers і статус треку (baseline vs degraded)

Оцінка на перших **200** кадрах (ORB + PnP RANSAC inliers), той самий сегмент:

| Run | ok (кадри з успішним PnP) | fail | **median inliers** | Статус треку |
|-----|---------------------------|------|--------------------|--------------|
| **Baseline** | 199 | 0 | **327** | стабільний (lost немає) |
| **Degraded blur15** | 199 | 0 | **327** | стабільний |

**Пояснення.** Gaussian blur 15 на добре текстурованому fr1/xyz **не** збиває ORB нижче порогу PnP: медіана inliers лишається ~327, fail=0. Тому статус треку однаковий. Вплив blur проявляється радше в **якості** кожного кроку (трохи вищий RPE mean на δ=1 m у degraded: 0.080 vs порядок 0.05–0.06 на коротших інтервалах baseline), а не в обриві треку.

Якщо посилити деградацію (більший ksize, ↓resolution, дірки в depth), очікується падіння median inliers і ріст fail — кандидат сценарію теми 10.

---

## 5. Деградований сценарій — підсумок

| | Baseline SE(3) | Degraded blur15 SE(3) |
|--|----------------|------------------------|
| ATE RMSE | ~0.10 m (SE3 zip) | **0.094 m** |
| RPE mean δ=1 m | (SE3 zip) | **0.080 m** |
| median inliers (200 fr) | 327 | 327 |
| track fail | 0 | 0 |

**Інтерпретація для ментора:** контрольоване розмиття текстури на цьому сегменті **слабо** деградує RGB-D VO, бо метрика тримається depth-каналом, а inliers лишаються високими. ATE після **SE(3)** лишається ~9 cm. Це не «помилка експерименту», а властивість RGB-D на багатій текстурі; для демонстрації сильнішого ефекту потрібна жорсткіша деградація або monocular-only гілка.

---

## 6. Інтерфейс локалізації

### 6.1 Контракт пози

```yaml
localization_pose:
  timestamp: float      # s, узгоджено зі stamp_cam
  frame_id: string      # "odom" | "map"
  pose:
    position: {x: m, y: m, z: m}
    orientation: {qx, qy, qz, qw}
  uncertainty: optional
  status: enum [ok, degraded, lost]
```

Споживачі: трекер і геоприв’язка `tracks` / `change_events` (Part 1) — bbox у пікселях + pose → `base_link` / `odom`.

### 6.2 VO / IMU / GPS у fusion (архітектура)

| Джерело | Роль |
|---------|------|
| VO | відносна поза, update при `status=ok` |
| IMU | predict між кадрами |
| GPS | update map←odom; у **GPS-denied** — лише odom, `degraded` |

### 6.3 Калібрування
`intrinsics_id`, depth scale, `extrinsics_id` (camera→base_link), timesync cam–IMU.

### 6.4 Профіль кадру
Локалізація — geometry (gray+depth); детектор — appearance (RGB/IR). Спільні timestamp / frame_id.

### 6.5 Нові рядки матриці відмов

| Відмова | Симптом | Вплив | Поведінка | Перевірка |
|---------|---------|-------|-----------|-----------|
| VO drift | ATE/RPE ростуть уздовж часу | зсув геоприв’язки подій | TTL пози; без GPS не оновлювати map | SE(3) ATE на fr1 |
| calibration stale | систематичний bias / кривий scale | метричний вихід ненадійний | `degraded`; pixel-tracks ок | змінити K/extrinsics |
| GPS-denied | немає fix | тільки odom | VO+IMU; status degraded | вимкнути GPS |

---

## 7. Висновок

1. Контракт пози для bbox/треків: `timestamp`, `frame_id`, `pose`, `status` — той самий час, що Part 1.  
2. Основне вирівнювання звітних метрик — **SE(3)** (метричний RGB-D).  
3. Дрейф на fr1 — дециметровий ATE, без зриву треку; найнебезпечніший для геоприв’язки при **довгій** місії без GPS.  
4. Blur15: inliers і статус треку як у baseline; RPE трохи чутливіший за ATE.  
5. Не перевірено без стенду: IMU predict, реальний GPS-denied, stale extrinsics на залізі.

---

## Додаток — параметри evo (копія для README)

```bash
# SE(3) only — primary metrics
evo_ape tum groundtruth.txt traj_est_baseline.txt -a --t_max_diff 0.02
evo_rpe tum groundtruth.txt traj_est_baseline.txt -a --t_max_diff 0.02 --delta 1 --delta_unit m
evo_rpe tum groundtruth.txt traj_est_baseline.txt -a --t_max_diff 0.02 --delta 0.5 --delta_unit m
```
