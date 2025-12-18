# MyFIN_bot
Это автоматизация n8n для Telegram‑бота учёта личных/бизнес‑финансов с хранением данных в Supabase и транскрибацией голосовых сообщений через OpenAI.

Это автоматизация n8n для Telegram‑бота учёта личных/бизнес‑финансов с хранением данных в Supabase и транскрибацией голосовых сообщений через OpenAI.[1]

## Функционал бота

- Принимает текст и голос: пользователь может писать суммы/описания или говорить голосом, бот переводит речь в текст и разбирает его.
- Создаёт записи транзакций в Supabase: доходы и расходы, категории (включая «Бизнес»), даты, пользовательский идентификатор.
- Обрабатывает специальные фразы: например, при фразе «Продал товар» транзакция автоматически помечается как доход с категорией «Бизнес».
- Выдаёт отчёты по кнопкам: доходы/расходы за сегодня, списки операций, разные представления данных.
- Управляет записями: отмена последней операции, обнуление баланса (массовое удаление/обнуление транзакций пользователя), вспомогательные системные сообщения.

## Структура автоматизации

- Вход:  
  - `Telegram Trigger`: принимает `message` и `callback_query` из Telegram‑бота.
- Основные ветки:  
  - Ветка сообщений: `Is Message` → `voice or text` → (голос: `Get a file` → `HTTP Request1` (OpenAI) → `Edit Fields`) или (текст: сразу в `Parse Text`/`Edit Fields1`) → `HTTP Request2` (POST в Supabase) → `Send a text message`.
  - Ветка кнопок: `обработчик кнопок (mode: Rules)` → для каждой кнопки своя цепочка: отчёты (`Income Report`, `Expense Report`, `HTTP Request3/4` + фильтры и JS), отмена последнего (`HTTP GET last` → `HTTP DELETE last`), обнуление баланса (`HTTP ziro balans`), сервисные ответы (`Answer Query a callbackX`, `Send a text messageX`).
- Логика отчётов:  
  - Получение транзакций из Supabase (GET‑запросы) → фильтры по дате/типу (`Today Filter`, `Today Filter Income/Expense`) → форматирование ответа в `Code in JavaScriptX` → вывод через `Send a text messageX`.
- Логика управления:  
  - «Отменить последнее»: выбор последней транзакции пользователя (`HTTP GET last` с фильтрами по `user_id`/дате) → удаление этой записи (`HTTP DELETE last`).
  - «Обнулить баланс»: запрос на массовое удаление / обнуление транзакций текущего пользователя (`HTTP ziro balans`).

## Необходимые настройки

### 1. Telegram Trigger

- Создать Telegram‑бота и получить токен (через BotFather).  
- В ноде `Telegram Trigger` указать токен бота и включить обновления `message` и `callback_query`.
- Убедиться, что Webhook/поллинг настроены по документации n8n (бот должен получать апдейты).

### 2. OpenAI (распознавание речи)

- В ноде `HTTP Request1` настроить POST‑запрос на `https://api.openai.com/v1/audio/transcriptions`.
- Передавать в теле запросов файл голосового сообщения (`Get a file` → `Merge`) и нужную модель (указать в JSON‑теле/форме).
- API‑ключ OpenAI прятать в Credentials/Variables n8n и подставлять через заголовок Authorization (не указывать в документации открытым текстом).

### 3. Supabase (хранение транзакций)

- Создать проект Supabase и таблицу, например `transactions`, со столбцами:  
  - `id` (primary key, uuid/serial)  
  - `user_id` (integer/text — Telegram ID пользователя)  
  - `amount` (numeric)  
  - `type` (text: `income`/`expense`)  
  - `category` (text, включая «Бизнес»)  
  - `description` (text)  
  - `created_at` (timestamp, по умолчанию now()).  
- В ноде `HTTP Request2` настроить POST на REST‑endpoint Supabase для вставки записи в `transactions` (URL формата `https://<project>.supabase.co/rest/v1/transactions`).
- В нодах `HTTP Request3/4`, `Income Report`, `Expense Report`, `HTTP GET last`, `HTTP DELETE last`, `HTTP ziro balans` настроить GET/DELETE на тот же endpoint с query‑параметрами:  
  - `user_id=eq.<динамический Telegram ID>`  
  - при необходимости фильтры по типу (`type=eq.income/expense`) и сортировке/лимиту (например, `order=created_at.desc&limit=1` для последней операции).
- Ключ Supabase (service role или anon с нужными правами) хранить в Credentials/Variables и подставлять через заголовки Authorization/`apikey` (без явного указания в тексте).

### 4. Разбор текста и категорий

- В ноде `Parse Text` (или `Code in JavaScript`) реализовать правила:  
  - Распознавание суммы и направления (доход/расход) по тексту сообщения.  
  - Для фразы «Продал товар» (в тексте или транскрипции) принудительно задавать:  
    - `type = 'income'`  
    - `category = 'Бизнес'`.
- В `Edit Fields1` привести данные к финальной структуре (поля для Supabase).

### 5. Кнопки и callback‑логика

- В `Code in JavaScriptX`, которые формируют inline‑кнопки, задать `callback_data` для каждой кнопки (например, `REPORT_INCOME_TODAY`, `REPORT_EXPENSE_TODAY`, `UNDO_LAST`, `ZERO_BALANCE`).
- В ноде `обработчик кнопок (mode: Rules)` создать правила по `callback_query.data` и направить каждую кнопку в свою цепочку:  
  - `UNDO_LAST` → `HTTP GET last` → `HTTP DELETE last` → сообщение‑подтверждение.  
  - `ZERO_BALANCE` → `HTTP ziro balans` → сообщение‑подтверждение.  
  - Отчётные кнопки → соответствующие GET‑запросы и форматирование ответа.
- Во всех запросах Supabase использовать динамический `user_id` через expression (`$json.callback_query.from.id` или `$json.message.from.id` в зависимости от ветки).

Этой инструкции достаточно, чтобы другой пользователь мог импортировать workflow в n8n, подключить свои Telegram/OpenAI/Supabase и получить полностью работающий фин‑бот с голосовым вводом, бизнес‑категорией «Продал товар», отчётами, отменой последней операции и обнулением баланса.
