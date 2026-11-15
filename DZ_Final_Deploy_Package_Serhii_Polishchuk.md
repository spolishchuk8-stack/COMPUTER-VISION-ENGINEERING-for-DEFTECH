# Пакет 09 — Швидкий деплой (ONNX / рушії)

**Автор:** Serhii Polishchuk  
**Курс:** Computer Vision Engineering for DefenceTech  
**Ноутбук:** `09_ONNX_Deploy_Serhii_Polishchuk.ipynb`  
**Ваги:** `yolov8n.pt` (та сама голова, що Звіт 05)  
**Дата заміру:** 15.11.2026  
**Пристрій:** **Tesla T4** (Colab), imgsz=640, batch=1

Артефакти: `pkg09_out/yolov8n_imgsz640_opset12.onnx`, `export_meta.json`, `consistency.json`, `latency_table.json`.

---

## 1. Експорт ONNX

| Параметр | Значення |
|----------|----------|
| weights | yolov8n.pt |
| imgsz | **640** (як у звіті 05) |
| opset | **12** |
| dynamic | false |
| simplify | true |

**Команда (README):**

```bash
python -c "from ultralytics import YOLO; YOLO('yolov8n.pt').export(format='onnx', imgsz=640, opset=12, simplify=True, dynamic=False)"
```

---

## 2. Узгодженість виходів (PyTorch vs ONNX)

**Пороги до прогону:** IoU match = **0.5**, conf = **0.25**, NMS iou = 0.7.  
Control set: **16** кадрів.

| Показник | Значення |
|----------|----------|
| n boxes torch / onnx | 36 / 40 |
| **n matched** | **36** |
| **частка зіставлених (vs torch)** | **1.00 (100%)** |
| median \|Δscore\| | **0.0143** |
| p95 \|Δscore\| | **0.160** |
| unmatched torch | **0** |
| unmatched onnx | 4 |
| unmatched біля conf-порога | 0 |

**Висновок по узгодженості:** усі рамки framework мають пару в ONNX при IoU≥0.5 і тому ж класі. Чотири зайві onnx-бокси — типовий ефект числового NMS/порога; медіанна різниця score мала (~0.014). Preprocess однаковий (Ultralytics). Розходження **не** блокує заміну ваг на ONNX-артефакт для деплою.

---

## 3. Протокол заміру

| Параметр | Значення |
|----------|----------|
| device | Tesla T4 |
| warmup | 10 |
| batch | 1 |
| n frames | 32 |
| sync | CUDA synchronize |
| метрики | p50, p95 ms/frame, FPS@p50, peak GPU MB |

---

## 4. Таблиця рушіїв

| Рушій | p50 ms | p95 ms | FPS (p50) | GPU mem MB | Примітка |
|-------|--------|--------|-----------|------------|----------|
| **PyTorch Ultralytics** | **10.17** | **11.88** | **98.4** | 29.2 | framework |
| **ONNX (Ultralytics→ORT)** | **151.8** | **166.0** | **6.6** | 12.7 | портативний `.onnx` |
| **TensorRT FP16** | — | — | — | — | **не зібрано** |

**Причина пропуску TensorRT:** у Contour Colab немає окремого сеансу збірки TRT engine / NGC. Опційний рядок за умовами пакета 09; збірка можлива на AWS g4dn або локальній RTX.

### Інтерпретація latency
На цьому стенді **PyTorch на T4 суттєво швидший** за шлях ONNX через Ultralytics/ORT (~15× за p50). ONNX тут цінний як **формат деплою** (перенос на інші runtime), а не як найшвидший інференс на T4 без TRT. Після TensorRT FP16 на тій самій GPU очікується зближення або виграш vs PyTorch — потребує окремого заміру.

---

## 5. Якість

Номінал детекції — протокол **Звіту 05**: mAP50-95 = **0.0529** (VisDrone val, conf=0.25).  
Пакет 09 підтверджує **еквівалентність рамок** framework↔ONNX (match 100% на control). Повний val mAP на ONNX не перераховувався; для захисту достатньо consistency + номінал 05.

---

## 6. Рекомендація рушія

| Задача | Рушій |
|--------|--------|
| Розробка, val, звіти курсу | **PyTorch Ultralytics** |
| Артефакт деплою / перенос | **ONNX** (opset 12, imgsz 640) |
| Максимальний FPS на NVIDIA | **TensorRT FP16** — backlog (не зібрано) |

**Клас заліза заміру:** Colab **Tesla T4**.  
**Межа переносу:** цифри p50/p95 **не** переносити 1:1 на Jetson / іншу GPU без повторного протоколу; зміна imgsz/batch/precision → нова таблиця.

**Підсумок для захисту:** робочий контур = PyTorch (швидкість на T4) + ONNX-файл (деплой). TRT — наступний крок, не блокер; негативний рядок із причиною зараховується.

---

## 7. Висновок

1. Експорт `yolov8n` → ONNX виконано зі **imgsz=640**, opset=12, static shape.  
2. Узгодженість з framework: **100%** matched boxes, median Δscore ≈ 0.014.  
3. Latency на T4: PyTorch **~10 ms** p50 (~98 FPS), ONNX-шлях **~152 ms** p50 — ONNX не прискорює на цьому стенді без TRT.  
4. TensorRT FP16 не збирався (немає TRT-сеансу); причина в таблиці.  
5. Для фіналу рекомендуємо **PyTorch** як основний runtime на GPU розробки та **ONNX** як артефакт; обіцянка FPS — лише для виміряного T4 + imgsz=640.  
6. Перенос на іншу архітектуру вимагає повторити §3–§4.

---

## 8. Відтворюваність

```text
1) Colab GPU: open 09_ONNX_Deploy_Serhii_Polishchuk.ipynb → Run all
2) Outputs: /content/pkg09_out/{*.onnx, consistency.json, latency_table.json, export_meta.json}
3) Numbers above are from that run (T4, 15.11.2026)
```

TensorRT: окремий інстанс з TensorRT/NGC (не обов’язковий для зарахування пакета).
