# Note 07 — Domain shift RGB → IR (VisDrone → HIT-UAV)

**Курс:** Computer Vision Engineering for DefenceTech  
**Автор:** Serhii Polishchuk  
**Дата:** 10.11.2026  
**Ноутбук:** `07_Domain_Shift_HIT_UAV.ipynb`  
**Конфіг мапінгів:** `configs/mappings.yaml`  
**HIT-UAV:** v1.2.1 (Zenodo), split train/val/test = 2029 / 290 / 579

---

## 1. Постановка

Детектор VisDrone (RGB) оцінюється на HIT-UAV (IR). Перевіряємо domain shift і три гіпотези:

| Гіпотеза | Інтервенція |
|----------|-------------|
| Зсув = відсутність кольору | **A** — fine-tune на grayscale VisDrone |
| Достатньо малої in-domain вибірки | **B** — fine-tune на HIT train subset |
| Grayscale-pretrain дає виграш над B | **A+B** — HIT зі старту з ваг A |

Порядок: **zero → A → B → A+B**. Один evaluator (imgsz / conf / iou).

---

## 2. Мапінги (зафіксовані до запусків)

### Оцінювання (VisDrone pred → 3 класи HIT)

| HIT-UAV | VisDrone class id |
|---------|-------------------|
| Person | 0 pedestrian + 1 people |
| Car | 3 car |
| Bicycle | 2 bicycle |

van/truck/bus/motor… у метрики трьох класів не входять. DontCare не в GT.

### Навчання (HIT → YOLO class_id у мітках)

| HIT | YOLO id | Примітка names у yaml |
|-----|---------|------------------------|
| Person | **0** | person |
| Car | **3** | COCO-слот motorcycle* |
| Bicycle | **2** | COCO-слот car* |
| OtherVehicle, DontCare | не пишемо | — |

\*У yaml залишено names як у ДЗ1/COCO, щоб не змінювалась голова. У звіті id2/id3 = Bicycle/Car.

---

## 3. Протокол

| Параметр | Значення | Примітка |
|----------|----------|----------|
| Старт zero/A/B | yolov8n.pt | checkpoint ДЗ1 недоступний у сесії |
| imgsz | 416 | CPU-only (ліміт Colab GPU) |
| conf / iou | 0.25 / 0.7 | однакові для всіх |
| A subset | 80 grayscale VisDrone | seed=42 |
| B / A+B subset | 100 HIT train | **той самий** список, seed=42 |
| epochs | 5 / 5 / 5 | CPU |
| device | CPU | GPU T4 недоступний |

Артефакти: `dz07_out/weights/{A,B,AB}_best.pt`, `metrics_all.csv`, yaml, lists.

---

## 4. Результати на HIT val (290 зображень)

| run | mAP50 | mAP50-95 | Precision | Recall |
|-----|-------|----------|-----------|--------|
| **zero** | 0.0007 | 0.0003 | 0.064 | 0.001 |
| **A** | 0.0021 | 0.0008 | 0.385 | 0.002 |
| **B** | 0.1048 | 0.0411 | 0.329 | 0.143 |
| **A+B** | **0.2267** | **0.1074** | **0.360** | **0.262** |

### Класи (якісно)
- **Person** дрібний (~10–15 px) — найчутливіший до зсуву.
- Zero майже не активує Car/Bicycle; після B/A+B з’являється recall.
- У логах Ultralytics: id2=`car` (=Bicycle), id3=`motorcycle` (=Car) через COCO names.

---

## 5. Гіпотези

1. **Найбільший зсув:** zero-shot ≈ 0 на всіх класах → повний gap RGB→IR (спектр/контраст/текстура), не лише розмір об’єктів.

2. **A («лише колір»):** **не підтверджена**. mAP50 0.002 vs 0.001 — grayscale VisDrone майже не закриває IR-зсув.

3. **B:** **так**, головний важіль. mAP50 ~0 → **0.10** на 100 кадрах HIT.

4. **A+B vs B:** **стабільний виграш** 0.23 vs 0.10 mAP50. Grayscale-pretrain корисний як ініціалізація.

5. **Ваги для теми 6:** **A+B** (`AB_best.pt`) — найкращий mAP50 і recall на HIT val.

6. **Дрібні об’єкти:** Person лишається найскладнішим; B/A+B підвищують recall, повне закриття gap потребує більше даних/епох і GPU.

---

## 6. Обмеження

- Train на CPU, 5 епох, subset 80–100 — абсолютні mAP занижені, але порядок **zero < A ≪ B < A+B** стійкий.
- Старт з yolov8n, не з власного best ДЗ1.
- Офіційний HIT val; train subset seed=42.
- Names COCO у yaml (сумісність голови); remap задокументований.

---

## 7. Висновок

Domain shift RGB→IR на HIT-UAV у zero-shot **критичний**. Адаптація «тільки grayscale» майже не працює. Найефективніше — короткий fine-tune на HIT; **A+B** дає додатковий виграш над B. Для теми 6 рекомендуються ваги **A+B**.
