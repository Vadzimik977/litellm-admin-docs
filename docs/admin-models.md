# Страница Models: управление моделями

Раздел `/ui/models-and-endpoints` — регистрация и управление LLM-моделями в LiteLLM. Здесь администратор добавляет модели (деплойменты), настраивает провайдера, ценообразование и параметры.

## Что показывает

- Список всех зарегистрированных моделей с провайдером
- Статус: активна / заблокирована
- Ценообразование: стоимость за input/output токен
- Настройки: `api_key`, `api_base`, `custom_llm_provider`

## Структура модели (деплоймента)

Каждая модель в LiteLLM — это **деплоймент** из трёх полей. Хранится в таблице `LiteLLM_ProxyModelTable`:

```
{
  "model_name":     <алиас / модельная группа>,
  "litellm_params": { "model": <реальный бэкенд>, ... },
  "model_info":     { "id": <uuid>, ... }
}
```

### model_name — алиас и модельная группа

- Публичное имя модели для клиентов (например `gpt-4o`, `claude-haiku-4.5`).
- Это же значение = **model_group** — по нему происходит маршрутизация и фолбэки.
- Несколько деплойментов могут иметь одинаковый `model_name` (для failover/heartbeat).
- Одна модель может иметь несколько алиасов (например `claude-haiku-4.5` и `anthropic/claude-haiku-4.5`) — при проверке ключа они эквивалентны.

### litellm_params — параметры провайдера

Обязательное поле — `model`, остальное опционально (схема разрешает любые доп. поля).

| Группа | Поля |
| ------ | ---- |
| **Обязательное** | `model` — реальное имя бэкенд-модели |
| **Провайдер** | `custom_llm_provider`, `api_base`, `api_version`, `region_name` |
| **Доступ** | `api_key` (в т.ч. `os.environ/KEY`), `azure_ad_token`, `vertex_project`, vertex/location/credentials, aws keys, `s3_bucket_name` |
| **Лимиты** | `tpm`, `rpm`, `max_budget`, `budget_duration`, `max_parallel_requests`, `timeout`, `stream_timeout`, `max_retries` |
| **Роутер** | `weight`, `order`, `drop_params`, `mock_response`, `tags`, `tag_regex` |
| **Генерация** | `temperature`, `top_p`, `max_tokens`, `presence_penalty`, `frequency_penalty`, `stop` — обычно передаются в запросе, но могут быть заданы как дефолт деплоймента |
| **Цены** | все ценовые поля (см. ниже раздел «Переопределение цен») |

### model_info — метаданные и цены

| Поле | Описание |
| ---- | -------- |
| `id` | UUID — генерируется автоматически, если не задан. **Ключ регистрации модели в реестре цен** |
| `mode` | Тип: `chat` / `embedding` / `completion` / `image_generation` / `audio_transcription` / `responses` / `ocr` |
| `base_model` | Для Azure — базовая модель для расчёта цены |
| `max_tokens` | Максимум токенов (по умолчанию 2048) |
| `blocked` | Заблокирована ли модель |
| `db_model` | true — модель из БД, а не из конфига |
| `created_by` / `updated_by` | Кто создал/изменил |
| ценовые поля | см. ниже раздел «Ценообразование» |

## Ценообразование

### Два источника цен

У LiteLLM **два слоя** ценообразования:

1. **Встроенный реестр** — `model_prices_and_context_window_backup.json` (2864 модели). Содержит цены, контекст и возможности для стандартных моделей. Читается через `litellm.utils.get_model_info()` по умолчанию.

2. **Переопределение в деплойменте** — то, что задаёт администратор в `model_info`/`litellm_params`. Добавленный деплоймент регистрируется в реестре под своим `model_info.id`, а при расчёте стоимости роутер **накладывает** `model_info` пользователя поверх данных реестра. Пользовательские цены всегда главнее встроенных.

### Поля цен (в `model_info`)

Базовые:

| Поле | Описание |
| ---- | -------- |
| `input_cost_per_token` | Цена входных токенов |
| `output_cost_per_token` | Цена выходных токенов |
| `cache_read_input_token_cost` | Цена токенов, прочитанных из кэша |
| `cache_creation_input_token_cost` | Цена токенов создания кэша |

Расширенные (для спец. провайдеров):

| Поле | Описание |
| ---- | -------- |
| `input_cost_per_character` / `output_cost_per_character` | По символам (Vertex) |
| `input_cost_per_audio_token` / `output_cost_per_audio_token` | Аудиотокены |
| `input_cost_per_image` / `output_cost_per_image` | По изображениям |
| `input_cost_per_second` / `output_cost_per_second` | По секундам (vision/audio) |
| `input_cost_per_video_per_second` / `output_cost_per_video_per_second` | Видео |
| `input_cost_per_query` | Rerank |
| `output_cost_per_reasoning_token` | Reasoning-токены |
| `ocr_cost_per_page` / `ocr_cost_per_credit` | OCR |
| `output_vector_size` | Размерность эмбеддингов (embedding) |

Пороговое ценообразование — цена меняется при превышении объёма контекста:

| Поле | Описание |
| ---- | -------- |
| `input_cost_per_token_above_128k_tokens` | Цена при входе > 128k токенов |
| `input_cost_per_token_above_200k_tokens` | Цена при входе > 200k |
| `input_cost_per_token_above_272k_tokens` | Цена при входе > 272k |
| `input_cost_per_token_above_512k_tokens` | Цена при входе > 512k |

### Как переопределить цену

Любой ценовой ключ можно задать двумя способами — в `model_info` ИЛИ прямо в `litellm_params` (поля из списка автоматически копируются в `model_info` при создании деплоймента):

```json
{
  "model_name": "my-chat-model",
  "litellm_params": {
    "model": "gpt-4o",
    "custom_llm_provider": "openai",
    "api_key": "os.environ/OPENAI_API_KEY",
    "input_cost_per_token": 0.0000025,
    "output_cost_per_token": 0.00001
  },
  "model_info": {
    "id": "0c4b5f...",
    "mode": "chat",
    "input_cost_per_token": 0.0000025,
    "output_cost_per_token": 0.00001
  }
}
```

Записанная цена регистрируется в `litellm.model_cost` под `model_info.id` и под ключом `custom_llm_provider/model` (например `openai/gpt-4o`).

### Как считается стоимость

Расчёт ведёт `cost_calculator.py` → `completion_cost()`. Базовая формула:

```
cost = prompt_tokens × input_cost_per_token
     + completion_tokens × output_cost_per_token
     + cache_read_tokens × cache_read_input_token_cost
     + cache_creation_tokens × cache_creation_input_token_cost
```

Для моделей типа Vertex/Audio стоимость считается по символам/секундам/изображениям/видео вместо токенов. Применяются service tier (`_flex`/`_priority`) и пороговые правила (`above_*k_tokens`). При отсутствии response токены считаются через `token_counter`.

### Пример из встроенного реестра

**`gpt-4o` (chat, openai):**

```json
{
  "cache_read_input_token_cost": 1.25e-06,
  "input_cost_per_token": 2.5e-06,
  "input_cost_per_token_batches": 1.25e-06,
  "litellm_provider": "openai",
  "max_input_tokens": 128000,
  "max_output_tokens": 16384,
  "max_tokens": 16384,
  "mode": "chat",
  "output_cost_per_token": 1e-05,
  "supports_function_calling": true,
  "supports_parallel_function_calling": true,
  "supports_prompt_caching": true,
  "supports_vision": true
}
```

**`text-embedding-3-small` (embedding, openai):**

```json
{
  "input_cost_per_token": 2e-08,
  "litellm_provider": "openai",
  "max_input_tokens": 8191,
  "mode": "embedding",
  "output_cost_per_token": 0.0,
  "output_vector_size": 1536
}
```

> Стоимость указывается в долларах за токен (например `2.5e-06` = $0.0000025 за входной токен).

### Возможности модели (flags)

В `model_info` также хранятся флаги поддерживаемых возможностей, определяющие кастомизацию в UI:

`supports_vision`, `supports_function_calling`, `supports_parallel_function_calling`, `supports_tool_choice`, `supports_system_messages`, `supports_response_schema`, `supports_prompt_caching`, `supports_audio_input`, `supports_audio_output`, `supports_pdf_input`, `supports_web_search`, `supports_native_streaming`, `supports_reasoning`, `supports_native_structured_output`, `supports_computer_use`.

## store_model_in_db

При `store_model_in_db: true` в `general_settings` модели из админ-панели сохраняются в БД и загружаются при старте прокси. При `false` — записи из панели игнорируются, используются только модели из `config.yaml`.

## Эндпоинты

| Метод | Путь | Описание |
| ----- | ---- | -------- |
| POST | `/model/new` | Регистрация модели |
| POST | `/model/update` | Полное обновление |
| PATCH | `/model/{model_id}/update` | Частичное обновление |
| POST | `/model/delete` | Удаление |
| POST | `/model/block` | Блокировка |
| POST | `/model/unblock` | Разблокировка |
| POST | `/model_group/make_public` | Сделать группу моделей публичной |

## Безопасность

Чувствительные значения (`api_key`) шифруются через `encrypt_value_helper` при сохранении в БД. Адрес `api_base` и сами креденшелы токена не хранятся в открытом виде.