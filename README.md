# YOLO Object Detection learning

Универсальный проект для обучения и запуска YOLO-моделей на любом датасете.
Пример задачи: обнаружение кота, миски и сухого корма на фотографиях и видео.

## Пример Классов

| ID | Название       | Описание           |
|----|----------------|--------------------|
| 0  | `cat`          | Кот                |
| 1  | `cat_bowl`     | Миска              |
| 2  | `dry_cat_food` | Сухой кошачий корм |

## Структура проекта

```
├── train/
│   ├── images/       # Обучающие фотографии
│   └── labels/       # Разметка в формате YOLOv8
├── valid/
│   ├── images/       # Валидационные фотографии
│   └── labels/
├── test/
│   ├── images/       # Тестовые фотографии
│   └── labels/
├── runs/             # Результаты обучения (веса, графики, метрики)
├── data.yaml         # Конфигурация датасета
├── yolov8n.pt        # Предобученная базовая модель
└── best_dry_cat_food_v8.onnx  # Экспортированная модель (ONNX)
```

## Датасет

- Источник: [Roboflow](https://roboflow.com), датасет `cat_feed v3`
- Лицензия: CC BY 4.0
- Формат меток: YOLOv8 (полигоны, нормализованные координаты)
- Рекомендуемое разбиение: **70 / 20 / 10** (train / valid / test) со **Stratified split** — чтобы все 3 класса присутствовали в каждой части

> **Важно:** при экспорте из Roboflow выбирай стратегию разбивки **Stratified**, иначе редкие классы могут выпасть из valid/test и модель будет деградировать при обучении.

## Установка

```bash
pip install ultralytics
```

## Обучение

```bash
yolo detect train model=yolov8n.pt data=data.yaml epochs=100 batch=8 imgsz=640 device=cpu
```

Параметры:
- `model` — базовая модель (`yolov8n.pt` — nano, самая лёгкая; `yolov8s.pt` — small, точнее)
- `epochs` — количество эпох; для маленького датасета достаточно 100
- `batch` — размер батча; при CPU уменьши до 4–8 если не хватает памяти
- `device` — `cpu` или `0` для GPU

```bash
# Другая модель
yolo detect train model=yolov8s.pt data=data.yaml epochs=100 batch=4

# На GPU
yolo detect train model=yolov8n.pt data=data.yaml epochs=100 batch=16 device=0

# Другой датасет
yolo detect train model=yolov8n.pt data=/path/to/other/data.yaml epochs=100 batch=8

# Возобновить прерванное обучение
yolo detect train resume model=runs/detect/train/weights/last.pt
```

После обучения веса сохраняются в `runs/detect/trainN/weights/`:
- `best.pt` — лучшая модель по метрике mAP
- `last.pt` — последний чекпоинт

Доступные модели (скачиваются автоматически): `yolov8n.pt`, `yolov8s.pt`, `yolov8m.pt`, `yolov8l.pt`, `yolov8x.pt`, `yolo11n.pt` и др.

## Проверка (инференс)

```bash
yolo detect predict model=runs/detect/train/weights/best.pt source=photo.jpg
```

```bash
# На папке с фото
yolo detect predict model=runs/detect/train/weights/best.pt source=test/images/

# Снизить порог уверенности (если модель не находит объекты)
yolo detect predict model=runs/detect/train/weights/best.pt source=photo.jpg conf=0.1

# Использовать ONNX-модель
yolo detect predict model=best_dry_cat_food_v8.onnx source=photo.jpg
```

Результаты сохраняются в `runs/detect/predictN/`.

Параметр `conf` — порог уверенности (0.25 по умолчанию). Если модель не находит объекты — попробуй снизить до 0.1.

## Оценка качества (метрики)

```bash
yolo detect val model=runs/detect/train/weights/best.pt data=data.yaml
```

Выводит precision, recall, mAP50, mAP50-95 по каждому классу.

Метрики по эпохам хранятся в `runs/detect/trainN/results.csv`.

## Экспорт в ONNX

ONNX-формат позволяет запускать модель без PyTorch — в браузере, на мобильных устройствах, в C++ и других средах.

```bash
yolo export model=runs/detect/train/weights/best.pt format=onnx
```

Готовый файл `best_dry_cat_food_v8.onnx` уже есть в корне проекта.

## Типичные проблемы

**Модель плохо распознаёт объекты после добавления новых фото**
— Проверь, что все 3 класса есть в valid и test наборах:
```bash
cat valid/labels/*.txt | awk '{print $1}' | sort | uniq -c
```
Если класс 2 отсутствует — переэкспортируй датасет из Roboflow со Stratified split.

**Низкий recall (модель не находит объекты)**
— Снизь порог `conf` при инференсе или добери больше обучающих фотографий.

**Обучение очень медленное**
— Добавь GPU: `device=0`. На CPU YOLOv8n тренируется ~25 секунд на эпоху при данном датасете.
