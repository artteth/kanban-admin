# 📚 Kanban Board v2 - Полное Руководство Разработчика

## 📋 Содержание

1. [Архитектура системы](#архитектура-системы)
2. [Настройка Google Apps Script](#настройка-google-apps-script)
3. [API: Чтение и запись данных](#api-чтение-и-запись-данных)
4. [Обход CORS через JSONP](#обход-cors-через-jsonp)
5. [Структура Google Таблицы](#структура-google-таблицы)
6. [Синхронизация данных](#синхронизация-данных)
7. [Известные проблемы и решения](#известные-проблемы-и-решения)

---

## 🏗 Архитектура системы

```
┌─────────────────────────────────────────────────────────────┐
│                     Frontend (HTML/JS)                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │ index.html  │  │ admin.html  │  │ select_user.html    │ │
│  │ (пользов.)  │  │ (админка)   │  │ (выбор пользоват.)  │ │
│  └──────┬──────┘  └──────┬──────┘  └──────────┬──────────┘ │
│         │                │                     │            │
│         └────────────────┴─────────────────────┘            │
│                          │                                  │
│                   JSONP API                                 │
└──────────────────────────┼──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│              Google Apps Script (Backend)                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ cod_v2.gs                                           │   │
│  │ - doGet()  - обработка GET запросов (JSONP)         │   │
│  │ - doPost() - обработка POST запросов (TG-API)       │   │
│  │ - getTasks(), getUsers() - чтение данных            │   │
│  │ - addTask(), updateTask() - запись данных           │   │
│  └─────────────────────────────────────────────────────┘   │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                  Google Таблица                             │
│  ┌──────────────┐  ┌──────────────┐                        │
│  │ Лист: Задания│  │ Лист: Пользо-│                        │
│  │ (8 колонок)  │  │ ватели (4)   │                        │
│  └──────────────┘  └──────────────┘                        │
└─────────────────────────────────────────────────────────────┘
```

---

## ⚙️ Настройка Google Apps Script

### 1. Развёртывание кода

**Шаги:**

1. Открой https://script.google.com/
2. Создай новый проект или открой существующий
3. Скопируй код из `cod_v2.gs` в редактор
4. Нажми **Сохранить** 💾

### 2. Настройка развёртывания

**Критически важные параметры:**

```
Deploy → New deployment:
  ├─ Type: Web app
  ├─ Execute as: Me (ваш Google аккаунт)
  └─ Who has access: Anyone ⚠️ (обязательно!)
```

**Почему "Anyone":**
- Позволяет делать запросы без CORS preflight
- JSONP работает только с публичным доступом
- Без этого будет ошибка 405

### 3. Получение API URL

После развёртывания скопируй URL вида:
```
https://script.google.com/macros/s/AKfycbw.../exec
```

Обнови этот URL во всех HTML файлах (строка ~1120):
```javascript
const API_URL = 'https://script.google.com/macros/s/YOUR_ID/exec';
```

### 4. Настройка Google Таблицы

**ID таблицы** (из `cod_v2.gs` строка 5):
```javascript
const SPREADSHEET_ID = '1TsUjce91h44W_PF4dzCqCwGTB_jqhjJxRWBsLiGPmjE';
```

**Название листов** (критично!):
- `Задания` — с большой буквы! (строка 6)
- `Пользователи` — с большой буквы! (строка 7)

---

## 📡 API: Чтение и запись данных

### Методы API

| Метод | Описание | Параметры |
|-------|----------|-----------|
| `getTasks` | Получить задачи | `userId` (опционально) |
| `getUsers` | Получить пользователей | — |
| `addTask` | Создать задачу | `title`, `plannedDate`, `assignedTo` |
| `updateTask` | Обновить задачу | `id`, `title`, `status`, `plannedDate`, `assignedTo` |
| `updateStatus` | Изменить статус | `id`, `status` |
| `deleteTask` | Удалить задачу | `id` |
| `addUser` | Добавить пользователя | `name`, `telegramId`, `role` |
| `updateUser` | Обновить пользователя | `userId`, `name`, `telegramId`, `role` |
| `deleteUser` | Удалить пользователя | `userId` |

### Чтение данных (GET / JSONP)

**Формат запроса:**
```javascript
// Через JSONP (основной способ)
apiJsonp('getTasks', { userId: 'user_123' })
  .then(data => console.log(data));

// URL выглядит так:
// API_URL?mode=jsonp&action=getTasks&userId=user_123&callback=jsonp_callback_...
```

**Формат ответа:**
```json
{
  "ok": true,
  "data": [
    {
      "id": "task_1773140608448_x1y2z3",
      "title": "Название задачи",
      "status": "todo",
      "startDate": "2026-03-10",
      "endDate": "",
      "duration": "",
      "plannedDate": "2026-03-15",
      "assignedTo": "user_123"
    }
  ]
}
```

### Запись данных (POST / JSONP)

**Через JSONP (GET с параметрами):**
```javascript
// Создание задачи
apiJsonp('addTask', {
  title: 'Новая задача',
  plannedDate: '2026-03-15',
  assignedTo: 'user_123'
});

// URL:
// API_URL?mode=jsonp&action=addTask&title=Новая+задача&plannedDate=2026-03-15&assignedTo=user_123&callback=...
```

**Обновление задачи:**
```javascript
apiJsonp('updateTask', {
  id: 'task_123',
  title: 'Обновлённое название',
  status: 'in-progress',
  plannedDate: '2026-03-20',
  assignedTo: 'user_456'
});
```

---

## 🔄 Обход CORS через JSONP

### Проблема

Google Apps Script **не поддерживает CORS preflight** (OPTIONS запрос).
При использовании `fetch` с `Content-Type: application/json` браузер отправляет OPTIONS запрос, на который GAS отвечает ошибкой 405.

### Решение: JSONP

**JSONP (JSON with Padding)** — старый способ обхода CORS, использующий `<script>` теги.

**Как работает:**

1. Создаётся `<script>` элемент с `src=API_URL?callback=myCallback&...`
2. Браузер загружает скрипт (CORS не применяется к `<script>`)
3. Сервер возвращает: `myCallback({"ok":true,"data":[...]})`
4. Функция `myCallback` выполняется с данными

**Реализация:**

```javascript
function apiJsonp(action, params = {}) {
  return new Promise((resolve, reject) => {
    const callbackName = 'jsonp_callback_' + Date.now();
    
    // Формируем URL с параметрами
    let urlParams = `mode=jsonp&action=${encodeURIComponent(action)}&callback=${callbackName}`;
    for (const key in params) {
      if (params[key] !== undefined && params[key] !== '') {
        urlParams += `&${encodeURIComponent(key)}=${encodeURIComponent(params[key])}`;
      }
    }
    const url = `${API_URL}?${urlParams}`;

    // Создаём script элемент
    const script = document.createElement('script');
    script.src = url;
    script.onerror = () => reject(new Error('JSONP request failed'));

    // Обработчик ответа
    window[callbackName] = (response) => {
      if (response && response.ok) {
        resolve(response.data || response);
      } else {
        reject(new Error(response?.error || 'API error'));
      }
      delete window[callbackName];
      document.body.removeChild(script);
    };

    document.body.appendChild(script);
  });
}
```

### Backend (Google Apps Script)

**doGet для JSONP:**

```javascript
function doGet(e) {
  const mode = e.parameter.mode || 'web';
  const action = e.parameter.action || 'getTasks';
  
  if (mode === 'jsonp') {
    const callback = e.parameter.callback || 'callback';
    
    let result;
    try {
      if (action === 'getTasks') {
        result = getTasks(e.parameter.userId);
      } else if (action === 'addTask') {
        result = addTask(
          e.parameter.title || '',
          e.parameter.plannedDate || '',
          e.parameter.assignedTo || ''
        );
      } else if (action === 'updateTask') {
        result = updateTask(e.parameter.id, {
          title: e.parameter.title || '',
          status: e.parameter.status || '',
          plannedDate: e.parameter.plannedDate || '',
          assignedTo: e.parameter.assignedTo || ''
        });
      }
      // ... другие действия
    } catch (err) {
      result = { ok: false, error: err.message };
    }

    // Возвращаем JSONP ответ
    const output = ContentService
      .createTextOutput(callback + '(' + JSON.stringify({ ok: true, data: result }) + ')')
      .setMimeType(ContentService.MimeType.JAVASCRIPT);
    return output;
  }
  
  // ... другие режимы
}
```

### Важные моменты

1. **Только GET запросы** — JSONP работает только через GET
2. **Все параметры в URL** — данные передаются как query параметры
3. **Ограничение на длину URL** — ~2000 символов (достаточно для задач)
4. **Безопасность** — нет встроенной защиты, нужна авторизация через userId

---

## 📊 Структура Google Таблицы

### Лист "Задания"

| Колонка | Название | Тип | Пример |
|---------|----------|-----|--------|
| A | ID | String | `task_1773140608448_x1y2z3` |
| B | Задание | String | `Настроить сервер` |
| C | Статус | String | `todo`, `in-progress`, `done` |
| D | Дата начала | Date | `10.03.2026 14:03:28` |
| E | Дата конца | Date | `15.03.2026 18:00:00` (авто при статусе done) |
| F | Время выполнения | String | `5 дн.` (авто при статусе done) |
| G | Плановая дата | Date | `2026-03-15` |
| H | AssignedTo | String | `user_1773140502981_abc` |

**Заголовки** (строка 1):
```
ID | Задание | Статус | Дата начала | Дата конца | Время выполнения | Плановая дата | AssignedTo
```

**Форматирование:**
- Строка 1: жирный шрифт, фон `#f5f7fa`
- Замороженная строка 1

### Лист "Пользователи"

| Колонка | Название | Тип | Пример |
|---------|----------|-----|--------|
| A | UserID | String | `user_1773140502981_abc` |
| B | Имя | String | `Иван Петров` |
| C | TelegramID | String | `@username` или `123456789` |
| D | Роль | String | `user` или `admin` |

**Заголовки** (строка 1):
```
UserID | Имя | TelegramID | Роль
```

---

## 🔁 Синхронизация данных

### Автообновление (index.html)

**Интервал:** 30 секунд

**Код:**
```javascript
const AUTO_REFRESH_SECONDS = 30;
let autoRefreshInterval = null;

function startAutoRefresh() {
  autoRefreshInterval = setInterval(() => {
    getTasks(currentUserId === 'admin' ? 'admin' : currentUserId)
      .then(newTasks => {
        // Сравниваем с текущими
        if (JSON.stringify(newTasks) !== JSON.stringify(tasks)) {
          tasks = newTasks;
          renderBoard();
          setSyncStatus('ready', '✅ Обновлено');
          setTimeout(() => setSyncStatus('ready', ''), 2000);
        }
      })
      .catch(e => console.error('Auto-refresh error:', e));
  }, AUTO_REFRESH_SECONDS * 1000);
}
```

### Оптимистичное обновление UI

**Проблема:** Ожидание ответа от сервера (2-3 сек) перед обновлением UI.

**Решение:** Сначала обновляем UI, потом отправляем на сервер.

**Код (moveTask):**
```javascript
async function moveTask(taskId, direction) {
  const task = tasks.find(t => t.id === taskId);
  if (!task) return;

  // 1. Определяем новый статус
  let newStatus = direction === 'forward' 
    ? (task.status === 'todo' ? 'in-progress' : 'done')
    : (task.status === 'done' ? 'in-progress' : 'todo');

  // 2. Оптимистичное обновление UI
  task.status = newStatus;
  pendingTaskIds.add(taskId);

  // 3. Удаляем старую карточку
  const oldCard = document.querySelector(`.task-card[data-id="${taskId}"]`);
  if (oldCard) oldCard.remove();

  // 4. Рендерим новую с анимацией
  renderTask(task, true);

  // 5. Добавляем класс синхронизации
  const newCard = document.querySelector(`.task-card[data-id="${taskId}"]`);
  if (newCard) {
    newCard.classList.add('syncing');
    newCard.classList.add('just-arrived');
  }

  // 6. Отправляем на сервер (асинхронно)
  try {
    await apiJsonp('updateStatus', { id: taskId, status: newStatus });
    
    // 7. Убираем классы после успеха
    pendingTaskIds.delete(taskId);
    const card = document.querySelector(`.task-card[data-id="${taskId}"]`);
    if (card) {
      card.classList.remove('syncing');
      card.classList.remove('just-arrived');
    }
  } catch (err) {
    // 8. При ошибке — перезагрузка
    pendingTaskIds.delete(taskId);
    alert('Ошибка при перемещении: ' + err.message);
    await loadTasks();
  }
}
```

### Анимации

**cardArrive** (появление карточки):
```css
@keyframes cardArrive {
  0% { opacity: 0; transform: translateY(-18px) scale(0.96); }
  60% { opacity: 1; transform: translateY(4px) scale(1.01); }
  100% { opacity: 1; transform: translateY(0) scale(1); }
}

.task-card.just-arrived {
  animation: cardArrive 0.35s cubic-bezier(0.22, 1, 0.36, 1) forwards;
}
```

**syncPulse** (синхронизация):
```css
@keyframes syncPulse {
  0%, 100% { 
    box-shadow: 0 2px 6px rgba(0,0,0,0.08), 0 0 0 0 rgba(74,144,217,0.7); 
  }
  50% { 
    box-shadow: 0 2px 6px rgba(0,0,0,0.08), 0 0 0 12px rgba(74,144,217,0); 
  }
}

.task-card.syncing {
  animation: syncPulse 1.2s ease-in-out infinite;
}
```

---

## 🐛 Известные проблемы и решения

### 1. CORS ошибка 405

**Симптомы:**
```
Preflight response is not successful. Status code: 405
Failed to load resource: Preflight response is not successful
```

**Причина:** Google Apps Script не поддерживает CORS preflight.

**Решение:** Использовать JSONP вместо fetch POST.

```javascript
// ❌ НЕ работает:
fetch(API_URL, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ action: 'addTask', ... })
});

// ✅ Работает:
apiJsonp('addTask', { title: '...', assignedTo: '...' });
```

### 2. Задачи не загружаются

**Симптомы:** Задачи есть в таблице, но на странице пусто.

**Причины:**

1. **Неправильное название листа**
   - Должно быть: `Задания` (с большой буквы)
   - Проверить в `cod_v2.gs`: `const SHEET_NAME_TASKS = 'Задания';`

2. **Фильтрация по userId**
   - Если пользователь не admin, показываются только его задачи
   - Проверить колонку H (AssignedTo) — должно совпадать с userId

3. **userId не передаётся**
   - Проверить в консоли: `console.log(currentUserId)`
   - Должен быть установлен после выбора пользователя

**Диагностика:**
```javascript
// В консоли браузера:
apiJsonp('getTasks').then(console.log).catch(console.error);

// В логах Google Apps Script:
// doGet → getTasksData: userId=...
// getTasksData: total rows=X
```

### 3. Задачи создаются, но не отображаются

**Симптомы:** Задача добавляется в таблицу, но после перезагрузки страницы её нет.

**Причина:** AssignedTo не совпадает с userId текущего пользователя.

**Решение:**
- Проверить, что `assignedTo` в задаче совпадает с `currentUserId`
- Для admin — задачи должны быть видны всегда

### 4. Редактирование не сохраняется

**Симптомы:** Изменения в задаче не записываются в таблицу.

**Причина:** В JSONP режиме не был обработан `updateTask`.

**Решение:** Добавить в `doGet`:
```javascript
} else if (action === 'updateTask') {
  const id = e.parameter.id;
  result = updateTask(id, {
    title: e.parameter.title || '',
    status: e.parameter.status || '',
    plannedDate: e.parameter.plannedDate || '',
    assignedTo: e.parameter.assignedTo || ''
  });
```

### 5. Триггер checkUpdates не найден

**Симптомы:**
```
Сбой выполнения: Script function not found: checkUpdates
```

**Причина:** Старый триггер по времени ссылается на несуществующую функцию.

**Решение:**
1. Открыть https://script.google.com/
2. Слева: **Триггеры** (значок будильника)
3. Удалить триггер для `checkUpdates`
4. Или добавить пустую функцию:
```javascript
function checkUpdates() {
  // Пустая функция для триггера
}
```

### 6. Карточка задачи исчезает при перемещении

**Симптомы:** При нажатии "Вперёд →" карточка пропадает.

**Причина:** `renderTask(task, true)` ищет элемент с id="true".

**Решение:**
```javascript
// ❌ НЕ работает:
function renderTask(task, containerId) {
  const container = document.getElementById(containerId);
  // ...
}

// ✅ Работает:
function renderTask(task, appendToColumn = false) {
  let container;
  if (appendToColumn === true) {
    container = document.getElementById(task.status + '-tasks');
  } else {
    container = document.getElementById(appendToColumn);
  }
  // ...
}
```

### 7. Долгая задержка при перемещении

**Симптомы:** После нажатия кнопки 2-3 секунды ничего не происходит.

**Причина:** Ожидание ответа от сервера перед обновлением UI.

**Решение:** Оптимистичное обновление (см. раздел "Синхронизация данных").

---

## 📝 Чек-лист развёртывания

### Google Apps Script

- [ ] Код `cod_v2.gs` скопирован в проект
- [ ] SPREADSHEET_ID правильный
- [ ] Названия листов: `Задания`, `Пользователи`
- [ ] Развёртывание: **Execute as: Me**
- [ ] Доступ: **Who has access: Anyone**
- [ ] API_URL скопирован в HTML файлы

### Google Таблица

- [ ] Лист `Задания` с 8 колонками
- [ ] Лист `Пользователи` с 4 колонками
- [ ] Заголовки выделены жирным
- [ ] Первая строка заморожена

### Тестирование

- [ ] Открыть `select_user.html` → создать пользователя
- [ ] Открыть `index.html` → выбрать пользователя
- [ ] Создать задачу → проверить в таблице
- [ ] Переместить задачу → проверить анимацию
- [ ] Изменить задачу в админке → проверить сохранение

---

## 🔐 Безопасность

### Текущая модель

- **Нет паролей** — доступ по userId
- **Нет шифрования** — все данные в открытом виде
- **Публичный API** — любой знает URL может читать данные

### Рекомендации

1. **Не хранить чувствительные данные** в таблице
2. **Ограничить доступ** к таблице (только доверенные лица)
3. **Регулярно менять API_URL** при компрометации

---

## 📞 Поддержка

### Логи для отладки

**Браузер:**
```javascript
// Включить логи
console.log('currentUserId:', currentUserId);
console.log('tasks:', tasks);

// Тест API
apiJsonp('getTasks').then(console.log).catch(console.error);
```

**Google Apps Script:**
```javascript
// В коде cod_v2.gs:
Logger.log('getTasksData: userId=' + userId);
Logger.log('getTasksData: total rows=' + data.length);

// Просмотр: Выполнения → Журнал
```

### Полезные ссылки

- [Google Apps Script Docs](https://developers.google.com/apps-script)
- [ContentService](https://developers.google.com/apps-script/reference/content/content-service)
- [JSONP Wikipedia](https://en.wikipedia.org/wiki/JSONP)

---

**Версия документации:** 1.0  
**Дата:** 10 марта 2026  
**Статус:** ✅ Рабочая версия
