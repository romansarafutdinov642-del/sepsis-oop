# Sepsis & AKI Research

Локальный учебно-исследовательский прототип расчёта риска сепсиса и ОПП.
По умолчанию: **DEMO MODEL · NOT FOR CLINICAL USE**. Прогнозы синтетические,
пороги не валидированы; приложение не является медицинским изделием.

## Запуск Docker

```sh
docker compose up --build
```

Интерфейс: http://localhost:8080 · API: http://localhost:8000 · Swagger: http://localhost:8000/docs

Первый запуск создаёт схему и 120 синтетических пациентов. Данные PostgreSQL
сохраняются в Docker volume, снимки — в `storage/xrays/`.

## Локальный запуск в Windows без Docker

Python 3.12+, PostgreSQL 17. Все файлы и кластер находятся в проекте.

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements-dev.txt
.\scripts\start_local.ps1
```

В текущей папке окружение уже установлено, достаточно последней команды.
Интерфейс: **http://127.0.0.1:8000/**, Swagger: http://127.0.0.1:8000/docs.
Скрипт использует PostgreSQL из `C:\Program Files\PostgreSQL\17\bin`;
другой путь передаётся через `-PostgresBin`.
База `research`, порт **55432**; существующие пользовательские базы не изменяются.
Перед созданием БД и миграциями скрипт проверяет, что порт занят именно кластером
из `.local/pgdata`; при обнаружении другого кластера запуск прекращается.
Ctrl+C останавливает API; `.\scripts\stop_local.ps1` останавливает проектный PostgreSQL.
Если PowerShell блокирует локальные скрипты, запустите:

```powershell
powershell -ExecutionPolicy Bypass -File .\scripts\start_local.ps1
```

Это меняет политику только для запускаемого процесса. Пароль не требуется для
изолированного локального demo-кластера; не публикуйте его в сеть.

## Что попробовать

1. Обзор → «Создать пациента»: исследовательский ID, возраст, пол.
2. В карточке нажать «Пример данных» либо заполнить показатели и единицы вручную.
3. Задать время наблюдения с учётом местного часового пояса.
4. При необходимости добавить JPG/PNG и время его получения <= наблюдения.
5. «Рассчитать и сохранить» — оба риска и опциональный multimodal сохраняются в БД.
6. «+ Наблюдение» — новые данные и другое время; появятся новая точка графика и история.

LOW/MEDIUM/HIGH, горизонты и диапазоны задаются в YAML на backend. Снимок в
демонстрационном режиме не влияет на риск; UI об этом сообщает.
Во время сохранения переключение пациента заблокировано. Если расчёт завершился
ошибкой после сохранения наблюдения, повторное нажатие завершает расчёт для того
же наблюдения без дублирования данных.

## Проверки

При запущенном проектном PostgreSQL:

```powershell
.\.venv\Scripts\python.exe -m pytest -q
.\.venv\Scripts\python.exe -m alembic check
node --check frontend/app.js
```

Браузерная проверка при работающем приложении:

```powershell
$env:PLAYWRIGHT_BROWSERS_PATH = "$PWD\.local\browsers"
.\.venv\Scripts\python.exe -m playwright install chromium
.\.venv\Scripts\python.exe -m scripts.browser_smoke
```

Скриншоты: `test-results/`. Pytest использует временную схему PostgreSQL и rollback;
browser smoke создаёт отдельного синтетического пациента `E2E-*`.
`TEST_DATABASE_URL` позволяет задать отдельную PostgreSQL для тестов.

## Данные, модели, документация

- [Архитектура](docs/ARCHITECTURE.md) и [БД](docs/DATABASE.md).
- [Доступ MIMIC, скачивание и ETL](docs/DATASETS.md).
- [Labels, training, SHAP, DenseNet, fusion и serving](docs/ML_PIPELINE.md).
- [API](docs/API.md), [исследовательский статус](docs/MEDICAL_DISCLAIMER.md).
- [Фактический отчёт о проверке](docs/VALIDATION.md).

MIMIC-IV 3.1 и MIMIC-CXR 2.1.0 не скачаны: PhysioNet требует подтверждённого
доступа. Есть официальные downloader-скрипты, demo generator, ETL и обучающий
конвейер. Реальных клинических метрик нет. Default API — **MOCK MODEL**.

`.env.example` содержит шаблон; Compose читает `.env`. Python без Compose читает
переменные процесса, не загружает `.env` автоматически. Пути defaults относительны
корня проекта. Реальный ML — отдельные зависимости `requirements-ml.txt`;
`INCLUDE_ML=true` добавляет их при Docker build.
В Compose веса DenseNet кэшируются в отдельном volume `torch_cache`; каталог
артефактов моделей подключён только для чтения. При первом извлечении embedding
требуется доступ к официальному источнику весов либо заранее заполненный кэш.

GitHub и удалённые Git-репозитории для работы приложения не используются.
