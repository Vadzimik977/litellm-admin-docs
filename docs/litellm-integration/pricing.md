# Ценообразование моделей

Auto AI Router поддерживает расчёт стоимости для каждой модели и логирование расходов. Цены загружаются из JSON-файла или удалённого URL при старте, периодически обновляются в фоне и объединяются с ценами, хранящимися в базе данных LiteLLM.

## Конфигурация

```yaml
server:
  model_prices_link: "file://price.json"
  model_prices_sync_interval: 5m  # опционально, по умолчанию 5m
```

| Настройка                            | Тип      | По умолчанию | Описание                                                    |
| ------------------------------------ | -------- | ------------ | ------------------------------------------------------------ |
| `server.model_prices_link`           | строка   | —            | Источник файла цен. Пусто — отключает ценообразование из файла |
| `server.model_prices_sync_interval`  | длительность | `5m`     | Как часто перечитывается источник после старта               |

Обе настройки поддерживают подстановку `os.environ/VAR_NAME`.

Допустимые значения для `model_prices_link`:

| Значение                                                                                        | Описание                     |
| ----------------------------------------------------------------------------------------------- | ---------------------------- |
| `file://price.json`                                                                             | Относительный путь к файлу   |
| `file:///data/prices.json`                                                                      | Абсолютный путь              |
| `https://prices.example.com/default.json`                                                       | Удалённый HTTPS URL          |
| `https://raw.githubusercontent.com/BerriAI/litellm/main/model_prices_and_context_window.json`   | Апстрим-цены LiteLLM          |

Файл должен быть валидным JSON и не превышать 100 МБ.

## Тарифные профили организаций

`organization_policies` может привязывать проверенную организацию LiteLLM к неизменяемому тарифному профилю в USD. Профили используют те же схемы источников, лимиты размера и поведение таймаутов, что и `server.model_prices_link`, но загружаются синхронно при старте и не обновляются до перезапуска процесса.

```yaml
organization_policies:
  - organization_id: os.environ/CLOUD_RU_ORGANIZATION_ID
    price_profile_id: cloud-ru-2026-09
    model_prices_link: /app/organization-prices/cloud-ru-2026-09.json
    model_allowlist:
      - openai/gpt-5.5-pro
    model_mappings:
      openai/gpt-5.5-pro: gpt-5.5-pro
    credential_denylist:
      - untrusted-provider-credential
```

Секция отключена, когда опущена. При наличии она требует `litellm_db.enabled: true`, `litellm_db.is_required: true` и `litellm_db.disable_spend_logs_write: false`.

Поиск профиля организации — точный и чувствительный к регистру. AIR оценивает запросы по необработанному публичному идентификатору модели из запроса клиента. Фолбэка на реестр по умолчанию, цены из БД, канонические/маршрутизированные/реальные идентификаторы провайдера или нормализованные имена нет.

Тарифный JSON организаций строгий. Дублирующиеся точные ключи, неизвестные поля строк, строки `null` и пустые ценовые объекты приводят к остановке при старте. Явно бесплатная модель должна всё равно содержать хотя бы одно распознанное ценовое поле с нулевым значением.

Когда `model_allowlist` опущен, организация видит глобальную вызываемую поверхность плюс ключи маппинга организации, подчинённые точным ценам профиля в `/v1/models`. Вызываемый запрос без точной строки профиля возвращает `503` до выбора провайдера. Когда `model_allowlist` присутствует и пуст — поверхность организации пуста. Когда присутствует и непуст — каждый перечисленный идентификатор должен быть маршрутизируемым и иметь точную строку профиля при старте.

### Исключение креденшелов провайдера

Опустите и `price_profile_id`, и `model_prices_link`, чтобы использовать глобальное ценообразование и модели AIR с денлистом креденшелов организаций.

```yaml
organization_policies:
  - organization_id: os.environ/ORGANIZATION_ID
    credential_denylist:
      - cometapi01
      - cheapgpt-openai-key-1
```

Эта политика использует стандартные алиасы моделей, поиск цен и расписание обновления цен. Ограничения по ключам и командам по-прежнему применяются. SpendLogs сохраняют идентичность организации без кастомного идентификатора биллинг-профиля или дайджеста.

Два тарифных поля должны указываться вместе или опускаться вместе. `model_allowlist` и непустой `model_mappings` требуют тарифного профиля. Политики с тарифным профилем сохраняют точное сопоставление публичных цен моделей.

Исключения креденшелов применяются при маршрутизации запроса. Каталог моделей неизменен. Запрос отклоняется, если каждый подходящий креденшл провайдера исключён.

`credential_denylist` содержит точные имена креденшелов провайдера (с учётом регистра). Подходящие креденшелы исключаются из начального выбора, привязки сессии, повторных попыток и фолбэка. Креденшелы роутера остаются подходящими. Ограничение следует за запросом через цепочку маршрутизаторов AIR. Неизвестные имена игнорируются локально и остаются доступными нижестоящим маршрутизаторам.

Список принимает до 1024 имён и 65536 закодированных байт. Каждое имя может содержать до 256 байт. Пустые имена, дубликаты и управляющие символы приводят к ошибке загрузки конфигурации. Опущенный или пустой список сохраняет стандартную маршрутизацию.

### Интервал обновления {#refresh-interval}

Цены читаются один раз при старте и далее перечитываются в фоне каждые `model_prices_sync_interval`. Задаётся любой Go-длительностью:

```yaml
server:
  model_prices_link: "https://raw.githubusercontent.com/BerriAI/litellm/main/model_prices_and_context_window.json"
  model_prices_sync_interval: 1h  # 30s, 15m, 1h, 24h ...
```

Поведение:

- Интервал применяется только когда задан `model_prices_link`. При пустой ссылке цикл синхронизации не запускается.
- Загрузка при старте происходит немедленно и не ждёт первого тика.
- Неудачное обновление (недоступный URL, нечитаемый файл, невалидный JSON) логируется как предупреждение, а ранее загруженные цены остаются в реестре. Следующий тик повторяет попытку.
- Успешное обновление атомарно заменяет весь реестр; выполняющиеся запросы продолжают использовать уже разрешённые цены.
- Отсутствующее или неположительное значение откатывается к значению по умолчанию `5m`.

Выбор значения:

| Источник                                | Рекомендуемый интервал | Обоснование                                                                      |
| --------------------------------------- | ---------------------- | -------------------------------------------------------------------------------- |
| Локальный файл, смонтированный в контейнер | `5m` (по умолчанию)    | Дёшево перечитывать; подхватывает правки без перезапуска                         |
| Локальный файл, обновляемый внешним джобом | `30s` – `1m`           | Сокращает окно, в котором расходы логируются по устаревшим ценам                  |
| Удалённый HTTPS URL (своя инфраструктура)  | `5m` – `15m`           | Балансирует свежесть против объёма запросов к хосту цен                          |
| Апстрим-JSON LiteLLM с GitHub               | `1h` – `24h`           | Файл меняется редко, а частый опрос рискует словить лимиты на удалённом хосте      |

Эффективный интервал печатается при старте:

```
26.08.26 10:15:03 [INFO] » Using model prices from link=file://price.json sync_interval=5m0s
```

Каждое успешное обновление логируется на уровне `debug` (`Model prices updated`), поэтому поднимите `server.logging_level` до `debug`, проверяя, что новый файл цен действительно подхватывается.

## Формат файла цен

Файл — это JSON-объект, где каждый ключ — имя модели, а каждое значение — описание цены:

```json
{
  "gpt-4o-mini": {
    "input_cost_per_token": 1.5e-07,
    "output_cost_per_token": 6e-07
  },
  "gemini-2.5-flash": {
    "input_cost_per_token": 3e-07,
    "output_cost_per_token": 2.5e-06,
    "input_cost_per_audio_token": 1e-06,
    "output_cost_per_reasoning_token": 2.5e-06
  },
  "claude-opus-4-1": {
    "input_cost_per_token": 1.5e-05,
    "output_cost_per_token": 7.5e-05,
    "cache_read_input_token_cost": 1.5e-06,
    "cache_creation_input_token_cost": 1.875e-05,
    "cache_creation_input_token_cost_above_1hr": 3e-05,
    "cache_read_input_token_cost_above_200k_tokens": 3e-06,
    "cache_creation_input_token_cost_above_200k_tokens": 3.75e-05,
    "cache_creation_input_token_cost_above_1hr_above_200k_tokens": 6e-05
  },
  "imagen-4.0-fast-generate-001": {
    "output_cost_per_image": 0.02
  },
  "gpt-4o-search-preview": {
    "input_cost_per_token": 2.5e-06,
    "output_cost_per_token": 1e-05,
    "search_context_cost_per_query": {
      "search_context_size_low": 0.025,
      "search_context_size_medium": 0.0275,
      "search_context_size_high": 0.03
    }
  }
}
```

### Почему цены указываются за 1 токен

Все цены за токен выражены как стоимость за **один токен** (не за 1000 и не за миллион). Это совпадает с форматом `model_prices_and_context_window.json` от LiteLLM, поэтому легко использовать апстрим-файл напрямую или вести собственный файл переопределений в том же формате.

Для справки:

- `$1.50 / 1M токенов` → `1.5e-06` (0.0000015)
- `$0.15 / 1M токенов` → `1.5e-07` (0.00000015)

### Доступные поля

| Поле                                                          | Описание                                                                    |
| ------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| `input_cost_per_token`                                        | Обычные входные токены                                                       |
| `output_cost_per_token`                                       | Обычные выходные токены                                                      |
| `input_cost_per_token_above_200k_tokens`                      | Входная ставка для токенов сверх порога 200k                                 |
| `output_cost_per_token_above_200k_tokens`                     | Выходная ставка для токенов сверх порога 200k                                |
| `input_cost_per_token_above_32k_tokens`                       | Входная ставка всей сессии, когда промпт превышает 32k токенов               |
| `output_cost_per_token_above_32k_tokens`                      | Выходная ставка всей сессии, когда промпт превышает 32k токенов              |
| `input_cost_per_token_above_128k_tokens`                      | Входная ставка всей сессии, когда промпт превышает 128k токенов              |
| `output_cost_per_token_above_128k_tokens`                     | Выходная ставка всей сессии, когда промпт превышает 128k токенов             |
| `input_cost_per_token_above_256k_tokens`                      | Входная ставка всей сессии, когда промпт превышает 256k токенов              |
| `output_cost_per_token_above_256k_tokens`                     | Выходная ставка всей сессии, когда промпт превышает 256k токенов             |
| `input_cost_per_token_above_272k_tokens`                      | Входная ставка всей сессии, когда промпт превышает 272k токенов              |
| `output_cost_per_token_above_272k_tokens`                     | Выходная ставка всей сессии, когда промпт превышает 272k токенов             |
| `input_cost_per_audio_token`                                  | Аудиовходные токены (фолбэк на `input_cost_per_token`, если отсутствует)     |
| `output_cost_per_audio_token`                                 | Аудиовыходные токены (фолбэк на `output_cost_per_token`, если отсутствует)   |
| `input_cost_per_image_token`                                  | Входные токены изображений                                                   |
| `output_cost_per_image_token`                                 | Выходные токены изображений                                                  |
| `output_cost_per_reasoning_token`                             | Токены рассуждения (фолбэк на `output_cost_per_token`)                       |
| `input_cost_per_cached_token`                                 | Стоимость чтения кэшированного промпта (алиас: `cache_read_input_token_cost`) |
| `cache_read_input_token_cost`                                 | Алиас, совместимый с LiteLLM, для `input_cost_per_cached_token`              |
| `cache_creation_input_token_cost`                             | Стоимость записи в кэш промпта (фолбэк на `input_cost_per_token`)            |
| `cache_read_input_token_cost_above_200k_tokens`               | Ставка чтения кэша всей сессии, когда промпт превышает 200k токенов          |
| `cache_creation_input_token_cost_above_200k_tokens`           | Ставка записи в кэш всей сессии (5m/без класса) сверх 200k                    |
| `cache_creation_input_token_cost_above_1hr`                   | Ставка записи в кэш Anthropic 1h (фолбэк на обычную ставку записи в кэш)      |
| `cache_creation_input_token_cost_above_1hr_above_200k_tokens` | Ставка записи в кэш Anthropic 1h сверх 200k                                  |
| `cache_read_input_token_cost_above_32k_tokens`                | Ставка чтения кэша всей сессии, когда промпт превышает 32k токенов           |
| `cache_creation_input_token_cost_above_32k_tokens`            | Ставка записи в кэш всей сессии, когда промпт превышает 32k токенов          |
| `cache_read_input_token_cost_above_128k_tokens`               | Ставка чтения кэша всей сессии, когда промпт превышает 128k токенов          |
| `cache_creation_input_token_cost_above_128k_tokens`           | Ставка записи в кэш всей сессии, когда промпт превышает 128k токенов         |
| `cache_read_input_token_cost_above_256k_tokens`               | Ставка чтения кэша всей сессии, когда промпт превышает 256k токенов          |
| `cache_creation_input_token_cost_above_256k_tokens`           | Ставка записи в кэш всей сессии, когда промпт превышает 256k токенов         |
| `cache_read_input_token_cost_above_272k_tokens`               | Ставка чтения кэша всей сессии, когда промпт превышает 272k токенов          |
| `cache_creation_input_token_cost_above_272k_tokens`           | Ставка записи в кэш всей сессии, когда промпт превышает 272k токенов         |
| `cache_read_input_audio_token_cost`                           | Ставка кэшированных аудиовходных данных (фолбэк на выбранную ставку чтения)  |
| `output_cost_per_cached_token`                                | Кэшированные выходные токены (фолбэк на `output_cost_per_token`)             |
| `output_cost_per_prediction_token`                            | Принятые предсказанные выходные токены (фолбэк на `output_cost_per_token`)   |
| `output_cost_per_image`                                       | Стоимость сгенерированного изображения (приоритет над `output_cost_per_image_token`) |
| `search_context_cost_per_query`                               | Стоимость Web Search за запрос/вызов, по ключу `search_context_size_*`        |
| `web_search_billing_unit`                                     | Режим оплаты Web Search: `per_query` или `per_prompt`                         |

## Расчёт стоимости

Все провайдеры возвращают специализированные счётчики токенов как **подмножества** итогов:

- `prompt_tokens` (Vertex AI, OpenAI) уже включает `audio_input_tokens`, `cached_input_tokens`
- `completion_tokens` (все провайдеры) уже включает `reasoning_tokens`, `audio_output_tokens`, prediction-токены
- Anthropic сообщает кэш-токены отдельно; API, совместимые с OpenAI, сообщают их в деталях prompt/input токенов

Чтобы не тарифицировать одни и те же токены по двум разным ставкам, калькулятор сначала вычисляет **обычные** (базовые) счётчики токенов, вычитая все специализированные подтипы, а затем добавляет каждый подтип обратно по своей ставке:

```
regular_input  = prompt_tokens - audio_input_tokens - cached_input_tokens - cache_creation_tokens
regular_output = completion_tokens - audio_output_tokens - reasoning_tokens
                                   - accepted_prediction_tokens - rejected_prediction_tokens

total = regular_input  × input_cost_per_token
      + regular_output × output_cost_per_token
      + audio_input_tokens  × input_cost_per_audio_token
      + audio_output_tokens × output_cost_per_audio_token
      + cached_text_tokens  × cache_read_input_token_cost
      + cached_audio_tokens × cache_read_input_audio_token_cost
      + cache_creation_5m_tokens × cache_creation_input_token_cost
      + cache_creation_1h_tokens × cache_creation_input_token_cost_above_1hr
      + cached_output_tokens   × output_cost_per_cached_token
      + reasoning_tokens            × output_cost_per_reasoning_token
      + accepted_prediction_tokens  × output_cost_per_prediction_token
      + rejected_prediction_tokens  × output_cost_per_token
      + image_count × output_cost_per_image
      + web_search_requests × search_context_cost_per_query[search_context_size]
```

Это значит, что каждый токен тарифицируется **ровно один раз**, независимо от того, как его сообщил провайдер.

### Оплата Web Search

Web Search оплачивается как отдельная стоимость инструмента, а не как токены. Калькулятор читает цены `search_context_cost_per_query`, совместимые с LiteLLM, и выбирает одну из:

- `search_context_size_low`
- `search_context_size_medium`
- `search_context_size_high`

AIR получает размер запроса из `web_search_options.search_context_size` или из определения инструмента `web_search` / `web_search_preview`. Если запрос не задаёт размер, используется `medium`.

AIR взимает плату только за подтверждённое использование ответа:

- `usage.server_tool_use.web_search_requests`
- `usage.web_search_requests`
- элементы `response.output[]` или `output[]` с `type: "web_search_call"`
- аннотации `url_citation` Chat Completions, когда такой контракт API подтверждает поиск
- Vertex/Gemini `groundingMetadata.webSearchQueries`

Простое включение инструмента не считается выполнением. Успешный ответ без подтверждённого использования тарифицируется как ноль поисков. Стриминговые запросы используют финальное использование провайдера или завершённый выходной ответ. Неполное событие инструмента не тарифицируется.

`per_query` умножает настроенную цену на подтверждённое число запросов. `per_prompt` ограничивает любое положительное число одним списанием. Записи LiteLLM Gemini 2.x без явной единицы используют `per_prompt`, тогда как записи Gemini 3.x явно используют `per_query`.

Количество и выбранный размер контекста записываются в метаданные расходов в `usage_object.server_tool_use` и `additional_usage_values.server_tool_use`; стоимость инструмента записывается в `cost_breakdown.tool_usage_cost` и `cost_breakdown.web_search_cost`.

### Обычные входные токены

Vertex AI и OpenAI включают аудио- и кэш-токены **внутри** `prompt_tokens`. Anthropic сообщает чтения и записи кэша отдельно на проводе, поэтому AIR сначала нормализует использование Anthropic до инклюзивного общего числа промпта. Формула затем использует одинаковую семантику для всех провайдеров:

- Vertex/OpenAI: `100 prompt − 5 audio − 20 cached = 75 regular`, затем +5 audio +20 cached по их ставкам
- Использование Anthropic на проводе: `100 input + 20 cache read = 120 нормализованный prompt`; биллинг использует `120 − 20 cached = 100 regular`, затем +20 cached по его ставке

### Обычные выходные токены

Все провайдеры включают reasoning внутри `completion_tokens`:

- OpenAI `o-series`: `completion_tokens_details.reasoning_tokens` — подмножество `completion_tokens`
- Vertex Gemini 2.5+: thinking-токены включены в `candidatesTokenCount`
- Anthropic с extended thinking: thinking-токены включены в `output_tokens`

Вычитание гарантирует, что reasoning тарифицируется по `output_cost_per_reasoning_token` (а не двойно списывается и по базовой выходной ставке).

### Многоуровневое ценообразование (порог 200k)

Некоторые модели взимают более высокую ставку, когда контекст превышает 200 000 токенов. Когда задан `input_cost_per_token_above_200k_tokens`:

```
below = min(prompt_tokens, 200_000)
above = prompt_tokens - 200_000          # только когда prompt_tokens > 200_000

# обычные токены делятся пропорционально между below/above
regular_above = regular_input × above / prompt_tokens
regular_below = regular_input - regular_above

input_cost = regular_below × input_cost_per_token
           + regular_above × input_cost_per_token_above_200k_tokens
```

Та же логика применяется к выходным токенам с помощью `output_cost_per_token_above_200k_tokens`.

Цены кэша следуют семантике всей сессии LiteLLM: когда `prompt_tokens > 200_000`, все токены чтения/записи кэша используют соответствующую ставку `*_above_200k_tokens`. Поля кэша всей сессии 32k/128k/256k/272k имеют приоритет над уровнем 200k, когда настроены, причём побеждает наибольший превышенный порог (см. «Долгий контекст» ниже).

### Долгий контекст (уровни всей сессии 32k / 128k / 256k / 272k / 512k)

Когда промпт превышает один из этих порогов, соответствующая ставка `*_above_<N>k_tokens` применяется ко **всей сессии**, а не только к токенам сверх порога — размер промпта выбирает уровень для обычных входных и выходных токенов, чтения и записи кэша. Ровно на пороге применяются базовые ставки (проверка строго «больше чем»).

272k был добавлен первым для таких моделей, как GPT-5.6; 32k/128k/256k добавлены для моделей Alibaba Cloud (Qwen 3.x, GLM), чьи опубликованные цены — это плоская ставка на диапазон длины входа (например 0–32k / 32k–128k / 128k–256k / >256k), а не инкрементальная ставка на переполнение. 512k добавлен для MiniMax-M3, чьё ценообразование зависит от общего числа входных токенов с порогом 512k.

Запись цены может настраивать любое подмножество из пяти порогов (например, только 32k и 256k, пропуская 128k). Для каждого компонента стоимости (вход, выход, чтение кэша, запись кэша) независимо AIR выбирает **наибольший настроенный порог, который превышает промпт**, в порядке убывания 512k → 272k → 256k → 128k → 32k, и пропускает любой порог, чьё поле ставки не задано. Если ни один из пяти не применим, расчёт откатывается к пропорциональному уровню 200k, описанному выше, а затем к базовой ставке.

Пример: модель только с `input_cost_per_token_above_128k_tokens` и `input_cost_per_token_above_256k_tokens` тарифицирует промпт 150k по ставке 128k, а промпт 300k — по ставке 256k; промпт 100k по-прежнему использует базовую ставку.

### Специализированные типы токенов

| Тип                 | Формула                                                                                                                                      |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| Аудиовход            | `audio_input_tokens × input_cost_per_audio_token` (фолбэк на обычную входную ставку)                                                          |
| Аудиовыход           | `audio_output_tokens × output_cost_per_audio_token` (фолбэк на обычную выходную ставку)                                                       |
| Чтение кэша          | Кэшированный текст использует `cache_read_input_token_cost`; кэшированное аудио — `cache_read_input_audio_token_cost` с фолбэком на выбранную ставку чтения кэша |
| Создание кэша        | Токены 5m и без класса используют `cache_creation_input_token_cost`; токены 1h — `cache_creation_input_token_cost_above_1hr`; оба безопасно фолбэкаются |
| Reasoning            | `reasoning_tokens × output_cost_per_reasoning_token` (фолбэк на обычную выходную ставку)                                                      |
| Принятый prediction  | `accepted_prediction_tokens × output_cost_per_prediction_token` (фолбэк на обычную выходную ставку)                                           |
| Отклонённый prediction | `rejected_prediction_tokens × output_cost_per_token` (всегда по обычной выходной ставке)                                                      |
| Изображения          | `image_count × output_cost_per_image` ИЛИ `output_image_tokens × output_cost_per_image_token`                                                 |
| Web Search           | `billable_web_search_count × search_context_cost_per_query[search_context_size]`, с `per_prompt`, ограниченным до единицы                          |

## Как загружаются цены

Загрузка обрабатывается `internal/models/price_loader.go`:

1. Значение `model_prices_link` проверяется для определения источника:
   - Пути, начинающиеся с `file://` или не содержащие `://`, читаются с диска.
   - Пути, начинающиеся с `http://` или `https://`, получаются по HTTP с лимитом 100 МБ.
2. JSON парсится в `map[string]*ModelPrice`.
3. Каждый ключ **нормализуется**: префикс провайдера удаляется, имя приводится к нижнему регистру.
   - `"openai/gpt-4-turbo"` → `"gpt-4-turbo"`
   - `"vertex_ai/gemini-2.5-pro"` → `"gemini-2.5-pro"`
   - Если два ключа нормализуются в одну строку, побеждает последний и логируется предупреждение.
4. Результирующая карта сохраняется в `ModelPriceRegistry` (потокобезопасный, `sync.RWMutex`).
5. Фоновая горутина повторяет шаги 1–4 каждые `server.model_prices_sync_interval` до выключения — см. раздел «Интервал обновления» выше.

### Объединение цен из БД

Когда база данных LiteLLM включена, цены, определённые в `LiteLLM_ModelTable`, объединяются поверх файлового реестра через `MergeDB`. Цены из БД имеют приоритет для любой модели, присутствующей в обоих источниках. Файловые цены остаются без изменений для всех остальных моделей.

Два независимых цикла пишут в реестр: обновление файла цен (`server.model_prices_sync_interval`, по умолчанию `5m`) заменяет всю карту, тогда как синхронизация таблицы моделей БД (`litellm_db.db_model_sync_interval`, по умолчанию `1m`) объединяет цены из БД поверх. Модель, существующая только в БД, поэтому отсутствует в реестре между обновлением файла и следующей синхронизацией БД. При включённом логировании расходов такой запрос отклоняется с `503 Model pricing unavailable`, поэтому держите `model_prices_sync_interval` на уровне `db_model_sync_interval` или выше, когда используются модели только из БД.

Записи кэша читаются из `cache_creation_tokens` или алиаса `cache_write_tokens`, совместимого с OpenAI, в объектах использования Chat Completions и Responses API.
`cache_creation_token_details` от Anthropic (`ephemeral_5m_input_tokens` и `ephemeral_1h_input_tokens`) сохраняется в метаданных лога расходов, тогда как существующие агрегатные колонки токенов создания кэша остаются обратно совместимыми. Подсчёты кэшированного аудио Gemini берутся из `cacheTokensDetails`, когда провайдер выдаёт разбивку по модальностям.

### Контракт хранения расходов

AIR сохраняет вышестоящую схему PostgreSQL LiteLLM без изменений. `LiteLLM_SpendLogs.spend` и daily-таблицы пользователей, команд, организаций и end users содержат общую стоимость. Разбивки кэша и Web Search хранятся в `LiteLLM_SpendLogs.metadata`.

Kafka и ClickHouse предоставляют ту же разбивку как типизированные поля, включая `web_search_requests` и `web_search_cost`. Используйте этот аналитический путь, когда требуется структурированная сверка по типу использования.

### Поиск цены

Когда запрос завершается, роутер вызывает `GetPrice(modelName)`, который нормализует имя и возвращает `*ModelPrice`. Если запись не найдена, расчёт стоимости пропускается, `spend` сохраняется как `0`, а разбивка стоимости в метаданных опускается.