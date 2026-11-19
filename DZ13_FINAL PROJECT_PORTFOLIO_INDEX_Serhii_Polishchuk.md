# Production Evidence Portfolio — Фінальний проєкт

**Курс:** Computer Vision Engineering for DefenceTech  
**Автор:** Serhii Polishchuk  
**Дата:** 19.11.2026  
**Репозиторій:** COMPUTER-VISION-ENGINEERING-for-DEFTECH  

Цей індекс збирає **докази** з ДЗ 1–4, пакета 09 (тема 11) і теми 12. Кожен блок читається **без усного коментаря**: числа й таблиці лежать у вказаних Markdown / логах.

---

## Карта блоків портфоліо

| # | Блок | Джерело в курсі | Головний артефакт (читати) |
|---|------|-----------------|----------------------------|
| 1 | **Дані і baseline** | ДЗ 1 | `DZ01_…/DZ01_Data_Baseline_Report_…md`, notebook audit |
| 2 | **Рамка чи маска і відео** | ДЗ 2 / Звіт 05 | `DZ05_…/DZ05_Report_…md` (detector vs mask, degradations, latency) |
| 3 | **Зміни і система** | ДЗ 3 / Note 07 + Doc 08 Part 1 | `DZ07.3_…/07_Change_Detection_Note_…md`, `DZ08_…/08_Part1_…md` |
| 4 | **Локалізація і сценарії** | ДЗ 4 | `DZ09_…/08_Part2_…md`, `DZ10_…/08_Part3_…md` |
| 5 | **Швидкодія** | після теми 11 | `DZ_Final_…/09_Deploy_Package_…md` |
| 6 | **Надійність** | після теми 12 | `ДЗ_Фінал_Тема 12/12_Reliability_Defense_…md` |
| 7 | **Методологія** | тема 12 | §4 того ж звіту (6 питань нової CV-задачі) |
| 8 | **Рішення** | журнал | `13_Decision_Journal_…md` (цей пакет) |
| 9 | **Контур** | захист | `13_Defense_Outline_…md` (цей пакет) |

Додатково: **таксономія відмов** — `ДЗ_Фінал_Тема 12/12_Taxonomy_Failures_…md`.

---

## 1. Дані і baseline (ДЗ 1)

| Що вимагається | Де в портфоліо |
|----------------|----------------|
| Аудит датасету | VisDrone2019-DET audit notebook / report DZ01 |
| Метрики baseline | zero-shot / early baseline у DZ01–DZ02 reports |
| Зріз дрібних об’єктів | аналіз small objects у audit / error analysis DZ03 |
| Одна інтервенція до/після | fine-tune / error analysis (DZ03 notebook + report) |

**Знаменник:** метрики на VisDrone val до і після обраної інтервенції (див. відповідний звіт у репо).

---

## 2. Рамка чи маска і відео (ДЗ 2)

З **Звіту 05** (протокол val, imgsz=640, conf=0.25):

| Шлях | mAP50-95 | median ms | Peak GPU MB |
|------|----------|-----------|-------------|
| **A Detector** `yolov8n` | **0.0529** | ~15.4 | 322 |
| B Mask→box | 0.0481 | ~16–18 | 464 |

**Рішення:** шлях **A**.  
**Деградації (одна вісь для доказу):** noise L0→L3, \(M_0=0.0529\) → \(M_3=0.0087\).  
**FPS / p95:** також пакет 09 на T4 (p50≈10 ms PyTorch).

---

## 3. Зміни і система (ДЗ 3)

| Елемент | Артефакт |
|---------|----------|
| Прототип change | `07_Change_Detection.ipynb`, events JSON (`frame`, `t`, `change_score`, `area`, `xyxy`) |
| Таксономія FP change | Note 07 (illumination, residual registration, …) |
| I/O, sync, frames | Doc 08 **Part 1** |
| Матриця відмов (база) | Part 1 + таксономія теми 12 |

---

## 4. Локалізація і сценарії (ДЗ 4)

| Частина | Зміст | Артефакт |
|---------|--------|----------|
| **Part 2** | VO TUM fr1/xyz, SE(3) ATE/RPE, blur15, inliers, інтерфейс пози | `08_Part2_Localization_VO_…md` |
| **Part 3** | ≥8 сценаріїв, виконані S1–S5/S8, \(M_0,M_s,\Delta\), рекомендація релізу | `08_Part3_…md` + `configs/scenarios_matrix.yaml` |

**Реліз (limited):** RGB + помірні деградації; **не** noise L3, IR zero-shot, long GPS-denied.

---

## 5. Швидкодія (пакет 09)

| Рушій | p50 ms | p95 ms | FPS p50 | Примітка |
|-------|--------|--------|---------|----------|
| PyTorch Ultralytics | **10.17** | **11.88** | **98** | T4, imgsz=640 |
| ONNX path | 151.8 | 166 | 6.6 | артефакт деплою |
| TensorRT FP16 | — | — | — | **не зібрано** (причина в звіті 09) |

Узгодженість рамок framework↔ONNX: **match 100%** (36/36), median Δscore ≈ 0.014.

---

## 6. Надійність (тема 12)

| Елемент | Значення |
|---------|----------|
| Вісь | Gaussian **noise** |
| \(M_0\) | 0.0529 mAP50-95 |
| \(M_k/M_0\) L1…L3 | 0.92 / 0.69 / **0.16** |
| Stream | `frame_dropped` кожен 5-й кадр (`logs/stream_status_log.csv`) |
| p95 стабільність | ~12 ms на T4 (кілька вікон) |
| Таксономія | окремий файл `12_Taxonomy_Failures_…md` |

---

## 7. Методологія (нова CV-задача)

**Задача:** детекція людей на прохідній / КПП (fixed EO), не aerial VisDrone.  
Шість питань і обґрунтування контуру — у `12_Reliability_Defense_…md` §4.  
Родина: YOLO one-stage; перший контур **без** Re-ID / face / radar fusion.

---

## 8–9. Рішення і контур

Див. у цій папці:
- `13_Decision_Journal_Serhii_Polishchuk.md` (≥3 записи)
- `13_Defense_Outline_Serhii_Polishchuk.md` (10–12 хв, 4 блоки)

---

## Обмеження (що не міряли)

- борт / фізична плата  
- INT8 квантизація  
- симулятор як обов’язковий стенд  
- TensorRT engine (рядок з причиною)  
- непрогнані рядки матриці Part 3: **S6 live stream**, **S7 GPS/range hardware**  
- повний VisDrone-MOT end-to-end у change (demo/proxy pairs)

---

## Відтворюваність (README-якір)

```text
# Детекція номінал / деградації — протокол Звіту 05 (notebook DZ05)
# ONNX + latency — 09_ONNX_Deploy_…ipynb → pkg09_out/
# VO — 09_VO_TUM_fr1_xyz_…ipynb → dz09_out/ + evo SE3 (-a)
# Change — 07_Change_Detection.ipynb → change_out/
# Надійність stream — configs/stream_fault.yaml + log у ДЗ_Фінал_Тема 12/logs/
```

Версії: Ultralytics YOLOv8n, imgsz=640, Python/Colab, GPU Tesla T4 (заміри 05/09/12).
