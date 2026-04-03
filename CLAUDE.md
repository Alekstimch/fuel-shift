# FuelShift MVP — CLAUDE.md

Рабочий контекст для Claude Code. Читай этот файл перед любой правкой кода.

---

## Что это за проект

Система автоматизированной проверки перерасхода топлива по вахтам.

**Стек:** Google Sheets + Google Apps Script + Telegram Mini App (один HTML-файл) + Telegram Bot.  
**Сервера нет.** Всё держится на Apps Script как бэкенд и GitHub Pages как хостинг для HTML.

**Бизнес-цель:** сократить ручной анализ одной вахты с 40–60 минут до нескольких минут.

---

## Ключевое правило расчёта

Система определяет **необъяснимую недостачу**, а не кражу.

```
Отклонение = Топливо нач. вахты
           + Заправки по картам
           + Входящие переливы (только подтверждённые)
           - Исходящие переливы (только подтверждённые)
           - Факт. расход по мониторингу
           - Топливо кон. вахты
```

Расход по дню из мониторинга: `нач. + заправлено - кон.`

Порог недостачи: `SHORTAGE_THRESHOLD = 10` литров (в `constants.gs`).

---

## Архитектура файлов Apps Script

```
constants.gs      — все константы, статусы, HEADERS-объекты
utils.gs          — parseDate, parseNum, formatDate, loadShifts, findShift*, normalizeGovNum, normalizePhone, getSheetCols
setup.gs          — первичная инициализация листов
formatting.gs     — форматирование таблицы, шапки, формулы
importMonitoring.gs — импорт и привязка данных мониторинга
importCards.gs    — импорт и привязка данных топливных карт
calculations.gs   — пересчёт итогов по вахтам
main.gs           — doGet, doPost, все API-функции, dashboard API
telegram.gs       — отправка уведомлений через бота
```

**Правило:** никогда не обращаться к столбцам по числовым индексам напрямую. Всегда использовать `getSheetCols(sheet, HEADERS_OBJECT)`.

---

## Соглашение по колонкам — критически важно

В проекте **два слоя констант:**

### 1. `*_HEADERS` — названия заголовков (строки)
Используются для поиска столбца по имени через `getSheetCols()`.

```javascript
const TRANSFER_HEADERS = {
  SENDER_NAME:   'Отправитель',
  STATUS:        'Статус заявки',
  // ...
};
```

### 2. `RESULT_COL` — числовые индексы (0-based)
Исключение: лист «Итоги» читается по фиксированным индексам в `_loadResultsMap()`.

```javascript
const RESULT_COL = {
  SHIFT_ID: 0,
  STATUS:   11,
  // ...
};
```

### Как правильно читать данные из листа:

```javascript
// ПРАВИЛЬНО
const cols = getSheetCols(sheet, MON_HEADERS);
const flag = row[cols.FLAG];

// НЕПРАВИЛЬНО — никогда так не делать
const flag = row[19]; // хрупко, сломается при добавлении столбца
const flag = row[MON_COL.FLAG]; // MON_COL не существует в текущей версии
```

---

## Названия листов (из `SHEETS` в constants.gs)

| Константа | Название листа |
|-----------|---------------|
| `SHEETS.DRIVERS` | Водители |
| `SHEETS.CARS` | Автомобили |
| `SHEETS.SHIFTS` | Вахты |
| `SHEETS.CARDS` | Топливные карты |
| `SHEETS.MONITORING` | Мониторинг |
| `SHEETS.TRANSFERS` | Переливы |
| `SHEETS.RESULTS` | Итоги |
| `SHEETS.LOG` | Лог |

---

## Статусы

```javascript
// Вахта
SHIFT_STATUS.ACTIVE   = 'Активна'
SHIFT_STATUS.OK       = 'Норма'
SHIFT_STATUS.CHECK    = 'Требует проверки'
SHIFT_STATUS.SHORTAGE = 'Недостача'

// Перелив
TRANSFER_STATUS.CREATED   = 'Создан'
TRANSFER_STATUS.CONFIRMED = 'Подтверждён'
TRANSFER_STATUS.ADJUSTED  = 'Скорректирован'
TRANSFER_STATUS.REJECTED  = 'Отклонён'

// Строки мониторинга и карт
ROW_STATUS.OK       = 'ОК'
ROW_STATUS.UNLINKED = 'Не привязано'
ROW_STATUS.ERROR    = 'Ошибка'
```

---

## API endpoints (doGet)

| action | функция | параметры |
|--------|---------|-----------|
| `getDrivers` | `apiGetDrivers()` | — |
| `getTransfers` | `apiGetTransfers()` | `driverId` |
| `getOutgoingTransfers` | `apiGetOutgoingTransfers()` | `driverId` |
| `getDashboard` | `apiGetDashboard()` | — |
| `getShiftDetail` | `apiGetShiftDetail()` | `shiftId` |
| `getDataQuality` | `apiGetDataQuality()` | — |

## API endpoints (doPost)

| action | функция |
|--------|---------|
| `createTransfer` | `apiCreateTransfer(body)` |
| `respondTransfer` | `apiRespondTransfer(body)` |
| `logError` | `apiLogError(body)` |

---

## Привязка данных к вахтам

**Мониторинг** → привязывается по госномеру + дата попадает в период вахты.  
**Карты** → привязывается по номеру топливной карты (из листа «Автомобили», колонка «Номер топливной карты») + дата попадает в период вахты.  
**Переливы** → привязываются по `shiftId` отправителя (активная вахта на момент создания).

Функции привязки в `utils.gs`:
- `findShiftByCarAndDate(shifts, govNum, date)`
- `findShiftByCardAndDate(shifts, cardNum, date)`

---

## Переливы — важные правила

- В расчёт попадают только `Подтверждён` и `Скорректирован`
- `Создан` и `Отклонён` — игнорируются при пересчёте вахты
- Уведомление получателю отправляется сразу при создании через `telegram.gs`
- ID перелива: `TRF_<timestamp>_<random>` — генерируется через `genId('TRF')`

---

## ID водителя

ID водителя = его номер телефона (последние 10 цифр).  
Нормализация: `normalizePhone(phone)` → берёт последние 10 цифр.  
Авторизация в Mini App: по нику Telegram (`d.tg` без `@`, строчными).

---

## HTML-файлы (хостинг на GitHub Pages)

```
miniapp.html   — Telegram Mini App для водителей (переливы)
dashboard.html — дашборд руководителя
```

**API_URL** в обоих файлах должен указывать на текущее развёртывание Apps Script.

### miniapp.html — три вкладки:
1. **Создать** — форма создания перелива
2. **Входящие** — переливы на подтверждение
3. **Мои** — исходящие переливы (история)

### Фикс клавиатуры:
Используется `window.visualViewport` API. Кнопка отправки фиксирована внизу через `position: fixed` + `translateY` по событию `resize`/`scroll` вьюпорта.

---

## Что не входит в MVP (не реализовывать)

- Смена автомобиля внутри одной вахты
- Роли пользователей и разграничение прав
- Reconciliation каждой заправки карты с каждым событием мониторинга
- Учёт наличных заправок
- Автоматический расчёт стоимости удержания
- API-интеграции с внешними системами

---

## Правила при правке кода

1. **Не трогать** `getSheetCols()` и `loadShifts()` без крайней необходимости — от них зависит весь проект.
2. **После любой правки** `main.gs` или других `.gs` файлов — нужно новое развёртывание Apps Script (Развернуть → Управление → Новая версия).
3. **Идемпотентность** — повторный запуск `recalculateShifts()` не должен ломать данные.
4. **Логировать** все ключевые операции через `Logger.log()`.
5. **Помечать флагами** проблемные строки — никогда не пропускать молча.
6. **Числовые индексы** — только для `RESULT_COL` в `_loadResultsMap()`. Везде остальное — через `getSheetCols()`.
