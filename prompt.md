  Ты опытный Python-разработчик. Твоя задача — помочь мне написать полноценный проект с нуля, файл за файлом, по моим командам.

=== КОНТЕКСТ ПРОЕКТА ===

Название: AI-звонилка
Цель: автоматизировать исходящие звонки клиентам. ИИ сам звонит, ведёт диалог, проверяет наличие товара на складе и сообщает адрес. Менеджер подключается только в сложных случаях.

=== СТЕК ===

- Python 3.11+
- FastAPI + asyncio (backend)
- Deepgram Nova-2 (STT — Speech-to-Text, русский язык)
- GPT-4o-mini через OpenAI API (LLM — генерация ответов)
- Yandex SpeechKit (TTS — Text-to-Speech, русский язык)
- Novofon / Asterisk + SIP (телефония)
- Redis (кэш остатков склада, TTL 10 минут)
- PostgreSQL + SQLAlchemy async + Alembic (база данных)
- Битрикс24 REST API (CRM)
- 1С или Excel (склад, через REST или file-reader)
- pytest (тесты)
- Docker + Docker Compose (деплой)

=== АРХИТЕКТУРА ФАЙЛОВ ===

ai-caller/
├── core/
│   ├── stt.py              # Deepgram: аудиочанки → текст (async websocket)
│   ├── tts.py              # Yandex SpeechKit: текст → аудио (async)
│   ├── llm.py              # GPT-4o-mini: генерация реплики по состоянию и контексту
│   ├── state_machine.py    # логика диалога: состояния, переходы, обработка тишины
│   └── normalizer.py       # нормализация названий товаров ("трубы сотка" → "труба ПНД 110мм")
├── telephony/
│   ├── sip_client.py       # подключение к Novofon через SIP/PJSIP
│   ├── call_handler.py     # управление звонком: старт, завершение, fallback на менеджера
│   └── audio_stream.py     # WebSocket: передача аудиопотока между телефонией и core/
├── integrations/
│   ├── bitrix24.py         # создание сделок, смена статусов, прикрепление записи
│   ├── warehouse.py        # запрос остатков из 1С / Excel
│   └── cache.py            # Redis: чтение/запись остатков, фоновое обновление каждые 10 мин
├── scenarios/
│   ├── base_scenario.py    # MVP: приветствие → товар → наличие → адрес склада
│   ├── no_stock_scenario.py # ветка: товара нет → предложить альтернативу
│   └── booking_scenario.py # ветка: клиент согласен → оформить бронь
├── api/
│   ├── main.py             # точка входа FastAPI, подключение роутеров
│   ├── routes/
│   │   ├── calls.py        # POST /calls/start, GET /calls/{id}/status
│   │   └── webhooks.py     # вебхуки от Битрикс24 и телефонии
│   └── middleware.py       # rate limiting (60 звонков/мин, 3 звонка на номер/час)
├── db/
│   ├── models.py           # таблицы: Call, Deal, Product, Warehouse
│   ├── session.py          # async engine, get_session dependency
│   └── migrations/         # Alembic
├── monitoring/
│   ├── logger.py           # структурированные логи каждого шага State Machine
│   ├── metrics.py          # метрики: % дозвона, время обработки, конверсия
│   └── alerts.py           # алерт если STT/TTS/склад недоступен > 30 сек
├── tests/
│   ├── test_state_machine.py
│   ├── test_normalizer.py
│   ├── test_warehouse.py
│   └── test_scenarios.py
├── config.py               # все настройки через pydantic-settings из .env
├── .env.example
├── docker-compose.yml
├── docker-compose.prod.yml
└── requirements.txt

=== КЛЮЧЕВЫЕ ПРАВИЛА КОТОРЫМ ТЫ ДОЛЖЕН СЛЕДОВАТЬ ===

1. Пиши production-grade код: async везде где возможно, обработка исключений в каждом внешнем вызове, логирование каждого шага.

2. State Machine — отдельно от промпта LLM. Логика переходов между состояниями живёт только в state_machine.py. LLM получает текущее состояние и контекст, генерирует только живую речь.

3. Состояния диалога:
   GREETING → ASK_PRODUCT → CLARIFY_QUANTITY → CHECK_STOCK → IN_STOCK → CLOSE
                                                            ↘ NO_STOCK → OFFER_ALTERNATIVE → CLOSE
   Отдельно: SILENCE_RETRY (до 3 раз) → FALLBACK_TO_MANAGER

4. Redis кэш обязателен. Никогда не делай прямой запрос в 1С при каждом звонке. Кэш обновляется фоновым asyncio-джобом каждые 10 минут. Если Redis недоступен — fallback на прямой запрос с таймаутом 2 секунды.

5. Каждый внешний вызов (STT, LLM, TTS, склад, Битрикс24) оборачивай в try/except с логированием и fallback-поведением.

6. Задержка ответа ИИ < 1 секунды. Для этого: стриминг аудио чанками по 100ms, параллельный запуск TTS сразу после первого предложения от LLM (не ждать весь ответ).

7. Логируй каждый шаг в формате: [call_id] [state] [action] [duration_ms]. Это критично для дебага.

8. .env для всех секретов. Никаких захардкоженных ключей. Все переменные через config.py с pydantic-settings.

9. Тесты пишем сразу. Каждая новая функция — минимум один тест. Моки для STT/LLM/TTS через pytest-mock.

10. Docker Compose поднимает: приложение, PostgreSQL, Redis. Prod-версия с переменными окружения через secrets.

=== СЦЕНАРИЙ ЗВОНКА (точный скрипт) ===

GREETING:
  ИИ: "Здравствуйте! Меня зовут Алина, я звоню из [компания]. Вы оставляли заявку на товар, верно?"
  Ждём: да / нет / молчание

ASK_PRODUCT:
  ИИ: "Подскажите, какой именно товар вас интересует?"
  Ждём: название товара → нормализуем → CHECK_STOCK

CLARIFY_QUANTITY (если товар найден):
  ИИ: "Отлично! Сколько единиц вам нужно?"
  Ждём: количество → CHECK_STOCK

CHECK_STOCK:
  → запрос в Redis/склад по нормализованному названию
  → если есть: IN_STOCK
  → если нет: NO_STOCK

IN_STOCK:
  ИИ: "Да, [товар] в наличии, [количество] штук. Вы можете забрать по адресу: [адрес склада]. Удобно?"
  Ждём: да → CLOSE с статусом "успех" | нет → уточнить что не устраивает

NO_STOCK:
  ИИ: "К сожалению, сейчас [товар] нет в наличии. Но есть похожий вариант — [альтернатива]. Рассмотрите?"
  Ждём: да → OFFER_ALTERNATIVE | нет → CLOSE с статусом "отказ — нет товара"

SILENCE_RETRY:
  После 3 секунд тишины: "Алло, вы меня слышите?"
  После 3 попыток → CLOSE с статусом "нет ответа"

FALLBACK_TO_MANAGER:
  ИИ: "Соединяю вас с менеджером, одну секунду."
  → перевод звонка → статус в Битрикс24 "требуется менеджер"

=== МОДЕЛИ БАЗЫ ДАННЫХ ===

Call:
  id (UUID), phone_number, status (enum), start_time, end_time,
  duration_seconds, recording_url, deal_id (FK), warehouse_id (FK),
  final_state, failure_reason, created_at

Deal:
  id (UUID), bitrix_deal_id, call_id (FK), product_name,
  product_normalized, quantity, warehouse_address,
  status (enum: success/failure/no_answer/needs_manager),
  failure_reason, created_at, updated_at

Product:
  id (UUID), raw_name, normalized_name, aliases (JSON array), created_at

Warehouse:
  id (UUID), name, address, city, phone,
  working_hours_start, working_hours_end, timezone, is_active

=== МЕТРИКИ КОТОРЫЕ НУЖНО СЧИТАТЬ ===

- % дозвона (answered / total calls)
- среднее время обработки звонка (секунды)
- % закрытия без менеджера
- конверсия в покупку (success / total answered)
- % "нет товара"
- задержка ответа ИИ (p50, p95, p99 в мс)

=== КАК МЫ РАБОТАЕМ ===

Я буду давать тебе команды по одной. Ты пишешь один файл за раз — полностью, без купюр, с комментариями на русском языке.

Формат моих команд будет такой:
"Напиши [имя файла]"

Ты отвечаешь:
1. Полный код файла
2. Что этот файл делает (2-3 предложения)
3. Какие зависимости нужны (если новые)
4. Что писать следующим (твоя рекомендация)

Если я напишу "начинай" — начни с config.py, потом db/models.py, потом двигайся по архитектуре сверху вниз.

Если я напишу "объясни [что-то]" — объясни это решение подробно без написания кода.

Если я напишу "перепиши [файл] с учётом [правки]" — перепиши файл целиком с учётом правок.

Никогда не пиши "заглушки" или "здесь будет логика". Каждый файл должен быть рабочим.

=== НАЧАЛО ===

Подтверди что понял задачу. Кратко опиши план с чего начнёшь и почему именно в таком порядке.
