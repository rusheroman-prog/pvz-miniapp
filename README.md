# Visa Kapitalbank × Uzum Market — Prize Order Management System

Система управления призовыми заказами на базе Google Apps Script + Telegram Mini App.  
Автоматизирует полный цикл: импорт заказов → сборка на складе → приёмка/выдача на ПВЗ.

---

## Архитектура

```
┌─────────────────────────────────────────────────────────┐
│                    Мастер-таблица                       │
│  MasterTable_v11.gs — импорт, статусы, SMS, Telegram   │
└───────────┬─────────────────────┬───────────────────────┘
            │ syncToSklad()       │ syncToPvz()
            ▼                     ▼
┌───────────────────┐   ┌──────────────────────────────────┐
│   Лист сборки     │   │         Лист ПВЗ                 │
│  Sborka_v4.gs     │   │  PVZ_Code.gs  +  PVZ_WebApp.gs  │
│  win_assembly.html│   │  win_rec/iss/ret.html            │
│  win_box.html     │   │  pvz_miniapp.html (TG Mini App)  │
└───────────────────┘   └──────────────────────────────────┘
            │                     │
            └──────┬──────────────┘
                   │ syncGlobalUpdates()
                   ▼
            Мастер-таблица (обновление статусов)
```

---

## Файлы проекта

### Google Apps Script

| Файл | Таблица | Назначение |
|------|---------|-----------|
| `MasterTable_v11.gs` | Мастер | Импорт заказов из файла KB, синхронизация статусов, SMS, диагностика |
| `PVZ_Code.gs` | ПВЗ | Приёмка, выдача, возврат через модальные окна, отчёт ТУ/РУ |
| `PVZ_WebApp.gs` | ПВЗ | REST API для Telegram Mini App (`doGet`) |
| `Sborka_v4.gs` | Сборка | Триггер сканирования, печать этикеток, меню |
| `Sborka_v4_addon.gs` | Сборка | Бэкенд модального окна сборки заказов |
| `Sborka_box_addon.gs` | Сборка | Бэкенд окна размещения в короб |
| `Sborka_tg_bot.gs` | Сборка | Telegram-уведомления: волна 1 (импорт) + волна 2 (9:30 ежедневно) |

### HTML-интерфейсы (модальные окна Google Sheets)

| Файл | Назначение |
|------|-----------|
| `win_assembly.html` | Окно сборки: список заказов → сканирование SKU → печать этикетки |
| `win_box.html` | Окно размещения в короб: скан заказа → скан короба |
| `win_rec.html` | Окно приёмки на ПВЗ |
| `win_iss.html` | Окно выдачи на ПВЗ |
| `win_ret.html` | Окно возврата на ПВЗ |

### Telegram Mini App

| Файл | Назначение |
|------|-----------|
| `pvz_miniapp.html` | SPA для операторов ПВЗ: приёмка, выдача, возврат, отчёт ТУ/РУ. Хостинг — GitHub Pages |

### Манифесты

| Файл | Назначение |
|------|-----------|
| `appsscript_Master.json` | OAuth-скоупы для Мастер-таблицы |
| `appsscript_Sborka_PVZ.json` | OAuth-скоупы для Сборки и ПВЗ |

---

## Структура данных

### Мастер-таблица (Лист1, A:P)

| Кол. | Поле | Описание |
|------|------|---------|
| A | Телефон | Номер клиента |
| B | Номер заказа | ID из файла KB |
| C | Товар | Название по SKU |
| D | Атрибут 1 | Размер |
| E | Атрибут 2 | Цвет |
| F | Кол-во | Агрегированное qty по SKU |
| G | ПВЗ | Код пункта выдачи |
| H | Маршрут | Номер рейса |
| I | Статус | Главный статус заказа |
| J | Номер короба | Из листа сборки |
| K–P | Статусы ПВЗ | Приёмка, выдача, возврат с датами |

### Лист сборки (Лист1, A:P)

| Кол. | Поле | Описание |
|------|------|---------|
| A–F | Телефон, заказ, товар, атрибуты, qty | Из Мастера |
| G | Факт | Отсканировано единиц |
| H | Статус | `Ожидает сборки` / `Собрано` / `Доставляется` |
| I | Сканер товара | Очищается после скана (триггер `handleEdit`) |
| J | Сканер короба | Очищается после скана |
| K–M | ПВЗ, маршрут, ячейка | Логистика |
| N | SKU | Штрихкод для сканирования |
| O | Номер короба | Постоянное хранение |
| P | Время сборки | Фиксируется при статусе `Собрано` |

### Лист ПВЗ (Лист1, A:P)

| Кол. | Поле | Описание |
|------|------|---------|
| A | Дата создания | Момент импорта из Мастера |
| B | Телефон | |
| C | ID заказа | |
| D | Код ПВЗ | |
| E | План | Суммарное кол-во позиций |
| F–H | Факт/Статус/Дата приёмки | |
| I | Ячейка хранения | Вводится оператором |
| J–L | Факт/Статус/Дата выдачи | |
| M–P | Факт/Статус/Дата/Причина возврата | |

---

## Жизненный цикл заказа

```
KB файл → Мастер (импорт) → Лист сборки + Лист ПВЗ
                                    │
                          Сборщик сканирует SKU
                                    │
                           Статус: Собрано
                                    │
                        Размещение в короб (скан)
                                    │
                           Статус: Доставляется
                                    │
                        ПВЗ принимает короб + ячейка
                                    │
                          Статус: Принят на ПВЗ
                                    │
                           SMS клиенту (Мастер)
                                    │
                        Клиент забирает → Выдан
                           или Возврат
```

---

## Настройка

### 1. Мастер-таблица

```javascript
// MasterTable_v11.gs — вставьте ID своих таблиц
const ID_SKLAD = "ВАШ_ID_ТАБЛИЦЫ_СБОРКИ";
const ID_PVZ   = "ВАШ_ID_ТАБЛИЦЫ_ПВЗ";
const ID_STOCK = "ВАШ_ID_ТАБЛИЦЫ_ОСТАТКОВ";

// Telegram (для уведомлений после импорта)
const TG_TOKEN_MASTER   = "ВАШ_ТОКЕН_БОТА";
const TG_CHAT_ID_MASTER = "ВАШ_CHAT_ID";
const TG_TAGS_MASTER    = ["@username1", "@username2"];
```

**Первый запуск:**
1. Apps Script → вставить `MasterTable_v11.gs`
2. Вставить `appsscript_Master.json` как манифест
3. Меню → ⚙️ Создать служебные листы
4. Меню → 🔌 Проверить соединения

### 2. Лист сборки

```javascript
// Sborka_v4.gs
const ID_PRIEMKA = "ВАШ_ID_ТАБЛИЦЫ_ОСТАТКОВ";

// Sborka_tg_bot.gs — уведомления для сборщиков
const TG_TOKEN   = "ВАШ_ТОКЕН_БОТА";
const TG_CHAT_ID = "ВАШ_CHAT_ID";
const TG_TAGS    = ["@username_сборщика"];
```

**Файлы скриптов для одного проекта Apps Script:**
```
Sborka_v4.gs        ← основной файл
Sborka_v4_addon.gs  ← бэкенд окна сборки
Sborka_box_addon.gs ← бэкенд окна короба
Sborka_tg_bot.gs    ← Telegram-уведомления
```

**HTML-файлы (создать как HTML-файлы в Apps Script):**
```
win_assembly  ← окно сборки
win_box       ← окно размещения в короб
```

**После вставки кода:**
```
1. Запустить updateHeaders()    — добавит заголовки колонок N, O, P
2. Запустить setupTrigger()     — создаст ежедневный триггер 9:30
3. Меню → 🤖 Telegram: тест    — проверить бота
4. Привязать триггер handleEdit → При изменении
```

### 3. Лист ПВЗ

**Файлы для одного проекта Apps Script:**
```
PVZ_Code.gs    ← основной файл (модальные окна)
PVZ_WebApp.gs  ← Web App API для Mini App
```

**HTML-файлы:**
```
win_rec  ← приёмка
win_iss  ← выдача
win_ret  ← возврат
```

**Первый запуск:**
```
1. Меню → ⚙️ Создать листы ТУ/РУ
2. Заполнить Справочник_ТУ: Код ПВЗ → ТУ → РУ
3. Для Mini App: Развернуть → Веб-приложение
   - Выполнять как: Я (владелец)
   - Доступ: Все
   Скопировать URL развёртывания
```

### 4. Telegram Mini App (pvz_miniapp.html)

```javascript
// pvz_miniapp.html — строка ~496
const API_URL = "https://script.google.com/macros/s/.../exec";
//                ↑ URL из шага "Развернуть → Веб-приложение" (Лист ПВЗ)
```

**Деплой на GitHub Pages:**
```bash
# 1. Создать репозиторий
# 2. Загрузить pvz_miniapp.html как index.html
# 3. Settings → Pages → Branch: main → Save
# URL: https://username.github.io/repo-name/
```

**Подключить к боту:**
```
@BotFather → /mybots → выбрать бота →
Bot Settings → Menu Button →
URL: https://username.github.io/repo-name/
Text: Открыть ПВЗ
```

**Роли (лист "Менеджеры_ПВЗ" в таблице ПВЗ):**

| Telegram ID | Роль | Имя |
|-------------|------|-----|
| 123456789 | ТУ | Иванов И. |
| 987654321 | РУ | Петров П. |

> Обычные операторы ПВЗ в таблицу не вносятся — они выбирают ПВЗ при первом входе.

---

## OAuth скоупы (appsscript.json)

### Мастер-таблица
```json
{
  "oauthScopes": [
    "https://www.googleapis.com/auth/spreadsheets",
    "https://www.googleapis.com/auth/script.container.ui",
    "https://www.googleapis.com/auth/script.scriptapp",
    "https://www.googleapis.com/auth/script.external_request"
  ]
}
```

### Сборка и ПВЗ
```json
{
  "oauthScopes": [
    "https://www.googleapis.com/auth/spreadsheets",
    "https://www.googleapis.com/auth/script.container.ui",
    "https://www.googleapis.com/auth/script.external_request",
    "https://www.googleapis.com/auth/script.scriptapp"
  ]
}
```

---

# ПВЗ Mini App — Visa Kapitalbank × Uzum Market

Telegram Mini App для операторов пунктов выдачи заказов (ПВЗ).  
Приёмка, выдача, возврат, поиск заказов и отчётность — прямо из Telegram, без установки приложений.

![Telegram Mini App](https://img.shields.io/badge/Telegram-Mini%20App-2CA5E0?logo=telegram)
![Google Apps Script](https://img.shields.io/badge/Backend-Google%20Apps%20Script-4285F4?logo=google)
![GitHub Pages](https://img.shields.io/badge/Hosting-GitHub%20Pages-181717?logo=github)
![No deps](https://img.shields.io/badge/Dependencies-2%20CDN-brightgreen)

---

## Как это работает

```
Оператор ПВЗ → нажимает "Открыть ПВЗ" в боте
      │
      ▼
pvz_miniapp.html (GitHub Pages)
      │
      │  GET ?action=...&tg_id=...&pvz=...
      ▼
PVZ_WebApp.gs (Google Apps Script Web App)
      │
      ▼
Google Sheets (Лист ПВЗ)
```

**Без серверов, без баз данных, без npm.**  
Весь бэкенд — один файл GAS, данные — Google Sheets.

---

## Экраны

| Экран | ID | Описание |
|-------|-----|---------|
| 🔍 Онбординг | `screen-onboard` | Первый вход: поиск и выбор ПВЗ из списка |
| 🏠 Главное меню | `screen-main` | Кнопки операций, счётчики, смена ПВЗ |
| 📥 Приёмка | `screen-reception` | Скан QR → проверка → ячейка → подтверждение |
| 📤 Выдача | `screen-issuance` | Скан QR → ячейка хранения → подтверждение |
| ↩️ Возврат | `screen-return` | Скан QR → причина → подтверждение |
| 📋 Заказы | `screen-orders` | Поиск + фильтры + аккордеон + быстрые действия |
| 📊 Отчёт | `screen-report` | Статистика и список для ТУ/РУ |

---

## Структура кода

Один HTML-файл, SPA без фреймворков.

```
pvz_miniapp.html
│
├── <style>
│   ├── CSS-переменные Telegram (--tg-theme-*)  — авто тёмная/светлая тема
│   ├── Адаптивный сканер (.scan-wrap, aspect-ratio 16/9, max-height 56vw)
│   ├── Лазерная анимация (.scan-frame::after, @keyframes scanLaser)
│   ├── Карточки заказов (.order-item, .order-item-header, .order-actions)
│   ├── Бейдж ячейки хранения (.cell-badge)
│   └── Кнопки быстрых действий (.action-btn.issue / .return)
│
├── HTML (7 экранов .screen, display:none по умолчанию)
│
└── <script>
    ├── КОНФИГ ──────── API_URL
    ├── СОСТОЯНИЕ ────── объект state
    ├── ИНИЦИАЛИЗАЦИЯ ── init, showOnboard, showMain, loadCounts
    ├── НАВИГАЦИЯ ────── goTo (автофокус + haptic на каждый переход)
    ├── КАМЕРА ──────── toggleCamera, scanLoop (throttle 250мс), stopCamera
    ├── ПРИЁМКА ─────── checkReception, saveReception
    ├── ВЫДАЧА ──────── findIssuance, saveIssuance, resetIssuance
    ├── ВОЗВРАТ ─────── checkReturn, selectReason, clearSelectedReasons, saveReturn
    ├── ЗАКАЗЫ ──────── loadOrders, setOrderTab, applyOrderFilters,
    │                   toggleOrderDetails, quickAction, renderOrders
    ├── ОТЧЁТ ───────── loadReport, filterReport, renderReport
    └── УТИЛИТЫ ─────── api, esc, showResult
```

---

## Объект `state`

```javascript
const state = {
  tgId:           // Telegram user ID (из tg.initDataUnsafe.user.id)
  name:           // Имя из Telegram
  pvzCode:        // Код ПВЗ — localStorage при старте, сервер как источник правды
  role:           // "Оператор" | "ТУ" | "РУ" | "Администратор"
  allPvz:         // Кэш списка ПВЗ для онбординга
  allOrders:      // Кэш заказов — фильтруется локально без доп. запросов
  recRow:         // Номер строки Sheets для текущей приёмки
  issRow:         // Номер строки для выдачи
  retRow:         // Номер строки для возврата
  selectedReason: // Выбранная кнопка-причина возврата
  cameras:        // { rec, iss, ret } — MediaStream объекты активных камер
  rafIds:         // { rec, iss, ret } — RAF ID для явного cancelAnimationFrame
}
```

---

## Ключевые технические решения

### Камера и QR-сканер

**Проблема:** jsQR запускается 60 раз/сек через RAF, что нагревает бюджетные телефоны  
и блокирует нажатия кнопок на складе.

**Решение — throttle 250мс:**

```javascript
let lastScanTime = 0;

function tick(now) {
  state.rafIds[prefix] = requestAnimationFrame(tick); // RAF не прерывается
  if (now - lastScanTime < 250) return;               // jsQR — только 4 раза/сек
  lastScanTime = now;
  // ... jsQR распознавание
}
```

**Освобождение памяти на iOS (WebKit держит последний кадр):**

```javascript
function stopCamera(prefix) {
  cancelAnimationFrame(state.rafIds[prefix]);          // явная отмена RAF
  state.cameras[prefix].getTracks().forEach(t => t.stop());
  document.getElementById(prefix + "Video").srcObject = null; // iOS fix
}
```

**Лазерная анимация** — визуальный индикатор активного сканирования:

```css
.scan-frame::after {
  content: ""; position: absolute; width: 100%; height: 2px;
  background: var(--green); box-shadow: 0 0 8px var(--green);
  animation: scanLaser 2s infinite linear alternate;
}
@keyframes scanLaser {
  0%   { transform: translateY(0); }
  100% { transform: translateY(min(55vw, 180px)); }
}
```

---

### Навигация

`goTo()` выполняет три вещи одновременно:

```javascript
function goTo(name) {
  tg.HapticFeedback.impactOccurred("light"); // тактильная отдача при каждом переходе
  stopAllCameras();                           // освобождает камеру при уходе с экрана
  // показываем нужный экран...
  setTimeout(() => {
    // автофокус — аппаратный Bluetooth/USB сканер работает сразу без тапа
    document.getElementById("recOrderInput")?.focus();
  }, 100);
}
```

---

### Экран «Заказы» (дашборд оператора)

**Комбинированная фильтрация** — таб + текстовый поиск работают одновременно:

```javascript
function applyOrderFilters() {
  const query = document.getElementById("orderSearchInput").value.toLowerCase();
  let filtered = state.allOrders;

  // Фильтр по статусу
  if (currentOrderTab === "stored") filtered = filtered.filter(o => o.status === "На хранении");

  // Поиск по ID заказа или номеру телефона
  if (query) filtered = filtered.filter(o =>
    o.orderId.toLowerCase().includes(query) || o.phone.toLowerCase().includes(query)
  );
  renderOrders(filtered);
}
```

**Аккордеон** — клик раскрывает карточку, остальные закрываются:

```javascript
function toggleOrderDetails(element) {
  tg.HapticFeedback.selectionChanged();
  document.querySelectorAll(".order-item.expanded").forEach(el => {
    if (el !== element) el.classList.remove("expanded");
  });
  element.classList.toggle("expanded");
}
```

**Quick Action** — бесшовный переход от списка к выдаче:

```javascript
function quickAction(action, orderId, event) {
  event.stopPropagation();  // аккордеон не сворачивается
  goTo(action);
  setTimeout(() => {
    document.getElementById("issOrderInput").value = orderId;
    findIssuance();  // автозапуск — оператор сразу видит ячейку
  }, 200);
}
```

**Флоу оператора:**
```
Клиент называет телефон
  → вводим в поиск → видим заказ
  → нажимаем карточку → раскрывается
  → нажимаем «📤 Выдать»
  → переходим в Выдачу с уже подставленным номером
```

---

### Защита от XSS

Все данные из Google Sheets проходят через `esc()` перед вставкой в `innerHTML`:

```javascript
function esc(str) {
  return String(str || "")
    .replace(/&/g, "&amp;").replace(/</g, "&lt;").replace(/>/g, "&gt;")
    .replace(/"/g, "&quot;").replace(/'/g, "&#39;");
}
```

---

## API

Только `GET` — не вызывает CORS preflight.  
`MimeType.TEXT` вместо `JSON` — исключает редирект GAS, при котором теряется `Access-Control-Allow-Origin`.

```
GET {API_URL}?action=check_reception&tg_id=123456&pvz=ТАШ-123&order_id=1087
→ {"ok":true,"data":{"found":true,"row":5,"pvz":"ТАШ-123","plan":3,"phone":"998901234567"}}
```

| `action` | Доп. параметры | Описание |
|----------|---------------|---------|
| `get_pvz_list` | — | Список всех кодов ПВЗ |
| `check_auth` | — | Роль по `tg_id` |
| `check_reception` | `order_id` | Проверка заказа |
| `save_reception` | `order_id`, `d0`=ячейка | Запись приёмки |
| `find_issuance` | `order_id` | Поиск ячейки |
| `save_issuance` | `order_id`, `d1`=row | Запись выдачи |
| `check_return` | `order_id` | Проверка заказа |
| `save_return` | `order_id`, `d0`=причина, `d1`=row | Запись возврата |
| `get_my_orders` | — | Заказы ПВЗ оператора |
| `get_report` | `d0`=filter | Отчёт ТУ/РУ |

---

## Роли

| Роль | Источник | Что видит |
|------|---------|----------|
| `Оператор` | По умолчанию | Только свой ПВЗ, без отчёта |
| `ТУ` | Лист `Менеджеры_ПВЗ` | Свои ПВЗ + отчёт |
| `РУ` | Лист `Менеджеры_ПВЗ` | Весь регион + отчёт |
| `Администратор` | Лист `Менеджеры_ПВЗ` | Всё |

Операторы не вносятся в таблицу — выбирают ПВЗ при первом входе, выбор хранится в `localStorage`.

---

## Быстрый старт

### 1. Бэкенд (Google Apps Script)

```
Apps Script → добавить PVZ_WebApp.gs
→ Развернуть → Новое развёртывание
  Тип: Веб-приложение
  Выполнять как: Я (владелец)
  Доступ: Все
→ Скопировать URL
```

### 2. Фронтенд (GitHub Pages)

```bash
git clone https://github.com/YOUR/pvz-miniapp.git
```

Вставить URL в строку ~496:
```javascript
const API_URL = "https://script.google.com/macros/s/.../exec";
```

```bash
mv pvz_miniapp.html index.html
git add . && git commit -m "init" && git push
# Settings → Pages → Branch: main → Save
```

### 3. Telegram

```
@BotFather → /mybots → Bot Settings → Menu Button
  URL:  https://YOUR.github.io/pvz-miniapp/
  Text: Открыть ПВЗ
```

---

## Зависимости

| Библиотека | CDN | Назначение |
|-----------|-----|-----------|
| [Telegram Web App JS](https://telegram.org/js/telegram-web-app.js) | Telegram | Интеграция, тема, HapticFeedback |
| [jsQR 1.4.0](https://unpkg.com/jsqr@1.4.0/dist/jsQR.js) | unpkg | Распознавание QR с камеры |

---

## Совместимость

| Среда | Камера | Аппаратный сканер |
|-------|--------|------------------|
| Telegram Android | ✅ | ✅ (автофокус) |
| Telegram iOS | ✅ | ✅ |
| Telegram Desktop | ✅ | ✅ |
| Браузер (отладка) | ✅ | ✅ |

---

## Диагностика

**Проверить API в браузере:**
```
{API_URL}?action=check_auth&tg_id=test&pvz=
→ {"ok":true,"data":{"role":"Оператор","name":"","isManager":false}}
```

| Ошибка | Причина | Решение |
|--------|---------|---------|
| CORS blocked | Старое развёртывание GAS | Создать **новое** развёртывание, обновить `API_URL` |
| Пустой ответ | GAS упал, вернул HTML | Открыть URL в браузере, прочитать ошибку GAS |
| Камера не работает | Нет разрешений Telegram | Настройки телефона → Telegram → Разрешения → Камера |
| `jsQR is not defined` | CDN недоступен | Проверить доступность `unpkg.com` |
| Телефон греется | Нет throttle | Убедиться что `lastScanTime` есть в `scanLoop` |
